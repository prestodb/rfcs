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

A SCALAR function such as `upper(COLUMN1)` could be annotated with `propagateStatistics: true`, and would reuse source statistics of `COLUMN1`.
Another example : `is_null(COLUMN1)` could be annotated with `distinctValueCount: 2`, `nullFraction: 0.0` and `avgRowSize: 1.0` and would have these stats values hardcoded


## Background
Currently, we compute new stats for only a handful of functions (`CAST(...)`, `is_null(Slice)`, arithmetic operators) by using a hand rolled implementation in [`ScalarStatsCalculator`](https://github.com/prestodb/presto/blob/0.288-edge7/presto-main/src/main/java/com/facebook/presto/cost/ScalarStatsCalculator.java#L118))

For other functions the optimizer cannot estimate statistics since it is unaware of how source stats need to be transformed.
[Related Issues](#related-issues) [1] and [2] are examples of queries we could have been optimized better if there was some way to 
derive source stats. `VariableStatsEstimate.unknown()` is propgated for these stats (Shown below as `?` )

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
see [duckdb#113](https://github.com/duckdb/duckdb/pull/1133/files#diff-0833c08516123b2a2b239dd25daabdf5a3b0da8c51e00fddbcfd7965c47bc4b6)


#### Apache spark:

Acc. https://jaceklaskowski.gitbooks.io/mastering-spark-sql/content/spark-sql-udfs-blackbox.html , spark treats UDFs as black box. 
In the example below, Spark seems to propagate stats for even UDFs (TODO : code citation needed)

```

scala> spark.udf.register("spl_upper", (_: String).toUpperCase())
res18: org.apache.spark.sql.expressions.UserDefinedFunction = SparkUserDefinedFunction($Lambda$4325/0x00007a599546f940@681a2cd2,StringType,List(Some(class[value[0]: string])),Some(class[value[0]: string]),Some(spl_upper),true,true)


scala> spark.sql("select * from employee as e, employee2 as e2 where spl_upper(e2.firstname) != spl_upper(e.lastname)").explain(true);
== Parsed Logical Plan ==
'Project [*]
+- 'Filter NOT ('spl_upper('e2.firstname) = 'spl_upper('e.lastname))
+- 'Join Inner
  :- 'SubqueryAlias e
  :  +- 'UnresolvedRelation [employee], [], false
+- 'SubqueryAlias e2
  +- 'UnresolvedRelation [employee2], [], false

== Analyzed Logical Plan ==
firstname: string, middlename: string, lastname: string, id: string, gender: string, salary: int, firstname: string, middlename: string, lastname: string, id: string, gender: string, salary: int
Project [firstname#88, middlename#89, lastname#90, id#91, gender#92, salary#93, firstname#275, middlename#276, lastname#277, id#278, gender#279, salary#280]
+- Filter NOT (spl_upper(firstname#275) = spl_upper(lastname#90))
+- Join Inner
:- SubqueryAlias e
  :  +- SubqueryAlias employee
:     +- View (`employee`, [firstname#88,middlename#89,lastname#90,id#91,gender#92,salary#93])
:        +- LogicalRDD [firstname#88, middlename#89, lastname#90, id#91, gender#92, salary#93], false
+- SubqueryAlias e2
+- SubqueryAlias employee2
+- View (`employee2`, [firstname#275,middlename#276,lastname#277,id#278,gender#279,salary#280])
+- LogicalRDD [firstname#275, middlename#276, lastname#277, id#278, gender#279, salary#280], false

== Optimized Logical Plan ==
Join Inner, NOT (spl_upper(firstname#275) = spl_upper(lastname#90))
:- LogicalRDD [firstname#88, middlename#89, lastname#90, id#91, gender#92, salary#93], false
+- LogicalRDD [firstname#275, middlename#276, lastname#277, id#278, gender#279, salary#280], false

== Physical Plan ==
  CartesianProduct NOT (spl_upper(firstname#275) = spl_upper(lastname#90))
:- *(1) Scan ExistingRDD[firstname#88,middlename#89,lastname#90,id#91,gender#92,salary#93]
+- *(2) Scan ExistingRDD[firstname#275,middlename#276,lastname#277,id#278,gender#279,salary#280]


scala> spark.sql("select * from employee as e, employee2 as e2 where e2.firstname != e.lastname").explain(true);
== Parsed Logical Plan ==
'Project [*]
+- 'Filter NOT ('e2.firstname = 'e.lastname)
+- 'Join Inner
  :- 'SubqueryAlias e
  :  +- 'UnresolvedRelation [employee], [], false
+- 'SubqueryAlias e2
  +- 'UnresolvedRelation [employee2], [], false

== Analyzed Logical Plan ==
firstname: string, middlename: string, lastname: string, id: string, gender: string, salary: int, firstname: string, middlename: string, lastname: string, id: string, gender: string, salary: int
Project [firstname#88, middlename#89, lastname#90, id#91, gender#92, salary#93, firstname#299, middlename#300, lastname#301, id#302, gender#303, salary#304]
+- Filter NOT (firstname#299 = lastname#90)
+- Join Inner
:- SubqueryAlias e
  :  +- SubqueryAlias employee
:     +- View (`employee`, [firstname#88,middlename#89,lastname#90,id#91,gender#92,salary#93])
:        +- LogicalRDD [firstname#88, middlename#89, lastname#90, id#91, gender#92, salary#93], false
+- SubqueryAlias e2
+- SubqueryAlias employee2
+- View (`employee2`, [firstname#299,middlename#300,lastname#301,id#302,gender#303,salary#304])
+- LogicalRDD [firstname#299, middlename#300, lastname#301, id#302, gender#303, salary#304], false

== Optimized Logical Plan ==
Join Inner, NOT (firstname#299 = lastname#90)
:- Filter isnotnull(lastname#90)
:  +- LogicalRDD [firstname#88, middlename#89, lastname#90, id#91, gender#92, salary#93], false
+- Filter isnotnull(firstname#299)
+- LogicalRDD [firstname#299, middlename#300, lastname#301, id#302, gender#303, salary#304], false

== Physical Plan ==
  CartesianProduct NOT (firstname#299 = lastname#90)
:- *(1) Filter isnotnull(lastname#90)
:  +- *(1) Scan ExistingRDD[firstname#88,middlename#89,lastname#90,id#91,gender#92,salary#93]
+- *(2) Filter isnotnull(firstname#299)
+- *(2) Scan ExistingRDD[firstname#299,middlename#300,lastname#301,id#302,gender#303,salary#304]

```

### Goals

1. Provide a mechanism for transforming source statistics for scalar functions
1. Show impact as plan changes for benchmark queries

### Non-goals
1. Non-SCALAR functions will not be annotated
1. Runtime metrics to measure latency/CPU impact of performing these transformations
1. Give accurate stats by either sampling or applying a statistical technique.

## Proposed Implementation

We propose a phase wise implementation plan -

* __Phase 1__.

 * Support builtin scalar functions stats propagation implementation for JAVA.

 * We plan to introduce following 2 Annotations i.e. `ScalarFunctionConstantStats` and `ScalarPropagateSourceStats` 
in java with fields as follows:

```java
/**
 * By default, a function is just a “black box” that the database system knows very little about the behavior of.
 * However, that means that queries using the function may be executed much less efficiently than they could be.
 * It is possible to supply additional knowledge that helps the planner optimize function calls.
 * Scalar functions are straight forward to optimize and can have impact on the overall query performance.
 * Use this annotation to provide information regarding how this function impacts following query statistics.
 * <p>
 * A function may take one or more input column or a constant as parameters. Precise stats may depend on the input
 * parameters. This annotation does not cover all the possible cases and allows constant values for the following fields.
 * Value Double.NaN implies unknown.
 * </p>
 */
@Retention(RUNTIME)
@Target(METHOD)
public @interface ScalarFunctionConstantStats
{
  // Min max value is NaN if unknown.
  double minValue() default Double.NaN;
  double maxValue() default Double.NaN;

  // Histogram
  HistogramTypes histogram() default HistogramTypes.UNKNOWN;

  /**
   * Does this function produces a constant Distinct value count regardless of `input column`'s source stats.
   */
  double distinctValuesCount() default Double.NaN;

  /**
   * Does this function produce a constant nullFraction, e.g. is_null(Slice) will alter column's null fraction
   * value to 0.0.
   */
  double nullFraction() default Double.NaN;

  /**
   * An `avgRowSize`: does this function impacts the size of each row e.g. a function like md5 may produce a
   * constant row size.
   */
  double avgRowSize() default Double.NaN;
}

```

```java

@Retention(RUNTIME)
@Target(PARAMETER)
public @interface ScalarPropagateSourceStats
{
    boolean propagateAllStats() default false;

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
    ROW_COUNT_TIMES_INV_NULL_FRACTION,
    /** use the value of TYPE_WIDTH in varchar(TYPE_WIDTH) */
    USE_TYPE_WIDTH_VARCHAR,
    /** Take max of type width of arguments with varchar type. */
    MAX_TYPE_WIDTH_VARCHAR,
    /** Stats are unknown and thus no action is performed. */
    UNKNOWN
}
```

Examples of using above annotations for various function in `StringFunctions.java`

1. upper(Slice)
```java
    @Description("converts the string to upper case")
    @ScalarFunction
    @LiteralParameters("x")
    @SqlType("varchar(x)")
    public static Slice upper(@ScalarPropagateSourceStats(propagateAllStats = true) @SqlType("varchar(x)") Slice slice)
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
    @ScalarFunctionConstantStats(distinctValuesCount = 2, nullFraction = 0, minValue = 0, maxValue = 1)
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
    public static Slice concat(@LiteralParameter("x") Long x,
            @ScalarPropagateSourceStats(propagateAllStats = true, nullFraction = USE_MAX_ARGUMENT, avgRowSize = SUM_ARGUMENTS, distinctValuesCount = ROW_COUNT) @SqlType("char(x)") Slice left,
            @SqlType("char(y)") Slice right)
    { //...
}
```
 __Propagate all stats but, for nullFraction, avgRowSize and distinctValuesCount follow as specified.__

4. An example of using both `ScalarFunctionConstantStats` and `ScalarPropagateSourceStats`.
```java
    @ScalarFunctionConstantStats(minValue = 0)
    public static long levenshteinDistance(
            @ScalarPropagateSourceStats(propagateAllStats = true, maxValue = MAX_TYPE_WIDTH_VARCHAR, distinctValuesCount = MAX_TYPE_WIDTH_VARCHAR) @SqlType("varchar(x)") Slice left,
            @SqlType("varchar(y)") Slice right){ 
    //... 
  }
```

* __Phase 2__. Annotate some of the builtin scalar functions with above annotation.

* __Phase 3__. Extend the support to non-built-in UDFs. Add documentations and blogs with usage examples.

* __Future work__.

The following example is a more powerful form of expressing how to estimate stats for functions based on their characteristics. Here,
we propose the use of expressions ( i.e an expression language with a fixed grammar ) In the following example of `substr` :
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

See prior art, for how postgres handles it.

## Metrics

How can we measure the impact of this feature?

1. Existing TPCDS and TPCH queries which use functions or `LIKE %STR` in join condition will benefit from this optimization.
2. Join order benchmark: Reference [2](#references)

## Other Approaches Considered

Based on the discussion, this may need to be updated with feedback from reviewers.

* __Approach 1__. Trying to find a heuristic based mechanism to estimate functions cost, for example : A function takes a single argument of type `varchar(20)` and
returns the value of same type i.e. `varchar(20)` and is SCALAR and deterministic, then we can apply heuristic and say that source statistics can be propagated as is.

Example 2: The input and output column do not match, input is `varchar(200)` and output is `varchar(10)`.Then we can say that average row size is changed.

__Pros:__

* Minimum impact to the user and codebase. Since this feature can be 
enabled/disabled using a session flag : `scalar_function_stats_propagation_enabled`

__Cons:__
* Problem with heuristic based approach is unknown nature of functions, and thus it can be difficult to come up with heuristics that are always accurate.

* __Approach 2__. Reference [1. Monsoon](#references). discusses various stochastic approaches to solve the same problem, a user provided stats calculator will complement such
approaches as one could compute via statistical process for those functions user provided stats computation is unavailable. Sampling and other stochastic processes are beyond the
scope for this RFC.

## Adoption Plan

- What impact (if any) will there be on existing users? Are there any new session parameters, configurations, SPI updates, client API updates, or SQL grammar?


Functions without this annotation will not be affect, to remain backwards compatible those functions without this Annotation will run as is. Also, a new session 
config will be introduced i.e. `optimizer.scalar-function-stats-propagation-enabled=false`, by setting this flag this feature can be turned off.

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
