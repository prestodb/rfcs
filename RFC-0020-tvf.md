# **RFC-0020 for Presto**

See [CONTRIBUTING.md](CONTRIBUTING.md) for instructions on creating your RFC and the process surrounding it.

## Presto Table Functions

Proposers

* Aditi Pandit (Aditi.Pandit@ibm.com)
* Michael Ohsaka
* Matthew Zhang
* Pratik Dabre

## Related Issues

https://github.com/prestodb/presto/issues/24648
https://github.com/prestodb/presto/pull/25032

## Background

Table functions (or Table valued functions) are functions that return a table in SQL.
They can be used in the SQL FROM clause just like regular tables in SQL queries. 
Table functions can have other tables as input arguments as well. This enables the user to compose 
them with other parts of the SQL query in a very flexible manner.

Different SQL engines have provided useful built-in table functions to the users. 
The standard doesn’t really have a list of specific table functions. But table functions are most 
popular as an extensibility feature. Over the years table functions have been used for 
building database connectivity extensions, sophisticated pattern recognition libraries, 
analytic functions, integrating machine learning with SQL, among other applications.
Ref https://docs.teradata.com/r/Enterprise_IntelliFlex_VMware/Database-Analytic-Functions

Table functions and particularly Polymorphic table functions are a SQL standard (from 2016). 
They are supported by many modern SQL engines including Trino.

References

- Trino https://trino.io/docs/current/develop/table-functions.html
- Spark https://spark.apache.org/docs/3.5.3/sql-ref-syntax-qry-select-tvf.html
- Snowflake https://docs.snowflake.com/en/sql-reference/functions-table
- Bigquery https://cloud.google.com/bigquery/docs/table-functions

Presto doesn’t support table functions so it limits its use-cases. At IBM we have several uses for
table functions most notably to facilitate orchestration with other data systems, tools and new
ML/GenAI applications.

We will design an SPI and follow SQL syntax compatible with Trino as we care for easy migration of 
user/ecosystem code between the two engines.

In terms of the implementation, we are strongly influenced by the ideas in
- [1] SQL/MapReduce: a practical approach to self-describing, polymorphic, and parallelizable
user-defined functions. https://www.vldb.org/pvldb/vol2/vldb09-464.pdf
- [2] Accelerating BigData analytics with Collaborative Planning in Teradata Aster 6 https://ieeexplore.ieee.org/abstract/document/7113378
- [3] Not Black-Box Anymore! Enabling Analytics-Aware Optimizations in Teradata Vantage https://vldb.org/pvldb/vol14/p2959-eltabakh.pdf

This document proposes adding table functions to Presto and Presto C++. The first section goes over 
SQL syntax for TVFs. After that the document outlines the SPI and then planner and execution changes.

Note: This API is being implemented for both the Java and C++ Presto variants. This document gives details in Java,
but the same is being implemented for C++ as well. Refer https://github.com/prestodb/presto/pull/25032 
for a PR porting Trino TVF code to Presto.

## Proposed Implementation
### TVF SQL syntax
The below SQL snippet gives an example of the invocation of a Table Valued function in a SQL query

```sql 
SELECT * FROM TABLE(
    mock.system.different_arguments_function( 
        INPUT_1 => TABLE(SELECT 'a') t1(c1) PARTITION BY c1 ORDER BY c1, 
        INPUT_3 => TABLE(SELECT 'b') t3(c3) PARTITION BY c3, 
        INPUT_2 => TABLE(VALUES 1) t2(c2), 
        ID => BIGINT '2001', 
        LAYOUT => DESCRIPTOR (x boolean, y bigint) 
        COPARTITION (t1, t3))) t 
```

A Table function in SQL occurs in the FROM clause just like a regular table. A table function is wrapped 
within a TABLE(...) function invocation clause.

In the above SQL, a table function named “diffierent_arguments_function” is registered in the “system” schema
in a connector called “mock”. So it is named with its fully resolved name “mock.system.different_arguments_function”.

A Table function can have multiple arguments, which can be of 3 types:
- Table arguments : Denoted with TABLE. 
  - These arguments are for rows of a table. The argument could resolve to a SELECT query, VALUES list or just a simple TABLE name. 
  - The clause can provide an alias for the TABLE argument. The alias could also have identifiers for individual column names. E.g. t1(c1), t2(c2) and t3(c3) above. 
  - The argument can have a PARTITION BY clause. If a PARTITION BY is specified then the input table rows are partitioned by those columns like a window function, there is a single table function invocation for each partition of data. The partitions can be on different nodes in a distributed cluster and can be executed in parallel, independent of each other. 
  - The argument can have an ORDER BY clause. If one is specified, then the input rows are ordered by that column in the iterator provided to the table function. 
  - COPARTITION specifies to co-locate the tables based on the partition columns. This can be thought of as an equi-join on the PARTITION_BY columns of the table arguments. 
  - You can also specify PRUNE WHEN EMPTY or KEEP WHEN EMPTY. These are used to optimize the query. With PRUNE WHEN EMPTY the function result is empty for an empty partition. But with KEEP WHEN EMPTY the function is executed even if the table argument has no rows.

- Descriptor arguments : Denoted with DESCRIPTOR (some systems like Oracle call these COLUMN arguments).
    - These arguments represent a column (more precisely field) list. 
    - They have a list of names (and optionally types). 
    - These are typically interpreted by the function in the context of some input table argument or the output table layout.

- Scalar arguments : These are constant scalar type arguments

Arguments can be passed by name or by position. The argument conventions cannot be mixed in a single invocation.

Passed by name : In this convention the arguments indicate the argument name as specified in the function
definition (described later), so they can be passed in arbitrary order. Arguments are resolved case-sensitive
and with upper casing of unquoted names. Arguments with default values can be skipped.

e.g.
```sql
SELECT * FROM TABLE(my_function(row_count => 100, column_count => 1))
```

Passed by position : In this convention the arguments are passed in the order in which they are 
declared in the function definition. A suffix of the argument list can be skipped provided they all 
have default values.

e.g.
```sql
SELECT * FROM TABLE(my_function(1, 100))
```

#### Polymorphism

Table functions are somewhat different from SQL scalar, aggregate or window functions because they are polymorphic
in nature. The polymorphism happens because table functions can be invoked with table arguments of different schema,
returning output tables with varying schema as well. Table function signatures could be fixed typed, in which case
no further analysis is required. But that’s usually not the case.

As an example, a quite popular Table function in Trino is exclude_columns. This table function is useful for
queries where you want to return nearly all columns from tables with many columns. You can avoid enumerating all
columns, and only need to specify the columns to exclude.

exclude_columns(input => table, columns => descriptor) → table

The argument input is a table or a query. The argument columns is a descriptor without types.
Example query using the orders table from the TPC-H dataset

```sql
SELECT * FROM TABLE(exclude_columns(
                        input => TABLE(orders), 
                        columns => DESCRIPTOR(clerk, comment)));
```

In any other invocation of exclude_columns, we could use another table and set of columns. Each of the
invocations would have a different output schema for the SQL query.

### TVF API
Table functions are very popular for extensibility. Clients or tools using Presto can add custom logic
and invoke it in a query via table functions. So it's important to have a simple but powerful API for them. The SQL
standard is not prescriptive about an API for table functions but provides some guidance.

An important concern in this work was to have an API compatible with Trino so that its easy for users/tools to 
interoperate between Presto and Trino. We examined the Trino API and found it quite sophisticated for our 
first cut implementation. There are more enhancements we have for optimizations and for advanced lateral
use which we will describe later on.

Table functions are implemented through connectors. A TableFunction derives from AbstractConnectorTableFunction.

```java
// ReturnTypeSpecification describes the table being returned. It could be a Generic Table that is 
// determined on analysis, pass-through and has the same schema as the input table or DescribedTable with a fixed
// schema.
@JsonTypeInfo(
        use = JsonTypeInfo.Id.NAME,
        include = JsonTypeInfo.As.PROPERTY,
        property = "@type")
@JsonSubTypes({
        @JsonSubTypes.Type(value = GenericTableReturnTypeSpecification.class, name = "generic_table"),
        @JsonSubTypes.Type(value = OnlyPassThroughReturnTypeSpecification.class, name = "only_pass_through_table"),
        @JsonSubTypes.Type(value = DescribedTableReturnTypeSpecification.class, name = "described_table")})
public abstract class ReturnTypeSpecification {}

public abstract class AbstractConnectorTableFunction
       implements ConnectorTableFunction
{
   private final String schema;
   private final String name;
   private final List<ArgumentSpecification> arguments;
   private final ReturnTypeSpecification returnTypeSpecification; 
   
   public AbstractConnectorTableFunction(String schema,
                                         String name,
                                         List<ArgumentSpecification> arguments,
                                         ReturnTypeSpecification returnTypeSpecification)
   {
       this.schema = requireNonNull(schema, "schema is null");
       this.name = requireNonNull(name, "name is null");
       this.arguments = Collections.unmodifiableList(new ArrayList<>(requireNonNull(arguments, "arguments is null")));
       this.returnTypeSpecification = requireNonNull(returnTypeSpecification, "returnTypeSpecification is null");
   } 
   
   @Override
   public String getSchema()
   {
       return schema;
   } 
   
   @Override
   public String getName()
   {
       return name;
   }
   
   @Override
   public List<ArgumentSpecification> getArguments()
   {
       return arguments;
   } 
   
   @Override
   public ReturnTypeSpecification getReturnTypeSpecification()
   {
       return returnTypeSpecification;
   } 
   
   @Override
   public abstract TableFunctionAnalysis 
        analyze(ConnectorSession session,
                ConnectorTransactionHandle transaction,
                Map<String, Argument> arguments);
} 
```

#### Function declaration
Like shown above, a Table function declaration comprises a function name, schema name, list of input
argument specifications and a return type specification. This declaration is used to populate a registry of
Table functions used at the co-ordinator. When a query is received and parsed, the registry is looked up to
identify and resolve the TableFunction.

##### Argument specifications
At an abstract level each table function argument has a name, required spec and potential default value.

```java
public abstract class ArgumentSpecification 
{
   private final String name;
   private final boolean required;
   // native representation
   private final Object defaultValue; 
   
   ArgumentSpecification(String name, boolean required, @Nullable Object defaultValue)
   {
       this.name = checkNotNullOrEmpty(name, "name");
       checkArgument(!required || defaultValue == null, "non-null default value for a required argument");
       this.required = required;
       this.defaultValue = defaultValue;
   } 
   
   public String getName()
   {
       return name;
   }
}
```

Though in concrete terms arguments like described in the previous section can be scalar, descriptor or table argument types.

##### ScalarArgument

Scalar argument matches a constant literal value in the table function invocation. In addition to the abstract properties,
a scalar argument also provides a field for the type of the variable

```java
public class ScalarArgumentSpecification
       extends ArgumentSpecification
{
   private final Type type;
   
   private ScalarArgumentSpecification(String name, Type type, boolean required, Object defaultValue)
   {
       super(name, required, defaultValue);
       this.type = requireNonNull(type, "type is null");
       if (defaultValue != null) {
           checkArgument(Primitives.wrap(type.getJavaType()).isInstance(defaultValue), 
                   format("default value %s does not match the declared type: %s", defaultValue, type));
       }
   } 
   
   public Type getType()
   {
       return type;
   }
} 
```

#### DescriptorArgument
Descriptor arguments are for a list of fields. The fields have a name and an optional type.

```java
public class DescriptorArgumentSpecification
       extends ArgumentSpecification
{
   private DescriptorArgumentSpecification(String name, boolean required, Descriptor defaultValue)
   {
       super(name, required, defaultValue);
   }
} 

public class Descriptor
{
   private final List<Field> fields; 
   
   @JsonCreator
   public Descriptor(@JsonProperty("fields") List<Field> fields)
   {
       requireNonNull(fields, "fields is null");
       checkArgument(!fields.isEmpty(), "descriptor has no fields");
       this.fields = unmodifiableList(fields);
   } 
   
   public static Descriptor descriptor(String... names)
   {
       List<Field> fields = Arrays.stream(names)
               .map(name -> new Field(name, Optional.empty()))
               .collect(Collectors.toList());
       return new Descriptor(fields);
   } 
   
   public static Descriptor descriptor(List<String> names, List<Type> types)
   {
       requireNonNull(names, "names is null");
       requireNonNull(types, "types is null");
       checkArgument(names.size() == types.size(), "names and types lists do not match");
       List<Field> fields = new ArrayList<>();
       for (int i = 0; i < names.size(); i++) {
           fields.add(new Field(names.get(i), Optional.of(types.get(i))));
       }
       return new Descriptor(fields);
   } 
   
   public static class Field
   {
       private final String name;
       private final Optional<Type> type; 
       
       @JsonCreator
       public Field(@JsonProperty("name") String name, @JsonProperty("type") Optional<Type> type)
       {
           this.name = checkNotNullOrEmpty(name, "name");
           this.type = requireNonNull(type, "type is null");
       }
   }
}
```

#### Table argument

This represents a table input. Table arguments have some additional properties.

##### Row or Set semantics

A table argument with set semantics is always invoked with a PARTITION BY clause and the table function is processed on
a partition by partition basis. If no partitioning on a table argument with set semantics then the entire input is
processed as a single partition.

A table argument with row semantics is processed on a row-by-row basis. Partitioning or ordering is not applicable.

##### Prune or keep when empty

Prune when empty property indicates that if the table argument is empty then the function returns empty result. 

If keep when empty then the function is executed even if the table argument is empty. It is similar to the inner/outer join property.

##### Pass-through columns

All columns of the table are pass-through if this property is set. Else only partition columns of the table are pass-through.

```java
public class TableArgumentSpecification
       extends ArgumentSpecification
{
   private final boolean rowSemantics;
   private final boolean pruneWhenEmpty; 
   private final boolean passThroughColumns; 
   
   private TableArgumentSpecification(String name, boolean rowSemantics, boolean pruneWhenEmpty, boolean passThroughColumns)
   {
       super(name, true, null);
       
       requireNonNull(pruneWhenEmpty, "The pruneWhenEmpty property is not set");
       checkArgument(!rowSemantics || pruneWhenEmpty, 
               "Cannot set the KEEP WHEN EMPTY property for a table argument with row semantics"); 
       
       this.rowSemantics = rowSemantics;
       this.pruneWhenEmpty = pruneWhenEmpty;
       this.passThroughColumns = passThroughColumns;
   } 
   
   public boolean isRowSemantics()
   {
       return rowSemantics;
   } 
   
   public boolean isPruneWhenEmpty()
   {
       return pruneWhenEmpty;
   } 
   
   public boolean isPassThroughColumns()
   {
       return passThroughColumns;
   }
   ….
}
```

### Analyze API

Since Table functions are polymorphic, they participate in the StatementAnalysis phase of the query processing.
StatementAnalysis happens early in query processing, immediately after the query is parsed. After a query is parsed,
the system proceeds with analyzing each parse node to validate it and infer its schema (column or table). When the
system encounters a TableFunction ParseNode in its tree, it first looks up the table function registry and resolves the
function from its signature specified above. From the function signature it can associate the input parameters and
can compile their schema. However, to continue further analysis it needs feedback from the TableFunction to get its
output schema. The TableFunction::analyze method serves this purpose.

The TableFunction::analyze takes as input the table function arguments and returns a TableFunctionAnalysis object. 
The function author is expected to validate the input arguments in this function and return the following 
information to the query processing:

- Return type of the function. The return type is a descriptor which is a list of fields each of which has a name and type.
- Required columns of the input tables passed. The table function might not use all the columns of the input tables
  passed to it. By telling the query processor which specific input columns are required, the optimizer can prune 
  un-needed input columns from the sub-graphs below it.
- TableFunctionHandle: This structure is used by the function to pass context determined at analysis to the runtime.
  When the table function is executed by a subsequent operator at the worker, the TableFunctionHandle is passed to it. 
  The handle can be used by the function to calculate some global state that each worker should know about during the
  execution. 
  - E.g. If the function is used to calculate the price of some item on a particular day, then the context can contain,
  say the dollar exchange rate on that day for further calculations.

```java
public final class TableFunctionAnalysis
{
   // a map from table argument name to list of column indexes for all columns required from the table argument
   private final Map<String, List<Integer>> requiredColumns;
   private final Optional<Descriptor> returnedType;
   private final ConnectorTableFunctionHandle handle; 
   
   private TableFunctionAnalysis(Optional<Descriptor> returnedType,
                                 Map<String, List<Integer>> requiredColumns,
                                 ConnectorTableFunctionHandle handle)
   {
       this.returnedType = requireNonNull(returnedType, "returnedType is null");
       returnedType.ifPresent(descriptor -> checkArgument(descriptor.isTyped(), "field types not specified")); 

       this.requiredColumns = Collections.unmodifiableMap(
               requiredColumns.entrySet().stream()
                       .collect(Collectors.toMap(
                               Map.Entry::getKey,
                               entry -> Collections.unmodifiableList(entry.getValue()))));
       this.handle = requireNonNull(handle, "handle is null");
   } 
   
   public Optional<Descriptor> getReturnedType()
   {
       return returnedType;
   } 
   
   public Map<String, List<Integer>> getRequiredColumns()
   {
       return requiredColumns;
   } 
   
   public ConnectorTableFunctionHandle getHandle()
   {
       return handle;
   }
} 

public abstract TableFunctionAnalysis 
    analyze(ConnectorSession session,
            ConnectorTransactionHandle transaction,
            Map<String, Argument> arguments);
```

### TVF Execution API

The Table function runtime processing could happen in 3 ways :
- Apply mechanism : Some table functions can be compiled to TableScans 
e.g. If they are a shorthand for a particular optimization. To convert a table function to a TableScan 
the author implements an apply function with the API described below.
- TableFunctionDataProcessor
In this style, the table function operates on a block of input rows to generate a block of output rows.
- TableFunctionSplitProcessor
In this style, the table function is a leaf table scan. Like other table scans, the author implements APIs to 
enumerate splits and generate output data for each input split.

##### Apply

A TableFunction is converted to a TableScan if it implements an apply function for the translation. 
The apply function is part of the Connector Metadata (as the connector is used for generating splits).

```java
@Override 
public Optional<TableFunctionApplicationResult<TableHandle>> applyTableFunction(Session session, TableFunctionHandle handle) 
{ 
    ConnectorId connectorId = handle.getConnectorId(); 
    ConnectorMetadata metadata = getMetadata(session, connectorId); 
 
    return metadata.applyTableFunction(session.toConnectorSession(connectorId), handle.getFunctionHandle()) 
        .map(result -> new TableFunctionApplicationResult<>( 
            new TableHandle(connectorId, result.getTableHandle(), handle.getTransactionHandle(), Optional.empty()), 
                result.getColumnHandles())); 
} 
```

##### TableFunctionSplitProcessor
If the TableFunction is a leaf operator and processes splits the author needs to implement a
TableFunctionSplitProcessor. It is called multiple times for the whole output until the split is processed.
A new TableFunction object is created for each new split.

```java
public interface TableFunctionSplitProcessor
{
   /**
    * This method processes a split. It is called multiple times until the whole output for the split is produced.
    *
    * @param split a {@link ConnectorSplit} representing a subtask.
    * @return {@link TableFunctionProcessorState} including the processor's state and optionally a portion of result.
    * After the returned state is {@code FINISHED}, the method will not be called again.
    */
   TableFunctionProcessorState process(ConnectorSplit split);
}
```

#### TableFunctionDataProcessor
If the TableFunction operates on other input tables, then the author needs to implement a TableFunctionDataProcessor.
Each process call of the function receives a list of pages, one for each input table. Each page has the required columns
requested during analysis. The rows in the pages are ordered by the row number based on the sort order.
If any of the sources is fully processed, then the page for it  is empty.

```java
public interface TableFunctionDataProcessor
{
   /**
    * This method processes a portion of data. It is called multiple times until the partition is fully processed.
    *
    * @param input a tuple of {@link Page} including one page for each table function's input table.
    * Pages list is ordered according to the corresponding argument specifications in {@link ConnectorTableFunction}.
    * A page for an argument consists of columns requested during analysis (see {@link TableFunctionAnalysis#getRequiredColumns()}}.
    * If any of the sources is fully processed, {@code Optional.empty)()} is returned for that source.
    * If all sources are fully processed, the argument is {@code null}.
    * @return {@link TableFunctionProcessorState} including the processor's state and optionally a portion of result.
    * After the returned state is {@code FINISHED}, the method will not be called again.
    */
   TableFunctionProcessorState process(List<Optional<Page>> input);
}
```

### TVF Planning

A Table function is executed at each worker node in the cluster. So we have a logical PlanNode and the corresponding
runtime Operator for it. The TableFunctionNode has fields for the arguments passed (for table arguments this includes
the partition and order by columns), output variables, source planNodes, co-partition lists and the TableFunctionHandle
previously determined during statement analysis.

```java
public TableFunctionNode(
       @JsonProperty("id") PlanNodeId id,
       @JsonProperty("name") String name,
       @JsonProperty("arguments") Map<String, Argument> arguments,
       @JsonProperty("outputVariables") List<VariableReferenceExpression> outputVariables,
       @JsonProperty("sources") List<PlanNode> sources,
       @JsonProperty("tableArgumentProperties") List<TableArgumentProperties> tableArgumentProperties,
       @JsonProperty("copartitioningLists") List<List<String>> copartitioningLists,
       @JsonProperty("handle") TableFunctionHandle handle)
{
  …
}
```


Important steps when planning table functions:
- TableFunction as a TableScan : A TableFunction can be planned with a TableScan if it implments an apply()
  function in the TableFunction API for the translation.

- TableFunctionOperator as a Leaf operator: If a table function doesn’t have any input tables,
  then its planned as a leaf PlanNode and operator. Its output table participates in further planning like a regular table.

- Table function operator with a single table input:
  - If the table function is a row-by-row function, then the table function can be executed as a regular Project/Filter
    node with its input table as an upstream source.
    - If the table has a partition by clause, then the table function is planned like a WindowFunction.
      - The difference between window and table operator is that window adds a single column to the input table and each
        input row gets a single value for it. In a table function, each partition is operated in parallel, but there
        isn’t necessarily a 1:1 relation between the input and output rows.
- Table function operator with multiple input tables: In this case, a join of the input tables is needed.
  The join conditions depend on the prune and empty property. The co-partitioning also adds a join equality condition.
  The input tables need  to be partitioned and ordered as per the partition by and order by clause, so that is done with
  a RowNumber window function.

This logic is implemented with a planning rule ImplementTableFunctionSource (https://github.com/mohsaka/presto/pull/14/files#diff-b32cac3ca5fff1ccde184d5532e4f3036cefd070815e140ca2e13e48be7aad29)

TableFunction foo
source T1(a1, b1) PARTITION BY a1 ORDER BY b1
source T2(a2, b2) PARTITION BY a2
Is transformed into:

```sql
TableFunctionDataProcessor foo 
       PARTITION BY (a1, a2), ORDER BY combined_row_number 
       - Project 
           marker_1 <= IF(table1_row_number = combined_row_number, table1_row_number, CAST(null AS bigint)) 
           marker_2 <= IF(table2_row_number = combined_row_number, table2_row_number, CAST(null AS bigint)) 
           - Project 
               combined_row_number <= IF(COALESCE(table1_row_number, BIGINT '-1') > COALESCE(table2_row_number, BIGINT '-1'), table1_row_number, table2_row_number) 
               combined_partition_size <= IF(COALESCE(table1_partition_size, BIGINT '-1') > COALESCE(table2_partition_size, BIGINT '-1'), table1_partition_size, table2_partition_size) 
               - FULL Join 
                   [table1_row_number = table2_row_number OR 
                    table1_row_number > table2_partition_size AND table2_row_number = BIGINT '1' OR 
                    table2_row_number > table1_partition_size AND table1_row_number = BIGINT '1'] 
                   - Window [PARTITION BY a1 ORDER BY b1] 
                       table1_row_number <= row_number() 
                       table1_partition_size <= count() 
                           - source T1(a1, b1) 
                   - Window [PARTITION BY a2] 
                       table2_row_number <= row_number() 
                       table2_partition_size <= count() 
                           - source T2(a2, b2) 
 
```

### Collaborative Planning

The above planning rules treat the table function as a black box. That is very limiting. Optimizer rules for
projection pruning, filter pushdown, limit pushdown are all blocked if a table function is used mid-pipeline
in a SQL query.

The next phase of this project is to enhance the Table functions to participate in planning with the optimizer. Ref
- Accelerating BigData analytics with Collaborative Planning in Teradata Aster 6
  https://ieeexplore.ieee.org/abstract/document/7113378
- Not Black-Box Anymore! Enabling Analytics-Aware Optimizations in Teradata Vantage
  https://vldb.org/pvldb/vol14/p2959-eltabakh.pdf

For a start, its important for the Table function to expose to the rest of the optimizer the following properties:
- Distribution : If a table function receives input partitioned by a certain key, then its extremely useful to know is the same partitioning applicable to the output as well. This can help avoid adding exchanges which repartition data on the same distribution. If the partition key is retained, and there is a filter on a partition key following the table function, then this property implies that the partition filter can be pushed into the input. This sort of filter pushdown can bring a lot of improvements.

- Predicate pushdown : Pushing down predicates on partitioning keys has huge benefits in query processing. But in general, predicates on pass-through columns can be pushed down to the input as well. So if the table function can exchange this metadata with the optimizer that brings huge benefits.

- Column projections : If the table function can exchange with the optimizer which output columns are affected by which input columns, then the optimizer has the information to move columns projections from the output to the input side of the table function.

- Limit pushdown : The optimizer can provide the table function any limit information for improving its runtime processing.

- Sortedness : Knowing if the output rows of the table function are sorted can enable the optimizer to remove unnecessary sorts (complete or partial).

All these are transforms are applicable for logical rules. We have to put more thought about the participation of the
table function in cost based optimizations. The table function should participate in HBO out of the box.


### Built-in table functions

We are providing the list of built-in functions from Trino as a start of this work. We will mostly add Spark functions
as well as this list evolves

Quoting from the Trino documentation

sequence
```sql
Use the sequence table function to return a table with a single column sequential_number containing a sequence of bigint:
sequence(start => bigint, stop => bigint, step => bigint) -> table(sequential_number bigint)

start is the first element in the sequence. The default value is 0.
stop is the end of the range, inclusive. The last element in the sequence is equal to stop, or it is the last value 
within range, reachable by steps.
step is the difference between subsequent values. The default value is 1.

Example query:
SELECT *
FROM TABLE(sequence(
                start => 1000000,
                stop => -2000000,
                step => -3)); 

The result of the sequence table function might not be ordered. If required, enforce ordering in the enclosing query:
SELECT *
FROM TABLE(sequence(
                start => 0,
                stop => 100,
                step => 5)) 
ORDER BY sequential_number
```

exclude_columns
```sql
Use the exclude_columns table function to return a new table based on an input table table, with the exclusion of all columns specified in descriptor:

exclude_columns(input => table, columns => descriptor) → table
The argument input is a table or a query. The argument columns is a descriptor without types.

Example query using the orders table from the TPC-H dataset, provided by the TPC-H connector:
SELECT * FROM TABLE(exclude_columns(input => TABLE(orders),
                                    columns => DESCRIPTOR(clerk, comment)));

The table function is useful for queries where you want to return nearly all columns from tables with many columns.
You can avoid enumerating all columns, and only need to specify the columns to exclude.
The below SQL snippet gives an example of the invocation of a Table Valued function in a SQL query
```

query
```sql
query(varchar) -> table 

The query function allows you to query the underlying database directly. It requires syntax native to the underlying
database, because the full query is pushed down and processed in the source database. This can be useful for accessing
native features which are not available in Presto or for improving query performance in situations where running a
query natively may be faster. 

The native query passed to the underlying data source is required to return a table as a result set. Only the data
source performs validation or security checks for these queries using its own configuration. Presto does not perform
these tasks. Only use passthrough queries to read data. 

As a simple example, query the example catalog and select an entire table: 

SELECT 
  * 
FROM 
  TABLE( 
    example.system.query( 
      query => 'SELECT 
        * 
      FROM 
        tpch.nation' 
    ) 
  ); 
As a practical example, you can use the MODEL clause from Oracle SQL: 

SELECT 
  SUBSTR(country, 1, 20) country, 
  SUBSTR(product, 1, 15) product, 
  year, 
  sales 
FROM 
  TABLE( 

    example.system.query( 
      query => 'SELECT 
        * 
      FROM 
        sales_view 
      MODEL 
        RETURN UPDATED ROWS 
        MAIN simple_model 
        PARTITION BY country 
        MEASURES sales 
        RULES 
          (sales['Bounce', 2001] = 1000, 
          sales['Bounce', 2002] = sales['Bounce', 2001] + sales['Bounce', 2000], 
          sales['Y Box', 2002] = sales['Y Box', 2001]) 
      ORDER BY 
        country' 
    ) 
  ); 
```

## Other approaches considered
n/a. Table functions are well established in 2016 SQL Standard and the approach described
here is based on the implementations in other query engines viz Trino.

## Adoption plan
- This feature adds new SPI and SQL grammar as described above.
  There are no changes to existing APIs or behavior.

## Test plan
A new suite of unit and integration tests will be added to cover the new functionality.