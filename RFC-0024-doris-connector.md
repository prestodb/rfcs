# **RFC-0024 for Presto**

See [CONTRIBUTING.md](CONTRIBUTING.md) for instructions on creating your RFC and
the process surrounding it.

## Apache Doris Connector based on the Arrow Flight base module

Proposers

* ningsh7

## [Related Issues]

* [RFC-0004: Arrow Flight connector template in Presto](RFC-0004-arrow-flight-connector.md)
* [prestodb/presto#22729: Add an Apache Arrow Flight connector template in Presto](https://github.com/prestodb/presto/issues/22729)
* [apache/doris#25514: Doris support for the Arrow Flight SQL protocol](https://github.com/apache/doris/issues/25514)
* [trinodb/trino#29120: Prior art - a working Doris connector PR for Trino by the same proposer](https://github.com/trinodb/trino/pull/29120)
* [trinodb/trino#28735: Interest check - Arrow-Flight-SQL-based connectivity for Doris/StarRocks](https://github.com/trinodb/trino/issues/28735)
* Presto implementation tracking issue to be filed after RFC acceptance, per
  [CONTRIBUTING.md](CONTRIBUTING.md).

## Summary

Add a first-class Apache Doris connector (`presto-doris`,
`connector.name=doris`) that reads data from Doris over the Arrow Flight SQL
protocol by extending the existing `presto-base-arrow-flight` module.

The connector discovers metadata through Flight SQL metadata calls against the
Doris frontend (FE), and executes table scans as Flight SQL statements. The
resulting `FlightInfo` endpoints, which point to Doris backends (BEs), are
mapped one-to-one to Presto splits. Presto workers then pull Arrow record
batches from multiple BEs in parallel and convert them directly into Presto
pages, avoiding row-oriented serialization on the wire.

This would be the first in-tree concrete implementation of
`presto-base-arrow-flight` targeting an open-source database, and would also
serve as a reference implementation for the base module.

## Background

Apache Doris is a widely deployed MPP analytical database. Today, the only way
to query Doris from Presto is through the MySQL connector, because Doris is
compatible with the MySQL wire protocol. This path has structural drawbacks:

1. Row-oriented transfer: Doris stores and computes in columnar blocks, but the
   MySQL protocol forces serialization to rows and re-conversion to columns
   inside Presto.
2. No parallelism: each table scan reads through a single MySQL connection to
   the FE, leaving BE-side parallelism unused.
3. Type and metadata fidelity gaps: for example, `LARGEINT`, `DATETIME(n)`
   precision, and `INFORMATION_SCHEMA` differences between Doris and MySQL
   cause friction with a connector designed for MySQL.

Since version 2.1, Doris exposes an Arrow Flight SQL endpoint configured via
`arrow_flight_sql_port` on both FE and BE nodes. The read path is: the client
authenticates and submits SQL to the FE; the FE plans the query and returns
endpoints pointing at the BEs that hold the result batches; the client then
pulls Arrow record batches from those BEs in parallel. Community benchmarks
report transfer speedups of one to two orders of magnitude compared with
MySQL-protocol reads for large result sets.

Presto already ships `presto-base-arrow-flight` through RFC-0004. It is
intentionally a framework rather than an out-of-the-box connector: a concrete
plugin must extend `BaseArrowFlightClientHandler` and provide server-specific
behavior such as authentication, metadata discovery, and descriptor
construction. This proposal contributes exactly such a concrete plugin for
Doris.

Prior art: the proposer maintains a working Doris connector for Trino
([trinodb/trino#29120](https://github.com/trinodb/trino/pull/29120)) which
validates the overall feasibility and performance of Flight-SQL-based reads
from Doris. That implementation follows a JDBC-style design; this RFC instead
follows Presto's established `presto-base-arrow-flight` architecture, as
described in [Other Approaches Considered](#optional-other-approaches-considered).

### [Optional] Goals

* Out-of-the-box catalog: `connector.name=doris` plus FE address and
  credentials is all that is needed for `SHOW SCHEMAS`, `SHOW TABLES`,
  `DESCRIBE`, and `SELECT` to work directly.
* Parallel scans: one Presto split per Flight endpoint (BE), with
  Arrow-to-Page conversion.
* Sound type mapping for all common Doris types, including nested types.
* Username/password authentication and TLS, reusing base-module SSL
  configuration.
* Architecture compatible with the Prestissimo (Presto C++) Arrow Flight
  connector path defined in RFC-0004, where all Java concrete modules map to a
  single C++ connector.

### [Optional] Non-goals

* Write path, including `INSERT` and `CREATE TABLE AS`.
* DDL.
* Aggregate or join pushdown. Initial scope includes projection, filter, and
  limit pushdown only.
* StarRocks or a fully generic Flight SQL connector, as discussed in
  [Other Approaches Considered](#optional-other-approaches-considered).
* Table statistics integration for the CBO.

## Proposed Implementation

A new Maven module `presto-doris` will depend on
`presto-base-arrow-flight`.

### Plugin wiring

* `DorisPlugin extends ArrowPlugin`: registers connector name `doris`.
* `DorisModule`: Guice bindings for `DorisFlightClientHandler`,
  `DorisFlightConfig`, and a `DorisArrowBlockBuilder` override where
  Doris-specific type conversion is needed.

### Configuration

The connector reuses the base module's server and SSL properties, and adds
Doris-specific catalog properties:

```properties
connector.name=doris
arrow-flight.server=<doris-fe-host>
arrow-flight.server.port=<fe arrow_flight_sql_port>
arrow-flight.server-ssl-enabled=true|false
doris.connection-user=<user>
doris.connection-password=<password>
doris.metadata.cache-ttl=1m
doris.flight.endpoint-rewrite=<map>
```

`doris.metadata.cache-ttl` and `doris.flight.endpoint-rewrite` are optional.
The endpoint rewrite property is described in
[Corner cases](#corner-cases).

### Authentication

`getCallOptions` performs the Flight handshake against the FE using
username/password authentication and attaches the resulting bearer token as
credential call options. Tokens are refreshed on expiry. TLS and mTLS reuse the
base module's SSL properties.

### Metadata

* `listSchemaNames` and `listTables` use Flight SQL metadata commands
  (`CommandGetDbSchemas`, `CommandGetTables`) against the FE, filtering out
  system schemas.
* `getFlightDescriptorForSchema` returns a descriptor wrapping a zero-row
  Flight SQL statement such as `SELECT * FROM <tbl> WHERE 1=0`, or a
  `CommandGetTables` request with schema. The resulting Arrow schema is mapped
  to Presto column metadata.

### Table scan and splits

`getFlightDescriptorForTableScan` builds a SQL statement from the table handle:
the pruned column list, a `WHERE` clause rendered from the pushed-down
`TupleDomain`, and a `LIMIT` when applicable. The statement is wrapped in a
Flight SQL `CommandStatementQuery`.

The coordinator obtains `FlightInfo` from the FE. Each endpoint, consisting of
a ticket plus BE locations, becomes one `ConnectorSplit`. Workers redeem
tickets via `DoGet` directly against BEs and stream record batches.
`ArrowBlockBuilder` converts Arrow vectors into Presto blocks.

### Type mapping

The connector maps Doris types through Arrow into Presto types:

| Doris type | Presto type | Notes |
| --- | --- | --- |
| `BOOLEAN` | `BOOLEAN` | |
| `TINYINT` / `SMALLINT` / `INT` / `BIGINT` | `TINYINT` / `SMALLINT` / `INTEGER` / `BIGINT` | |
| `LARGEINT` | `DECIMAL(38,0)` | Open question: `VARCHAR` instead? |
| `FLOAT` / `DOUBLE` | `REAL` / `DOUBLE` | |
| `DECIMAL(p,s)` | `DECIMAL(p,s)` | DECIMALV3 semantics |
| `DATE` | `DATE` | |
| `DATETIME(n)` | `TIMESTAMP(n)` | No time zone; semantics documented |
| `CHAR` / `VARCHAR` / `STRING` | `CHAR` / `VARCHAR` | |
| `JSON` / `VARIANT` | `JSON` | Fallback `VARCHAR` if needed |
| `ARRAY` / `MAP` / `STRUCT` | `ARRAY` / `MAP` / `ROW` | Recursive mapping |
| `IPV4` / `IPV6` | `VARCHAR` | |
| `HLL` / `BITMAP` / `QUANTILE_STATE` | Unsupported | Clear error message; open question: `VARBINARY` |

### Pushdown

Projection pushdown through column pruning is always enabled. Filter pushdown
is limited to deterministic, `TupleDomain`-expressible predicates on types with
identical comparison semantics in both systems. `LIMIT` pushdown is supported.
Anything else is evaluated by Presto.

Generated SQL uses Doris identifier quoting with backticks, and case
insensitivity is handled explicitly.

### Corner cases

FlightInfo may return BE addresses, such as private IPs in Kubernetes, that
are unreachable from Presto workers. The connector provides an optional
endpoint rewrite or override property and documents the network requirements.

The implementation must handle:

* Empty result sets and zero endpoints.
* Schema drift between planning and execution, failing with a retryable error.
* Query cancellation propagated through Flight cancellation.
* Transient gRPC failures retried at split granularity, safe because scans are
  read-only.
* Doris versions earlier than 2.1, which do not expose a Flight endpoint,
  failing fast with an actionable message.
* `DATETIME` precision greater than Doris supports.
* NULL-only columns represented by Arrow Null vectors.
* Very wide rows or large batches, with memory accounting in the page builder.

### Prestissimo

Per RFC-0004, all concrete Java modules map to the same Prestissimo Flight
connector, and the worker-side data path is implementation-agnostic: a split
contains the ticket and locations needed to fetch data. Since this connector
keeps Doris-specific logic on the coordinator and metadata side, Presto C++
clusters should be able to use it with the existing native Flight connector,
provided the deployment uses an Arrow-Flight-enabled build. End-to-end
certification on Prestissimo is listed as follow-up work.

## [Optional] Metrics

Correctness will be measured by running the TPC-H suite over the Doris catalog
and comparing results with the same data queried through the MySQL connector.

Performance will be measured with scan-heavy TPC-H and ClickBench queries
against the same Doris cluster, comparing the Doris connector with the MySQL
connector. PoC numbers will be attached to this RFC: `<TBD>`.

## [Optional] Other Approaches Considered

Status quo, using the MySQL connector: this works today, but it is
row-serialized and single-stream, and has type fidelity gaps described in the
Background section.

`presto-base-jdbc` plus an Arrow Flight SQL JDBC driver, which is the approach
used by the proposer's Trino PR: this minimizes connector code, but the driver
materializes rows internally, losing columnar page-building benefits. It also
uses one connection per scan instead of endpoint-level splits, bypasses the
`presto-base-arrow-flight` investment, and has no Prestissimo path.

A generic `presto-arrow-flight-sql` connector for any Flight SQL backend, such
as Doris, StarRocks, or Dremio: this is attractive long-term, but type mapping,
identifier semantics, and endpoint behavior are server-specific, making a
generic connector hard to test and guarantee. This RFC scopes to Doris while
structuring the code so that a generic Flight SQL layer could be extracted
later. Reviewer input on this trade-off is explicitly requested.

## Adoption Plan

There is no impact on existing users. This is a net-new, optional plugin and
catalog type. It does not require SPI changes, SQL grammar changes, or new
session properties. The only new user-facing configuration is the catalog
configuration listed above.

Existing Doris-via-MySQL catalogs continue to work unchanged. Users may migrate
catalog by catalog. A documentation section will describe behavioral
differences, including type mapping and pushdown behavior.

Documentation will include a new `connector/doris.rst` page and an entry in
the connector index. A blog post showing setup against Doris 2.1 or later can
be added separately.

Out-of-scope follow-up work includes the write path, aggregate pushdown,
statistics for the CBO, StarRocks or a generic Flight SQL flavor, and
Prestissimo end-to-end certification.

## Test Plan

Unit tests:

* Arrow-vector-to-Presto-block conversion for every supported type, including
  NULLs and extreme values.
* SQL generation tests covering quoting, predicate rendering, and limit
  handling.

Integration tests:

* A Dockerized single-node Doris deployment with FE and BE in CI, following the
  testing-server pattern already used by `presto-base-arrow-flight`.
* Fixtures covering all mapped types.
* TPC-H tiny correctness tests.
* Authentication, TLS, and cancellation tests.

PoC evidence: the proposer's working Trino connector
([trinodb/trino#29120](https://github.com/trinodb/trino/pull/29120))
demonstrates Flight-SQL-based reads from Doris end to end. Comparative
benchmark numbers, comparing the Doris connector with the MySQL connector on
identical data, will be attached: `<TBD>`.
