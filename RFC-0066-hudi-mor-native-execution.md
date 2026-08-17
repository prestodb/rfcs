# **RFC-0066 for Presto**

## Native execution support for Hudi MERGE_ON_READ tables (Prestissimo/Velox)

Proposers

* Jianjian Xie

## [Related Issues]

* apache/hudi#18308 tracks the broader Hudi-on-Velox effort.
* facebookincubator/velox#15790 adds a native Avro reader to Velox. This proposal depends on it for reading Avro-encoded log blocks.
* facebookincubator/velox#18530 is the companion enhancement issue for the Velox side of this proposal.

## Summary

Enable Presto C++ (Prestissimo) workers to run snapshot `SELECT` queries against Hudi MERGE_ON_READ (MOR) tables. Today `presto-hudi` supports MOR only on JVM workers, so native clusters cannot read MOR tables at all. This RFC proposes a native Hudi connector in Velox, built around a C++ HoodieLogFormat reader and a record-key merger in the same style as the Velox Iceberg connector. It also covers the presto-native-execution protocol and converter glue, and the coordinator changes needed to schedule Hudi splits to native workers.

## Background

Hudi MERGE_ON_READ tables store data as base Parquet files plus delta log files. A snapshot read must merge the base rows with the log records by record key, as of the query instant. In Presto today:

* The Java connector (`presto-hudi`) supports both COPY_ON_WRITE and MERGE_ON_READ via `HudiRecordCursors` on top of `HoodieRealtimeInputFormat`. The merge runs inside the JVM worker using the Hudi Java library.
* Presto C++ workers have no Hudi support. Organizations running native clusters either keep a JVM cluster around for Hudi workloads, or restrict themselves to read-optimized queries that silently miss un-compacted updates.
* Velox has no Hudi connector. The Hudi community tracks interest in one in apache/hudi#18308.

The Iceberg connector already solved the same class of problem. There, the Java coordinator handles all catalog and metadata work and sends pre-resolved file lists to the worker, while Velox implements the data-plane readers (positional deletes, equality deletes, deletion vectors) in C++ without an external Iceberg SDK. This proposal applies that architecture to Hudi MOR.

Correctness is defined by the existing JVM path: for the same table and instant, the native path must return the same rows that `HudiRecordCursors` returns today.

### Goals

* MOR snapshot reads on native workers: base file plus log files, merged by record key as of the query instant.
* Read-optimized MOR and plain COW reads through the same code path (an empty log-file list).
* Partitioned and non-partitioned tables.
* Log-only file slices with no base file. These occur routinely on MOR tables and are handled, not rejected.
* The native path is gated by a configuration flag that defaults to off, with fallback to the existing JVM path.

### Non-goals

The following are out of scope for the first version. The design should not make any of them harder later:

* Time-travel and incremental queries.
* Writes, compaction, clustering.
* Pushing filters or projections into the base-file reader or log-block reader. The first version reads all columns and lets Velox filter above the reader.
* Using the Hudi metadata table for file listing or data skipping on the native side.

## Proposed Implementation

The work spans three layers, each independently testable:

```
Coordinator (Java, presto-hudi)
  HudiSplitManager → HudiSplit { tableBasePath, partitionPath,
                                 baseFile?, logFiles[], instantTime, partitionKeys, ... }
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

### 1. Velox connector (`velox/connectors/hive/hudi/`)

A standalone connector that follows the current Velox Iceberg connector for structure and diverges only in the reader body:

* `HudiConnectorSplit` (extends `HiveConnectorSplit`) adds the table base path, partition path, an optional base file, the log file list, the commit instant, and a read-options map that also carries storage/filesystem configuration. It implements serde the way `HiveIcebergSplit` does.
* `HudiConnector` / `HudiConnectorFactory` (name `"hudi"`), with `HudiConnector` extending `HiveConnector` and creating a `HudiDataSource`.
* `HudiDataSource` (extends `HiveDataSource`) overrides split-reader creation to return a `HudiSplitReader`. Partition-key constants and remaining-filter evaluation reuse the `HiveDataSource` machinery unchanged.
* `HudiSplitReader` holds the Hudi-specific logic. On split preparation it opens the base Parquet file (if present) through Velox's DWIO reader, parses each log file, and builds a record-key index of the log records (inserts, updates, deletes). On `next()` it reads a batch of base rows, replaces or drops rows that the log index overrides, emits log-only inserts after the base scan drains, and drops any `_hoodie_*` meta column the query did not select. Base-only slices (read-optimized MOR and COW) run the same path with an empty log list.

Two new pieces of C++ code sit behind a CMake option (`VELOX_ENABLE_HUDI`, default OFF):

* `HoodieLogFormatReader` (roughly 500-800 lines) parses the Hudi log-file binary format: magic bytes, block headers, and the typed blocks. It reads DATA_BLOCK payloads through Velox's Avro reader (facebookincubator/velox#15790), collects record keys from DELETE_BLOCKs, and applies COMMAND_BLOCK rollbacks. The reference implementation is `HoodieLogFormatReader` in hudi-common (Java).
* `HudiRecordMerger` (roughly 300-500 lines) takes a base-file batch and the record-key index and produces the merged output. Updates replace base rows after precombine resolution, deletes drop them, and log-only inserts are emitted after the base scan. This mirrors what `HoodieRealtimeRecordReader` does in Java.

Both components are synchronous C++ with no async runtime; reads block the driver thread the same way any other file I/O does. Because the merge is by record key rather than positional, no `Mutation` bitmap is involved.

#### Memory

All buffers (base rows, log records, the record-key index) are allocated through Velox's `MemoryPool`, so the memory arbitrator can see, throttle, and reclaim them like any other connector. The record-key index is held in memory during the merge. Its size is bounded by the number of log records in the slice, which the coordinator controls through split sizing. If very large slices become a problem, a streaming merge that processes log blocks incrementally is a possible follow-up.

#### Error handling

Parser and merge errors raise Velox errors carrying the split context (path, instant). A slice that uses a Hudi feature the native path does not handle yet raises a typed "unsupported" error so the coordinator can fall back to the JVM path for that query or table. A missing base file is expected and handled normally.

### 2. presto-native-execution (protocol + converter)

* A `presto_protocol` connector yml declares the `HudiSplit` protocol subclass with `key: hudi` (plus the types it references), wired into the protocol codegen the same way the Iceberg yml is.
* `HudiPrestoToVeloxConnector` converts the deserialized JSON `HudiSplit` into a Velox `HudiConnectorSplit` (base file, log files, instant, table base path, partition keys, storage options). Column-handle and table-handle conversion delegate to the Hive base, exactly like `IcebergPrestoToVeloxConnector`.
* Registration adds the `"hudi"` protocol converter and the Velox `HudiConnectorFactory`.

### 3. Presto Java (`presto-hudi`)

`HudiSplit` is JSON-annotated today but never actually travels to a native worker. The coordinator-side changes:

* Ensure `HudiSplit`/`HudiFile` serialize everything the native converter needs: table base path, partition path, base file path and length, log file paths, instant time, partition keys, storage options.
* Register the connector as native-compatible so the coordinator schedules Hudi splits to native workers, following what `presto-iceberg` and `presto-hive` do (handle resolver, connector serde, native protocol recognition).
* Add a configuration flag `hudi.native-execution-enabled`, default off, that routes eligible queries to native workers and otherwise falls back to the JVM `HudiRecordCursors` path (also on unsupported slices).

### Data flow (snapshot query)

1. The coordinator plans the scan. `HudiSplitManager` lists the latest merged file slices at or before the query instant (existing `getLatestMergedFileSlicesBeforeOrOn` logic) and emits `HudiSplit`s carrying each base file and its logs.
2. Splits serialize as `_type: "hudi"` JSON to the native workers.
3. On the worker, the protocol layer deserializes the split, the converter builds a `HudiConnectorSplit`, and the Velox Hudi connector takes over.
4. `HudiSplitReader` reads the base Parquet file, parses the log files, merges by record key, and rows flow up the Velox pipeline like any other scan.

Schema and partition specifics:

* The base file's Parquet schema and the log blocks' Avro schema are both read into Velox row types and projected onto the reader output type. The `_hoodie_*` meta columns appear only when the query selects them.
* Partition columns arrive as constants from the split's partition keys, the Hive way; they are not read from files.
* Filters and projections stay above the reader in the first version. Pushdown into the base reader and log blocks is a later optimization.

### Risks / open items

1. HoodieLogFormat fidelity. The log-file binary format is defined by Hudi's Java source (`HoodieLogFormat.java`, `HoodieLogBlock.java`) rather than a standalone spec. A spike that parses a known log file and diffs the result against the Java reader settles the format details (block types, magic-byte versions, corrupt and partial-block handling) before the full build.
2. Avro reader readiness. The Velox Avro reader (facebookincubator/velox#15790) is still in review. If it does not land in time, the connector can start with Parquet-based log blocks (Hudi 1.x supports both) and add Avro log-block support when the reader merges.
3. Coordinator-to-native routing. We need to confirm the exact registration that makes the scheduler send splits of a non-Hive connector to native workers; Iceberg is the reference.
4. Filesystem and credential parity. The log reader uses Velox's filesystem API, so it inherits the same storage access (S3/HDFS/local) and auth as the Parquet reader. This needs verification for all storage backends.
5. Schema evolution across log blocks. If the table schema changed between log blocks (column adds, renames, type widening), the merge must handle mismatched schemas. The Java reader relies on Avro schema resolution, and the Velox Avro reader needs to support the same.

## [Optional] Other Approaches Considered

### hudi-rs via C FFI

hudi-rs, the Apache Hudi Rust implementation, ships C FFI bindings that return merged data as an `ArrowArrayStream`, and the Hudi community's stated plan for MOR-on-Velox (apache/hudi#18308) builds on it. Log-format parsing and merge logic already exist there, and using it would keep us aligned with upstream. The costs are what ruled it out for the first version: a Rust toolchain becomes a build dependency of Velox and Prestissimo; merged buffers live in the arrow-rs allocator outside Velox's `MemoryPool`, so the memory arbitrator cannot see or reclaim them; and hudi-rs brings a tokio async runtime into a synchronous reader path. The connector split and protocol layers proposed here do not depend on the reader implementation, so a hudi-rs-backed reader could still be swapped in later if upstream matures.

### Coordinator pre-merge

The coordinator reads log files with the Hudi Java library, performs the merge, writes a temporary merged Parquet file, and the native worker sees a plain Parquet split. This needs no new C++ code, but it adds write-then-read I/O and latency, doubles temporary storage, and pushes CPU work onto the coordinator, which defeats the purpose of native-worker acceleration. Not pursued.

### Pure C++ reader (chosen)

The Velox Iceberg connector shows that implementing format readers in C++ is practical: its delete readers total roughly 700 lines with no external SDK, and memory management, error handling, and threading stay uniform with the rest of Velox. Hudi MOR needs roughly 1000-1500 more lines than Iceberg did (the log-format parser plus merge logic) on top of the Avro reader. In exchange there is no Rust dependency, no FFI boundary, and memory accounting is exact from the start.

## Adoption Plan

* No impact on existing users by default. The native path sits behind `hudi.native-execution-enabled` (default off), and JVM clusters are unaffected entirely.
* No SQL grammar, client API, or SPI changes. New surface area is the configuration flag, the Velox CMake option, and the `hudi` native protocol type.
* When the flag is on, unsupported slices and features fall back to the JVM path rather than failing the query, so the flag can be enabled incrementally per deployment.
* Documentation: the Hudi connector page gains a native-execution section covering the flag and the initial limitations (no pushdown, no time-travel or incremental on native).
* Natural follow-ups, each independent of this RFC: filter and projection pushdown into the base reader, metadata-table-based file skipping on the native side, time-travel and incremental support, and a streaming merge for very large file slices.

## Test Plan

* A spike comes first: a small Velox C++ test that reads a single known Hudi log file, parses the blocks, and verifies the extracted records match the Java reader's output. This settles the binary-format details before the full build.
* Velox unit tests over small MOR tables (prebuilt fixtures): read a slice through `HudiSplitReader` and assert the merged output, including log-only slices and logs carrying updates and deletes.
* End-to-end tests in presto-native-execution, following the Iceberg native test harness: snapshot `SELECT`s on an MOR table (for example the `stock_ticks_mor` style used by Hudi's docker demo) against a native worker.
* A correctness oracle: diff native results against the JVM `HudiRecordCursors` path for the same tables and instants.
* A memory test: confirm all buffers (base rows, log records, record-key index) are allocated through the `MemoryPool` and released properly.
