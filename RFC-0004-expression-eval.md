# **RFC0 for Presto**

## [Title]

Proposers

* Tim Meehan

## [Related Issues]

* RFC-0003

## Summary

Constant folding is a technique used in compilers to evaluate constant expressions at compile time. This RFC proposes to add constant folding to Presto's expression evaluation engine. This will allow Presto to evaluate constant expressions at query planning time, reducing the amount of work that needs to be done at query execution time.

RFC-0003 proposed a mechanism to add constant folding support via remote function evaluation.  This RFC has a similar goal, but represents an improvement over that approach.

## Background

Unfenced functions are functions which execute within the same process as the underlying evaluation engine.  Typically, this term is used in the context of user-defined functions, and it is mean to contrast with fenced functions, which execute in a separate process.  Unfenced functions are more efficient than fenced functions, but they are also more dangerous, as they can crash the process in which they are running.

Unfenced functions may often be implemented to provide an efficient set of functions that are common to a business unit or group.  They allow the users of Presto to efficiently customize Presto into a different flavor specific to their business needs.

A key feature of unfenced functions is that they can be used to implement constant folding.  This is because the function can be evaluated at query planning time, and the result can be used as a constant in the query plan.  This can be a significant performance improvement, as it can reduce the amount of work that needs to be done at query execution time.  For example, if an unfenced function is evaluated over a partition column, the result can be used to prune partitions before the query is executed.

Currently in Presto C++, there is no constant folding support for functions which are written in C++.  In practice, constant folding works for functions which have a corresponding Java implementation, but not for functions which are implemented in C++.  It works by essentially just using a forked set of functions written in Java to evaluate the constant expressions, and then during execution relying on the C++ functions to evaluate the non-constant expressions.  This has a few drawbacks:

1) It requires a forked set of functions to be written in Java, which is a maintenance burden.
2) An unfenced function written in C++ must have a corresponding Java implementation in order to work properly.

RFC-0003 proposed a mechanism to both add a function registry to allow C++ implemented functions to be exposed to the coordinator without a corresponding Java implementation, and to add constant folding support via remote function evaluation.  This RFC has a similar goal, but represents an improvement over that approach.

In this RFC, we propose to optimize row expressions as a whole, not just function calls.  This has a few advantages:

1) It allows for constant folding to be applied to expressions which are not function calls, such as special form expressions.
2) It can reduce the overhead of constant folding by allowing the evaluation engine to evaluate the expression as a whole, rather than evaluating each function call individually.

### [Optional] Goals

* Allow for a pluggable implementation of expression evaluation and use this in the Presto optimizer for constant folding
* Allow connector authors to use this plugin to optimize their own expressions (for example, for filter pushdown)

### [Optional] Non-goals

## Proposed Implementation

A new SPI will be added to Presto to allow for a pluggable implementation of expression evaluation.  A default implementation of this will be added which delegates to the existing expresion evaluation code, but this may be overridden by a custom plugin.

A new plugin will be implemented which will optimize row expressions by serializing them and sending them to the Presto sidecar for evaluation.  The Presto sidecar will have a new endpoint which can take in a serialized row expression and return the result of evaluating it.

> Endpoint: /v1/evaluate
> 
> HTTP verb: POST
> 
>Body: JSON list of serialized row expressions
>
> Response: JSON list of optimized serialized row expressions
 
The new plugin will be integrated into the Presto optimizer through a new optimizer rule similar to `RowExpressionOptimizer`, except which uses the new SPI to evaluate the expressions.  There will be a feature toggle which can enable or disable the use of this new SPI, however eventually it will become mandatory to use it.

## [Optional] Metrics

This is a 0 to 1 feature, and its success will be largely dictated by two metrics:

1) The performance of evaluating expressions, particularly large, complicated expressions.  This can be measured by comparing the time it takes to evaluate a query with and without the new SPI.
2) The ability to perform constant folding in C++ clusters with custom functions which are only registered in C++.

## [Optional] Other Approaches Considered

A per-function approach was initially considered, however this is chosen in favor of that because not only is the API cleaner, we also expect that it can scale better.

## Adoption Plan

- This feature will be disabled by default.
- After a period of time, the default will be to enable it.

## Test Plan

In addition to using the existing test coverage for expression evaluation to catch bugs in the expression optimizer, we will also use a fuzzer to generate random expressions and evaluate them with and without the new SPI to ensure that the results are the same.