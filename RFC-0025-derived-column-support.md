# RFC-0025 Derived column

## Proposers

* Prashant Sharma
* Tim Meehan

## Related Issues

* https://github.com/prestodb/presto/issues/27436

## Summary

Derived column is an existing feature for traditional database systems [1], it is a new concept for modern distributed database system. This RFC will cover what it takes
to bring derived columns support to presto and the benefits.

**What is a derived column?**

A column created by applying a SQL expression or a UDF to one or more existing columns in a table.

**Why do we need that, since we can always apply a UDF to a column during project, filter or join?**

Indeed, a derived column consumes O(N) storage, where N is the number of rows in the table. We still need them because, the performance benefits outweigh the disadvantage
of extra storage it consumes. Let us understand with the following use case example:

A compute engine like Presto can easily push down a filter predicate e.g. `SELECT col1, col2, FROM table T1 WHERE col1='constant_value'` , this allows for pruning the number of
rows required for TableScan by applying the filtering WHERE col1=’constant_value’. This is not true of when a SQL expression is involved in the filter predicate, let us take an
example `SELECT col1, col2, FROM table T1 WHERE lower(col1)='constant_value'`. While optimizers can easily push down the filter predicate, however, it can not be used
in filtering using the lower and upper bound metrics, for example Iceberg manifest statics and Parquet row group statistics. As a result, we end up scanning a large number of rows.

So, to support push down of certain predicates (non-monotonic) and reduce the amount of data scanned, derived column bring massive performance improvements.
Derived columns have already been proven in RDBMS system e.g. DB2 [1], and now we intend to bring them to Presto.

### Problem

Let us take the following example:

```
presto:test> explain select *from taxis where lower(store_and_fwd_flag) != lower('N');
                                                                                                                                                                                 Query Plan                                                                                                                                                                                 
----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
 - Output[PlanNodeId 6][vendor_id, trip_id, trip_distance, fare_amount, store_and_fwd_flag] => [vendor_id:bigint, trip_id:bigint, trip_distance:decimal(5,2), fare_amount:double, store_and_fwd_flag:varchar]                                                                                                                                                               
         Estimates: {source: CostBasedSourceInfo, rows: ? (?), cpu: 500.00, memory: 0.00, network: ?}                                                                                                                                                                                                                                                                       
     - RemoteStreamingExchange[PlanNodeId 207][GATHER - COLUMNAR] => [vendor_id:bigint, trip_id:bigint, trip_distance:decimal(5,2), fare_amount:double, store_and_fwd_flag:varchar]                                                                                                                                                                                         
             Estimates: {source: CostBasedSourceInfo, rows: ? (?), cpu: 500.00, memory: 0.00, network: ?}                                                                                                                                                                                                                                                                   
         - ScanFilter[PlanNodeId 0,236][table = TableHandle {connectorId='iceberg2', connectorHandle='taxis$data@2530820414725702192', layout='Optional[taxis$data@2530820414725702192]'}, filterPredicate = (lower(store_and_fwd_flag)) <> (VARCHAR'n')] => [vendor_id:bigint, trip_id:bigint, trip_distance:decimal(5,2), fare_amount:double, store_and_fwd_flag:varchar] 
                 Estimates: {source: CostBasedSourceInfo, rows: 6 (250B), cpu: 250.00, memory: 0.00, network: 0.00}/{source: CostBasedSourceInfo, rows: ? (?), cpu: 500.00, memory: 0.00, network: 0.00}                                                                                                                                                                    
                 vendor_id := 1:vendor_id:bigint (1:22)                                                                                                                                                                                                                                                                                                                     
                 trip_id := 2:trip_id:bigint (1:22)                                                                                                                                                                                                                                                                                                                         
                 fare_amount := 4:fare_amount:double (1:22)                                                                                                                                                                                                                                                                                                                 
                 trip_distance := 3:trip_distance:decimal(5,2) (1:22)                                                                                                                                                                                                                                                                                                       
                 store_and_fwd_flag := 5:store_and_fwd_flag:varchar (1:22)                                                                                                                                                                                                                                                                                                  
                                                                                                                                                                                                                                                                                                                                                                            
(1 row)

Query 20260326_110542_00004_bmix8, FINISHED, 1 node
http://127.0.0.1:8080/ui/query.html?20260326_110542_00004_bmix8
Splits: 1 total, 1 done (100.00%)
[Latency: client-side: 126ms, server-side: 115ms] [0 rows, 0B] [0 rows/s, 0B/s]
```

1. `lower('N')` is constant folded and pushed down along with the predicate `filterPredicate = (lower(store_and_fwd_flag)) <> (VARCHAR'n')` to the Table scan node. However,
   we see `lower(store_and_fwd_flag)` stays as is, table scan cannot perform this filtering efficiently as the underlying connector does not know about the UDF lower,
   and thus ends up scanning large amount of data. The reason is, entire data is first brought inside presto and then filtering is performed.
2. While reading from Table formats e.g. Iceberg, this problem take another shape all together. In Iceberg table, we can easily push down the filter to table scan node,
   but we cannot use it to decide which files to skip for reading. Let me explain this in more detail, Iceberg manifest files store lower and upper bound metrics for each
   column and these metrics decide whether to skip the file for reading or include it. This also happens at file format level, for example Parquet footer statics have the
   information which row group should be having that information. In the case of `lower(col)` UDF function, the sorting order of the column is modified and as a result we cannot
   simply
   compute `lower(lower_bound_of_column)` and `upper(upper_bound_of_column)` to get the same file skipping behavior. This is because `lower` is not a monotonic function.
3. We see `?` in cost estimates, because estimator does not know anything about the UDF `lower` it is not able to perform cost estimates. As a result, in a more complex
   query, when such UDF's appear the join order can be a poor choice, and Plan can be a suboptimal plan. This was also addressed in [RFC-0005](RFC-0005-functions-stats.md).
   In this approach the missing estimates for UDF's can also be addressed by replacing them with derived columns.

### How does derived columns help?

1. `lower(store_and_fwd_flag)` becomes `_1_store_and_fwd_flag`, then we can rewrite the query as `select *from taxis where _1_store_and_fwd_flag != lower('N');` This query
   can be executed as a regular query, and the problem of underlying system unable to prune data for scanning is solved, because now we are dealing with columns instead of UDFs.
2. For the case of Iceberg Table Format, it stores metrics and stats per column (and even bloom filters) in it's metadata that helps skip data files from scanning.
3. The newly derived columns `_1_store_and_fwd_flag` has the stats on it, this gives better selectivity estimates and as a result optimizers can do a better job at
   planning and coming up with an optimal plan. Therefore, problem of missing stats is solved as well.

### Other Computing Engines

#### Spark and Trino

Spark and Trino currently does not do any special handling for this, i.e. Derived column is a novel concept to all distributed database engines.

This spec does not focus on what changes are needed to support this feature in other Distributed database Engines than Prestodb.

#### Mysql, MS-SQL, IBM DB2, Maria DB.

All of the above support this feature as per the defined SQL standard.

1. Maria DB: https://mariadb.com/docs/server/reference/sql-statements/data-definition/create/generated-columns
2. IBM DB2: https://www.ibm.com/docs/en/db2-for-zos/12.0.0?topic=statements-alter-table#db2z_sql_altertable__syntaxdiagram_app_ztw_21c__title__1
3. MS-SQL: https://learn.microsoft.com/en-us/sql/t-sql/statements/create-table-transact-sql?view=sql-server-ver17

## Goals

* Align with SQL Standard for Derived column followed by many RDBMS systems. [Ref](#mysql-ms-sql-ibm-db2-maria-db)
* Present the derived column as a possible solution for solving the data skipping at the TableScan operator.
* Propose metadata changes to Iceberg table properties to provide a portable solution for supporting Derived column .
* Cover the necessary planner changes to benefit from Derived columns.
* Discuss performance benchmarks of with and without this feature.
* Remain backward compatible with previous versions of presto engines and other engines who have not yet adopted derived columns.

The ultimate goal of all of the above is that we can allow data skipping at metadata level and derive performance benefit at the expense of some extra storage.

## Proposed Implementation

In this design, we will focus on Iceberg format initially and lay foundation for extending to other connectors.

### Presto DDL changes:

Create Table:

```sql
CREATE TABLE [ IF NOT EXISTS ]
table_name (
  { column_name data_type [GENERATED ALWAYS]  AS   ( <expression> ) [VIRTUAL | PERSISTENT] [NOT NULL] [ COMMENT comment ] [ WITH ( property_name = expression [, ...] ) ]
  | LIKE existing_table_name [ { INCLUDING | EXCLUDING } PROPERTIES ]
  | [ CONSTRAINT constraint_name ] { PRIMARY KEY | UNIQUE } ( { column_name [, ...] } ) [ { ENABLED | DISABLED } ] [ [ NOT ] RELY ] [ [ NOT ] ENFORCED ] }
  [, ...]
)
[ COMMENT table_comment ]
[ WITH ( property_name = expression [, ...] ) ]
```

For example:

```sql
CREATE TABLE test_table1 (                     
    "c1" bigint,                                                 
    "c2" varchar,                                                
    "c3" double,
    "c2_derived" varchar AS lower(c2) PERSISTENT
 )                                                               

```

Alter Table:

```sql 
ALTER TABLE [ IF EXISTS ] name ADD COLUMN [ IF NOT EXISTS ] column_name data_type [GENERATED ALWAYS]  AS   ( <expression> ) [VIRTUAL | PERSISTENT]
``` 

GENERATED ALWAYS: Enforce that the derived column, cannot be directly insert via INSERT statements.

VIRTUAL: Create a derived column, however do not persist it's data. It is always generated using the specified derived column SQL Expression for the column.
Insert to a VIRTUAL columns are not permitted. The current version of RFC does not cover implementation of VIRTUAL columns and is mentioned in Future Work.

PERSISTENT: Create a derived column and also persist it's data by generating using the specified derived column SQL Expression.

### Caveats: Deviations from SQL standard.

1. We allow direct insert into derived columns. If the table is written by another engine which does not support derived columns, write to derived columns cannot be restricted.
2. It is thus responsibility of the user to maintain derived column information as up to date. Any stale entry in derived column will lead optimizer rewrite to produce incorrect
   result when the derived column feature flags are enabled.
3. We can provide warning, the derived column data is out of sync. And then it is user's responsibility they can drop and recreate the column. (Future work).
4. In addition to the above, there are other possibilities for example the builtin UDF itself is updated by the plugin owner - such events are very hard to auto-detect.
   Since the builtin UDFs are not versioned it can lead to possibly incorrect results. 
5. We do not support UDFs with 'side effects', however, since there are no ways to detect what the UDF may be doing - it is hard to detect.

### What is allowed in derived column expressions

1. All built-in deterministic scalar functions.
2. AND / OR expressions.
3. Arithmetic expressions both binary and unary.
4. If and CASE Statements.
5. isNull and Cast expressions.
6. Deterministic UDFs

### What is not allowed in derived columns expressions

1. Any of the aggregate functions.
2. A sub query expressions.
3. IN Query and Exists query.
4. Non-deterministic functions.

### Table properties

Following table properties are added.

1. `presto.derived-columns.spec.json` :
    ```json 
   {
            "derivedColumnSpecs" : [ {
               "derivedColumnType" : "PERSISTENT",
               "derivedColumnExpression" : "SQL expression",
               "derivedColumnName" : "derived Column"
               }, {
               "derivedColumnType" : "VIRTUAL",
               "derivedColumnExpression" : "SQL expression2",
               "derivedColumnName" : "derived Column2"
               }]
    }
    ```
3. This table property is hidden and not intended for the user to be entered/updated manually,
   rather they are autopopulated when user performs the CREATE TABLE or ALTER TABLE query.
4. There properties are persisted inside the table properties of the iceberg table i.e. metadata.json.
   for example:

```json
{
  "properties": {
    "presto.derived-columns.spec.json": "{\n  \"derivedColumnSpecs\" : [ {\n    \"derivedColumnType\" : \"PERSISTENT\",\n    \"derivedColumnExpression\" : \"lower(c2)\",\n    \"derivedColumnName\" : \"c2_derived\"\n  }, {\n    \"derivedColumnType\" : \"PERSISTENT\",\n    \"derivedColumnExpression\" : \"c1*10.5\",\n    \"derivedColumnName\" : \"c1_derived\"\n  } ]\n}",
    "write.format.default": "PARQUET",
    "write.delete.mode": "merge-on-read",
    "write.metadata.previous-versions-max": "100",
    "write.parquet.compression-codec": "ZSTD",
    "write.metadata.metrics.max-inferred-column-defaults": "100",
    "write.metadata.delete-after-commit.enabled": "false",
    "commit.retry.num-retries": "4",
    "write.update.mode": "merge-on-read",
    "read.split.target-size": "134217728"
  }
}
```

### Feature Flags

1. `derived_columns.enable` : Boolean feature flag to enable or disable derived column feature for query rewrite.

### Session Flag

1. `derived_columns_enabled` : Boolean session flag to enable or disable derived column feature for query rewrite per session.

### SPI changes to ColumnMetadata,

```java
/**
 * This class stores the derived column information.
 */
public class DerivedColumnSpec
{
    private final DerivedColumnType derivedColumnType;
    private final String derivedColumnExpression;
    private final String derivedColumnName;
    private final int derivedColumnFieldId;
    private final String derivedColumnReturnType;

    /**
     * The field derivedColumnType, derivedColumnFieldId and derivedColumnReturnType are used for validation only,
     * these values establish if derived column information has gone stale (by an external update) and needs a refresh.
     *
     * @param derivedColumnType A derived column can either be a GENERATED ALWAYS and PERSISTENT or VIRTUAL or just PERSISTENT.
     * @param derivedColumnExpression A derived column expression, a generic SQL expression that presto recognizes.
     * @param derivedColumnName Name of the derived column
     * @param derivedColumnFieldId field ID is connector dependent sequence number for the column.
     * @param derivedColumnReturnType return type of this column, used for validation purpose.
     */
    @JsonCreator
    public DerivedColumnSpec(
            @JsonProperty("derivedColumnType") DerivedColumnType derivedColumnType,
            @JsonProperty("derivedColumnExpression") String derivedColumnExpression,
            @JsonProperty("derivedColumnName") String derivedColumnName,
            @JsonProperty("derivedColumnFieldId") Integer derivedColumnFieldId,
            @JsonProperty("derivedColumnReturnType") String derivedColumnReturnType)
    {
        this.derivedColumnType = requireNonNull(derivedColumnType, "derivedColumnType is null");
        this.derivedColumnExpression = requireNonNull(derivedColumnExpression, "derivedColumnExpression is null");
        this.derivedColumnName = requireNonNull(derivedColumnName, "derivedColumnName is null");
        this.derivedColumnFieldId = requireNonNull(derivedColumnFieldId, "derivedColumnFieldId is null");
        this.derivedColumnReturnType = requireNonNull(derivedColumnReturnType, "derivedColumnReturnType is null");
    }
}
```

ColumnMetadata is enhanced with a new optional field of type `DerivedColumnSpec`.

```
   private final Optional<DerivedColumnSpec> derivedColumnSpec;

    private ColumnMetadata(
            String name,
            Type type,
            boolean nullable,
            String comment,
            String extraInfo,
            boolean hidden,
            Map<String, Object> properties,
            Optional<DerivedColumnSpec> derivedColumnSpec)
    {
        checkNotEmpty(name, "name");
        requireNonNull(type, "type is null");
        requireNonNull(properties, "properties is null");

        this.name = name;
        this.type = type;
        this.nullable = nullable;
        this.comment = comment;
        this.extraInfo = extraInfo;
        this.hidden = hidden;
        this.properties = properties.isEmpty() ? emptyMap() : unmodifiableMap(new LinkedHashMap<>(properties));
        this.derivedColumnSpec = derivedColumnSpec;
    }

```

### Optimizer changes for SELECT's filter plan rewrite

1. Load the Expression from derived column spec provided in table properties.
2. Get filter predicate from user's query and perform a sub expression search for expressions mentioned in Derived column spec.
3. If the Expression defined in the spec is found, rewrite the filter predicate by replacing the target expression with derived column.

### Optimizer changes for SELECT's projection plan rewrite.

Projection rewrite is useful in situation where a Expression is compute-intensive for example in a query

```sql
SELECT col1, sort_each_row(col2) FROM TABLE1;
```

May be rewritten as:

```sql
SELECT col1, sorted_col2 AS 'sort_each_row(col2)' FROM TABLE1;
```

This transformation is possible by rewriting the projections.

### POC: With only SELECT filter rewrite turned on

```sql
presto:perf_test> CREATE TABLE test (c1 BIGINT, c2 VARCHAR, c2_derived varchar AS lower(c2) PERSISTENT, c1_derived decimal(19,2) AS c1 * 10.5);
CREATE TABLE

Query 20260525_134157_00039_nqqq6, FINISHED, 0 nodes
http://127.0.0.1:8080/ui/query.html?20260525_134157_00039_nqqq6
Splits: 0 total, 0 done (0.00%)
[Latency: client-side: 13ms, server-side: 8ms] [0 rows, 0B] [0 rows/s, 0B/s]

presto:perf_test> INSERT INTO test VALUES (123, 'B', lower('B'), 123 * 10.5), (120, 'C', lower('C'), 120 * 10.5), (121, 'A', lower('A'), 121 * 10.5);
INSERT: 3 rows

Query 20260525_134200_00040_nqqq6, FINISHED, 1 node
http://127.0.0.1:8080/ui/query.html?20260525_134200_00040_nqqq6
Splits: 35 total, 35 done (100.00%)
[Latency: client-side: 79ms, server-side: 76ms] [0 rows, 0B] [0 rows/s, 0B/s]

presto:perf_test> show create table test;
                                          Create Table                                           
-------------------------------------------------------------------------------------------------
 CREATE TABLE iceberg.perf_test.test (                                                           
    "c1" bigint,                                                                                 
    "c2" varchar,                                                                                
    "c2_derived" varchar AS lower(c2) PERSISTENT,                                                
    "c1_derived" decimal(19,2) AS c1*10.5 PERSISTENT                                             
 )                                                                                               
 WITH (                                                                                          
    "format-version" = '2',                                                                      
    location = 'file:/Users/prashantsharma/work/presto_config/hive-data/iceberg/perf_test/test', 
    "read.split.target-size" = 134217728,                                                        
    "write.delete.mode" = 'merge-on-read',                                                       
    "write.format.default" = 'PARQUET',                                                          
    "write.metadata.delete-after-commit.enabled" = false,                                        
    "write.metadata.metrics.max-inferred-column-defaults" = 100,                                 
    "write.metadata.previous-versions-max" = 100,                                                
    "write.update.mode" = 'merge-on-read'                                                        
 )                                                                                               
(1 row)

Query 20260525_134339_00047_nqqq6, FINISHED, 1 node
http://127.0.0.1:8080/ui/query.html?20260525_134339_00047_nqqq6
Splits: 1 total, 1 done (100.00%)
[Latency: client-side: 29ms, server-side: 17ms] [0 rows, 0B] [0 rows/s, 0B/s]

presto:perf_test> select c1,c2 from test WHERE c1 * 10.5 > 1200.0 and lower(c2) = lower('A');
 c1  | c2 
-----+----
 121 | A  
(1 row)

Query 20260525_134435_00048_nqqq6, FINISHED, 1 node
http://127.0.0.1:8080/ui/query.html?20260525_134435_00048_nqqq6
Splits: 17 total, 17 done (100.00%)
[Latency: client-side: 55ms, server-side: 44ms] [3 rows, 771B] [68 rows/s, 17.1KB/s]

presto:perf_test> explain select c1,c2 from test WHERE c1 * 10.5 > 1200.0 and lower(c2) = lower('A');
                                                                                                                                                                       Query Plan                                                                                                                                                                        
---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
 - Output[PlanNodeId 6][c1, c2] => [c1:bigint, c2:varchar]                                                                                                                                                                                                                                                                                               
         Estimates: {source: CostBasedSourceInfo, rows: 2 (96B), cpu: 816.00, memory: 0.00, network: 96.00}                                                                                                                                                                                                                                              
     - RemoteStreamingExchange[PlanNodeId 221][GATHER - COLUMNAR] => [c1:bigint, c2:varchar]                                                                                                                                                                                                                                                             
             Estimates: {source: CostBasedSourceInfo, rows: 2 (96B), cpu: 816.00, memory: 0.00, network: 96.00}                                                                                                                                                                                                                                          
         - ScanFilter[PlanNodeId 0,254][table = TableHandle {connectorId='iceberg', connectorHandle='test$data@4741565407715570639', layout='Optional[test$data@4741565407715570639]'}, filterPredicate = ((c1_derived) > (DECIMAL'1200.0')) AND ((c2_derived) = (VARCHAR'a'))] => [c2:varchar, c2_derived:varchar, c1:bigint, c1_derived:decimal(19,2)] 
                 Estimates: {source: CostBasedSourceInfo, rows: 3 (408B), cpu: 408.00, memory: 0.00, network: 0.00}/{source: CostBasedSourceInfo, rows: 2 (204B), cpu: 816.00, memory: 0.00, network: 0.00}                                                                                                                                              
                 c2 := 2:c2:varchar (1:27)                                                                                                                                                                                                                                                                                                               
                 c2_derived := 3:c2_derived:varchar                                                                                                                                                                                                                                                                                                      
                     :: [["a"]]                                                                                                                                                                                                                                                                                                                          
                 c1 := 1:c1:bigint (1:27)                                                                                                                                                                                                                                                                                                                
                 c1_derived := 4:c1_derived:decimal(19,2)                                                                                                                                                                                                                                                                                                
                     :: [("1200.0", <max>)]                                                                                                                                                                                                                                                                                                              
                                                                                                                                                                                                                                                                                                                                                         
(1 row)

Query 20260525_134440_00049_nqqq6, FINISHED, 1 node
http://127.0.0.1:8080/ui/query.html?20260525_134440_00049_nqqq6
Splits: 1 total, 1 done (100.00%)
[Latency: client-side: 38ms, server-side: 29ms] [0 rows, 0B] [0 rows/s, 0B/s]

```

Here the user configured expression for derived column is `lower(c2)` and `c1*10.5`, the query's filter predicate is `c1 * 10.5 > 1200.0 and lower(c2) = lower('A')`
and after query rewrite the plan has `filterPredicate = ((c1_derived) > (DECIMAL'1200.0')) AND ((c2_derived) = (VARCHAR'a'))`, which implies, the subexpression
`lower(c2)` was replaced by derived column `c2_derived` and `c1 * 10.5` was replaced by `c1_derived`.

POC PR: https://github.com/prestodb/presto/pull/27832

### Design Considerations

Fundamentally, alternative approaches involve two basic consideration. 1) We want derived column feature to be portable, For example, A table read by Presto today,
may be updated by Spark and then Spark should be able to update it correctly.
Second basic consideration is, we remain compatible with any existing Engines or previous version of Iceberg. Reading or writing by such an engine should not find/render the data
corrupted if they happen to read or write to the Iceberg table with derived column on it, nor it should run into errors. For example, A table with derived column on it in
Prestodb, should not cause failure while ingesting data via Apache Spark.

**What should happen if, the table written by Presto aware of derived column is read by another version of presto unaware of derived column feature?**
The presto version unaware of derived column feature will see derived column as an extra column in the table. And thus will have to adapt to the new schema for both SELECT
and INSERT/UPDATE queries.

**What should happen if, the table written by Presto aware of derived column is appended by another engine unaware of derived column feature?**
The resulting table should not be corrupted. Because of schema mismatch the older scripts may have to be upgraded to include the extra derived column.

**If the derived feature flag is turned off, does it make derived column disappear?**
No, derived column will appear as any other column in the table. It will not be hidden and the optimization of rewriting plan with derived column will be turned off.

Challenges:

1) Suppose we append the extra column to the underlying data files and do not have schema evolution visible. Then append to the underlying files shall not corrupt it.
2) What if there is addition of derived column and then addition of actual column and then again derived column and so on. This can be a problem, when a writer unaware of
   derived column feature tries to write data. It can corrupt the internal records.
3) Suppose we evolve the schema for Iceberg table on adding a derived column, this is the best alternative because it will not corrupt the data and we remain backward compatible
   with engines unaware of this feature. However, it will break the requirement that insert from the engine unaware of the derived column feature will not work, unless the INSERT
   queries themselves are modified. This is also not desirable.

## Alternatives considered.

### 1. Use table properties to configure derived column information (instead of alter column syntax).

#### Pros:

1. Easy to set and unset via table properties. Requires no changes to parser to support new syntax.
2. Allows for a design where no changes to Iceberg are required i.e. Iceberg sees derived column as a first class column created by user. And table properties
   tells the planner and table writer to rewrite SELECT and INSERT queries to take advantage of derived column feature.
3. Since table properties are stored inside the Iceberg metadata.json, it will allow the feature to be portable to other engines.

#### Cons:

1. User's set table properties per derived column. It is counterintuitive, because the derived column property is column specific. It is also counterintuitive, because derived
   column is actually a real column underneath a property does not give intuition of that.
2. The feature becomes not portable, for example a user's existing sql scripts to load data via other engines will require them to include additional column(s).

### 2. Use puffin files to store derived column metadata.

#### Pros:

1. Provides a way to store information in Iceberg table itself, and thus we can make derived column metadata portable across engines.
2. Does not require us to modify Iceberg Spec.

#### Cons:

1. Puffin files are designed for storing stats and not any other information.
2. This approach will never be accepted by OSS community.
3. The engines unaware of this feature can corrupt table while writing. See Challenges 2).

### 3. Use Puffin files to store extra column stats (i.e. stats for column that does not really exist)

We can get files/partitions skipping behaviour at best, but we will not get row skipping unless we have the extra column actually store. And if we do have a real column
then column stats are automatically generated i.e. the current Iceberg connector behaviour. This can corrupt stats when writes are performed by engine unaware of derived
columns.

### 4. Use hive metastore (or some external metastore) to store derived column metadata.

The biggest disadvantage is, metadata changes are not portable. Iceberg is a table format and a user would like their table be read and written in same way across all engines.
It is not always possible to ship hive metastore along with the iceberg table to different engines. This can corrupt data when writes are performed by engine unaware of derived
columns. See Challenges 2)

### 5. What happens when derived columns' data goes stale?

A stale derived column can produce wrong results when the feature is enabled.
We shall provide builtin semantics to refresh derived columns, similar to Materialized views REFRESH.

## Future work

* Extend derived column benefits to other connectors, including jdbc based ones.
* Support for INSERT/UPDATE/MERGE rewrite.
* Support virtual columns and make derived columns a first class feature in Iceberg Format Spec. ( https://github.com/apache/iceberg/issues/15923)
* Make derived column rewrite available across other engines.
* Support for GENERATED ALWAYS, requires support for INSERT, therefore it will be covered in next iteration of this work.
* Provide a warning when Derived column is not in-sync. This can be done once we have INSERT/UPDATE/MERGE support.
* We can also determine if an expression is suitable candidate for turning into derived column and suggest it to user.

## Adoption Plan

- We plan on providing usage guide and relevant documentation to help adoption.
- Provide guidance on work around for lack of insert/update rewrite support.
- Insert/update rewrite support will be provided in the next phase of the work.

## Usage Guide:

1: Connect to a Iceberg catalog (currently only supported for iceberg Catalog.)

via presto cli for example: `./presto-cli --server host:port --catalog iceberg --user username`

2: Select a schema where you would like to create the table.

```
presto> use perf_test;
USE
```

3: Create a table with derived column information on it.

```sql
create table test (c1 BIGINT, c2 VARCHAR, c2_derived varchar GENERATED ALWAYS AS lower(c2) PERSISTENT);
```

4: Insert some records, we need to keep derived column in sync with the expression

```sql
presto:perf_test> Insert into test values (100, 'some String', lower('some String')) , (120, 'stRing', lower('stRing'));
INSERT: 2 rows
```

5: Enable derived column query rewrite by setting the session flag

```sql
presto:perf_test> set session iceberg.derived_columns_enabled=true;
SET SESSION
```

6: Perform select and if the query happens to use derived column expression, the optimization will be applied

```sql
SELECT c1 from test WHERE c1 > 100 AND lower(c2) = 'string';
```

One can examine, `explain` query to see if the rewrite really happened.

```sql
EXPLAIN SELECT * from test WHERE c1 > 100 AND lower(c2) = 'string';
```

Produces:

```
presto:perf_test> EXPLAIN SELECT c1 from test WHERE c1 > 100 AND lower(c2) = 'string';
                                                                                                                                                        Query Plan                                                                                                                                                        
--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
 - Output[PlanNodeId 6][c1] => [c1:bigint]                                                                                                                                                                                                                                                                                
         Estimates: {source: CostBasedSourceInfo, rows: 1 (9B), cpu: 485.00, memory: 0.00, network: 9.00}                                                                                                                                                                                                                 
     - RemoteStreamingExchange[PlanNodeId 288][GATHER - COLUMNAR] => [c1:bigint]                                                                                                                                                                                                                                          
             Estimates: {source: CostBasedSourceInfo, rows: 1 (9B), cpu: 485.00, memory: 0.00, network: 9.00}                                                                                                                                                                                                             
         - ScanFilterProject[PlanNodeId 0,331,2][table = TableHandle {connectorId='iceberg', connectorHandle='test$data@3907958604296965094', layout='Optional[test$data@3907958604296965094]'}, filterPredicate = ((c1) > (BIGINT'100')) AND ((c2_derived) = (VARCHAR'string')), projectLocality = LOCAL] => [c1:bigint] 
                 Estimates: {source: CostBasedSourceInfo, rows: 2 (18B), cpu: 238.00, memory: 0.00, network: 0.00}/{source: CostBasedSourceInfo, rows: 1 (9B), cpu: 476.00, memory: 0.00, network: 0.00}/{source: CostBasedSourceInfo, rows: 1 (9B), cpu: 485.00, memory: 0.00, network: 0.00}                            
                 c2 := 2:c2:varchar (1:24)                                                                                                                                                                                                                                                                                
                 c2_derived := 3:c2_derived:varchar                                                                                                                                                                                                                                                                       
                     :: [["string"]]                                                                                                                                                                                                                                                                                      
                 c1 := 1:c1:bigint (1:24)                                                                                                                                                                                                                                                                                 
                     :: [("100", <max>)]                                                                                                                                                                                                                                                                                  
                                                                                                                                                                                                                                                                                                                          
(1 row)
```

The rewritten filter predicate looks like: ` filterPredicate = ((c1) > (BIGINT'100')) AND ((c2_derived) = (VARCHAR'string')`

## Test Plan

- Relevant tests both integration and unit tests to test this feature end to end will be added.

## References

### 1. DB2 derived column - Predicate derivation and monotonicity detection in DB2 UDB - https://ieeexplore.ieee.org/abstract/document/1410205