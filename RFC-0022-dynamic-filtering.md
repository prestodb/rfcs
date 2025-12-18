# RFC: Dynamic Partition Pruning with Runtime Filters

Proposers

* Tim Meehan

## Related Issues

* [#7428: Add support for dynamic filtering](https://github.com/prestodb/presto/issues/7428) - Original feature request
* [#8588: Dynamic Partition Pruning](https://github.com/prestodb/presto/pull/8588) - Earlier coordinator-only implementation
* [#15758: Dynamic Filter for Hash Join](https://github.com/prestodb/presto/issues/15758) - Distributed dynamic filtering discussion

## Summary

This RFC proposes implementing distributed dynamic partition pruning for Presto. The feature extracts filters from hash join build sides on workers, distributes them to probe-side workers, and applies them at multiple pruning levels (partition, file, row group, and row filtering).

The runtime filter uses Presto's existing `TupleDomain` structure, which supports both min/max ranges and discrete values (up to configurable limit). This enables lossless pruning for low-cardinality joins and range-based pruning for high-cardinality joins.

The SPI adds a `getColumnsWithRangeStatistics()` method to `ConnectorTableLayout`, allowing connectors to advertise which columns have statistics that enable efficient pruning. The scheduler uses this information along with existing cost-based statistics to decide whether to wait for dynamic filters before generating splits.

## Background

### The Problem: Unnecessary I/O in Star Schema Joins

Star schema joins often read massive amounts of data that will be filtered out by the join condition. A typical pattern involves joining a large fact table with a filtered dimension table. Without dynamic filtering, the system reads all rows from the fact table, builds a hash table from the small filtered dimension, then discards most fact rows during the probe phase. The wasted I/O from reading rows that will be immediately filtered out is the core problem this RFC addresses.

### Existing Limitations

Presto's current dynamic filtering only works for broadcast joins where build and probe are colocated on the same worker. Filtering happens at the row group and row levels within Velox, but because it occurs on the worker after splits are already scheduled, it cannot skip partitions or files during split generation. For partitioned joins, the build side is distributed across workers, so no single worker sees all build-side values—local filtering would incorrectly prune valid join keys.

This RFC enables partition and file pruning for both join types by collecting filters to the coordinator before split scheduling. For broadcast joins, the filter from any single worker is complete (all workers have the same build data). For partitioned joins, filters from all workers must be merged.

### Goals

1. **Distributed filtering**: Extract filters on workers, distribute to probe-side workers
2. **Multi-level pruning**: Apply filters at partition, file, row group, and row levels
3. **C++ support**: Integration with Velox execution engine
4. **Low overhead**: Opt-in behavior with conservative defaults

### Non-goals

1. **Java worker support**: This RFC targets C++/Velox deployments only
2. **Cross-query filter caching**: Filters are query-scoped, not cached across queries
3. **Non-equijoin support**: Only hash equijoins (joins with equality predicates like `a.id = b.id`) are supported. Joins using range or inequality predicates are not supported.

## Proposed Implementation

### 1. High-Level Flow

The implementation follows six steps: planning, filter construction, collection, merging, split scheduling, and application.

During **query planning**, `AddDynamicFilterRule` (a new optimizer in presto-main-base) identifies hash joins where the build side is selective. It populates the `dynamicFilters` field on the `JoinNode` (a map from filter ID to join column) and marks the probe-side `TableScanNode` with the corresponding filter ID. The optimizer does not need to know whether the column is a partition column or has file-level statistics—that decision is deferred to the connector.

During **build-side execution**, the `HashBuild` operator collects distinct join key values while constructing the hash table. If cardinality stays below a configurable limit (default 10K), it creates a `TupleDomain` with discrete values for perfect filtering. Otherwise, it falls back to a range (min/max only). Upon completion, the filter is stored in the task's outputs collection and `outputsVersion` in `TaskStatus` is incremented.

The **coordinator collects filters** via `DynamicFiltersFetcher`, a new component that detects version changes in `TaskStatus` during the existing polling cycle. It then long-polls the worker's `/v1/task/{taskId}/outputs/filter/{version}` endpoint to fetch new filters incrementally. The version-based protocol handles multiple `HashBuild` operators in the same task completing at different times.

For **filter merging**, the coordinator uses the existing `LocalDynamicFilter` to accumulate partial filters from multiple build-side workers (for partitioned joins). It merges using `TupleDomain.columnWiseUnion()` with UNION semantics: since each worker sees a disjoint subset of build data (hash-partitioned), valid join keys are those seen by ANY worker, so discrete values become set unions and ranges expand to cover all workers' min/max bounds. For broadcast joins with a single build worker, the filter is immediately complete.

During **split scheduling**, the scheduler analyzes the table layout's `getColumnsWithRangeStatistics()` and `getStreamPartitioningColumns()` along with the build/probe cardinality ratio to compute a recommended wait duration. The connector receives a `DynamicFilter` object containing the columns covered, the constraint future, and this wait hint—allowing it to wait upfront, start immediately, or use a hybrid approach.

**Filter application** happens at four levels. The connector's split source (e.g., `HiveSplitSource`, `IcebergSplitSource`) uses the filter to prune partitions and files during split generation by testing partition/file statistics against the filter's range or discrete values. For the remaining levels—row-group and row-level filtering—the coordinator pushes the completed merged filter to probe-side workers via HTTP (Phase 2). Velox's `ParquetReader` then prunes row groups and applies row-level filtering using type-specific filters: `BigintValuesUsingHashTable`/`BigintRange` for integers, `BytesValues`/`BytesRange` for strings, `DoubleRange`/`FloatRange` for floating point, and `TimestampRange` for timestamps.

![Dynamic Partition Pruning Flow](RFC-0022/RFC-0022-dynamic-filtering-diagram.png)

### 2. Component Details

#### 2.1 Query Planning

`AddDynamicFilterRule` is a new `PlanOptimizer` that runs after join reordering but before fragment creation. It identifies hash joins where the build side is selective and dynamic filtering would be beneficial. The rule uses a cost model to decide whether to generate a filter, comparing the estimated benefit (probe-side I/O reduction) against the cost (filter collection and distribution overhead).

A filter is generated when the join column is a partition column on the probe side (partition pruning is always beneficial), when the join column is in `getColumnsWithRangeStatistics()` and the estimated build/probe cardinality ratio is below a threshold, or when the estimated build cardinality is low enough that discrete values will be collected. When none of these conditions are met, the optimizer skips generating the filter to avoid runtime overhead for filters that won't provide meaningful pruning. This follows the approach used by Doris and Spark.

When a beneficial join is identified, the rule populates the `dynamicFilters` field on the `JoinNode` and marks the probe-side `TableScanNode` with the filter ID.

#### 2.2 Runtime Filter Structure

The filter uses Presto's existing `TupleDomain<ColumnHandle>`, which supports both discrete values and ranges. For cardinality ≤10K, the filter contains all distinct values (8N bytes, 0% false positives). For cardinality >10K, it falls back to a range containing only min/max (16 bytes). Memory usage is bounded by a configurable limit on discrete values.

**Limitation**: Range-only filters may be ineffective for sparse value distributions. For example, if the build side contains values {1, 1000000}, the range [1, 1000000] prunes nothing. This is an accepted trade-off: discrete values provide exact filtering for low-cardinality joins (the common case for dimension tables), while range provides best-effort filtering for high-cardinality joins. Future work may add bloom filters for the high-cardinality case (see Future Work).

#### 2.3 Build-Side Construction

Velox already supports dynamic filter pushdown within a driver. When the hash table is ready, `HashProbe` calls `VectorHasher::getFilter()` to create `BigintValues` filters (discrete values), and the driver pushes them down to `TableScan` operators via `Driver::pushdownFilters()`. This works for broadcast joins where build and probe are colocated.

For partitioned joins (build distributed across workers), we need to extract the filter and send it back to the coordinator. The mechanism works as follows:

**How C++ knows to produce filters for coordinator collection:**

The Java coordinator already includes `dynamicFilters` in the `JoinNode` protocol message (a map from filter ID to the join column). When `PrestoToVeloxQueryPlan` converts the Presto plan to a Velox plan, we extract this map and store it in a new field on the Velox `Task`. Specifically:

- During plan conversion, if `protocol::JoinNode::dynamicFilters` is non-empty, we register each filter ID with the Velox `Task`
- When `HashBuild::noMoreInput()` completes, it checks if this join has registered filter IDs
- If so, it extracts filters and reports them to `PrestoTask` for coordinator collection
- If not (e.g., optimizer didn't select this join for dynamic filtering), no extraction happens

This ensures only joins explicitly marked by the optimizer produce filters for coordinator collection.

**Filter extraction and reporting:**

1. **Filter extraction** (C++): `HashBuild` completes the hash table and calls `joinBridge_->setHashTable()` to hand it to `HashProbe`. If the join has registered dynamic filter IDs, we call `table_->hashers()` to get the `VectorHasher` objects for join key columns. For low-cardinality keys, `VectorHasher::getFilter()` returns a `BigintValues` filter with discrete values. For high-cardinality keys (where VectorHasher overflows), we build a range filter from the tracked min/max. These are converted to a `TupleDomain` JSON representation.

2. **Storage in PrestoTask** (C++): The filter is stored in a new `outputs_` map in `PrestoTask`, keyed by filter ID. We atomically increment `outputsVersion` in the task's status. This requires adding a callback from Velox's `Task` to Presto's `PrestoTask` when filters are ready.

3. **Version detection** (Java): The coordinator's `TaskInfoFetcher` already polls `TaskStatus` periodically. We add an `outputsVersion` field to `TaskStatus`. When the coordinator detects a version change, it knows new outputs (filters) are available.

4. **Filter fetch** (Java → C++): The coordinator calls a new HTTP endpoint `GET /v1/task/{taskId}/outputs/filter/{version}`. This endpoint is registered in `TaskResource.cpp` and returns all filters created since the specified version as JSON.

5. **Long-polling**: The endpoint supports `X-Presto-Max-Wait` header (like existing endpoints) so the coordinator can wait up to N seconds for filters to become available, reducing polling overhead.

The coordinator then merges filters from all build partitions using `LocalDynamicFilter` and distributes the merged result.

```
ALGORITHM BuildFilter(values):
    min ← MAX_INT
    max ← MIN_INT
    distinctValues ← empty set

    FOR EACH value IN values:
        min ← MIN(min, value)
        max ← MAX(max, value)
        IF size(distinctValues) < LIMIT:
            distinctValues.insert(value)

    IF size(distinctValues) < LIMIT:
        RETURN TupleDomain.discreteValues(distinctValues)
    ELSE:
        RETURN TupleDomain.range(min, max)
```

#### 2.4 Filter Collection

The coordinator collects filters through an extension to `HttpRemoteTask`. The existing `ContinuousTaskStatusFetcher` polls `TaskStatus` from workers. When it detects an `outputsVersion` change, `HttpRemoteTask` fetches the new filters via the `/v1/task/{taskId}/outputs/filter/{version}` endpoint and passes them to the coordinator's `LocalDynamicFilter` (created by `SqlQueryScheduler` for each distributed dynamic filter in the query).

The version-based protocol handles multiple `HashBuild` operators in the same task completing at different times, allowing incremental filter collection without re-fetching.

#### 2.5 Filter Merging

For partitioned joins with multiple build workers, the coordinator uses the existing `LocalDynamicFilter` to merge partial filters. `SqlQueryScheduler` creates the `LocalDynamicFilter` with the partition count set to the number of build-side tasks (known from the stage execution plan). As `HttpRemoteTask` fetches filters, it passes them to `LocalDynamicFilter` via `getTupleDomainConsumer()`, which accumulates them using `TupleDomain.columnWiseUnion()`. When all build-side tasks have reported, it completes the `resultFuture`.

```
ALGORITHM MergeFilters(filter1, filter2):
    # UNION semantics: each worker sees disjoint subset of build data
    IF both have discrete values:
        RETURN union of value sets
    ELSE IF both have ranges:
        RETURN range(MIN(min1, min2), MAX(max1, max2))
    ELSE IF one discrete, one range:
        RETURN range covering both (discrete values + range bounds)
```

#### 2.6 Scheduler Integration

`SqlQueryExecution` creates a `LocalDynamicFilter` for each dynamic filter in the query. The scheduler analyzes the table layout's `getColumnsWithRangeStatistics()` and `getStreamPartitioningColumns()` along with the build/probe cardinality ratio from `StatsAndCosts` to compute a recommended wait duration. `SplitSourceFactory` passes a `DynamicFilter` object to `SplitManager.getSplits()` containing the columns covered, the constraint future, and this wait hint.

#### 2.7 Filter Distribution

**Phase 1: Coordinator-side (partition/file pruning)**

Filters are applied during split generation on the coordinator. The connector receives the `DynamicFilter` with its future and wait hint, then decides how to proceed: wait upfront for maximum pruning, start immediately, or use a hybrid approach. No filter distribution to workers is needed.

**Phase 2: Worker-side (row-group/row-level filtering)**

The coordinator pushes the merged filter to probe-side workers via a new endpoint:

```
POST /v1/task/{taskId}/inputs/filter/{filterId}

Body:
{
  "complete": true,
  "filterType": "tupleDomain",
  "tupleDomain": { "columnDomains": { ... } }
}

Response 200 OK
Response 404 Not Found (task does not exist)
```

The `complete` field indicates this is a fully merged filter safe for pruning. Workers reject filters where `complete: false`.

**Coordinator-side flow:**

1. `LocalDynamicFilter` completes with merged filter
2. Coordinator identifies probe-side tasks whose plan contains a `TableScanNode` referencing this filter ID
3. Coordinator calls the endpoint on each probe-side worker

**Worker-side flow:**

1. `PrestoTask` receives filter, verifies `complete: true`, stores in task-level map
2. Calls `veloxTask->addDynamicFilter(filterId, filter)` to inject into Velox
3. Velox's `TableScan` (which registered interest in this filter ID during plan conversion) applies filter at two levels:
    - **Row-group pruning**: `ParquetReader` tests row group statistics against the filter's range
    - **Row-level filtering**: Values tested against type-specific Velox filters (hash lookup for discrete values, range comparison for min/max)

**Timing**: Probe execution may start before the filter arrives. Early splits proceed without row-level filtering but still benefit from Phase 1 partition/file pruning. When the filter arrives mid-execution, Velox applies it to subsequent row groups and rows.

#### 2.8 Filter Application

Filters apply at four cascading levels. At partition and file levels, the connector's split source (e.g., `HiveSplitSource`, `IcebergSplitSource`) tests partition/file statistics against the filter when generating splits. It builds a `TupleDomain` from the partition's min/max and intersects with the filter; if the result is NONE, the partition is skipped. At row group level, Velox's `ParquetReader` applies the same logic to row group statistics. At row level, Velox converts the `TupleDomain` to type-specific filters (hash table lookup for discrete values, range comparison for min/max).

#### 2.9 Serialization

C++ workers serialize filters to JSON for the coordinator to fetch. The format wraps the filter in an envelope that identifies the filter type, allowing future extension to bloom filters:

```json
{
  "filterType": "tupleDomain",
  "tupleDomain": {
    "columnDomains": { ... }
  }
}
```

In the future, this could be extended:

```json
{
  "filterType": "bloomFilter",
  "bloomFilter": { ... }
}
```

The coordinator deserializes the envelope, then dispatches to the appropriate codec (`JsonCodec<TupleDomain<ColumnHandle>>` for Phase 1). For the return path (coordinator to probe-side workers), the same envelope format is used. Size scales linearly with cardinality: ~8KB for 1K values, ~80KB for 10K values, and <1KB for range-only filters.

#### 2.10 Module Organization

New components are added to presto-main-base (`AddDynamicFilterRule`) and presto-main (`DynamicFiltersFetcher`), with existing components reused or extended (`LocalDynamicFilter` in presto-main-base, `SourcePartitionedScheduler`). The C++ worker in presto-native-execution adds `TupleDomainBuilder` for filter construction and `TupleDomainParser` for JSON deserialization, plus conversion logic to Velox's type-specific filter classes. Connectors (Hive, Iceberg, Delta) implement partition/file pruning in their split sources, and Velox handles row group and row-level filtering in `ParquetReader` and the scan operators.

#### 2.11 Error Handling and Failure Modes

**Build stage failure**: If the build stage fails before producing filters, the `LocalDynamicFilter` future is completed exceptionally. The split source should handle this by proceeding without the filter (graceful degradation). The query continues but without partition pruning benefit.

**Filter collection timeout**: If `dynamic_filter_max_wait_time` expires before all build workers report, `LocalDynamicFilter` completes with `TupleDomain.all()` (no filtering). This prevents indefinite blocking. Connectors that are waiting on the future proceed with all partitions.

**Partial filter collection**: If some but not all build workers report before timeout, the coordinator **cannot** use any filter—neither discrete values nor ranges. An unreported worker might have values outside the partial range, and using a partial filter would incorrectly prune valid join keys. The coordinator completes with `TupleDomain.all()` (no filtering). This is the safe choice: we lose the pruning benefit but maintain correctness.

**Worker restart mid-query**: If a build worker restarts, the coordinator detects task failure through the existing mechanism and fails the query. Dynamic filters do not introduce new failure modes here.

**Network partition**: If the coordinator cannot reach a worker to fetch filters, the existing task status polling will eventually detect the unreachable worker and fail the query or retry.

**Memory pressure**: If discrete value collection exceeds limits, the filter falls back to range-only during construction (not a failure). If the coordinator's memory is pressured, it can complete with partial filters early.

### 3. API/SPI Changes

**Existing infrastructure reused:**

- `LocalDynamicFilter` (presto-main-base): Collects and merges filters using `TupleDomain.columnWiseUnion()`. Provides `getResultFuture()` for async notification when filter is complete.

- `JoinNode.getDynamicFilters()` (presto-spi): Already maps filter IDs to build-side variables.

**SPI extension for table layout:**

A new `getColumnsWithRangeStatistics()` method on `ConnectorTableLayout` allows connectors to advertise which columns have range statistics (min/max) tracked at a granularity that enables efficient predicate evaluation before reading rows. This includes partition columns, columns with manifest or block statistics, and indexed columns in databases. The default implementation returns partition columns only. Iceberg, Delta, and Hudi would return all columns since they track min/max in manifests; Hive would return partition columns; JDBC connectors might return indexed columns.

**Wait hint computation:**

The scheduler computes a recommended wait duration using information from the layout and existing cost-based statistics. Research on semi-join reduction[^1] establishes that filters are beneficial when `selectivity × probe_IO_cost > wait_cost`. The scheduler checks if the filter covers columns in `getColumnsWithRangeStatistics()` or `getStreamPartitioningColumns()`, and uses the build/probe cardinality ratio from `StatsAndCosts`. This hint is passed to the connector, which makes the final decision on how to handle the future.

Both error cases degrade gracefully: waiting unnecessarily adds latency but the query completes; not waiting wastes some I/O but later splits still benefit from the filter when it arrives.

**SPI extension for split manager:**

A new `DynamicFilter` class is passed to `ConnectorSplitManager.getSplits()`. For queries with multiple joins producing multiple dynamic filters on the same probe table, the coordinator combines them into a single `DynamicFilter` containing the union of all covered columns, a constraint future, and a recommended wait duration. The future resolves progressively as individual filters complete, returning the intersection of all complete filters so far. When filter A completes, the future resolves with A's predicate; when filter B completes, it resolves again with the intersection of A and B. This allows the connector to apply filters adaptively as they become available rather than waiting for all filters to complete. The connector can wait upfront, start immediately and apply filters as they arrive, or use a hybrid approach.

**Protocol extensions:**

- Add `outputsVersion` field to `TaskStatus` (incremented when any collected output is ready)
- Add HTTP endpoint `GET /v1/task/{taskId}/outputs/filter/{version}` in C++ workers

**HTTP Endpoint Specification:**

```
GET /v1/task/{taskId}/outputs/filter/{version}

Headers:
  X-Presto-Max-Wait: <duration>  (optional, e.g., "1s")

Response 200 OK:
{
  "version": 3,
  "filters": {
    "df_1": {
      "filterType": "tupleDomain",
      "tupleDomain": { "columnDomains": { ... } }
    }
  }
}

Response 204 No Content:
  (returned if no new filters since version {n} and max-wait expires)

Response 404 Not Found:
  (task does not exist)
```

The endpoint returns all filters with version > `n`. The coordinator tracks the last fetched version per task and requests incrementally.

**No breaking changes**: The new SPI method has a default implementation.

### 4. User-Facing Configuration and Metrics

**Configuration**:

```properties
# System config
dynamic-filter.enabled=true
dynamic-filter.max-wait-time=0s
dynamic-filter.max-size=1MB
dynamic-filter.distinct-values-limit=10000
dynamic-filter.queue-threshold=0.7
```

**Session properties**:

```sql
set session dynamic_filter_max_wait_time='500ms';
set session dynamic_filter_max_size='2MB';
set session dynamic_filter_enabled=false;
```

**Metrics** (exposed in query JSON):

Metrics are organized into three categories: timing, pruning, and decision quality.

**Timing metrics** capture when key events occurred: filter completion time (when the merged filter became available), split generation start time, actual wait time used by the connector, and recommended wait time from the scheduler.

**Pruning metrics** measure effectiveness: total files considered, files pruned by the filter, files scheduled before the filter was ready, and—critically—files scheduled before the filter that would have been pruned if the filter had been available (the retroactive check).

**Decision quality metrics** are derived from timing and pruning:
- **Pruning rate** (files pruned / total files): If low, waiting wasn't worthwhile.
- **Missed opportunity rate** (files scheduled early that would have pruned / files scheduled early): If high, the wait time should be increased.
- **Wait efficiency** (actual wait minus filter completion time): Negative means the connector gave up too early; large positive means it waited longer than necessary.

These metrics enable tuning: consistently high missed opportunity rates suggest increasing wait time; consistently low pruning rates suggest decreasing wait time or adjusting selectivity thresholds. Aggregating these metrics by table and column over many queries enables data-driven parameter tuning.

## Other Approaches Considered

### Alternative: Bloom Filters for High Cardinality

**Approach**: Use bloom filters (DataSketches) for joins with >10K distinct values.

**Pros**:
- Bounded memory
- Handles high cardinality joins efficiently
- Probabilistic pruning better than no pruning

**Cons**:
- Only helps at row filtering level (not partition/file/row group pruning)
- Bloom filters too conservative for definitive pruning (false positive rate)
- Adds significant complexity: DataSketches dependency, custom serialization, merge logic
- Primary benefit is I/O reduction (partition/file pruning), which bloom doesn't help

**Why rejected**: Bloom filters cannot be used for partition/file/row-group pruning because they only support exact membership tests, not range overlap tests against column statistics. The primary benefit of dynamic filtering is I/O reduction from pruning at those levels, which requires discrete values or ranges.

### Comparison: Trino's Dynamic Filtering

Trino implements a similar push-based architecture: the coordinator collects filters from build-side workers and pushes them to probe-side workers. The main differences are:

1. **Optimizer**: Trino generates dynamic filters for all equi-join clauses on INNER/RIGHT joins when the feature is enabled, with no cost model or selectivity check. This RFC uses a cost model that skips generating filters when the join column is not in partition columns or `getColumnsWithRangeStatistics()`, when the build/probe cardinality ratio is above threshold, or when the estimated build cardinality is too high.

2. **Transport**: Both systems use dedicated endpoints for filter collection (`DynamicFiltersFetcher` in Trino, similar fetcher in this RFC). For distribution to probe workers, Trino bundles filters in `TaskUpdateRequest`; this RFC uses a dedicated endpoint.

3. **SPI design**: Both systems combine multiple dynamic filters targeting the same table scan into a single object with progressive resolution—Trino calls this "narrowing." This RFC additionally introduces `getColumnsWithRangeStatistics()` on the table layout, allowing connectors to advertise prunable columns, and the scheduler uses this along with CBO statistics to compute a recommended wait duration passed to the connector. Trino leaves the wait decision entirely to the connector without scheduler guidance.

4. **Runtime**: Trino targets Java workers; this RFC targets C++/Velox.

The core collection and distribution mechanisms are architecturally similar.

## Rollout

### Phase 1: Coordinator-Side Pruning

Collect filters from build-side workers and use them for partition/file pruning during split generation on the coordinator. This phase delivers the primary I/O reduction benefit.

**Scope**:
- Filter extraction on C++ workers, collection by coordinator
- `LocalDynamicFilter` merging
- New `DynamicFilter` class and SPI method for `ConnectorSplitManager.getSplits()`
- Hive and Iceberg connector support for partition/file pruning

**Not included**: Row-group and row-level filtering on workers (filters stay on coordinator).

### Phase 2: Worker-Side Filtering

Distribute merged filters back to probe-side workers for row-group and row-level filtering. This phase adds incremental benefit beyond partition/file pruning.

**Scope**:
- New endpoint `POST /v1/task/{taskId}/inputs/filter/{filterId}` for coordinator to push filters
- Worker-side storage and injection into Velox via `Task::addDynamicFilter()`
- Velox row-group pruning in `ParquetReader`
- Velox row-level filtering via type-specific filters

### Impact on Existing Users

**No breaking changes**:
- Feature is opt-in (0s default wait time)
- Existing queries run unchanged
- No SPI breaking changes (default methods provided)

### Future Work (Out of Scope)

The following enhancements are explicitly deferred:

1. **Java worker support**: Extend filtering to Java-based execution
2. **Bloom filter integration**: For high-cardinality joins (>10K), use `Constraint.predicate()` to wrap a bloom filter for row-level filtering while using range in `getSummary()` for partition/file pruning
3. **Cost-based timeout**: Automatically calculate wait time based on estimated build time

## Test Plan

- **Unit tests**: Filter construction, JSON serialization round-trip, merge logic (Java and C++)
- **Integration tests**: End-to-end query with filter distribution, multi-worker partitioned joins, graceful degradation on failure
- **Connector tests**: Iceberg manifest pruning
- **Performance**: TPC-DS queries with selective dimension filters—Q5, Q17, Q25, Q35 showed >50% improvement with dynamic partition pruning in Trino benchmarks; Q1, Q7 showed 30-50% improvement

## References

[^1]: Bernstein & Chiu, ["Using Semi-Joins to Solve Relational Queries,"](https://dl.acm.org/doi/10.1145/322234.322238) JACM 1981
