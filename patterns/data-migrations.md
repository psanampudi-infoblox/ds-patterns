# Migration Decision Guide

This guide helps you decide **how** to ship a database change safely:
when to use a **down** migration, a **forward‑fix**, or a **staged
rollout**, and what patterns/checklists to apply.

------------------------------------------------------------------------

## Core Concepts (clear definitions)

-   **Up migration (↑)**\
    The change that moves the schema/data from version *N → N+1*.
    Example: adding a column, creating an index, backfilling a value.

-   **Down migration (↓)**\
    The exact inverse of the up migration that restores *N+1 → N*. It
    must be *logically valid* (restores semantics) and *operationally
    safe* (won't corrupt data), or else it should be **disabled** and
    replaced by a documented **reversal plan** (see forward‑fix).

-   **Forward‑fix**\
    When reverting is risky or lossy, ship a *new* up migration (N+2)
    that fixes/repairs the mistake introduced in N+1. Think of this as
    "rollback by rolling forward."

-   **Staged rollout** (a.k.a. **expand‑migrate‑contract**)\
    A zero‑/near‑zero‑downtime strategy where you:

    1)  **Expand**: add new structures in a backward‑compatible way,\
    2)  **Migrate**: dual‑write and backfill to the new structures,\
    3)  **Contract**: switch reads to the new structures and later
        remove old ones.

-   **Reversible vs. Irreversible migration**

    -   *Reversible*: changes that can be undone without permanent data
        loss (e.g., adding an index).
    -   *Irreversible*: destructive or lossy transforms (e.g., dropping
        a column with data or merging columns).

-   **Operational blast radius** The potential impact on
    availability/latency/throughput if the migration goes wrong (table
    locks, long scans, cache churn, etc.).

------------------------------------------------------------------------

## Quick Decision Matrix

  -------------------------------------------------------------------------
  Situation               Default Strategy          Why
  ----------------------- ------------------------- -----------------------
  Pure additive schema    **Staged rollout          Safe, backward
  (nullable column, new   (Expand)** + standard     compatible; down is
  table, new index        **Down**                  trivial
  concurrently)                                     

  Large table ALTER that  **Staged rollout** +      Avoids downtime/locks
  can lock/scan           online/index‑concurrent   
                          tools                     

  Destructive change      **Staged rollout** →      Lossy rollback: don't
  (drop column/table with **Forward‑fix** (no true  pretend a real down
  data)                   Down)                     exists

  Data transformation     **Staged rollout** +      Use derived/dual
  (split/merge columns)   backfill +                fields; avoid lossy
                          **Forward‑fix**           down

  Hot path constraint     **Staged rollout** +      Prevents write failures
  change (NOT NULL, FK    gated enforcement         mid‑deploy
  add)                                              

  Multi‑tenant (shared    **Staged rollout** with   Global changes need
  schema)                 feature flags per tenant  tenant‑safe toggles

  Schema‑per‑tenant       **Looped migrations per   Parallelize with
                          schema** with concurrency limits; track
                          per‑tenant successes
  -------------------------------------------------------------------------

------------------------------------------------------------------------

## Flow (choose your path)

1)  **Classify** the change
    -   Additive? Destructive? Transformative? Long‑running?
        Tenant‑scoped?
2)  **Select strategy**
    -   If additive → staged rollout (Expand) + keep Down.\
    -   If destructive or lossy → **no down**; use forward‑fix plan.\
    -   If long‑running/locking risk → online/async approach (concurrent
        index, shadow copy + swap).
3)  **Design the rollout**
    -   Expand → Dual‑write/Backfill → Read‑switch → Contract.\
    -   Gate switches with feature flags and/or per‑tenant toggles.
4)  **Safety checks**
    -   Can the whole migration be wrapped in a transaction? (Postgres:
        many DDLs yes; MySQL: limited.)\
    -   Do you need batches/chunking?\
    -   Is there a live‑traffic read path that depends on the changed
        object? Add compatibility shims.
5)  **Decide rollback posture**
    -   If truly reversible, supply a working Down.\
    -   If not, explicitly **disable down** and document Forward‑fix
        runbook.

------------------------------------------------------------------------

## Patterns (with concrete SQL-ish examples)

### 1) Expand → Migrate → Contract (EMC)

**Goal:** Add `users.timezone` required by new features; avoid downtime.

1)  **Expand** (backward‑compatible)

``` sql
ALTER TABLE users ADD COLUMN timezone TEXT NULL;
-- Postgres: create index concurrently
CREATE INDEX CONCURRENTLY idx_users_timezone ON users(timezone);
```

2)  **Migrate**

-   App release A: **dual‑write** `timezone` (if client supplies it);
    keep reading old behavior if missing.
-   **Backfill** in batches:

``` sql
-- Pseudocode
UPDATE users
SET timezone = 'UTC'
WHERE timezone IS NULL
LIMIT 10_000;
-- Loop with cursor until done
```

3)  **Contract**

-   App release B: read `timezone` everywhere, enforce NOT NULL
    gradually:

``` sql
ALTER TABLE users ALTER COLUMN timezone SET NOT NULL; -- only after backfill done
```

-   Later: remove old fallback logic.

**Down?**\
- Additive portion is reversible (`DROP INDEX`,
`ALTER TABLE ... DROP COLUMN`), but after NOT NULL + prod usage, prefer
**forward‑fix** if issues arise.

------------------------------------------------------------------------

### 2) Rename without breaking callers

**Direct rename is risky**; prefer aliasing.

**Expand** - Add new column, keep old:

``` sql
ALTER TABLE orders ADD COLUMN placed_at TIMESTAMPTZ NULL;
```

-   Dual‑write: whenever app writes `created_at`, also write
    `placed_at`.

**Migrate** - Backfill `placed_at = created_at` in batches.

**Read switch** - App reads `placed_at` primarily; falls back to
`created_at` if NULL.

**Contract** - Remove reads of `created_at`; later drop it.

**Down**: don't rely on renaming back; treat as forward‑fix.

------------------------------------------------------------------------

### 3) Constraint hardening (NOT NULL / FK)

**Two‑phase enforcement** 1) **Soft‑enforce** at app layer; backfill
existing rows.\
2) **Add NOT NULL / FK** only after metrics show zero violations:

``` sql
ALTER TABLE invoices ALTER COLUMN customer_id SET NOT NULL;
ALTER TABLE invoices ADD CONSTRAINT invoices_customer_fk
  FOREIGN KEY (customer_id) REFERENCES customers(id) NOT VALID;
-- Postgres: validate without long locks
ALTER TABLE invoices VALIDATE CONSTRAINT invoices_customer_fk;
```

**Down**: Can drop constraints, but if app already assumes them,
forward‑fix instead.

------------------------------------------------------------------------

### 4) Large index builds

-   **Postgres**: `CREATE INDEX CONCURRENTLY` (no long exclusive
    locks).\
-   **MySQL/InnoDB**: Use online DDL (`ALGORITHM=INPLACE` where
    possible) or a tool like `pt-online-schema-change`.\
-   **ClickHouse**: Prefer materialized views or projections; heavy ops
    often require copy+swap patterns.

**Down**: `DROP INDEX` is trivial; risk is operational, not logical.

------------------------------------------------------------------------

### 5) Destructive Drop

-   Replace with **soft delete** first (or shadow copy to a `*_bak`
    table).
-   If you must drop:
    -   Take a snapshot/backup and **document forward‑fix-only**.
    -   Optionally export column data to object storage for emergency
        restore.

**Down**: **DISABLED** (documented as not supported due to data loss).

------------------------------------------------------------------------

## Tenancy Considerations

-   **Shared schema (tenant_id column)**
    -   Global migration affects everyone; must be **backward
        compatible** and **flagged** by tenant when enabling new
        behavior.
    -   For backfills, throttle by tenant to avoid noisy neighbor
        effects.
-   **Schema-per-tenant**
    -   Iterate per schema; track successes in a control table
        (`schema_migrations(tenant_schema, version, applied_at)`).
    -   Concurrency cap (e.g., 10 schemas at a time), with
        retry/backoff; produce a final per‑tenant report.

------------------------------------------------------------------------

## Checklists (pre‑flight & runbook)

### Pre‑flight (design time)

-   [ ] Categorize: additive / destructive / transform / long‑running.\
-   [ ] Choose **EMC staged rollout** by default; choose **forward‑fix**
    if irreversible.\
-   [ ] Validate transactional behavior (DB engine specifics).\
-   [ ] Estimate runtime & locks (size, indexes, hot paths).\
-   [ ] Plan backfill strategy: batch size, pause/resume, idempotency.\
-   [ ] Define toggles: feature flag, per‑tenant enablement, read‑switch
    guard.\
-   [ ] Write observability plan: metrics, query latency, lock wait,
    error rates.\
-   [ ] Declare rollback posture: *real down* vs *forward‑fix*, and link
    the fix plan.

### Deployment runbook

1.  **Expand** migration (DDL online where possible).\
2.  Deploy **dual‑write** app version.\
3.  Start **backfill job** (idempotent, chunked, with progress
    checkpointing).\
4.  Monitor: backlog, errors, CPU/IO, replication lag, lock waits.\
5.  **Read switch** behind a flag; canary by tenant/percentage.\
6.  Bake time; monitor.\
7.  **Contract** (drop old paths) during a low‑risk window.\
8.  Archive/cleanup artifacts.

### Backfill best practices

-   Use **key‑range or time‑range** chunking.\
-   **Small transactions** (e.g., 1--10k rows) to reduce lock times &
    WAL/redo pressure.\
-   **Sleep/yield** between batches to keep p99 healthy.\
-   **Idempotency**: re‑run safe; store last processed key.\
-   **Deadline awareness**: allow abort and resume.

------------------------------------------------------------------------

## When to supply a real **Down** vs. disable it

-   **Provide a Down** when:
    -   The change is purely additive or easily reversible (indexes,
        non‑NOT‑NULL columns, new tables).\
    -   You're pre‑cutover in EMC and haven't deleted or mutated
        original data.
-   **Disable Down** (document forward‑fix) when:
    -   Column/table drops with data, lossy transformations, or
        externalized side effects.\
    -   The application already switched reads and user workflows
        produced new data only compatible with the new schema.

**Documentation tip:** In the migration file header, add:

    -- REVERSIBILITY: forward-fix only
    -- FORWARD-FIX PLAN: <link to runbook/PRD/Ticket>

------------------------------------------------------------------------

## Observability & Guardrails

-   **Before**: establish baseline R/W QPS, p95/p99 latencies, lock wait
    metrics, replication lag.\
-   **During**: alert on lock waits, statement timeouts, error spikes,
    lag thresholds.\
-   **After**: verify row counts, FK consistency, uniqueness, and app
    KPIs.

**Kill switch:** A feature flag or job toggle that can instantly pause
backfills or revert read‑switch without schema changes.

------------------------------------------------------------------------

## Example "irreversible" note with forward‑fix plan

> **Migration N+1:** Drop `users.full_name`; keep
> `first_name`/`last_name`.\
> **Reversibility:** Irreversible (data loss).\
> **Forward‑fix plan:** If breakage occurs, ship N+2 to reintroduce
> `full_name` **computed** as `first_name || ' ' || last_name` and
> update callers; no attempt to restore original formatting.

------------------------------------------------------------------------

## Tooling Guidance

-   Prefer frameworks with **versioned history** and **locking**
    (Flyway, Liquibase, Rails, Alembic, Ecto).\
-   For PostgreSQL, use **`CONCURRENTLY`**, **`NOT VALID` →
    `VALIDATE CONSTRAINT`**, **`statement_timeout`** and
    **`lock_timeout`** guards.\
-   For MySQL, specify **`ALGORITHM=INPLACE, LOCK=NONE`** where
    available or use **pt‑online‑schema-change**.\
-   Bake migration code into the same CI/CD with **dry‑run in staging**
    using prod‑sized data samples.

------------------------------------------------------------------------

## TL;DR rule of thumb

1)  **Default to Staged (EMC)**.\
2)  **Only promise Down** when it's truly safe.\
3)  **Assume forward‑fix** for anything destructive or
    data‑transforming.\
4)  **Backfills are jobs**, not ad‑hoc updates---make them idempotent,
    chunked, observable.\
5)  **Flags + telemetry** turn risky migrations into routine operations.

## Glossary

- **Up migration (↑)**: Change that moves schema/data from version N to N+1.
- **Down migration (↓)**: Inverse of the up migration to restore N+1 to N; may be disabled when unsafe.
- **Forward‑fix**: Roll forward with a new migration (N+2) to repair issues when rollback is risky or lossy.
- **Staged rollout (Expand → Migrate → Contract, EMC)**: Backward‑compatible add, dual‑write/backfill, then switch reads and remove old paths.
- **Expand**: Introduce new structures or columns in a backward‑compatible way.
- **Migrate (phase)**: Dual‑write and backfill data into new structures.
- **Contract**: Switch reads to new structures and later remove deprecated ones.
- **Dual‑write**: Temporarily write to both old and new schema paths to enable safe cutovers.
- **Backfill**: Batch process to populate new fields/tables from existing data.
- **Read switch**: Application change that prioritizes reading from the new structure (with fallback during transition).
- **Reversible migration**: Change that can be safely undone without permanent data loss.
- **Irreversible migration**: Destructive or lossy change (e.g., dropping columns with data) that cannot be fully undone.
- **Operational blast radius**: Expected impact on availability/latency/throughput if the migration misbehaves.
- **Online DDL**: Techniques/tools to alter schema with minimal blocking (e.g., Postgres CREATE INDEX CONCURRENTLY; MySQL ALGORITHM=INPLACE, LOCK=NONE).
- **Shadow copy + swap**: Create a copy (or new structure), populate it, then atomically switch over.
- **Soft delete**: Mark data as deleted (flag/tombstone) instead of immediate physical drop.
- **Constraint hardening**: Two‑phase approach to enforce constraints (app‑level first, then DB NOT NULL/FK once clean).
- **NOT VALID → VALIDATE CONSTRAINT (Postgres)**: Add FK without immediate full validation, then validate later with reduced locking.
- **Chunking/batching**: Processing data in small ranges to control lock time, resource use, and tail latency.
- **Idempotency**: Ability to safely retry operations without changing the final result (crucial for backfills/jobs).
- **Kill switch**: Feature flag or job toggle to pause/abort backfills or revert read switches quickly.
- **Feature flag**: Controlled toggle to gate behaviors (global or per‑tenant) during rollout.
- **Shared schema**: Multi‑tenant pattern using a tenant_id column; requires tenant‑safe rollouts.
- **Schema‑per‑tenant**: Each tenant has its own schema; migrate per schema with concurrency caps and progress tracking.
- **Control table (schema_migrations)**: Metadata table tracking applied versions (optionally per tenant/schema).
- **Concurrency cap**: Limit on simultaneous migrations/backfills to protect system health.
- **Replication lag**: Delay between primary and replica; monitor during heavy writes/backfills.
- **Lock wait**: Time spent waiting for locks; alert and tune timeouts to avoid stalls.
- **statement_timeout / lock_timeout (Postgres)**: Guards to abort long/blocked statements safely.
- **Forward‑fix only (disabled Down)**: Explicit posture noting rollback is not supported; link the fix plan.
- **Backward‑compatible**: Changes that old code can run against without errors (key for Expand phase).
- **Observability plan**: Metrics/traces/logs required to monitor migration safety (p95/p99, errors, locks, lag).
