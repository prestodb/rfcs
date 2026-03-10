# RFC: Go Client Library v2

Proposers

* Ethan Zhang @ethanyzhang

## Related Issues

- [prestodb/presto-go-client](https://github.com/prestodb/presto-go-client) - Current official Go client (unmaintained since early 2025)

## Summary

This RFC proposes replacing the current `prestodb/presto-go-client` with a new Go client library (`presto-go`) as the official Presto Go client v2. The existing client has been effectively unmaintained, carries a mandatory Kerberos dependency, lacks support for modern Go features (generics), and is missing key functionality including OAuth2 authentication, mutual TLS, query introspection APIs, and Trino compatibility. The proposed v2 client addresses all of these gaps while providing a cleaner architecture, better resource management, and comprehensive test coverage.

The v2 client is already in production use by [pbench](https://github.com/prestodb/pbench), Presto's benchmarking tool, and will continue to be actively maintained. Trino compatibility was a deliberate design goal to enable benchmark comparisons between the two engines using a single client library.

## Background

### Problems with the Current Client

The current official Go client (`prestodb/presto-go-client`) has several significant limitations:

1. **Unmaintained**: The repository has seen no meaningful development since early 2025. Open issues and feature requests remain unaddressed.

2. **Mandatory Kerberos dependency**: The `gokrb5` library is a direct dependency of the root module, meaning all consumers pull in Kerberos-related packages (crypto, LDAP, DNS, RPC) even if they never use Kerberos authentication. This adds ~8 transitive dependencies and increases binary size for every user.

3. **No OAuth2 token refresh**: The current client supports static bearer tokens via the `AccessToken` DSN parameter, but has no support for OAuth2 client credentials flow, automatic token refresh, or custom token sources. In long-running workloads, static tokens expire and queries fail silently.

4. **Limited TLS**: Only CA certificate loading is supported. There is no mutual TLS (client certificate) support, no skip-verify option for development, and no exported TLS configuration helper.

5. **No query introspection APIs**: The client only supports query execution via `database/sql`. There is no way to call `/v1/query/{queryId}` for query info, `/v1/cluster` for cluster info, or `/v1/queryState` for query state — APIs that are essential for monitoring, debugging, and building admin tooling.

6. **Verbose complex type handling**: ARRAY, MAP, and ROW types require ~30+ explicit scanner types (`NullSliceBool`, `NullSlice2Bool`, `NullSlice3Bool`, `NullSliceString`, ...) instead of using Go generics.

7. **No Trino compatibility**: Headers are hardcoded to `X-Presto-*`. Users targeting Trino clusters must manually manage header translation. This is a barrier for tools like pbench that need to benchmark both Presto and Trino with a single client library.

8. **Thread safety gaps**: The `Conn` type does not protect header mutations with synchronization, making concurrent use unsafe.

9. **Resource management issues**: Missing request body buffering for 503 retries (body exhaustion), incomplete HTTP response body cleanup, and no explicit network error classification for retry decisions.

10. **No streaming API**: Large result sets must be consumed entirely through `database/sql` rows iteration with no way to process batches incrementally with backpressure control.

### Feature Comparison

| Feature | Current Client (v1) | Proposed Client (v2) |
|---------|-------------------|---------------------|
| Query execution | Yes | Yes |
| `database/sql` driver | Yes | Yes |
| Query info API (`/v1/query`) | No | Yes |
| Cluster info API (`/v1/cluster`) | No | Yes |
| Query state API (`/v1/queryState`) | No | Yes |
| Batch streaming (`Drain`) | No | Yes |
| Pre-minted query IDs | No | Yes |
| Basic auth | Yes (HTTPS only) | Yes |
| Kerberos | Yes (forced dependency) | Yes (opt-in module) |
| Static bearer token | Yes (`AccessToken` param) | Yes (`access_token` param) |
| OAuth2 client credentials flow | No | Yes (opt-in module, automatic token refresh) |
| Mutual TLS | No | Yes |
| TLS skip-verify | No | Yes |
| Trino compatibility | No | Yes (automatic header translation) |
| Go generics for complex types | No (~30 explicit types) | Yes (`NullSlice[T]`, `NullMap[K,V]`, `NullRow[T]`) |
| Interval type parsing | No (string only) | Yes (`time.Duration` for day-to-second) |
| Transaction isolation levels | Basic | All four Presto levels + read-only |
| Session isolation | No | Yes (cloneable sessions with independent state) |
| Thread-safe sessions | No | Yes (`sync.RWMutex`) |
| Retry body buffering | No | Yes (`GetBody` reconstruction) |
| Gzip compression | Implicit | Yes (with proper resource cleanup) |
| Mock test server | No (integration tests only) | Yes (`prestotest` package, stdlib only) |
| CI enforcement | None | 80% coverage threshold, lint, govulncheck |

## Proposed Implementation

### Module Structure

The v2 client is organized as a root module with two optional auth submodules. This keeps the core dependency footprint minimal while allowing users to opt into heavier auth libraries only when needed.

```
github.com/prestodb/presto-go               # root module
  presto/                                    # main package
    utils/                                   # BiMap utility (same module)
    query_json/                              # query info/stats types (same module)
    prestotest/                              # mock server for testing (same module)
    prestoauth/kerberos/                     # separate module (gokrb5 dependency)
    prestoauth/oauth2/                       # separate module (golang.org/x/oauth2 dependency)
```

Auth submodules use `replace` directives for local development and are published as independent Go modules so consumers only import what they need.

### Core Architecture

#### Client and Session

The `Client` type owns the HTTP client and server URL. It embeds a default `Session` which holds all per-request state (catalog, schema, user, timezone, transaction ID, session parameters, client tags, and persistent request options).

```go
client, _ := presto.NewClient("http://presto:8080")
client.User("analyst").Catalog("hive").Schema("default")

// Create isolated sessions for concurrent use
session := client.NewSession()
session.Catalog("iceberg").Schema("warehouse")
```

Sessions are cloneable and thread-safe via `sync.RWMutex`. All setters use a fluent pattern. This enables patterns like connection pooling with per-query session customization.

#### RequestOptions Pattern

Authentication is injected via `RequestOption` functions that modify each outgoing HTTP request. These persist on the session and apply to all requests including `FetchNextBatch`, ensuring auth tokens are present throughout a query's lifecycle.

```go
// Auth modules return RequestOptions
opt, _ := oauth2.NewRequestOption(oauth2.Config{...})
session.RequestOptions(opt)
```

This pattern decouples auth from the core client, enables composition of multiple options, and allows third-party auth implementations without modifying the library.

#### Query Lifecycle

1. `Session.Query()` sends POST to `/v1/statement` and returns `QueryResults`
2. Results are fetched batch-by-batch via `FetchNextBatch()` or streamed via `Drain()`
3. Context cancellation automatically sends DELETE to cancel the query server-side
4. All HTTP response bodies are read and closed on every code path (success, error, retry)

#### Query Info and Performance Analysis (`query_json`)

The `query_json` subpackage provides structured Go types for the full Presto query info JSON response (`/v1/query/{queryId}`), including `QueryInfo`, `QueryStats`, `StageInfo`, `StageExecutionStats`, `OperatorSummary`, `FailureInfo`, and `Session`. These types include custom JSON unmarshalers for Presto-specific formats like SI-unit sizes (`"1.5MB"`) and duration strings (`"2.00m"`).

The `PrepareForInsert()` methods on these types compute derived metrics (bytes/sec, bytes/CPU-sec, rows/CPU-sec), flatten the stage tree, assemble query plans, and format session properties — making them ready for insertion into analytics databases. This enables building query performance investigation and monitoring tools on top of the client without reimplementing Presto's query info parsing.

#### database/sql Driver

The driver is registered as `"presto"` (and supports `"trino"` scheme for Trino clusters). It implements `driver.Driver`, `driver.Connector`, `driver.Conn`, `driver.QueryerContext`, `driver.ExecerContext`, `driver.ConnBeginTx`, `driver.Rows`, `driver.Stmt`, and `driver.Tx`.

**DSN format**: `presto://[user[:password]@]host[:port][/catalog[/schema]][?params]`

Key driver features:
- Client-side parameter interpolation with SQL injection prevention
- Full type mapping including temporal types, intervals, and base64-decoded varbinary
- All four Presto isolation levels (`READ UNCOMMITTED`, `READ COMMITTED`, `REPEATABLE READ`, `SERIALIZABLE`) and `READ ONLY` mode
- Automatic HTTPS upgrade when any `ssl_*` DSN parameter is set
- `ConnectorOption` pattern for programmatic configuration (`WithSessionSetup`, `WithHTTPClient`)

#### Type System

Scalar types map directly to Go types:

| Presto Type | Go Type |
|---|---|
| `BIGINT`, `INTEGER`, `SMALLINT`, `TINYINT` | `int64` |
| `DOUBLE`, `REAL` | `float64` |
| `BOOLEAN` | `bool` |
| `VARCHAR`, `CHAR`, `DECIMAL`, `JSON` | `string` |
| `VARBINARY` | `[]byte` (base64-decoded) |
| `DATE`, `TIMESTAMP`, `TIMESTAMP WITH TIME ZONE`, `TIME`, `TIME WITH TIME ZONE` | `time.Time` |
| `INTERVAL DAY TO SECOND` | `time.Duration` |
| `INTERVAL YEAR TO MONTH` | `string` (Presto's `"Y-M"` format) |
| `ARRAY`, `MAP`, `ROW` | `string` (JSON) |

Complex types use generic scanner types that implement `sql.Scanner` and `driver.Valuer`:

```go
var tags presto.NullSlice[string]       // ARRAY
var props presto.NullMap[string, int]   // MAP
var addr presto.NullRow[Address]        // ROW (scans into struct)
```

### Authentication Modules

#### Kerberos (`prestoauth/kerberos`)

Separate Go module using `gokrb5/v8`. Supports keytab-based authentication with configurable SPN, realm, and krb5.conf path. Available via DSN parameters or programmatic API.

#### OAuth2 (`prestoauth/oauth2`)

Separate Go module using `golang.org/x/oauth2`. Supports:
- Static bearer tokens (pre-obtained JWTs)
- Client credentials flow with automatic token refresh
- Custom `TokenSource` for integration with metadata services, token files, etc.

### Testing Infrastructure

The `prestotest` package provides `MockPrestoServer`, a stdlib-only (`net/http`) mock Presto server for unit testing. It supports:
- Query registration by SQL text with configurable columns, data, and errors
- Batch simulation via `DataBatches` and `QueueBatches` controls
- Latency injection
- No external dependencies (gin was removed)

Beyond testing the client library itself, `prestotest.MockPrestoServer` is an exported package that downstream Go applications can use to test their own Presto-dependent code without spinning up a real cluster. Application developers can register expected queries with canned responses and verify their code handles batching, errors, and edge cases correctly — all in fast, hermetic unit tests.

CI enforces 80% code coverage, runs `go vet`, `staticcheck`, `gofmt`, and `govulncheck` on both the root module and auth submodules.

## Other Approaches Considered

### 1. Incrementally patching the existing client

Rejected because the architectural limitations (mandatory Kerberos dep, no session isolation, hardcoded Presto headers, ~30 explicit nullable types) cannot be fixed without breaking changes. A v2 with a clean design is more maintainable.

### 2. Wrapping the existing client

Rejected because the underlying resource management issues (body leaks, missing retry buffering, thread safety) would persist. A wrapper cannot fix these without reimplementing the HTTP layer.

### 3. Forking the existing client

Rejected because the v1 codebase would require near-total rewrite to support sessions, request options, generics, and modular auth. Starting fresh with a tested architecture was more practical.

## Adoption Plan

### Impact on existing users

- **No breaking changes to the existing v1 client.** The v1 repository remains available for users who depend on it.
- The v2 client is a new module path, so users opt in by changing their import.
- A migration guide will document DSN parameter mapping between v1 and v2.

### Migration path

| v1 DSN Parameter | v2 Equivalent |
|---|---|
| `source` | `source` |
| `catalog` | Path segment: `/catalog` |
| `schema` | Path segment: `/catalog/schema` |
| `session_properties` | Individual query params |
| `KerberosEnabled` | Use `prestoauth/kerberos` module |
| `AccessToken` | `access_token` param or `prestoauth/oauth2` module |
| `SSLCertPath` | `ssl_ca` |
| `custom_client` | `WithHTTPClient` connector option |

### Documentation

- Comprehensive README with examples for all features
- GoDoc documentation on all exported types and functions
- CLAUDE.md for AI-assisted development

### Out of scope for this RFC

- Prepared statement protocol support (Presto uses client-side interpolation)
- Connection pooling beyond what `database/sql` provides
- Async query submission API

## Test Plan

### Current test coverage

The v2 client has comprehensive test coverage enforced at 80% minimum by CI:

- **Unit tests**: Type conversion, DSN parsing, parameter interpolation, interval parsing, isolation level mapping, TLS config building
- **Integration-style tests**: Full query lifecycle using `prestotest.MockPrestoServer` — query execution, batch fetching, context cancellation, transactions with all isolation levels, streaming via Drain, error handling
- **Auth module tests**: Kerberos SPNEGO token injection, OAuth2 static token and client credentials flow, DSN parameter parsing and stripping
- **Race detection**: All tests run with `-race` flag
- **Lint**: `go vet`, `staticcheck`, `gofmt` on root and submodules
- **Security**: `govulncheck` on root and submodules

### Validation plan for adoption

1. Run the v2 client against a live Presto cluster with the same query workloads used to validate the v1 client
2. Verify wire compatibility by comparing HTTP request/response headers and bodies between v1 and v2 for identical queries
3. Benchmark connection setup, query execution, and batch fetching latency against v1
4. Test all auth modes (basic, Kerberos, OAuth2) against appropriately configured clusters
5. Validate Trino compatibility against a Trino cluster
