# **RFC-0006 for Presto: Constant Folding via Expression Evaluation to the Presto Sidecar Process**

## [Title]

Proposers

* Tim Meehan

## [Related Issues]

* RFC-0003

## Summary

Constant folding is a technique used in compilers to evaluate constant expressions at compile time. With the advent of Prestissimo, there is now a forked expression evaluation logic, and as of now it is required to duplicate the implementations of all custom C++ functions with identical Java functions. This RFC proposes unify the expression evaluation logic into C++ to address problems created by this fork. This will allow Presto to evaluate constant expressions at query planning time using the same expression evaluation engine used by query execution, in particular for constant folding, which will reduce the amount of work that needs to be done at query execution time.

RFC-0003 proposed a mechanism to add constant folding support via remote function evaluation to address this problem.  This RFC has a similar goal, but represents an improvement over that approach.

## Background

Unfenced functions are functions which execute within the same process as the underlying evaluation engine.  Typically, this term is used in the context of user-defined functions, and it is meant to contrast with fenced functions, which execute in a separate process.  Unfenced functions are more efficient than fenced functions, but they are also more dangerous, as they can crash the process in which they are running.

Unfenced functions may often be implemented to provide an efficient set of functions that are common to a business unit or group.  They allow the users of Presto to efficiently customize Presto into a different flavor specific to their business needs.

A key feature of unfenced functions is that they can be used to implement constant folding.  This is because the function can be evaluated at query planning time, and the result can be used as a constant in the query plan.  This can be a significant performance improvement, as it can reduce the amount of work that needs to be done at query execution time.  For example, if an unfenced function is evaluated over a partition column, the result can be used to prune partitions before the query is executed.

Currently in Presto C++, constant folding will not work during query optimization for functions which are written in C++ and which do not have a Java counterpart.  In practice, constant folding may work for functions which have a corresponding Java implementation, but not for functions which are implemented in C++.  However, even for such functions, due to inconsistencies in the implementation between the C++ and Java functions, there still may be correctness issues.  This is because it works by essentially just using a forked set of functions written in Java to evaluate the constant expressions, and then during execution relying on the C++ functions to evaluate the non-constant expressions.  This has a few drawbacks:

1) It requires a forked set of functions to be written in Java, which is a maintenance burden.
2) An unfenced function written in C++ must have a corresponding Java implementation in order to work properly.
3) Even with an extensive reimplemented set of functions in C++, they often have slightly different semantics, which create hard to fix bugs and create problems for the user experience.

RFC-0003 proposed a mechanism to both add a function registry to allow C++ implemented functions to be exposed to the coordinator without a corresponding Java implementation, and to add constant folding support via remote function evaluation.  This RFC has a similar goal, but represents an improvement over that approach.

In this RFC, we propose to optimize row expressions as a whole, not just function calls.  This has a few advantages:

1) It allows for constant folding to be applied to expressions which are not function calls, such as special form expressions.
2) It can reduce the overhead of constant folding by allowing the evaluation engine to evaluate the expression as a whole, rather than evaluating each function call individually.

### [Optional] Goals

* Allow for a pluggable implementation of expression evaluation and use this in the Presto optimizer for constant folding
* Allow connector authors to use this plugin to optimize their own expressions (for example, for filter pushdown)

### [Optional] Non-goals
* Due to the introduction of the sidecar and overhead associated with it, it is not expected that this implementation will be as efficient as the native Java implementation.  However, for most use cases, even OLAP use cases, this should be an acceptable tradeoff.

## Proposed Implementation

### SPI

A new SPI will be added to Presto to allow for a pluggable implementation of expression evaluation called `RowExpressionInterpreterService`.  A default implementation of this will be added which delegates to the existing expression evaluation code, but this may be overridden by a custom plugin.

`RowExpressionInterpreterService` can be configured to announce what features it has available for constant folding.

### Main

A new class, `DelegatingRowExpressionInterpreter` will be developed.  It will serve the same purpose as the existing [`RowExpressionInterpreter`](https://github.com/prestodb/presto/blob/master/presto-main/src/main/java/com/facebook/presto/sql/planner/RowExpressionInterpreter.java), except it will function differently.

In the first pass, an entire expression tree will be traversed to identify the largest subtrees which are eligible for optimization.  During this pass, it will also resolve any variables references that can be resolved, and create a map from the original expression to the resolved expression.  It will then send the expressions that can be optimized as a list to the `RowExpressionInterpreterService`.  Finally, we map the output from this service to the original un-resolved expressions, and traverse the entire expression tree once again, rewriting all expressions which were simplified with the output from the service.

The `RowExpressionInterpreter` is made available from a new manager, `ExpressionManager`.  The new ExpressionManager will be integrated into the Presto optimizer through a new optimizer rule similar to `RowExpressionOptimizer`, except which uses the new SPI to evaluate the expressions.  There will be a feature toggle which can enable or disable the use of this new SPI, however eventually it will become mandatory to use it.  Plugin authors may also use this new `RowExpressionInterpreter` to simplify expressions as necessary, as it will be injected into the Connector code.

### Plugin

A new plugin will be implemented which will optimize row expressions by serializing them and sending them to the Presto sidecar for evaluation.  This will be a REST call to the local sidecar.

### Sidecar

The Presto sidecar will have a new endpoint which can take in a serialized row expression and return the result of evaluating it.

> Endpoint: /v1/evaluate
>
> HTTP verb: POST
>
> Body: JSON list of serialized row expressions
>
> Response: JSON list of optimized serialized row expressions

#### Note on integration with Velox
Currently Velox only supports constant folding for expressions whose entire arguments are all constant expressions (i.e., constant values).  Constant folding, however, entails many specific scenarios which may contain non-constant values (outlined below).  It will be determined at a later time if these optimizations would go into Velox or into Prestissimo, however it is presumed they must be added in C++ in order to prevent multiple hops between the coordinator and the sidecar.

#### Optimizations required for feature parity with Java constant folding support

##### Special Form Expressions

All expressions must recursively be simplified.

`IF`

* Expression can be constant folded
    * `IF (1=1, column1, column2)` => `column1`

`NULLIF`
* Either argument is null
    * `NULLIF(null, 123)` => `null`
* Both arguments are constants
    * `NULLIF(123.0d, 123.0f) => CAST(123.0 AS DOUBLE) == CAST(123.0 AS DOUBLE) ? NULL : 123.0d`

`ISNULL`
* If the argument can be simplified to a value, check if it’s null
    * `ISNULL(NULL) `=> `TRUE`

`AND`
* Either argument evaluates to false
    * `false && column2` => `false`
    * `column1 && false` => `false`
* Either argument evaluates to true
    * `column1 && true` => `column1`
    * `true && column2` => `column2`
* Both arguments evaluate to null
    * `null && null` => `null`

`OR`
* Either argument evaluates to false
    * `false || column2` => `column2`
    * `column1 || false` => `column1`
* Either argument evaluates to true
    * `column1 || true` => `true`
    * `true || column2` => `true`
* Both arguments evaluate to null
    * `null && null` => `null`

`COALESCE`
* A null constant expression is encountered in the argument list
    * `COALESCE(column1, column2, null, column3)` => `COALESCE(column1, column2, column3)`
* A non-null constant expression is encountered in the argument list
    * `COALESCE(column1, column2, 123, column3)` => `COALESCE(column1, column2, 123)`
    * `COALESCE(123, column1, column2, column3)` => `123`
* Multiple identical expressions are reduced to one
    * `COALESCE(column1, column2, column3, column2, column4)` => `COALESCE(column1, column2, column3, column4)`
    * Counter-example: `COALESCE(column1, NonDeterministicFn(column2), column3, NonDeterministicFn(column2), column4)` => `COALESCE(column1, NonDeterministicFn(column2), column3, NonDeterministicFn(column2), column4) `
        * Note: we must check if the expression that is duplicated is deterministic or not

`IN`
* Null target
    * `null IN (1, 2, 3)` => `null`
* Null in operands list
    * `123 IN (column1, null)` => `null`
* A constant target and an operand list with mixed column referenced and constants
    * `123 IN (456, column1, column2)` => `123 IN (column1, column2)`

`DEREFERENCE`
* A constant is dereferenced
    * `ROW(1, ‘a’, true)[3]` => `true`
    * `ROW(1, ‘a’, true)[2]` =>` ‘a’`
    * `CAST(ROW(1, ‘a’, true) AS ROW(number BIGINT, character VARCHAR, bool BOOLEAN)).number` => 1

`CASE`
* The input resolves to a constant and there is a match in the when clauses
    * 
        ```sql
             CASE 2
                   WHEN 1 THEN 'one'
                   WHEN 2 THEN 'two'
                   WHEN column3 THEN 'three'
                   ELSE 'many'
               END
          ```
=> `two`
* The input resolves to a constant, and the when clauses are all constants, and there is no match to the when clauses
    *
          ```sql
             CASE 3
                    WHEN 1 THEN 'one'
                    WHEN 2 THEN 'two'
                    ELSE 'many'
             END
          ```
=> `many`

##### Call expressions
Certain call expressions have properties which we can simplify.

`LIKE`
* `’a’ LIKE null` => `null`
* `’a’ LIKE ‘%’ ESCAPE null` => `null`

`CAST`
* `CAST(1 AS INTEGER)` => `1`

### Example

As an example, consider the expression `abs(0.02 * price * 0.3) + floor(2/3)`.

![before.png](RFC-0006%2Fbefore.png)

We can simplify this expression by constant folding its constituent subexpressions.  Highlighted below are the expressions identified as being able to be constant folded.

![identified.png](RFC-0006%2Fidentified.png)

These may be sent to the sidecar process, where they will be optimized using the same evaluation engine code used during execution.

![simplified.png](RFC-0006%2Fsimplified.png)


## [Optional] Metrics

This is a 0 to 1 feature, and its success will be largely dictated by two metrics:

1) The performance of evaluating expressions, particularly large, complicated expressions.  This can be measured by comparing the time it takes to evaluate a query with and without the new SPI.
2) The performance of evaluating simple expressions and its impact to query latency.
3) The ability to perform constant folding in C++ clusters with custom functions which are only registered in C++.

## [Optional] Other Approaches Considered

A per-function approach was initially considered, however this is chosen in favor of that because not only is the API cleaner, we also expect that it can scale better.

## Adoption Plan

- This feature will be disabled by default.
- After a period of time, the default will be to enable it.

## Test Plan

End to end tests: we will extend the test cases in `TestExpressionInterpreter` to execute against the Presto sidecar.  Additionally, we will add a functional testing framework that asserts behavior under a variety of conditions, including the sidecar is down (covered in RFC-0003), the sideecar is slow, and errors are returned.
Fuzzer testing: to ensure correct results, we will leverage a fuzzer which will automatically generate expressions and compare the results between two Presto coordinators: one which uses the Presto sidecar to optimize expressions, and one which does it in-memory in Java.
