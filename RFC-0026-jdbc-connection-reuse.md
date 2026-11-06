# **RFC-0026 for Presto**

See [CONTRIBUTING.md](CONTRIBUTING.md) for instructions on creating your RFC and the process surrounding it.

## Connection reuse and pooling for JDBC connectors

Proposers

* Nandakumar Balagopal (@Nandakumar-Balagopal)

## Related Issues

* [prestodb/presto#27638](https://github.com/prestodb/presto/pull/27638) - open PR giving the Oracle connector the pool that ships with its driver.
* [prestodb/presto#27536](https://github.com/prestodb/presto/issues/27536) - `jdbc-fetch-size`, added once in `presto-base-jdbc` for every JDBC connector.
* [trinodb/trino#14653](https://github.com/trinodb/trino/pull/14653) - Trino's `ReusableConnectionFactory`. Phase 1 below follows its design.
* [trinodb/trino#6719](https://github.com/trinodb/trino/issues/6719), [trinodb/trino#7069](https://github.com/trinodb/trino/pull/7069) - Trino's `LazyConnectionFactory`, for connections opened during planning and never used.
* [trinodb/trino#15888](https://github.com/trinodb/trino/issues/15888) - request for generic JDBC pooling in Trino, open since January 2023. The only reply suggests putting PgBouncer in front of the database.
* [trinodb/trino#19205](https://github.com/trinodb/trino/issues/19205) - Trino's Oracle pool ignores per-user credentials. Still open; the fix proposed in [#30713](https://github.com/trinodb/trino/pull/30713) was closed without being merged.
* [trinodb/trino#6801](https://github.com/trinodb/trino/issues/6801) (fixed), [trinodb/trino#30177](https://github.com/trinodb/trino/issues/30177) (open) - Trino's Oracle pool runs out of connections under concurrent queries.

## Summary

Two opt-in changes in `presto-base-jdbc`, proposed as two phases:

1. **Reuse within a query.** The metadata calls of one query share a connection instead of opening one each. Nothing is reused across queries, and splits and page sinks are not affected.
2. **An optional shared pool.** One pool per set of credentials, which a connector can replace with its driver's own pool or refuse.

## Background

`BaseJdbcClient` opens a new connection for every metadata call and closes it before returning. Reading `information_schema.columns` for a schema of 21 tables and 21 views makes the PostgreSQL connector open 88 connections, and over a 24 ms link that query takes 18 s.

No JDBC connector in Presto pools connections today. The only proposal is [#27638](https://github.com/prestodb/presto/pull/27638) for Oracle, whose driver ships a pool, and it is still open.

### Goals

* Fewer connections per query for metadata, with no change for connectors or catalogs that do not opt in.
* A pool that is safe with credential pass-through and under concurrent reads and writes.
* Let a connector use its driver's pool instead, or refuse pooling.

### Non-goals

* Reusing connections across queries or users.
* Changing how splits and page sinks get their connections, beyond the optional pool.
* Reducing the number of metadata queries sent to the database. That is a separate change, described under Other Approaches Considered.

## Proposed Implementation

A prototype of both phases is at [Nandakumar-Balagopal/presto@jdbc-connection-pool](https://github.com/prestodb/presto/compare/master...Nandakumar-Balagopal:presto:jdbc-connection-pool), as two commits against master, one per phase. Each builds and its tests pass on its own.

### Modules

Mostly `presto-base-jdbc`. A connector that sets connection properties of its own builds `DriverConnectionFactory` through the constructor that takes no config, which has nothing to read the catalog's settings from, so such a catalog would accept the new properties and ignore them. MySQL and Oracle are in that position today and each needs a one-line change to pass the config. A connector that overrides metadata methods and wants reuse there also switches those call sites to the new method below.

Each connector's plugin directory gains Commons DBCP 2 and Commons Pool 2, which `presto-base-jdbc` declares, so a connector built against it picks them up without a change of its own.

### Terms used below

* **Metadata connection:** opened by a metadata call, which closes it before returning. These are the only connections that are reused.
* **Data connection:** held by a split or a page sink across many calls, possibly on different threads. These are never reused.
* **Kept connection:** a metadata connection held for a short time after it is closed, so that the next metadata call of the same query can take it.
* **Dirty connection:** a connection whose session a caller changed (read-only, auto-commit, catalog, schema, isolation, savepoints, and similar). It is closed instead of kept.
* **Credential set:** the connection properties a pool is keyed by, including user and password, which differ per user when credentials are passed through.

### Interfaces

`ConnectionFactory` gets two default methods, so existing implementations still compile:

```java
public interface ConnectionFactory extends AutoCloseable
{
    // Splits, page sinks and any existing caller. Unchanged, never reused.
    Connection openConnection(JdbcIdentity identity) throws SQLException;

    // Metadata calls. May return a connection another metadata call of the same query used.
    default Connection openConnection(ConnectorSession session, JdbcIdentity identity) throws SQLException
    {
        return openConnection(identity);
    }

    // The query in the session has ended; release anything kept for it.
    default void cleanupQuery(ConnectorSession session) {}
}
```

`JdbcClient` gets a default `cleanupQuery(ConnectorSession)`. `JdbcMetadata.cleanupQuery` calls it, and `BaseJdbcClient` passes it to the connection factory. The engine already calls `ConnectorMetadata.cleanupQuery` when a query finishes (`QueryStateMachine`), so no SPI change is needed.

The metadata methods of `BaseJdbcClient` that have a session call the new method: `getSchemaNames`, `getTableNames`, `getTableHandle`, `getColumns`, `createTable`, `addColumn`, `renameColumn`, `dropColumn`, `dropTable`, `truncateTable`.

Pooling is internal to `DriverConnectionFactory`. A connector that wants its driver's pool supplies its own `ConnectionFactory`, which is where a driver pool such as Oracle's UCP ([#27638](https://github.com/prestodb/presto/pull/27638)) fits. Reuse works on top of either.

**Opting out.** A connector that supplies its own `ConnectionFactory` is never pooled, because pooling lives only inside `DriverConnectionFactory`. A connector that does use `DriverConnectionFactory` but whose backend should not be pooled builds it through `DriverConnectionFactory.withoutPooling`, naming itself; a catalog of that connector setting `connection-pool.enabled=true` then fails to load with a message naming the property and the connector, rather than accepting the property and quietly doing nothing. Reuse is unaffected by that, since it holds no connection past the query that opened it.

Trino's pool lives inside the Oracle connector, so there is nothing for another connector to refuse, and its reuse (`query.reuse-connection`) is a catalog property that defaults to on. Here both features default to off, and a connector can refuse either.

### How it works

Reuse (`connection-reuse.enabled=true`) keys connections by query id, identity and connection properties. When a metadata call closes its connection, the connection is kept for `connection-reuse.retention`, and the next metadata call of the same query takes it, on any thread. Only one caller holds a kept connection at a time, and each caller gets its own handle, so closing twice or using a handle after closing cannot affect another caller. A metadata call nested inside another on the same thread gets the connection its caller holds, which keeps a pool smaller than the nesting depth from deadlocking.

At `cleanupQuery` the query's kept connections are closed, and a connection released after that point is closed rather than kept. The engine does not call `cleanupQuery` for metadata-only queries such as `SHOW TABLES`, so the retention time is the backstop. A dirty connection is closed on release; nothing is reset, so no reset can fail on any driver. At most `connection-reuse.max-retained` connections are kept across all queries.

The pool (`connection-pool.enabled=true`) uses Apache Commons DBCP 2. There is one pool per credential set, at most `max-credential-sets` of them, and a pool whose credentials go unused for `max-idle-time` is closed. On hand-out, read-only and auto-commit are restored, an open transaction is rolled back, and `reset-sql` runs if set. Connections are retired after `max-lifetime`, so a password change or a revoked grant is eventually picked up. A caller whose pool is evicted while it waits is served by the replacement pool.

Metadata calls wait up to `max-wait` for a connection, but splits and page sinks do not wait by default. A query that joins two tables of the catalog, or reads from it while writing to it, holds one connection and needs a second; if every connection were held by a caller waiting for another, the waiting callers would occupy the engine's threads and nothing would release them. A split therefore takes an idle connection if there is one and otherwise opens its own outside the pool. Those extra connections are capped per node by `connection-pool.max-overflow`, and once those permits are gone a caller waits for the pool after all. A node can hold at most `max-size + max-overflow` connections, which matters because running the database out of connection slots fails queries for every other client, not only for this catalog.

A connection is always checked before it is handed out. A link that drops without closing leaves every connection in the pool dead, and a dead connection that fails a query and is returned stays in the pool, so failures keep landing for as long as the pool holds one. The check costs a round trip - 1-3 ms against PostgreSQL, but 260-300 ms against Teradata, paid once per connection a query borrows - so it is skipped for a connection that was returned within `connection-pool.validation-bypass-window`, on the grounds that what makes a pooled connection stale is time spent idle. Inside that window only `Connection.isClosed` is consulted, which a driver answers without talking to the database. The check has a 5 s timeout, and a connection whose driver has already reported a connection-level failure is replaced without waiting for it.

Configuration, all per catalog:

| Property | Default |
| --- | --- |
| `connection-reuse.enabled` | `false` |
| `connection-reuse.retention` | `2s` |
| `connection-reuse.max-retained` | `10` |
| `connection-pool.enabled` | `false` |
| `connection-pool.max-size` | `10` |
| `connection-pool.max-overflow` | `10` |
| `connection-pool.max-wait` | `30s` |
| `connection-pool.max-idle-time` | `5m` (minimum `1s`) |
| `connection-pool.max-lifetime` | `30m` |
| `connection-pool.max-credential-sets` | `100` |
| `connection-pool.validation-bypass-window` | `500ms` (`0ms` checks every borrow) |
| `connection-pool.reset-sql` | unset |

## Metrics

Exposed per pool: pools created, connections borrowed, total borrow wait, connections opened outside the pool because it was exhausted, and how often a caller was refused an overflow connection and had to wait. The prototype has these as getters; exporting them through JMX is still to do.

To judge whether the change is worth having, measure:

* Connections opened per query, counted at the database (PostgreSQL `log_connections`).
* Latency of metadata queries and scans against a database with realistic round-trip time.
* How often splits open a connection outside the pool, which shows whether `max-size` is too small.

## Other Approaches Considered

**Each connector uses its driver's pool.** This works for Oracle, and this design allows it: the connector supplies its own `ConnectionFactory`. It does not work for most other drivers, which do not ship one and say so:

* PostgreSQL JDBC: "In general it is not recommended to use the PostgreSQL provided connection pool. Check your application server or check out the excellent jakarta commons DBCP project." ([docs](https://jdbc.postgresql.org/documentation/datasource/))
* Microsoft JDBC Driver for SQL Server: "it does not provide its own pooling implementation." ([docs](https://learn.microsoft.com/en-us/sql/connect/jdbc/using-connection-pooling))
* MySQL Connector/J relies on the pool of an application server or a pooling library. ([docs](https://dev.mysql.com/doc/connector-j/en/connector-j-usagenotes-j2ee-concepts-connection-pooling.html))

Writing a pool per connector also repeats the parts that are hard to get right, and each of those parts has gone wrong in the one per-connector pool that exists. In Trino's Oracle pool: per-user credentials are ignored ([#19205](https://github.com/trinodb/trino/issues/19205), open, with the fix in [#30713](https://github.com/trinodb/trino/pull/30713) closed unmerged), auto-commit is not reset so the connector resets it itself ([#8559](https://github.com/trinodb/trino/pull/8559), fixed), and concurrent inserts exhaust the pool ([#6801](https://github.com/trinodb/trino/issues/6801), fixed). In `presto-base-jdbc` each of those is fixed once for every connector; per connector, each is fixed again. The reuse in Phase 1 is not pooling at all - it changes when `BaseJdbcClient` opens connections, which only `presto-base-jdbc` can do, and Trino made the same change in the same place.

**Reuse keyed by thread, applied to all connections.** This was the first prototype. It needed no interface change, but it reused connections across queries of the same user, and it shared data connections that splits and page sinks hold across calls. Stress tests found two page sinks sharing one transaction, so an aborted insert's rows were committed, and connections handed to two threads at once. It was also slower, because it cannot reuse across the threads of one query.

**A pool that makes every caller wait.** Measured with a pool of 4: 12 concurrent `INSERT ... SELECT` queries all failed with a pool timeout, and joins hung for longer than `max-wait`, because the waiting splits held the engine's threads.

**Pool without reuse.** On the measurements in the Test Plan this is now about as quick as reuse, because the validation round trip it used to pay on every borrow is skipped inside the bypass window. It still keeps connections open between queries where reuse does not, and nested metadata calls can deadlock a pool smaller than the nesting depth, which reuse avoids by handing the nested call the connection its caller already holds.

**Metadata caching.** `metadata-cache-ttl` does not cover `information_schema` listing or splits. It also cannot be set on its own; that will be filed as a separate issue.

**HikariCP instead of DBCP 2.** Either works. DBCP 2 is already in Presto's dependency management, so the prototype uses it.

**An external proxy such as PgBouncer.** This is what the one reply on [#15888](https://github.com/trinodb/trino/issues/15888) suggests. It needs a second piece of infrastructure per database, it does not exist for every backend Presto has a JDBC connector for, and in transaction pooling mode it interacts badly with server-side prepared statements, as noted in the Test Plan.

**Opening fewer connections.** Trino's `LazyConnectionFactory` avoids opening connections that are never used, and some metadata paths could pass the connection they hold instead of opening another. Both are complementary and can be done separately.

## Adoption Plan

* **Impact on existing users:** none unless a catalog sets the new properties. The SPI additions are default methods, so out-of-tree connectors compile unchanged and keep the current behaviour.
* **Phasing:** Phase 1 (reuse) is a small change and delivers most of the benefit. Phase 2 (pool, and the connector opt-out) follows as a separate PR once Phase 1 is in; in the prototype each phase is a self-contained commit that compiles and passes the suite without the other. A later release could turn reuse on by default, as Trino does, after it has run in production.
* **Migration:** none. No behaviour is removed.
* **Documentation:** a section in the JDBC connector docs covering the properties and how to size the pool. Total connections at the database are at most nodes x credential sets x (`max-size` + `max-overflow`). It also needs a `reset-sql` value per database, and a note that driver timeouts (for PostgreSQL, `loginTimeout` and `socketTimeout`) are still needed for a database that stops responding.
* **Security:** reuse never crosses queries or identities. With the pool, Presto users who share one database account can be served the same database session one after another, so session state can pass between them unless `reset-sql` clears it. `DISCARD ALL` does on PostgreSQL. MySQL has no SQL statement for this. With credential pass-through there is one pool per user, bounded by `max-credential-sets`.
* **Out of scope:** `JdbcMetadataCache` merges concurrent lookups of the same table by different users, so one user can receive another's result or error. This happens with reuse and pooling off, and will be filed separately. Lazy connections and retrying failed connection attempts, which Trino has, are also left for later.

## Test Plan

**Unit tests (H2, in the prototype, 43 tests):**

* Nested calls share a connection. Reuse never crosses queries or identities.
* A kept connection moves to another thread of the same query, but a held one is not shared across threads.
* `cleanupQuery` closes kept connections, including one released after the query ended.
* Dirty connections are closed, not kept.
* Double close and use after close.
* The retention cap and expiry.
* Splits and page sinks never share a connection, including two writers interleaved on one thread and closed on different threads.
* Splits do not wait on an exhausted pool, while metadata calls do and time out.
* A waiter on an evicted pool is served by the replacement.
* A connection that died while idle is not handed out; borrows inside the bypass window skip the check and borrows outside it do not.
* A connector that refuses pooling rejects `connection-pool.enabled` naming itself, loads without it, and a connector passing its own connection properties still gets a pool.
* `ForwardingConnection` forwards every `Connection` method.
* Configuration defaults and mappings.

**Integration tests:** the existing `presto-base-jdbc` suite passes against master with no failures, 114 tests with Phase 1 applied on its own and 121 with both phases. A product test against PostgreSQL with each mode is proposed.

**Proof of concept.** PostgreSQL 16 in Docker behind a 24 ms round-trip proxy, through the `postgresql` connector, on a two-node cluster (one coordinator, one worker). Connections were counted at the database with `log_connections`.

| Query | No change | Reuse | Pool | Reuse + pool |
| --- | --- | --- | --- | --- |
| `information_schema.columns`, 21 tables + 21 views | 18.0 s, 88 connections | 3.6 s, 1 | 3.3 s, 0 | 3.6 s, 0 |
| `count(*)` on one table | 1.52 s, 3 | 1.34 s, 2 | 1.18 s, 0 | 1.18 s, 0 |
| Three-table join | 2.35 s, 9 | 1.43 s, 4 | 1.30 s, 0 | 1.10 s, 0 |
| 8 of the first query at once | 19.7 s, 116 | 4.6 s, 8 | 4.5 s, 0-3 | 4.9 s, 0-3 |

Latencies are the median of 5 runs, or of 3 for the concurrent row. The pool columns open few or no connections because the pool already holds them from earlier runs. Reuse and the pool each remove most of the cost on their own; combining them helps most on the join, which touches three tables and so needs more than one connection at a time. On the metadata rows the pool alone is as quick as reuse, because the validation round trip it used to pay on every borrow is skipped inside the bypass window.

Validation cost depends on the driver, which is why that window exists. `Connection.isValid` takes 1-3 ms on PostgreSQL but 260-300 ms on a Teradata instance reached over a cloud link, and a query pays it once per connection it borrows. Five `SHOW COLUMNS` statements against Teradata took 60 s and 67 s with every borrow checked, against 37 s and 46 s with the default 500 ms window.

**Failure modes.** Each of these was reproduced against the running cluster, with the pool on, before deciding whether it needed a fix.

* *The database runs out of connection slots.* With PostgreSQL limited to 15 connections and splits opening their own without limit, 12 of 16 concurrent queries failed with `FATAL: sorry, too many clients already`, which also locks out every other client of that database. `max-overflow` was added for this; 16 and then 24 concurrent queries all succeeded afterwards, with the client backends capped where expected.
* *A link that drops without closing* (toxiproxy black-hole, then restored). Queries fail while the link is down, bounded by the driver's `socketTimeout` rather than hanging. Recovery needs validation on borrow: with it off, two bursts of eight queries failed completely and recovery stayed erratic for five more bursts, because a dead connection that fails a query and is returned stays in the pool. This is why validation is no longer optional.
* *The database closes idle sessions.* Tested both orderings, with PostgreSQL `idle_session_timeout` above and below the pool's `max-idle-time`. No query failed either way; when the server wins the race its `FIN` makes the check fail immediately.
* *A connection retired by age.* With `max-lifetime` at one minute and a query every few seconds, the backend pids turn over on schedule and no query fails across the change.
* *A statement the server cancels* (`statement_timeout`). The error reaches the user unchanged, the connection returns to the pool usable, and repeated cancellations leave no backends behind.
* *The database restarts under a live pool.* While it is down, queries fail in under a second with the driver's own message. The first query after it accepts connections again succeeds, with no lingering failures.
* *PgBouncer in transaction pooling mode.* With `max_prepared_statements=0`, a plain JDBC client that reuses one `PreparedStatement` fails with `prepared statement "S_1" already exists`, as reported by PgBouncer users for years. It did not reproduce through Presto, pooled or not, because the connectors inline pushed-down predicates into the SQL and build a fresh statement per query. Current PgBouncer also enables prepared statement support by default. This is a driver and proxy interaction rather than something the pool introduces, but it is worth knowing before recommending a proxy as the alternative.

**Other backends.** MySQL 8.4 and Teradata were exercised beyond metadata, with reuse and the pool on: scans with splits, filtered scans, joins of two tables of the same catalog, aggregates, `CREATE TABLE AS`, `INSERT ... SELECT`, reading a table while writing to it in one query, and twelve of those at once. Every result was correct and no query failed. On MySQL the pool holds what it should: nothing was pooled before the connector passed its config, and afterwards a node held up to `max-size` and the two nodes together held exactly twice that.

Teradata reads and `CREATE TABLE AS` behave the same way, but `INSERT ... SELECT` reports the rows it wrote and leaves the table unchanged. That happens with reuse and the pool both off as well, so it is a connector bug rather than one of these features, and it is being followed up separately.

**Not yet tested:** a cluster of more than two nodes, and a backend reached over a link slower than the 24 ms used here.
