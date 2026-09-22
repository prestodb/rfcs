# **RFC-0068 for Presto**

See [CONTRIBUTING.md](CONTRIBUTING.md) for instructions on creating your RFC and the process surrounding it.

## Read-only DuckLake connector

Proposers

* Jianjian Xie (@jja725)

## [Related Issues]

* prestodb/presto#28525: draft implementation of this RFC (read-only, PostgreSQL catalog, Java workers)
* [DuckLake](https://ducklake.select) and its [specification](https://ducklake.select/docs/stable/specification/introduction)

## Summary

Add a `presto-ducklake` connector so Presto can query tables stored in the
DuckLake lakehouse format. DuckLake keeps all table metadata in a SQL
catalog database and stores row data as Parquet files on a file system or
object store. The connector reads the catalog over JDBC, plans splits from
it, and reads the Parquet files with Presto's existing Parquet reader.

Structurally the connector is the Iceberg connector's read side with the
catalog implemented over JDBC. Where Iceberg asks the Iceberg library for a
snapshot, a schema, or a list of file scan tasks, this connector runs the SQL
the DuckLake specification prescribes against the catalog database. The first
version is read-only, supports the PostgreSQL catalog dialect, and runs on
Java workers. Its split and handle shapes are kept compatible with Iceberg's
so that a later Prestissimo (Velox) translation is mechanical.

## Background

DuckLake is an open table format from the DuckDB project. Unlike Iceberg and
Delta, which store table metadata as files next to the data, DuckLake stores
metadata in ordinary SQL tables (`ducklake_snapshot`, `ducklake_table`,
`ducklake_data_file`, and so on) in a catalog database, and stores row data as
Parquet files. Catalog-wide snapshots give cross-table transactional
consistency, and every metadata row carries a `begin_snapshot` and
`end_snapshot`, so time travel is a predicate rather than a separate code
path. Single-user deployments typically keep the catalog in a DuckDB file;
multi-user and production deployments keep it in PostgreSQL.

Teams that adopt DuckDB and DuckLake for ingestion and interactive analysis
have no way to reach those tables from Presto today. The workaround is to
copy the data into Iceberg or Hive tables, which duplicates storage, loses the
DuckLake snapshot history, and lags the source. A native connector lets Presto
serve those tables directly, at a consistent snapshot, with the same results
DuckDB itself returns, including positional deletes, compacted files, and
rows that DuckLake inlined into the catalog.

### Goals

* Correct `SELECT` results over DuckLake tables at any snapshot, matching
  what DuckDB returns, including deletes, compaction, schema evolution, and
  inlined data.
* Time travel by snapshot id and by timestamp, plus a way to discover
  snapshots.
* Coordinator-side pruning from the partition and per-file statistics that
  DuckLake already stores, and table statistics for the cost-based optimizer.
* Reuse of Presto's existing Parquet reader, file system support, and
  caching, and of the Iceberg connector's read-path design, so the new module
  carries as little novel machinery as possible.
* A shape that a Prestissimo implementation can translate without new
  protocol concepts.

### Non-goals

Each of these fails with a clear error in the first version rather than
returning a wrong or partial answer:

* Writes of any kind: DDL, DML, views, `ANALYZE`. Writes need catalog
  transactions, statistics collection, and Parquet writing with field ids,
  which roughly doubles the work; the catalog interface is the seam for
  adding them later.
* Catalog dialects other than PostgreSQL. SQLite, MySQL, and DuckDB-file
  catalogs slot in through a catalog-type switch.
* Presto C++ (native) workers.
* Encrypted data files, files imported with name mapping instead of Parquet
  field ids, deletion vectors stored in Puffin files, DuckLake views, and the
  `variant` and `geometry` types.

## Proposed Implementation

### Modules involved

* New module `presto-ducklake`, package `com.facebook.presto.ducklake`,
  connector name `ducklake`, packaged as `plugin/ducklake`.
* Reused unchanged: `presto-parquet` and the Hive Parquet page source,
  `presto-hive`'s file system, authentication, and caching modules,
  `presto-hive-metastore` for the `HdfsEnvironment` wiring, and the
  PostgreSQL JDBC driver already in the build.
* Touched only for wiring: the root `pom.xml` module list, the documentation
  toctree, and the server provisio packaging. No engine, SPI, or other
  connector changes.

### DuckLake facts the design relies on

* A catalog row is visible at snapshot `S` when
  `S >= begin_snapshot AND (S < end_snapshot OR end_snapshot IS NULL)`.
  Snapshot ids are catalog-wide, so one id names the same moment for every
  table.
* Data files are Parquet whose `field_id` equals the DuckLake `column_id`,
  nested fields included. Column identity survives renames.
* At most one delete file per data file per snapshot. Delete files are
  Parquet with `file_path` and `pos` columns, the same layout as Iceberg
  positional deletes, plus an optional `_ducklake_internal_snapshot_id`
  column when deletes from several snapshots were merged.
* Data files produced by compaction may carry a per-row
  `_ducklake_internal_snapshot_id` column and a non-null `partial_max`; a
  read at a snapshot below `partial_max` must filter rows by that column.
* File paths may be relative to the table path, which may be relative to the
  schema path, which may be relative to the catalog's `data_path`.
* Per-file column statistics are string-encoded min/max plus null counts.
  They are bounds, not exact values.
* Partition transforms are `identity`, `bucket(N)`, `year`, `month`, `day`,
  and `hour`; calendar transforms store calendar values (2024, 7, 15), not
  Iceberg's epoch ordinals.
* Small inserts may live in catalog tables named
  `ducklake_inlined_data_<table_id>_<schema_version>`, each row carrying its
  own `row_id`, `begin_snapshot`, and `end_snapshot`.
* A column added after a file was written reads as the column's
  `initial_default` for that file's rows.

### Catalog access

A `DuckLakeCatalog` interface is the only thing the metadata layer, the split
manager, and the inlined-data reader talk to. It resolves snapshots, schemas,
tables, columns, partition specs, data files with their delete files,
partition values, and per-file column statistics, and returns small immutable
model objects. The PostgreSQL implementation issues the specification's SQL
through JDBC and applies the snapshot-visibility predicate to every query.
Every per-table query is also scoped to the table and to the snapshot, so a
long-lived catalog does not make planning slower than the table's live file
count warrants.

A `ducklake.catalog.type` property selects the implementation through a Guice
module switch, as `iceberg.catalog.type` does. `POSTGRESQL` is the only value
in the first version; other dialects add an enum value and a module.

Each transaction gets its own metadata instance with a snapshot-scoped cache
of catalog lookups. Because catalog rows for a given snapshot are immutable,
caching per snapshot is safe and keeps repeated lookups within one query to a
single round trip.

### Metadata, handles, and time travel

Snapshot resolution happens once, when the table handle is created: the
latest snapshot when no version clause is given, the named snapshot for
`FOR SYSTEM_VERSION AS OF`, and the latest snapshot committed at or before the
given time for `FOR SYSTEM_TIME AS OF`; the `BEFORE` variants resolve to a
strictly earlier snapshot. An unknown snapshot id, a time older than the
first snapshot, or a table or schema that did not exist yet at the requested
snapshot each fail with a message naming the problem.

```sql
SELECT * FROM ducklake.sales.orders FOR SYSTEM_VERSION AS OF 42;
SELECT * FROM ducklake.sales.orders
    FOR SYSTEM_TIME AS OF TIMESTAMP '2026-09-09 21:21:45.128 UTC';
SELECT * FROM ducklake.sales."orders$snapshots";
```

The table handle carries the resolved snapshot and the table's schema
serialized as JSON, the role `tableSchemaJson` plays in Iceberg, so workers
never talk to the catalog database for metadata. Column handles carry the
DuckLake `column_id` as their identity, the nested children rebuilt from
`parent_column`, the Presto type, and the column's `initial_default` text.
The layout handle holds the data, predicate, and requested columns, as
Iceberg's does.

Every table also exposes a `"<table>$snapshots"` system table listing
snapshot id, commit time, schema version, author, commit message, and the
changes each snapshot made, which is how users find ids for time travel.

### Type mapping

DuckLake's primitive types map to the obvious Presto types: the signed
integers, `float32`/`float64`, `decimal(P,S)`, `date`, `varchar`, `json`,
`uuid`, and `blob` map one to one; `uint8`, `uint16`, and `uint32` widen to
the next signed type; `time` and every `timestamp` variant truncate to
millisecond precision, as the Iceberg connector does; `timestamptz` maps to
`TIMESTAMP WITH TIME ZONE`; and `list`, `struct`, and `map` map to `ARRAY`,
`ROW`, and `MAP`.

`uint64`, `int128`, `uint128`, `timetz`, `interval`, `variant`, and
`geometry` are unsupported. Loading a table with such a column fails with an
error naming the column. Failing the whole table rather than hiding the
column is deliberate: a hidden column would be a silent difference from every
other DuckLake client. Bulk listings such as `information_schema.columns`
skip such a table and log a warning, so one unsupported table does not break
metadata browsing for the rest of the catalog.

### Split planning

For a scan, the catalog returns in one query the table's current data files
joined with their current delete file, their partition values, and their
per-column statistics. Pruning runs on the coordinator before splits are
emitted:

* Partition pruning handles `identity` exactly and `year`, `month`, `day`,
  and `hour` by applying the transform to the predicate's range bounds.
  `bucket(N)` is not pruned in the first version.
* Statistics pruning parses each file's `min_value` and `max_value` into the
  column's type and intersects them with the predicate domain. A column
  whose bounds are missing, whose `contains_nan` flag is set, or whose value
  fails to parse is skipped for that file.

Both directions err toward reading a file, never toward skipping one.

There are two split kinds. A Parquet split carries the path, offset and
length, file size, partition keys by column id, the delete files with their
sizes and record counts, `row_id_start`, `partial_max`, and a split weight.
This is a subset of `IcebergSplit`, so a native translation to a Velox Hive
split with Iceberg-style positional delete files needs no new fields. An
inlined split names the inlined table and the snapshot and is read entirely
over JDBC. Files that need encryption, name mapping, or Puffin deletion
vectors are rejected at planning time, before any worker opens a file.

### Read path

The page source provider follows the Parquet branch of the Iceberg page
source provider:

* Columns resolve by Parquet `field_id` matching the DuckLake `column_id`,
  nested fields included; the Hive Parquet reader and its subfield pruning
  are used unchanged.
* A column absent from a file yields its `initial_default`, converted once
  per split into a native value. A default the connector cannot represent
  fails the query with an error naming the column rather than reading back
  as `NULL` for every pre-existing row.
* Positional deletes are read into a bitmap and applied as a filter on the
  hidden row-position channel, as Iceberg's position delete filter does.
  When the delete file carries a snapshot column, only deletes at or before
  the query snapshot count.
* For a compacted file whose `partial_max` exceeds the query snapshot, the
  embedded snapshot column is read and rows above the query snapshot are
  dropped.
* Inlined rows are read from the catalog database with the row's own
  snapshot predicate and converted with the same type mapping. The
  PostgreSQL dialect stores several DuckLake types under different physical
  types (for example text for dates and timestamps, `bytea` for strings), so
  the conversion is dialect-aware. Nested types inside inlined rows are not
  supported in the first version. Workers therefore need network access to
  the catalog database, which the connector configuration already provides.
* Hidden columns `$path`, `$row_id` (`row_id_start` plus file position, or
  the inlined row's own id), and `$row_position` are available.

### Statistics

Table statistics for the cost-based optimizer are aggregated from
`ducklake_file_column_stats` over the data files that survive pruning: row
count as the surviving files' record counts minus their delete counts, total
size, per-column null fraction and data size, and min/max ranges for
integer, floating-point, decimal, and date columns. DuckLake stores no
distinct-value counts, so none are reported. Inlined rows are not counted.

`ducklake_table_stats` is deliberately not used for the row count: it counts
rows ever written rather than live rows and is not snapshot-scoped, so it
overstates tables that have seen deletes and is wrong for time travel.

### Configuration and errors

| Property | Meaning | Default |
| --- | --- | --- |
| `ducklake.catalog.type` | Catalog database kind | required, `POSTGRESQL` |
| `ducklake.catalog.connection-url` | JDBC URL of the catalog database | required |
| `ducklake.catalog.connection-user` / `-password` | Catalog credentials | none |
| `ducklake.catalog.schema` | Database schema holding the `ducklake_*` tables | `public` |
| `ducklake.minimum-assigned-split-weight` | Lower bound on a split's weight, as in Iceberg | `0.05` |

The Hive file system properties (S3, GCS, Azure, HDFS configuration
resources) and the Hive Parquet reader and cache session properties apply as
they do for the Iceberg connector. A `DuckLakeErrorCode` enum with its own
base offset covers catalog connection failure, invalid metadata, unsupported
type, unsupported feature, missing data file, and bad delete file.

### Path to native execution

The Parquet split is a subset of the Iceberg split, so the Prestissimo work
is a protocol translation of the DuckLake split to a Velox Hive split with
Iceberg-style positional delete files, plus native handling of the
`initial_default` substitution and the `_ducklake_internal_snapshot_id`
filter, both of which are simple per-row projections. Inlined splits are the
one piece with no native counterpart; the options are to read them on the
coordinator and ship them as values, or to give the native worker a small
JDBC path. That decision is left to the native follow-up.

### Related engine observations

Building the connector surfaced three engine behaviours worth their own
issues; none is changed by this proposal:

* `MetadataUtil.getOptionalTableHandle` falls back to an unversioned lookup
  when a connector's versioned lookup returns `null`, so a table absent at
  the requested snapshot would silently be served from the latest snapshot.
  The connector throws `TableNotFoundException` instead.
* The analyzer resolves a versioned table reference's columns from the
  current schema, so time travel pins the data but not the column set. The
  Iceberg connector behaves the same way; the connector documents it.
* `presto-parquet` reads and writes UUID halves byte-swapped relative to
  `UuidType`'s canonical layout. The connector compensates on read.

## [Optional] Other Approaches Considered

* **Extend the Iceberg connector.** DuckLake's semantics differ in ways that
  pervade the Iceberg code: catalog-wide snapshots, calendar partition
  values, inlined data, and a catalog reached by SQL rather than through the
  Iceberg library's typed objects. Reusing the design while keeping a
  separate module avoids threading DuckLake conditionals through Iceberg.
* **A JDBC connector that pushes queries to DuckDB.** DuckDB would do all
  scanning on one machine, defeating distributed execution, and every worker
  would need a DuckDB runtime. Reading the catalog and files directly keeps
  Presto in charge of parallelism, caching, and memory accounting.
* **Start from the Delta or Hudi connectors.** The Iceberg connector is the
  healthier template for a Parquet-plus-positional-deletes read path, and
  matching its shapes is what makes the native follow-up mechanical.
* **Row counts from `ducklake_table_stats`.** Rejected for the reasons in the
  Statistics section.

## Adoption Plan

* No impact on existing users: the connector is a new, optional plugin with
  no engine, SPI, or SQL grammar changes and no new third-party dependencies.
* New configuration properties are listed above; the only new session
  property is `minimum_assigned_split_weight`, alongside the reused Hive
  Parquet and cache properties.
* A documentation page under the connector section describes configuration,
  time travel, the type mapping, and the limitations, and is part of the
  implementation.
* Out of scope for this RFC, to be addressed independently: writes, other
  catalog dialects, native execution, encryption, name mapping, Puffin
  deletion vectors, views, the unsupported types, `bucket(N)` pruning,
  nested inlined values, and connection pooling for the catalog.

## Test Plan

The implementation is tested against a real DuckLake catalog rather than a
hand-written approximation: a `pg_dump` of a catalog written by the DuckLake
extension into PostgreSQL, plus the Parquet files DuckDB wrote, checked in
under the module's test resources with a script that regenerates them. The
fixture covers TPC-H tiny, every primitive and nested type, an unsupported
type, identity and year/month partitioning, single and rewritten delete
files, schema evolution with a defaulted added column, data inlining, and
file compaction. Tests run against a Testcontainers PostgreSQL loaded from
the dump, with a local-server mode for machines without Docker.

Unit tests cover type mapping, snapshot predicates, partition transform
evaluation, statistics parsing, delete bitmap construction, default-value
parsing, inlined-value conversion, and JSON round trips of every handle and
split. Integration tests cover reads and hidden columns, partition and
statistics pruning, positional deletes, compacted files, schema evolution,
inlined rows, both time travel syntaxes including the error cases, table
statistics through `SHOW STATS` and `EXPLAIN` estimates, metadata queries,
and every read-only guard. The draft implementation passes 292 tests and the
module's full `verify` gate, including checkstyle, spotbugs, pmd, and
dependency analysis.
