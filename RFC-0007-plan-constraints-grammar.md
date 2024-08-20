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

### Grammar (ANTLR 4)

```
grammar PlanConstraints;

// Lexer rules
WS         : [ \t\r\n]+ -> skip ;
NUMBER     : [0-9]+ ;
LOJ        : 'LOJ' ;
ROJ        : 'ROJ' ;
IJ         : 'IJ' ;
LPAREN     : '(' ;
RPAREN     : ')' ;
JOIN_DIST_PARTITIONED : '[P]' ;
JOIN_DIST_REPLICATED : '[R]' ;
CARD : 'CARD' ;
JOIN : 'JOIN' ;
CONSTRAINTS_START_MARKER :'/*!';
CONSTRAINTS_END_MARKER :'*/';

fragment DIGIT
    : [0-9]
    ;

fragment LETTER
    : [A-Z]
    | [a-z]
    ;

IDENTIFIER
    : (LETTER | '_') (LETTER | DIGIT | '_' | '@' | ':')*
    ;

identifier
    : IDENTIFIER
    ;

standAloneRelation
    : identifier
    ;

groupedRelation
    : LPAREN joinedRelation RPAREN
    ;

joinTypeOrDefault
    : joinType
    | { "IJ" } // Default value
    ;

joinType
    : LOJ
    | ROJ
    | IJ
    ;

joinAttribute
    : JOIN_DIST_PARTITIONED
    | JOIN_DIST_REPLICATED
    ;

joinedRelation
    : standAloneRelation joinTypeOrDefault standAloneRelation (joinAttribute)? #ss
    | standAloneRelation joinTypeOrDefault groupedRelation (joinAttribute)? #sg
    | groupedRelation joinTypeOrDefault standAloneRelation (joinAttribute)? #gs
    | groupedRelation joinTypeOrDefault groupedRelation (joinAttribute)? #gg
    ;


cardinalityConstraint
    : CARD LPAREN joinedRelation NUMBER RPAREN
    | CARD LPAREN standAloneRelation NUMBER RPAREN
    ;

joinConstraint : JOIN LPAREN joinedRelation RPAREN;

planConstraint
    : joinConstraint
    | cardinalityConstraint;

planConstraintString : CONSTRAINTS_START_MARKER (planConstraint)* CONSTRAINTS_END_MARKER;


```


### Examples of constraints

1. Inner Join constraints - 
    1. `join (a (b c))` - Join the relations a,b,c as a right deep tree (denoted by the brackets). Use regular rules for determining join distribution
    2. `join ((a c [R]) b [P])` - In addition to the join order, use a  REPLICATED `[R]`  join for sub-plan `(a c)`  and PARTITIONED `[P]`  for `(a c) b`
    3. If an inner join condition does not exist between nodes a CrossJoin is automatically inferred
2. Cardinality constraints -
    1. `card (c 10)` - Set the output row count estimate of `c` to `10` 
    2. `card (c o 10)` - When/If considering a join node of shape `(c o)` set the output row count estimate to `10`
3. Both type of constraints -
    `join (c o) card (c o 10)` - Force a join-sub-graph of nodes `c InnerJoin o`. For this join node, set the output row count estimate to `10`

#### Full SQL examples of queries with constraints

Force the join of `c` and `cte` which is otherwise ignored, see [19354](https://github.com/prestodb/presto/issues/19354) 
```
/*! join ((c cte) n) */
-- 
with cte as (
    select min(orderkey) as min
    from orders
)
select count(*)
from customer c,
    nation n,
    cte
where c.custkey = cte.min
    and n.nationkey = c.nationkey
```

Force the inner join of `l` and `o`, which is otherwise ignored, see [19894](https://github.com/prestodb/presto/issues/19894) 
```
/*! join (s (l o)) */
select 1
from supplier s,
    lineitem l,
    orders o
where l.orderkey = o.orderkey
```


### Other points of note
- Relation names loosely resolve to WITH query aliases (CTE definitions), table names and aliases. A detailed description of the name resolution is out of scope of this RFC (this will be covered in the description of the implementation PR)
