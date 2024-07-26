# **RFC007 for Presto**

## Grammar for Plan Constraints

Proposers

* @aaneja
* @ClarenceThreepwood

## Related Issues

* [Overview doc](https://prestodb.io/wp-content/uploads/Search-Space-Improvements-Plan-Constraints.pdf) on Plan Constraints as a tool to control search space
* PrestoDB blog on - [Elevating Presto Query Optimization](https://prestodb.io/blog/2024/03/21/elevating-presto-query-optimization/)

## Summary

This document proposes a grammar for specifying plan constraints

## Background

Plan constraints can be used to lock down critical aspects of an execution plan, such as access method, join method, and join order. 


## Proposed Implementation

We propose a grammar for specifying independent plan constraints, which take the form of a SQL comment block that would build an object graph of the constraints. Multiple constraints can be specified in a single place. The grammar is open for extension as we develop more mechanisms to lock down plans.

In this first cut, users can build constraints to control 
- Join orders and distributions for INNER JOIN's
- Cardinality (row counts) for base relations and join sub-plans

### Grammar

```
planConstraintString : /*! planConstraint [, ...] */

planConstraint : joinConstraint
| cardinalityConstraint

joinConstraint : joinType (joinNode) [distributionType]

cardinalityConstraint : CARD (joinNode cardinality)

distributionType : [P]
   | [R]

joinType : JOIN	(defaults to inner join)
| IJ
| LOJ
| ROJ

cardinality : integer constant (positive)

joinNode : (relationName relationName [, ...])

| joinConstraint

| (relationName joinNode)

| (joinNode relationName)

| (joinNode joinNode [, ...])
```


### Examples of constraints

1. Inner Join constraints - 
    1. `join (a (b c))` - Join the relations a,b,c as a right deep tree (denoted by the brackets). Use regular rules for determining join distribution
    2. `join (((a c) [R] b) [P])` - In addition to the join order, use a  REPLICATED `[R]`  join for sub-plan `(a c)`  and PARTITIONED `[P]`  for `(a c) b`
    3. If an inner join condition does not exist between nodes a CrossJoin is automatically inferred
2. Cardinality constraints -
    1. `card (c 10)` - Set the output row count estimate of `c` to `10` 
    2. `card ((c o) 10)` - When considering a join node of shape `(c o)` set the output row count estimate to `10`

### Other points of note
- Relation names loosely resolve to WITH query aliases (CTE definitions) and table names.A detailed description of the name resolution is out of scope of this RFC (this will be covered in the implementation PR description)
