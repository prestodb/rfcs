# **RFC-0004-jdbc-join-pushdown for Presto**

See [CONTRIBUTING.md](CONTRIBUTING.md) for instructions on creating your RFC and the process surrounding it.

## Jdbc join pushdown in presto

Proposers

* Ajas M M


## [Related Issues]

https://github.com/prestodb/presto/issues/23152

## Summary

When presto identifies a join query which is specific to a Jdbc remote datasource, it split the join query into multiple select query based on the tables involved in the join query and select all the records from each tables using the jdbc connector without passing the join condition. Then the results of these sub-queries are fetched into the presto workers, additional operations such as filters, joins, sorts are applied before sending back to the user.

If we  "Push down" or send these joins which involves on same catalog/remote datasource as part of the  SQL along with join condition to the remote data source it increase the performance 3x to 10x.



## Background



This implementation is to address a performance limitation of Presto federation of SQLs of JDBC connector to remote data sources such as DB2, Postgres, Oracle etc. Currently, in Presto, we have predicate pushdown (WHERE condition pushdown) to some extent in JDBC connectors and not having any join pushdown or join condition pushdown capabilities. 

This cause high performance impact on join queries and it is raised by some of our client. While comparing with competitors we also missing the Jdbc join pushdown capabilities.

We did a poc by changing the presto generated PlanNode to handle jdbc join pushdown and it icreases the performance from 3x on postgres and  8x on db2 remote datasource. Now we need to perform its actual implemetation 



## Proposed Implementation

If presto get a join query which is trying to join tables either from same datasource or from different datasource, it is receiving as a string formated sql query. Using presto parser and analyser, presto validate the syntax and converted to Query (Statement) object. This Query object is converted to presto internal referance architecture called Plan, using its logical and physical optimizers. Finally this plan is executed by the executor. 

Currently for the join query, presto create a final plan which contains seperate TableScanNode for each table that participated on join query and this TableScanNode info is used by the connector to create the select query. On top of this select query result, presto apply join condition and other predicate to provide the final result.

In the proposed implementation, while performing the logical optimization, instead of creating seperate TableScanNode for each table of the JoinNode, it uses a new single TableScanNode for hold all the table details which sattisfied the "JoinPushdown condition" (JoinPushdown condition pointed below) in the join query. This new TableScanNode also hold join criteria for that "JoinPushdown condition" satisfied tables. Using this new TableScanNode it build join query at connect level and return the join result to presto. Now the further predicate which was not pushed down to connector level will apply on the result by the presto and return the final result.

#### The JoinPushdown condition

 1. Table: left table and right table should be on same connector
 2. Join clause: Join condition should be from same connectors table
 3. Pushdown flag: A global setting is there for enable JdbcJoinPushdown
 4. Filter Criteria: The filter criteria should be from same connector


## Highlevel Design

In presto, on Logical planning it converting the Query object  to  Presto Object called PlanNode by using the parser and analyser. In Physical planning this PlanNode passes through different logical and Physical optimiser - Rules, to optimize the PlanNode and to create the final Plan object.

Basically the PlanNode or the Plan is a tree datastructure which represent the sql query, and is able to understand and process by presto. When a join Query received on logical planning, presto create a PlanNode with JoinNode. JoinNode is a tree structure which can hold another node, left tables, right tables, join conditions, projections and filters related to that join query. If there are multiple tables to join then it create a join tree structure where the left side of the JoinNode will be another JoinNode which hold sub joins to resolve multiple table.  The logical PlanNode is created in such a way, where the first table (which is the from clause table) is resolved first either from the left TableScanNode or from JoinNode hierarchy using left dept first algorithm, then its adjacent table (very next table) as right side.  So the order and position of the tables in the join query plays an important role to determine query pushdown. Below is the example of PlanNode that created for the join query.

![PlanNode](https://github.com/Ajas-Mangal/jdbc-join-pushdown/assets/175085180/1c99d1ac-ff1f-4c76-9488-01c8a9b2cbb2)

For doing jdbc join pushdown, we need to create a new logical optimiser called JoinPushdown optimiser which is applicable only for JdbcConnector. In this optimizer the PlanNode tree traversal happen against connector specific JoinNode on left depth first manner. During left depth traversal, if the JoinNode left and right tables are from same connector,  and if join conditions is from same connector and if satisfies all other 'join pushdown condition' then it will create a new TableScanNode by replacing that JoinNode and merging that JoinNode left and right TableScanNode into single one. Then the left depth travesal reach to its parent JoinNode and validate left table, right table, join condition agaisnt the 'join pushdown condition'. If that also matches the connector and join condition then that table also add to previously created ‘single table scan’ and replace the JoinNode.  This JoinNode replacement to ‘single table scan’ will continue until the traversal find any fail case on 'join pushdown condition' or end the JoinNode hierarchy. Onces it fail or complete then it will immediately return the node without traversing on remaining parent node or pushing down the remaining parent nodes. For the JoinNode which is not pushingdown (not converted to new TableScanNode) will follow presto default behaviour to handle the JoinNode.

After JoinPushdown optimizer, the PlanNode ensure only one TableScanNode against the JoinNode if the JoinNode matches 'join pushdown condition'. So there is only one split is generated and not break the join query to multiple select statement for the JoinNode which satisfies 'join pushdown condition'. Now the single new TableScanNode split will transfer to the connector and the connetor will generate the join query from the new TableScanNode split as it contains all the tables and join conditions related to that join query.

![joinpushdown_summary](https://github.com/Ajas-Mangal/jdbc-join-pushdown/assets/175085180/b0b4bddb-5ef9-40fc-a51f-d6c34ba36bdb)

#### Proposed JoinPushdwon working on federated query

This jdbc join pushdown proposal is based on the existing presto flow and it is optimizing the PlanNode using JdbcJoinPushdown optimizer to pushing down the join node to connector level. 

Presto always produces a left deep Join tree for a PlanNode (JoinNode). The JoinNode never met another JoinNode on right, and the right always will be a source for TableScanNode. But the left of the JoinNode could be another JoinNode. 

That is, for a join query the left always resolve from 'FROM' clause table. So if we use a federated join query (which involves multiple connectors) we could pushdown all the tables of the connector which is defined as 'FROM' table. No other connector tables are going to pushing down as the left is always resolve from 'FROM' clause table and the right is the other connector level table. Below is the possible pushdown if we use postgres table as from table.


![PlanNode_optimized](https://github.com/Ajas-Mangal/jdbc-join-pushdown/assets/175085180/77ff4b8c-3d25-4d1c-800c-d8885791407a)


For doing federated join pushdown (pushing down tables from multiple connector) either we need to create a bushy PlanNode by grouping the tables or need to rewrite the PlanNode. It is not planning on this proposal and here we are planning to pushing down all the tables from the single connector which is defined as 'FROM' clause.

## Lowlevel Design

#### Below is the detailed flow for jdbc join pushdown.

![join_pushdown_detailed_design](https://github.com/Ajas-Mangal/jdbc-join-pushdown/assets/175085180/26c4ba30-28aa-47c5-991a-2abebac79ac1)

#### Below are new points that Proposed for Join Pushdown Implementation

* JoinNode make available on ConnectorPlanOptimizer

* On ApplyConnectorOptimization Optimizer, add new ConnectorPlanOptimizer called JdbcJoinPushdown

* JdbcJoinPushdown needs to implement ConnectorPlanOptimizer and the optimze method use ConnectorPlanRewriter to rewrite the JoinNode to a TableScanNode

* Handle JoinNode on JdbcJoinPushdown ConnectorPlanRewriter. 

* visitJoinNode to identify which Join nodes can be pushed down and handled by the connector. Multiple JoinNodes need to handle here.

* If the connector, identifies a JoinNode that it wants to rewrite, collapsing the JoinNode into a new TableScanNode property by removing the JoinNode.

* Handle join query or  parameters for join query on new TableScanNode property may be a new data structure on JdbcTableHandle

* Handle output projection for multiple table (from collapsed Join) using new TableScanNode(JdbcTableHandle)

* Modify Jdbc split to handle the new TableScanNode property

* Create Querybuilder to create join query from the new JdbcSplit property

* Handle where clause associated with JoinNode

* Handle other predicate like Limit, rowcount, nvl, abs(), etc.

##  1) JoinNode make available on ConnectorPlanOptimizer

Currently we are focusing only for Jdbc join pushdown. The join pushdown should be at connector level.  If we use join queries from multiple connector like iceberg, postgres and db2 then, 

  * No join pushdown is expected for iceberg tables,
  * If there are multiple tables to join on postgres then that join should pushed down to connector as a single postgres join query 
  * and if there are multiple tables on db2 connector which satisfies 'join pushdown condition' then that also need to pushed down to db2 
    connector as a single join query. 

Now the presto is responsible for return the final result by joining above iceberg, postgres and db2 connector response.

For implementing connector level join pushdown, first we need JoinNode and make available it for accessing on connector level. For that JoinNode should move to spi package (com.facebook.presto.spi.plan) from sql planner (com.facebook.presto.sql.planner.plan). It includes,

* Moving dependent class to spi package.

        * JoinNode.java

        * AbstarctJoinNode.java

        * SemiJoinNode.java

* Refactor codes that use above class : Refactor the imports 

* Currently visitJoin() is implemented on InternalPlanVistor. This class should not move to spi, instead add visit join method on PlanVistor 
  and extend PlanVistor on JoinNode instead of InternalPlanVistor

* Correct all build fails, test fails and method miss by implementing them for new JoinNode.

* Do sample unit testing and ensure Join query coverage.

## 2) On ApplyConnectorOptimization Optimizer, add new ConnectorPlanOptimizer called 

We  have ApplyConnectorOptimization Optimizer to add connector specific plan optimizer. This Optimizer is created on application initialization and load values while read catalog properties. Existing general flow for optimizer is as follows. 

![ConnectorPlanOptimizer](https://github.com/Ajas-Mangal/jdbc-join-pushdown/assets/175085180/7ff937c4-f3d8-40c1-acb8-80c540749f90)


#### New Changes required as part of this enhancement

#### Add new optimizer JdbcJoinPushdown to JdbcPlanOptimizerProvider

While starting application, it build ‘ApplyConnectorOptimization’ with logical and physical plan optimiser. The optimizer provider for JDBC Connectors for  ‘ApplyConnectorOptimization’ is  JdbcPlanOptimizerProvider. Now JdbcPlanOptimizerProvider provide physical plan optimizer called JdbcComputePushdown. We need to create a new optimizer JdbcJoinPushdown and should implement JdbcPlanOptimizerProvider getLogicalPlanOptimizers method to return this JdbcJoinPushdown optimizer.

   public class JdbcPlanOptimizerProvider
   
           implements ConnectorPlanOptimizerProvider
   
   {
   
       
   
       @Inject
   
       public JdbcPlanOptimizerProvider(…….)
   
       {
   
           ……………
   
           ……………
           
           …………..
     
         }
   
       @Override
   
       public Set<ConnectorPlanOptimizer> getLogicalPlanOptimizers()
   
       {
   
           return ImmutableSet.of(new JdbcJoinPushdown());
   
       }
   
       @Override
   
       public Set<ConnectorPlanOptimizer> getPhysicalPlanOptimizers()
   
       {
   
          …….
   
          …..,..
          
          ……..
   
       }
   
       
   
   }

## 3) JdbcJoinPushdown Rule creation

In presto, when we use a join query, presto create a PlanNode with JoinNode. JoinNode is a tree structure which can hold left and right table details, join conditions, projections and filter details. If there are multiple tables to join then it create a join tree structure where the left side of the JoinNode will be another JoinNode which hold sub joins to resolve multiple table.  The logical PlanNode is created in such a way, where the first table (which is the from clause table) is resolved first either from the left TableScanNode or from JoinNode hierarchy using left dept first algorithm, then its adjacent table (very next table) as right side.  So the order and position of the tables in the join query plays an important role to determine query pushdown. 

![PlanNode](https://github.com/Ajas-Mangal/jdbc-join-pushdown/assets/175085180/aa065f91-1472-4869-8c53-7a39cf43f77c)

In JoinPushdown optimiser, we need to travers the JoinNode on left depth first manner. During traversal, if the left and right tables are from same connector and its join conditions are  from the same connector then it satisfies the  “join pushdown condition” and needs to create a ‘single table scan’ (join query builder) and replace JoinNode with ‘single table scan’ if the global pushdown flag is enabled. Then need to check with its parent JoinNode right table to match above “join pushdown condition”. If that also matches the connector and join condition then that table also add to previously created ‘single table scan’ and replace the JoinNode.  

#### The JoinPushdown condition

 * Table: left table and right table should be on same connector
 * Join clause: Join condition should be from same connectors table
 * Pushdown flag: A global setting is there for enable JdbcJoinPushdown
 * Filter Criteria: The filter criteria should from same connector

.

This JoinNode replacement to ‘single table scan’ will continue until the traversal find any of the below condition and immediately return the node without travers or pushing down the parent nodes. 

a) left and right tables are from different catalog

b) join condition are from different catalog

c) Filter condition is not from same catalog


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
