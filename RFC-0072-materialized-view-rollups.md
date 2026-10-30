# **RFC-0072 for Presto: Materialized View Rollups**

## Materialized Views over Materialized Views, Aggregate Merge Rules, and Partition Transforms

Proposers

* Abhash Jain (@abhash09)

## Related Issues

* [RFC-0016: Materialized Views](RFC-0016-materialized-views.md) describes the current materialized view architecture. This RFC is a separate proposal:
  * It does not change RFC-0016's connector-driven refresh contract.
  * Its partition transform contract could later be reused by that refresh path.
  * It does extend the legacy `REFRESH MATERIALIZED VIEW ... WHERE` path, which RFC-0016 plans to deprecate. See "Refresh predicates".
* [prestodb/presto#26612](https://github.com/prestodb/presto/issues/26612): Materialized Views support in Presto.
* Implementation, as a stack of pull requests:
  1. [prestodb/presto#28717](https://github.com/prestodb/presto/pull/28717): `feat(spi): Add materialized view rollup extension points`
  2. [prestodb/presto#28718](https://github.com/prestodb/presto/pull/28718): `feat(spi): Add materialized view derivation metadata`
  3. [prestodb/presto#28719](https://github.com/prestodb/presto/pull/28719): `feat(analyzer): Support materialized views over materialized views`
  4. [prestodb/presto#28720](https://github.com/prestodb/presto/pull/28720): `feat(analyzer): Translate transformed MV refresh predicates`
  5. [prestodb/presto#28721](https://github.com/prestodb/presto/pull/28721): `fix(analyzer): Preserve system properties when analyzing views`

## Summary

A rollup is a materialized view (MV) that reads another MV and aggregates it at a coarser grain. For example, it can roll up per-region daily totals into daily totals. Presto rejects this today. This RFC proposes to allow it as an opt-in feature. A rollup is accepted only when the analyzer can prove that its result equals the same aggregation over the base tables.

The proposal has five parts:

1. **Plugin extension points** for two things:
   * aggregate merge strategies, which say which aggregate combines the partial results of another;
   * partition transforms, which say how an MV partition value maps to a range of source partition values.
2. **Derivation metadata** in `MaterializedViewDefinition`: per-column provenance and provider-defined transform metadata. Both are versioned and backward compatible.
3. **MVs over MVs.** `CREATE MATERIALIZED VIEW` can read other MVs when a session property is enabled and both catalogs advertise a new connector capability. Dependency and aggregate checks fail closed.
4. **Transform-aware refresh predicates.** A `REFRESH MATERIALIZED VIEW ... WHERE` predicate on a transformed partition column is rewritten into the exact source range.
5. **A session fix.** Views and MVs analyzed as the caller now see the caller's system session properties.

Everything is off by default except two analyzer changes: `OR` pushdown in `REFRESH ... WHERE`, and the session fix. The Adoption Plan describes both.

## Background

### Rollups

A common layout is a fine-grained pre-aggregation plus coarser summaries built from it. Today an MV can only read base tables, so each summary re-scans the base data. `CREATE MATERIALIZED VIEW` over an MV fails with `CreateMaterializedView on a materialized view <name> is not supported.`

Lifting that restriction alone is unsafe. A source MV holds partial aggregates, and an ordinary aggregate over them can return a plausible but wrong value:

| Source MV column | Rollup | Result |
|---|---|---|
| `COUNT(*) AS orders` | `COUNT(orders)` | Wrong. It counts groups, not rows. Both are `BIGINT`, so types do not catch it. |
| `COUNT(*) AS orders` | `SUM(orders)` | Correct. |
| `SUM(amount) AS revenue` | `SUM(revenue)` | Correct. |
| `MAX(amount) AS largest` | `MAX(largest)` | Correct, even if rows repeat. |
| `AVG(amount) AS avg_amount` | `AVG(avg_amount)` | Wrong. An average of averages weights every group equally. |

Other ways a rollup can go wrong:

* A join can repeat source MV rows. Repeated rows inflate `SUM` and `COUNT` but not `MIN` or `MAX`.
* `DISTINCT` on a rollup aggregate drops equal partial results.
* `FILTER`, or `WHERE` on an aggregated column, selects partial results instead of source rows.
* Subqueries, `WITH`, set operations and logical views can hide an aggregate from a check.

### Partition transforms

An MV can be partitioned at a coarser grain than its source, using a function of the source partition column (for example, month from day). A refresh predicate on such a column cannot be copied to the source as-is. One target value covers a range of source values, so a copied predicate would refresh only part of that range.

Today a refresh predicate on a column that is not directly mapped stays on top of the view query instead of becoming a source-table predicate. Presto has no contract that describes this kind of mapping. Nothing records:

* whether the mapping is one-to-one or many-to-one;
* which source range a target value covers;
* how to store and reload the mapping.

### Session properties during view and MV expansion

The analyzer builds a derived session to analyze a view or MV definition. That session copies the connector properties but only one system property, `legacy_timestamp_with_timezone`. Every other system property falls back to its default inside the view.

Here is an example with legacy materialized views (the default) and `materialized_view_data_consistency_enabled = false`:

* `SELECT count(*) FROM mv` reads only the MV storage.
* The same query through a logical view over `mv` still stitches in base data, because the view's session has the default value `true`.

The two queries return different results.

### Goals

* Allow MVs over MVs, opt-in per session and per connector.
* At creation time, reject any rollup that cannot be proven correct, with an error that says what to change.
* Keep implementation-specific functions out of the engine core. Plugins supply merge rules and partition transforms, and the engine has a small built-in set.
* Persist derivation metadata in the MV definition so that later tiers and refreshes can be validated. The format is versioned and backward compatible.
* Translate refresh predicates on transformed partition columns into exact source ranges.
* Let view and MV expansion honor the caller's system properties, while keeping definer views owned by other users, row filters and column masks isolated from them.

### Non-goals

* Automatic query rewrite or routing to rollup tiers.
* Scheduling or ordering refreshes across tiers.
* Rollup tiers that use `WITH`, subqueries, set operations or window functions.
* Changes to RFC-0016 stitching, `MaterializedViewStatus` or connector-driven refresh.
* Connectors other than Hive in the first phase. Other connectors opt in when they are ready.

## Proposed Implementation

### Modules

The change touches three modules:

* `presto-spi`: the new contracts.
* `presto-main-base`: the registries, validation, refresh predicate translation and the session property.
* `presto-hive`: the capability and metadata persistence.

The appendix maps each part to its classes and its pull request.

### Terminology

* **Source MV and rollup tier.** A source MV is an MV that another MV reads. The rollup tier is the MV that reads it.
* **Merge strategy.** For a source aggregate `f`, the aggregate `g` such that `g` over the partial results of `f` equals `f` over the rows. The built-in strategies are `count → sum`, `sum → sum`, `min → min` and `max → max`.
* **Duplicate-insensitive.** An aggregate whose result does not change when input rows repeat. The built-in ones are `min`, `max`, `arbitrary` and `any_value`. `arbitrary` and `any_value` have no merge function.
* **Column provenance.** How an MV output column was produced: its function, kind, type, and whether it used `DISTINCT`.
* **Partition transform.** A deterministic scalar function, registered by a plugin, that maps a source partition column to an MV partition column.
* **Canonical target value.** A value the transform can produce, such as the first day of a month for a month transform.
* **Source range.** The half-open interval `[lower, upper)` of source values that map to one canonical target value.

In this RFC, "derived column" means an MV output column that is not a direct copy of a source column. It is unrelated to the generated columns of RFC-0025.

### SPI

#### Plugin registration

```java
public interface Plugin
{
    default Iterable<MaterializedViewAggregationMergeStrategy> getMaterializedViewAggregationMergeStrategies()
    {
        return emptyList();
    }

    default Iterable<MaterializedViewPartitionTransformProvider> getMaterializedViewPartitionTransformProviders()
    {
        return emptyList();
    }
}
```

The engine registers both when the plugin is installed. The rules are:

* Registration fails on a duplicate source function, function name or transform ID.
* Plugins cannot override the built-in strategies.
* A batch of strategies is registered atomically.
* Names are case-insensitive.

#### Aggregate merge strategy

API outline:

```java
public final class MaterializedViewAggregationMergeStrategy
{
    public MaterializedViewAggregationMergeStrategy(String sourceFunctionName, Optional<String> mergeFunctionName, boolean duplicateInsensitive);

    public String getSourceFunctionName();

    public Optional<String> getMergeFunctionName();

    public boolean isDuplicateInsensitive();
}
```

A plugin can add strategies for aggregates that the engine does not cover. For example, a plugin that ships a partial/merge pair of functions for averages can register `new MaterializedViewAggregationMergeStrategy("avg_partial", Optional.of("avg_merge"), false)`.

#### Partition transforms

```java
public interface MaterializedViewPartitionTransformProvider
{
    Set<String> getTransformIds();

    Set<String> getFunctionNames();

    Optional<MaterializedViewPartitionTransform> analyze(FunctionMetadata function, List<MaterializedViewPartitionTransformArgument> arguments);

    MaterializedViewPartitionTransform decode(Map<String, String> properties);
}

public interface MaterializedViewPartitionTransform
{
    String TRANSFORM_PROPERTY = "transform";

    String getTransformId();

    MaterializedViewPartitionTransformProperties getProperties();

    Map<String, String> encode();

    boolean isCanonicalTargetValue(ConstantExpression value);

    ConstantExpression apply(ConstantExpression sourceValue);

    MaterializedViewPartitionRange getSourceRange(ConstantExpression targetValue);
}
```

Value classes:

* `MaterializedViewPartitionTransformArgument` is either the source column (with its type) or a constant.
* `MaterializedViewPartitionTransformProperties` holds:
  * the source argument index;
  * the constant argument indexes;
  * the mapping cardinality, `IDENTITY` or `MANY_TO_ONE`;
  * a positive format version.
* `MaterializedViewPartitionRange` has non-null lower and upper bounds of the same type. The lower bound is inclusive and the upper bound is exclusive.

The contract:

* Function names and transform IDs are unique across all installed providers.
* A transform function must be a deterministic scalar function. It is looked up in the Java built-in namespace and in the configured default function namespace (for example, a native function namespace).
* Version 1 transforms are deterministic, preserve nulls, and are monotonically non-decreasing. A transform either keeps the source value (`IDENTITY`) or folds a contiguous range of source values into one target value (`MANY_TO_ONE`).
* `encode()` returns everything needed to rebuild the transform, including the `transform` key. `decode()` returns a transform with the same ID.

#### Derivation metadata in `MaterializedViewDefinition`

API outline:

```java
public Map<String, ColumnProvenance> getColumnProvenance();

public static final class ColumnProvenance
{
    public static final int CURRENT_VERSION = 1;

    public int getVersion();

    public Optional<String> getFunctionName();

    public TransformationSubtype getSubtype();

    public Optional<String> getReturnType();

    public boolean isDistinct();
}

public static final class TableColumn
{
    public Map<String, String> getColumnMappingInfo();
}
```

* `columnProvenance` is keyed by output column name.
  * It covers every named output column, including aggregates with no source-column lineage such as `COUNT(*)`.
  * `functionName` is the lowercase function name. It is recorded only when the function resolves to a built-in function handle, so a custom function that happens to be named `sum` is not recorded as `sum`.
  * `subtype` is one of `IDENTITY`, `TRANSFORMATION` or `AGGREGATION`.
* `columnMappingInfo` holds the encoded transform of a partition transform column. It is empty for ordinary mappings.
* Provenance stores a stable descriptor rather than a resolved `FunctionHandle`. Function handles are not guaranteed to stay compatible across releases, and a view definition outlives many releases.
* Both maps are optional in JSON. Definitions written before this change deserialize with empty maps, and a missing `distinct` reads as `false`. An empty provenance map means "unknown", not "every column is an identity".

#### Connector capability and session property

* `ConnectorCapabilities.SUPPORTS_MATERIALIZED_VIEW_ON_MATERIALIZED_VIEW` means the connector stores the metadata needed to validate rollups. The Hive connector advertises it. Iceberg and other connectors do not advertise it until they opt in.
* `materialized_view_on_materialized_view_enabled` is a boolean session property, default `false`. It allows `CREATE MATERIALIZED VIEW` to read MVs.

### Code flow

#### `CREATE MATERIALIZED VIEW` over an MV

1. During analysis, each source table that is an MV is checked:
   * If the session property is off, the existing "not supported" error is raised.
   * Both the target catalog and the source MV's catalog must advertise the capability.
   * The chain below the source is walked. Self-references, cycles, more than four MV levels and malformed base-table catalog metadata are rejected.
2. When the statement executes, the query-shape and aggregate rules below are checked.
3. Column mappings, provenance and transform metadata are extracted and passed to the connector with the definition. The Hive connector stores provenance and base-table catalogs.

The new MV may sit at most four MV levels above a base table, counting itself. Each extra level compounds staleness, because freshness is only checked one level up. A cycle leaves no valid refresh order.

All the rules below fail closed: anything the analyzer cannot prove correct is rejected.

**Query shape.** These rules apply when the query reads an MV:

* The query is a single `SELECT` over tables, joins and `UNNEST`. It has no `WITH`, subqueries or set operations.
* Source MVs are read directly, not through a logical view.
* Window functions are not allowed.
* A source MV cannot share its schema and table name with a table from another catalog in the same query. Column lineage does not record catalogs, so the two could not be told apart.

**Each aggregate in `SELECT` and `HAVING`:**

1. The aggregate is selected directly, not wrapped in another expression, and resolves to a built-in function.
2. It reads at least one column. `COUNT(*)` is rejected because there is nothing to check it against. It reads at most one column of the source MVs.
3. In two cases, the aggregate must be duplicate-insensitive or use `DISTINCT`:
   * It reads a non-aggregate column of the source MV. That column holds one value per source group, not per source row.
   * It reads only columns of other tables in a join. The join can repeat their rows.
4. If it reads an aggregated column, all of these must hold:
   * The column has provenance of the supported version, with a known function.
   * The aggregate is the registered merge function for that source function.
   * The source aggregate did not use `DISTINCT`, unless the source function is duplicate-insensitive.
   * The aggregate has no `FILTER` clause.
   * The aggregate uses `DISTINCT` only if it is duplicate-insensitive.
   * Its argument is the column itself or a widening `CAST`: to a wider integer type, from `REAL` to `DOUBLE`, or to a `DECIMAL` with at least as many integer and fractional digits.
   * If the `FROM` clause is a join or `UNNEST`, source rows can repeat, so the aggregate must be duplicate-insensitive.

**Other clauses.** `WHERE`, `GROUP BY`, join criteria and `SELECT DISTINCT` cannot read an aggregated column of a source MV.

**Provenance propagation.**

* A pass-through column keeps its source provenance, so a rollup of a rollup can still be checked.
* An aggregated column passed through a join or `UNNEST` loses its function name, unless its function is duplicate-insensitive.
* An expression over an aggregated column is recorded as an aggregate with no function.

The last two cases cannot be merged at the next tier.

Example (Hive connector):

```sql
CREATE MATERIALIZED VIEW sales_daily
WITH (partitioned_by = ARRAY['sale_date'])
AS SELECT customer_region, COUNT(*) AS orders, SUM(amount) AS revenue, MAX(amount) AS largest_order, sale_date
FROM sales
GROUP BY customer_region, sale_date;

SET SESSION materialized_view_on_materialized_view_enabled = true;

CREATE MATERIALIZED VIEW sales_daily_total
WITH (partitioned_by = ARRAY['sale_date'])
AS SELECT SUM(orders) AS orders, SUM(revenue) AS revenue, MAX(largest_order) AS largest_order, sale_date
FROM sales_daily
GROUP BY sale_date;
```

`sales_daily_total` is accepted. If `SUM(orders)` is replaced with `COUNT(orders)`, creation fails with:

```
Column orders of <schema>.sales_daily was produced by count, so rolling it up requires sum, not count.
```

Two more examples:

* `COUNT(*)` fails because it reads no column of the source MV.
* `AVG(revenue)` fails because `revenue` was produced by `sum`, and rolling it up requires `sum`.

#### Partition transforms at creation

An output column is a partition transform when it is a direct call to a registered transform function. The source argument is a single direct source column, and the other arguments are literals. The analyzer then:

1. builds the argument list;
2. calls `provider.analyze`;
3. checks the returned properties against the arguments;
4. stores `transform.encode()` in the column's `columnMappingInfo`.

These cases are rejected with `NOT_SUPPORTED`:

* a wrapped transform, such as `CAST(transform(...) AS ...)`;
* a transform over an expression or over several source columns;
* non-literal configuration arguments;
* a transform in a set-operation MV;
* a provider that rejects the call;
* a registered function that is not a deterministic scalar.

#### Refresh predicates

The analyzer decodes each stored transform through the registry. Unknown or malformed metadata fails analysis.

* **`MANY_TO_ONE`.** A comparison against a canonical target value `v` expands to the source range `[lo, hi)` of `v`:

  | View predicate | Source predicate |
  |---|---|
  | `c = v` | `s >= lo AND s < hi` |
  | `c <> v` | `s < lo OR s >= hi` |
  | `c >= v` | `s >= lo` |
  | `c > v` | `s >= hi` |
  | `c < v` | `s < lo` |
  | `c <= v` | `s < hi` |

  An `IN` list becomes an `OR` of expanded equalities. A non-canonical value is rejected, and the error names the canonical value. Other operators are rejected.
* **`IDENTITY`.** The comparison is copied unchanged.
* **`OR`.** An `OR` is pushed to a source table when both conditions hold:
  * every branch is a supported comparison or `IN` on refresh columns that map to the same single table;
  * that table is not on the outer side of an outer join.

  Before this change, an `OR` always stayed on the view.

For illustration, assume that a plugin registers a `MANY_TO_ONE` provider for `date_trunc('month', <date>)`. An MV is partitioned by `sale_month`, defined as `date_trunc('month', sale_date)`. This statement reads only `sale_date >= DATE '2026-01-01' AND sale_date < DATE '2026-02-01'`:

```sql
REFRESH MATERIALIZED VIEW sales_monthly WHERE sale_month = DATE '2026-01-01';
```

`WHERE sale_month = DATE '2026-01-15'` is rejected as non-canonical.

`REFRESH ... WHERE` is supported only when `legacy_materialized_views` is enabled, which is the default today. RFC-0016 plans to deprecate it. The RFC-0016 path already rejects `WHERE` and derives the refresh scope from connector staleness. This RFC extends `WHERE` because it is the default path today. The transform contract itself does not depend on `WHERE`. Open Question 1 asks whether the RFC-0016 path should reuse it.

#### Session properties during view and MV expansion

The derived session now inherits the caller's system properties when the effective user does not change:

| Expansion | Inherits caller system properties |
|---|---|
| Invoker view or MV | Yes |
| Definer view or MV owned by the caller | Yes |
| Definer view or MV owned by another user | No (unchanged) |
| Row filter and column mask expressions | No (unchanged) |

Copying properties across an identity change would be unsafe, because caller-controlled settings could change the meaning of owner-defined SQL or of security expressions. Connector properties are propagated as before.

### User-facing metrics

None.

## Security Considerations

* **Only registered transforms reach provider code.** A provider applies only to the function names it registers. The function must also resolve in the Java built-in namespace or the configured default namespace, and it must be a deterministic scalar function.
* **Plugin trust.** Transform providers are plugin code. They run on the coordinator with the same trust as any other installed plugin. Merge strategies are plain data.
* **Stored metadata is validated before use.** Analysis fails on any of these:
  * an unknown transform ID;
  * a decoded transform whose ID does not match;
  * metadata that the provider rejects;
  * missing provenance;
  * an unsupported provenance version.
* **Effective-user boundary.** Caller system properties are inherited only when the effective user stays the same. Definer views and MVs owned by another user never see them.
* **Security expressions.** Row filters and column masks are analyzed in a separate session that never inherits caller system properties.
* **Connector opt-in.** The capability is a declaration that the engine trusts, not something it enforces. The provenance check is a second guard: if a connector advertises the capability but does not store provenance, rollups over its MVs are rejected.

## Other Approaches Considered

* **Remove the MV-over-MV restriction and rely on types.** Rejected. `COUNT` over a `COUNT` column goes from `BIGINT` to `BIGINT` and returns a plausible wrong number.
* **Persist a resolved `FunctionHandle` as provenance.** Rejected. Handles are not guaranteed to stay compatible across releases, and MV definitions are long-lived. A name, subtype, type and version are stable.
* **Allow arbitrary query shapes and try to prove them correct.** Rejected for the first phase. Subqueries, CTEs, set operations and views can hide aggregates from the check. The single-`SELECT` rule can be relaxed later, one shape at a time, with tests.
* **Build specific partition functions into the engine core.** Rejected. Partition layouts differ between deployments. A provider SPI keeps implementation-specific functions and formats in plugins.
* **Put all rollup semantics in connectors.** Rejected, for two reasons:
  * Connectors never see the analyzed query. `createMaterializedView` receives the definition, not the analysis, so lineage, function resolution and query shape are not available to them.
  * Each connector would have to reimplement the same rules, and results could differ between connectors.
* **Copy the refresh literal to the source column.** Rejected. For a many-to-one transform, that refreshes an incomplete source range.
* **Always inherit caller system properties during view analysis.** Rejected, for the identity reason given under "Session properties during view and MV expansion".

## Adoption Plan

* **Defaults.** Apart from the two changes below, existing MVs, plugins and queries behave as before:
  * the session property is off;
  * plugins register nothing unless they implement the new methods;
  * only the Hive connector advertises the capability.
* **On-by-default changes.** The release notes call out both:
  * Invoker views, and definer views owned by the caller, now see the caller's system session properties during analysis. This is a bug fix.
  * In `REFRESH ... WHERE`, an `OR` over directly mapped columns of a single table now filters that table instead of the view output. This applies only when the table is not on the outer side of an outer join.
* **New SPI.** The additions are listed under SPI above. All of them are backward compatible for existing plugins and stored definitions. There are no SQL grammar or client API changes.
* **Existing MVs.**
  * New MVs record provenance and transform metadata automatically.
  * MVs created before this change have none. They keep working, but they cannot be rollup sources until they are recreated, and the error message says so.
  * In the Hive connector, recreating an MV drops its table, so the new MV starts empty and must be refreshed again.
  * No migration tool is proposed initially. A metadata-upgrade mechanism can be considered if recreation proves costly in practice.
* **Rollout.** The feature starts as experimental and off by default. Enabling it by default will be discussed separately, after more usage and more connector support.
* **Documentation.**
  * The session property, the rollup rules and examples, in the materialized view docs.
  * An SPI guide for merge strategies and transform providers.
  * The connector capability, in the connector developer docs.
* **Out of scope.** The non-goals above, and additional built-in merge strategies. Each can be proposed separately.

## Test Plan

Unit and integration tests in the implementation pull requests:

* **`presto-spi`.**
  * `TestMaterializedViewAggregationMergeStrategy`, `TestMaterializedViewPartitionRange` and `TestMaterializedViewPartitionTransformProperties`.
  * `TestMaterializedViewDefinition`: JSON round trips and backward compatibility.
* **`presto-main-base`.**
  * `TestMaterializedViewAggregationMergeRegistry`: built-ins, plugin strategies, duplicate rejection and atomic batches.
  * `TestMaterializedViewPartitionTransformRegistry`: namespace lookup, the deterministic scalar requirement, provider ownership and decoding.
  * `TestMaterializedViewColumnMappingExtractor`: provenance, transform validation, and every accept and reject rule above.
  * `TestAnalyzer`: refresh predicate expansion, canonical values, identity transforms and `OR` pushdown.
  * `TestRowFilter` and `TestColumnMask`: caller system properties do not reach security expressions or definer views owned by other users.
* **`presto-hive`.**
  * `TestHiveConnectorFactory`: the capability is advertised.
  * `TestHiveMaterializedViewLogicalPlanner`, end to end:
    * a `COUNT` rollup over a `COUNT` column is rejected, and the `SUM` rollup is accepted;
    * provenance and base-table catalogs survive the metastore round trip;
    * `materialized_view_data_consistency_enabled` is honored through nested logical views.

Before the feature is enabled by default:

* End-to-end tests for refreshing and querying multi-tier chains, including when a source tier is stale.
* Tests for each additional connector that opts in.

## Open Questions

## Appendix: Reference Implementation

The implementation is a stack of five pull requests:

| Part | Pull request | Main classes |
|---|---|---|
| Extension points | [#28717](https://github.com/prestodb/presto/pull/28717) | `Plugin`, the `com.facebook.presto.spi.materializedview` package, `MaterializedViewAggregationMergeRegistry`, `MaterializedViewPartitionTransformRegistry`, `FunctionAndTypeManager`, `PluginManager` |
| Derivation metadata | [#28718](https://github.com/prestodb/presto/pull/28718) | `MaterializedViewDefinition`, `MaterializedViewColumnMappingExtractor`, `CreateMaterializedViewTask` |
| MVs over MVs | [#28719](https://github.com/prestodb/presto/pull/28719) | `StatementAnalyzer`, `MaterializedViewColumnMappingExtractor`, `CreateMaterializedViewTask`, `SystemSessionProperties`, `ConnectorCapabilities`, `HiveConnector`, `HiveMetadata` |
| Refresh predicates | [#28720](https://github.com/prestodb/presto/pull/28720) | `RefreshMaterializedViewPredicateAnalyzer` |
| Session fix | [#28721](https://github.com/prestodb/presto/pull/28721) | `StatementAnalyzer` |

`CREATE MATERIALIZED VIEW` over an MV, by class:

```
StatementAnalyzer (analysis)
  validateBaseTables
    source is not an MV                        -> unchanged
    session property disabled                  -> existing "not supported" error
    target or source catalog lacks capability  -> reject
    walk the source chain                      -> reject self-reference, cycles,
                                                  more than four MV levels,
                                                  malformed base-table catalogs

CreateMaterializedViewTask (execution)
  MaterializedViewColumnMappingExtractor
    validateAggregateMergePartners             -> query-shape and aggregate rules
    column mappings, provenance, transform metadata
  ConnectorMetadata.createMaterializedView     -> Hive stores provenance and
                                                  base-table catalogs
```

The session fix replaces the single derived-session builder in `StatementAnalyzer` with two:

* `createViewSession` inherits caller system properties when the effective user is unchanged.
* `createSecurityExpressionSession` never inherits them, and is used for row filters and column masks.
