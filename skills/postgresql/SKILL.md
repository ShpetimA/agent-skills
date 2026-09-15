---
name: postgresql
description: Design, review, diagnose, or change PostgreSQL schemas, queries, transactions, indexes, and production database behavior. Use for PostgreSQL-specific concurrency, performance, MVCC, vacuum, or query-planning decisions; not for generic SQL alone.
---

# PostgreSQL Engineering

Use PostgreSQL's actual execution and concurrency model when making database decisions. Inspect the repository's database client, migration system, transaction helper, connection pool, query builder, and operational conventions before choosing an implementation.

This skill is client- and framework-neutral. Do not introduce Effect, an ORM, a connection pooler, or a database library merely because this skill is active. In an Effect codebase, preserve its established resource and error-handling conventions while applying the PostgreSQL guidance below.

## Change safely

- Identify the data invariant, the concurrent operations that can violate it, and the query paths that must remain fast before changing schema or SQL.
- Keep transactions short. Do not await remote calls, user input, queue work, or unbounded computation while holding a transaction or row lock.
- Parameterize values. Treat dynamic identifiers and SQL fragments as a separate, tightly controlled concern; bind parameters do not make arbitrary identifiers safe.
- State the transaction and retry contract where a change can conflict, deadlock, serialize, or be delivered more than once.
- Use the project's migration process. A production schema change needs its lock and rewrite behavior understood; do not assume every DDL statement is online.

## MVCC, transactions, and locks

PostgreSQL updates create new row versions. Old versions stay until no active snapshot can need them, then VACUUM can reclaim their space. This gives readers a consistent snapshot without blocking writers, but makes update-heavy tables, long transactions, and delayed vacuum operational concerns.

- At the default `READ COMMITTED` level, each statement sees a new snapshot. A read-compute-write sequence can therefore use stale application state even though the eventual `UPDATE` locks the row.
- Prefer an atomic SQL update when the invariant permits it, such as `SET balance = balance - $1` with a predicate that enforces the lower bound.
- When correctness requires reading a row and then deciding how to modify it, lock it deliberately with `SELECT ... FOR UPDATE` in a short transaction.
- Use `SKIP LOCKED` for work distribution only: a skipped row means another worker can legitimately own it. Do not use it for work that must happen now, such as a balance transfer or an authorization decision.
- For multi-row operations, acquire locks in a globally consistent order, usually by primary key. This prevents opposing transactions from forming a deadlock cycle.
- Treat `40P01` (deadlock) and serialization failures as retryable only when the whole operation is retry-safe. Retry from a fresh transaction and recompute from fresh reads; do not rerun only the statement that failed.
- Use `SERIALIZABLE` when a cross-row invariant cannot be expressed as a row lock or atomic write. It can abort a valid attempt, so the caller needs a bounded full-transaction retry strategy.

## Indexes and plans

An index is a read-path tradeoff, not a generic optimization. It consumes storage, cache, write work, and WAL for every affected write.

- Start from a real query shape: filters, joins, ordering, limit, selected columns, and expected selectivity.
- A sequential scan can be correct when a query returns much of a table. Do not add an index solely because a plan says `Seq Scan`.
- Order composite B-tree indexes around the query's leading equality filters, then its range or ordering needs. An index on `(customer_id, created_at)` does not provide a direct date-only access path.
- Do not infer index usefulness from the definition alone. Inspect `EXPLAIN (ANALYZE, BUFFERS)` for representative, safe-to-run reads and compare estimates with actual rows and buffer activity.
- `EXPLAIN ANALYZE` executes its statement. Never use it unguarded on a production mutation; use an explicit rollback in a controlled session or another safe diagnostic approach.
- Index-only scans still depend on MVCC visibility. Including every selected column to chase an index-only plan can make writes and cache behavior worse; measure before adding `INCLUDE` columns.

## Vacuum, bloat, and long transactions

- Plain `VACUUM` makes dead-tuple space reusable and helps maintain visibility information; it normally does not shrink the operating-system file. Reclaiming file size requires a rewrite-oriented approach with different locking and operational consequences.
- Autovacuum thresholds scale with table size. A high-churn large table may need table-specific tuning, based on observed dead tuples, write rate, query behavior, and maintenance capacity—not a blanket global change.
- Long-running and idle-in-transaction sessions hold old snapshots, delay cleanup, and can endanger transaction-ID freeze work. Monitor and eliminate them before treating bloat as an index problem.
- Diagnose performance regressions with evidence across query plans, table/index size, dead tuples, maintenance activity, lock waits, active transactions, connection pressure, and database I/O. A slow query is not automatically an indexing problem.

## Verification

Match verification to the risk:

- Test data invariants under concurrent access, especially read-modify-write paths, workers, transfers, reservations, and status transitions.
- Use a real PostgreSQL instance for SQL semantics, migration behavior, lock interactions, and plan-sensitive changes. Mocks cannot prove PostgreSQL's locking, planner, or transaction behavior.
- Test retries as complete attempts, including idempotency of any external effect that follows a successful commit.
- Record the observed query shape or relevant operational evidence when adding an index or changing a performance-sensitive query. Do not claim a speedup from theory alone.

## Common mistakes to reject

- "The row lock prevents lost updates." It does not repair a stale value computed before the lock; use atomic SQL or lock before reading.
- "More indexes make reads faster." Extra indexes tax writes and can displace useful data from cache.
- "VACUUM shrinks disk usage." Ordinary vacuum reuses space inside the relation; it does not normally return it to the OS.
- "`SKIP LOCKED` makes contention disappear." It changes work ownership semantics by allowing a caller to receive only currently free rows.
- "A retry is just rerunning the failed query." A serialization or deadlock retry needs a new transaction and fresh decision inputs.
