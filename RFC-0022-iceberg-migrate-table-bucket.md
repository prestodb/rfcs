# RFC0022 for Presto iceberg-migrate-table-bucket

## Iceberg Migrate Table Bucket Procedure

**Proposers:** 
  Prabhu Shankar

## Related Issues


- prestodb/presto#26779

## Summary
Adding a new distributed system procedure `system.migrate_table_bucket` for the iceberg connector that copies all data files of an iceberg table 
from their current location to a new base path and atomically writes the table's meta data to point to the new file locations.

## Background
Users who are trying to migrate iceberg tables to a target location, standard data transfer utlities does not work. As Ice berg's meta data relies on absolute path references. 
There is no in built presto mechanism to do this atomically. 
The current steps for users are
1. Forced to manaully copy files and edit meta data (error prone)
2. Re-ingest all data (expensive)
   

This procedure solves this by leveraging the existing distributed procedure framework 
(RFC-0021) to parallelize the file copy across workers and then commit the path 
remapping atomically via an Iceberg RewriteFiles transaction.

### Goals
- Allow users to relocate Iceberg table data files to a new base path via a single SQL call.
- Perform the copy in a distributed, parallelized manner.
- Atomically update Iceberg metadata so the table is consistent throughout.

### Non-goals
- Tables with delete files are not supported in this version (must compact first).
- Does not handle incremental/partial migrations.
- Does not delete files from the old location (that is left to the user).

## Proposed Implementation

### Modules Involved
- presto-iceberg: New class `MigrateTableBucketProcedure` implementing Provider<DistributedProcedure>.
- Uses existing `TableDataRewriteDistributedProcedure` SPI.
- Uses `IcebergDistributedProcedureHandle` to pass new_base_path and table_location to workers.

### SQL Interface
```sql
CALL iceberg.system.migrate_table_bucket(
    schema => 'my_schema',
    table_name => 'my_table',
    new_base_path => 's3://new-bucket/path/to/table'
)
```

### Code Flow

**begin phase (coordinator):**
1. Validate the table has no delete files (throws NOT_SUPPORTED if any exist).
2. Normalize new_base_path (strip trailing slash).
3. Return an IcebergDistributedProcedureHandle carrying new_base_path and table_location in relevantData.

**distributed phase (workers):**
1. For each DataFile, compute the relative path by stripping the table location prefix.
2. Construct newPath = new_base_path + "/" + relativePath.
3. If newPath == oldPath, skip the copy (no-op, encode old -> old in fragment).
4. Otherwise, copy bytes using FileIO (64KB buffer), then encode 
   oldPath\0newPath as a fragment Slice.

**finish phase (coordinator):**
1. Decode all fragments into an oldPath -> newPath map.
2. Scan the current snapshot; for each DataFile, look up the remapped path.
3. Build new DataFile objects with updated paths.
4. Commit an Iceberg RewriteFiles transaction atomically.

### Key Classes / Methods
- `MigrateTableBucketProcedure#beginCallDistributedProcedure`
- `MigrateTableBucketProcedure#copyFileAndBuildFragment` (static, callable from workers)
- `MigrateTableBucketProcedure#finishCallDistributedProcedure`
- `MigrateTableBucketProcedure#decodeFragments`

### Error Handling
- Rejects tables with delete files upfront.
- Throws IllegalStateException if a worker fragment is missing for any data file (prevents partial commits).
- IO failures during copy are wrapped as UncheckedIOException with source/dest context.

## Metrics

- Number of files copied vs skipped (same-path no-ops).
- Bytes transferred during migration.
- Procedure duration.

## Other Approaches Considered

- Based on the discussion, we can update this part. 

## Adoption Plan

- **Impact on existing users None.** This is a new opt-in procedure.
- **New SQL:** `CALL iceberg.system.migrate_table_bucket(...)` - new procedure, no changes to existing SQL grammar.
- **New session parameters/configs:** None.
- **Behaviour changes:** None to existing functionality.
- **Migration tools:** This procedure itself is the migration tool.
- **Documentation:** A new section in the Iceberg connector docs describing the 
  procedure, its parameters, limitations (no delete files), and example usage.
- **Out of scope:** Automatic cleanup of old files after migration; support for 
  tables with delete files.

## Test Plan

- Unit tests for `copyFileAndBuildFragment`: same-path no-op, cross-bucket copy, relative path computation for files inside vs outside the table location prefix.
- Unit tests for `decodeFragments`: valid payloads, missing delimiter (malformed).
- Integration test: call `migrate_table_bucket` on a real Iceberg table, verify all data files are readable at the new location and the old paths are gone from metadata.
- Negative test: call on a table with delete files - should throw `NOT_SUPPORTED`.
- Negative test: simulate a missing fragment for one file - should throw 
  `IllegalStateException` and not commit.
