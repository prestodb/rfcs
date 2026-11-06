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

Star schema joins read massive amounts of data that gets filtered out by the join. A large fact table joins with a filtered dimension table: we read all fact rows, build a hash table from the small dimension, then discard most fact rows during probe. This wasted I/O is the problem.

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

![Dynamic Partition Pruning Flow](RFC-0022/RFC-0022-dynamic-filtering-diagram.png)

### Pruning Levels

Filters apply at four levels, each with different information available:

| Level | Location | What's Tested | Example Statistics |
|-------|----------|---------------|-------------------|
| Partition | Coordinator (split generation) | Partition column values | `dt=2024-01-15` |
| File | Coordinator (split generation) | File-level min/max | Iceberg manifest, Parquet footer |
| Row group | Worker (scan operator) | Row group statistics | Parquet row group min/max |
| Row | Worker (scan operator) | Individual values | Hash lookup or range comparison |

The same filter is passed to all levels. Each level evaluates columns it has statistics for; columns without statistics pass through unchecked.

### Filter Representation

Filters use Presto's existing `TupleDomain`, which supports discrete values and ranges:

- **Discrete values** (cardinality ≤ configurable limit, default 10K): All distinct values stored, ~8 bytes per value. Enables exact membership testing with 0% false positives.
- **Range** (cardinality > limit): Only min/max stored, 16 bytes total. Enables range overlap testing but may be ineffective for sparse distributions (e.g., {1, 1000000} produces range [1, 1000000] which prunes nothing).

Future work may add bloom filters for high-cardinality cases (see Future Work).

### Query Planning

`AddDynamicFilterRule` is a new `PlanOptimizer` that runs after join reordering but before fragment creation. It generates a dynamic filter when:

1. The join column is a partition column on the probe side (always beneficial)
2. The join column is in `getColumnsWithRangeStatistics()` and the build/probe cardinality ratio is below 0.5
3. The estimated build cardinality is below the discrete values limit

When none of these conditions hold, the optimizer skips filter generation to avoid overhead. This follows the approach used by Doris and Spark.

The rule populates the `dynamicFilters` field on `JoinNode` and marks the probe-side `TableScanNode` with the filter ID.

**Transitive filter propagation:**

When joins are chained, the optimizer propagates dynamic filters transitively through equality relationships to reach table scans deeper in the plan tree. Consider `HashJoin(HashJoin(A, B), C)` with conditions `A.x = B.x` and `B.x = C.x`:

- C is the build side of the outer join, producing a filter on column `x`.
- The probe side is `HashJoin(A, B)`, where `A.x = B.x` establishes an equivalence.
- The optimizer follows this equivalence chain: C's filter on `x` applies to B's column `x`, and since `B.x = A.x`, it also applies to A's table scan on column `x`.
- Result: A's `TableScanNode` is marked with a dynamic filter from the C join, even though they are not directly joined.

The rule maintains equivalence sets of columns across joins as it traverses the plan tree. For each hash join with a dynamic filter, it walks the probe subtree to find all table scans reachable through equality chains. This means a single build-side filter can be applied to multiple table scans simultaneously.

Note that for some join orderings, transitivity is implicit. With `HashJoin(A, HashJoin(B, C))`, B is filtered by C before joining with A, so the filter from B to A already reflects C's filtering without explicit transitive propagation. Explicit transitivity is needed when the intermediate table (B) is on the build side rather than the probe side.

### Build-Side Filter Extraction

Velox already supports dynamic filter pushdown within a driver. When the hash table is ready, `HashProbe` calls `VectorHasher::getFilter()` to create `BigintValues` filters (discrete values), and the driver pushes them down to `TableScan` operators via `Driver::pushdownFilters()`. This works for broadcast joins where build and probe are colocated.

For partitioned joins (build distributed across workers), we need to extract the filter and send it back to the coordinator. The mechanism works as follows:

The `JoinNode` protocol message includes a `dynamicFilters` map (filter ID → join column). During plan conversion, `PrestoToVeloxQueryPlan` registers these filter IDs with the Velox `Task`. When `HashBuild::noMoreInput()` completes, it checks for registered filter IDs and extracts filters if present.

Filter extraction calls `table_->hashers()` to get `VectorHasher` objects for join key columns. For low-cardinality keys, `VectorHasher::getFilter()` returns discrete values; for high-cardinality keys (where VectorHasher overflows), we build a range from tracked min/max. The filter is stored in `PrestoTask::dynamicFilters_` for coordinator collection.

### Filter Collection and Merging

`DynamicFilterFetcher` long-polls `GET /v1/task/{taskId}/dynamicFilters?since=N` on each build-side worker. After processing, it sends `DELETE?through=N` to free worker memory. The endpoint supports `X-Presto-Max-Wait` for long polling.

For partitioned joins, `CoordinatorDynamicFilter` merges filters from multiple workers using `TupleDomain.columnWiseUnion()`. Since each worker sees a disjoint subset of build data (hash-partitioned), merging uses UNION semantics: discrete values become set unions, ranges expand to cover all workers' min/max bounds. For broadcast joins, a single worker's filter is complete.

`SectionExecutionFactory` pre-registers each filter via `DynamicFilterService`. The filter uses all-or-nothing semantics: `getCurrentConstraint()` returns `TupleDomain.all()` until all expected partitions arrive. Timeout does not count as completion — a timed-out filter returns `all()`, not partial data.

### Scheduler Integration

`SplitSourceFactory` looks up filters from `DynamicFilterService` and passes them to `ConnectorSplitManager.getSplits()`. The `DynamicFilter` contains the columns covered, the constraint future, and a recommended wait duration.

**Deadlock prevention**: When the probe-side split source blocks waiting for a filter, no splits are produced. But the build pipeline producing the filter cannot start until tasks are created. `SourcePartitionedScheduler` detects this (split source blocked + no tasks scheduled) and forces task creation to break the cycle.

### Filter Distribution

**Phase 1 (coordinator-side)**: Filters apply during split generation. The connector receives the `DynamicFilter` and decides whether to wait upfront, start immediately, or use a hybrid approach. No distribution to workers needed.

**Phase 2 (worker-side)**: The coordinator pushes merged filters to probe-side workers:

```
POST /v1/task/{taskId}/inputs/filter/{filterId}
Body: { "complete": true, "filterType": "tupleDomain", "tupleDomain": { ... } }
```

Workers verify `complete: true`, then call `veloxTask->addDynamicFilter()` to inject the filter. Velox applies it to row groups (via `ParquetReader` statistics) and rows (via type-specific filters like `BigintValuesUsingHashTable` for discrete values).

Probe execution may start before the filter arrives—early splits proceed without row-level filtering but still benefit from partition/file pruning.

### Serialization

C++ workers serialize filters to JSON with a `filterType` envelope for future extensibility (e.g., bloom filters). Size scales linearly: ~8KB for 1K values, ~80KB for 10K values, <1KB for range-only.

### Module Organization

| Module | New Components |
|--------|---------------|
| presto-main-base | `AddDynamicFilterRule`, `CoordinatorDynamicFilter`, `DynamicFilterService` |
| presto-main | `DynamicFilterFetcher` |
| presto-native-execution | `TupleDomainBuilder`, `TupleDomainParser`, Velox filter conversion |
| Connectors | Partition/file pruning in split sources |

`LocalDynamicFilter` remains for colocated dynamic filtering but is not reused—it requires fixed partition count at construction and doesn't implement the `DynamicFilter` SPI.

### Error Handling

All failure modes degrade gracefully to no filtering (`TupleDomain.all()`):

- **Build stage failure**: Future completes exceptionally; split source proceeds without filter
- **Timeout**: `distributed_dynamic_filter_max_wait_time` expires before all workers report; filter returns `all()`
- **Partial collection**: All-or-nothing — filter returns `all()` until all expected partitions arrive
- **Memory pressure**: Falls back to range-only during construction

Worker restarts and network partitions are handled by existing task failure mechanisms.

### API/SPI Changes

**SPI extension for table layout:**

```java
// ConnectorTableLayout
default Set<ColumnHandle> getColumnsWithRangeStatistics() {
    return getStreamPartitioningColumns().orElse(Set.of());
}
```

Connectors advertise which columns have min/max statistics enabling predicate evaluation before reading rows. Iceberg/Delta/Hudi return all columns (manifest stats); Hive returns partition columns; JDBC might return indexed columns.

**SPI extension for split manager:**

`ConnectorSplitManager.getSplits()` receives a `DynamicFilter` containing covered columns, a constraint future, and a recommended wait duration. For multiple joins on the same probe table, filters are combined into a single composite that resolves progressively as individual filters complete.

**Protocol extensions:**

```
GET  /v1/task/{taskId}/dynamicFilters?since=N   # Long-poll for new filters
DELETE /v1/task/{taskId}/dynamicFilters?through=N  # Free worker memory
POST /v1/task/{taskId}/inputs/filter/{filterId}    # Push filter to probe worker
```

**No breaking changes**: New SPI methods have default implementations.

### Configuration and Metrics

**Configuration**:

```properties
distributed-dynamic-filter.enabled=true
distributed-dynamic-filter.max-wait-time=0s              # Default: no waiting (opt-in)
distributed-dynamic-filter.max-size=1MB
distributed-dynamic-filter.distinct-values-limit=10000   # Above this, fall back to range
```

The `distributed-` prefix distinguishes from the existing built-in local dynamic filtering (`enable_dynamic_filtering`). The two modes are mutually exclusive — enabling both throws `INVALID_SESSION_PROPERTY`.

**Metrics** (exposed in query JSON):

- **Timing**: filter completion time, actual wait, recommended wait
- **Pruning**: total files, files pruned, files scheduled before filter ready
- **Missed opportunities**: files scheduled early that would have been pruned (enables wait time tuning)

## Other Approaches Considered

### Alternative: Bloom Filters for High Cardinality

Rejected because bloom filters only support exact membership tests, not range overlap tests against column statistics. The primary benefit of dynamic filtering is I/O reduction from partition/file pruning, which requires discrete values or ranges. Bloom filters would only help at row-level filtering—deferred to future work.

### Comparison: Trino's Dynamic Filtering

Trino uses a similar push-based architecture. Key differences:

| Aspect | This RFC | Trino |
|--------|----------|-------|
| Optimizer | Cost model (skips low-value filters) | All equi-joins |
| Transport | Idempotent GET with separate DELETE | GET has delete side effect |
| SPI | `getColumnsWithRangeStatistics()` + wait hint | Connector decides wait |
| Runtime | C++/Velox | Java |

The core collection and distribution mechanisms are architecturally similar.

## Rollout

**Phase 1**: Coordinator-side pruning (partition/file). Filter extraction, collection, merging, and connector integration. Delivers the primary I/O reduction benefit.

**Phase 2**: Worker-side filtering (row-group/row). Push merged filters to probe workers for Velox integration.

**No breaking changes**: Feature is opt-in (0s default wait). New SPI methods have defaults.

**Future work**: Java worker support, bloom filter integration for high-cardinality joins, cost-based timeout calculation.

## Overhead

Filter collection adds one long-poll connection per build-side task (negligible with Netty's non-blocking I/O) plus JSON serialization (~80KB for 10K values). The cost model ensures filters are only generated when the expected I/O reduction exceeds this overhead.

Dynamic filters compose with existing predicate pushdown—static predicates are pushed at planning time, dynamic filters add runtime constraints. Both feed into the same `TupleDomain` evaluation path.

## Test Plan

- **Unit tests**: Filter construction, JSON serialization round-trip, merge logic (Java and C++)
- **Integration tests**: End-to-end query with filter distribution, multi-worker partitioned joins, graceful degradation on failure
- **Connector tests**: Iceberg manifest pruning
- **Performance**: TPC-DS queries with selective dimension filters (Q5, Q17, Q25, Q35 showed >50% improvement in Trino benchmarks)
