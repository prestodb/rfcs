# **RFC0006 for Presto**


## Defining behavior for NaN values

Proposers

* Rebecca Schlussel

## Related Issues

* https://github.com/prestodb/presto/issues/21936 
* https://github.com/prestodb/presto/issues/21877 
* https://github.com/facebookincubator/velox/pull/7237 

## Summary

Presto is inconsistent in its ordering and comparison of NaN (not a number) values. We propose defining NaN in Presto such that NaN = NaN and sorts as largest, and changing Presto behavior accordingly.

## Background

### Problem

Presto behaves inconsistently with regard to treatment of NaNs for ordering and comparison. We would like to be consistent across Presto, and additionally we would like Presto and Velox to be consistent as we move to native workers.

There are several categories of comparison and ordering of NaNs in Presto:
* Less than and greater than operators (<, >)
* Functions that choose the largest and smallest values
* Functions that sort arrays and maps
* Order bys
* File format stats describing min and max
* File format sorted columns
* Equality and inequality operators (=, <>, IN, NOT IN)
* Equality for the purpose of matching join keys
* Distinct (`SELECT distinct`)
* Group by (is NaN grouped into a single group)
* Distinct aggregations (for example, `count(distinct x)`)
* Map keys (Can a NaN key appear twice? Does `map_column[nan()]` return any value?)
* Functions that remove duplicate values from an array or map

### IEEE-754 Floating point definition

IEEE-754 is the standard for defining floating point arithmetic.  According to that standard any functions involving NaN should return NaN, and <, >, = should all return false.  <> should return true. However, it also defines a total ordering according to which NaN can be sorted.  According to the total ordering, NaN has an absolute value greater than infinity, so  -NaN < -∞ < ∞ < NaN.

### ISO/ANSI SQL Spec

The ANSI SQL spec does not give clear direction regarding NaNs. 

From the SQL:2016 spec:
				
					
> Approximate numeric and decimal floating-point values consist of a mantissa and an exponent. The mantissa is a signed numeric value, and the exponent is a signed integer that specifies the magnitude of the mantissa. Approximate numeric and decimal floating-point values have a precision. The precision of approximate numeric values is a positive integer that specifies the number of significant binary digits in the mantissa. The precision of decimal floating-point values is a positive integer that specifies the number of significant decimal digits in the mantissa. The value of an approximate numeric or decimal floating-point type is the mantissa multiplied by a factor determined by the exponent.				
An <exact numeric literal> ENL consists of either a <period> followed by an <unsigned integer> or an <unsigned integer> followed by an optional <period> and an optional <unsigned integer>. The declared type of ENL is an exact numeric type.				
> An <approximate numeric literal> ANL consists of a <mantissa> that is an <exact numeric literal>, the letter 'E' or 'e', and an <exponent> that is a <signed integer>. It is implementation-defined whether the declared type of ANL is an approximate numeric type or the decimal floating-point type. If M is the value of the <mantissa>					
and E is the value of the <exponent>, then M * 10E is the apparent value of ANL. If the declared type of ANL is an approximate numeric type, then the actual value of ANL is approximately the apparent value of ANL, according to implementation-defined rules. If the declared type of ANL is the decimal floating-point type, the actual value of ANL is either exactly or approximately the apparent value of ANL, according to implementation- defined rules. 
>….							
>Operations on numbers are performed according to the normal rules of arithmetic, within implementation-defined limits, except as provided for in Subclause 6.29, “<numeric value expression>”. 

Some read the above as disallowing NaNs because floating point types are defined only as mantissa and exponent, and there are no special values defined like NaN, infinity, and -infinity.  I don't think it's clear one way or the other, and it looks like most databases do support NaN values.

With regard to sorting, the ANSI spec says the following:							>											
> f)  PVi is said to precede QVi if the value of the <comparison predicate> “PVi <comp op> QVi” is True for the applicable <comp op>.			
>			 							
> g)  If PVi and QVi are not the null value and the result of “PVi <comp op> QVi” is Unknown, then the relative ordering of PVi and QVi is implementation-dependent.

### Current Presto behavior

Here we give a short summary of Presto's current behavior.  There is a complete review here: https://github.com/prestodb/presto/wiki/Presto-NaN-behavior 

#### Comparison 

NaN = NaN returns false.
NaN <> NaN returns true.
Joins on NaN keys do not match.
For maps and arrays, NaNs are considered distinct from each other, and will appear multiple times in a set_agg, union, or even as map keys.
For group bys, distinct, and distinct aggregations, NaNs are not considered distinct from each other. There will only be one group for all NaNs, and they will be deduplicated by a distinct operation.

#### Sorting 

<,>, <=, >= all return false.
`min()`, `max()`, `min_by()`, `max_by()`, `min(x, n)` and `max(x,n)` have inconsistent behavior depending on when NaN is encountered.
`greatest()` and `least()` throw an error on NaN input.
`map_top_n()` returns wrong results if NaN shows up in the input.
Order by sorts NaN as larger than infinity.
Most array and map sorting and topn functions consider NaN as largest.
`array_min()` and `array_max()` return NaN if any input value is NaN.

When it comes to pushing filters into the scan. Tuple domains, I believe would consider NaN largest (I'm not confident about this, and  need to dig a bit more to be certain). 
tupleDomainFilters testDouble() will return false if the value is NaN unless the filter is a NOT IN (following the IEEE definition and Presto's behavior for basic operators).  
Orc files, at least as written by Presto and Velox won't include min/max stats if any of the values are NaN.  
Parquet files written by Presto will exclude min and max stats if there are NaNs in the data.  Those written by velox will have min and max stats, but won't include NaNs in the computation.

### Other databases 

#### PostgreSQL

PostgreSQL uses NaN to represent undefined calculational results. In general, any operation with a NaN input yields a NaN. The only exception is when the operation's other inputs are such that the same output would be obtained if the NaN were to be replaced by any finite or infinite numeric value; then, that output value is used for NaN too. (An example of this principle is that NaN raised to the zero power yields one.)
Note: In most implementations of the “not-a-number” concept, NaN is not considered equal to any other numeric value (including NaN). In order to allow numeric values to be sorted and used in tree-based indexes, PostgreSQL treats NaN values as equal, and greater than all non-NaN values.
https://www.postgresql.org/docs/current/datatype-numeric.html#DATATYPE-NUMERIC-DECIMAL 

#### Spark SQL

There is special handling for not-a-number (NaN) when dealing with float or double types that does not exactly match standard floating point semantics. Specifically:
NaN = NaN returns true.
In aggregations, all NaN values are grouped together.
NaN is treated as a normal value in join keys.
NaN values go last when in ascending order, larger than any other numeric value. (this holds for min/max functions as well, such as array_max)
 https://spark.apache.org/docs/3.0.0-preview/sql-ref-nan-semantics.html

#### Snowflake

In Snowflake, NaN=NaN and is sorted as greater than infinity: https://docs.snowflake.com/en/sql-reference/data-types-numeric#special-values 

#### BigQuery

NaN is considered smaller than -infinity for sorting. 
All NaN values are considered equal for grouping and distinct operations
min/max functions return NaN if NaN is in the input
BigQuery does not have functions that do distincts or sorting on arrays or maps
https://cloud.google.com/bigquery/docs/reference/standard-sql/data-types 
https://cloud.google.com/bigquery/docs/reference/standard-sql/aggregate_functions#max

#### MySQL

Mysql doesn't support NaNs at all (see reference here https://dev.mysql.com/doc/refman/8.0/en/type-conversion.html)

#### Oracle
Oracle orders NaN greatest with respect to all other values, and evaluates NaN as equal to NaN.

https://docs.oracle.com/en/database/oracle/oracle-database/19/sqlrf/Data-Types.html#GUID-33A52FDB-BA5C-474E-96D3-40390BA5F5F4

#### Db2 for z/os and Db2 LUW

DB2 has two kinds of floating point numbers.  The first, DOUBLE or FLOAT, does not support NaN or infinity values.  The second, DECFLOAT, supports infinities as well as quiet and signaling NaNs. It is described as an IEEE-754 number, but does not follow the IEEE spec for comparisons.  The NaNs compare as equal to each other and greater in magnitude than any other number. The ordering among the different special values is as follows: -NAN < -SNAN < -INFINITY < 0 < INFINITY < SNAN <NAN.

https://www.ibm.com/docs/en/db2-for-zos/13?topic=numbers-double-precision-floating-point-double-float
https://www.ibm.com/docs/en/db2-for-zos/13?topic=numbers-decimal-floating-point-decfloat
https://www.ibm.com/docs/en/db2-for-zos/13?topic=comparison-numeric-comparisons 

https://www.ibm.com/docs/en/db2/11.5?topic=list-numbers#r0008469__title__7
https://www.ibm.com/docs/en/db2/11.5?topic=list-numbers#r0008469__title__8
https://www.ibm.com/docs/en/db2/11.5?topic=types-assignments-comparisons#r0008479__numcomp__title__1
## Goals

* Define a consistent behavior for ordering and comparisons of NaNs in Presto
* Make behavior easy to implement and debug
* Make behavior easy for users to understand
* Align behavior with Velox for consistency across Java and C++ workers
* [Nice to have] Align behavior with Spark

## Proposed Implementation

We propose defining ordering and comparisons for NaNs such that `NaN = NaN` and sorts biggest.

With this proposal we stray from the IEEE definition and follow the example of PostgreSQL, Spark, and others.  In this case for all equality purposes such as =, distinctness, grouping, and join keys, we would define NaN=NaN.  We would sort NaN as greater than infinity for all purposes, including for min/max and > and < comparisons.

This was by far the most common behavior across the DBS we looked at, including Spark, which at Meta is the engine we care most about compatibility with.  It is not IEEE-754 compliant, but no other DB we looked at was (either they did not support NaNs or they had at least some features where they strayed from IEEE compliance). Given how erratically we behaved previously, it is safe to say that our users have not been relying on IEEE-754 compliance - if they have, they certainly have not been getting it. 

This definition is consistent across all equality, comparison, and ordering operations meaning that users can always know what to expect. Though it would require some changes in Presto, the implementation should be straightforward and reduces the chances of bugs. Velox has a proposal for the same: https://github.com/facebookincubator/velox/pull/7237.

The disadvantage is that we aren't strictly following the IEEE-754 definitions.

We propose changing the behavior of all of our functions and operators to match this definition (see the spreadsheet linked in the description of Presto's current behavior for the full list). Notably, this will involve changing basic operators (<, >, =, <>) and joins. Another important concern is that Presto may read files and metadata that were written by other systems, and it can write files and metadata that will be read by other systems. When Presto writes stats for max values it should follow the expectations for the service it is writing to, and when it reads max values it should know whether those will adhere to its own NaN ordering definition before using that metadata for its own optimization.

We will also document the new behavior.
 
## Other Approaches Considered

### IEEE-754 and throw an error for order by

In this approach we follow the strict IEEE-754 definition for equality and comparisons. 
NaN = NaN would be false. 
NaN less than or greater than anything would be false
NaN <> NaN would be true.
Any function on NaN input including min/max would return NaN.
Grouping or distinct on NaN keys would return separate groups for each NaN value.
In cases where we could not simply return NaN, we would throw an exception.  That would mean that for order by on a double column containing NaN we would throw an error (as presumably would `array_sort`). 

This has the advantage of adhering to IEEE. It also provides a consistent definition of our operators to reduce chances of bugs (no difference between results you would get from applying < or > vs. what you would get from doing a sort).

A disadvantage is that users are very unlikely to want to get an error for order by just because their data has NaN in it - especially if we otherwise support NaN values. it also still leaves the funny behavior with maps/hashing where you can put things in a map but can't get it out. There are also no other systems that behave this way.

### IEEE-754 with total order

In this option we implement the full IEEE spec including the total order definition to use for sorting and order by.
NaN = NaN would be false.
NaN less than or greater than anything would be false.
NaN <> NaN is true. 
Group bys and distincts would return separate groups for each NaN value.
Order by on a double column containing NaN would give you the IEEE defined total order.  
An open question would be the behavior of functions like min or max (are they functions on NaN input that should return NaN or are they sorting functions that should return based on the total order? what about when it's a topn function like min(x, n) or max(x, n)?).  

Advantages here are full compliance with the IEEE spec.

Disadvantages are the inconsistency between comparison and sorting operators can lead to confusion and bugs. It keeps the funny behavior with maps where you can put NaNs in but can't get them out (and not sure if it would cause implementation problems if we did group bys this way). It does not seem like other databases do this either.

### Disallow NaNs

In this option we would throw an exception if a NaN value would be encountered. We might then want to do the same for  infinity and -infinity as the logic would be the same. 

This option might be in greater compliance with the SQL spec, which does not specify the special constants NaN, infinity, and -infinity). This approach also ensures that the user never misuses NaNs in their data due to not understanding how Presto handles them. 

A big disadvantage here is that Presto reads tables that were written by other systems that do support NaN. We need to be able to read those files and not return an error. We could have some kind of function that does some special handling if you ask it to, but the whole thing would be a bit convoluted. 
s

## Adoption Plan

- Adopting this definition will require changing behavior for many Presto functions. For some the NaN behavior is a bit surprising already and should not need a special flag (you can put as many NaN keys in a map as you want, but can never get them out again).  Others are basic operations like `<, >, =, <>` or joins, and sudden changes in these behaviors could cause problems for some users.
- We will have a temporary configuration/session property `use_new_nan_behavior` for changes to these basic functions to help with the transition. This property will be removed after a couple releases and we will switch fully to the new definition.
- We will document the new behavior.

## Test Plan

We will add a test class TestNanQueries with tests to ensure correct behavior for all functions and operations that handle NaNs. The tests will use queries that execute on workers, rather than constants that get interpreted on the coordinator, so that the test can also be used to ensure consistency between native and Java workers.
