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

How do you intend to implement the feature? This section can be as detailed as possible with large subsections of its own, or may be a few sentences depending on the scope of the feature proposed. Explain the design in enough detail for existing users/contributors to understand. Design should include all the corner cases you can think of. Feel free to include any new SPI method signatures, class hierarchies or system contracts here. It is recommended to mention any methods, variables, classes, or SQL language additions which you think are needed to provide a broader view of the code changes. Please mention/describe the below on a high level -

1. What modules are involved
2. Any new terminologies/concepts/SQL language additions
3. Method/class/interface contracts which you deem fit for implementation.
4. Code flow using bullet points or pseudo code as applicable
5. Any new user facing metrics that can be shown on CLI or UI.

If presto get a join query which is trying to join tables from same datasource or from different datasource, it is receiving as a string for mated sql query. Using presto parser and analyser, presto validate the syntax and converted to Query (Statement) object. This Query object is converted to presto internal referance architecture called Plan, using its logical and physical optimizers. Finally this plan is executed by the executor. 

Currently for the join query, presto create a final plan which contains seperate TableScanNode for each table that participated on join query and this TableScanNode info is used by the connector to create the select query. On top of this select query result, presto apply join condition and other predicate to provide the final result.

In the proposed implementation, while performing the logical optimization, instead of creating seperate TableScanNode for each table, it uses a new single TableScanNode for hold all the table details which sattisfied the "JoinPushdown condition" (explaine below) in the join query. This new TableScanNode also hold join criteria for that "JoinPushdown condition" satisfied tables. Using this new TableScanNode it build join query at connect level and return the join result to presto. Now the further predicate which was not pushed down to connector level will apply on the result by the presto and return final result.

#### The JoinPushdown condition

 1. Table: left table and right table should be on same connector
 2. Join clause: Join condition should be from same connectors table
 3. Pushdown flag: A global setting is there for enable JdbcJoinPushdown
 4. Filter Criteria: The filter criteria should be able to pushing down to Jdbc, if there is a filter on JoinQuery.
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
