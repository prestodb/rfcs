# **RFC0004 for Presto**

## Propagate statistics for SCALAR functions

Proposers

* Prashant Sharma (@ScrapCodes)
* Anant Aneja (@aaneja)

## [Related Issues]

1. https://github.com/prestodb/presto/issues/20513
2. https://github.com/prestodb/presto/issues/19356


## Summary
Provide a set of annotations for the scalar functions to specify how input statistics are transformed due to application of the function.

A function is just a “black box” for the database system and thus the function may be executed much less efficiently than they could be.
It is possible to supply additional knowledge that helps the planner optimize function calls.

A SCALAR function such as `upper(COLUMN1)` could be annotated with `propagateStatistics: true`, and would reuse source statistics of `COLUMN1`.
Another example : `is_null(COLUMN1)` could be annotated with `distinctValueCount: 2`, `nullFraction: 0.0` and `avgRowSize: 1.0` and would have these stats values hardcoded

## Background
Currently, we compute new stats for only a handful of functions (`CAST(...)`, `is_null(Slice)`, arithmetic operators) by using a hand rolled implementation in
[`ScalarStatsCalculator`](https://github.com/prestodb/presto/blob/0.289-edge7/presto-main/src/main/java/com/facebook/presto/cost/ScalarStatsCalculator.java#L118))

For other functions the optimizer cannot estimate statistics since it is unaware of how source stats need to be transformed.
[Related Issues](#related-issues) [1] and [2] are examples of queries we could have been optimized better if there was some way to 
derive source stats. [VariableStatsEstimate](https://github.com/prestodb/presto/blob/0.289-edge7/presto-main/src/main/java/com/facebook/presto/cost/VariableStatsEstimate.java#L35)
is used to store stats and `VariableStatsEstimate.unknown()` is propagated in cases where stats are missing (Shown below as `?` )

```
presto:tpcds_sf1_parquet> explain select 1 FROM customer, customer_address WHERE c_birth_country = upper(ca_country);
                                                                                                                                                                    Query Plan                          >
-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------->
 - Output[_col0] => [expr:integer]                                                                                                                                                                      >
         Estimates: {source: CostBasedSourceInfo, rows: ? (?), cpu: ?, memory: 2240933.00, network: ?}                                                                                                  >
         _col0 := expr (1:16)                                                                                                                                                                           >
     - RemoteStreamingExchange[GATHER] => [expr:integer]                                                                                                                                                >
             Estimates: {source: CostBasedSourceInfo, rows: ? (?), cpu: ?, memory: 2240933.00, network: ?}                                                                                              >
         - Project[projectLocality = LOCAL] => [expr:integer]                                                                                                                                           >
                 Estimates: {source: CostBasedSourceInfo, rows: ? (?), cpu: ?, memory: 2240933.00, network: 2240933.00}                                                                                 >
                 expr := INTEGER'1'                                                                                                                                                                     >
             - InnerJoin[("upper" = "c_birth_country")][$hashvalue, $hashvalue_6] => []                                                                                                                 >
                     Estimates: {source: CostBasedSourceInfo, rows: ? (?), cpu: 18093907.00, memory: 2240933.00, network: 2240933.00}                                                                   >
                     Distribution: REPLICATED                                                                                                                                                           >
                 - Project[projectLocality = LOCAL] => [upper:varchar(20), $hashvalue:bigint]                                                                                                           >
                         Estimates: {source: CostBasedSourceInfo, rows: 50000 (244.14kB), cpu: 6830175.00, memory: 0.00, network: 0.00}                                                                 >
                         $hashvalue := combine_hash(BIGINT'0', COALESCE($operator$hash_code(upper), BIGINT'0')) (1:34)                                                                                  >
                     - ScanProject[table = TableHandle {connectorId='local_hms', connectorHandle='HiveTableHandle{schemaName=tpcds_sf1_parquet, tableName=customer_address, analyzePartitionValues=Optio>
                             Estimates: {source: CostBasedSourceInfo, rows: 50000 (244.14kB), cpu: 880175.00, memory: 0.00, network: 0.00}/{source: CostBasedSourceInfo, rows: 50000 (244.14kB), cpu: 36>
                             upper := upper(ca_country) (1:34)                                                                                                                                          >
                             LAYOUT: tpcds_sf1_parquet.customer_address{}                                                                                                                               >
                             ca_country := ca_country:varchar(20):10:REGULAR (1:33)                                                                                                                     >
                 - LocalExchange[HASH][$hashvalue_6] (c_birth_country) => [c_birth_country:varchar(20), $hashvalue_6:bigint]                                                                            >
                         Estimates: {source: CostBasedSourceInfo, rows: 100000 (488.28kB), cpu: 5822799.00, memory: 0.00, network: 2240933.00}                                                          >
                     - RemoteStreamingExchange[REPLICATE] => [c_birth_country:varchar(20), $hashvalue_7:bigint]                                                                                         >
                             Estimates: {source: CostBasedSourceInfo, rows: 100000 (488.28kB), cpu: 3581866.00, memory: 0.00, network: 2240933.00}                                                      >
                         - ScanProject[table = TableHandle {connectorId='local_hms', connectorHandle='HiveTableHandle{schemaName=tpcds_sf1_parquet, tableName=customer, analyzePartitionValues=Optional.>
                                 Estimates: {source: CostBasedSourceInfo, rows: 100000 (488.28kB), cpu: 1340933.00, memory: 0.00, network: 0.00}/{source: CostBasedSourceInfo, rows: 100000 (488.28kB), >
                                 $hashvalue_8 := combine_hash(BIGINT'0', COALESCE($operator$hash_code(c_birth_country), BIGINT'0')) (1:23)                                                              >
                                 LAYOUT: tpcds_sf1_parquet.customer{}                                                                                                                                   >
                                 c_birth_country := c_birth_country:varchar(20):14:REGULAR (1:23)      
```

This proposal discusses alternatives to adding special cases for optimising scalar udfs or functions.

### Prior-Art

#### PostgreSQL
PostgreSQL allows the user to provide a support function which can compute either propagate/calculating stats.
e.g. https://www.postgresql.org/docs/15/xfunc-optimization.html . Postgresql refers to these
as `planner support functions` (postgresql/src/include/nodes/supportnodes.h).

#### DuckDb
DuckDb also allows the user to specify a similar helper function to transform source stats,
see [duckdb#1133](https://github.com/duckdb/duckdb/pull/1133/files#diff-0833c08516123b2a2b239dd25daabdf5a3b0da8c51e00fddbcfd7965c47bc4b6)

### Goals

1. Provide a mechanism for transforming source statistics for scalar functions
2. Show impact as plan changes for benchmark queries

### Non-goals
1. Non-SCALAR functions will not be annotated
2. Runtime metrics to measure latency/CPU impact of performing these transformations
3. Give accurate stats by either sampling or applying a statistical technique.

## Proposed Implementation

We propose a phase wise implementation plan -

### Phase 1 - JAVA Function annotations

 * Support builtin scalar functions stats propagation implementation for JAVA.

 * We plan to introduce the following 2 Annotations i.e. `ScalarFunctionConstantStats` and `ScalarPropagateSourceStats` 
in java with fields as follows:

```java
@Retention(RUNTIME)
@Target(METHOD)
public @interface ScalarFunctionConstantStats
{
  // Min max value is NaN if unknown.
  double minValue() default Double.NaN;
  double maxValue() default Double.NaN;

  /**
   * The constant distinct values count to be applied.
   */
  double distinctValuesCount() default Double.NaN;

  /**
   * The constant nullFraction of the resulting function. For example, is_null(<column>) should alter this value to 0.0
   * value to 0.0.
   */
  double nullFraction() default Double.NaN;

  /**
   * A constant representing the average output size per row from this function. For example, hash functions like md5 or SHA512 will always produce a constant-length output.
   * constant row size.
   */
  double avgRowSize() default Double.NaN;
}

```

Note: Constant stats takes precedence over all.

```java
@Retention(RUNTIME)
@Target(PARAMETER)
public @interface ScalarPropagateSourceStats
{
    boolean propagateAllStats() default true;

    StatsPropagationBehavior minValue() default StatsPropagationBehavior.UNKNOWN;
    StatsPropagationBehavior maxValue() default StatsPropagationBehavior.UNKNOWN;
    StatsPropagationBehavior distinctValuesCount() default StatsPropagationBehavior.UNKNOWN;
    StatsPropagationBehavior avgRowSize() default StatsPropagationBehavior.UNKNOWN;
    StatsPropagationBehavior nullFraction() default StatsPropagationBehavior.UNKNOWN;
    StatsPropagationBehavior histogram() default StatsPropagationBehavior.UNKNOWN;
}
```

`StatsPropagationBehavior` is an enum with following options to control how stats are handled.

```java

public enum StatsPropagationBehavior
{
    /** Use the max value across all arguments to derive the new stats value */
    USE_MAX_ARGUMENT,
    /** Sum the stats value of all arguments to derive the new stats value */
    SUM_ARGUMENTS,
    /** Propagate the source stats as-is */
    USE_SOURCE_STATS,
    /** Use the value of output row count. */
    ROW_COUNT,
    /** Use the value of row_count * (1 - null_fraction). */
    NON_NULL_ROW_COUNT,
    /** use the value of TYPE_WIDTH in varchar(TYPE_WIDTH) */
    USE_TYPE_WIDTH_VARCHAR,
    /** Take max of type width of arguments with varchar type. */
    MAX_TYPE_WIDTH_VARCHAR,
    /** Stats are unknown and thus no action is performed. */
    UNKNOWN
}
```

`FunctionMetadata` is expanded to include stats, which is available for processing at
[ScalarStatsCalculator](https://github.com/prestodb/presto/blob/0.289-edge7/presto-main/src/main/java/com/facebook/presto/cost/ScalarStatsCalculator.java#L124)

```java
public class FunctionMetadata
{
    // existing fields...
    private final Map<Signature, ScalarStatsHeader> statsHeader;
    // ...
}
```

Examples of using above annotations for various function in `StringFunctions.java`

1. upper(Slice)
```java
    @Description("converts the string to upper case")
    @ScalarFunction
    @LiteralParameters("x")
    @SqlType("varchar(x)")
    public static Slice upper(
            @ScalarPropagateSourceStats @SqlType("varchar(x)") Slice slice)
    {
        return toUpperCase(slice);
    }
```

2. starts_with(Slice, Slice): an example of constant stats.
```java
    @SqlNullable
    @Description("Returns whether the first string starts with the second")
    @ScalarFunction("starts_with")
    @LiteralParameters({"x", "y"})
    @SqlType(StandardTypes.BOOLEAN)
    @ScalarFunctionConstantStats(distinctValuesCount = 2, minValue = 0, maxValue = 1)
    public static Boolean startsWith(@SqlType("varchar(x)") Slice x, @SqlType("varchar(y)") Slice y)
    {
        if (x.length() < y.length()) {
            return false;
        }

        return x.equals(0, y.length(), y, 0, y.length());
    }
```

3. concat(Slice, Slice): 

```java
   @Description("concatenates given character strings")
    @ScalarFunction
    @LiteralParameters({"x", "y", "u"})
    @Constraint(variable = "u", expression = "x + y")
    @SqlType("char(u)")
    public static Slice concat(
            @LiteralParameter("x") Long x,
            @ScalarPropagateSourceStats(
                    nullFraction = USE_MAX_ARGUMENT,
                    avgRowSize = SUM_ARGUMENTS,
                    distinctValuesCount = ROW_COUNT) @SqlType("char(x)") Slice left,
            @SqlType("char(y)") Slice right)
    { //...
}
```

 __Propagate all stats but, for nullFraction, avgRowSize and distinctValuesCount follow as specified.__

4. An example of using both `ScalarFunctionConstantStats` and `ScalarPropagateSourceStats`.
```java
@Description("computes Levenshtein distance between two strings")
@ScalarFunction
@LiteralParameters({"x", "y"})
@SqlType(StandardTypes.BIGINT)
@ScalarFunctionConstantStats(minValue = 0, avgRowSize = 8)
public static long levenshteinDistance(
        @ScalarPropagateSourceStats(
                propagateAllStats = false,
                maxValue = MAX_TYPE_WIDTH_VARCHAR,
                distinctValuesCount = MAX_TYPE_WIDTH_VARCHAR,
                nullFraction = SUM_ARGUMENTS) @SqlType("varchar(x)") Slice left,
        @SqlType("varchar(y)") Slice right)
{ //... 
}
```

__`minValue` is 0 and `maxValue` and `distinctValuesCount` can be max of varchar(type_width) of `left` and `right` i.e. max(x, y),
`nullFraction` is sum of source stats of both `left` and `right`__

A precise value of `minValue` can even be more than 0 for a particular dataset and same for `maxValue` can be different too, the goal is to estimate with some 
knowledge of function characteristics. It is not always possible to provide an accurate value, a best-effort estimate is used.

### Phase 2: C++ implementation.

For C++ functions, [VectorFunctionMetadata](https://github.com/facebookincubator/velox/blob/0adc62e3aaf647596ab476f6300a4165e60f6b91/velox/expression/FunctionMetadata.h)
is expanded to include `constantStats` and `transformSourceStats` as follows.

```c++

struct VectorFunctionMetadata { 
   // ....
   
  /// Constant stats produced by the function, NAN represent unknown values.
  /// If constant stats are provided they take precedence over the results of
  /// transformSourceStats
  VectorFunctionConstantStatistics constantStats;

  /// map of function position argument to source statistics transformation.
  std::map<int, VectorFunctionPropagateSourceStats> transformSourceStats;
};
```

Where `VectorFunctionConstantStatistics` and `VectorFunctionPropagateSourceStats` is  defined as follows :

```c++
struct VectorFunctionPropagateSourceStats {

  bool propagateAllStats{true};
  
  StatsPropagationBehavior lowValue{UNKNOWN};

  StatsPropagationBehavior highValue{UNKNOWN};

  StatsPropagationBehavior nullFraction{UNKNOWN};

  StatsPropagationBehavior averageRowSize{UNKNOWN};

  StatsPropagationBehavior distinctValuesCount{UNKNOWN};
};

struct VectorFunctionConstantStatistics {
  double lowValue{NAN};

  double highValue{NAN};

  double nullFraction{NAN};

  double averageRowSize{NAN};

  double distinctValuesCount{NAN};
};
```

`StatsPropagationBehavior` is an enum that defines different ways stats for functions can be estimated.

```c++
namespace facebook::velox::exec {
enum StatsPropagationBehavior {
  /** Use the max value across all arguments to derive the new stats value */
  USE_MAX_ARGUMENT,
  /** Sum the stats value of all arguments to derive the new stats value */
  SUM_ARGUMENTS,
  /** Propagate the source stats as-is */
  USE_SOURCE_STATS,
  /** Use the value of output row count. */
  ROW_COUNT,
  /** Use the value of row_count * (1 - null_fraction). */
  ROW_COUNT_TIMES_INV_NULL_FRACTION,
  /** use the value of TYPE_WIDTH in varchar(TYPE_WIDTH) */
  USE_TYPE_WIDTH_VARCHAR,
  /** Take max of type width of arguments with varchar type. */
  MAX_TYPE_WIDTH_VARCHAR,
  /** Stats are unknown and thus no action is performed. */
  UNKNOWN
};
}
```

Examples for C++ functions:

Following is how metadata would look like for
[ConcatFunction::metadata](https://github.com/facebookincubator/velox/blob/0adc62e3aaf647596ab476f6300a4165e60f6b91/velox/functions/prestosql/StringFunctions.cpp#L273):
```c++
static exec::VectorFunctionMetadata metadata() {
return {
    .supportsFlattening = true,
    .transformSourceStats = {
        {0,
         exec::VectorFunctionPropagateSourceStats{
             .propagateAllStats= false,
             .nullFraction = exec::SUM_ARGUMENTS,
             .averageRowSize = exec::SUM_ARGUMENTS}}}};
}
```

#### How does a C++ worker communicate functions stats behavior and constant stats to the coordinator.

The ongoing work described in [RFC-0003](RFC-0003-native-spi.md), with function registries already includes function metadata as field .
Since we are introducing fields inside the
[FunctionMetadata.h](https://github.com/facebookincubator/velox/blob/0adc62e3aaf647596ab476f6300a4165e60f6b91/velox/expression/FunctionMetadata.h)
the new fields have to be sent as is for further processing to the coordinator.

In both Java and C++, builtin functions have extra metadata defined in FunctionMetadata class, which is used while computing estimated stats
for optimizer.

### Phase 3: Annotate some of the builtin scalar functions with appropriate annotations.

See: [Appendix](#appendix) 1, for list of functions.

### Phase 4: Extend the support to non-built-in UDFs. Add documentation and blogs with usage examples.

### Future work:

#### Define an expression language:
The following example is a more powerful form of expressing how to estimate stats for functions based on their characteristics. Here,
we propose the use of expressions ( i.e. an expression language with a fixed grammar ) In the following example of `substr` :
For average row size we have used the value of length argument. For distinct values count it is more complex, e.g. if length is same as
the upper bound for `varchar(x)` i.e. `x` , then use source stat `distinct values count` of the argument `utf8`. 
```java
    @Description("substring of given length starting at an index")
    @ScalarFunction
    @LiteralParameters("x")
    @SqlType("varchar(x)")
    public static Slice substr(
            @ScalarPropagateSourceStats(
                    nullFraction = USE_SOURCE_STATS,
                    distinctValuesCount = "value(arg_length) <3 ? estimate_ndv(x, value(arg_length), BASIC_LATIN, min(output_row_count, source_stats_ndv())) * (1- source_stats_null_fraction()) : UNKNOWN",
                    avgRowSize = "value(arg_length)" ) @SqlType("varchar(x)") Slice utf8,
            @SqlType(StandardTypes.BIGINT) long start,
            @SqlType(StandardTypes.BIGINT) long length)
    { //...
    }
```

`substr` is at the heart of all `LIKE %X` statements, which is widely used. Without a more powerful framework support it is difficult to support a stats estimation. 

#### virtual function
C++ [VectorFunction](https://github.com/facebookincubator/velox/blob/0adc62e3aaf647596ab476f6300a4165e60f6b91/velox/expression/VectorFunction.h) class definition to include
virtual function which can be overridden by each function to provide how the source stats are transformed.

## Metrics

How can we measure the impact of this feature?

1. Existing TPCDS and TPCH queries which use functions or `LIKE %STR` in join condition will benefit from this optimization.
2. Join order benchmark: Reference [2](#references)

## Other Approaches Considered

Based on the discussion, this may need to be updated with feedback from reviewers.

### Approach 1 
Trying to find a heuristic based mechanism to estimate functions cost, for example : A function takes a single argument of type `varchar(20)` and
returns the value of same type i.e. `varchar(20)` and is SCALAR and deterministic, then we can apply heuristic and say that source statistics can be propagated as is.

Example 2: The input and output column do not match, input is `varchar(200)` and output is `varchar(10)`.Then we can say that average row size is changed.

#### Pros

* Minimum impact to the user and codebase. Since this feature can be 
enabled/disabled using a session flag : `scalar_function_stats_propagation_enabled`

#### Cons
* Problem with heuristic based approach is unknown nature of functions, and thus it can be difficult to come up with heuristics that are always accurate.

### Approach 2: 
Reference [Monsoon](#references), discusses various stochastic approaches to solve the same problem, a user provided stats calculator will complement such
approaches as one could compute via statistical process for those functions user provided stats computation is unavailable. Sampling and other stochastic processes are beyond the
scope for this RFC.

## Adoption Plan

- What impact (if any) will there be on existing users? Are there any new session parameters, configurations, SPI updates, client API updates, or SQL grammar?


Functions without this annotation will not be affect, to remain backwards compatible those functions without this Annotation will run as is. A new session flag,
`scalar_function_stats_propagation_enabled` and a new feature config will be introduced i.e. `optimizer.scalar-function-stats-propagation-enabled`,
by setting this session flag or feature flag, this feature can be turned on or off.

The existing users are not impacted, as we are not changing any existing APIs, however those who wish to leverage new API can migrate their functions. 


- How should this feature be taught to new and existing users? Basically mention if documentation changes/new blog are needed?
  - Documentation and blogs are planned to introduce this feature to the Presto community.

- What related issues do you consider out of scope for this RFC that could be addressed in the future independently of the solution that comes out of this RFC?
  - An ability to provide an expression to compute stats. 

## Test Plan

How do we ensure the feature works as expected? Mention if any functional tests/integration tests are needed. Special mention for product-test changes. If any PoC has been done already, please mention the relevant test results here that you think will bolster your case of getting this RFC approved.

- Both functional tests and integration tests are needed, including unit tests where applicable.
- A POC is available: https://github.com/prestodb/presto/pull/22974

## References

1. [Monsoon: Multi-step optimization and execution of queries with partially obscured predicates](https://scholar.google.com/citations?view_op=view_citation&hl=en&user=2AjuufAAAAAJ&citation_for_view=2AjuufAAAAAJ:UeHWp8X0CEIC)
2. "How Good Are Query Optimizers, Really?"
   by Viktor Leis, Andrey Gubichev, Atans Mirchev, Peter Boncz, Alfons Kemper, Thomas Neumann
   PVLDB Volume 9, No. 3, 2015
   http://www.vldb.org/pvldb/vol9/p204-leis.pdf

## Appendix

1. List of functions: Note: At this stage an exhaustive list is not ready (WIP).
   1. All functions in presto-main/src/main/java/com/facebook/presto/operator/scalar/MathFunctions.java
   2. All functions in presto-main/src/main/java/com/facebook/presto/operator/scalar/StringFunctions.java
   3. `getHash` in presto-main/src/main/java/com/facebook/presto/operator/scalar/CombineHashFunction.java
   4. `yearFromTimestamp` in presto-main/src/main/java/com/facebook/presto/operator/scalar/DateTimeFunctions.java

