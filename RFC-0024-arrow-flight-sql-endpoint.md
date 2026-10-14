# **RFC-0024 for Presto**

## Arrow Flight SQL endpoint for Presto

Proposers

* Niels Pardon @nielspardon

## [Related Issues]

Related issues may include Github issues, PRs or other RFCs.

* [RFC-0004 — Arrow Flight connector template in Presto](RFC-0004-arrow-flight-connector.md) — the inverse capability: Presto *consuming* data from external Flight services via a connector. This RFC instead exposes Presto *itself* as a Flight SQL server.
* [RFC-0022 — Go Client Library v2](RFC-0022-go-client-v2.md) — context on Presto's client protocol and authentication surface.
* Presto client (wire) protocol: `/v1/statement` REST API.

> A tracking GitHub issue in `prestodb/presto` will be created once this RFC is accepted.

## Summary

This RFC proposes an **optional Arrow Flight SQL endpoint** for Presto. When enabled, the Presto coordinator exposes an [Apache Arrow Flight SQL](https://arrow.apache.org/docs/format/FlightSql.html) service in addition to its existing HTTP/REST client protocol. This lets clients query Presto over Flight SQL — most notably through [ADBC](https://arrow.apache.org/adbc/) (Arrow Database Connectivity) drivers — and receive results as native Apache Arrow record batches instead of the JSON-based REST protocol.

The goals are:

1. **Standards-based connectivity.** Any Flight SQL / ADBC-capable client (Python, Go, Rust, C/C++, Java, R, JDBC-over-ADBC, etc.) can connect to Presto without a Presto-specific client library.
2. **Columnar, zero-copy-friendly result transport.** Arrow is a columnar in-memory format. Delivering results as Arrow avoids row-oriented JSON (de)serialization and integrates directly with the Arrow ecosystem (pandas/Polars, DuckDB, Spark, etc.).
3. **Performance headroom.** Flight is built on gRPC/HTTP2 and supports parallel, partitioned data streams (`FlightEndpoint`s), giving us a path to distribute result fetching across workers in a later phase.

The endpoint is **off by default** and additive: it does not change or replace the existing REST client protocol.

## Background

### What is Arrow Flight SQL / ADBC?

[Apache Arrow Flight](https://arrow.apache.org/blog/2019/10/13/introducing-arrow-flight/) is a gRPC-based framework for high-performance transport of large datasets, transferring Arrow record batches over the wire. **Arrow Flight SQL** is a protocol layered on top of Flight that defines the messages needed to interact with a database: executing SQL queries, retrieving result schemas, listing catalogs/schemas/tables, fetching type information, and managing prepared statements.

**ADBC** (Arrow Database Connectivity) is a vendor-neutral client API — analogous to JDBC/ODBC but Arrow-native — with a Flight SQL driver. A database that speaks Flight SQL is automatically reachable from the ADBC Flight SQL driver in every language ADBC supports.

### Why Presto needs this

Today, clients talk to Presto through the `/v1/statement` REST protocol: a query is `POST`ed, and the client follows a chain of `nextUri` links polling for JSON-encoded result pages. This protocol works well but has limitations for modern analytical and data-science workloads:

1. **Row-oriented JSON.** Results are serialized as JSON, which is verbose and CPU-intensive to produce and parse, and loses type fidelity for some types. Analytical consumers (DataFrame libraries, columnar engines) must then re-encode the data into a columnar form.
2. **No standard ecosystem driver.** Each language needs a Presto-specific client (e.g. `presto-go`, `presto-python-client`, the JDBC driver). Flight SQL / ADBC would let Presto reuse the broad, maintained ADBC driver ecosystem.
3. **Columnar tools are first-class citizens now.** Tools like pandas, Polars, DuckDB, and Spark consume Arrow natively. A Flight SQL endpoint makes Presto a drop-in source for these tools with minimal impedance mismatch.
4. **Industry direction.** Other engines (e.g. Dremio, InfluxDB 3.x, and Trino's Arrow-based spooling) have adopted Arrow/Flight transport. Presto already has substantial Arrow investment via the Arrow Flight *connector* (RFC-0004) and via Velox's Arrow interoperability in Prestissimo. Reusing that investment on the *serving* side is a natural next step.

This RFC is explicitly the **complement** of RFC-0004. RFC-0004 makes Presto a Flight *client* of other systems. This RFC makes Presto a Flight *server* for its own query results.

### Goals

* Provide an optional, standards-compliant Arrow Flight SQL endpoint on the Presto coordinator.
* Reuse Presto's existing authentication and access-control stack — the same identities, authenticators, and authorization rules that apply to REST apply to Flight SQL.
* Map results faithfully to the Arrow type system with no loss relative to the REST protocol.
* Support both Java Presto and Prestissimo (native) execution.
* Define a phased path: a correct coordinator-side implementation first, distributed worker-side Arrow streaming later.

### Non-goals

* **Replacing the REST protocol.** The REST client protocol remains the default and is unaffected.
* **Defining a new query language or semantics.** Flight SQL carries ordinary Presto SQL; this RFC does not change SQL parsing, planning, or execution semantics.
* **A new Presto-built client library.** We rely on the existing ADBC / Flight SQL client ecosystem. (A thin Presto-flavored convenience wrapper is out of scope but not precluded.)
* **Server-to-server / internal communication.** This endpoint is for external clients querying Presto; it is not a replacement for Presto's internal task/exchange protocol.
* **Write/DML semantics beyond what Presto SQL already supports.** Flight SQL's bulk-ingest (`DoPut`) action is out of scope for the initial proposal.

## Proposed Implementation

### Overview

We add an optional Flight SQL service hosted **inside the coordinator process**, listening on a dedicated gRPC port. It is enabled via configuration and disabled by default. Internally it is implemented as a separable server module so its dependencies (gRPC, Arrow, Flight) are only loaded when the endpoint is enabled.

```
                         ┌──────────────────────────────────────────┐
                         │              Coordinator                  │
  ADBC / Flight SQL      │                                           │
  client  ──gRPC/HTTP2──▶│  Flight SQL service  ──▶ existing query   │
  (Python, Go, Rust,     │  (this RFC)              dispatch/engine  │
   Java, C++, JDBC-ADBC) │      │                       │           │
                         │      └── Arrow encoder ◀──────┘           │
                         │              (pages → Arrow batches)      │
                         └──────────────────────────────────────────┘
                                 (REST /v1/statement endpoint
                                  continues to work unchanged)
```

### Modules involved

* **`presto-flight-sql-server` (new, Java).** The Flight SQL service: a gRPC server implementing the Flight + Flight SQL producer interfaces, wired into coordinator startup as an optional module. Depends on `arrow-flight-sql` and `arrow-vector`.
* **`presto-main` (coordinator).** Lifecycle wiring (start/stop the service with the coordinator), configuration binding, and reuse of the dispatch/query manager and authentication/authorization components.
* **`presto-client` / SPI.** Shared type-mapping utilities (Presto type ↔ Arrow type) where reuse is possible.
* **Prestissimo (native), phase 2.** Worker-side Arrow stream production using Velox's Arrow bridge so result data can be streamed directly from workers (see *Phasing*).

### New concepts / terminology

* **Flight SQL service** — the gRPC endpoint on the coordinator.
* **`FlightDescriptor`** — identifies a query (a SQL command or a prepared-statement handle).
* **`FlightInfo`** — the response describing a result set: its Arrow schema plus one or more `FlightEndpoint`s.
* **`FlightEndpoint` / `Ticket`** — an opaque token a client redeems (via `DoGet`) to stream a partition of the result. In phase 1 there is a single endpoint pointing back at the coordinator; in phase 2 endpoints can point at workers.
* **Arrow encoder** — component that converts Presto result `Page`s / columns into Arrow `VectorSchemaRoot` record batches.

### Flight SQL surface (initial)

The service implements the standard Flight SQL producer actions. The most important:

| Flight SQL action | Behavior in Presto |
|---|---|
| `CommandStatementQuery` (via `GetFlightInfo`) | Submit a SQL statement; plan/start it; return `FlightInfo` with the result Arrow schema and a `Ticket`. |
| `DoGet(Ticket)` | Stream the result of the statement as Arrow record batches. |
| `CommandPreparedStatementQuery` | Execute a previously prepared statement. |
| `ActionCreatePreparedStatement` / `ActionClosePreparedStatement` | Map onto Presto's `PREPARE` / `DEALLOCATE` and parameter binding. |
| `CommandGetCatalogs` / `CommandGetDbSchemas` / `CommandGetTables` / `CommandGetTableTypes` | Map onto Presto metadata queries (`information_schema` / `SHOW`). |
| `CommandGetSqlInfo` / `CommandGetXdbcTypeInfo` | Advertise server capabilities and the Presto→XDBC type mapping. |
| `GetSchema` | Return the Arrow schema of a query without executing it (uses describe-output / prepared metadata). |
| `CancelFlightInfo` / closing the stream | Cancel the underlying Presto query. |

### Code flow (phase 1 — coordinator-side translation)

1. **Handshake / authentication.** The client opens a Flight connection and performs the Flight handshake. The handshake is mapped onto Presto's existing authentication (see *Authentication & authorization*). On success the server issues a bearer token the client sends on subsequent calls; the token maps to a Presto session identity.
2. **`GetFlightInfo(CommandStatementQuery)`.** The service translates the request into a Presto query submission through the **same dispatch path the REST endpoint uses** (query manager / dispatch manager), establishing a session from the authenticated identity and Flight call headers (catalog, schema, session properties passed as Flight metadata).
3. **Schema resolution.** The service obtains the output column metadata for the query and converts it to an Arrow `Schema`. It returns a `FlightInfo` containing this schema and a single `FlightEndpoint` whose `Ticket` encodes the Presto `queryId` (and a cursor).
4. **`DoGet(Ticket)`.** The service resumes the query, and as result `Page`s become available it converts each to an Arrow `VectorSchemaRoot` (one or more record batches) and writes them to the gRPC stream. Backpressure is handled by gRPC flow control; the service pulls the next page only as the client drains the stream.
5. **Completion / error / cancel.** When the query finishes, the stream completes. Query failures are surfaced as Flight errors carrying the Presto error code and message. If the client cancels the stream or calls `CancelFlightInfo`, the service cancels the Presto query.

Pseudo-code for the producer core:

```text
getFlightInfo(descriptor):
    session   = buildSession(authIdentity, flightHeaders)
    queryId   = dispatch.createQuery(session, sql)
    columns   = dispatch.getOutputColumns(queryId)        # may block until schema known
    arrowSchema = toArrowSchema(columns)
    ticket    = encode(queryId, cursor=0)
    return FlightInfo(arrowSchema, [FlightEndpoint(ticket, location=self)])

doGet(ticket):
    (queryId, cursor) = decode(ticket)
    schemaRoot = allocate(arrowSchemaOf(queryId))
    for page in resultPages(queryId, cursor):             # pulls pages as client drains
        fillVectors(schemaRoot, page)                     # Presto Block → Arrow FieldVector
        writer.putNext(schemaRoot)
    writer.completed()
```

### Type mapping

Presto types are mapped to Arrow types. Representative mappings:

| Presto type | Arrow type |
|---|---|
| `BOOLEAN` | `Bool` |
| `TINYINT`/`SMALLINT`/`INTEGER`/`BIGINT` | `Int(8/16/32/64, signed)` |
| `REAL` / `DOUBLE` | `FloatingPoint(SINGLE/DOUBLE)` |
| `DECIMAL(p,s)` | `Decimal128(p,s)` (or `Decimal256` if needed) |
| `VARCHAR`/`CHAR` | `Utf8` |
| `VARBINARY` | `Binary` |
| `DATE` | `Date32` |
| `TIME` / `TIME WITH TIME ZONE` | `Time64(MICROSECOND)` (+ tz handling) |
| `TIMESTAMP` / `TIMESTAMP WITH TIME ZONE` | `Timestamp(MICROSECOND[, tz])` |
| `ARRAY<T>` | `List<T>` |
| `MAP<K,V>` | `Map<K,V>` |
| `ROW(...)` | `Struct` |
| `JSON`, `IPADDRESS`, `UUID`, geo, etc. | `Utf8`/`Binary` with the original Presto type recorded in Arrow field metadata |

A canonical mapping table will be defined in the implementation and exposed through `CommandGetXdbcTypeInfo`. Types without a precise Arrow equivalent are carried as `Utf8`/`Binary` with the Presto type name preserved in field-level Arrow metadata so clients can round-trip/identify them.

### Authentication & authorization

The endpoint **reuses Presto's existing authentication** rather than introducing a parallel scheme:

* The Flight handshake/headers are adapted to Presto's configured authenticators — password/LDAP (username + password), JWT/OAuth2 (bearer), Kerberos, and certificate auth — so the same credentials that work over REST work over Flight SQL.
* After handshake, the client receives a bearer token representing the authenticated session; it is presented on subsequent Flight calls (standard Flight SQL bearer flow) and resolved back to the Presto identity.
* The resulting identity flows into a normal Presto `Session`, so **all existing access control** (system and connector `AccessControl`, row filters / column masks per RFC-0010, etc.) applies unchanged.
* **Transport security:** TLS is supported on the Flight gRPC port; mutual TLS may be used as an additional transport-level client authentication mechanism. Production deployments are expected to enable TLS, since Arrow data and bearer tokens cross the wire.

Session attributes Presto clients normally send as HTTP headers (`X-Presto-Catalog`, `X-Presto-Schema`, session properties, `extraCredentials`, client tags) are carried as Flight call metadata / gRPC headers and mapped to the same session fields.

### Phasing

**Phase 1 — Coordinator-side translation (this RFC's core).**
A single `FlightEndpoint` points back at the coordinator. The coordinator runs the query through the existing engine and encodes result pages to Arrow. This works identically for Java and native execution (native workers already return data the coordinator assembles), is the smallest correct change, and establishes the protocol, type mapping, auth integration, and conformance.

The coordinator is the data funnel in this phase, just as it is for the REST protocol today.

**Phase 2 — Distributed Arrow streaming from workers (future).**
`FlightInfo` returns *multiple* `FlightEndpoint`s whose `Location`s point at workers, and each `Ticket` addresses a partition of the result. Clients fetch partitions in parallel directly from workers, removing the coordinator as the single funnel. This phase leans on:

* Prestissimo/Velox's Arrow bridge to produce Arrow batches on native workers with minimal copying (the same C Data Interface bridge referenced by RFC-0004).
* A mechanism for the coordinator to hand out worker locations + tickets and for workers to serve `DoGet` for their partitions.

Phase 2 is a substantially larger change and is described here only to show the design is forward-compatible; it would be detailed and ratified separately. Notably, this is also where Flight SQL's performance advantage over the REST protocol becomes most pronounced.

### User-facing configuration

New coordinator config properties (names indicative):

```properties
# Enable the optional Flight SQL endpoint (default: false)
flight-sql.enabled=true

# gRPC listener
flight-sql.port=8443

# Transport security (recommended in production)
flight-sql.tls.enabled=true
flight-sql.tls.keystore-path=...
flight-sql.tls.keystore-password=...

# Optional limits
flight-sql.max-concurrent-streams=...
flight-sql.max-batch-rows=...
```

No new SQL grammar, session parameters, or SPI changes are required for phase 1 beyond internal wiring. (Phase 2 would introduce worker-side serving and likely new internal SPI.)

## [Optional] Metrics

* **Adoption / load:** number of active Flight SQL connections and streams, queries submitted via Flight vs REST, bytes/record-batches served over Flight.
* **Performance:** end-to-end latency and throughput (rows/sec, bytes/sec) for representative result sizes, compared against the REST/JSON path; CPU spent in result encoding (Arrow vs JSON).
* **Reliability:** Flight error rate, cancellations, handshake/auth failures.
* These are exposed through Presto's existing metrics/JMX surface so they appear alongside other coordinator metrics.

## [Optional] Other Approaches Considered

1. **Standalone Flight SQL gateway / sidecar process.** A separate process translating Flight SQL ↔ Presto REST. Pros: independent scaling and failure domain, no new heavy dependencies in the coordinator. Cons: an extra hop and extra deserialization (REST JSON → Arrow) that defeats much of the columnar benefit, plus a separate component to operate and secure. We chose **embedded-in-coordinator** so the endpoint shares coordinator state and the existing dispatch/auth path directly, and so phase 2 can stream Arrow without an intermediary. (A gateway remains viable for users who cannot enable the endpoint in-process and could be proposed separately.)
2. **Pluggable/optional server module loaded into the coordinator.** Essentially the chosen approach, framed as a cleanly separable module so gRPC/Arrow/Flight dependencies are only present when enabled. This RFC adopts that packaging discipline within the embedded approach.
3. **Distributed worker streaming from day one.** Maximum performance, but a much larger and riskier change touching workers, exchange, and Prestissimo simultaneously. Deferred to phase 2 to keep the initial change reviewable and correct.
4. **Extending the REST protocol with an Arrow content type instead of Flight.** Would give columnar results without gRPC, but would not deliver the standard ADBC/Flight SQL ecosystem connectivity that is a primary goal, nor the partitioned-stream model needed for phase 2.

## Adoption Plan

* **Impact on existing users:** none by default. The endpoint is off unless `flight-sql.enabled=true`. The REST protocol, existing clients, JDBC driver, and SQL grammar are unchanged. No SPI changes are required for phase 1.
* **New configuration:** the `flight-sql.*` properties above. No new session parameters or SQL grammar.
* **Dependencies:** enabling the endpoint pulls in Arrow Flight SQL and gRPC. These are isolated to the optional module so default deployments are unaffected in size and attack surface.
* **Phasing out old behavior:** nothing is phased out — this is purely additive and complementary to REST.
* **Migration tools:** none required. Users adopt by pointing an ADBC / Flight SQL client at the new endpoint.
* **Teaching / documentation:**
  * New documentation page: enabling and securing the Flight SQL endpoint, configuration reference, TLS/auth setup.
  * Client how-tos: connecting with the ADBC Flight SQL driver from Python, Go, Rust, Java, and via JDBC-over-ADBC.
  * Type-mapping reference (Presto ↔ Arrow ↔ XDBC).
  * A blog post announcing the capability is recommended.
* **Out of scope / future:**
  * Phase 2 distributed worker streaming (separate RFC).
  * Flight SQL `DoPut` bulk ingestion / write paths.
  * A Presto-branded convenience client wrapper over ADBC.

## Test Plan

* **Unit tests** for the Presto→Arrow type mapping (every supported type, including nested `ARRAY`/`MAP`/`ROW`, decimals, temporal types with/without time zone, and nulls) and for ticket encode/decode.
* **Service/integration tests** that start a coordinator with the endpoint enabled and drive it with a real Flight SQL client:
  * Execute queries and assert returned Arrow batches match the REST result (same rows/values/types).
  * Metadata commands (`GetCatalogs`/`GetTables`/`GetTableTypes`/type info) return correct results.
  * Prepared statements with parameter binding.
  * Cancellation (client-initiated and `CancelFlightInfo`) actually cancels the Presto query.
  * Error propagation: Presto errors surface as Flight errors with the correct error code/message.
* **Authentication/authorization tests:** each configured auth mechanism (password/LDAP, JWT/OAuth2, Kerberos, mTLS) over Flight; verify access control, row filters, and column masks apply identically to REST.
* **ADBC conformance:** run the ADBC Flight SQL driver test suite against the endpoint where applicable, in Python and at least one other language.
* **TLS / mTLS** handshake and negative tests (bad cert, expired token).
* **Performance benchmarks:** compare Flight (Arrow) vs REST (JSON) for representative result-set sizes — latency, throughput, and coordinator CPU in result encoding — to quantify the benefit and guard against regressions. PoC numbers will be added here as they become available.
* **Both engines:** the integration suite runs against both Java Presto and Prestissimo (native) to confirm parity.
