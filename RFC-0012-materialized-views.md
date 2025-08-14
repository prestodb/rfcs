# **RFC0012 for Presto: Materialized Views**

## Materialized Views Architecture

Proposers

* Tim Meehan

## Related Issues

Related issues may include Github issues, PRs or other RFCs.

* Existing Hive Materialized Views implementation in presto-hive module
* Proposed Iceberg Materialized Views support

## Summary

This RFC proposes a redesign of Presto's Materialized Views architecture to move from a Hive-centric, analysis-phase-heavy implementation to a connector-agnostic, planning-phase-driven architecture that better supports modern table formats like Apache Iceberg. The design emphasizes a minimal SPI with advanced features like stitching handled through the existing `ConnectorPlanOptimizerProvider` mechanism rather than complex SPI methods.

## Background

The current Materialized Views implementation in Presto was designed specifically for Hive and has several architectural limitations that prevent modern connectors from leveraging their advanced capabilities (detailed in the Problems section below). Modern table formats like Apache Iceberg provide advanced features (hidden partitioning, snapshot isolation, incremental scans) that could enable much more efficient materialized view implementations, but the current architecture cannot leverage these capabilities.

### Goals

* Create a connector-agnostic materialized views framework
* Move complex MV logic from analysis phase to planning phase
* Enable connectors to implement efficient, format-specific refresh strategies
* Support incremental refresh patterns natively
* Provide a complete Iceberg connector implementation leveraging snapshot-based change tracking

### Non-goals

* Backward compatibility with existing Hive MV implementation (it's undocumented and unused)
* Supporting materialized views across different connectors (cross-connector MVs)
* Automatic view selection/routing for arbitrary queries

## Current Implementation and How It Works

The existing materialized views implementation is located primarily in the presto-hive module and follows this flow:

### 1. Metadata Storage
- MVs are stored as Hive tables with special properties
- View definition stored in table properties as serialized SQL
- Base table references tracked in metadata

### 2. Query Analysis Phase
During analysis, the system:
- Parses the MV definition from metadata
- Constructs the refresh query by analyzing the view definition
- Determines partitioning alignment between MV and base tables
- Calculates freshness based on partition modification times

### 3. Refresh Execution (Hive-Specific)
- Full refresh: Drops and recreates the entire MV table
- Partial refresh: Attempts to update only modified partitions (detected via Hive partition modification timestamps)
- Uses INSERT OVERWRITE for partition updates
- **Limitation**: Can only detect partition-level changes through Hive metastore's `transient_lastDdlTime`

### 4. Query Rewriting
- Simple pattern matching to detect if a query can use an MV
- Limited to exact matches or simple projections
- No cost-based selection between multiple MVs

## Problems with Current Implementation

### 1. Hive-Centric Architecture

The current implementation is built around Hive's partition replacement semantics, making it unusable for most other connectors. The framework requires INSERT OVERWRITE support and restricts WHERE clauses to partition columns only through the `validRefreshColumns` field. This architecture prevents JDBC connectors like PostgreSQL and MySQL from using materialized views since they don't support partition-level INSERT OVERWRITE operations. It also forces Apache Iceberg to fake Hive partition semantics rather than leveraging its superior snapshot-based change tracking, and prevents any connector from doing row-level incremental refresh on non-partition columns (e.g., `WHERE customer_id = 123`).

### 2. Heavy Analysis Phase

Performing MV logic during analysis causes early binding problems where refresh decisions are made before the optimizer runs. For example, the system cannot choose between full versus incremental refresh based on the actual amount of changed data, nor can it decide whether to stitch fresh and stale partitions versus recomputing everything based on cost estimates. Predicate validation against `validRefreshColumns` happens during analysis, restricting refresh to only partition-level operations and preventing row-level refresh that modern formats like Iceberg could support. The WHERE clause blindly determines which partitions to overwrite without checking if they actually have stale data, potentially rewriting unchanged partitions unnecessarily.

### 3. Limited Refresh Capabilities

The current implementation is constrained by Hive's architecture and can only perform incremental refresh at partition granularity, unable to detect or refresh individual row changes within partitions. Non-partitioned tables must always do full refresh with no incremental support. Freshness detection relies on Hive partition modification times which don't exist in modern formats like Iceberg that use snapshot-based tracking. Additionally, there's no concept of "acceptable staleness" or grace periods that would allow users to balance freshness requirements with refresh costs.

### 4. Inconsistent Access Control Model

Views can use either definer rights (execute with creator's permissions) or invoker rights (execute with querying user's permissions). The current implementation mixes both models inconsistently. During creation, it checks if the creating user has access to base tables following an invoker rights pattern. During refresh, it checks if the MV owner has access (suggesting definer rights) but then executes with the invoker's identity (invoker rights), creating a confusing hybrid model. Users need SELECT permission on the MV itself for queries, which is standard. The problem is that the refresh operation validates against owner permissions but executes with invoker permissions. This creates security and operational issues where both the MV owner and the refresh invoker need base table permissions, making the permission check on the owner effectively meaningless since execution uses invoker's identity. There's no true definer rights support, preventing controlled access to aggregated data, and this differs from both regular Presto views (which properly support both modes) and industry standard (most analytical databases default to definer rights for MVs).

## Proposed Changes

### 1. Simplified SPI Methods for Materialized Views

Keep the SPI minimal and focused on basic operations, with advanced stitching handled through `ConnectorPlanOptimizerProvider`:

```java
public interface ConnectorMetadata {
    // Core MV operations
    void createMaterializedView(
        ConnectorSession session,
        SchemaTableName viewName,
        ConnectorMaterializedViewDefinition definition,
        boolean replace,
        boolean ignoreExisting);
    
    void dropMaterializedView(
        ConnectorSession session,
        SchemaTableName viewName);
    
    Optional<ConnectorMaterializedViewDefinition> getMaterializedView(
        ConnectorSession session,
        SchemaTableName viewName);
    
    // Simple freshness check - no stitching information
    MaterializedViewFreshness getMaterializedViewFreshness(
        ConnectorSession session,
        SchemaTableName viewName);
    
    // Check if connector handles refresh natively (e.g., PostgreSQL)
    default boolean delegateMaterializedViewRefreshToConnector(
        ConnectorSession session,
        SchemaTableName viewName) {
        return false;  // Most connectors use Presto's refresh
    }
    
    // Perform native refresh (only called if delegated)
    default CompletableFuture<?> refreshMaterializedView(
        ConnectorSession session,
        SchemaTableName viewName) {
        throw new PrestoException(NOT_SUPPORTED, 
            "Connector does not support delegated refresh");
    }
    
    // Get refresh layout for Presto-managed refresh
    // (only called if delegateMaterializedViewRefreshToConnector returns false)
    default MaterializedViewRefreshLayout getRefreshLayout(
        ConnectorSession session,
        SchemaTableName viewName,
        MaterializedViewStatus currentStatus,
        Constraint<String> refreshConstraint) {
        // Default: full refresh
        return MaterializedViewRefreshLayout.fullRefresh();
    }
}
```

### 2. Move Refresh Logic to Planning Phase

Instead of rewriting REFRESH to INSERT during analysis, handle it during planning:

```java
// Keep REFRESH as first-class statement through analysis
public class RefreshMaterializedView extends Statement {
    private final QualifiedName name;
    private final Optional<Expression> where;  // Refresh constraint
}

// New planner components
public class RefreshMaterializedViewPlanner {
    public PlanNode planRefresh(
        RefreshMaterializedView statement,
        PlannerContext context) {
        
        // Get MV definition
        MaterializedViewDefinition mvDef = metadata.getMaterializedView(
            statement.getName());
        
        // Check if connector handles refresh natively
        if (metadata.delegateMaterializedViewRefreshToConnector(
                session, statement.getName())) {
            // Create a delegated refresh node (executed by connector)
            return new DelegatedRefreshNode(statement.getName());
        }
        
        // Presto-managed refresh: get current status from connector
        MaterializedViewStatus status = metadata.getMaterializedViewStatus(
            session, statement.getName(), TupleDomain.all());
        
        // Ask connector for refresh layout
        MaterializedViewRefreshLayout layout = metadata.getRefreshLayout(
            session,
            statement.getName(), 
            status,
            convertWhereToConstraint(statement.getWhere()));
        
        // Build appropriate query plan based on layout
        if (layout.getIncrementalFilter().isPresent()) {
            return buildIncrementalRefreshPlan(mvDef, layout);
        } else {
            return buildFullRefreshPlan(mvDef);
        }
    }
    
    private PlanNode buildIncrementalRefreshPlan(
            MaterializedViewDefinition mvDef,
            MaterializedViewRefreshLayout layout) {
        
        // Parse original MV query
        Query mvQuery = sqlParser.parse(mvDef.getOriginalSql());
        
        // Build plan with delta filters
        PlanNode basePlan = buildPlanForQuery(mvQuery);
        
        // Apply incremental filters from connector
        PlanNode deltaDataPlan = applyIncrementalFilters(
            basePlan, 
            layout.getIncrementalFilter().get());
        
        // Create refresh execution node
        return new RefreshMaterializedViewNode(
            deltaDataPlan,
            mvDef.getStorageTable(),
            RefreshType.INCREMENTAL);
    }
}
```

### 3. Connector-Driven Refresh

Allow connectors to provide information needed for refresh without exposing internal strategy:

```java
public class MaterializedViewRefreshLayout {
    // What data needs to be refreshed (computed by connector)
    private final Optional<TupleDomain<String>> incrementalFilter;
    private final Optional<String> message; // User-facing message about refresh
    private final Optional<Long> estimatedRowsToProcess; // For cost-based decisions
    
    // The connector internally decides refresh strategy based on the filter
}
```

### 4. Improved Freshness Model

```java
public class MaterializedViewStatus {
    public enum State {
        FULLY_MATERIALIZED,   // Completely up-to-date
        PARTIALLY_MATERIALIZED, // Some data is stale
        NOT_MATERIALIZED,      // No data materialized
        UNKNOWN               // Cannot determine status
    }
    
    private final State state;
    private final Optional<Instant> lastRefreshTime;
    private final Optional<TupleDomain<String>> materializedDomain;
    private final Optional<Duration> staleness;
}
```

### 5. Clear Separation of Responsibilities

**Presto Runtime Handles:**
- SQL parsing and validation
- Query planning and general optimization (join reordering, predicate pushdown, etc.)

**Connector Handles:**
- MV metadata storage
- Freshness tracking
- Refresh strategy selection
- MV-specific optimizations (e.g., stitching fresh/stale partitions, incremental scans, snapshot-based change detection)

### 6. Consistent Access Control Model

Align materialized views with regular views for access control:

```java
// Modified StatementAnalyzer.visitRefreshMaterializedView
@Override
protected Scope visitRefreshMaterializedView(
        RefreshMaterializedView node, 
        Optional<Scope> scope) {
    
    MaterializedViewDefinition view = getMaterializedView(viewName);
    
    // Determine access control mode (same as regular views)
    Identity refreshIdentity;
    AccessControl refreshAccessControl;
    
    if (view.getOwner().isPresent() && 
        !view.getOwner().get().equals(session.getIdentity().getUser())) {
        // Definer mode - use owner's permissions
        refreshIdentity = new Identity(
            view.getOwner().get(), 
            Optional.empty(),
            session.getIdentity().getExtraCredentials());
        refreshAccessControl = new ViewAccessControl(accessControl);
    } else {
        // Invoker mode - use session user's permissions  
        refreshIdentity = session.getIdentity();
        refreshAccessControl = accessControl;
    }
    
    // Build session with appropriate identity
    Session refreshSession = buildOwnerSession(
        session, 
        Optional.of(refreshIdentity.getUser()),
        metadata.getSessionPropertyManager(),
        viewName.getCatalogName(),
        view.getSchema());
    
    // Analyze refresh query with proper access control
    StatementAnalyzer queryAnalyzer = new StatementAnalyzer(
        analysis,
        metadata,
        sqlParser,
        refreshAccessControl,  // Use appropriate access control
        refreshSession,        // Use appropriate session
        warningCollector);
    
    // ... rest of refresh logic
}
```

This approach enables definer rights for materialized views (consistent with regular views in Presto), while also supporting invoker rights when needed.

## How Iceberg Will Handle It

The Iceberg connector will leverage these new APIs to implement efficient materialized views:

### 1. Storage Model

```java
public class IcebergMaterializedView {
    // View component (stored in catalog)
    private final String viewName;
    private final String viewDefinition;
    private final Schema schema;
    
    // Table component (Iceberg table)
    private final Table storageTable;
    
    // Base table snapshot ID when MV was last fully refreshed
    // For partial refresh, individual partition freshness is derived from
    // Iceberg's manifest metadata rather than stored explicitly
    private final long baseTableSnapshotId;
}
```

### 2. Enhanced Partitioning Support

Iceberg will enhance the existing `partitioned_by` property to support partition transforms while maintaining backward compatibility:

```sql
-- Current Hive approach (still supported)
CREATE MATERIALIZED VIEW sales_summary
WITH (
    partitioned_by = ARRAY['year', 'month']  -- Simple column names
)
AS SELECT year, month, SUM(amount) FROM sales GROUP BY year, month;

-- Enhanced Iceberg approach (same property, new capabilities)
CREATE MATERIALIZED VIEW sales_summary
WITH (
    partitioned_by = ARRAY['year(sale_date)', 'month(sale_date)', 'bucket(16, customer_id)']
)
AS SELECT sale_date, customer_id, SUM(amount) FROM sales GROUP BY sale_date, customer_id;
```

The enhanced `partitioned_by` property accepts both simple column names for backward compatibility (`'region'`, `'product_id'`) and new partition transform expressions (`'year(ts)'`, `'bucket(N, col)'`, `'truncate(N, str)'`). This enables Iceberg's hidden partitioning where the MV storage table can be partitioned differently from the visible columns.

### 3. Per-Partition Snapshot-Based Freshness (Derived from Iceberg Metadata)

```java
@Override
public MaterializedViewFreshness getMaterializedViewFreshness(
    ConnectorSession session,
    SchemaTableName viewName) {
    
    IcebergMaterializedView mv = catalog.loadMaterializedView(viewName);
    Table mvStorageTable = mv.getStorageTable();
    Table baseTable = catalog.loadTable(mv.getBaseTableName());
    
    // For non-partitioned MVs, use simple snapshot comparison
    if (mvStorageTable.spec().isUnpartitioned()) {
        long currentSnapshot = baseTable.currentSnapshot().snapshotId();
        long mvSnapshot = mv.getBaseTableSnapshotId();
        
        if (currentSnapshot == mvSnapshot) {
            return MaterializedViewFreshness.fresh(Instant.now());
        } else {
            return MaterializedViewFreshness.stale(
                Instant.now(), 
                format("Base table changed: snapshot %d -> %d", 
                       mvSnapshot, currentSnapshot));
        }
    }
    
    // For partitioned MVs, derive per-partition freshness from Iceberg metadata
    // rather than storing it explicitly
    Map<StructLike, Long> partitionLastModified = 
        derivePartitionSnapshotsFromMetadata(mvStorageTable);
    
    Set<StructLike> stalePartitions = new HashSet<>();
    for (Map.Entry<StructLike, Long> entry : partitionLastModified.entrySet()) {
        StructLike partition = entry.getKey();
        long mvPartitionSnapshot = entry.getValue();
        
        // Check if base table has changes in this partition since MV was written
        boolean hasChanges = baseTable.newIncrementalAppendScan()
            .fromSnapshot(mvPartitionSnapshot)
            .toSnapshot(baseTable.currentSnapshot().snapshotId())
            .filter(partitionPredicate(partition))
            .planFiles()
            .iterator()
            .hasNext();
        
        if (hasChanges) {
            stalePartitions.add(partition);
        }
    }
    
    if (!stalePartitions.isEmpty()) {
        // Note: We don't return partition names to engine - it doesn't understand them
        // The getRefreshLayout() method will convert partitions to TupleDomain
        return MaterializedViewFreshness.stale(
            Instant.now(),
            format("%d partitions have stale data", stalePartitions.size()));
    } else {
        return MaterializedViewFreshness.fresh(Instant.now());
    }
}

/**
 * Derive per-partition snapshot information from Iceberg's manifest metadata.
 * This avoids storing redundant state in the MV definition.
 */
private Map<StructLike, Long> derivePartitionSnapshotsFromMetadata(Table mvTable) {
    Map<StructLike, Long> partitionSnapshots = new HashMap<>();
    
    // Scan manifest entries to find when each partition was last written
    for (ManifestFile manifest : mvTable.currentSnapshot().allManifests()) {
        try (ManifestReader<DataFile> reader = ManifestFiles.read(manifest, mvTable.io())) {
            for (ManifestEntry<DataFile> entry : reader) {
                DataFile file = entry.file();
                StructLike partition = file.partition();
                long snapshotId = entry.snapshotId();
                
                // Track the latest snapshot that modified this partition
                partitionSnapshots.merge(partition, snapshotId, Math::max);
            }
        }
    }
    
    return partitionSnapshots;
}
```

### 4. Constraint-Based Incremental Refresh

The new architecture supports constraint-based refresh where predicates from the `REFRESH MATERIALIZED VIEW WHERE` clause are passed to connectors for partition alignment detection.

#### How Iceberg Detects Changed Partitions

Unlike Hive's timestamp-based approach, Iceberg uses its incremental scan API to detect exactly what changed:

```java
// Iceberg knows exactly what data files were added/modified between snapshots
private Set<StructLike> detectChangedPartitions(
        Table baseTable,
        long lastRefreshSnapshot,
        long currentSnapshot) {
    
    Set<StructLike> changedPartitions = new HashSet<>();
    
    // Incremental scan finds ALL files added/modified between snapshots
    IncrementalAppendScan scan = baseTable.newIncrementalAppendScan()
        .fromSnapshot(lastRefreshSnapshot)
        .toSnapshot(currentSnapshot);
    
    // Each file tells us which partition it belongs to
    for (FileScanTask task : scan.planFiles()) {
        DataFile file = task.file();
        StructLike partition = file.partition();
        changedPartitions.add(partition);
    }
    
    return changedPartitions;
}

// This is FAR more accurate than Hive's approach because:
// 1. Based on actual data changes, not metadata timestamps
// 2. Tracks changes at file level, not just partition level  
// 3. Can detect deletes and updates, not just appends
// 4. Works even for non-partitioned tables
```

```java
@Override
public MaterializedViewRefreshLayout getRefreshLayout(
    ConnectorSession session,
    SchemaTableName viewName,
    MaterializedViewStatus status,
    Constraint<String> refreshConstraint) {
    
    if (status.getState() == FULLY_MATERIALIZED) {
        return MaterializedViewRefreshLayout.noRefreshNeeded();
    }
    
    IcebergMaterializedView mv = catalog.loadMaterializedView(viewName);
    Table storageTable = catalog.loadTable(mv.getStorageTableName());
    
    // Analyze if constraint aligns with partitions
    PartitionAlignment alignment = analyzePartitionAlignment(
        refreshConstraint, 
        storageTable.spec());
    
    switch (alignment.getType()) {
        case FULLY_ALIGNED:
            // Constraint matches partition boundaries - efficient refresh
            // Connector internally knows this is partition-aligned but doesn't expose it
            return new MaterializedViewRefreshLayout(
                Optional.of(buildPartitionFilter(alignment)),
                Optional.of(alignment.getAffectedPartitions()),
                Optional.empty());
                
        case NOT_ALIGNED:
            // Phase 1: Throw error for non-aligned predicates
            // Phase 2 (future): Support row-level refresh with MERGE
            throw new PrestoException(
                INVALID_REFRESH_CONSTRAINT,
                format("Refresh predicate does not align with partitions. " +
                       "View is partitioned by: %s", 
                       formatPartitionSpec(storageTable.spec())));
                       
        case EMPTY:
            // No constraint - full refresh
            // Connector internally knows this is full refresh but doesn't expose it
            return new MaterializedViewRefreshLayout(
                Optional.empty(),
                Optional.empty(),
                Optional.of("Full refresh required - no constraint provided"));
    }
}

// Example: Converting Iceberg partitions to TupleDomain for engine
private TupleDomain<String> buildPartitionFilter(
        PartitionAlignment alignment,
        Table storageTable) {
    
    TupleDomain<String> unionedFilter = TupleDomain.none();
    
    // Iceberg knows partitions internally (e.g., struct{year=2024, month=1})
    for (StructLike partition : alignment.getAffectedPartitions()) {
        // Convert Iceberg partition to predicates engine understands
        Map<String, Domain> domains = new HashMap<>();
        
        // Example: partition struct{year=2024, month=1} becomes:
        // year = 2024 AND month = 1
        PartitionSpec spec = storageTable.spec();
        for (PartitionField field : spec.fields()) {
            Object value = partition.get(field.sourceId());
            domains.put(
                field.name(),
                Domain.singleValue(typeFor(field), value));
        }
        
        TupleDomain<String> partitionFilter = TupleDomain.withColumnDomains(domains);
        unionedFilter = unionedFilter.union(partitionFilter);
    }
    
    return unionedFilter;
}

// Example with hidden partitioning:
// MV created with: partitioned_by = ARRAY['month(sale_date)', 'bucket(16, customer_id)']
// MV has columns: sale_date (date), customer_id (bigint), total_amount (decimal)
// Storage table internally partitioned by: month transform + hash bucket
//
// REFRESH MATERIALIZED VIEW sales_summary 
// WHERE sale_date BETWEEN DATE '2024-01-15' AND DATE '2024-02-15';
//
// 1. Constraint passed to connector: TupleDomain{sale_date=[2024-01-15, 2024-02-15]}
// 2. Iceberg detects actual changes via incremental scan:
//    - Base table had updates for customer_ids: 42, 156, 891
//    - These hash to buckets: 2, 7, 11
//    - Changes occurred on: 2024-01-20, 2024-01-25, 2024-02-10
// 3. Connector returns ONLY affected partition combinations:
//    TupleDomain.union(
//      TupleDomain{sale_date=[2024-01-15, 2024-01-31], $bucket=2},  // customer 42
//      TupleDomain{sale_date=[2024-01-15, 2024-01-31], $bucket=7},  // customer 156
//      TupleDomain{sale_date=[2024-02-01, 2024-02-15], $bucket=11}  // customer 891
//    )
// 4. Engine refreshes only 3 partitions instead of 32, treating $bucket as opaque
//    The 29 unchanged partition combinations are preserved in the MV
```

#### MERGE for Row-Level Refresh (Future Phase)

1. **Phase 1 (Initial)**: Support only partition-aligned refresh
    - Clear error messages when predicates don't align
    - Efficient for time-series and naturally partitioned data

2. **Phase 2 (Future)**: Add row-level refresh when MERGE is available
    - Use Iceberg's RowDelta API for fine-grained updates
    - Support arbitrary predicates in refresh WHERE clause

### 5. Advanced Stitching via ConnectorPlanOptimizerProvider

When a materialized view is partially stale, advanced connectors like Iceberg can provide sophisticated "stitching" that combines fresh data from the MV with updated data from base tables. Rather than adding complexity to the SPI, this is achieved through the existing `ConnectorPlanOptimizerProvider` mechanism.

#### MaterializedViewScanNode with Lazy Planning

A new plan node type with separate SPI interface and runtime implementation:

```java
// In presto-spi - read-only interface for connectors
public interface MaterializedViewScan {
    TableHandle getTable();
    List<Symbol> getOutputSymbols();
    Map<Symbol, ColumnHandle> getAssignments();
    MaterializedViewDefinition getDefinition();
    
    // Lazy planning - connector can request the planned MV query
    PlanNode getDefinitionPlan();
    
    // Join analysis utilities (computed by engine)
    JoinAnalysis getJoinAnalysis();
    boolean hasOuterJoins();
    boolean isColumnFromOuterJoin(Symbol column);
}

// In presto-main - concrete implementation
public class MaterializedViewScanNode extends PlanNode implements MaterializedViewScan {
    private final TableHandle table;
    private final List<Symbol> outputSymbols;
    private final Map<Symbol, ColumnHandle> assignments;
    private final MaterializedViewDefinition definition;
    private final Supplier<PlanNode> lazyDefinitionPlan;
    private final Supplier<JoinAnalysis> lazyJoinAnalysis;
    
    // Created by LogicalPlanner with all necessary context
    public MaterializedViewScanNode(
            PlanNodeId id,
            TableHandle table,
            List<Symbol> outputSymbols,
            Map<Symbol, ColumnHandle> assignments,
            MaterializedViewDefinition definition,
            SqlParser sqlParser,
            Analysis analysis,
            Session session,
            Metadata metadata,
            PlanNodeIdAllocator idAllocator,
            SymbolAllocator symbolAllocator) {
        super(id);
        this.table = table;
        this.outputSymbols = outputSymbols;
        this.assignments = assignments;
        this.definition = definition;
        
        // Lazy planning using concrete planner components
        this.lazyDefinitionPlan = Suppliers.memoize(() -> {
            // Parse and analyze the MV SQL
            Query query = sqlParser.createStatement(definition.getOriginalSql());
            Analyzer analyzer = new Analyzer(session, metadata, sqlParser,
                    accessControl, queryExplainer, parameters);
            Analysis mvAnalysis = analyzer.analyze(query);
            
            // Plan it
            LogicalPlanner planner = new LogicalPlanner(session, idAllocator, 
                    metadata, typeAnalyzer);
            return planner.plan(mvAnalysis);
        });
        
        // Lazy join analysis
        this.lazyJoinAnalysis = Suppliers.memoize(() -> {
            PlanNode plan = getDefinitionPlan();
            return JoinAnalyzer.analyzeJoins(plan);  // Engine utility
        });
    }
    
    @Override
    public PlanNode getDefinitionPlan() {
        return lazyDefinitionPlan.get();
    }
    
    @Override
    public JoinAnalysis getJoinAnalysis() {
        return lazyJoinAnalysis.get();
    }
    
    @Override
    public boolean hasOuterJoins() {
        return getJoinAnalysis().hasOuterJoins();
    }
    
    @Override
    public boolean isColumnFromOuterJoin(Symbol column) {
        return getJoinAnalysis().isOuterJoined(column);
    }
}
```

The design separates concerns by providing `MaterializedViewScan` as a read-only interface in the SPI for connectors to inspect, while `MaterializedViewScanNode` in the runtime contains all the actual planning machinery. The implementation uses concrete planner components rather than abstract contexts, ensuring type safety and clear dependencies. Both the query planning and join analysis are computed lazily on-demand.

#### Design Principle: Optimizer-Based Stitching

The SPI remains simple with only basic freshness checking. Advanced connectors implement stitching by:
1. Providing a `ConnectorPlanOptimizer` that recognizes MV table scans
2. Analyzing the MV's embedded query plan to understand its structure
3. Rewriting the plan to use standard operators (UnionNode, FilterNode, TableScanNode)

#### Simple Freshness Model (SPI)

```java
public class MaterializedViewFreshness {
    public enum Freshness {
        FRESH,      // Completely up-to-date
        STALE,      // Has stale data
        UNKNOWN     // Cannot determine
    }
    
    private final Freshness freshness;
    private final Optional<Instant> lastRefreshTime;
    private final Optional<String> hint;  // Human-readable hint
}
```

#### How It's Created During Planning

```java
// In RelationPlanner.visitTable() - when planning a table reference
@Override 
protected RelationPlan visitTable(Table node, Void context) {
    Query namedQuery = analysis.getNamedQuery(node);
    Scope scope = analysis.getScope(node);
    
    if (namedQuery != null) {
        // It's a view or CTE
        return process(namedQuery, context);
    }
    
    TableHandle tableHandle = analysis.getTableHandle(node);
    
    // ... build outputSymbols, assignments, etc. ...
    
    // Check if this table is actually a materialized view
    Optional<MaterializedViewDefinition> mvDef = 
        metadata.getMaterializedView(session, tableHandle);
    
    if (mvDef.isPresent()) {
        // Create MaterializedViewScanNode for connector optimization
        return new RelationPlan(
            new MaterializedViewScanNode(
                idAllocator.getNextId(),
                tableHandle,
                outputSymbols,
                assignments,
                mvDef.get(),
                sqlParser,
                analysis,
                session,
                metadata,
                idAllocator,
                symbolAllocator),
            scope,
            outputSymbols);
    }
    
    // Regular table - create TableScanNode
    return new RelationPlan(
        new TableScanNode(
            idAllocator.getNextId(),
            tableHandle,
            outputSymbols,
            assignments),
        scope,
        outputSymbols);
}
```

#### Connector Optimizer Implementation (Iceberg)

```java
public class IcebergMVOptimizer implements ConnectorPlanOptimizer {
    
    @Override
    public PlanNode optimize(PlanNode plan, 
                           ConnectorSession session,
                           VariableAllocator variableAllocator,
                           PlanNodeIdAllocator idAllocator) {
        
        return SimplePlanRewriter.rewriteWith(
            new MVStitcher(session, variableAllocator, idAllocator), 
            plan);
    }
    
    private class MVStitcher extends SimplePlanRewriter<Void> {
        
        @Override
        public PlanNode visitMaterializedViewScan(MaterializedViewScanNode node, RewriteContext<Void> context) {
            // Get detailed freshness from Iceberg (internal, not SPI)
            IcebergFreshnessDetails details = getIcebergFreshness(node.getTable());
            
            if (details.isCompletelyFresh()) {
                // Keep as simple table scan - no planning needed!
                return new TableScanNode(
                    idAllocator.getNextId(),
                    node.getTable(),
                    node.getOutputSymbols(),
                    node.getAssignments());
            }
            
            if (details.isTooStale() || node.hasOuterJoins()) {
                // Full recomputation needed
                return node.getDefinitionPlan();
            }
            
            // Can do incremental refresh - get the planned definition
            PlanNode definition = node.getDefinitionPlan();  // Triggers planning
            boolean hasAggregation = hasAggregationNodes(definition);
            
            // Build sophisticated stitching plan with standard nodes
            List<PlanNode> unionParts = new ArrayList<>();
            
            // Add fresh data from MV
            if (!details.getFreshPartitions().isEmpty()) {
                unionParts.add(new TableScanNode(
                    node.getTable(),
                    createPartitionFilter(details.getFreshPartitions())));
            }
            
            // Add stale data computation
            for (PartitionInfo partition : details.getStalePartitions()) {
                if (hasAggregation || partition.hasDeletes()) {
                    // Must recompute - filter definition to this partition
                    unionParts.add(new FilterNode(
                        definition,
                        createPartitionPredicate(partition)));
                } else {
                    // Can safely append new data only
                    unionParts.add(createIncrementalScan(partition));
                }
            }
            
            // Return standard UNION node - runtime knows how to execute
            return new UnionNode(
                idAllocator.getNextId(),
                unionParts,
                outputMappings);
        }
    }
}
```

#### Rationale for This Design

This approach keeps the core SPI simple by avoiding any stitching concepts, allowing each connector to fully control its optimization strategy through the existing `ConnectorPlanOptimizerProvider` infrastructure. Basic connectors can work without implementing an optimizer at all, while advanced connectors can provide sophisticated optimizations without the runtime needing to understand connector-specific concepts

#### What Each Component Handles

Simple connectors like PostgreSQL and MySQL implement only basic freshness checking, leaving the runtime to handle the fresh/stale decision logic. These connectors don't need to understand complex stitching strategies or provide optimizers, making their implementation straightforward and minimal.

Advanced connectors like Iceberg and Hive implement comprehensive freshness checking in their metadata layer and provide sophisticated optimizers for stitching fresh and stale data. They leverage their internal knowledge of snapshots, partitions, and other format-specific features to optimize materialized view operations beyond what the generic runtime could achieve.

The runtime creates MaterializedViewScanNode instances with embedded view definitions and runs any connector-provided optimizers if available. When no optimizer is provided, the runtime falls back to simple fresh/stale logic, ensuring that all connectors get basic materialized view functionality even without custom optimization.

### 6. Storage Table Location Strategy

Iceberg will provide flexible configuration for where MV storage tables are created:

```sql
-- Default: Same schema as MV
CREATE MATERIALIZED VIEW catalog.sales.monthly_summary AS ...
-- Creates storage table: catalog.sales.$materialized_view_storage$monthly_summary

-- Option: Different schema for storage
CREATE MATERIALIZED VIEW catalog.sales.monthly_summary
WITH (
    storage_schema = 'sales_mv_storage',
    partitioned_by = ARRAY['year(sale_date)', 'month(sale_date)']
)
AS SELECT ...
-- Creates storage table: catalog.sales_mv_storage.monthly_summary
```

**Storage Configuration Options**:
- `storage_schema`: Schema for storage table (default: same as MV)
- `storage_table_prefix`: Prefix for storage table names (default: `$materialized_view_storage$`)

**Session-level Defaults**:
```sql
SET SESSION iceberg.default_mv_storage_schema = 'mv_storage';
SET SESSION iceberg.mv_storage_table_prefix = 'mv_';
```

### 6. Atomic Operations and Catalog Support

Materialized view refresh requires atomically updating both the storage table (with new data) and the view metadata (with new snapshot references and timestamps), ensuring that queries never see an inconsistent state where data and metadata are out of sync. Both REST catalogs and Hive metastore catalogs can provide this atomicity through different mechanisms:

#### REST Catalog (Multi-Table Transactions)
```java
private void atomicRefreshWithRestCatalog(
    IcebergMaterializedView mv,
    Table newData,
    long newBaseSnapshot) {
    
    // REST catalog supports multi-table transactions
    Transaction transaction = catalog.newTransaction();
    
    // Update storage table
    transaction.updateTable(mv.getStorageTable())
        .appendFiles(newData.newScan().planFiles())
        .commit();
    
    // Update view metadata
    transaction.updateView(mv.getViewName())
        .setProperty("base_snapshot_id", String.valueOf(newBaseSnapshot))
        .setProperty("last_refresh_time", String.valueOf(System.currentTimeMillis()))
        .commit();
    
    // Atomic commit across both tables
    transaction.commitTransaction();
}
```

#### Hive Metastore Catalog (Lock-Based Atomicity)
```java
private void atomicRefreshWithHiveCatalog(
    ExtendedHiveMetastore metastore,
    MetastoreContext context,
    IcebergMaterializedView mv,
    Table newData,
    long newBaseSnapshot) {
    
    // Acquire exclusive locks on both view and storage table
    long viewLockId = metastore.lock(context, mv.getSchema(), mv.getViewName())
        .orElseThrow(() -> new PrestoException(ICEBERG_COMMIT_ERROR, "Failed to lock view"));
    long storageLockId = metastore.lock(context, mv.getSchema(), mv.getStorageTableName())
        .orElseThrow(() -> new PrestoException(ICEBERG_COMMIT_ERROR, "Failed to lock storage table"));
    
    try {
        // With locks held, perform atomic updates
        
        // Step 1: Update storage table through Iceberg transaction
        Transaction tableTransaction = mv.getStorageTable().newTransaction();
        tableTransaction.newAppend()
            .appendFiles(newData.newScan().planFiles())
            .commit();
        tableTransaction.commitTransaction();
        
        // Step 2: Update view metadata
        updateViewMetadata(mv.getViewName(), ImmutableMap.of(
            "storage_snapshot_id", String.valueOf(mv.getStorageTable().currentSnapshot().snapshotId()),
            "base_snapshot_id", String.valueOf(newBaseSnapshot),
            "last_refresh_time", String.valueOf(System.currentTimeMillis())
        ));
        
        // Both operations completed successfully with locks held
    } finally {
        // Release locks in reverse order
        metastore.unlock(context, storageLockId);
        metastore.unlock(context, viewLockId);
    }
}
```

## Adoption Plan

### Impact on Existing Users

- **Breaking Change**: The existing Hive MV implementation will be removed (this feature is currently unused and undocumented)
- **New Session Properties**:
    - `materialized_view_grace_period`
    - `materialized_view_refresh_strategy`
    - `materialized_view_incremental_enabled`


### Documentation Requirements

- SPI implementation guide explaining the connector interfaces required for materialized view support
- Users guide covering system configurations and session properties for materialized views
- Iceberg-specific documentation for features like snapshot-based freshness and partition transforms

### Out of Scope

- Cross-connector materialized views
- Automatic MV selection for arbitrary queries
- View maintenance policies (TTL, auto-refresh)
- Cost-based refresh scheduling

## Test Plan

### Unit Tests

- Basic tests for SPI implementations
- Planner rule tests for refresh logic
- Freshness calculation tests
- Incremental filter generation tests
- Partition alignment detection tests

### Integration Tests

- End-to-end MV creation and refresh
- Concurrent refresh and query execution
- Transaction rollback scenarios
- Large-scale incremental refresh