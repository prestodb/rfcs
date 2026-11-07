# **RFC 0022: Query Results Cache**

Proposers

* Tim Meehan (@tdcmeehan)
* Jalpreet Singh Nanda (@imjalpreet)

## Abstract

This RFC proposes a Query Results Cache for Presto that caches completed `SELECT` query results so semantically equivalent future queries skip execution entirely. The cache intercepts at `SqlQueryExecution.start()` after optimization, before scheduling. On a cache hit, pages are read from `TempStorage` and fed directly to the client via the existing `OutputBuffer → ExchangeClient → HTTP` response pipeline, bypassing scheduling and execution entirely.

---

## Motivation

Presto users frequently re-execute identical or semantically equivalent analytical queries, dashboard refreshes, repeated report runs, BI tool polling. Each re-execution incurs the full cost of scheduling, split enumeration, and execution despite producing an identical result. A query results cache eliminates this redundant work, reducing latency for repeated queries and load on underlying data sources.

The design builds on two existing Presto subsystems:

- **HBO canonicalization**: provides a deterministic plan hash as cache key, including connector version metadata (e.g., Iceberg snapshot IDs) so data changes automatically produce different hashes.
- **TempStorage**: pluggable storage SPI (local disk, S3) for serialized result pages.

This follows the same pattern as the Fragment Result Cache, intercept at the execution boundary, applied at whole-query level on the coordinator instead of per-split on workers.

```
SQL → Parse → Analyze → Optimize → Compute plan hash (includes table version)
   ├── HIT:  Read pages from TempStorage → OutputBuffer → Client
   └── MISS: Fragment → Schedule → Execute → Capture pages → Store
```

---

## Goals

- Eliminate redundant execution of identical `SELECT`s.
- Single storage layer via `TempStorage` SPI for both metadata and result pages.
- Version-aware invalidation: cache keys include connector-specific version metadata (e.g., Iceberg snapshot IDs), so any data mutation produces a different hash and is an automatic cache miss. Only connectors that declare `SUPPORTS_RESULT_CACHE` (guaranteeing ACID-consistent handle identity) are eligible.

## Non-Goals (v1)

- Partial reuse / predicate stitching
- DML caching
- Non-deterministic function caching
- KMS envelope encryption (deferred to v2)

---

## Proposal

### Cache Key

SHA-256 of the canonicalized query plan combined with a session properties fingerprint, computed via HBO's `CanonicalPlanGenerator`.

Uses a new `RESULT_CACHE` canonicalization strategy. Like `CONNECTOR`, it supports all plan node types and delegates to `ConnectorTableLayoutHandle.getIdentifier()` for connector-specific normalization. Unlike `CONNECTOR` and the other HBO strategies, it preserves all constants, filter predicates, projection constants, and scan predicates are never stripped. Two queries with different predicate values produce different cache keys.

Canonicalization normalizes variable names to positional (`_col_0`, ...), sorts predicates/join criteria, and strips source locations. Serialized to JSON with deterministic key ordering, then hashed.

**Session properties fingerprint**: a deterministic hash of the subset of session properties that can affect query results. This includes any connector specific session properties(e.g., `legacy_timestamp`, etc.) that influence scan or execution behavior. Properties that are purely operational (e.g., `query_max_execution_time`, `query_priority`) are excluded from the fingerprint as they do not affect result content. The set of result-affecting properties is declared explicitly via a new `SessionPropertyManager.getResultAffectingProperties()` method, so additions to Presto's session property surface are opt-in to the fingerprint rather than silently excluded by default.

**Full key:** `SHA-256(canonical_plan_hash + session_properties_fingerprint)`: the plan hash includes connector version metadata (e.g., snapshot IDs), so a data change produces a different key. The session fingerprint ensures two users or sessions with different result-affecting configurations never share a cache entry. No separate invalidation mechanism is needed for data changes.

---

### Connector Requirements

#### `SUPPORTS_RESULT_CACHE` Capability

Only connectors that declare `SUPPORTS_RESULT_CACHE` in `ConnectorCapabilities` are eligible for query result caching. This capability is a contract: **the connector guarantees that its `ConnectorTableLayoutHandle` identity (as returned by `getIdentifier()` with the `RESULT_CACHE` strategy) changes whenever the underlying data changes.** This requires ACID transaction support, without it, there is no reliable way to detect data mutations via the handle alone.

If any input table in a query belongs to a connector that does not declare `SUPPORTS_RESULT_CACHE`, the query is ineligible for caching.

```java
// ConnectorCapabilities.java
public enum ConnectorCapabilities {
    // ... existing entries ...
    SUPPORTS_RESULT_CACHE   // Handle identity is ACID-consistent with table data
}
```

#### Version Metadata in `getIdentifier()`

The `RESULT_CACHE` canonicalization strategy **preserves version metadata** in `getIdentifier()`, the opposite of HBO strategies, which strip it for better statistics matching. The `PlanCanonicalizationStrategy` is already passed to `getIdentifier()`, so connectors can branch on it.

For connectors that use the default `getIdentifier()` implementation (`return this`), the full handle identity, including version metadata, is included in the hash automatically. This is the desired behavior for `RESULT_CACHE`.

**Iceberg:** The default `getIdentifier()` (`return this`) already includes `snapshotId` via `IcebergTableLayoutHandle`. No connector code change needed, the `RESULT_CACHE` strategy naturally picks up the snapshot ID. A new Iceberg snapshot (from any write) produces a different plan hash and an automatic cache miss. This mirrors how `IcebergAbstractMetadata.getMaterializedViewStatus()` uses snapshot ID comparison to detect MV staleness.

---

### SPI Changes

#### `TempStorage` Additions

```java
public interface TempStorage {
    // ... existing methods ...
    boolean exists(TempDataOperationContext context, TempStorageHandle handle) throws IOException;
    boolean createIfNotExists(TempDataOperationContext context, TempStorageHandle handle, byte[] data) throws IOException;
}
```

- `exists()`: checks handle liveness without opening the full stream (`Files.exists()` locally, `HeadObject` on S3).
- `createIfNotExists()`: atomically creates an object only if it does not already exist. Returns `true` if the object was created, `false` if it already existed. S3 implementation uses `PutObject` with `If-None-Match: *`, S3 returns `412 Precondition Failed` if the key exists, which the implementation catches and returns `false`. Used for cache write deduplication.

#### `StorageCapabilities` Addition

```java
public enum StorageCapabilities {
    REMOTELY_ACCESSIBLE,
    AUTO_EXPIRATION,  // storage handles TTL-based expiration natively
}
```

#### `TempDataOperationContext` Addition

Add `Optional<Duration> expireAfter` to the existing context object passed to `create()`. Backends that support `AUTO_EXPIRATION` use this to set native expiration (e.g., S3 object expiration header). Backends that do not support it ignore the field.

---

### Storage Layout

Metadata and pages are stored together in `TempStorage` under a key-derived path:

```
cache/<plan_hash>/metadata.json
cache/<plan_hash>/page_0
cache/<plan_hash>/page_1
...
```

#### `QueryResultsCacheEntry`

Data class for the metadata file (`presto-spi`):

```java
public class QueryResultsCacheEntry {
    List<String> columnNames;
    List<String> columnTypeSignatures;
    long creationTimeMillis;
    long expirationTimeMillis;
    long totalRows;
    long totalBytes;
    int pageCount;
    List<Long> pageCrcs;            // CRC-32C of each encrypted page, indexed by page number
    long resultHash;                // CRC-32C of all pageCrcs concatenated — whole-result integrity check
    Optional<byte[]> encryptionKey; // DEK
}
```

#### Integrity Verification

Integrity verification operates at two levels:

**Per-page CRC (`pageCrcs`)**: a CRC-32C is computed over each page's **encrypted** bytes (after `AesSpillCipher` has run, before handoff to `TempStorage`) at write time, stored as a `List<Long>` indexed by page number. On read, the reader recomputes the CRC of each page as it is deserialized and compares against the stored value. A mismatch indicates a corrupt or partially uploaded page and is treated as a cache miss, the query falls through to normal execution.

**Whole-result hash (`resultHash`)**: a CRC-32C computed over all `pageCrcs` concatenated in page order, stored in the metadata. This is checked first on read, before individual pages are fetched, as a cheap early signal that the entry is intact. A mismatch skips fetching pages entirely and falls through immediately to normal execution.

CRCs are computed over the **encrypted** page bytes, not the plaintext. Integrity and confidentiality are therefore independent: corruption is detectable without decrypting the page first, and the CRC does not leak plaintext structure.

Any integrity check failure (per-page or whole-result) is treated identically to a cache miss. The query proceeds to normal execution and the corrupted entry is overwritten on completion via the normal write path. Failures are emitted as a metric (`cache.integrity_failure_count`) for visibility.

`QueryResultsCacheManager` reads/writes directly via `TempStorage`. Cache existence is checked via `TempStorage.exists()` on the metadata handle. Cross-coordinator sharing is supported when using shared `TempStorage` (e.g., S3).

---

### Read Path

In `SqlQueryExecution.start()`, after `createLogicalPlanAndOptimize()`:

1. Compute canonical plan hash.
2. Check `TempStorage.exists()` on the metadata handle for the key.
3. On exists: read and deserialize `QueryResultsCacheEntry` from metadata file. Validate: schema match (column names + types vs current `OutputNode`), expiration check, whole-result `resultHash` check.
4. On hit: `serveCachedResult()` — read `SerializedPage` objects from `TempStorage`, verify per-page CRC, enqueue into `OutputBuffer`, transition to finishing. Client sees no difference from normal execution.
5. On miss: register query ID + cache key for population on completion, proceed with normal scheduling.

Non-deterministic functions (`rand()`, `now()`, `uuid()`) are detected during hash computation; cache is not consulted. Any storage or integrity failure on the read path is treated as a cache miss, cache infrastructure is never on the critical path to query success.

---

### Write Path

`QueryResultsCacheWriter` hooks into the query lifecycle via `addFinalQueryInfoListener` (same pattern as `HistoryBasedPlanStatisticsTracker`).

**Page capture**: tee-write at the `ExchangeClient`:

As pages arrive at the coordinator's `ExchangeClient` during normal execution, a cache-aware listener asynchronously writes each batch to `TempStorage` in the background. The client receives pages with normal latency, cache writes are off the critical path.

- A running byte counter tracks total result size. If it exceeds `max-result-size`, the tee-write is abandoned and partial files are cleaned up.
- On successful `SELECT` completion, the `QueryResultsCacheWriter` collects the accumulated `TempStorageHandle` references, builds a `QueryResultsCacheEntry`, and stores metadata in the cache provider.
- If the query fails or is cancelled, partial cache writes are discarded.

This uses the same pattern as `FileFragmentResultCacheManager.cachePages()`, but incremental rather than post-completion.

Integration point: `SqlQueryManager.createQuery()`, alongside existing HBO tracking.

---

### Write Deduplication (S3)

When multiple coordinators experience a cache miss for the same plan hash simultaneously, only one should execute and populate the cache, others should wait for the result. This avoids redundant execution across coordinators.

Uses [S3 conditional writes](https://docs.aws.amazon.com/AmazonS3/latest/userguide/conditional-writes.html) (`If-None-Match: *` on `PutObject`) to implement lock-free writer election. No external coordination service required.

#### Protocol

1. On cache miss, before scheduling, the coordinator calls `TempStorage.createIfNotExists()` on a lock object at `cache/<plan_hash>/lock`. The lock body contains `{"coordinatorId": "...", "creationTimeMillis": ...}`.

2. **Wins** (`createIfNotExists` returns `true`): This coordinator is the writer. Proceeds with normal scheduling, execution, and cache population. On completion **or failure**, deletes the lock object. Deleting the lock on failure is essential, it is the signal losing coordinators use to detect that the writer is no longer making progress.

3. **Loses** (`createIfNotExists` returns `false`): Another coordinator is already executing this query. The coordinator polls for **both** `cache/<plan_hash>/lock` and `cache/<plan_hash>/metadata.json` using exponential backoff (starting at 1 second, doubling each interval, capped at 30s), to reduce unnecessary S3 API calls during long-running query execution. Two outcomes are possible on each poll:
    - **`metadata.json` appears**: the writer completed successfully. Serve from cache.
    - **`lock` disappears without `metadata.json` appearing**: the writer crashed or the query failed. Fall through immediately to normal execution and compete for a new lock via `createIfNotExists`. This avoids waiting for the full polling timeout in failure scenarios.

   If polling exceeds a configurable timeout (default: equal to the query's execution time limit) without either condition being met, fall through to normal execution.

4. **Crashed writer**: The lock object includes `creationTimeMillis`. A coordinator that loses the election checks the lock's age on each poll. If the lock is older than a configurable staleness threshold (default: **10 minutes**), it treats the writer as crashed, deletes the stale lock, and competes for a new lock via `createIfNotExists`. The `S3 Expires` header is also set on lock objects as a backstop for garbage collection.

#### Fallback

If `createIfNotExists` throws (e.g., non-S3 backend that does not support conditional writes), the coordinator falls through to normal execution and cache population. Duplicate writes to the same cache key are idempotent, the last writer wins, which is safe since results for the same plan hash are identical.

```
Coordinator A (cache miss)          Coordinator B (cache miss)
  │                                   │
  ├─ createIfNotExists(lock) → true   ├─ createIfNotExists(lock) → false
  │  (I'm the writer)                 │  (someone else is writing)
  │                                   │
  ├─ Schedule + Execute               ├─ Poll for lock + metadata.json
  │                                   │     ↓ (exponential backoff)
  ├─ Write pages + metadata           ├─ metadata appears → serve from cache
  │                                   │     OR
  ├─ Delete lock                      ├─ lock gone, no metadata → fall through
  ▼                                   ▼
```

---

### Invalidation

- **Data-change detection via cache key**: Version metadata (e.g., Iceberg snapshot IDs) is included in the plan hash via the `RESULT_CACHE` canonicalization strategy. Any data mutation produces a new version, a different hash, and an automatic cache miss. No separate stats comparison is needed, invalidation is a natural consequence of the cache key design.

- **TTL with access-based extension**: Default 1 hour, configurable per-session. Each `QueryResultsCacheEntry` stores `expirationTimeMillis`, checked on read before serving. On a cache hit, if remaining TTL is below 50%, the metadata file is overwritten asynchronously with a new `expirationTimeMillis` (fire-and-forget S3 `PUT`, not on the critical read path). Total lifetime is capped at a configurable maximum (default 24h). S3 object expiration on page data files is set to `max_lifetime + buffer` to ensure they outlive any metadata extension.

- **Manual bypass**: `query_results_cache_bypass = true`: skip read, still populate. `query_results_cache_invalidate = true`: skip read, overwrite entry (guarded by a config flag; intended for testing only).

- **Storage cleanup**: Cache writes pass `expireAfter` via `TempDataOperationContext`. S3 sets native object expiration (with a configurable buffer to prevent deletion during in-flight reads). The read-path `expirationTimeMillis` check in the metadata is the source of truth for staleness; backend expiration is garbage collection only. Stale entries (unreachable due to changed plan hash) are cleaned up by TTL expiration.

Read-path correctness does not depend on the cleanup strategy. The `pageCount` field in `QueryResultsCacheEntry` defines the expected number of pages. If any page is missing or fails its CRC check, the query falls through to normal execution. Results are never silently truncated.

---

### Encryption

Pages are encrypted via the existing `AesSpillCipher` / `PagesSerde` pipeline (AES-256-CTR, fresh 256-bit key per entry, new IV per operation).

- **Write**: Create `AesSpillCipher`, serialize pages with cipher-aware `PagesSerde`, store DEK in `QueryResultsCacheEntry.encryptionKey`.
- **Read**: Reconstruct cipher from stored DEK, decrypt during deserialization, `cipher.destroy()` after.

**v1 key management**: DEK stored in the metadata file alongside page references. Envelope encryption with KMS deferred to v2. Storage-layer encryption (SSE-S3/SSE-KMS) can be used as a complementary layer.

> ⚠️ **v1 encryption limitation: no key isolation**: In v1, the DEK is co-located with the encrypted pages under the same S3 path. Any principal with `s3:GetObject` access to the bucket can retrieve both `metadata.json` (containing the DEK) and the encrypted pages in a single API call, and therefore decrypt the cached results.

**v2 key management (planned)**: Full key isolation via envelope encryption with KMS, the DEK will be encrypted by a KMS master key that never leaves KMS and stored as a wrapped key in metadata. Decrypting cached results will require both `s3:GetObject` and `kms:Decrypt` permissions, which are independently auditable and separately grantable.

---

### Security

**Shared cache entries**: Cache entries are shared across users. Access control is enforced during the analysis phase (before the cache is consulted), so unauthorized users cannot reach the cache lookup.

**Row filters and column masks**: Row filters and column masks are injected into the plan during analysis/planning as `FilterNode` and `ProjectNode` respectively. Because `AccessControl.getRowFilters()` dispatches per-user, returning different `ViewExpression` expressions for different identities, different users produce different plan trees and therefore different cache keys. No special handling is needed; per-user cache isolation is a natural consequence of the plan hash. If a filter expression references `current_user`, the `$current_user` function is deterministic and constant-folded to a string literal during optimization, so it does not trigger a non-deterministic bypass, it simply produces a different literal per user and a different cache key.

**Non-deterministic functions**: Detected during cache key computation; cache is not consulted.

---

### Configuration

#### Server Properties

| Property | Default | Description |
|---|---|---|
| `query-results-cache.enabled` | `false` | Master switch |
| `query-results-cache.ttl` | `1h` | Default TTL |
| `query-results-cache.max-result-size` | `100MB` | Max cacheable result size |
| `query-results-cache.temp-storage` | `local` | TempStorage backend name |
| `query-results-cache.encryption-enabled` | `true` | Encrypt cached pages (see v1 limitation above) |
| `query-results-cache.write-dedup-enabled` | `true` | Enable write deduplication via conditional writes (S3 only) |
| `query-results-cache.write-dedup-poll-interval` | `1s` | Base polling interval for waiting coordinators (exponential backoff) |
| `query-results-cache.write-dedup-lock-staleness-threshold` | `10m` | Age after which a lock is considered stale |

#### Session Properties

`query_results_cache_enabled`, `query_results_cache_ttl`, `query_results_cache_bypass`, `query_results_cache_invalidate`.

---

### S3 TempStorage

A new `TempStorage` implementation backed by S3, enabling cross-coordinator cache sharing and native object expiration.

**Module**: New Maven module `presto-temp-storage-s3`, following the plugin pattern. Ships as a plugin; loaded via `etc/temp-storage/s3.properties`.

#### Implementation

`S3TempStorage` implements `TempStorage`. `S3TempStorageFactory` implements `TempStorageFactory` with `getName()` returning `"s3"`.

**Capabilities**: `REMOTELY_ACCESSIBLE`, `AUTO_EXPIRATION`.

**Handle**: `S3TempStorageHandle` wraps an S3 key (string). `serializeHandle()` / `deserialize()` encode/decode the key as UTF-8 bytes. `getPathAsString()` returns the full `s3://bucket/key` URI.

**Operations**:

| Method | S3 API | Notes |
|---|---|---|
| `create()` | — | Returns an `S3TempDataSink` that buffers writes and uploads on `commit()` via `PutObject`. If `expireAfter` is set in the context, sets the `Expires` header on the object. |
| `open()` | `GetObject` | Returns the response `InputStream`. |
| `remove()` | `DeleteObject` | Best-effort; S3 deletes are eventually consistent but idempotent. |
| `exists()` | `HeadObject` | Returns `true` on 200, `false` on 404. |
| `createIfNotExists()` | `PutObject` with `If-None-Match: *` | Returns `true` on success, `false` on `412 Precondition Failed`. Used for cache write deduplication. |

**Key layout**: Objects are stored under a configurable prefix:

```
<key-prefix>/cache/<plan_hash>/metadata.json
<key-prefix>/cache/<plan_hash>/page_0
...
```

**Expiration buffer**: When `expireAfter` is provided via `TempDataOperationContext`, the factory adds a configurable buffer (default 1 hour, via `s3.expiration-buffer`) before setting the S3 `Expires` header. This prevents S3 lifecycle rules from deleting objects while a cache-hit read is in flight.

#### S3 Configuration

Properties file: `etc/temp-storage/s3.properties`

```properties
temp-storage-factory.name=s3
s3.bucket=my-presto-temp-storage
s3.key-prefix=presto/temp
s3.region=us-east-1
s3.endpoint=              # optional, for S3-compatible stores
s3.expiration-buffer=1h   # buffer added to expireAfter before setting Expires header
s3.iam-role=              # optional
s3.aws-access-key=        # optional
s3.aws-secret-key=        # optional
```

If neither IAM role nor access key/secret key pair is configured, authentication uses the default AWS credential chain (environment variables, instance profile, etc.), consistent with existing Presto S3 integrations (e.g., `PrestoS3FileSystem`).

---

## Implementation Plan

### New Files

- `presto-spi`: `QueryResultsCacheEntry.java`
- `presto-main-base`: `QueryResultsCacheManager.java`, `QueryResultsCacheWriter.java`, `QueryResultsCacheConfig.java`
- `presto-temp-storage-s3`: `S3TempStorage.java`, `S3TempStorageFactory.java`, `S3TempStorageHandle.java`, `S3TempDataSink.java`, `S3TempStorageConfig.java`

### Modified Files

`PlanCanonicalizationStrategy.java` (add `RESULT_CACHE`), `IcebergTableLayoutHandle.java` (implement `getIdentifier()`), `StorageCapabilities.java` (add `AUTO_EXPIRATION`), `TempDataOperationContext.java` (add `expireAfter`), `TempStorage.java` (add `exists()`, `createIfNotExists()`), `LocalTempStorage.java` (implement `exists()`, stub `createIfNotExists()`), `SqlQueryExecution.java` (read path + write dedup), `ExchangeClient.java` (tee-write listener for cache page capture), `SystemSessionProperties.java`, `ServerMainModule.java`, `SqlQueryManager.java`.

### Phases

| Phase | Scope |
|---|---|
| 1 | `TempStorage` SPI additions (`exists()`, `createIfNotExists()`) + `QueryResultsCacheEntry` data class |
| 2 | Write path: `QueryResultsCacheWriter`, `ExchangeClient` tee-write, size limit, write dedup lock protocol |
| 3 | Read path: `SqlQueryExecution` interception, plan hash + session fingerprint, `serveCachedResult()`, integrity checks |
| 4 | Encryption: `AesSpillCipher` integration, DEK in metadata, CRC over encrypted bytes |
| 5 | Invalidation + cleanup: TTL, access-based extension, bypass/invalidate session properties, stale lock handling |
| 6 | S3 `TempStorage` plugin: new Maven module, conditional writes, expiration buffer |
| 7 | Testing: unit, integration, performance, security (RLS isolation) |

---

## Future Work

- **Predicate stitching**: Partial cache reuse for partition-decomposable queries when only some partitions change. Requires per-partition stats.
- **Cross-coordinator sharing**: Already supported when using shared `TempStorage` (e.g., S3). Write deduplication via S3 conditional writes prevents redundant execution across coordinators.
- **Adaptive caching**: Track hit rates per query pattern; skip caching for one-off queries.
- **KMS envelope encryption (v2)**: Full key isolation, DEK encrypted by a KMS master key, stored as a wrapped key in metadata. Requires both `s3:GetObject` and `kms:Decrypt` to decrypt results.
- **Non-ACID connector support**: Define a path for connectors that cannot guarantee ACID-consistent handle identity (e.g., Hive non-transactional tables) to participate via an alternative invalidation mechanism.
