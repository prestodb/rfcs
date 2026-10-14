# RFC: Arrow IPC query results over HTTP

Proposers

* Jianjian Xie @jja725

## Related Issues

* [Arrow Flight SQL endpoint proposal, prestodb/rfcs#63](https://github.com/prestodb/rfcs/pull/63) — the closest related design. Both proposals convert query output to Arrow on the coordinator first; this proposal uses the existing HTTP statement protocol instead of adding a Flight SQL service.
* [RFC-0065: ADBC Driver for Presto](RFC-0065-adbc-driver-for-presto.md) and the [Presto ADBC driver](https://github.com/adbc-drivers/presto) — an initial consumer of the proposed transport.
* [RFC-0022: Go Client Library v2](RFC-0022-go-client-v2.md) — the existing HTTP client lifecycle.
* [RFC-0004: Arrow Flight connector](RFC-0004-arrow-flight-connector.md) — Presto reading Arrow data, rather than serving query results.
* [Native ADBC connector proposal, prestodb/rfcs#64](https://github.com/prestodb/rfcs/pull/64) — also an input connector, not a client result endpoint.
* [Arrow IPC specification](https://arrow.apache.org/docs/format/Columnar.html#serialization-and-interprocess-communication-ipc) — the binary format used here.

An implementation issue will be created in `prestodb/presto` after RFC acceptance.
The filename uses the contribution guide's `00xx` placeholder pending numbering.

## Summary

Add an opt-in result format to Presto's existing HTTP statement protocol:

* Submit ordinary SQL to `/v1/statement`.
* Keep JSON for query status, errors, warnings, session updates, and `nextUri`.
* Return a `dataUri` descriptor instead of JSON rows when output is available.
* Fetch that URI over HTTP to receive a raw, optionally compressed **Arrow IPC
  stream**, with no JSON or base64 wrapping around the result bytes.

The initial implementation converts Presto output pages to Arrow on the Java
coordinator. It supports both Java workers and Prestissimo, without changing
worker exchange serialization or requiring a new listener, gRPC, or Flight SQL.
Existing JSON clients are unchanged, and the feature is disabled by default.

This is incremental result streaming through **bounded, independently decodable
segments**, not one HTTP response held open for the entire query. Clients present
the segments as one logical Arrow batch reader and must still read the final
control response to determine whether the query succeeded.

**Scope boundary:** add Arrow as a result representation within Presto's
existing HTTP query lifecycle. Do not introduce a separate query-long streaming
service or recreate Flight's result-discovery, progress, and cancellation APIs.

## Background

### Motivating workload

Training and analytical jobs often need a small projection of a large fact table
and a few small dimension tables. A common pipeline runs a separate distributed
job to project, filter, join, and materialize a training dataset before a trainer
can start consuming it.

For workloads whose dimension tables fit in trainer memory, another option is:

1. Load the small tables into in-memory lookup structures.
2. Submit a partition-filtered, column-projected SQL query for the fact table.
3. Consume the fact rows as Arrow batches and perform the small lookups while
   preparing training batches.

This can avoid an intermediate dataset and its preparation stage. It is not a
claim that local joins always beat Presto or Spark: large joins, global shuffles,
repeated epochs, durable snapshots, and reproducible feature materialization may
still need a different pipeline. The proposed interface is general SQL, not an
ML-specific or Hive-specific `table + partition + columns` service.

For example:

```sql
SELECT entity_id, feature_a, feature_b, label
FROM hive.training.fact_events
WHERE ds = '2026-10-01'
```

Applications may instead put their joins in this SQL and let Presto execute
them. Partition pruning and projection remain normal planner/connector
responsibilities; this RFC changes only result delivery. Reading side tables
and fact tables in separate queries does not create a common snapshot. Users
requiring that guarantee must select suitable snapshots or use one query.

### The missing transport

Presto's current [statement result implementation][query-source] supports JSON
rows and a `binaryResults=true` mode. The latter puts base64-encoded Presto
serialized pages in `QueryResults.binaryData`; it is not Arrow IPC. Supporting
it in an Arrow client would require knowledge of Presto block encodings and
conversion on the client.

The current [ADBC driver database factory][adbc-factory] uses the Presto Go
client through `database/sql`, and its [statement implementation][adbc-statement]
uses `sqlwrapper`. An Arrow API at the application boundary therefore does not
mean Arrow is the format on the network.

Arrow IPC moves result values over the wire as typed columnar buffers. Clients
still perform network reads, allocate memory, and decompress compressed buffers;
this is not an end-to-end zero-copy guarantee. Compressed Arrow is also not
guaranteed to be smaller than compressed Parquet or compressed JSON. Throughput,
CPU cost, and transferred bytes must be measured separately.

### Relationship to Flight SQL

Proposal #63 prioritizes a standard Flight SQL endpoint and its client
ecosystem. It explicitly considers an HTTP Arrow alternative. This RFC develops
that alternative for deployments that want to retain their HTTP authentication,
routing, and statement lifecycle.

The trade-off is deliberate: a standard Arrow reader can decode each data
response, but a Presto-aware client must manage the surrounding HTTP protocol.
An unmodified ADBC Flight SQL driver cannot connect to this endpoint.

Neither proposal needs to block the other. They should share result type
mapping, page-to-Arrow conversion, memory-accounting rules, and conformance
fixtures. This RFC does not supersede #63 or promise its future worker-parallel
transport.

The overlap from a more elaborate HTTP streaming design would primarily be with
[Arrow Flight's data-delivery layer](https://arrow.apache.org/docs/format/Flight.html):
schema/endpoint discovery, tickets, `DoGet`, progress polling, and cancellation.
[Flight SQL](https://arrow.apache.org/docs/format/FlightSql.html) adds standardized
SQL execution, prepared statements, and database metadata on top. Building
equivalent capabilities in a custom HTTP service would retain the need for a
Presto-specific client while duplicating much of that protocol work.

This is not a claim that Flight makes server integration or recovery free.
Either transport needs Presto query integration, authorization, resource
accounting, and Arrow conversion; Flight does not by itself provide durable
midstream resume or exactly-once application consumption. The boundary here is
to reuse the existing HTTP result lifecycle, rather than define a parallel one.
If a standardized query-long streaming service is needed, prefer collaborating
on #63 instead of extending this RFC into a second such service.

### Why bounded segments for the first implementation

Presto already has result tokens, `nextUri`, and a cached last response for
repeated token requests in [`Query`][query-source]. Attaching a bounded Arrow
payload to that result entry follows the existing model. Each segment is itself
a complete Arrow IPC stream; segmenting does not mean using a custom Arrow
encoding or converting values back to JSON.

For the motivating ML workload, the first objective is to deliver projected
results as compressed columnar batches without client-side JSON-to-Arrow
conversion. Bounded segments support incremental consumption before the query
finishes; a query-long HTTP response is not required for that objective.

Segments give payload ownership and replay an explicit boundary. A failed data
GET can replay the retained segment before any of its batches are exposed, and
a slow consumer can stop advancing `nextUri`. Correct retention, final status,
memory accounting, and cleanup still require implementation; this is not just
a serializer switch.

A query-long response could reduce per-segment request and buffering overhead.
Opening such a response is not inherently difficult, and a fail-on-disconnect
contract could remain simple. Adding recovery after partial delivery and a
separate status/progress lifecycle is where the design grows toward Flight.
There is no benchmark evidence yet that avoiding segment boundaries justifies
that additional scope for this workload.

Start with bounded, multi-batch segments and measure their request overhead
against conversion CPU, memory, and end-to-end consumption rate. Tune segment
size before considering a different transport. A transport change would need
separate justification and review alongside #63, not be an implicit next phase
of this proposal.

### Goals

* Eliminate row-oriented JSON and client-side row-to-Arrow rebuilding on the
  opted-in result path.
* Preserve query semantics, output order, access control, and error reporting.
* Define bounded memory, backpressure, retry, and cleanup behavior.
* Work with both Java and native execution through one initial server path.
* Give the Presto ADBC driver a batch-oriented result API without changing the
  application's ADBC API.

### Non-goals

* A new connector SPI, SQL syntax, storage reader, or training framework.
* Changing internal exchange serialization or adding direct worker downloads.
* Distributed ADBC partitions, durable result spooling, cross-coordinator resume,
  or exactly-once application processing.
* Query-long HTTP data responses, transparent midstream resume, or a separate
  result-discovery/progress/cancellation service. Use existing HTTP query
  control; evaluate broader streaming-service requirements with #63.
* Bulk ingestion, replacing JSON, or replacing Flight SQL.
* Supporting every Presto type in the first transport version through lossy
  implicit conversions.

## Proposed Implementation

### 1. User-visible flow

```text
Application / Presto ADBC driver
    |
    | POST SQL; GET nextUri                 JSON control responses
    | GET arrowData.dataUri                 Arrow IPC stream responses
    v
Existing HTTP entry point / gateway
    |
Java coordinator
    | statement lifecycle + bounded result-segment cache
    | Presto Page / Block -> Arrow vectors -> IPC writer
    ^
    | existing SerializedPage exchange
Java workers or Prestissimo / Velox workers
```

The driver loads one bounded segment, validates it, and exposes its record
batches. It follows `nextUri` only after consuming that segment. Fetching results
is serial in version 1; asynchronous prefetch and parallel endpoints are not
part of the contract.

An HTTP segment may contain multiple Arrow record batches. Neither segment
boundaries nor Arrow batch boundaries define a training minibatch; the
application may combine or subdivide batches. Size segments for transport and
memory efficiency rather than issuing one data GET per training minibatch.

### 2. Opt-in and compatibility

All headers, JSON fields, resource paths, and configuration keys below are
**proposed additions**, not existing supported APIs.

```http
POST /v1/statement
X-Presto-User: example_user
X-Presto-Catalog: hive
X-Presto-Result-Format: arrow-ipc-v1
X-Presto-Arrow-Compression: zstd
Content-Type: text/plain
Accept: application/json

SELECT entity_id, feature_a, feature_b, label
FROM hive.training.fact_events
WHERE ds = '2026-10-01'
```

* No format header means today's JSON behavior, including existing
  `binaryResults` behavior. `arrow-ipc-v1` is mutually exclusive with
  `binaryResults=true`; the combination returns HTTP 400 before dispatch.
* Version 1 supports IPC body compression `none` and `zstd`; omitted compression
  means `none`. Unsupported format/version/codec or a disabled Arrow feature
  returns HTTP 406 **before query creation** on a server implementing this RFC.
  A compression header without the Arrow format is rejected with HTTP 400.
* Every control response for an accepted Arrow query, including queued and
  terminal responses, acknowledges `resultFormat: "arrow-ipc-v1"` and
  `arrowCompression`. Both are fixed for the query and preserved by server-side
  retries before output publication. Later headers cannot change them.
* An older server may ignore unknown headers and execute the SQL normally.
  Therefore, the client must verify the acknowledgement; it cannot infer support
  from HTTP 200, a version string, or an empty `data` field.
* The initial driver exposes explicit `json` and `arrow` transport choices;
  JSON remains the default. If Arrow was required but not acknowledged, cancel
  the original query where possible and report unsupported transport. Do not
  automatically submit the SQL again. This matters especially for DML.
* After Arrow is selected, there is no mid-query JSON fallback. Unsupported
  result types produce a query error before the first data descriptor.

JSON control responses retain their existing HTTP response headers, including
catalog/schema, session, prepared-statement, and transaction updates. Clients
process these on every control response, not just on submission. A data request
does not establish a second SQL session or carry session mutations.

### 3. Control envelope and data resource

When results are available, the control response includes an optional
`arrowData` object. This example is abbreviated; the digest is symbolic:

```json
{
  "id": "20261001_120000_00001_example",
  "resultFormat": "arrow-ipc-v1",
  "arrowCompression": "zstd",
  "columns": [
    {"name": "entity_id", "type": "bigint"},
    {"name": "feature_a", "type": "double"},
    {"name": "feature_b", "type": "double"},
    {"name": "label", "type": "bigint"}
  ],
  "data": null,
  "binaryData": null,
  "arrowData": {
    "dataUri": "https://presto.example/v1/statement/arrow/20261001_120000_00001_example/7?slug=opaque",
    "segmentId": "7",
    "byteLength": 1048576,
    "rowCount": 65536,
    "sha256": "<64 lowercase hexadecimal digits>"
  },
  "nextUri": "https://presto.example/v1/statement/executing/20261001_120000_00001_example/8?slug=opaque",
  "stats": {"state": "RUNNING"},
  "error": null,
  "warnings": []
}
```

`segmentId` is an opaque identifier unique within the query, not a row offset.
`byteLength` and `sha256` describe the exact IPC response bytes, including the
schema, compressed message bodies where selected, and the end marker.
`rowCount` counts rows across all record batches in the segment.

The new resource is:

```http
GET /v1/statement/arrow/{queryId}/{segmentId}?slug=opaque
Accept: application/vnd.apache.arrow.stream

HTTP/1.1 200 OK
Content-Type: application/vnd.apache.arrow.stream
Content-Length: 1048576
Cache-Control: private, no-store, no-transform

<Arrow schema, record batches, explicit IPC end-of-stream marker>
```

Each body is a self-contained IPC **stream**, not an IPC file, with the same
schema and schema metadata across a query. The server uses IPC metadata version
V5 and little-endian buffers. Version 1 emits ordinary arrays, flattening Presto
dictionary and run-length-encoded blocks rather than requiring Arrow dictionary
or run-end encoding support. There are no cross-segment dictionary dependencies.

An Arrow control response never carries JSON `data` or `binaryData`.
Queued/running responses may have no `arrowData`. An empty, successfully executed
rowset gets one schema-only segment with `rowCount: 0`, even if no worker page
contains rows. Control errors never include a new data descriptor.

IPC compression applies to record-batch buffers. HTTP content compression is
disabled on this resource to avoid double compression and keep byte length and
digest validation unambiguous. HTTP/1.1 and HTTP/2 are both usable; HTTP chunks
are not Arrow batch boundaries. Clients request an identity HTTP content coding
and reject a transformed response rather than silently changing the contract.

### 4. Replay, acknowledgement, and completion

The control token and the retained data segment form one atomic result entry.
The current `Query` already caches the last control result for repeated token
requests; Arrow extends that entry to own the immutable encoded segment.

1. Generate and fully encode a bounded segment **before** publishing its
   descriptor. Reserve its memory before consuming exchange pages. An encoding
   failure fails the query rather than publishing a partial segment.
2. Retain the exact encoded bytes. A repeated control token returns the same
   envelope and descriptor; repeated data GETs return those same bytes while
   retained. A data GET does not poll the exchange or advance any token.
3. The client reads and validates the complete segment before exposing any of
   its batches: length, SHA-256, IPC framing/end marker, schema, and row count.
   This costs bounded client buffering but permits replay after a truncated GET
   without duplicating batches already delivered to the application.
4. Following the envelope's `nextUri` acknowledges its segment. It releases the
   previous replay entry and atomically installs the next one. A client must
   not advance until it no longer needs to retry or consume the previous
   segment. Old tokens/segments return HTTP 410; unknown/future ones return 404.
5. **Every envelope with `arrowData` has a `nextUri`, including the last data
   segment.** After acknowledging that segment, the client continues polling
   until a terminal control response has no `nextUri` and no `arrowData`.
   Only a terminal response without `error` means query success.

The extra final control response is intentional. An IPC end marker means only
"this segment ended," not "the query succeeded." Arrow permits stream closure
as an end condition, so merely receiving a clean EOF is also insufficient to
detect all truncations; the descriptor validation above is required.

Execution can fail after earlier segments were delivered. Such rows are
provisional: the driver reports the terminal Presto error, and consumers that
need all-or-nothing results must discard or roll back their partial output.
If a query fails while an already published segment remains outstanding, keep
that immutable entry replayable until acknowledgement or expiry and then expose
the failure. A repeated cached control response is not a fresh success signal.

The existing no-retry-after-output rule must apply when an Arrow descriptor is
published, not when JSON rows are emitted or a data GET completes. In particular,
automatic same-cluster or cross-cluster re-execution cannot occur after
publication. Pre-output retries must preserve format selection and identity.
Coordinator loss fails the reader; version 1 does not resume a query elsewhere.
Ambiguous submission failures do not authorize replaying the POST.

Only one logical consumer may advance a query. Concurrent requests for the same
control token share one generation operation. Data GETs use reference-counted
handles so advancing cannot free bytes still being written. Limit concurrent
data GETs per query and reserve their retained bytes; an advance that cannot
reserve the bounded old-plus-new overlap waits asynchronously or returns a
retryable HTTP 429 without acknowledging the old token. No unbounded queue of
segments is created.

### 5. Errors, cancellation, and slow readers

Query execution/conversion failures use the existing `QueryError` structure in
the control protocol. Data-resource failures use HTTP status codes, never JSON
inserted into an Arrow response:

| Condition | Behavior |
|---|---|
| Authentication or ownership check fails | Existing authentication/authorization error policy; no result bytes |
| Segment expired or acknowledged | HTTP 410; fail the reader, do not rerun SQL |
| Unknown query/segment or invalid query capability | HTTP 404, consistent with non-disclosure policy |
| Temporary admission/concurrency limit | HTTP 429 with `Retry-After`; retry the same URI |
| Connection breaks during transfer | Discard unexposed segment and retry the same URI within its retention window |
| Integrity or schema validation fails | Fail the reader and cancel; do not accept a partial result |
| Query fails after some segments | Finish the control protocol with `QueryError`; no successful reader EOF |

Transient transport/5xx retries are bounded and apply only to GETs, never to
resubmission. An HTTP 200 whose body is not Arrow is a protocol error.

Closing the reader issues the existing full-query cancellation DELETE using
its current control `nextUri`. `partialCancelUri` cancels a stage, not the whole
query, and is not used as the reader's cancellation mechanism. Cancellation and
expiry release replay bytes, Arrow allocations, and exchange resources; active
HTTP writers are stopped and their references released safely. A disconnected
data GET alone does not cancel the query because it may be retried.

Repeated GETs of the current **control** URI are idempotent heartbeats while the
application works on a batch. Active data transfers also record heartbeats.
Heartbeats extend the idle timeout but do not create new segments. They do not
extend an administrator-configured maximum segment retention deadline or the
query's maximum lifetime. Expiry cancels an unfinished query and invalidates the
segment; a reader must treat expiry as failure, even if it had seen `FINISHED` in
earlier statistics.

The initial POST returns control only; it does not publish a segment. Thus any
outstanding segment always has a repeatable GET control URI for heartbeats.
After final acknowledgement, ordinary query cleanup can remove its protocol
state. Before it, completed execution alone must not evict unread Arrow data;
retention and idle limits, rather than an unbounded finished-query cache, apply.

### 6. Schema and supported types

Validate the entire output schema recursively before publishing the first
segment. Each field includes `presto.type` with its original SQL type signature;
schema metadata includes `presto.arrow.mapping.version = "1"`. Mapping is
independent of the connector that produced the data.
Use conservative nullable fields unless non-nullability is guaranteed by the
result contract (such as map keys); observed non-null batches do not narrow the
schema.

The required initial mapping is:

| Presto result type | Arrow representation | Contract |
|---|---|---|
| `BOOLEAN` | `Bool` | Preserve nulls |
| `TINYINT`, `SMALLINT`, `INTEGER`, `BIGINT` | Signed `Int8/16/32/64` | No narrowing |
| `REAL`, `DOUBLE` | `Float32/64` | Interpret `REAL` bits correctly; preserve infinities, NaN values, and signed zero, without promising NaN payload identity |
| `DECIMAL(p,s)` | `Decimal128(p,s)` | Exact unscaled integer and scale, including short/long decimals |
| `VARCHAR`, `CHAR(n)` | `Utf8` | `CHAR` uses SQL value semantics, including padding; record length/type metadata |
| `VARBINARY` | `Binary` | Raw bytes, not base64 |
| `DATE` | `Date32` | Signed days since the Unix epoch |
| `TIME` | `Time32(MILLISECOND)` | Wall-clock milliseconds since midnight; initially only non-legacy timestamp semantics |
| `TIMESTAMP` | `Timestamp(MILLISECOND, timezone absent)` | Wall-clock value with no implied UTC zone; initially only non-legacy timestamp semantics |
| `ARRAY<T>` | `List<T>` | Null list, empty list, and null elements remain distinct |
| `MAP<K,V>` | `Map<K,V>`, `keysSorted=false` | Non-null keys; preserve values and entry sequence without promising SQL map ordering |
| `ROW(...)` | `Struct` | Preserve field order and parent/child nullability |
| `UNKNOWN` (untyped null) | `Null` | Preserve row count even for all-null results |

Top-level output names, including duplicates, are preserved; consumers must be
able to address columns by ordinal. Unnamed row fields receive deterministic
ordinal-based names and metadata indicating that the SQL field was unnamed.
No application should use generated field names as column identity.

Temporal mapping is a correctness boundary, not just a choice of vector class.
The millisecond mapping describes the currently supported Presto types; higher
precision types must not be silently truncated into it. A future precision
extension needs a specified mapping and conformance tests. Legacy timestamp
sessions with `TIME`/`TIMESTAMP` output are rejected initially, rather than
shipping session-dependent raw longs as portable Arrow timestamps.

`TIME WITH TIME ZONE` and `TIMESTAMP WITH TIME ZONE` are **not supported in
mapping version 1**. Presto can preserve a zone per value, whereas an Arrow
timestamp has one zone attribute per field. Silently converting to a zone-less
timestamp or UTC would lose information. Applications may explicitly cast or
normalize in SQL when that is their intended result. A later mapping could
define a struct containing the instant/local time and zone identifier, shared
with #63, but it must not change version 1's schema silently.

Other types, including JSON, UUID, IP address, intervals, sketches, and
connector-specific types, are initially rejected unless explicitly cast in
SQL to a supported type. The query error identifies the column/nested path and
unsupported type. This is preferable to inventing an undocumented binary
encoding or silently serializing an object to text.

### 7. Server integration and SPI surface

Public source references below are pinned to the inspected Presto revision:

| Module / implementation | Proposed change |
|---|---|
| `presto-main`, [queued statement resource][queued-source] | Validate format/codec before dispatch; persist immutable selection across queued/executing transitions and pre-output retries |
| `presto-main`, [executing statement resource][executing-source] and [`Query`][query-source] | Reuse token lifecycle, output metadata, and `PagesSerde`; add Arrow generation/replay state instead of `RowIterable` materialization |
| `presto-main`, [`LocalQueryProvider`][provider-source] | Coordinate query purging with bounded unread-result retention; close owned resources on expiry, cancellation, and shutdown |
| `presto-main`, [query response utilities][response-source] | Preserve control headers and rewrite the new `dataUri` through the same validated external-prefix/routing mechanism as other query URIs |
| `presto-main`, new `ArrowResultResource` | Authenticate, authorize, locate, and serve immutable IPC bytes; never independently drain exchange pages |
| `presto-client`, [`QueryResults`][results-source] | Add optional format/codec acknowledgement and typed `ArrowData` descriptor; absent fields preserve legacy behavior |
| `presto-common-arrow`, [`BlockArrowWriter`][writer-source] | Reuse and extend block/vector conversion with an explicit client-result mapping policy and session-aware validation |
| `presto-common-arrow`, [`ArrowBatchSource`][batch-source] | Reuse suitable batching logic, but separate page conversion from its existing `ConnectorPageSource` input contract |

The existing shared writer includes temporal mappings that discard zone
information; its mere existence is not proof of conformance with the public
result contract. Audit each supported type, including `CHAR` and decimals.
Do not change existing connector/federation mappings underneath their users.
Factor reusable primitives and add an explicit mapping policy for client
results, with independent regression tests for existing callers.

The coordinator consumes query-output `Page`s, not a fake connector page source.
Implementation boundaries could look like the following **engine-internal**
interfaces; these are illustrative, not proposed plugin contracts:

```java
interface QueryOutputArrowEncoder extends AutoCloseable
{
    Schema schema();
    // Append a bounded region; caller retains any unconsumed page suffix.
    void append(Page page, int offset, int length);
    EncodedResultSegment finishSegment();
}
```

`EncodedResultSegment` owns immutable bytes, descriptor metadata, and their
memory reservation. Encoding jobs use a bounded executor. Reserve/generate/
publish transitions serialize per query, but expensive conversion and socket
I/O do not run under a global query-manager lock. Query finalization cannot
observe drained final pages before their segment has been installed or failed.

`LocalQueryProvider` currently drops protocol entries when the query manager
forgets a query. The implementation must retain the terminal status/session
updates needed for the final acknowledgement and keep the bounded protocol
entry alive independently until acknowledgement or expiry. It must not depend
on a later `getFullQueryInfo` succeeding after that query has been purged.

**SPI changes: none.** No `Connector`, `ConnectorPageSource`, `BlockEncodingSerde`,
SQL grammar, worker task protocol, or `presto_protocol.yml` additions are
required. Arrow types stay out of `presto-client` and public connector SPI
signatures; only a lightweight JSON descriptor is added to the client protocol.
Reuse the repository's Arrow dependency versions. Package vector, allocator,
and compression dependencies in coordinator distribution/wiring, without pulling
in Flight/gRPC or claiming that a disabled feature removes packaged libraries.

Native workers continue producing their existing serialized pages; the Java
coordinator deserializes and converts them just as for Java workers. The Velox
Arrow C Data bridge is an in-process interoperability mechanism, not by itself
an HTTP wire format. Moving conversion to workers would additionally require
IPC serialization, ordering, authorization, routing, and replay design; defer
that to separate work.

#### Statements without ordinary rowsets

Format selection does not classify the SQL as read-only. Preserve existing
`updateType`/`updateCount`, transaction, prepared-statement, and session behavior.
The current update-count extraction depends on JSON `data`; refactor it to
inspect the typed output page before either encoder runs. If the existing
protocol synthesizes a result row for a no-output statement, encode the same
logical row in Arrow, not a special JSON exception. This includes coordinator-
only statements and metadata queries whose output types are supported.

### 8. Memory, backpressure, and deployment

Version 1 retains only bounded result state, not the full query result. Stop
polling the exchange when the outstanding segment has not been acknowledged;
existing exchange/output-buffer limits then propagate backpressure to workers.
This does not remove the exchange buffers already present in the coordinator.

Proposed coordinator settings:

| Property | Initial policy |
|---|---|
| `query-results.arrow.enabled` | `false` |
| `query-results.arrow.max-batch-rows` | 65,536, additionally constrained by bytes |
| `query-results.arrow.target-segment-size` | 4 MiB encoded target, not a hard maximum |
| `query-results.arrow.max-segment-size` | 16 MiB hard limit on complete encoded IPC bytes, including schema/framing |
| `query-results.arrow.max-uncompressed-segment-size` | 64 MiB hard limit on uncompressed IPC bytes, including schema/framing |
| `query-results.arrow.max-memory-per-query` | Required finite budget for pages, vectors, encoding scratch, and replay overlap; initial default to be chosen from allocation tests |
| `query-results.arrow.max-memory` | Required finite coordinator-wide Arrow budget; default sized before enabling in a distribution |
| `query-results.arrow.max-concurrent-encoders` | Bounded concurrency; default chosen by load testing |
| `query-results.arrow.max-segment-retention` | Finite hard deadline independent of heartbeats; initial candidate 5 minutes |

These byte/row values are starting points for evaluation, not performance
claims. Existing query timeout and client-abandonment policies also apply.
Operators must size memory/concurrency explicitly during the experimental
rollout; production defaults are a release gate.

Page size and row count alone cannot enforce these limits. Split page regions
and account for variable-width/nested values, offset buffers, allocator rounding,
compression scratch, and source-page retention. Bound source-page admission and
vector growth before allocation; recompressing an already oversized allocation
is not a safety mechanism. A schema, one row, or an upstream page that cannot
fit the configured reservations fails with an actionable resource-limit error.
Do not bypass the cap because one value is large. Retaining a page suffix also
retains and accounts for its backing buffers.

Arrow child allocators, heap-resident replay bytes, and overlapping in-flight
read references must all count toward the per-query and global budgets. Bridge
allocator reservations into Presto's query memory tracking rather than relying
only on an independent Arrow allocator limit. On a finished engine query, the
protocol result owner retains and releases those reservations until final
acknowledgement or expiry.

On HTTP deployments, reuse the configured authenticators, TLS, query identity,
access-control evaluation, and resource-group admission. Every data GET validates
the query capability and, in authenticated deployments, the requesting principal
and effective user against the query's permitted identity. A guessed query ID
must not retrieve results. Do not interpret `X-Presto-User` alone as proof of
identity. Row filters and column masks remain upstream of result encoding.
Apply the same identity checks to Arrow control advancement, replay, and
cancellation, so the data endpoint is not the only protected part of the new
protocol.

The data URI is sensitive: redact its capability from logs, use `no-store`,
and do not return public object-store links. Clients may forward credentials
only to the configured trusted coordinator/gateway origins, not arbitrary
redirect targets. Unauthenticated deployments retain their existing trust
model; Arrow does not make them secure.

Gateways must route the new URI to the owning coordinator, preserve binary
bodies/content type, apply external URI rewriting, and permit sufficiently large
bounded responses. Proxy buffering, timeouts, auth challenges, and compression
middleware need integration tests. This reuses HTTP infrastructure but does
**not** promise that every existing gateway works without configuration changes.

### 9. Client and ADBC integration

Add a batch-oriented HTTP query reader alongside the Go client's row-oriented
`database/sql` path. Reuse its HTTP transport, authentication, session-header
handling, cancellation, and safe GET retry policy; do not decode Arrow batches
into Go row values just to rebuild them with `sqlwrapper`.

The ADBC driver's query execution path selects this reader in Arrow mode.
Existing metadata/ingest implementations may remain on the JSON path, but both
paths must share connection/session state and cannot independently mutate a
transaction or session concurrently. Application-visible schema follows the
negotiated version 1 mapping, which may differ from the existing JSON driver's
textual representations; that is part of opting in, not an invisible change.

Illustrative reader state machine:

```text
control = submit(sql, format="arrow-ipc-v1", compression="zstd")
require_format_acknowledgement(control)
while true:
    apply_control_headers(control)
    if control.error:
        fail_reader(control.error)          # even after earlier batches
    if control.arrowData:
        bytes = fetch_same_uri_with_bounded_retries(control.arrowData)
        validate_length_digest_ipc_schema_and_row_count(bytes)
        reader = standard_arrow_ipc_reader(bytes)
        yield_all_batches_with_correct_buffer_lifetimes(reader)
    if control.nextUri is absent:
        finish_reader_successfully()
        break
    control = get(control.nextUri)          # acknowledges previous segment
```

This pseudocode omits idle heartbeats and cleanup for brevity. `Release`/close
cancels an unfinished query, and no success is reported until the terminal
control response. A retained application batch must keep its buffers alive
after the segment reader advances; the driver cannot reuse them prematurely.
Client buffering and decoded allocations have separate finite limits, including
validation of declared compressed-buffer lengths before decompression.

## Other Approaches Considered

| Approach | Why not choose it for this RFC? |
|---|---|
| Flight SQL (#63) | Preferred direction when a standardized streaming service is needed. This RFC instead extends existing HTTP result pagination; share Arrow conversion and conformance work, not a second streaming-service protocol. |
| Existing `binaryResults=true` | Avoids JSON value formatting, but retains base64 and Presto-specific decoding/conversion in clients. Useful as a benchmark, not a new public Arrow contract. |
| Arrow bytes base64-encoded inside `QueryResults` | Simple envelope change, but adds base64 work and expansion; clients still parse a large JSON payload. |
| Whole `/v1/statement` response negotiated as Arrow | Removes the extra data GET, but has no simple place for queued state, session updates, statistics, and errors after headers have been sent. |
| One query-long HTTP Arrow stream | Could reduce requests and first-batch buffering, but robust recovery and separate query control add scope overlapping Flight. A simple fail-on-disconnect stream is possible, but no measured benefit currently justifies changing the existing lifecycle. Not part of this RFC or an assumed follow-up phase. |
| Multipart JSON + Arrow responses | Could save a round trip, but adds multipart framing/client work and makes retry boundaries less straightforward. |
| JSON-to-Arrow sidecar | Works with unchanged servers but leaves server JSON serialization and client/gateway JSON parsing in the critical path. |
| Worker-native Arrow immediately | Potentially reduces coordinator conversion cost, but changes distributed execution and serving. First establish correctness and measure whether coordinator conversion is the limiting factor. |
| Durable spooling to object storage | Better decoupling for very slow or resumable consumers, but introduces storage lifecycle, credentials, costs, and a different reliability contract. |

The chosen design adds a data GET and segment buffering. Those costs may matter
for tiny results; JSON remains available. Larger bounded segments amortize the
extra round trip at the cost of memory and latency. HTTP connections can be
reused across segment requests; segmentation does not require a new connection
or TLS handshake per batch.

## Adoption Plan

1. Agree on the envelope/replay contract and the type mapping with #63 and ADBC
   maintainers. Build shared conversion/conformance work first where practical.
2. Implement behind the disabled-by-default server flag. No existing REST mode,
   JDBC/CLI behavior, connector SPI, or SQL feature is deprecated.
3. Add explicit Arrow transport selection to the Presto Go/ADBC clients. Keep
   JSON as their default; unsupported servers/types fail clearly in Arrow mode.
4. Canary through real HTTP entry points on Java and native clusters. Enable for
   selected batch consumers only after correctness, memory, and cancellation
   gates pass. Do not automatically execute user SQL twice for comparisons:
   shadow benchmarks use explicitly approved read-only workloads.
5. Publish protocol/version, supported-type, retention/memory, gateway, and
   client examples. Pin the cross-language Arrow versions tested. Consider a
   future negotiated `auto` mode separately once safe compatibility behavior
   and schema expectations are established.

The rollout delivers the bounded-segment path only; it does not require a
second query-long HTTP implementation. Shared conversion/type-mapping fixtures
should remain transport-independent so later Flight SQL work can reuse them
without requiring this HTTP protocol to evolve into Flight.

Disable the feature for **new submissions** to roll back; already accepted Arrow
queries must either finish in Arrow mode or receive a clear cancellation/error.
Do not change their format midstream. Client rollback is explicit JSON selection
for new queries. No data migration is required.

## Metrics and Test Plan

No Arrow-over-HTTP prototype or performance result is claimed by this RFC.
The shared Arrow conversion code is useful prior work, not end-to-end evidence.

### Correctness and compatibility

* Extend `presto-client`'s `TestQueryResults` with old/new envelope round trips,
  omitted fields, format acknowledgement, and mutually exclusive data variants.
* Extend server statement tests (`presto-main`'s `TestServer` and focused new
  protocol tests) for queueing, empty results, many segments, final
  acknowledgement, metadata/coordinator-only statements, update counts,
  session changes, prepared statements, and transaction headers.
* Test an old client against the new server with no opt-in, and an Arrow-required
  client against an old/disabled server. Verify unsupported format/codec is
  rejected before dispatch on an aware server; ignored headers on an old server
  must not cause a duplicate SQL execution.
* Test each supported type and nested combination on both engines: nulls,
  integer boundaries, decimal signs/precision/scale, `CHAR` padding, Unicode,
  empty versus null collections, dictionary/RLE input blocks, all-null columns,
  duplicate names, negative epoch values, DST boundaries, and ordered results.
  Assert explicit rejection of zoned temporal types, legacy temporal output,
  unsupported nested types, and oversized values.
* Share public IPC fixtures across Arrow Java, Go, and Python (at minimum),
  covering `none`/`zstd`, V5 framing, schema-only results, and buffer ownership.
  Compare against typed expected values, not just potentially lossy JSON text.
  Run applicable Presto ADBC conformance tests for the batch path.

### Failure and resource tests

* Drop connections in the schema, a record-batch body, between batches, and just
  before EOS. Retry the same segment and verify identical bytes and no duplicate
  batch delivery. Test a valid IPC prefix with missing later batches.
* Repeat and race control/data GETs; lose a control response after generation;
  advance while an old data GET is active; expire an acknowledged URI; request
  an unknown token. Verify one exchange drain per control token.
* Fail execution before and after publication, including after the last data
  segment but before terminal status. Ensure no automatic query re-execution
  after publication and no successful reader EOF on error.
* Pause consumers, send heartbeats, omit heartbeats, hit hard retention limits,
  cancel during encoding/transfer, and restart the coordinator. Verify bounded
  reservations and cleanup on every path, including finished engine queries.
* Stress incompressible strings, highly compressible large values, nested
  offsets, schema size, concurrent readers, and allocation failures. Account
  for encoder scratch, source-page suffixes, replay overlap, and client
  decompression, not just final payload size.
* Exercise configured authentication modes and principal/impersonation rules,
  invalid/stolen capabilities under a different identity, row filters/masks,
  credential-expiry refresh, TLS, and the actual gateway's routing, rewriting,
  timeout, and compression behavior.

### Performance evaluation

Use the same queries, data, client host, concurrency, and execution engine for:

1. JSON results with the deployment's normal HTTP compression.
2. Existing serialized-page/base64 results where a compatible decoder exists.
3. Arrow IPC over HTTP with no IPC compression and with ZSTD.
4. Flight SQL if an implementation is available.

Cover narrow/wide schemas, fixed/variable-width and nested data, small/large
results, local/high-latency links, and slow/fast consumers. Sweep segment byte
targets and Arrow batch sizes independently, with HTTP connection reuse enabled.
For the ML workload, measure time spent waiting for result requests versus
conversion, dimension lookups, and training-data preparation before attributing
an ingestion bottleneck to segmentation. Report:

* Time to first application batch, total elapsed time, rows/s, and wire bytes.
* Coordinator conversion/compression CPU, heap, off-heap memory, GC, and peak
  retained bytes; worker and client CPU/memory separately.
* Segment count/size, compression ratio, data GET latency, retry count,
  allocator/admission failures, cancellation/expiry cleanup time, and time
  blocked on consumers.
* Per-segment control/data request overhead and buffering latency versus
  conversion CPU and useful consumption time, to guide segment-size tuning.
* Ordinary JSON-query throughput/latency under mixed Arrow and JSON load.

Expose these through existing coordinator metrics/JMX using bounded labels
(format/codec/outcome, not SQL, user, or query-ID labels). Keep per-query detail
in query diagnostics.

Release gates are exact value/schema conformance, unchanged non-opted-in
behavior, enforced memory limits, no leaks after cancellation/expiry, and a
measured CPU/throughput benefit on representative bulk-result workloads without
unacceptable impact on other queries. Numeric performance targets and resource
defaults will be agreed from the prototype results, not asserted in advance.

## Open Questions for Review

1. Should HTTP Arrow and #63 share a named client-result mapping policy in
   `presto-common-arrow`, with the strict initial temporal scope above?
2. Is one replayable segment per control token, plus a mandatory final control
   acknowledgement, the right initial latency/reliability trade-off?
3. Which resource defaults and gateway/client combinations must be qualified
   before this is enabled outside the experimental rollout?

[query-source]: https://github.com/prestodb/presto/blob/a872d5a97f9c2270532532fbc04d94e7e074eef6/presto-main/src/main/java/com/facebook/presto/server/protocol/Query.java
[queued-source]: https://github.com/prestodb/presto/blob/a872d5a97f9c2270532532fbc04d94e7e074eef6/presto-main/src/main/java/com/facebook/presto/server/protocol/QueuedStatementResource.java
[executing-source]: https://github.com/prestodb/presto/blob/a872d5a97f9c2270532532fbc04d94e7e074eef6/presto-main/src/main/java/com/facebook/presto/server/protocol/ExecutingStatementResource.java
[response-source]: https://github.com/prestodb/presto/blob/a872d5a97f9c2270532532fbc04d94e7e074eef6/presto-main/src/main/java/com/facebook/presto/server/protocol/QueryResourceUtil.java
[provider-source]: https://github.com/prestodb/presto/blob/a872d5a97f9c2270532532fbc04d94e7e074eef6/presto-main/src/main/java/com/facebook/presto/server/protocol/LocalQueryProvider.java
[results-source]: https://github.com/prestodb/presto/blob/a872d5a97f9c2270532532fbc04d94e7e074eef6/presto-client/src/main/java/com/facebook/presto/client/QueryResults.java
[writer-source]: https://github.com/prestodb/presto/blob/a872d5a97f9c2270532532fbc04d94e7e074eef6/presto-common-arrow/src/main/java/com/facebook/plugin/arrow/BlockArrowWriter.java
[batch-source]: https://github.com/prestodb/presto/blob/a872d5a97f9c2270532532fbc04d94e7e074eef6/presto-common-arrow/src/main/java/com/facebook/plugin/arrow/ArrowBatchSource.java
[adbc-factory]: https://github.com/adbc-drivers/presto/blob/f1bd473c41418622145d697e53563beab47da07c/go/db_factory.go
[adbc-statement]: https://github.com/adbc-drivers/presto/blob/f1bd473c41418622145d697e53563beab47da07c/go/statement.go
