# **RFC0009-jdbc-join-push-down for Presto**

## Jdbc join pushdown in presto

Proposers

* Ajas M M
* Haritha K
* Thanzeel Hassan
* Glerin Pinhero


## Related Issues

https://github.com/prestodb/presto/issues/23152

## Summary

At present, when a query joins multiple tables, it creates a separate TableScanNode for each table. Each TableScanNode select all the records from that table. The join operation is then executed in-memory in Presto using a JOIN node by applying JoinCriteria, FilterPredicate and other criteria (like order by, limit, etc.).

However, if the query joins tables from the same JDBC datasource, it would be more efficient to let the datasource handle the join instead of creating a separate TableScanNode for each table and joining them in Presto. If we "Push down" or send these joins to remote JDBC datasource it increases the query performance. i.e., decreases the query execution time. We have seen improvements from 3x to 10x.

For example, for the below postgres join query if we push down the join to a single TableScanNode, then the Presto Plan and performance will be as follows :

**Join Query**

```
SELECT order_id,
       c_customer_id
FROM postgresql.public.orders o
    INNER JOIN postgresql.public.customer c ON c.c_customer_id = o.customer_id;
```



**Original presto plan**

```

 - Output[PlanNodeId 9][order_id, c_customer_id] => [order_id:integer, c_customer_id:char(16)]
    - RemoteStreamingExchange[PlanNodeId 266][GATHER] => [order_id:integer, c_customer_id:char(16)]
        - InnerJoin[PlanNodeId 4][("customer_id" = "c_customer_id")][$hashvalue, $hashvalue_11] => [order_id:integer, c_customer_id:char(16)]
                Distribution: PARTITIONED
            - RemoteStreamingExchange[PlanNodeId 264][REPARTITION][$hashvalue] => [customer_id:char(16), order_id:integer, $hashvalue:bigint]
                    Estimates: {source: CostBasedSourceInfo, rows: ? (?), cpu: ?, memory: 0.00, network: ?}
                - ScanProject[PlanNodeId 0,326][table = TableHandle {connectorId='postgresql', connectorHandle='postgresql:public.orders:null:public:orders', layout='Optional[{domains=ALL, additionalPredicate={}}]'}, projectLocality = LOCAL] => [customer_id:char(16), order_id:integer, $hashvalue_10:bigint]
                        Estimates: {source: CostBasedSourceInfo, rows: ? (?), cpu: ?, memory: 0.00, network: 0.00}/{source: CostBasedSourceInfo, rows: ? (?), cpu: ?, memory: 0.00, network: 0.00}
                        $hashvalue_10 := combine_hash(BIGINT'0', COALESCE($operator$hash_code(customer_id), BIGINT'0')) (1:45)
                        LAYOUT: {domains=ALL, additionalPredicate={}}
                        order_id := JdbcColumnHandle{connectorId=postgresql, columnName=order_id, jdbcTypeHandle=JdbcTypeHandle{jdbcType=4, jdbcTypeName=int4, columnSize=10, decimalDigits=0, arrayDimensions=null}, columnType=integer, nullable=true, comment=Optional.empty} (1:45)
                        customer_id := JdbcColumnHandle{connectorId=postgresql, columnName=customer_id, jdbcTypeHandle=JdbcTypeHandle{jdbcType=1, jdbcTypeName=bpchar, columnSize=16, decimalDigits=0, arrayDimensions=null}, columnType=char(16), nullable=true, comment=Optional.empty} (1:45)
            - LocalExchange[PlanNodeId 297][HASH][$hashvalue_11] (c_customer_id) => [c_customer_id:char(16), $hashvalue_11:bigint]
                    Estimates: {source: CostBasedSourceInfo, rows: ? (?), cpu: ?, memory: 0.00, network: ?}
                - RemoteStreamingExchange[PlanNodeId 265][REPARTITION][$hashvalue_12] => [c_customer_id:char(16), $hashvalue_12:bigint]
                        Estimates: {source: CostBasedSourceInfo, rows: ? (?), cpu: ?, memory: 0.00, network: ?}
                    - ScanProject[PlanNodeId 1,327][table = TableHandle {connectorId='postgresql', connectorHandle='postgresql:public.customer:null:public:customer', layout='Optional[{domains=ALL, additionalPredicate={}}]'}, projectLocality = LOCAL] => [c_customer_id:char(16), $hashvalue_13:bigint]
                            Estimates: {source: CostBasedSourceInfo, rows: ? (?), cpu: ?, memory: 0.00, network: 0.00}/{source: CostBasedSourceInfo, rows: ? (?), cpu: ?, memory: 0.00, network: 0.00}
                            $hashvalue_13 := combine_hash(BIGINT'0', COALESCE($operator$hash_code(c_customer_id), BIGINT'0')) (2:12)
                            LAYOUT: {domains=ALL, additionalPredicate={}}
                            c_customer_id := JdbcColumnHandle{connectorId=postgresql, columnName=c_customer_id, jdbcTypeHandle=JdbcTypeHandle{jdbcType=1, jdbcTypeName=bpchar, columnSize=16, decimalDigits=0, arrayDimensions=null}, columnType=char(16), nullable=true, comment=Optional.empty} (2:12)

```

**Joinpushdown presto plan**

```
 - Output[PlanNodeId 9][order_id, c_customer_id] => [order_id:integer, c_customer_id:char(16)]
        Estimates: {source: CostBasedSourceInfo, rows: ? (?), cpu: ?, memory: 0.00, network: ?}
    - RemoteStreamingExchange[PlanNodeId 233][GATHER] => [order_id:integer, c_customer_id:char(16)]
            Estimates: {source: CostBasedSourceInfo, rows: ? (?), cpu: ?, memory: 0.00, network: ?}
        - TableScan[PlanNodeId 217][TableHandle {connectorId='postgresql', connectorHandle='postgresql:public.orders:null:public:orders', layout='Optional[{domains=ALL, additionalPredicate={}}]'}] => [order_id:integer, c_customer_id:char(16)]
                Estimates: {source: CostBasedSourceInfo, rows: ? (?), cpu: ?, memory: 0.00, network: 0.00}
                LAYOUT: {domains=ALL, additionalPredicate={}}
                order_id := JdbcColumnHandle{connectorId=postgresql, columnName=order_id, jdbcTypeHandle=JdbcTypeHandle{jdbcType=4, jdbcTypeName=int4, columnSize=10, decimalDigits=0, arrayDimensions=null}, columnType=integer, nullable=true, comment=Optional.empty} (1:45)
                c_customer_id := JdbcColumnHandle{connectorId=postgresql, columnName=c_customer_id, jdbcTypeHandle=JdbcTypeHandle{jdbcType=1, jdbcTypeName=bpchar, columnSize=16, decimalDigits=0, arrayDimensions=null}, columnType=char(16), nullable=true, comment=Optional.empty} (2:12)
``` 

**Original presto plan performance**

![Original presto plan performance](RFC-0009-jdbc-join-push-down/1_perf_wxd.png)

**Joinpushdown presto plan performance**

![Joinpushdown presto plan performance](RFC-0009-jdbc-join-push-down/2_perf_joinwxd.png)  


## Background

This implementation is to address a performance limitation of Presto federation of SQLs of JDBC connectors to remote data sources such as DB2, Postgres, Oracle etc. Currently, Presto support predicate pushdown (WHERE condition pushdown) to some extent in JDBC connectors, but it does not have any join pushdown capabilities. This causes high performance impact on join queries and it is raised by some of our clients.

## Proposed Implementation

At present, if presto get a join query (from the CLI or UI) which is trying to join tables either from same datasource or from different datasource, it is received as a string formatted sql query. Presto validates the syntax and converts it to Query (Statement) object using presto parser and analyzer. This Query object is converted to presto internal reference architecture called Plan, using its logical and physical optimizers. Finally, this plan is executed by the executor.

![Joinpushdown presto plan performance](RFC-0009-jdbc-join-push-down/basic_wrk.png)  

Basically the PlanNode or the Plan is a tree datastructure which represents the sql query. When a join Query is received in logical planning, presto creates a PlanNode with a JoinNode. JoinNode is a tree structure which can hold another node, left table, right table, join conditions, projections and filters related to that join query. If there are multiple tables to join then it create a join tree structure where the left side of the JoinNode will be another JoinNode which hold sub joins to resolve multiple tables. The logical PlanNode is created in such a way, where the first table (the first table in the from clause) is resolved first either from the left TableScanNode or from JoinNode hierarchy using left dept first algorithm, then its adjacent table (very next table) as right side.  So the order and position of the tables in the join query plays an important role to determine query pushdown. Below is the example of PlanNode that is created for the join query.

 ![PlanNode](RFC-0009-jdbc-join-push-down/Existing_prestoPlan.png) 

Currently while executing a JoinNode, presto creates separate TableScanNodes for each table that is participating in the join query. This TableScanNode info is used by the connector to create the select query for that table. On top of this select query result, presto apply join condition and other predicates to provide the final result.

![Joinpushdown presto plan performance](RFC-0009-jdbc-join-push-down/cur_join_wrks.png) 

In the proposed implementation, all tables from the join query are grouped based on the Jdbc connector (data source). A single TableScanNode is created for each jdbc connector by using the grouped table info, wherever it is possible. It ensures a single TableScanNode against a connector rather than against each table of a join query. 

**For example consider below join query**

``` 
select  *

from postgresql.pg.mypg_table1 t1

join postgresql.pg.mypg_table2 t2 on t1.pgfirsttablecolumn = t2.pgsecondtablecolumn

Join db2.db2.mydb2_table1 t3 on t3.dbthirdtablecolumn = t2.pgsecondtablecolumn

JOIN db2.db2.mydb2_table2 t4 ON t3.dbthirdtablecolumn = t4.dbfourthtablecolumn

JOIN db2.db2.mydb2_table3 t5 ON t4.dbfourthtablecolumn = t5.dbfifthtablecolumn
``` 

**Here we have five tables,** 

mypg_table1 and  mypg_table2 from postgresql connector

mydb2_table1, mydb2_table2 and mydb2_table3 from db2 connector


At present, presto creates five select statement (TableScanNodes) for this query as follows

| No | Node Description                     | SQL Query                                                         |
|----|--------------------------------------|-------------------------------------------------------------------|
| 1  | TableScanNode for mypg_table1        | `select t * from postgresql.pg.mypg_table1 t1`                    |
| 2  | TableScanNode for mypg_table2        | `select t * from postgresql.pg.mypg_table2 t2`                    |
| 3  | TableScanNode for mydb2_table1       | `select t * from db2.db2.mydb2_table1 t3`                         |
| 4  | TableScanNode for mydb2_table2       | `select t * from db2.db2.mydb2_table2 t4`                         |
| 5  | TableScanNode for mydb2_table3       | `select t * from db2.db2.mydb2_table3 t5`                         |



In our proposed implementation, we restrict select statement (TableScanNode) creation based on tables and instead create select statement (TableScanNode) against each connector by grouping the tables based on the connector

| No | Node Description                                  | SQL Query                                                                                                  |
|----|---------------------------------------------------|------------------------------------------------------------------------------------------------------------|
| 1  | TableScanNode [mypg_table1, mypg_table2] for PostgreSQL | `select * from postgresql.pg.mypg_table1 t1, postgresql.pg.mypg_table2 t2 where t1.pgfirsttablecolumn=t2.pgsecondtablecolumn` |
| 2  | TableScanNode [mydb2_table1, mydb2_table2, mydb2_table3] for DB2 | `select * from db2.db2.mydb2_table1 t3, db2.db2.mydb2_table2 t4, db2.db2.mydb2_table5 t5 where t3.dbthirdtablecolumn = t4.dbfourthtablecolumn and t4.dbfourthtablecolumn = t5.dbfifthtablecolumn` |


For performing this jdbc join pushdown,  we need to create two logical optimizers GroupInnerJoinsByConnector and JdbcJoinPushdown. 

GroupInnerJoinsByConnector optimizer is a PlanOptimizer which is responsible for flattening the JoinNode and adding the sundered nodes to a data structure called MultiJoinNode. 

GroupInnerJoinsByConnector optimizer will work on MultiJoinNode and will group TableScanNodes based on connector name if the connector supports join pushdown. This optimizer will create a single TableScanNode by using a new data structure called ConnectorTableHandleSet from the grouped TableScanNode. ConnectorTableHandleSet is a set of ConnectorTableHandles which is generated from grouped TableScanNode. This optimizer also creates a combined overall predicate and overall assignments for the ConnectorTableHandleSet and will add these to the newly created TableScanNode. This newly created TableScanNode structure will replace the source list of MultiJoinNode.
GroupInnerJoinsByConnector optimizer will then work on re-creating join node with updated MultiJoinNode structure. The low level design is available [here](https://github.com/Thanzeel-Hassan-IBM/rfcs/blob/main/RFC-0009-jdbc-join-push-down.md#low-level-design)

![After GroupInnerJoinsByConnector Optimizer](RFC-0009-jdbc-join-push-down/after_Group_opt.png)  

JdbcJoinPushdown optimizer is a ConnectorPlanOptimizer, specific to jdbc tables and it generate a single JdbcTableHandle from the grouped ConnectorTableHandle. The low level design is available [here](https://github.com/Thanzeel-Hassan-IBM/rfcs/blob/main/RFC-0009-jdbc-join-push-down.md#low-level-design)

After GroupInnerJoinsByConnector optimizer and JdbcJoinPushdown optimizer, we will invoke existing Predicatepushdown optimizer. PredicatePushdown optimizer will pushdown the filter and join criteria to the re-created JoinNode using the overall predicate and overall assignment. 

[Predicatepushdown optimizer](RFC-0009-jdbc-join-push-down/after_predicate.png)

After Predicatepushdown optimizer the flow will invoke existing JdbcComputePushdown optimizer and it will pushdown the overall join criteria to the additional predicates.

After all optimization the PlanNode will pass to the presto-base-jdbc module to create the final join query. The final join query is prepared at the connector level using the Querybuilder. It is explained in the low level design [here](https://github.com/Thanzeel-Hassan-IBM/rfcs/blob/main/RFC-0009-jdbc-join-push-down.md#low-level-design).

## Join query pushdown in presto Jdbc datasource

Presto validate Join operation (PlanNode) specifications to perform join pushdown. The specifics for the supported pushdown of table joins varies for each data source, and therefore for each connector. However, there are some generic conditions that must be met in order for a join to be pushed down in jdbc connector

1) The Jdbc connector should be able to process the Join operation.

Presto Jdbc connector will process almost every Join operation except presto functions and operators. 

When we use some aggregate, math operations or datatype conversion along with join query it is converted to presto functions and applied to Join operation. Any join query which creates intermediate presto functions, cannot be handled by the connector and hence will not be pushed down.


| No | Condition which create presto function                   | SQL Query                                    |
|----|-------------------------------------|-------------------------------------------------------------------|
| 1  | abs(int_clumn) = int_cilumn2        | `Select * from table a join table b on abs(a.col1) = b.col2;`                            
| 2  | int_sum_column = int_value1_column1 + int_value1_column2       | `Select * from table a join table b on a.col1 = b.col2 + b.col3;` 
| 3  | cast(varchar_20_column, varchar(100)) = varchar100_column       | `Select * from table a join table b on cast(a.varchar_20_column, varchar(100)) = b.varchar100_column;` 


2) Join operation should be an inner join and should have at least one column to join with another table.

Note: Other optimizers in Presto may change the Join operation. We can call this as Inference. Sometimes presto will change a Pushdown capable Inner join to another Join operation incapable of pushdown (Eg: Infering to remove join condition/predicate in the plan). This will lead to pushdown capability being removed. And sometimes presto will change Join operation to a pushdown capable one. (Eg: Infering to create Inner join from Right/Left join)

Examples to explain presto change an inner join to another Join operation : 

Suppose we have a query like this: 

`Select * from table a join table b on a.col1 = b.col2 and a.col1 = 5;`

Presto will change this from an inner join to two different select statements like this: 

`Select * from table a where a.col1 = 5;`

`Select * from table b where b.col2 = 5;`

Then it does a cross join with these two results. We will not do pushdown in this case.

3) Join criteria (joining column) should be of Datatypes and operators that support join pushdown. 

| No | DataType support join pushdown                   | Operations                                           |
|----|-------------------------------------|-------------------------------------------------------------------|
| 1  | <datatype>        | `=, <, >, <=, >=, !=, <>`                    |
| 2  | <datatype2>       | `=, <, >, <=, >=, !=, <>`  
| 3  | <datatype3>       | `=, <, >, <=, >=, !=, <>` 


4) All tables from same connector will be grouped based on above specifications and pushed down to underlying datasource. 

5) Enable presto Join pushdown capabilities by setting the session flag enable-join-query-pushdown = true.

## Low level Design



For performing Jdbc JoinPushdown we required below implementations :

1. Create new PlanOptimizer called GroupInnerJoinsByConnector for JdbcJoinPushdown

2. Load GroupInnerJoinsByConnector optimizer based on session flag

3. Create a plan rewriter for GroupInnerJoinsByConnector by implementing SimplePlanRewriter

4. Flatten all TableScanNode, filter, outputVariables and assignment to a new data structure called MultiJoinNode

5. Use MultiJoinNode to group Jdbc Tables based on connector name 

6. Build join relation for the grouped tables from all the join predicates

7. Create Single TableScanNode for grouped tables and add as MultiJoinNode source list

8. Recreate left deep join node from the MultiJoinNode source list

9. Build overall filter for the newly created join node

10. Enable JdbcJoinPushdown at connector level 

11. Pushdown the overall filter to the newly created TableScanNode.

12. Create JoinQuery based on JdbcConnector


## [Optional] Metrics

How can we measure the impact of this feature?

## [Optional] Other Approaches Considered

Based on the discussion, this may need to be updated with feedback from reviewers.

## Adoption Plan

- What impact (if any) will there be on existing users? Are there any new session parameters, configurations, SPI updates, client API updates, or SQL grammar?
- If we are changing behaviour how will we phase out the older behaviour?
- If we need special migration tools, describe them here.
- When will we remove the existing behaviour, if applicable.
- How should this feature be taught to new and existing users? Basically mention if documentation changes/new blog are needed?
- What related issues do you consider out of scope for this RFC that could be addressed in the future independently of the solution that comes out of this RFC?

## Test Plan

How do we ensure the feature works as expected? Mention if any functional tests/integration tests are needed. Special mention for product-test changes. If any PoC has been done already, please mention the relevant test results here that you think will bolster your case of getting this RFC approved.
