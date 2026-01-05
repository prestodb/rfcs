# **RFC-0022 for Presto**

## Vector Similarity Search with JVector Integration in the Iceberg connector

Proposers

* Nivin C S
* Shijin K
* Dilli Babu Godari
* Nandakumar B

## Related Issues
N/A

## Summary

This RFC proposes integrating JVector, a high-performance vector similarity search library, into Presto to enable efficient Approximate Nearest Neighbor (ANN) search capabilities. This integration will allow users to perform semantic search, similarity matching, and other vector-based operations directly within Presto SQL queries, supporting modern AI/ML workloads including embeddings from Large Language Models (LLMs) and other machine learning models.

## Background

### Motivation

Vector embeddings have become fundamental to modern AI/ML applications, including:
- **Semantic Search**: Finding similar documents, images, or other content based on meaning rather than exact matches
- **Recommendation Systems**: Identifying similar items or users based on behavioral embeddings
- **Retrieval-Augmented Generation (RAG)**: Retrieving relevant context for LLM applications
- **Anomaly Detection**: Identifying outliers in high-dimensional feature spaces
- **Image and Video Search**: Finding visually similar content

Currently, Presto lacks native support for efficient vector similarity search. Users must either:
1. Export data to specialized vector databases (adding complexity and data movement)
2. Implement inefficient brute-force similarity calculations in SQL
3. Use external services, breaking the unified query interface

This RFC proposes native vector search capabilities in Presto, enabling users to:
- Store vector embeddings alongside structured data in Iceberg tables
- Perform efficient ANN searches using SQL
- Leverage Presto's distributed architecture for scalable vector search
- Maintain data locality and avoid unnecessary data movement

### Why JVector?

JVector (https://github.com/jbellis/jvector) is chosen for the following reasons:
- **Pure Java Implementation**: Seamless integration with Presto's Java codebase
- **High Performance**: Competitive with native implementations (HNSW algorithm)
- **Apache 2.0 License**: Compatible with Presto's licensing
- **Disk-Based Indexes**: Supports large-scale datasets without memory constraints
- **Multiple Similarity Functions**: Cosine, Euclidean, Dot Product
- **Active Development**: Well-maintained with regular updates

### Use Case Example

Consider a document search system with embeddings:

```sql
-- Create table with embeddings
CREATE TABLE documents (
    doc_id BIGINT,
    title VARCHAR,
    content VARCHAR,
    embedding ARRAY(REAL),  -- 768-dimensional embedding
    created_date DATE
) WITH (
    partitioning = ARRAY['created_date']
);

-- sample INSERT query including a realistic 768-dimension embedding example (shortened for readability)
INSERT INTO documents (
    doc_id,
    title,
    content,
    embedding,
    created_date
)
VALUES (
    1001,
    'Vector Search in Presto',
    'This document explains how vector similarity search works in Presto.',
    ARRAY[
        0.0123, -0.3456, 0.7891, 0.4567, -0.1123,
        0.9981, -0.2234, 0.3345, 0.6678, -0.5543
        -- ... continue until 768 REAL values
    ],
    DATE '2026-03-01'
);

-- Create a vector index using stored procedure
CALL system.create_vector_index(
    table_name => 'catalog.schema.table_name',
    column_name => 'embedding_column',
    index_name => 'embedding_idx',
    similarity_function => 'COSINE',  -- COSINE, EUCLIDEAN, DOT_PRODUCT (optional)
    m => 16,                          -- HNSW M parameter (default: 16) (optional)
    ef_construction => 100            -- HNSW ef_construction (default: 100) (optional)
);

-- Find top 10 most similar documents to a query
SELECT d.doc_id, d.title, ann.score
FROM documents d
JOIN TABLE(approx_nearest_neighbors(
    query_vector => ARRAY[0.1, 0.2, ..., 0.768],
    column_name => 'iceberg.default.documents.embedding',
    limit => 10
)) ann ON d.doc_id = ann.row_id
ORDER BY ann.score
LIMIT 10;
```

## Proposed Implementation

### Architecture Overview (index building flow)

The implementation follows a **begin-execute-finish pattern** with distributed execution: once user calls ` CALL CREATE_VEC_INDEX` statement, the coordinator validates parameters and generates splits for each partition. Each worker then processes its assigned splits to build the index. The coordinator aggregates results and finalizes the index creation.

```
┌─────────────────────────────────────────────────────────────┐
│                        Coordinator                          │
│  ┌─────────────────────────────────────────────────────┐    │
│  │  BEGIN PHASE                                        │    │
│  │  DistributedProcedure.begin()                       │    │
│  │  - Validate parameters (table, column, HNSW params) │    │
│  │  - Load table metadata and partitions               │    │
│  │  - Create BuildVectorIndexHandle                    │    │
│  │  - Return handle to engine                          │    │
│  └─────────────────────────────────────────────────────┘    │
│                             ↓                               │
│  ┌─────────────────────────────────────────────────────┐    │
│  │  SPLIT GENERATION                                   │    │
│  │  - Engine generates splits per partition            │    │
│  │  - Each split contains partition info + handle      │    │
│  │  - Distribute splits to available workers           │    │
│  └─────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────┘
                             │
                             │ Distribute Splits
                             ▼
         ┌───────────────────┴───────────────────┐
         │                   │                   │
┌────────▼─────────┐  ┌──────▼──────────┐  ┌────▼───────────┐
│   Worker 1       │  │   Worker 2      │  │   Worker N     │
│ ┌──────────────┐ │  │ ┌─────────────┐ │  │ ┌────────────┐ │
│ │Read Vectors  │ │  │ │Read Vectors │ │  │ │Read Vectors│ │
│ │from Part. 1  │ │  │ │from Part. 2 │ │  │ │from Part. N│ │
│ └──────────────┘ │  │ └─────────────┘ │  │ └────────────┘ │
│ ┌──────────────┐ │  │ ┌─────────────┐ │  │ ┌────────────┐ │
│ │Build HNSW    │ │  │ │Build HNSW   │ │  │ │Build HNSW  │ │
│ │Index         │ │  │ │Index        │ │  │ │Index       │ │
│ │(JVector)     │ │  │ │(JVector)    │ │  │ │(JVector)   │ │
│ └──────────────┘ │  │ └─────────────┘ │  │ └────────────┘ │
│ ┌──────────────┐ │  │ ┌─────────────┐ │  │ ┌────────────┐ │
│ │Write Index   │ │  │ │Write Index  │ │  │ │Write Index │ │
│ │to S3         │ │  │ │to S3        │ │  │ │to S3       │ │
│ └──────────────┘ │  │ └─────────────┘ │  │ └────────────┘ │
│ ┌──────────────┐ │  │ ┌─────────────┐ │  │ ┌────────────┐ │
│ │Write Fragment│ │  │ │Write Fragment│ │  │ │Write Fragment│
│ │to PageSink   │ │  │ │to PageSink  │ │  │ │to PageSink │ │
│ └──────────────┘ │  │ └─────────────┘ │  │ └────────────┘ │
└──────────────────┘  └─────────────────┘  └────────────────┘
         │                   │                   │
         └───────────────────┴───────────────────┘
                             │
                             │ Fragments (Slices)
                             ▼
┌─────────────────────────────────────────────────────────────┐
│                        Coordinator                          │
│  ┌─────────────────────────────────────────────────────┐    │
│  │  FINISH PHASE                                       │    │
│  │  DistributedProcedure.finish()                      │    │
│  │  - Collect fragments from all workers               │    │
│  │  - Deserialize BuildIndexFragment objects           │    │
│  │  - Verify all partitions completed successfully     │    │
│  │  - Aggregate statistics (total vectors, build time) │    │
│  │  - Create VectorIndexMetadata                       │    │
│  │  - Update table properties atomically               │    │
│  └─────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────┘
```

#### Index Building: SPI Design and Architecture

This section describes how connector plugins implement distributed vector index building using Presto's **DistributedProcedure** framework. Similar to the Table Function API, this leverages **existing Presto SPI interfaces** without introducing new ones.

#### SPI Overview: Distributed Procedure Interfaces

Index building uses Presto's `DistributedProcedure` mechanism for distributed execution with a **begin-finish pattern**. **No new SPI interfaces are introduced**. Connector plugins implement these interfaces to add index building capabilities.

**Key SPI Interfaces :**

| Interface | Package | Purpose | When Called |
|-----------|---------|---------|-------------|
| `DistributedProcedure` | `com.facebook.presto.spi.procedure` | Declares procedure signature with begin/finish lifecycle | Coordinator when procedure is invoked |
| `ConnectorDistributedProcedureHandle` | `com.facebook.presto.spi` | Marker interface for procedure execution context | Passed from begin() to finish() phase |
| `ConnectorMetadata` | `com.facebook.presto.spi.connector` | Has default implementations | procedures are self-contained |
| `ConnectorPageSinkProvider` | `com.facebook.presto.spi.connector` | Creates page sinks for workers to write index build results | Worker during index building |
| `ConnectorSplitManager` | `com.facebook.presto.spi.connector` | **Already implemented for table scans** - reused automatically | Coordinator generates splits for source TableScanNode |

**Interface Responsibilities:**

1. **DistributedProcedure**:
   - Defines procedure signature (name, parameters, type)
   - Implements `begin()` method with ALL validation and setup logic
   - Implements `finish()` method with result aggregation AND direct metadata updates
   - Two-phase execution: begin (coordinator) → workers execute → finish (coordinator)
   - **Directly updates Iceberg table properties** using Iceberg Table API (no metadata class involvement)

2. **ConnectorDistributedProcedureHandle** (Marker Interface):
   - Connector-specific implementation stores execution context
   - Example: `BuildVectorIndexHandle` stores table info, column name, HNSW parameters
   - Passed from `begin()` to `finish()` to maintain state

3. **ConnectorMetadata**:
   - Presto engine directly invokes procedure's `begin()` and `finish()` methods
   - **No routing or delegation logic needed**

4. **ConnectorPageSinkProvider**:
   - Creates page sinks for workers to write index build results
   - Workers write fragments (index metadata) back to coordinator
   - Handles serialization of build results (index paths, statistics)

5. **Split generation**:
   - For DistributedProcedure, the Presto engine internally handles split generation based on the table layout
   - The `begin()` method returns a `ConnectorDistributedProcedureHandle` which the engine uses to create the execution plan
   - Workers receive tasks through the standard Presto task distribution mechanism, not through explicit split generation

**Design Principle**: The `DistributedProcedure` follows a **TableScan-based execution model**:

**Execution Flow:**
1. **begin()**: Coordinator validates parameters, loads table, returns `ConnectorDistributedProcedureHandle`
2. **Plan Creation**: Presto engine creates execution plan:
   ```
   TableFinishNode
     └── CallDistributedProcedureNode
           └── TableScanNode (source table)
   ```
3. **Split Generation**: `TableScanNode` generates splits using standard table scan logic (via `ConnectorSplitManager.getSplits()` for the **source table**, not the procedure itself)
4. **Worker Execution**:
   - Workers receive splits from the TableScanNode
   - Read data from source table partitions
   - Process data through procedure logic (e.g., build index for partition)
   - Write results as fragments via `ConnectorPageSink`
5. **finish()**: Coordinator collects fragments, aggregates results, updates table properties
#### Index Building Execution Flow

Connector plugins implement the above SPI interfaces with index-building-specific logic:

**1. Distributed Procedure Declaration**

Declares the `create_vector_index` procedure with begin/finish lifecycle:

```java
@Description("Build a vector index for approximate nearest neighbor search")
public class BuildVectorIndexDistributedProcedure extends DistributedProcedure {
    
    private final IcebergMetadataFactory metadataFactory;
    
    /**
     * Constructor:
     * - Declares procedure signature with SQL parameters
     * - Injects IcebergMetadataFactory for creating metadata instances to load tables
     *
     * @param metadataFactory Factory for creating ConnectorMetadata instances
     */
    @Inject
    public BuildVectorIndexDistributedProcedure(IcebergMetadataFactory metadataFactory) {
        super(
            DistributedProcedureType.VECTOR_INDEX_BUILD,
            "system",
            "create_vector_index",
            ImmutableList.of(
                // SQL procedure parameters (what users pass in CALL statement)
                new Argument("table_name", VARCHAR),
                new Argument("column_name", VARCHAR),
                new Argument("index_name", VARCHAR),
                new Argument("m", BIGINT, false, 16),  // optional, default 16
                new Argument("ef_construction", BIGINT, false, 100),  // optional, default 100
                new Argument("similarity_function", VARCHAR, false, "COSINE")  // optional, default COSINE
            )
        );
        // Store injected dependency for use in begin() and finish()
        this.metadataFactory = requireNonNull(metadataFactory, "metadataFactory is null");
    }
    
    /**
     * Begin phase: ALL validation and setup logic here.
     *
     * Steps:
     * 1. Extract and validate procedure arguments
     * 2. Create metadata instance and load Iceberg table using IcebergUtil
     * 3. Validate column exists and has correct type (ARRAY(REAL))
     * 4. Verify no existing vector index on the column
     * 5. Return BuildVectorIndexHandle with execution context
     *
     * The returned handle is used by Presto to create a TableScanNode.
     * The TableScanNode will use the source table's TableHandle to generate splits.
     *
     * @param session The connector session
     * @param procedureContext Procedure execution context
     * @param tableLayoutHandle Handle to the target table
     * @param arguments Procedure arguments (table_name, column_name, etc.)
     * @return BuildVectorIndexHandle containing execution context with partition information
     */
    @Override
    public ConnectorDistributedProcedureHandle begin(
        ConnectorSession session,
        ConnectorProcedureContext procedureContext,
        ConnectorTableLayoutHandle tableLayoutHandle,
        Object[] arguments) 
    
    /**
     * Finish phase: Aggregate results and UPDATE TABLE PROPERTIES DIRECTLY.
     *
     * Steps:
     * 1. Collect and aggregate index metadata fragments from all workers
     * 2. Verify all partitions completed successfully
     * 3. Create metadata instance and load Iceberg table using IcebergUtil
     * 4. Update table properties atomically using Iceberg's native Table API
     *
     * @param session The connector session
     * @param procedureContext Procedure execution context
     * @param procedureHandle BuildVectorIndexHandle from begin phase
     * @param fragments Index metadata fragments from workers
     */
    @Override
    public void finish(
        ConnectorSession session,
        ConnectorProcedureContext procedureContext,
        ConnectorDistributedProcedureHandle procedureHandle,
        Collection<Slice> fragments) 
}
```

**2. Worker-Side Index Builder**

Worker component that builds indexes for assigned partitions:

```java
public class PartitionIndexBuilder {
    
    /**
     * Build vector index for a single partition.
     * Called on worker node for each partition split.
     *
     * Execution steps:
     * 1. Read vectors and row IDs from partition
     * 2. Create JVector GraphIndexBuilder with HNSW parameters
     * 3. Add vectors incrementally to build graph
     * 4. Create node→row ID mapping
     * 5. Write index and mapping files to S3
     * 6. Write build result to page sink (returned as fragment)
     *
     * @param procedureHandle BuildVectorIndexHandle with execution context
     * @param partitionKey Partition identifier
     * @return BuildIndexResult with index location and statistics
     */
    public BuildIndexResult buildPartitionIndex(
        BuildVectorIndexHandle procedureHandle,
        String partitionKey);
}
```

**3. Page Sink Provider for Worker Results**

Creates page sinks for workers to write index build results:

```java
public class IcebergPageSinkProvider implements ConnectorPageSinkProvider {
    
    /**
     * Create page sink for distributed procedure execution.
     * Workers use this to write index build results back to coordinator.
     *
     * The page sink:
     * 1. Receives index metadata from worker (index path, statistics)
     * 2. Serializes metadata as fragments (Slice objects)
     * 3. Returns fragments to coordinator for aggregation
     *
     * @param transactionHandle The transaction handle
     * @param session The connector session
     * @param procedureHandle BuildVectorIndexHandle from begin phase
     * @param pageSinkContext Page sink context
     * @return ConnectorPageSink for writing build results
     */
    @Override
    public ConnectorPageSink createPageSink(
        ConnectorTransactionHandle transactionHandle,
        ConnectorSession session,
        ConnectorDistributedProcedureHandle procedureHandle,
        PageSinkContext pageSinkContext);
}
```

**4. Connector-Specific Data Structures**

These implement SPI marker interfaces:

```java
/**
 * Implements ConnectorDistributedProcedureHandle
 * Stores execution context from begin() to finish() phase
 */
@JsonSerialize
@JsonDeserialize
public class BuildVectorIndexHandle implements ConnectorDistributedProcedureHandle {
    private final String catalogName;
    private final String schemaName;
    private final String tableName;
    private final String columnName;
    private final String indexName;
    private final int dimension;
    private final int m;
    private final int efConstruction;
    private final String similarityFunction;
    private final List<String> partitionKeys;
    
    // JSON serialization methods
}

/**
 * Fragment data written by workers
 * Serialized as Slice and collected by coordinator
 */
public class BuildIndexFragment {
    private final String partitionKey;
    private final String indexPath;
    private final String mappingPath;
    private final long vectorCount;
    private final long buildTimeMs;
    private final long indexSizeBytes;
    
    // Serialization to/from Slice
    public Slice toSlice();
    public static BuildIndexFragment fromSlice(Slice slice);
}

/**
 * Aggregated metadata for the complete index
 * Created in finish() phase and stored in table properties
 */
public class VectorIndexMetadata {
    private final String indexName;
    private final String columnName;
    private final String similarityFunction;
    private final int dimension;
    private final int m;
    private final int efConstruction;
    private final Map<String, String> partitionIndexPaths;
    private final long creationTimestamp;
    private final long totalVectors;
    private final long totalBuildTimeMs;
}
```

### Architecture Overview (ANN Search flow)

The implementation follows a distributed architecture leveraging Presto's coordinator-worker model: once user triger the `approx_nearest_neighbors` function, the coordinator will generate the query plan and dispatch the task to workers. The workers will load the index and perform the search, then return the results to the coordinator. The coordinator will then merge the results and return the final result to the user.

```
┌─────────────────────────────────────────────────────────────┐
│                        Coordinator                          │
│  ┌─────────────────────────────────────────────────-───┐    │
│  │  Query Planning & Optimization                      │    │
│  │  - Parse TVF call                                   │    │
│  │  - Resolve vector index metadata                    │    │
│  │  - Generate splits for partitioned indexes          │    │
│  │  - Plan result aggregation                          │    │
│  └───────────────────────────────────────────────-─────┘    │
│  ┌────────────────────────────────────────────────-────┐    │
│  │  Top-K Aggregator                                   │    │
│  │  - Merge results from workers                       │    │
│  │  - find global top k result                         │    │
│  │  - Return final results                             │    │
│  └─────────────────────────────────────────────────-───┘    │
└─────────────────────────────────────────────────────────────┘
                            │
                            │ Distribute Splits
                            ▼
        ┌───────────────────┴───────────────────┐
        │                   │                   │
┌───────▼────────┐  ┌───────▼────────┐  ┌──────▼─────────┐
│   Worker 1     │  │   Worker 2     │  │   Worker N     │
│ ┌────────────┐ │  │ ┌────────────┐ │  │ ┌────────────┐ │
│ │Index Cache │ │  │ │Index Cache │ │  │ │Index Cache │ │
│ └────────────┘ │  │ └────────────┘ │  │ └────────────┘ │
│ ┌────────────┐ │  │ ┌────────────┐ │  │ ┌────────────┐ │
│ │ANN Search  │ │  │ │ANN Search  │ │  │ │ANN Search  │ │
│ │Partition 1 │ │  │ │Partition 2 │ │  │ │Partition N │ │
│ └────────────┘ │  │ └────────────┘ │  │ └────────────┘ │
│ Local Top-K    │  │ Local Top-K    │  │ Local Top-K    │
└────────────────┘  └────────────────┘  └────────────────┘
        │                   │                   │
        └───────────────────┴───────────────────┘
                            │
                            ▼
                    Results to Coordinator
```

### ANN Search SPI Design and Architecture

This section clarifies the **SPI boundaries** and describes how connector plugins integrate with Presto's engine to implement vector similarity search using the Table-Valued Function (TVF) framework.

#### SPI Overview: Existing Presto Interfaces

This RFC leverages **existing Presto SPI interfaces** from the Table-Valued Function (TVF) framework . **No new SPI interfaces are introduced**. Connector plugins implement these interfaces to add vector similarity search capabilities.

**Key SPI Interfaces:**

| Interface | Package | Purpose | When Called |
|-----------|---------|---------|-------------|
| `ConnectorTableFunction` | `com.facebook.presto.spi.function.table` | Declares table function signature and implements analysis logic | Coordinator during query analysis |
| `TableFunctionProcessorProvider` | `com.facebook.presto.spi.function.table` | Factory that creates split processors | Worker when split is assigned |
| `TableFunctionSplitProcessor` | `com.facebook.presto.spi.function.table` | Executes search on a single split/partition | Worker during split execution |
| `ConnectorSplitManager` | `com.facebook.presto.spi.connector` | Generates splits for parallel execution using table function handle | Coordinator during split generation |
| `ConnectorTableFunctionHandle` | `com.facebook.presto.spi.function.table` | Marker interface for analysis context | Passed from coordinator to workers |
| `ConnectorSplit` | `com.facebook.presto.spi` | Marker interface for split data  | Distributed to workers |

**Interface Responsibilities:**

1. **ConnectorTableFunction**:
   - Defines function signature (name, arguments, return type)
   - Implements `analyze()` method to validate arguments, load index metadata, and return output schema
   - Returns `TableFunctionAnalysis` with `ConnectorTableFunctionHandle` containing execution context

2. **TableFunctionProcessorProvider**:
   - Factory pattern for creating processors
   - One processor instance created per split on each worker
   - Receives `ConnectorTableFunctionHandle` and `ConnectorSplit` to create processor

3. **TableFunctionSplitProcessor**:
   - Core execution logic for vector search
   - `process()` method called repeatedly until returns `Finished` state
   - Returns `TableFunctionProcessorState` with result pages or completion status
   - States: `Processed` (with/without page), `Finished`, or `Blocked` (for async I/O)

4. **ConnectorSplitManager**:
   - Generates `ConnectorSplit` objects for parallel execution
   - For vector search: creates one split per partition
   - Each split contains partition-specific information (index path, query vector, limit)

5. **ConnectorTableFunctionHandle** (Marker Interface):
   - Connector-specific implementation stores analysis-time context
   - Example: `VectorSearchHandle` stores query vector, column name, limit, index metadata

6. **ConnectorSplit** (Marker Interface):
   - Connector-specific implementation stores split-specific data
   - Example: `VectorSearchSplit` stores partition key, index path, mapping path, query vector

**Design Principle**: The SPI follows the **leaf operator pattern** where the table function acts as a data source (not a transformation). The engine handles split distribution, parallel execution, and result aggregation automatically.

#### Search Execution Flow

Connector plugins implement the above SPI interfaces with vector-search-specific logic:

**1. Table Function Declaration**

Declares the `approx_nearest_neighbors` function with its signature and behavior:

```java
@Description("Approximate Nearest Neighbors search for vector similarity")
public class ApproxNearestNeighborsFunction extends AbstractConnectorTableFunction {
    
    /**
     * Constructor declares the table function signature using Presto SPI.
     *
     * Function Signature:
     * - Name: "approx_nearest_neighbors"
     * - Schema: "system"
     *
     * Input Arguments (no TABLE argument):
     * 1. query_vector: ARRAY(REAL) - The query vector to search for
     * 2. column_name: VARCHAR - Fully qualified column name (catalog.schema.table.column)
     * 3. limit: BIGINT - Maximum number of nearest neighbors to return (top-K)
     *
     * Return Type: DescribedTableReturnTypeSpecification
     * - Describes a table with the following schema:
     *   - row_id: BIGINT - Row identifier from the source table
     *   - score: REAL - Similarity score (lower is more similar for distance metrics)
     *
     * Implementation Details:
     * - Uses AbstractConnectorTableFunction as base class
     * - Return type is constructed using DescribedTableReturnTypeSpecification
     * - Column descriptors are created with ColumnMetadata for each output column
     * - No TABLE input parameter - column_name is used to resolve table and index
     */
    public ApproxNearestNeighborsFunction() { ... }
    
    /**
     * Analyze the table function call and return output schema.
     * Validates arguments, loads index metadata, creates execution handle.
     *
     * @param session The connector session
     * @param transaction The transaction handle
     * @param arguments Function arguments (query_vector, column_name, limit)
     * @return TableFunctionAnalysis with output schema and VectorSearchHandle
     */
    @Override
    public TableFunctionAnalysis analyze(
        ConnectorSession session,
        ConnectorTransactionHandle transaction,
        Map<String, Argument> arguments);
}
```

**2. Processor Provider Implementation**

Factory that creates split processors for each partition:

```java
public class IcebergTableFunctionProcessorProvider implements TableFunctionProcessorProvider {
    
    /**
     * Create a processor for the table function split.
     * Called once per split on each worker node.
     *
     * @param session The connector session
     * @param functionHandle VectorSearchHandle from analysis phase
     * @param split VectorSearchSplit containing partition info
     * @return ANNSplitProcessor instance for this split
     */
    @Override
    public TableFunctionSplitProcessor getTableFunctionSplitProcessor(
        ConnectorSession session,
        ConnectorTableFunctionHandle functionHandle,
        ConnectorSplit split);
}
```

**3. Split Processor Implementation**

Executes ANN search for a single partition:

```java
public class ANNSplitProcessor implements TableFunctionSplitProcessor {
    
    /**
     * Process the split and return results.
     * Called repeatedly until FINISHED state is returned.
     *
     * Execution steps:
     * 1. Load index from cache (memory-mapped file)
     * 2. Execute JVector ANN search
     * 3. Translate node IDs to row IDs
     * 4. Build result page with (row_id, score) columns
     *
     * @param split VectorSearchSplit to process
     * @return TableFunctionProcessorState (Processed with page, or Finished)
     */
    @Override
    public TableFunctionProcessorState process(ConnectorSplit split);
}
```

**4. Split Manager Extension**

Generates vector search splits for parallel execution:

```java
public class IcebergSplitManager implements ConnectorSplitManager {
    
    /**
     * Generate splits for vector search using table function handle.
     * Creates one VectorSearchSplit per partition based on index metadata.
     *
     * The function handle (VectorSearchHandle) contains:
     * - Table information (catalog, schema, table name)
     * - Index metadata (partition paths, index locations)
     * - Query parameters (vector, column, limit)
     *
     * @param transaction The transaction handle
     * @param session The connector session
     * @param function VectorSearchHandle containing table and index information from analysis phase
     * @return ConnectorSplitSource producing VectorSearchSplit objects (one per partition)
     */
    @Override
    public ConnectorSplitSource getSplits(
        ConnectorTransactionHandle transaction,
        ConnectorSession session,
        ConnectorTableFunctionHandle function);
}
```

**5. Connector-Specific Data Structures**

These implement SPI marker interfaces:

```java
/**
 * Implements ConnectorTableFunctionHandle
 * Stores context from analysis phase for execution phase
 */
@JsonSerialize
@JsonDeserialize
public class VectorSearchHandle implements ConnectorTableFunctionHandle {
    private final float[] queryVector;
    private final String columnName;
    private final long limit;
    private final VectorIndexMetadata indexMetadata;
    private final String catalogName;
    private final String schemaName;
    private final String tableName;
    private final SchemaTableName schemaTableName;
    private final List<String> partitionKeys;
    private final Map<String, String> partitionIndexPaths;
    
    // JSON serialization methods
}

/**
 * Implements ConnectorSplit
 * Represents a partition-specific search task
 */
@JsonSerialize
@JsonDeserialize
public class VectorSearchSplit implements ConnectorSplit {
    private final String partitionKey;
    private final String indexPath;
    private final String mappingPath;
    private final float[] queryVector;
    private final long limit;
    
    // JSON serialization methods
}
```

**Top-K Result Aggregation at the Coordinator**

The `approx_nearest_neighbors` table function produces an unordered stream of`(row_id, score)` rows from each worker. Global top-K aggregation is achieved using Presto’s existing `TopN` planning and execution mechanisms.

The planner inserts `TopN` operators based on standard SQL semantics:

- The table function exposes `score` as an orderable column
- The user query applies `ORDER BY score LIMIT k`
- During plan optimization, Presto introduces:
  - Partial `TopN` operators on workers
  - A final `TopN` operator on the coordinator


### SQL API

#### Index Creation

```sql
-- Create a vector index using stored procedure
CALL system.create_vector_index(
    table_name => 'catalog.schema.table_name',
    column_name => 'embedding_column',
    index_name => 'embedding_idx',
    similarity_function => 'COSINE',  -- COSINE, EUCLIDEAN, DOT_PRODUCT (optional)
    m => 16,                          -- HNSW M parameter (default: 16) (optional)
    ef_construction => 100            -- HNSW ef_construction (default: 100) (optional)
);

-- Drop an index
CALL system.drop_vector_index(
    table_name => 'catalog.schema.table_name',
    index_name => 'embedding_idx'
);

-- Refresh an index (rebuild with latest data)
CALL system.refresh_vector_index(
    table_name => 'catalog.schema.table_name',
    index_name => 'embedding_idx'
);

-- List indexes for a table
SELECT * FROM system.vector_indexes 
WHERE table_name = 'catalog.schema.table_name';
```

#### Vector Search

```sql
-- Basic ANN search
SELECT * FROM TABLE(
    approx_nearest_neighbors(
        query_vector => ARRAY[0.1, 0.2, 0.3, ..., 0.768],
        column_name => 'catalog.schema.documents.embedding',
        limit => 10
    )
);

-- Join with original table to get full rows
SELECT d.doc_id, d.title, d.content, ann.score
FROM documents d
JOIN TABLE(
    approx_nearest_neighbors(
        query_vector => ARRAY[0.1, 0.2, 0.3, ..., 0.768],
        column_name => 'catalog.schema.documents.embedding',
        limit => 10
    )
) ann ON d.row_id = ann.row_id
ORDER BY ann.score;

-- Search with additional filters (partition pruning)
SELECT d.doc_id, d.title, ann.score
FROM documents d
JOIN TABLE(
    approx_nearest_neighbors(
        query_vector => ARRAY[0.1, 0.2, 0.3, ..., 0.768],
        column_name => 'catalog.schema.documents.embedding',
        limit => 10
    )
) ann ON d.row_id = ann.row_id
WHERE d.created_date >= DATE '2024-01-01'
ORDER BY ann.score;
```

### Metadata Integration

#### Index Metadata Storage

Vector index metadata is stored in the Iceberg metadata layer in snapshot summary:

```json
{
  "format-version": 2,
  "snapshots": [{
    "snapshot-id": 1234567890,
    "summary": {
      "vector-index-column": "embedding",
      "vector-index-location": "s3://bucket/.../index.hnsw",
      "vector-mapping-location": "s3://bucket/.../mapping.bin",
      "vector-index-snapshot-id": 1234567890,
      "vector-dimension": 768,
      "vector-count": 1000000,
      "vector-similarity-function": "COSINE",
      "vector-index-m": 16,
      "vector-index-ef-construction": 100,
      "vector-index-build-timestamp": 1704470400000
    }
  }]
}
```
**Benefits:**
- ✅ Versioning: Index tied to specific snapshot
- ✅ Consistency: Validate index via snapshot ID comparison
- ✅ Cleanup: Expired snapshots auto-clean index references
- ✅ No External Dependencies: No separate metadata store

#### Index Discovery

**IcebergVectorIndexMetadataStore**

Stores and retrieves vector index metadata from Iceberg table properties. Implements `VectorIndexMetadataStore` interface.

```java
public interface VectorIndexMetadataStore {
    
    /**
     * Retrieve index metadata for a column from Iceberg table properties.
     * Reads the "vector_indexes" property and parses JSON metadata.
     *
     * @param session The connector session
     * @param tableName The table to query
     * @param columnName The indexed column
     * @return Optional containing metadata if index exists
     */
    Optional<VectorIndexMetadata> getIndexMetadata(
        ConnectorSession session,
        SchemaTableName tableName,
        String columnName);
    
    /**
     * Save index metadata to Iceberg table properties.
     * Serializes metadata to JSON and commits as table property update.
     *
     * @param session The connector session
     * @param tableName The table to update
     * @param metadata The index metadata to save
     */
    void saveIndexMetadata(
        ConnectorSession session,
        SchemaTableName tableName,
        VectorIndexMetadata metadata);
}
```

**Note on Metadata Store Usage:**

The `VectorIndexMetadataStore` interface is primarily used for **read operations** during query planning and index discovery. For **write operations** (creating/updating indexes), the `BuildVectorIndexDistributedProcedure` directly updates Iceberg table properties using the Iceberg Table API in its `finish()` method

The metadata store's `saveIndexMetadata()` method can be implemented as a convenience wrapper around this pattern if needed for other use cases, but it's not required for the distributed procedure implementation.

### Partitioning Strategy

#### Partition-Aware Indexing

For partitioned tables, create separate indexes per partition:

```
Table: documents (partitioned by created_date)
├── Partition: created_date=2024-01-01
│   ├── Data files: s3://bucket/table/created_date=2024-01-01/*.parquet
│   └── Index: s3://bucket/table/.vector_index/embedding_idx/created_date=2024-01-01/
│       ├── index.hnsw
│       └── mapping.bin
├── Partition: created_date=2024-01-02
│   ├── Data files: s3://bucket/table/created_date=2024-01-02/*.parquet
│   └── Index: s3://bucket/table/.vector_index/embedding_idx/created_date=2024-01-02/
│       ├── index.hnsw
│       └── mapping.bin
└── ...
```

#### Partition Pruning

Leverage Presto's partition pruning for efficient searches:

```sql
-- Only search partitions matching the filter
SELECT d.doc_id, d.title, ann.score
FROM documents d
JOIN TABLE(
    approx_nearest_neighbors(
        query_vector => ARRAY[...],
        column_name => 'catalog.schema.documents.embedding',
        limit => 10
    )
) ann ON d.row_id = ann.row_id
WHERE d.created_date BETWEEN DATE '2024-01-01' AND DATE '2024-01-31'
ORDER BY ann.score;

-- Presto will only load and search indexes for January 2024 partitions
```

#### Dynamic Partition Management

**PartitionAwareIndexManager**

Manages indexes for individual partitions, supporting incremental index updates as partitions are added or removed.

```java
public interface PartitionAwareIndexManager {
    
    /**
     * Build index for a newly added partition.
     * Called when new data is inserted into a partition.
     *
     * @param session The connector session
     * @param tableName The table containing the partition
     * @param indexName The index to update
     * @param newPartition Information about the new partition
     */
    void buildPartitionIndex(
        ConnectorSession session,
        SchemaTableName tableName,
        String indexName,
        PartitionInfo newPartition);
    
    /**
     * Remove index for a dropped partition.
     * Cleans up index files and updates metadata.
     *
     * @param session The connector session
     * @param tableName The table containing the partition
     * @param indexName The index to update
     * @param droppedPartition Information about the dropped partition
     */
    void dropPartitionIndex(
        ConnectorSession session,
        SchemaTableName tableName,
        String indexName,
        PartitionInfo droppedPartition);
}
```

### Performance Optimizations

#### 1. Index Caching

**VectorIndexCache**

Worker-side cache for loaded indexes and mappings using any caching mechanism. Reduces repeated downloads from S3/HDFS.

```java
public interface VectorIndexCache {
    
    /**
     * Get or load an index from cache.
     * Downloads from storage if not cached.
     *
     * @param indexPath Storage path to index file
     * @return Loaded OnDiskGraphIndex
     */
    OnDiskGraphIndex getIndex(String indexPath);
    
    /**
     * Get or load a node-to-row mapping from cache.
     *
     * @param indexPath Storage path to mapping file
     * @return Loaded NodeRowIdMapping
     */
    NodeRowIdMapping getMapping(String indexPath);
    
    /**
     * Invalidate cached entries for an index.
     * Called when index is rebuilt.
     *
     * @param indexPath Path to invalidate
     */
    void invalidate(String indexPath);
}
```

#### 2. Adaptive Search Parameters

**AdaptiveSearchParameterOptimizer**

Dynamically adjusts HNSW search parameters (ef_search) based on index size, K value, and target recall to balance speed and accuracy.

```java
public interface AdaptiveSearchParameterOptimizer {
    
    /**
     * Calculate optimal ef_search parameter for HNSW search.
     * Uses heuristics based on index size and target recall.
     *
     * @param indexSize Number of vectors in the index
     * @param k Number of results requested
     * @param targetRecall Desired recall rate (0.0 to 1.0)
     * @return Optimal ef_search value
     */
    int calculateOptimalEfSearch(
        int indexSize,
        int k,
        double targetRecall);
}
```

### Configuration

#### Connector Configuration

Add to `catalog/iceberg.properties`:

```properties
# Enable vector similarity search
iceberg.vector-search.enabled=true

# Index cache size (in MB)
iceberg.vector-search.index-cache-size=1024

# Maximum number of parallel index builds
iceberg.vector-search.max-parallel-builds=4

# Default similarity function
iceberg.vector-search.default-similarity-function=COSINE

# Default HNSW parameters
iceberg.vector-search.default-m=16
iceberg.vector-search.default-ef-construction=100
iceberg.vector-search.default-ef-search=50
```

#### Session Properties

```sql
-- Set search parameters for current session
SET SESSION iceberg.vector_search_ef_search = 100;
SET SESSION iceberg.vector_search_timeout = '30s';
```

### Error Handling

```java
public enum VectorSearchErrorCode implements ErrorCodeSupplier {
    VECTOR_INDEX_NOT_FOUND(1, EXTERNAL),
    VECTOR_DIMENSION_MISMATCH(2, USER_ERROR),
    VECTOR_INDEX_CORRUPTED(3, EXTERNAL),
    VECTOR_SEARCH_TIMEOUT(4, EXTERNAL),
    INVALID_SIMILARITY_FUNCTION(5, USER_ERROR);
    
    private final ErrorCode errorCode;
    
    VectorSearchErrorCode(int code, ErrorType type) {
        this.errorCode = new ErrorCode(
            code + 0x0600_0000,  // Vector search error range
            name(),
            type);
    }
    
    @Override
    public ErrorCode toErrorCode() {
        return errorCode;
    }
}
```

#### Incremental Index Updates

**IncrementalIndexUpdater**

Supports adding new vectors to existing indexes without full rebuild (future enhancement).

```java
public interface IncrementalIndexUpdater {
    
    /**
     * Add new vectors to an existing index.
     * Loads existing index, builds incremental index, and merges.
     *
     * @param indexPath Path to existing index
     * @param newVectors New vectors to add
     * @param newRowIds Corresponding row IDs
     */
    void addVectors(
        Path indexPath,
        List<float[]> newVectors,
        List<Long> newRowIds);
}
```

## Adoption Plan

### Phase 1: Core Infrastructure 

**Goals**:
- Implement basic vector index metadata layer
- Integrate JVector library
- Create index building infrastructure
- Implement single-partition index support
- Implement `approx_nearest_neighbors` table function
- Create search execution infrastructure

**Deliverables**:
- `VectorIndexMetadata` and `VectorIndexManager` interfaces
- `VectorIndexBuilder` for single partition
- Metadata storage in Iceberg properties
- Basic stored procedures for index management
- `ApproxNearestNeighborsFunction` implementation
- `ANNSplitProcessor` for search execution

**Success Criteria**:
- Can create and query indexes on non-partitioned tables
- Index metadata persisted and discoverable
- Basic unit tests passing
- Can execute vector searches via SQL
- Results are accurate.

### Phase 2: Distributed Execution

**Goals**:
- Implement partitioned index support
- Enable distributed index building
- Implement basic result aggregation
- Implement parallel search execution

**Deliverables**:
- `DistributedVectorIndexBuilder`
- Partition-aware index management
- Parallel search across workers
- `TopKAggregator` for result merging
- Integration with Presto's TVF framework

**Success Criteria**:
- Can handle partitioned tables with 100+ partitions
- Index building parallelized across workers
- Search executes in parallel
- Integration tests passing

### Phase 3: Optimizations

**Goals**:
- Implement index caching
- Performance tuning

**Deliverables**:
- `VectorIndexCache` implementation
- Performance benchmarks

**Success Criteria**:
- Can handle 1B+ vectors with partitioning
- Memory usage optimized

### Phase 4: Advanced Features

**Goals**:
- Incremental index updates
- Production hardening
- Add adaptive search parameters

**Deliverables**:
- Incremental index update support
- Production monitoring and metrics
- Adaptive ef_search calculation

**Success Criteria**:
- Can update indexes without full rebuild
- Monitoring dashboards available

### Backward Compatibility

- No breaking changes to existing Presto APIs
- New feature is opt-in via configuration
- Existing queries continue to work unchanged
- Index creation is explicit (not automatic)


## Monitoring and Observability

### Metrics

```java
public class VectorSearchMetrics {
    
    // Index building metrics
    private final Counter indexBuildsStarted;
    private final Counter indexBuildsCompleted;
    private final Counter indexBuildsFailed;
    private final Timer indexBuildDuration;
    
    // Search metrics
    private final Counter searchesExecuted;
    private final Timer searchLatency;
    private final Histogram searchResultSize;
    private final Counter cacheHits;
    private final Counter cacheMisses;
    
    // Resource metrics
    private final Gauge indexCacheSize;
    private final Gauge indexCacheMemoryUsage;
    private final Counter indexLoads;
    private final Timer indexLoadDuration;
}
```

### Logging

```java
// Index building
log.info("Building vector index for table %s, column %s, partition %s",
    tableName, columnName, partition);
log.info("Index built successfully: %d vectors, dimension %d, took %dms",
    vectorCount, dimension, duration);

// Search execution
log.debug("Executing vector search: table=%s, limit=%d, partitions=%d",
    tableName, limit, partitionCount);
log.debug("Search completed: found %d results in %dms",
    resultCount, duration);

// Errors
log.error("Failed to build index for partition %s: %s",
    partition, error.getMessage(), error);
```

## Test Plan

To ensure the JVector integration works correctly and maintains backward compatibility, the following testing strategy will be implemented:

### 1. Backward Compatibility

* Ensure that existing CI tests pass for Iceberg connector where vector search is not enabled.
* Verify that tables without vector indexes continue to work as before.
* Confirm that existing queries are not affected when vector search feature is disabled.

### 2. Unit Tests

Add comprehensive unit tests for vector search components in the Iceberg connector:

* **Index Building Tests**:
  - Test index creation with various HNSW parameters (M, ef_construction)
  - Test different similarity functions (COSINE, EUCLIDEAN, DOT_PRODUCT)
  - Test index building with different vector dimensions (128, 384, 768, 1536)
  - Test error handling for invalid parameters

* **Search Execution Tests**:
  - Test ANN search with various K values
  - Test search result ordering (by score)
  - Test node-to-row ID mapping correctness

* **Aggregation Tests**:
  - Test top-K aggregation from multiple workers
  - Test global top-K correctness with overlapping results

### 3. Integration Tests

Add end-to-end integration tests covering:

* **Basic Vector Search**:
  - Create table with vector column
  - Create vector index
  - Execute `approx_nearest_neighbors()` function
  - Verify results match expected top-K

* **Partitioned Table Search**:
  - Create partitioned table with vectors
  - Create indexes for all partitions
  - Execute search with partition pruning
  - Verify only relevant partitions are searched

* **SQL Operations**:
  - Test `CREATE_VECTOR_INDEX` stored procedure
  - Test `DROP_VECTOR_INDEX` stored procedure
  - Test `REFRESH_VECTOR_INDEX` stored procedure
  - Test querying `system.vector_indexes` table

* **Join Operations**:
  - Test joining ANN results with original table

* **Edge Cases**:
  - Test with duplicate vectors
  - Test with zero vectors
  - Test with very high/low dimensional vectors
  - Test with NaN/Infinity values (should error gracefully)

### 4. Error Handling Tests

Test error scenarios:

* Invalid query vector dimensions
* Missing or corrupted index files
* Insufficient memory for index loading
* Network failures during S3 access
