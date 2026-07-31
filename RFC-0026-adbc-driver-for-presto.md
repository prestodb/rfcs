# RFC: ADBC Driver for Presto

Proposers

* Jianjian Xie @jja725

## Related Issues

- [adbc-drivers/dev#230](https://github.com/adbc-drivers/dev/issues/230) - Proposal to add a Presto driver to the `adbc-drivers` organization
- [jja725/adbc-presto](https://github.com/jja725/adbc-presto) - Working prototype of the driver
- [RFC-0022: Go Client Library v2](RFC-0022-go-client-v2.md) - The Presto Go client that this driver builds on
- [RFC-0004: Arrow Flight Connector](RFC-0004-arrow-flight-connector.md) - Established the canonical Arrow-to-Presto type mapping in `presto-common-arrow`
- [prestodb/rfcs#64](https://github.com/prestodb/rfcs/pull/64) - RFC for a native ADBC connector in C++ workers (complementary server-side work)
- [adbc-drivers/trino](https://github.com/adbc-drivers/trino) - The Trino ADBC driver whose architecture this driver follows

## Summary

This RFC proposes an official ADBC (Arrow Database Connectivity) driver for Presto. The driver is implemented in Go on top of the ADBC `driverbase-go` framework, wrapping the Presto Go client v2 (RFC-0022) over Presto's REST protocol. It requires no server-side changes: any ADBC-capable application — Python (pandas, polars, DuckDB), R, Go, C/C++, Rust, Java — can query Presto and receive results as Arrow record batches through the standard ADBC driver manager, with no per-language client work.

The driver would be hosted in the [adbc-drivers](https://github.com/adbc-drivers) GitHub organization, following the precedent set by the Trino driver, with endorsement and adoption by the PrestoDB community. A working prototype exists at [jja725/adbc-presto](https://github.com/jja725/adbc-presto) with 85 passing integration tests.

## Background

ADBC is an Apache Arrow project that defines a database-neutral API for connecting to databases and retrieving results in Arrow's columnar format. It plays the same role as JDBC/ODBC, but where JDBC and ODBC are row-oriented — forcing analytical clients to transpose row-based results back into columnar structures — ADBC keeps data columnar end to end. For analytical workloads that land in pandas, polars, or Arrow-native compute engines, this removes a serialization/transposition step on every result set.

The ADBC ecosystem has matured quickly: drivers exist for PostgreSQL, SQLite, Snowflake, BigQuery, DuckDB, Flight SQL, and Trino. Client-side, the ADBC driver manager gives Python, R, Go, C/C++, Rust, and Java applications a uniform API. A database that lacks an ADBC driver is increasingly invisible to this tooling. Trino gained a driver through the `adbc-drivers` organization; Presto currently has none.

Presto is well positioned to close this gap cheaply. RFC-0022 delivered a modern, maintained Go client for Presto's REST protocol, and the ADBC project provides `driverbase-go`, a framework that implements most of the ADBC specification generically over a `database/sql`-style driver. The proposed driver composes these two pieces, which is the same architecture the Trino driver uses.

Two alternatives are worth noting briefly. JDBC/ODBC bridges (ADBC ships a JDBC adapter) work today but retain the row-oriented conversion cost and JVM dependency. An Arrow Flight SQL endpoint on the Presto coordinator would offer true columnar wire transfer, but requires substantial server-side work — that direction is being explored independently (see [prestodb/rfcs#64](https://github.com/prestodb/rfcs/pull/64) and the Arrow Flight SQL endpoint proposal), and a REST-based ADBC driver is complementary: it gives Presto users the ADBC API surface today, and the transport underneath can improve later without changing application code.

### Goals

- An ADBC-compliant driver for Presto, hosted in the `adbc-drivers` GitHub organization
- Query execution returning Arrow record batches
- Catalog, schema, and table introspection via the standard ADBC `GetObjects`/`GetTableSchema` APIs
- TLS support (custom CA, mutual TLS, skip-verify) inherited from the Go client v2
- Presto session property passthrough
- Packaging as a cgo shared library consumable by the ADBC driver manager from any bound language

### Non-goals

- Server-side changes to Presto (Arrow Flight endpoint, columnar result encoding)
- Per-language driver implementations or packaging beyond the standard ADBC shared-library mechanism
- Replacing existing Presto clients (JDBC, presto-python-client, presto-go); ADBC is an additional access path

## Proposed Implementation

### Architecture

The driver composes three layers, mirroring the Trino ADBC driver:

```
ADBC application (Python / R / Go / C++ / Rust ...)
        |
ADBC driver manager (per-language bindings)
        |
adbc-presto (Go, cgo shared library)
  ├── driverbase-go        # generic ADBC implementation
  ├── sqlwrapper           # adapts database/sql drivers to driverbase
  └── presto-go client v2  # Presto REST protocol (RFC-0022)
        |
Presto coordinator (/v1/statement REST API)
```

1. **`driverbase-go`** implements the ADBC object model (Database, Connection, Statement) and standard metadata APIs generically.
2. **`sqlwrapper`** bridges driverbase to any Go `database/sql` driver, handling result-set-to-Arrow conversion.
3. **Presto Go client v2** (`presto-go-client/v2`, RFC-0022) provides the REST protocol implementation: query lifecycle, batching, retries, TLS, and authentication.

The Presto-specific code is concentrated in a thin layer: URI parsing, metadata queries, type mapping overrides, and the PrestoDB-specific behaviors described below.

### Connection URIs

The driver accepts three URI forms:

- Native: `presto://user:pass@host:8080/catalog/schema`
- Explicit scheme: `http://host:8080` or `https://host:443`
- Bare: `host:port` (defaults to HTTP)

Standard options (user, catalog, schema, TLS settings) are recognized as URI query parameters or ADBC options. Unrecognized URI query parameters are forwarded to Presto as session properties, so users can set any session property without driver changes:

```
presto://analyst@coordinator:8080/hive/default?query_max_run_time=10m
```

### TLS and authentication

TLS configuration is delegated to the Go client v2: custom CA certificates, client certificates for mutual TLS, and a skip-verify option for development. Basic authentication is supported via the URI userinfo. Additional auth modes available in the Go client (Kerberos, OAuth2 via its opt-in modules) can be exposed as ADBC options in follow-up work.

### Type mapping

Presto already defines a canonical Arrow correspondence in `presto-common-arrow` (`ArrowBlockBuilder.getPrestoTypeFromArrowField`), introduced with the Arrow Flight connector (RFC-0004). The driver targets that mapping:

| Presto type | Arrow type (canonical, per `presto-common-arrow`) |
|---|---|
| `BOOLEAN` | `Bool` |
| `TINYINT` / `SMALLINT` / `INTEGER` / `BIGINT` | `Int8` / `Int16` / `Int32` / `Int64` |
| `REAL` / `DOUBLE` | `Float32` / `Float64` |
| `DECIMAL(p,s)` | `Decimal128(p,s)` |
| `VARCHAR` / `CHAR` | `Utf8` |
| `VARBINARY` | `Binary` |
| `DATE` | `Date32` |
| `TIME` | `Time` |
| `TIMESTAMP` | `Timestamp(ms)` |
| `ARRAY(T)` | `List` |
| `MAP(K,V)` | `Map` |
| `ROW(...)` | `Struct` |

The initial driver deviates from the canonical mapping in three places, because Presto's REST protocol returns results as JSON and the Go client surfaces certain types as strings (see the type table in RFC-0022):

| Presto type | Initial driver mapping | Reason |
|---|---|---|
| `DECIMAL(p,s)` | `Utf8` (decimal string) | REST returns decimals as JSON strings; lossless as text |
| `ARRAY` / `MAP` / `ROW` | `Utf8` (JSON) | REST returns nested values as JSON; Go client exposes them as strings |
| `TIMESTAMP` / `TIME` | millisecond precision | Presto's REST responses carry millisecond precision |

These are transport limitations, not design choices. Aligning fully with the canonical mapping — native `Decimal128` and native Arrow nested types — is planned follow-up work in the driver (parsing the JSON representations into typed Arrow builders), and would become free if Presto later serves columnar results (see Adoption Plan, out of scope).

### PrestoDB-specific behaviors

The driver follows the Trino ADBC driver's structure, but several PrestoDB/Trino differences require Presto-specific handling:

1. **Namespace tracking.** PrestoDB has no `current_catalog`/`current_schema` SQL functions, so the driver tracks the current catalog and schema client-side, updating its state on `USE` and on the `X-Presto-Set-Catalog`/`X-Presto-Set-Schema` response headers, rather than querying the server.
2. **Explicit casts on ingestion.** PrestoDB rejects literals that require narrowing (for example, inserting an integer literal into a `SMALLINT` column), so the bulk-ingestion path emits explicit `CAST(? AS type)` expressions.
3. **Statistics.** `SHOW STATS` returns a different column structure than Trino; the ADBC statistics API is backed by Presto-specific parsing.
4. **Error mapping.** Errors map from Presto's error-type/error-code model to ADBC status codes directly, rather than through SQLSTATE.

### Packaging

- Go module, consumable directly by Go applications via the ADBC Go API
- cgo shared library (`libadbc_driver_presto`) built for the ADBC driver manager, which is how Python, R, C/C++, Rust, and Java consumers load the driver
- Docker-based validation environment that runs the integration suite against a live Presto server

### Modules involved

No Presto server or client modules change. The driver is a new repository (`adbc-drivers/presto` proposed) depending on:

- `github.com/prestodb/presto-go-client/v2` (v2 client, RFC-0022)
- `github.com/adbc-drivers/driverbase-go` (`driverbase`, `sqlwrapper`, `validation`)
- `github.com/apache/arrow-adbc/go/adbc` and `github.com/apache/arrow-go/v18`

## [Optional] Metrics

Impact can be measured by:

- Adoption: driver downloads/imports, GitHub stars/issues on the driver repository
- Ecosystem reach: inclusion in ADBC driver manager documentation and package indexes alongside the Trino/PostgreSQL/Snowflake drivers
- Community signal: issues and PRs from users running the driver against production Presto clusters

## Adoption Plan

- **Impact on existing users: none.** The driver is a new, client-side component. No new session parameters, configuration properties, SPI changes, or SQL grammar in Presto itself.
- **Hosting and governance:** the driver is donated to the `adbc-drivers` GitHub organization (per [adbc-drivers/dev#230](https://github.com/adbc-drivers/dev/issues/230)), where it gains the organization's CI, release, and packaging infrastructure. The PrestoDB community endorses it as the recommended ADBC access path; interested Presto maintainers are welcome as co-maintainers.
- **Documentation:** add an ADBC section to the Presto client documentation on prestodb.io, pointing to the driver with connection examples for Python and Go. A blog post announcing the driver would help adoption.
- **Out of scope, addressable independently:**
  - Native `Decimal128` and Arrow nested-type decoding in the driver (parsing REST JSON into typed builders)
  - Kerberos and OAuth2 ADBC options (the underlying Go client already supports both)
  - A columnar result transport in Presto (Arrow Flight SQL endpoint, [prestodb/rfcs#64](https://github.com/prestodb/rfcs/pull/64)); if that lands, this driver can adopt it transparently while keeping the same ADBC API surface

## Test Plan

The prototype already carries a substantial test suite, which moves with the driver:

- **85 passing integration tests** against a live Presto server, covering query execution, Arrow schema/batch correctness, catalog/schema/table metadata (`GetObjects`, `GetTableSchema`), namespace tracking across `USE` statements, session property passthrough, bulk ingestion with explicit casts, statistics, and error mapping
- **Docker-based test harness** that provisions a Presto server for the integration suite, runnable locally and in CI
- **ADBC validation suite**: `driverbase-go` ships conformance scaffolding used by other drivers in the organization; the Presto driver runs the same conformance tests
- On donation to `adbc-drivers`, the driver adopts the organization's CI (lint, race detection, multi-platform shared-library builds)
