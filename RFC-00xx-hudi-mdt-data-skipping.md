# **RFC-00xx for Presto**

See [CONTRIBUTING.md](CONTRIBUTING.md) for instructions on creating your RFC and the process surrounding it.

## Hudi data skipping with metadata table column statistics

Proposers

* Jianjian Xie (@jja725)

## Related Issues

* apache/hudi#16892 — Support Presto and Trino Hudi connector with MDT and data skipping on 1.x release (HUDI-9110)
* apache/hudi#15246 — Metadata Indexes integration with Presto/Trino (HUDI-4394), Presto sub-issues:
  * apache/hudi#15181 — [presto-hudi connector] Support column stats pruning
  * apache/hudi#16957 — partition stats pruning
  * apache/hudi#16958 / #16968 — expression index pruning / expressions
  * apache/hudi#16959 — record index based skipping
  * apache/hudi#16969 — push index scan down to workers
* apache/hudi#18837 — RFC-105: Trino-Hudi connector moved into the Hudi repository, including MDT index support
* apache/hudi#17042 — hanging Trino queries with column stats enabled (no repro yet)
* Reference implementation: prestodb/presto#28781

## Summary

Use the column statistics that Hudi writers already store in the Hudi metadata table (MDT)
to skip Hudi copy-on-write (COW) Parquet files that cannot match a query's predicate. Files
are pruned on the coordinator during split enumeration, before any split is created. Workers,
split formats and readers are unchanged, so Java and native (Prestissimo) workers both benefit.

The first phase targets Hudi COW tables read through the Hive connector (`HudiDirectoryLister`).
It adds one config property and one session property, both off by default, and a small
connector-independent module, `presto-hudi-common`, that the `presto-hudi` connector can reuse
in a later phase.

## Background

Hudi 0.11+ can maintain a metadata table at `<table>/.hoodie/metadata`. Its `column_stats`
partition stores, for each data file and indexed column, the minimum value, maximum value,
null count and value count. Spark's Hudi integration uses these statistics for data skipping
(`hoodie.enable.data.skipping`). For a selective predicate on an indexed column, it can avoid
opening most files in a partition.

Presto has nothing comparable today:

* The Hive connector's `HudiDirectoryLister` can use the MDT `files` partition for listing
  (`hive.hudi-metadata-enabled`), but it schedules a split for every listed file.
* The `presto-hudi` connector's `HudiSplitManager` can also use the MDT for listing, but it
  does not read `column_stats`.
* Pruning happens only after a split has been scheduled: Parquet footer and row-group
  statistics in the reader. Opening each file still costs a coordinator split, a worker task
  slot, and at least one footer read from object storage.

The upstream Hudi epic for Presto and Trino (apache/hudi#15246, #16892) has made progress only
for Trino. The Trino connector now lives in the Hudi repository (RFC-105, apache/hudi#18837)
with column-stats, partition-stats, record-level and secondary index support. The Presto
sub-issues were triaged in Aug 2026 as blocked: the Presto connector lives in prestodb/presto,
so this work has to happen here. This RFC proposes that work, starting with the
highest-value, lowest-risk part: column-stats pruning for COW tables.

For queries with selective filters on columns that are naturally clustered (event time, IDs,
sequence numbers) or explicitly clustered by Hudi clustering, min/max pruning can remove a
large fraction of files. Because the work happens before splits exist, the savings show up as
fewer splits, less coordinator scheduling work, fewer worker task slots used, and fewer
object-store requests.

### Goals

* Prune Hudi COW Parquet files using MDT `column_stats`, with no false negatives: a file that
  contains a matching row is never pruned.
* Never fail a query because of the optional index. A missing, building or unreadable index
  means "no pruning", except for authorization failures and cancellation.
* Make no changes to workers, splits, readers or the SPI, so the feature works with native
  execution.
* Keep pruning logic connector-independent so the `presto-hudi` connector can adopt it.
* Report observable runtime metrics for the benefit and cost of pruning.

### Non-goals

* Merge-on-read (MOR) tables. Log files can contain rows outside the base file's statistics.
  Sound MOR pruning needs per-file-slice reasoning and is a separate phase.
* Partition stats, record-level index, secondary index and expression index.
  `HudiColumnStatsReader` is designed so these can be added later. See *Future work*.
* Building or maintaining indexes. Presto only reads indexes that writers have produced.
* Pushing index lookups to workers (apache/hudi#16969).

## Proposed Implementation

### Modules involved

| Module | Change |
|---|---|
| `presto-hudi-common` (new) | Connector-independent pruning: stats reader, stats → `Domain` conversion, evaluation, batching. Depends only on `presto-common` and the Hudi bundle. |
| `presto-hive` | Passes the data-column predicate to `HudiDirectoryLister`, wraps its file iterator with a pruning iterator, and adds the config and session properties. |
| `presto-common` | Six new `RuntimeMetricName` constants. |
| `presto-docs` | Hive connector documentation. |

No changes are made to the SPI, `presto-main`, worker code, `presto-native-execution` or Velox.

### New configuration

| Config property | Session property | Default | Description |
|---|---|---|---|
| `hive.hudi-mdt-data-skipping-enabled` | `hudi_mdt_data_skipping_enabled` | `false` | Prune Hudi COW files using MDT column statistics. |

The feature requires `hive.parquet.use-column-names=true`, because the statistics are keyed by
column name. It is independent of `hive.hudi-metadata-enabled`: files can be listed from
storage while statistics come from the MDT, or both can come from the MDT.

### Classes and contracts (`presto-hudi-common`)

```java
// Reads per-file statistics as Presto domains. Implementations decide which MDT state to read.
public interface HudiColumnStatsReader
{
    // key: (relative partition path, file name)
    // value: column name -> Domain that contains every value of that column in the file.
    // A missing file or column means "unknown" and must not cause pruning.
    Map<Pair<String, String>, Map<String, Domain>> readColumnStats(
            List<Pair<String, String>> files, Map<String, Type> columns) throws IOException;
}

public final class LatestHudiColumnStatsReader implements HudiColumnStatsReader { ... }

public final class HudiColumnStatsPruner
{
    public static final int MAX_BATCH_SIZE = 128;   // files per MDT lookup
    // At most 32 predicate columns; wider predicates disable pruning.
    public HudiColumnStatsPruner(TupleDomain<String> predicate, HudiColumnStatsReader reader);
    public boolean isUseful();                      // false if no supported column is constrained
    public Set<Pair<String, String>> retainFiles(List<Pair<String, String>> files) throws IOException;
}

final class HudiColumnStatsConverter { static Domain toDomain(Type type, HoodieMetadataColumnStats stats); }
final class HudiStatisticsEvaluator  { static boolean mayMatch(TupleDomain<String> predicate, Map<String, Domain> fileDomains); }
```

**Statistics to Domain (soundness rules)**

* If `min` and `max` are present, the domain is the range `[min, max]`. If `nullCount > 0`,
  null is allowed. If `nullCount == valueCount`, the domain is `onlyNull`.
* A missing bound, overflow, a deleted entry, duplicate entries for the same file and column,
  or inconsistent counts produce `Domain.all(type)`, which never prunes.
* The Hudi Avro schema type must match the Presto type exactly, with no coercions. A column
  added, dropped or retyped by schema evolution is treated as unknown.
* Phase 1 supports `BOOLEAN`, `INTEGER` and `BIGINT`. Other types produce `Domain.all`.
  `VARCHAR`, `DATE`, `TIMESTAMP` and `DECIMAL` are added once their statistics encodings and
  ordering are checked against Presto semantics. Strings especially need care: Hudi writes
  byte-wise min/max, which may be truncated.

`mayMatch` returns false only when the intersection of the query domain and the file domain is
empty for at least one predicate column.

### Code flow (Hive connector)

```
HiveSplitManager.getSplits
  dataColumnPredicate = layout.domainPredicate restricted to top-level REGULAR columns
  -> BackgroundHiveSplitLoader -> DelegatingPartitionLoader -> StoragePartitionLoader
       -> new HudiDirectoryLister(conf, session, table, dataColumnPredicate, hdfsEnvironment)
            eligible =  session flag && parquet_use_column_names
                     && tableType == COPY_ON_WRITE && baseFileFormat == PARQUET
                     && no bootstrap base path && not bucketed
                     && tableConfig.isMetadataPartitionAvailable(COLUMN_STATS)
            pruner = eligible ? new HudiColumnStatsPruner(predicate,
                                   doAs(user, LatestHudiColumnStatsReader)) : empty
       -> list(partition):
            files = HiveFileIterator(...)              // unchanged listing
            if pruner present && partition format == table format && partition not bucketed:
                return new HudiPruningIterator(files, pruner, tableLocation, disabledFlag, runtimeStats)
            return files
HudiPruningIterator.computeNext
  take <= 128 files from the listing
  key each file as (path relative to the table, file name); non-.parquet or out-of-table files pass through
  retained = pruner.retainFiles(keys)                  // one MDT getColumnStats call
  emit retained files in original order, using the original HiveFileInfo objects
```

`isMetadataPartitionAvailable(COLUMN_STATS)` reads `hoodie.properties`, so a table without
the index pays no MDT cost. An index still being built is not marked available.

### Failure semantics

| Condition | Behaviour |
|---|---|
| No MDT, no `column_stats`, index building, no supported predicate column | No pruner and no lookups (zero overhead beyond the table-config check) |
| Recoverable lookup failure (`IOException`, `RuntimeException`, `LinkageError` such as a missing codec) | Keep the whole batch, disable pruning for the rest of the scan, log a warning once, and increment `hudiDataSkippingFallbacks` |
| Authorization failure (`AccessControlException`, `AccessDeniedException`, `SecurityException`, `PERMISSION_DENIED`) | Fail with `PERMISSION_DENIED`. Silently ignoring it would hide misconfiguration and could mask policy |
| Interruption / cancellation | Propagate |
| File listing failure | Unchanged behaviour |

### Resource management

Each batch opens and closes its own `HoodieBackedTableMetadata`. A query that stops early
(`LIMIT`, cancellation) or abandons the iterator therefore cannot leak MDT readers or file
handles. Lookups run synchronously on the split-loader thread as the scan user and are bounded
by the storage client's timeouts. The cost of reopening a reader per batch is measured by
`hudiDataSkippingLookupTime`. Reusing a reader per scan is an optimization that can come
later, once there is data to justify it.

### Snapshot consistency (known limitation of phase 1)

`LatestHudiColumnStatsReader` reads the **latest** MDT state. It is not bound to the instant
used for file listing. Because lookups use exact file names, a concurrent commit cannot
attribute one file's statistics to a different file. A new file written after the listing is
never looked up. What is not guaranteed in phase 1 is the case where a file listed at instant
T has its statistics entry rewritten after T. In Hudi COW, a file name identifies immutable
content (a file group ID plus a commit time), so that case should not arise. It is still
documented as a limitation and not claimed as a guarantee. Phase 2 adds a snapshot-bound reader
behind `HudiColumnStatsReader` that reads the MDT as of the listing instant, with no worker or
split changes.

### User-facing metrics

All metrics are recorded in the query's `RuntimeStats` on the coordinator, so they appear in
`EXPLAIN ANALYZE`, the UI and `QueryCompletedEvent`:

| Metric | Unit | Meaning |
|---|---|---|
| `hudiDataSkippingCandidateFiles` | count | Files that entered the pruning path |
| `hudiDataSkippingCandidateBytes` | bytes | Total length of those files |
| `hudiDataSkippingSkippedFiles` | count | Files pruned by statistics |
| `hudiDataSkippingSkippedBytes` | bytes | Total length of pruned files |
| `hudiDataSkippingLookupTime` | nanos | Time spent in MDT lookups; `count` = number of batches |
| `hudiDataSkippingFallbacks` | count | Scans where a recoverable failure disabled pruning |

## Metrics

* Ratios of skipped to candidate files and bytes for selective queries on indexed columns.
* Reduction in the number of splits, worker CPU and wall time, and object-store GET requests
  for the same queries with the feature off and on.
* Cost: `hudiDataSkippingLookupTime` compared with the coordinator split-enumeration time of the
  scan, and the time to first split.

## Other Approaches Considered

1. **Implement only in the `presto-hudi` connector.** Many deployments read Hudi tables through
   the Hive connector, using `HoodieParquetInputFormat` registered in the metastore. Putting the
   logic in `presto-hudi-common` gives both connectors the same pruning code. The `presto-hudi`
   adapter is a small follow-up.
2. **Use Hudi's engine-side file index (`HoodieFileIndex` / `ColumnStatsIndexSupport`).** These
   classes are Spark-specific and depend on Spark expressions and DataFrames. They are not in
   `hudi-presto-bundle`. Translating `TupleDomain` directly is simpler and keeps Presto's type
   semantics in Presto's hands.
3. **Prune on workers** (apache/hudi#16969). This removes MDT access from the coordinator, but
   a split and task are still spent on every file, and it would need worker changes in both
   Java and Velox. Pruning on the coordinator before splits exist removes more work and changes
   less code.
4. **Rely on Parquet footer and row-group statistics only.** This already happens, but only
   after the split is scheduled and the footer is fetched. MDT statistics avoid both.
5. **Wait for a `hudi-presto` module in apache/hudi**, as was done for Trino. As of Aug 2026,
   upstream Hudi triage considers the Presto items out of scope for that repository. Doing the
   work here unblocks users now. If the connector later moves, `presto-hudi-common` moves with it.

## Adoption Plan

* **Impact on existing users:** none by default. The feature is off unless
  `hive.hudi-mdt-data-skipping-enabled` or `hudi_mdt_data_skipping_enabled` is set. Constructor
  signatures of some `presto-hive` internals (`HudiDirectoryLister`, `StoragePartitionLoader`,
  `DelegatingPartitionLoader`, `BackgroundHiveSplitLoader`) gain a `TupleDomain<String>`
  parameter. These are not SPI classes.
* **New configuration:** one config property and one session property, documented in
  `connector/hive.rst`.
* **Rollout:** ship off by default, gather metrics from early adopters, then consider enabling
  it by default once the snapshot-bound reader (phase 2) lands and more types are supported.
* **Documentation:** a new *Hudi Copy-on-Write Data Skipping* section in the Hive connector
  docs, with a warning describing the snapshot limitation.
* **Out of scope / future work:**
  1. Snapshot-bound statistics reader (phase 2).
  2. More types: `DATE`, `TIMESTAMP`, `DECIMAL`, `VARCHAR`.
  3. `presto-hudi` connector adapter (`HudiSplitManager`).
  4. Partition-stats pruning (apache/hudi#16957), record-level and secondary index lookups for
     equality predicates (apache/hudi#16959).
  5. MOR support.
  6. Reuse of the MDT reader per scan and a dedicated lookup deadline, if the metrics show the
     per-batch cost is significant (compare apache/hudi#17042).

## Test Plan

Unit tests:

* `TestHudiColumnStatsConverter`: null counts, missing bounds, overflow, inconsistent
  statistics, unsupported types.
* `TestHudiStatisticsEvaluator`: intersection semantics, missing columns, type mismatches.
* `TestHudiColumnStatsPruner`: batch and column limits, `isUseful`, `TupleDomain.none`. It also
  includes a deterministic generated-data soundness test: 500 files × 10 predicate forms, where
  every file that contains a matching row must be retained.
* `TestHudiPruningIterator`: batching, order preservation, pass-through of ineligible files,
  fallback, permission and cancellation propagation, metrics.
* `TestHudiDataColumnPredicate`: only top-level regular columns reach the pruner.

Integration-style tests:

* `TestHudiDataSkipping` builds a COW table with real Parquet data files and a real MDT
  `column_stats` partition (HFiles). It then compares the rows read with pruning on and off,
  across both listing modes. It covers:
  * a matching file with no statistics
  * tables with no MDT
  * an index that is still being built
  * an index advertised in table properties but unreadable
  * predicates on unindexed columns
* Existing `TestHudiDirectoryLister`, `TestStoragePartitionLoader`,
  `TestBackgroundHiveSplitLoader` and `TestHiveClientConfig` tests are updated.

No product-test changes are required for phase 1. A product test against a Spark-written Hudi
table with `hoodie.metadata.index.column.stats.enable=true` would be a useful addition, and
reviewers can suggest one if they want it.
