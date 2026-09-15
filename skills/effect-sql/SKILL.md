---
name: effect-sql
description: Implement or review SQL-backed Effect code using Effect SQL clients, SqlClient, SqlSchema, transactions, and migrations. Use for any supported SQL dialect; load database-specific conventions separately when a task needs them.
---

# Effect SQL conventions

Use Effect SQL with a dialect client and handwritten SQL and migrations. Keep SQL beside the behavior that owns it.

The Effect SQL APIs are shared across drivers; preserve the repository's selected client and its dialect. PostgreSQL, MySQL, SQLite, D1, LibSQL, and SQL Server have different DDL, locking, time, conflict, and query-planning semantics. Do not treat SQL that happens to work for one as portable to another.

## Query safety

- Interpolate values through the SQL tag.
- Never concatenate untrusted values into SQL.
- Represent dynamic identifiers, sort directions, or clauses with explicit whitelisted query branches.
- Define result-returning queries with `effect/unstable/sql/SqlSchema`. Select `findAll`,
  `findNonEmpty`, `findOne`, or `findOneOption` according to the query's cardinality, and use
  `SqlSchema.void` when a schema-encoded request executes a statement with no result.
- Give each query an Effect Schema for its request and one schema for each returned row.
- Keep each `SqlSchema` definition beside the behavior and handwritten SQL it owns. Do not create a
  generic typed-database service or repository around `SqlClient`; `SqlSchema` is the reusable
  schema adapter.
- Keep stored table and column identifiers in `snake_case` when the repository has not established
  another SQL convention.
- When the selected client supports result-name transformation, configure it once at the client
  layer. Returned row schemas and TypeScript values should use the repository's normal TypeScript
  casing; do not repeat casing aliases in each query.
- Define one Effect Schema against the transformed result shape and return the `SqlSchema` result
  directly. Do not execute into `rows` and manually call `Schema.decodeUnknownEffect` afterward.
- Add a separate row type or mapper only for a real semantic transformation, never solely to rename
  stored fields.
- Brand the projected schema when the decoded value represents authority such as a lease fence.
- Keep provider and network calls outside authoritative database transactions.
- Use database time for leases, retry eligibility, and fencing comparisons when the selected
  database can provide a consistent server-side clock.
- Encode concurrency invariants in constraints, transactions, locks, and fenced updates.

## Behavior and query modules

When a behavior service owns several SQL actions, split it into one service module and one sibling
`*-queries.ts` module:

- Keep request and row schemas, `SqlSchema` definitions, and handwritten SQL private in the query
  module.
- Export one `makeFeatureQueries` Effect factory. It obtains `SqlClient`, defines the focused query
  operations, and returns them as one inferred object.
- Yield that factory inside the owning behavior Layer. Do not wrap an internal query collection in a
  `Context.Service`, generic repository, or generic database abstraction.
- Let one query represent one focused persistence action or one database-atomic state transition.
  Do not split SQL that must remain atomic merely to make each statement shorter.
- Keep decoding, authorization, domain rejection, policy, orchestration, and mapping infrastructure
  failures in the behavior service.
- Keep transaction boundaries in the behavior service so the complete domain workflow visibly owns
  its atomicity. Query modules must not silently start transactions.
- Compose reusable queries in service operations instead of exporting service implementation
  functions for HTTP handlers or tests to assemble themselves.
- Export a row schema or query input type only when another behavior genuinely shares that database
  boundary. Prefer inference from the returned query object otherwise.

## Transactions and locking

- Audit every competing writer before choosing a lock order. When the selected database supports
  row locking, use one canonical order from the aggregate or root row to its child rows across every
  code path.
- Acquire the strongest required lock on first touch. Do not acquire a shared lock and later upgrade
  the same row to an exclusive lock.
- Before relying on locks across nested `withTransaction` scopes, verify that the pinned Effect SQL
  implementation reuses the same connection and implements nesting with the database's supported
  mechanism, such as savepoints.
- Prove consequential ordering with a deterministic integration test on the selected database:
  hold the root lock, start competing writers, verify they have not acquired child locks, release
  the root lock, and require every writer to complete within a timeout.

Before finishing a transaction change, confirm that every competing writer follows the same
root-to-child order, no path upgrades a shared lock, nested scopes retain connection ownership, and
the contention test passes without deadlock.

## Formatting

Apply the repository's SQL formatter when it has one. Otherwise apply these rules to SQL inside
tagged template literals and migrations:

- Write SQL keywords and database identifiers in lowercase.
- Use `snake_case` identifiers.
- Indent with spaces, never tabs.
- Indent blocks by two spaces.
- Keep each table column definition on one line when it fits.
- Within each `create table` block, align the column name, data type, nullability, default, and
  trailing comma into consistent visual columns. Calculate alignment from the longest entry in that
  block; do not leave one row shifted because its identifier is longer.
- Write `null` explicitly for nullable table columns so nullability is scannable beside `not null`.
- Show allowed state values vertically inside `check (... in (...))` constraints instead of hiding a
  state machine in one long line.
- Put selected and returned columns on vertical lines.
- Make `from`, joins, `where`, `and`, `order by`, and `returning` visually obvious.
- Align related columns and comparison operators when it improves scanning.
- Keep trailing commas consistent within a query.
- Omit the final semicolon in tagged template queries; include semicolons in migration files.

Before finishing a SQL edit, scan every modified DDL and query block as a rectangular shape.
Formatting is complete only when related tokens occupy the same visual columns throughout each
block.

## Scoped data isolation

Apply these rules to any data model with an ownership or tenant scope:

- Authenticate and authorize before executing scoped behavior.
- Derive the scope identifier from authenticated authority, never from an unchecked request field.
- Include the scope identifier in primary keys, unique constraints, foreign keys, joins, lookups,
  and mutations where the data model requires scope isolation.
- Keep cross-scope worker or operator operations explicit and separate from scoped operations.
- Do not depend on connection roles, session variables, or privileged database functions for
  ordinary application-level scope isolation unless the repository deliberately establishes and
  verifies that model.
- Prove that one scope cannot read, mutate, or reference another scope's rows with integration
  tests against the selected database.

## Migrations

- Make the schema readable from an empty database.
- Give every foreign key, unique rule, check, and index a domain reason.
- Include scope in keys and constraints where the data model requires it.
- Prefer constraints over application-only validation for durable invariants.
- Add indexes from actual access paths and observed query behavior.
- Use the selected database's documented migration, transaction, locking, and online-DDL behavior;
  dialect-specific syntax and rollout constraints do not become portable merely because the Effect
  SQL client API is shared.
