# RFC-0023 for Presto: Syntax and SPI for Presto ANN Vector Search

Proposers
- Zhichen Xu
- Ge Gao
- Ke Wang
- Feilong Liu
- Amit Dutta
- Sreeni Viswanadha
- James Gill

## Summary

This RFC proposes the syntax and SPI to natively support Approximate Nearest Neighbor (ANN) vector search within Presto.

## Background

Vector similarity search is fundamental to AI/ML applications (RAG, recommendations, anomaly detection). Today, Presto users must either:
- Use external vector databases (data duplication, sync overhead)
- Use brute-force `CROSS JOIN` + distance (O(nxm), prohibitive at scale)

Native ANN support enables single-system analytics+vector workloads with transparent optimization.

## User Journeys

##### Journey 1: Create index for specific partitions now

"I want to create a vector index for ds='2026-01-01' through '2026-01-03'."

```sql
CREATE VECTOR INDEX vector_index ON TABLE candidates_table(id, embedding) WITH (
    index_type = 'ivf_rabitq4',
    distance_metric = 'cosine',
    index_options = 'nlist=100000,nb=8',
    partitioned_by = ARRAY['ds']
) UPDATING FOR
    ds BETWEEN '2026-01-01' AND '2026-01-03'
```

Behavior: Indices are built immediately for partitions ds=2026-01-01, 2026-01-02 and 2026-01-03.

##### Journey 2: Create index metadata without populating any partition

"I need the index definition created now, but the index should only build when I instruct."

```sql
CREATE VECTOR INDEX vector_index ON TABLE candidates_table(id, embedding) WITH (
    index_type = 'ivf_rabitq4',
    distance_metric = 'cosine',
    index_options = 'nlist=100000,nb=8',
    partitioned_by = ARRAY['ds']
)
```

##### Journey 3: Populate vector index for specific partitions now

"I need to populate the vector index for 2026-01-04."

```sql
UPDATE VECTOR INDEX vector_index WHERE
    ds = '2026-01-04'
```

##### Journey 4: Search with implicit index lookup

"I want to run ANN search automatically using the index registered on my table."

```sql
SELECT
    queries.id,
    VECTOR_SEARCH(
        candidates.id,
        candidates.embedding,
        queries.embedding,
        'cosine',
        10
    )
FROM candidates CROSS JOIN queries
WHERE
    candidates.ds BETWEEN '2026-01-01' AND '2026-01-04'
    AND queries.ds = '2026-01-04'
GROUP BY
    queries.id
```

Behavior: Optimizer uses the registered index automatically.


## Proposed Changes

### Create a Vector Index

#### Create Partitioned Index

Partitioned index builds a standalone index for each base table partition. Most of the prod base tables are anticipated to fall in this user scenario, where:
- The individual table partition is large.
- A new partition is added every day or every hour.
- Past partitions are occasionally backfilled.

```sql
CREATE VECTOR INDEX vector_index ON TABLE candidates_table(id, embedding) WITH (
    index_type = 'ivf_rabitq4',
    distance_metric = 'cosine',
    index_options = 'nlist=100000,nb=8',
    partitioned_by = ARRAY['ds']
) UPDATING FOR
    ds BETWEEN '2026-01-01' AND '2026-01-03'
```

**ON `TABLE candidates_table(id, embedding)` clause**
- `embedding` is the embedding column to create the index on.
- `id` is the unique identifier column that the user needs to provide.
    - Presto will not enforce the uniqueness and will leave it to the user.
    - Alternatively, use the hidden column $row_id for id and remove the requirement from users.

**`UPDATING FOR` clause**
- For each of the index partitions mapped to the range, an index will be created.
    - For the mapping of index partition to base table partition, refer to the table property partitioned_by.

**Table properties**
- `partitioned_by`: Partition keys of the index.
    - For each of the index partitions, an index will be created.
    - Must be a subset of the base table partition keys.
- `index_type`: Index type. Different ConnectorFactory could define a different enum of index types to support.
- `distance_metric`: Distance metric, ex. cosine similarity, inner product, L2 distance.
- `index_options`: Options specific to the index type, in varchar type that is decoded to Map afterwards.

It is allowed to omit the `UPDATING FOR` clause as below. In this case, we will only create the vector index (metadata) but not populate the per-partition indices. Users use UPDATE VECTOR INDEX below to populate the per-partition indices.

```sql
CREATE VECTOR INDEX vector_index ON candidates_table(id, embedding) WITH (
    index_type = 'ivf_rabitq4',
    distance_metric = 'cosine',
    index_options = 'nlist=100000,nb=8',
    partitioned_by = ARRAY['ds']
)
```

#### Create Unpartitioned Index

Unpartitioned index builds a single index for the base table over the partitions specified by UPDATING FOR clause. It is used for the scenario when:
- The base table is small enough, given the range of partitions specified by UPDATING FOR clause.
- The required latency and resource for populating the single index is acceptable to the user.
- The user is interested in an ad-hoc index creation and ANN search.

`UPDATING FOR` clause is used as below to specify the range of partition data to build the single index over.

```sql
CREATE VECTOR INDEX vector_index ON candidates_table(id, embedding) WITH (
    index_type = 'ivf_rabitq4',
    distance_metric = 'cosine',
    index_options = 'nlist=100000,nb=8'
) UPDATING FOR
    ds BETWEEN '2026-01-01' AND '2026-01-03'
```

When no `UPDATING FOR` clause is specified, we will only create the vector index (metadata) but not populate the single index.

```sql
CREATE VECTOR INDEX vector_index ON candidates_table(id, embedding) WITH (
    index_type = 'ivf_rabitq4',
    distance_metric = 'cosine',
    index_options = 'nlist=100000,nb=8'
)
```

#### Execution and SPI

It will be a general and flexible interface if we make the execution similar to CTAS with a function `create_vector_index()` introduced. Plugins could have their own implementation of the *create_vector_index()* function.

- Add an AST `CreateVectorIndex` node that extends `Statement`.

```java
public class CreateTableAsSelect extends Statement { }
```

- Add a branch to `LogicalPlanner.planStatementWithoutOutput()` for `CreateVectorIndex` AST node and create the logical plan nodes.

```
TableFinishNode
    |__ TableWriteNode
            |__ RelationPlan (create_vector_index())
```

- Add `CreateVectorIndexReference` that extends `TableWriterNode.WriterTarget`

```java
public static class CreateVectorIndexReference extends WriterTarget { }
```

Logically it is equivalent to:

```sql
CREATE TABLE vector_index WITH (
    partitioned_by = ARRAY['ds']
) AS SELECT
    create_vector_index(
        id,
        embedding,
        'ivf_rabitq4',      -- index_type
        'cosine',           -- distance_metric
        'nlist=100000,nb=8' -- index_options
    )
FROM candidates_table
WHERE
    ds BETWEEN '2026-01-01' AND '2026-01-03'
```

#### Metadata

- Introduce to `MetadataManager`
    - `createVectorIndex()`
    - `beginCreateVectorIndex()`
    - `finishCreateVectorIndex()`
- Introduce to ConnectorMetadata
    - `createVectorIndex()`
    - `beginCreateVectorIndex()`
    - `finishCreateVectorIndex()`
- HiveMetadata implementation
    - Similar to CTAS, plus a boolean table parameter `is_vector_index`.

### Update Vector Index

PrestoSQL interface to rebuild the index. After the index is created and has metadata stored in Metastore, UPDATE VECTOR INDEX could be:
- Manually triggered by the user.

#### Update Partitioned Index

```sql
UPDATE VECTOR INDEX vector_index WHERE
    ds = '2026-01-04'
```

Update only the index partition mapped to the base table partition ds = '2026-01-04'.

```sql
UPDATE VECTOR INDEX vector_index
```

Update all the indices corresponding to all the base table partitions.

#### Update Unpartitioned Index

```sql
UPDATE VECTOR INDEX vector_index WHERE
    ds = '2026-01-04'
```

Rebuild the single unpartitioned index based on the WHERE clause of the original vector index definition stored in Metadata.


### Vector Search

#### Search with a new function VECTOR_SEARCH()

```sql
SET SESSION vector_search_index = 'di:vector_index:nprobe=50';

SELECT
    queries.id,
    VECTOR_SEARCH(
        candidates.id,
        candidates.embedding,
        queries.embedding,
        'cosine',
        10
    )
FROM candidates CROSS JOIN queries
WHERE
    candidates.ds BETWEEN '2026-01-01' AND '2026-01-04'
    AND queries.ds = '2026-01-04'
GROUP BY
    queries.id
```

#### OSS Implementation

OSS implementation will be exact vector search and implement a ConnectorPlanOptimizer to rewrite to a plan with MAX_BY().

```sql
SELECT
    queries.id,
    MAX_BY(
        candidates.id,
        COSINE_SIMILARITY(candidates.embedding, queries.embedding),
        10
    )
FROM candidates CROSS JOIN queries
WHERE
    candidates.ds BETWEEN '2026-01-01' AND '2026-01-04'
    AND queries.ds = '2026-01-01'
GROUP BY
    queries.id
```

#### Metadata

Rewrite needs the following metadata

`vector_index` The vector index for search and its schema.
- Introduce metastore.getVectorIndex() API.
    - Functionality-wise, similar to metastore.getTable().

### Table Function

Table function provides an alternative syntax which can be more intuitive for simple use cases.

```sql
SELECT * FROM TABLE(vector_search(
    DOCUMENTS => TABLE(candidates_table),
    DOC_VECTOR_COL => 'embedding',
    QUERIES => TABLE(queries_table),
    QUERY_VECTOR_COL => 'query_embedding',
    TOP_K => 10,
    DOC_ID => 'id',
    QUERY_ID => 'id',
    DISTANCE_METRIC => 'cosine'
))
```

#### Execution and SPI

Our proposal is to introduce generic SPI nodes: VectorSearchNode and VectorSearchOptions, which together provide an abstract interface for vector search. This abstract interface will only include mandatory parameters, allowing actual implementations to extend it.

For the open-source version (OSS), we will provide a default implementation that uses exact search as the aggregation function version does.


## References

### Google BigQuery

**CREATE VECTOR INDEX:**

```sql
CREATE VECTOR INDEX my_index ON my_table(embedding_column) OPTIONS(
  index_type = 'IVF',                    -- or 'TREE_AH' (ScaNN)
  distance_type = 'COSINE',              -- 'COSINE', 'EUCLIDEAN', 'DOT_PRODUCT'
  ivf_options = '{"num_lists": 1000}'
);
```

**VECTOR_SEARCH Table Function:**

```sql
SELECT * FROM VECTOR_SEARCH(
  TABLE my_embeddings,
  'embedding_column',
  (SELECT embedding FROM queries),
  top_k => 10,
  distance_type => 'COSINE'
);
```

- **Key Features:** Table function style, named parameters (=>), subquery for batch queries, auto-index usage
- **Limitation:** No pre-filtering before search
- **Docs:** [BigQuery Vector Search Intro](https://cloud.google.com/bigquery/docs/vector-search-intro)

### Snowflake

**Vector Data Type:**

```sql
CREATE TABLE embeddings (
  id INT,
  embedding VECTOR(FLOAT, 768)
);
```

**Similarity Search (Scalar Functions):**

```sql
SELECT id, VECTOR_COSINE_SIMILARITY(embedding, query_vec) AS score FROM embeddings ORDER BY score DESC LIMIT 10;
SELECT id, VECTOR_L2_DISTANCE(embedding, query_vec) AS dist FROM embeddings ORDER BY dist ASC LIMIT 10;
SELECT id, VECTOR_INNER_PRODUCT(embedding, query_vec) AS score FROM embeddings ORDER BY score DESC LIMIT 10;
```

- **Key Features:** Native VECTOR(FLOAT, N) type, scalar functions with ORDER BY...LIMIT, implicit/automatic indexing, pre-filtering supported
- **Docs:** [Snowflake Vector Data Type](https://docs.snowflake.com/en/sql-reference/data-types-vector)

### Databricks

**CREATE VECTOR SEARCH INDEX:**

```sql
CREATE VECTOR SEARCH INDEX my_index ON TABLE my_catalog.my_schema.my_table (embedding_column) USING DELTA_SYNC WITH (
  pipeline_type = 'TRIGGERED',    -- or 'CONTINUOUS'
  embedding_dimension = 768
);
```

**Search via SQL:**

```sql
SELECT * FROM ai_query(
  'my_vector_search_endpoint',
  query => 'search text',
  num_results => 10
);
```

- **Key Features:** Managed sync pipeline (auto-updates), Delta Lake native, multi-interface (SQL/SDK/REST), hybrid search
- **Note:** Primary interface is Python SDK; SQL via ai_query wrapper
- **Docs:** [Databricks Vector Search](https://docs.databricks.com/en/generative-ai/vector-search.html)

### Comparison Matrix

| Feature              | Presto (Proposed)         | BigQuery         | Snowflake         | Databricks         |
|----------------------|--------------------------|------------------|-------------------|--------------------|
| Search Style         | Table function + Aggregate| Table function   | Scalar + ORDER BY | Managed endpoint   |
| Index DDL            | CREATE VECTOR INDEX...WITH(...) | ...OPTIONS(...) | Automatic         | ...USING DELTA_SYNC|
| Partitioned Index    | Native                   |                  |                   |                    |
| Auto-Index Lifecycle | auto_index=TRUE          |                  | N/A               | CONTINUOUS         |
| Pre-filtering        |                          |                  |                   |                    |
| Batch Queries        | Queries table            | Subquery         |                   | SDK                |
| Distance Metrics     | cosine, l2, inner_product| COSINE, EUCLIDEAN, DOT_PRODUCT | All 3 functions | COSINE, L2, DOT_PRODUCT |


## Design Rationale

1. **BigQuery-aligned syntax**  
   CREATE VECTOR INDEX and VECTOR_SEARCH table function provide familiar semantics

2. **Partition-aware indexing**  
   Unique to Presto; critical for large-scale warehouse workloads with daily partitions

3. **Pre-filtering advantage**  
   WHERE before search enables partition pruning (BigQuery lacks this)

4. **Dual API**  
   Table function for simplicity + aggregate function for SQL composability

5. **Ephemeral index**  
   Ad-hoc search without persistent index (similar to Snowflake's automatic approach)
