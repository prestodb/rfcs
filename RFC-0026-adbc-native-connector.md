# **RFC-0026 for Presto**

## Native ADBC Connector for C++ Workers

Proposers

* Jianjian Xie

## Related Issues

* [RFC-0018: Java Connector Federation in C++ Workers](RFC-0018-java-connector-federation.md) solves the same class of problem through a Java Arrow Flight server. See the comparison section below.
* [RFC-0009: JDBC join push down](RFC-0009-jdbc-join-push-down.md): coordinator-side pushdown machinery this RFC reuses.
* [RFC-0004: Arrow Flight connector](RFC-0004-arrow-flight-connector.md): prior art for Arrow-native ingestion in the native worker.
* [velox#18258](https://github.com/facebookincubator/velox/pull/18258): generic ADBC connector for Velox, the scan layer for this RFC.
* [velox discussion #18266](https://github.com/facebookincubator/velox/discussions/18266): design discussion for the Velox side.

## Summary

Let C++ workers scan relational databases (MySQL first, then PostgreSQL and other systems with ADBC drivers) directly and in-process through [Arrow ADBC](https://arrow.apache.org/adbc/), a vendor-neutral C API whose drivers return query results as Arrow data.

The coordinator keeps the existing connectors (`connector.name=mysql`, built on `presto-base-jdbc`) for metadata and pushdown planning. When the cluster runs native workers, each split also carries the dialect-specific part of the remote query, already rendered: the quoted `FROM` clause, the `WHERE` clause with `?` placeholders, and the typed constants for those placeholders. The worker adds the `SELECT` list from the scan's column assignments, binds the constants through `AdbcStatementBind`, runs the query through an ADBC driver loaded as a shared library, and imports each Arrow batch into Velox vectors without copying.

This complements RFC-0018 rather than replacing it. Federation can forward splits from any Java connector, at the cost of an extra service and an extra data hop. The ADBC path covers the systems that have ADBC drivers and removes both.

## Background

### Motivation

We are migrating production workloads that use the Presto MySQL connector to Prestissimo. Today those catalogs can't run on a native cluster. The existing option is FlightShim connector federation ([docs](https://prestodb.github.io/docs/current/presto_cpp/federation.html)), which works, but means operating a separate JVM service next to the native cluster and forwarding every read over Arrow Flight. For databases with an ADBC driver, loading the driver directly in the worker is simpler to operate: there is no extra fleet to size, deploy, and monitor, and the catalog setup stays the same as for any other native connector.

The primary motivation is operational simplicity, not a throughput claim. Throughput numbers will come from the MySQL migration (see Metrics).

### How `base-jdbc` executes a scan today

The design maps point for point onto the existing `presto-base-jdbc` flow:

* Metadata, type mapping, and dialect knowledge live in coordinator-side `JdbcClient` plugins. For example, `MySqlClient` supplies the backtick identifier quote.
* `JdbcComputePushdown`, a connector plan optimizer, translates filter expressions into a `JdbcExpression` (a SQL fragment plus a list of bound constants) stored in the table layout.
* `getSplits` produces one `JdbcSplit` per scan, carrying the remote catalog/schema/table name, the pushed-down `TupleDomain`, and the optional `JdbcExpression`. `JdbcMetadata.getTableLayoutForConstraint` reports the constraint as *unenforced*, so the engine re-applies the predicate above the scan. Remote filtering is purely an optimization and never changes results.
* On the Java worker, `QueryBuilder` renders `SELECT <columns> FROM <table> WHERE <TupleDomain conjuncts> AND <JdbcExpression>` as a `PreparedStatement` with `?` placeholders and binds the constants separately. Literals are never inlined into SQL text. Results are read row by row through a `RecordCursor` and converted to pages.

### ADBC and Velox

* **ADBC** is Apache Arrow's database connectivity standard: a stable C API (`AdbcDatabase` / `AdbcConnection` / `AdbcStatement`) implemented by per-database drivers, with results delivered as an Arrow `ArrowArrayStream`. Drivers exist for PostgreSQL, SQLite, Snowflake, Flight SQL, BigQuery, DuckDB, and MySQL ([adbc-drivers/mysql](https://github.com/adbc-drivers/mysql)). A small driver manager with no dependencies loads drivers as shared libraries.
* **Velox has a generic ADBC connector** ([velox#18258](https://github.com/facebookincubator/velox/pull/18258)). One connector serves every database. The driver is chosen by configuration, driver-specific options pass through a config prefix, and each Arrow chunk of the result is imported through the Arrow C bridge, with Velox vectors taking ownership of the driver's buffers.

ADBC drivers produce Arrow and Velox consumes Arrow, so the data path from the database wire protocol to Velox vectors stays in the worker process with one format conversion (wire to Arrow, inside the driver). The JVM isn't in the data path and there's no second serialization hop. This also drops a conversion the Java path always pays: `base-jdbc` reads rows through a `RecordCursor` and converts them to columnar pages (under RFC-0018, the row-to-Arrow conversion happens in the Flight server instead).

### Goals

* Scan ADBC-capable databases from C++ workers, starting with MySQL.
* Reuse the coordinator's existing connectors, metadata, and pushdown planning. Users keep the same catalog on the coordinator.
* Keep the worker generic: adding a database means deploying its ADBC driver and writing a catalog file, plus registering its connector name.

### Non-goals

* Replacing RFC-0018. Systems without ADBC drivers (MongoDB, Elasticsearch, Cassandra, Redis, and any other Java connector) stay with federation.
* Writes (`INSERT`/`CTAS`). The Velox connector is read-only.
* Distributed connectors (Hive, Iceberg, Delta Lake).
* ADBC on the coordinator. Metadata volume is small and JDBC there is not a bottleneck.

## Proposed Implementation

### 1. Architecture Overview

```mermaid
graph LR
    subgraph "Coordinator - Java"
        A[connector.name=mysql<br/>presto-base-jdbc metadata + pushdown<br/>renders FROM/WHERE + parameters per split]
    end

    subgraph "C++ Worker"
        B[jdbc protocol structs +<br/>JdbcPrestoToVeloxConnector]
        V[Velox AdbcConnector]
        C[ADBC driver<br/>.so, in-process]
    end

    D[(MySQL)]

    A -->|JdbcSplit with nativeQuery| B
    B -->|AdbcConnectorSplit| V
    V -->|Prepare, Bind, ExecuteQuery| C
    C -->|MySQL wire protocol| D
    D -->|rows| C
    C -->|ArrowArrayStream, imported without copy| V
```

Split scheduling, the task protocol, and exchanges don't change. The new coordinator-to-worker contract is one optional field on `JdbcSplit` plus C++ protocol structs for the existing JDBC handles.

### 2. Coordinator: `presto-base-jdbc`

There is no new connector module. The change lives in `presto-base-jdbc`, so every `base-jdbc` connector can use it. Phase 1 registers only `mysql` on the worker.

* **Native mode is detected automatically.** `JdbcSplitManager` checks `ConnectorSystemConfig.isNativeExecution()`, the same switch Hive's optimizer provider uses. There is no catalog property to set. Java-worker clusters keep their current behavior exactly.
* **`JdbcSplit` gains `Optional<JdbcNativeQuery> nativeQuery`.** It is absent unless the cluster is native, and Java workers ignore it.
  * `JdbcNativeQuery { String identifierQuote; String fromWhere; List<JdbcParameter> parameters; }`
  * `JdbcParameter { String type; String value; }`, where `type` is the Presto type signature and `value` is a string encoding.
* **The split renders `FROM` and `WHERE`.** `QueryBuilder.buildNativeQuery` uses the same per-column rendering as `buildSql`, without a JDBC `Connection`. It quotes the catalog, schema, and table with the client's identifier quote, renders one conjunct per `TupleDomain` column domain plus the `JdbcExpression`, and appends the same `/* user : queryId */` comment the Java path sends.
* **The `SELECT` list is left to the worker.** `JdbcComputePushdown` only rewrites Filter-over-TableScan, so the projected columns aren't known when splits are created. The worker already has them from the scan's column assignments and quotes each name with the split's `identifierQuote`. Moving the `SELECT` list to the coordinator would need a new optimizer that also covers bare scans, which isn't worth it for Phase 1.

**Parameter encoding.** Constants travel as strings so the protocol stays simple and exact:

| Presto type | `value` |
| --- | --- |
| boolean | `true` / `false` |
| tinyint, smallint, integer, bigint | `Long.toString` |
| date | days since the epoch, `Long.toString` |
| real, double | `Float.toString` / `Double.toString` (may be `1.0E10`, `NaN`, `Infinity`, `-Infinity`) |
| varchar(n), char(n) | the raw string |
| decimal(p,s) | the unscaled integer; the scale comes from `type`, e.g. `decimal(10,2)` with `12345` is 123.45 |

Any other type (timestamp, time, varbinary, ...) has no encoding. A column domain that needs one is left out of the `WHERE` clause, and a `JdbcExpression` with any such constant is dropped whole. Leaving a conjunct out only widens the remote result, and the engine re-applies the full predicate, so this costs pushdown coverage, never correctness.

### 3. Worker: protocol structs

Presto's native protocol structs are generated, never written by hand (see `presto_protocol/README.md`). A new `presto_protocol/connector/jdbc/` directory follows the `hive`, `iceberg`, and `tpch` layout. The YAML lists the Java classes, and the generator emits `presto_protocol_jdbc.{h,cpp}`. Two classes need `special/` overrides: `JdbcSplit`, for the optional `nativeQuery`, and `JdbcColumnHandle`, which needs `operator<` because column handles are used as map keys.

```cpp
namespace facebook::presto::protocol::jdbc {
using JdbcConnectorProtocol = ConnectorProtocolTemplate<
    JdbcTableHandle,
    JdbcTableLayoutHandle,
    JdbcColumnHandle,
    NotImplemented, // InsertTableHandle
    NotImplemented, // OutputTableHandle
    JdbcSplit,
    NotImplemented, // PartitioningHandle
    JdbcTransactionHandle,
    NotImplemented, // DistributedProcedureHandle
    NotImplemented, // DeleteTableHandle
    NotImplemented>; // IndexHandle
} // namespace facebook::presto::protocol::jdbc
```

The protocol is registered under the connector name, which is also the JSON `@type` the coordinator emits: `"mysql"`. Each additional database is one more registration of the same structs under its name (`postgresql`, ...).

### 4. Worker: mapping onto the Velox connector

`JdbcPrestoToVeloxConnector` converts the protocol structs:

* **Column handle:** `JdbcColumnHandle.columnName` becomes a Velox `AdbcColumnHandle`. Names are kept as is, with no lowercasing, since the remote database is case sensitive.
* **Table handle:** a table-mode `AdbcTableHandle`. Its name is only a display name, because every split supplies its own `FROM` clause.
* **Split:** an `AdbcConnectorSplit` with `sqlSuffix = nativeQuery.fromWhere`, the split's `identifierQuote`, and the parameters parsed into typed Velox `Variant`s. A split without `nativeQuery` (the coordinator is not in native mode) fails with a clear user error, as does a parameter value that doesn't parse for its type.

`Registration.cpp` registers the protocol, this mapping, and the Velox `AdbcConnectorFactory`, all under `mysql`, behind a `PRESTO_ENABLE_ADBC` build flag (default `OFF`) that also turns on `VELOX_ENABLE_ADBC_CONNECTOR`.

### 5. Velox: split suffix and parameter binding

[velox#18258](https://github.com/facebookincubator/velox/pull/18258) provides the execution layer: the vendored driver manager, connection and statement lifecycle per split, Arrow stream import with buffer ownership transfer, and driver errors turned into Velox exceptions. This RFC needs two small extensions on top of it:

* **The split can carry the end of the query.** `AdbcConnectorSplit` gains an optional `sqlSuffix` that replaces ` FROM <tableName>`, and an optional `identifierQuote` that overrides the connector's default. The data source builds `SELECT <quoted projected columns><sqlSuffix>`, or `SELECT 1<sqlSuffix>` when no columns are projected (`count(*)`).
* **The split can carry parameters.** `parameterTypes` and `parameters` (typed `Variant`s, with serde). When they're present, the data source builds a one-row Arrow batch, then calls `AdbcStatementPrepare` and `AdbcStatementBind` before `AdbcStatementExecuteQuery`.

Both are generic. Nothing in them is specific to MySQL or to Presto.

### 6. Configuration

Coordinator and worker share the catalog name. Each side reads the properties it needs:

```properties
# Coordinator: etc/catalog/mysql.properties (unchanged)
connector.name=mysql
connection-url=jdbc:mysql://mysql.internal:3306
connection-user=presto
connection-password=...

# C++ worker: etc/catalog/mysql.properties
connector.name=mysql
adbc.driver=/opt/adbc/libadbc_driver_mysql.so
adbc.option.uri=mysql://presto:...@mysql.internal:3306/
```

Every `adbc.option.<name>` key passes through to `AdbcDatabaseSetOption`, so driver-specific options (TLS, timeouts, session settings) need no Presto changes. Worker credentials should come from the deployment's secret mechanism, not plaintext catalog files. That's the same posture as any native connector that talks to external storage.

## End-to-end walk-through

Take this query against the catalog above:

```sql
SELECT name, price FROM mysql.tpch.items WHERE qty >= 5 AND name = 'bolt'
```

**1. Coordinator planning.** The `mysql` connector plans as it always has. Metadata comes from JDBC, and the predicate lands in the layout's `TupleDomain` (`qty >= 5`, `name = 'bolt'`). The layout reports it as unenforced, so a `FilterNode` stays above the scan.

**2. Split creation.** Because the cluster runs native workers, `JdbcSplitManager` adds the rendered native query to the split. The `TaskUpdateRequest` carries it as JSON:

```json
{
  "@type": "mysql",
  "connectorId": "mysql",
  "catalogName": "tpch",
  "tableName": "items",
  "tupleDomain": { "columnDomains": [ ... ] },
  "nativeQuery": {
    "identifierQuote": "`",
    "fromWhere": " FROM `tpch`.`items` WHERE (`qty` >= ?) AND (`name` = ?)/* alice : 20261005_180000_00001_abcde */",
    "parameters": [
      { "type": "integer", "value": "5" },
      { "type": "varchar(64)", "value": "bolt" }
    ]
  }
}
```

The order of `parameters` matches the `?` placeholders. Empty optionals and null fields are omitted, as Jackson does for every other connector. (The field names match the prototype's protocol tests; the values here are illustrative.)

**3. Worker conversion.** The `mysql` protocol parses the split, and `JdbcPrestoToVeloxConnector` turns it into an `AdbcConnectorSplit`: `sqlSuffix` is `fromWhere`, `identifierQuote` is `` ` ``, and the parameters become `INTEGER 5` and `VARCHAR 'bolt'`. The scan's assignments are `name` and `price`, so the data source runs:

```sql
SELECT `name`, `price` FROM `tpch`.`items` WHERE (`qty` >= ?) AND (`name` = ?)/* alice : 20261005_180000_00001_abcde */
```

**4. ADBC calls.** On a connection from the connector's shared `AdbcDatabase`:

```
AdbcStatementNew
AdbcStatementSetSqlQuery(<SQL above>)
AdbcStatementPrepare
AdbcStatementBind(one-row batch: p0 INT32 = 5, p1 UTF8 = "bolt")
AdbcStatementExecuteQuery -> ArrowArrayStream
```

**5. Results.** The driver sends a server-side prepared statement over the MySQL protocol and turns the rows into Arrow batches. Each batch is checked against the expected column names and types, imported into Velox vectors without copying, and passed to the `FilterNode`, which re-applies the predicate.

Driver errors (bad credentials, missing table, a type the driver can't produce) fail the query with the driver's message and ADBC status code.

## Comparison with RFC-0018 (Java Connector Federation)

Federation can run any Java connector, so the two proposals overlap wherever an ADBC driver exists. Elsewhere federation is the only option. Where they overlap:

| Dimension | RFC-0018 federation | This RFC (native ADBC) |
| --- | --- | --- |
| Connector coverage | Any Java connector (JDBC, MongoDB, ES, Cassandra, Redis, ...) | Systems with ADBC drivers (MySQL, PostgreSQL, SQLite, Snowflake, Flight SQL, BigQuery, ...) |
| Data path | DB → Flight server (row to Arrow in the JVM) → Flight RPC → worker | DB → in-process driver (wire to Arrow) → worker; one conversion, no extra hop |
| New infrastructure | A Flight server fleet to deploy, size, monitor, and upgrade | ADBC driver shared libraries on workers; no new service |
| JVM in data path | Yes | No |
| Coordinator side | The same Java connector | The same Java connector; splits also carry the rendered `FROM`/`WHERE` and parameters |
| Pushdown semantics | Whatever the Java connector does | Same planning path. Constants that can't be encoded drop their conjunct, and the engine re-applies the predicate |
| Credential surface | Centralized on the Flight server | Distributed to every worker |
| Connection fan-out | The Flight server can pool and limit connections | Each worker connects directly: N workers × concurrent splits (see Open Questions) |
| Failure isolation | The Flight server is a shared dependency | Worker-local; a driver crash takes down that worker's process |
| Latency | Two network hops plus Flight serialization | One network hop |

**Recommended posture: adopt both.** Use the ADBC path for relational catalogs where drivers exist and federation for everything else. A deployment can split per catalog: `mysql.properties` on ADBC, `mongodb.properties` on federation. Users see the same catalogs, SQL, and pushdown either way.

Two risks of the ADBC path and how we handle them:

* **Driver maturity varies.** The PostgreSQL and Snowflake drivers are mature. The MySQL driver (written in Go, shipped as a shared library) is newer. Deployments should qualify and pin a driver version per catalog.
* **Type-mapping edge cases** (unsigned MySQL integers, zero dates, `DECIMAL` precision) are decided in two places: the driver (wire to Arrow) and the coordinator's JDBC type mapping (which picks the Presto type). When the two disagree, the scan fails with a type error instead of returning wrong data, and the test plan covers the known cases.

## Why the connector lives in Velox

The Velox discussion ([#18266](https://github.com/facebookincubator/velox/discussions/18266)) asked whether this belongs in Velox or in Prestissimo, which already hosts connectors such as `arrow_flight` and the federation connector in `presto_cpp/main/connectors`.

The scan layer (driver loading, the statement lifecycle, Arrow import, parameter binding) has nothing Presto-specific in it, so another engine that embeds Velox, such as Gluten, could use it unchanged. The Presto-specific part is small and stays in Prestissimo: the protocol structs and the mapping from `JdbcSplit`. Velox maintainers have said they're open to either repository, and the split between the layers makes a move cheap if that's preferred: the mapping code depends only on the `AdbcConnectorSplit` and handle constructors. For now the Velox changes are carried in our Velox fork and will be upstreamed with velox#18258.

## Metrics

* Scan throughput and per-split latency against a real MySQL instance, compared with the same catalog on Java workers and through RFC-0018 federation. The migration will produce these, and they will be shared on #18266.
* Rows and bytes per split, ADBC connection open latency, and driver error rates, as runtime stats on the scan operator.
* Remote connection counts per catalog, to check the fan-out concern.

A scan is one non-splittable stream pulled by one driver thread, as it is with `base-jdbc` today (one split per scan). The benchmark will show whether that ceiling matters for the tables we migrate. If it does, the follow-up is coordinator-side split generation over a key range, which helps the Java path too.

## Other Approaches Considered

**Only RFC-0018 federation.** It works, but for relational databases it adds a permanent extra service and an extra hop to the connectors most likely to carry heavy scans.

**A new `presto-adbc` coordinator connector.** An earlier draft of this RFC proposed one. It would have duplicated each database's `JdbcClient`, needed a dialect-selection property, and made users switch `connector.name`. Reusing the existing connectors avoids all three, and the split change is small.

**Inlining literals instead of binding.** An earlier draft inlined a safe literal subset in Phase 1 and deferred binding. Binding through `AdbcStatementBind` matches `QueryBuilder`'s placeholder protocol exactly and avoids dialect-specific string escaping (MySQL's `NO_BACKSLASH_ESCAPES`, PostgreSQL's `standard_conforming_strings`), so it moved into Phase 1.

**Reimplement each wire protocol in C++.** A hand-written MySQL client in Velox would duplicate the work the ADBC driver ecosystem already does and maintains.

**ODBC.** Broad driver coverage, but row-oriented: every result would pay a row-to-columnar conversion in the worker.

**Embedding the Java connector in the worker over JNI.** RFC-0018 rejected this too, because of memory management and debugging across the boundary.

## Open Questions

* **Connection cap in Phase 1?** Worker-to-database fan-out (N workers × concurrent splits) is the main operational objection to direct connections. Should a simple per-worker, per-catalog max-connections setting ship in Phase 1?
* **Per-catalog opt-out.** Native mode is cluster-wide today, so every `mysql` catalog on a native cluster takes the ADBC path. Is a catalog property needed to route one catalog through federation instead?
* **Column expressions.** `MySqlClient` reads geometry columns through `ST_AsBinary(...)`. Phase 1 only scans plain columns, and a geometry column fails at plan conversion. Should the column handle carry the rendered select expression in Phase 2?

## Adoption Plan

* No change for existing users. Java-worker clusters behave exactly as before, and native clusters gain MySQL catalogs. There are no SQL grammar or client API changes. The protocol change is one optional field on `JdbcSplit`.
* Phase 1: MySQL, read-only. Column projection, `TupleDomain` and `JdbcExpression` pushdown with parameter binding for boolean, integer types, real/double, varchar/char, date, and decimal. Prestissimo builds with `PRESTO_ENABLE_ADBC=ON`, and drivers ship as deployment artifacts, not build dependencies.
* Phase 2: PostgreSQL (one more registration of the same structs), timestamp parameters, rendered column expressions, limit pushdown, join pushdown reuse (RFC-0009), and per-catalog connection limits.
* Phase 3 (independent): writes through `AdbcStatementBind` and bulk ingest.
* Documentation: a page on catalog setup for both sides, driver deployment, and when to choose ADBC or federation.

## Test Plan

* **Java (`presto-base-jdbc`):** the native renderer and `buildSql` agree on the `WHERE` text and parameter order. Each encoding has tests, and constants that can't be encoded drop their conjunct. Splits are created only in native mode. A test pins the Jackson property names the C++ structs depend on.
* **Velox:** `TableScan` tests against a fake in-process ADBC driver cover the bound batch's values and types, scans with and without parameters, the split suffix, `SELECT 1` for `count(*)`, error propagation, and serde. No database is needed in CI.
* **Prestissimo:** JSON round trips of `JdbcSplit` and the other handles, and conversion tests for every parameter type including overflow, `NaN`/`Infinity`, and malformed values.
* **End to end:** MySQL 8 in Docker, seeded with int, unsigned int, bigint, decimal, double, varchar, date, NULLs, and an empty table. A coordinator and one native worker loading `libadbc_driver_mysql.so` run full scans, projections, `count(*)`, filters on each type, `IN` lists, `IS NULL`, a filter with a timestamp constant (not pushed down, still correct), a join with `tpch.tiny`, an aggregation, and error cases. Every result is compared with Java workers on the same database.
* **Performance:** scan-heavy queries on the same MySQL catalog through Java workers, federation, and native ADBC, reported as described in Metrics.
