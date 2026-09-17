# **RFC-0066 for Presto**

## Native execution support for Hudi MERGE_ON_READ tables (Prestissimo/Velox)

Proposers

* Jianjian Xie

## [Related Issues]

* apache/hudi#18308 tracks the broader Hudi-on-Velox effort.
* prestodb/presto#28148 tracks the Presto implementation.
* facebookincubator/velox#18228 tracks the native Avro reader work. This proposal depends on it for reading Avro-encoded log blocks.
* facebookincubator/velox#15790 is the original Avro reader prototype.
* facebookincubator/velox#18530 is the companion enhancement issue for the Velox side of this proposal.

## Summary

Enable Presto C++ (Prestissimo) workers to run `SELECT` queries against Hudi MERGE_ON_READ (MOR) tables, with the query type — snapshot (default) or read-optimized — chosen per query through a catalog session property, the way Spark's `hoodie.datasource.query.type` works. Today `presto-hudi` supports MOR only on JVM workers, so native clusters cannot read MOR tables at all. This RFC proposes a native Hudi connector in Velox, built around a C++ HoodieLogFormat reader and a record-key merger in the same style as the Velox Iceberg connector. It also covers the presto-native-execution protocol and converter glue, and the coordinator changes needed to schedule Hudi splits to native workers. For deployments that read Hudi tables through the Hive connector, an opt-in, Trino-compatible redirect (`hive.hudi-catalog-name`) forwards Hudi-format tables to the Hudi catalog at planning time, so existing `hive.*` SQL reaches the same native path without extending the Hive connector's own Hudi integration.

## Background

Hudi MERGE_ON_READ tables store data as base Parquet files plus delta log files. A snapshot read must merge the base rows with the log records by record key, as of the query instant. In Presto today:

* The Java connector (`presto-hudi`) supports both COPY_ON_WRITE and MERGE_ON_READ via `HudiRecordCursors` on top of `HoodieRealtimeInputFormat`. The merge runs inside the JVM worker using the Hudi Java library.
* Presto C++ workers have no Hudi support. Organizations running native clusters either keep a JVM cluster around for Hudi workloads, or restrict themselves to read-optimized queries that silently miss un-compacted updates.
* Velox has no Hudi connector. The Hudi community tracks interest in one in apache/hudi#18308.

Query-type selection is just as lopsided across engines. Spark is the only engine where the Hudi query type is a per-read choice (`hoodie.datasource.query.type` = `snapshot` | `read_optimized` | `incremental`). In Presto and Trino, the read-optimized-versus-snapshot decision is baked into table registration — which input format the table, or its `_ro`/`_rt` views, were synced with. This RFC brings the per-query model to Presto native execution for the two table views: one registered MOR table, read as snapshot or read-optimized at query time. Incremental (change-pulling) reads remain Spark-only across SQL engines and stay out of scope here. Trino has meanwhile removed Hudi reads from its Hive connector altogether: since Trino 411, `hive.hudi-catalog-name` redirects Hudi-format tables to the Hudi catalog. This RFC adopts the same redirection model for Presto's Hive connector.

The Iceberg connector already solved the same class of problem. There, the Java coordinator handles all catalog and metadata work and sends pre-resolved file lists to the worker, while Velox implements the data-plane readers (positional deletes, equality deletes, deletion vectors) in C++ without an external Iceberg SDK. This proposal applies that architecture to Hudi MOR.

Correctness is defined by the existing JVM path: for the same table and instant, a table that is declared native-compatible must return the same rows that `HudiRecordCursors` returns today.

Hudi read semantics are broader than parsing the log-file container and comparing a precombine field. Hudi table versions, log-block versions and formats, record merge modes, payload classes, record-key generation, rollback blocks, partial updates, and schema evolution can all affect the snapshot result. Reproducing the entire Java behavior in C++ will take time. Native support therefore grows through an explicit compatibility matrix: the first version supports a deliberately narrow, tested subset, and later releases add semantics only after differential testing against the JVM reader. A table outside the supported matrix is rejected before native splits are scheduled; this RFC does not introduce JVM fallback from native execution.

### Goals

* MOR snapshot reads on native workers: base file plus log files, merged by record key as of the query instant.
* Query-level query-type selection: `snapshot` (default) or `read_optimized`, chosen per query via a catalog session property, with a catalog-config default a deployment can change. No separate `_ro`/`_rt` table registration is needed.
* A path for existing Hive-catalog consumers: an opt-in redirect (`hive.hudi-catalog-name`) forwards Hudi-format tables from the Hive connector to the Hudi catalog at planning time, with no SQL changes.
* Read-optimized MOR and plain COW reads through the same code path (an empty log-file list).
* Partitioned and non-partitioned tables.
* Log-only file slices with no base file. These occur routinely on MOR tables and are handled, not rejected.
* The native path is gated by a configuration flag that defaults to off.
* Native compatibility is checked on the coordinator before scheduling. Unsupported table configurations fail with an actionable `NOT_SUPPORTED` error.
* Preserve the existing JVM behavior on JVM clusters.

### Non-goals

The following are out of scope for the first version. The design should not make any of them harder later:

* Time-travel and incremental (change-pulling) queries, including CDC-format change images (Spark's `hoodie.datasource.query.type=incremental` and `incremental.format=cdc`). No SQL engine offers these for Hudi today — they are Spark-only — and the split and reader machinery here is their natural base, but they are follow-up RFC material.
* Extending the Hive connector's own Hudi integration to MOR snapshot. Native workers reject Hive splits that carry Hudi delta-log paths rather than reading the base file and silently dropping updates; Hive-catalog consumers get snapshot semantics through the redirect instead.
* Writes, compaction, clustering.
* Pushing filters or projections into the base-file reader or log-block reader. The first version reads all columns and lets Velox filter above the reader.
* Using the Hudi metadata table for file listing or data skipping on the native side.
* Full semantic coverage for every Hudi table version, log-block format, record merge mode, custom payload, and schema-evolution pattern in the first release. These are added incrementally through the compatibility matrix.

## Proposed Implementation

The work spans three layers, each independently testable, plus a coordinator-only Hive-connector redirect that connects the existing Hive catalog surface to them:

```
Coordinator (Java, presto-hudi)
  HudiSplitManager → HudiSplit { tableBasePath, partitionPath,
                                 baseFile?, logFiles[], instantTime, partitionKeys,
                                 nativeMergeInfo, ... }
        │  JSON over the native protocol (_type: "hudi")
        ▼
presto-native-execution (C++, presto_cpp)
  presto_protocol connector yml   → generated HudiSplit protocol class
  HudiPrestoToVeloxConnector      → builds Velox HudiConnectorSplit
  Registration                    → registers the "hudi" connector + converter
        │
        ▼
Velox (new: velox/connectors/hive/hudi/)
  HudiConnector : HiveConnector
    → HudiDataSource : HiveDataSource
      → HudiSplitReader: reads base Parquet via DWIO, parses log files via a
        native HoodieLogFormat reader, merges by record key, projects to the
        output type (all buffers Velox-native)
```

### Supported semantics and eligibility

The coordinator is responsible for deciding whether a table can use the native path. Before split generation, it reads the Hudi table configuration and validates the declared table semantics against a versioned native compatibility matrix. The worker performs strict validation of block-level details that are visible only while reading log files. The matrix covers at least:

* Hudi table version and timeline layout version.
* Base-file and log-block formats and versions.
* Record merge mode, merge strategy ID, payload class, and precombine field.
* Record-key fields, key generator, and whether Hudi meta fields are populated.
* Rollback, partial-update, CDC, and schema-evolution behavior.

The coordinator converts the supported parts of this configuration into a typed `HudiNativeMergeInfo` carried by the layout or split. This avoids asking the C++ worker to infer table semantics from an arbitrary options map. An unsupported table configuration fails during planning with a message naming the property. An unsupported block type or version that can only be discovered while reading fails the query on the worker. There is no per-split or mid-query fallback to a JVM worker.

The compatibility matrix is expected to expand over time. A semantic is added only after the native implementation has differential fixtures covering the corresponding JVM behavior. The connector documentation lists the supported matrix for each release.

### Query types and query-level selection

The query type is a per-query choice made through a catalog session property, not a property of how the table was registered:

| Property | Scope | Values | Default | Meaning |
| -------- | ----- | ------ | ------- | ------- |
| `hudi.query_type` | session | `snapshot`, `read_optimized` | the catalog default below | Which view of the table the query reads. |
| `hudi.default-query-type` | catalog config | `snapshot`, `read_optimized` | `snapshot` | Deployment-wide default for `hudi.query_type`. |

Semantics:

* **snapshot** — the merged state of the table as of the query instant: base files plus delta logs, merged by record key. The behavior described throughout this RFC.
* **read_optimized** — the latest base files at or before the query instant; log files are excluded from the splits, and log-only file slices are invisible. This is the same result a `_ro`-synced table gives today, without the second table registration. Uncompacted updates and deletes are not visible, so results can trail snapshot by up to the compaction interval. That staleness is the point of choosing it: the query is a plain columnar scan with no log parsing, no record-key index, and no merge, and it is not subject to the native log-byte bound.

For COW tables the two query types return identical results.

The OSS default stays `snapshot`, matching the JVM `presto-hudi` behavior this RFC treats as its correctness oracle. The catalog-config default exists for fleets migrating MOR workloads from Hive-connector deployments whose split listing already resolves Hudi tables to their base files: setting `hudi.default-query-type=read_optimized` keeps results and cost stable through the migration, while any individual query opts into freshness with `SET SESSION hudi.query_type = 'snapshot'`.

Because a read-optimized query is fully described by a split with an empty log-file list, the query type never travels to the worker: the coordinator applies it during file-slice listing, and the native protocol and reader are unchanged by it.

A session property needs no SQL grammar or client changes and is the established Presto mechanism for connector-scoped read options. Incremental (change-pulling) reads are the natural later extension of this surface — Spark models them as a third `query_type` value — but they are out of scope here and belong to a follow-up RFC.

### 1. Velox connector (`velox/connectors/hive/hudi/`)

A standalone connector that follows the current Velox Iceberg connector for structure and diverges only in the reader body:

* `HudiConnectorSplit` (extends `HiveConnectorSplit`) adds the table base path, partition path, an optional base file, the log file list, the query instant, and the typed native merge information validated by the coordinator. It implements serde the way `HiveIcebergSplit` does. Filesystem credentials remain worker catalog configuration and are not serialized in the split.
* `HudiConnector` / `HudiConnectorFactory` (name `"hudi"`), with `HudiConnector` extending `HiveConnector` and creating a `HudiDataSource`.
* `HudiDataSource` (extends `HiveDataSource`) overrides split-reader creation to return a `HudiSplitReader`. Partition-key constants reuse the Hive machinery. Data-column filters are deliberately withheld from the base and log readers; the existing Velox `Filter` operator above the table scan evaluates them after the merged row vector is produced.
* `HudiSplitReader` holds the Hudi-specific logic. On split preparation it opens the base Parquet file (if present) through Velox's DWIO reader, parses each log file, and builds a record-key index of the log records (inserts, updates, deletes). On `next()` it reads a batch of base rows, replaces or drops rows that the log index overrides, emits log-only inserts after the base scan drains, and drops any `_hoodie_*` meta column the query did not select. Base-only slices (read-optimized MOR and COW) run the same path with an empty log list.

Two new pieces of C++ code sit behind a CMake option (`VELOX_ENABLE_HUDI`, default OFF):

* `HoodieLogFormatReader` parses the Hudi log-file binary format: magic bytes, block headers, and the typed blocks included in the current compatibility matrix. It reads DATA_BLOCK payloads through Velox's Avro reader, collects record keys from DELETE_BLOCKs, and applies COMMAND_BLOCK rollbacks. The reference implementation is `HoodieLogFormatReader` in hudi-common (Java).
* `HudiRecordMerger` takes a base-file batch, the record-key index, and `HudiNativeMergeInfo`, then produces the merged output according to the supported record merge mode. Updates replace or combine with base rows, deletes drop them, and log-only inserts are emitted after the base scan. This mirrors the supported behavior of `HoodieRealtimeRecordReader` and `HoodieRecordMerger` rather than assuming all tables use identical precombine semantics.

Both components are synchronous C++ with no async runtime; reads block the driver thread the same way any other file I/O does. Because the merge is by record key rather than positional, no `Mutation` bitmap is involved.

#### Memory

All buffers (base rows, decoded log records, and the record-key index) are allocated through the connector's Velox `MemoryPool`. This makes usage visible to query limits and memory arbitration, but does not by itself make the index reclaimable.

Presto currently creates one Hudi split per file slice and includes all log files in that slice; split weight does not divide the slice. The first version therefore enforces a configurable upper bound on total log-file bytes per native split during coordinator eligibility checking. Slices above the bound are rejected before scheduling. The worker also checks the bound defensively while opening the split.

The in-memory index is non-reclaimable in the first version. A later streaming or spillable merger can remove this restriction, but must provide an explicit `MemoryReclaimer`/spill contract and arbitration tests before the coordinator-side bound is relaxed.

#### Error handling

Parser and merge errors raise Velox errors carrying the split context (path, instant, table version, and merge mode). Unsupported semantics should normally be caught by coordinator eligibility checking. If the worker nevertheless encounters an unsupported or malformed block, the query fails; it is not retried on a JVM worker. A missing base file is expected and handled normally.

### 2. presto-native-execution (protocol + converter)

The Hudi connector has its own Java handle classes; native support requires more than a split subclass. A `presto_protocol` connector yml explicitly declares:

* `HudiColumnHandle` as a `ColumnHandle` subclass.
* `HudiTableHandle` as a `ConnectorTableHandle` subclass.
* `HudiTableLayoutHandle` as a `ConnectorTableLayoutHandle` subclass.
* `HudiSplit` as a `ConnectorSplit` subclass.
* Minimal value types used by these handles, including file descriptors, partition values, and `HudiNativeMergeInfo`.

The native wire shape should be minimal. In particular, it should flatten the fields needed by the worker instead of serializing the full Java `HudiPartition` and Hive metastore `Storage` object graph.

`HudiPrestoToVeloxConnector` explicitly implements:

* Split conversion into `HudiConnectorSplit`.
* `HudiColumnHandle` conversion into the corresponding Hive/Velox column handle.
* `HudiTableLayoutHandle` conversion into a `HiveTableHandle`, including partition columns but no data-column `ScanSpec` filters. The planner's `FilterNode` remains responsible for the unenforced data predicate.

Common Hive conversion helpers may be factored out and reused, but Hudi handles cannot be deserialized as Hive handles. This follows the actual Iceberg pattern: Iceberg has connector-specific protocol classes and explicit split, column-handle, and table-handle conversion.

Registration adds the `"hudi"` protocol converter and `HudiConnectorFactory`. Native workers also require a matching Hudi catalog entry; coordinator and worker upgrade ordering is covered by the adoption plan.

### 3. Presto Java (`presto-hudi`)

`HudiSplit` is JSON-annotated today but never actually travels to a native worker. The coordinator-side changes:

* Bind `ConnectorSystemConfig` in the Hudi connector so it can distinguish JVM and native deployments.
* Add coordinator-side compatibility validation using the Hudi table configuration and known file-slice metadata.
* Serialize a minimal native split containing table and partition paths, base and log file descriptors, query instant, partition values, and normalized `HudiNativeMergeInfo`.
* Register the Hudi handles with the connector protocol and make the catalog available to native workers, following the Iceberg deployment pattern.
* Register the `hudi.query_type` session property in `HudiSessionProperties`, defaulted from the `hudi.default-query-type` catalog config, and apply it during file-slice listing: snapshot lists the latest merged file slices, read-optimized lists the latest base files and emits splits with empty log lists.
* Add a configuration flag `hudi.native-execution-enabled`, default off. JVM clusters continue to use `HudiRecordCursors`. On native clusters, a Hudi scan is accepted only when the flag is enabled and the table passes compatibility validation; otherwise planning fails with `NOT_SUPPORTED`.

### 4. Hive connector redirection (`hive.hudi-catalog-name`)

Most existing deployments read Hudi tables through the Hive connector's input-format integration, under `hive.*` table names. Extending that path to native MOR snapshot would duplicate this RFC's coordinator work against an untyped split — the Hive protocol carries Hudi log files only as `customSplitInfo` strings — with no planning-time hook to validate the compatibility matrix, and would double the differential test surface for identical semantics. Trino resolved the same tension by removing Hudi reads from its Hive connector and redirecting them to the Hudi catalog. This RFC adopts that model:

* A new Hive connector configuration `hive.hudi-catalog-name`, unset by default. Unset means today's behavior: COW and read-optimized reads work as before, and a native worker that receives a Hive split carrying Hudi delta-log paths fails with an actionable error rather than reading the base file and silently dropping updates.
* When set, planning redirects any Hive table whose storage format is a recognized Hudi input format to the same schema and table name in the named Hudi catalog. SQL text is untouched: `hive.schema.table` keeps working, served by the Hudi connector.
* Redirection happens before split generation, so redirected queries get the full Hudi connector surface: compatibility validation, `hudi.query_type`, and the catalog default. A deployment that sets `hudi.default-query-type=read_optimized` on the target catalog preserves the cost and freshness profile its Hive-connector consumers already had, making the redirect behavior-neutral on day one with snapshot an explicit per-query opt-in.
* Presto has no table redirection today. This adds one narrow, additive SPI hook — a default method on `ConnectorMetadata`, modeled on Trino's `redirectTable`, returning an optional target catalog, schema, and table — plus the resolution step in the engine metadata layer. Nothing in the hook is Hudi-specific; Iceberg and Delta redirects can reuse it.
* Access control, masking, and `information_schema` visibility for a redirected table are evaluated against the target catalog.
* The redirect is coordinator-only and independently deliverable: neither the Velox connector nor the native protocol depends on it, and JVM clusters benefit equally.

### Data flow

1. The coordinator loads the Hudi table configuration, validates it against the native compatibility matrix, and builds `HudiNativeMergeInfo`.
2. `HudiSplitManager` lists the latest merged file slices at or before the query instant (existing `getLatestMergedFileSlicesBeforeOrOn` logic), applies the native log-size bound, and emits `HudiSplit`s carrying each base file and its logs.
3. Splits serialize as `_type: "hudi"` JSON to the native workers.
4. On the worker, the protocol layer deserializes the split, the converter builds a `HudiConnectorSplit`, and the Velox Hudi connector takes over.
5. `HudiSplitReader` reads the base Parquet file, parses the log files, and merges by record key. Rows then flow to the existing Velox `Filter` operator and the rest of the pipeline.

The numbered steps describe a snapshot query. A read-optimized query changes step 2 to list the latest base files only and emit splits with empty log lists; nothing downstream of split generation changes.

Schema and partition specifics:

* The base file's Parquet schema and the log blocks' Avro schema are both read into Velox row types and projected onto the reader output type. The `_hoodie_*` meta columns appear only when the query selects them.
* Partition columns arrive as constants from the split's partition keys, the Hive way; they are not read from files.
* Projections into the base reader and log blocks are deferred in the first version. The reader may still add merge-required or filter-only columns that are not part of the final output.

### Filter ordering

Data-column filters must be evaluated after base and log records are merged. Applying a predicate to the base file first is incorrect because a log update can move a row into or out of the predicate.

The existing JVM connector already preserves this ordering:

* `HudiMetadata.getTableLayoutForConstraint` returns the full constraint as the `unenforcedConstraint`.
* The planner therefore keeps a `FilterNode` above the `TableScanNode`.
* `HudiPageSourceProvider` passes `TupleDomain.all()` to the COW Parquet reader, and the MOR path passes no predicate into `HoodieRealtimeInputFormat`.

The native connector preserves the same contract. Partition predicates may still be used by the coordinator for partition pruning. The Hudi protocol converter does not turn data-column domains into Hive `ScanSpec` filters, so `HudiSplitReader` produces merged rows and the existing Velox `Filter` operator evaluates the planner's `FilterNode` above the scan. Dynamic filters on mutable data columns are ignored by the scan in the first version rather than pushed into the base reader. Predicate pushdown is future work and requires Hudi-aware reasoning that proves it is safe across base and log records.


### Risks / open items

1. Semantic convergence. The Java Hudi reader remains the source of truth, but complete parity will take multiple releases. The compatibility matrix, coordinator validation, and differential suite prevent unsupported semantics from silently producing different results.
2. HoodieLogFormat fidelity. The log-file binary format is defined by Hudi's Java source (`HoodieLogFormat.java`, `HoodieLogBlock.java`) rather than a standalone spec. Golden fixtures must cover each supported block and table version, including corrupt, partial, and rollback behavior.
3. Avro reader readiness. The Velox Avro reader is tracked by facebookincubator/velox#18228. General MOR support cannot be declared until the required Avro schema-resolution and decoding capabilities land. A Parquet-log-only implementation may be developed earlier, but is documented and gated as a restricted compatibility subset.
4. Filesystem and credential parity. The log reader uses Velox's filesystem API and worker catalog configuration, so it should inherit the same storage access (S3/HDFS/local) and auth as the Parquet reader. This needs verification for all supported storage backends without sending credentials in splits.
5. Schema evolution across log blocks. If the table schema changed between log blocks (column adds, renames, type widening), the merge must handle mismatched schemas. A schema-evolution case remains unsupported until Velox reproduces the Java/Avro resolution behavior and the differential suite covers it.
6. Memory bound. File slices can contain large log chains. The first release trades coverage for predictable memory by rejecting slices above the configured native limit; streaming or spillable merge is required before lifting that limit.
7. Redirection surface. Cross-catalog redirection moves where access control, masking, and metadata listings are evaluated — to the target Hudi catalog. Deployments with catalog-scoped policies must mirror them on the Hudi catalog before enabling the redirect; it ships unset, and reverting is a configuration change, which keeps the cutover controlled.

## [Optional] Other Approaches Considered

### hudi-rs via C FFI

hudi-rs, the Apache Hudi Rust implementation, ships C FFI bindings that return merged data as an `ArrowArrayStream`, and the Hudi community's stated plan for MOR-on-Velox (apache/hudi#18308) builds on it. Log-format parsing and merge logic already exist there, and using it would keep us aligned with upstream. The costs are what ruled it out for the first version: a Rust toolchain becomes a build dependency of Velox and Prestissimo; merged buffers live in the arrow-rs allocator outside Velox's `MemoryPool`, so the memory arbitrator cannot see or reclaim them; and hudi-rs brings a tokio async runtime into a synchronous reader path. The connector split and protocol layers proposed here do not depend on the reader implementation, so a hudi-rs-backed reader could still be swapped in later if upstream matures.

### Coordinator pre-merge

The coordinator reads log files with the Hudi Java library, performs the merge, writes a temporary merged Parquet file, and the native worker sees a plain Parquet split. This needs no new C++ code, but it adds write-then-read I/O and latency, doubles temporary storage, and pushes CPU work onto the coordinator, which defeats the purpose of native-worker acceleration. Not pursued.

### Pure C++ reader (chosen)

The Velox Iceberg connector shows that implementing table-format-specific readers in C++ can integrate cleanly with Velox memory management, error handling, and threading. A pure C++ Hudi reader avoids a Rust build dependency and FFI boundary and keeps connector-owned buffers inside Velox's memory accounting. The tradeoff is that Presto and Velox take responsibility for tracking Hudi log-format and merge-semantics changes.

#### Maintenance policy for the pure C++ implementation

The C++ implementation is a compatibility layer, not an independent redefinition of Hudi semantics. It follows these maintenance rules:

* **Declared compatibility:** Each release documents the supported Hudi table versions, log-block versions and formats, record merge modes, payload classes, and schema-evolution cases.
* **Default deny:** Unknown table versions, merge strategies, payload classes, block types, or block versions fail validation instead of being interpreted as the closest known behavior.
* **Differential source of truth:** Golden fixtures are generated with released Hudi Java versions and read by both `HoodieRealtimeInputFormat` and the native reader. Native support is expanded only when these results match.
* **Upgrade gate:** Upgrading the Hudi dependency used by `presto-hudi` requires running the native differential suite and reviewing changes to `HoodieLogFormat`, `HoodieLogBlock`, `HoodieRecordMerger`, payload handling, and schema resolution.
* **Isolation:** Log parsing, merge policy, and Velox vector production remain separate components so format-version changes do not silently alter merge semantics.
* **Ownership:** The Velox Hudi connector has named maintainers in the Velox component ownership metadata. The Presto Hudi maintainers own the Java/native compatibility matrix and upgrade tests.
* **Upstream alignment:** Relevant semantic or format changes are tracked through Apache Hudi and Velox issues. Where practical, fixtures and compatibility findings are contributed upstream.

If this maintenance model proves too costly, the protocol and connector layers remain independent of the reader implementation and can adopt a future hudi-rs integration once its allocator and runtime integration satisfy Velox requirements.

## Adoption Plan

* No impact on existing JVM users. JVM clusters continue to use `HudiRecordCursors`.
* The native path sits behind `hudi.native-execution-enabled`, default off. On a native cluster, disabling the flag or using an unsupported table produces `NOT_SUPPORTED`; there is no JVM fallback.
* No SQL grammar or client API changes. One additive SPI default method enables table redirection; existing connectors are unaffected. New surface area is the configuration flag, the query-type session property with its catalog-config default, the `hive.hudi-catalog-name` redirect, the Velox CMake option, and the `hudi` native protocol type.
* Upgrade order is native workers first, then coordinators. Workers without the Hudi connector/protocol must not receive Hudi tasks. During a rolling upgrade, the coordinator-side flag remains off until all native workers have the matching connector and catalog configuration.
* The redirect ships unset. Enabling order: configure the Hudi catalog on coordinators and workers, mirror access-control and masking policies onto it, validate representative tables through the Hudi catalog directly, then set `hive.hudi-catalog-name`. Unsetting it restores the legacy Hive path; no data or metastore changes are involved either way.
* The feature is enabled incrementally by catalog or deployment after representative tables pass compatibility validation and JVM/native differential tests.
* Documentation: the Hudi connector page gains a native-execution section covering the flag, compatibility matrix, memory limit, filter ordering, and initial limitations.
* Natural follow-ups, each independent of this RFC: filter and projection pushdown into the base reader, metadata-table-based file skipping on the native side, time-travel and incremental (change-pulling) queries with CDC-format output — Spark's reader defines their semantics, and the query-type surface here extends to a third value for them — a streaming merge for very large file slices, and reuse of the redirection SPI hook for Iceberg and Delta tables registered in Hive catalogs.

## Test Plan

Testing is organized around the declared compatibility matrix. The Java Hudi reader is the correctness oracle.

### Protocol and eligibility tests

* Java JSON and C++ protocol round trips for `HudiColumnHandle`, `HudiTableHandle`, `HudiTableLayoutHandle`, `HudiSplit`, and `HudiNativeMergeInfo`.
* Coordinator validation tests for every supported and rejected table property, merge mode, payload class, key configuration, declared log format, and schema-evolution category.
* Verify unsupported tables fail before split scheduling and produce an actionable error.
* Verify unsupported block types and versions discovered by the worker fail the query with split and block context.
* Mixed-version deployment tests: new coordinator with an old worker, old coordinator with a new worker, missing worker catalog, and the CMake feature disabled.

### Log parser and merge unit tests

* Golden log fixtures generated by each supported Hudi Java version. Cover data, delete, command/rollback, corrupt, partial, and log-only blocks where applicable to the compatibility matrix.
* Base plus log merges covering inserts, updates, deletes, repeated updates to the same key, duplicate keys across blocks, multiple delta commits, and log-only file slices.
* Record merge modes and payload behaviors are tested separately. Custom or unsupported mergers must be rejected rather than approximated.
* Record-key cases: single and composite keys, null handling, populated and non-populated Hudi meta fields, and partitioned/non-partitioned tables.
* Schema cases: column add, default value, rename, removal, type promotion, and incompatible changes. Each case is either differential-tested as supported or validation-tested as rejected.

### Filter-ordering tests

* Verify a base row that does not match a predicate but is updated by a log record to match is returned.
* Verify a base row that matches a predicate but is updated not to match is removed.
* Cover deletes, log-only inserts, filter-only columns, partition predicates, and dynamic filters on mutable data columns.
* Assert that data-column predicates are absent from the base-file and log-reader `ScanSpec` and are evaluated against the merged output.

### Query-type tests

* Session-property and config validation: invalid `hudi.query_type` values, the catalog-config default applying when the session property is unset, the session property overriding the default in both directions, and both query types against tables that fail compatibility validation.
* Read-optimized parity: query-level `read_optimized` returns the same rows as an equivalent `_ro`-registered table read through the JVM path, and as listing the latest base files directly.
* Read-optimized semantics: uncompacted log updates and deletes are absent, log-only file slices are excluded, and results equal snapshot once compaction has caught up.
* COW equivalence: `snapshot` and `read_optimized` return identical results on COW tables.

### Redirection tests

* With `hive.hudi-catalog-name` unset, legacy behavior is unchanged: COW and read-optimized reads through the Hive connector, and native rejection of Hive splits carrying delta-log paths.
* With it set, `hive.schema.table` over Hudi-format tables returns results identical to querying the Hudi catalog directly — both query types, JVM and native clusters.
* Non-Hudi tables are never redirected. A missing or non-Hudi target catalog fails planning with a clear error.
* Access control resolves against the target catalog, exercised for both granted and denied cases, including column-level policies.
* `hudi.query_type` and `hudi.default-query-type` apply to redirected queries.

### Differential and end-to-end tests

* Run the same snapshot query at the same instant through native workers and JVM `HudiRecordCursors`, then compare unordered results.
* Generate randomized commit sequences containing inserts, updates, deletes, rollbacks, compaction boundaries, and supported schema changes, and compare the native and JVM results after each instant.
* End-to-end tests in `presto-native-execution`, following the Iceberg native test harness, for partitioned, non-partitioned, COW, MOR, and log-only tables.
* Verify filesystem and credential behavior for every storage backend supported by the native deployment.

### Memory and performance tests

* Confirm all connector-owned buffers are allocated from the connector `MemoryPool` and released after the split.
* Verify coordinator and worker enforcement at, below, and above the native log-byte limit.
* Run under memory arbitration and confirm that an oversized or memory-exhausted split fails cleanly without untracked RSS growth.
* Record peak index memory, base/log bytes read, rows inserted/updated/deleted, CPU time, and wall time.
* Compare native and JVM execution on representative MOR tables. Performance results are reported per axis—latency, CPU, scanned bytes, and peak memory—rather than as a single multiplier.
