# Migration Decision Guide

This guide helps you decide how to ship a database change safely: when to use a Down migration, a Forward‑Fix, or a Staged Rollout (EMC), and what patterns / checklists / guardrails to apply.

---

## TL;DR (Quick Rules of Thumb)

1. Default to Staged (Expand → Migrate → Contract).
2. Only promise Down when it's truly safe (purely additive or reversible).
3. Assume Forward‑Fix for destructive or data-transforming changes.
4. Treat backfills as jobs: idempotent, chunked, observable, abortable.
5. Flags + telemetry turn risky migrations into routine operations.
6. Classify risk early (R1/R2/R3) to decide required controls.
7. Never destroy data in the same release you introduce a replacement path.

---

## (Reordered) Table of Contents

- TL;DR (above)
- Core Concepts
- Quick Decision Matrix
- Migration Flow
- Patterns (EMC, Rename, Constraint Hardening, Large Index, Destructive Drop, etc.)
- Tenancy Considerations
- Risk Classification (NEW)
- Verification & Data Integrity (NEW)
- Backfill Job Template (NEW)
- Multi-Region & Replication (NEW)
- Partitioning & Sharding (NEW)
- ENUM / Type Evolution (NEW)
- Feature Flag Best Practices (NEW)
- Forward-Fix Header Template (NEW)
- Automation & Guardrails (Timeouts, Control Tables) (NEW)
- Testing & Pre-Prod Strategy (NEW)
- Security & Compliance Notes (NEW)
- Anti-Patterns (NEW)
- Observability Deep Dive (Expanded)
- Rollback Playbooks (Clarified)
- Tooling Ecosystem Hooks (NEW)
- Decision Cheat Cards (NEW)
- Checklists (Pre-flight & Runbook)
- When to Supply a Down vs Forward-Fix
- Glossary

---

## Core Concepts (clear definitions)

-   **Up migration (↑)**<a id="gloss-up-migration"></a>\
    The change that moves the schema/data from version *N → N+1*.
    Example: adding a column, creating an index, backfilling a value.

-   **Down migration (↓)**<a id="gloss-down-migration"></a>\
    The exact inverse of the up migration that restores *N+1 → N*. It
    must be *logically valid* (restores semantics) and *operationally
    safe* (won't corrupt data), or else it should be **disabled** and
    replaced by a documented **reversal plan** (see forward‑fix).

-   **Forward‑fix**<a id="gloss-forward-fix"></a>\
    When reverting is risky or lossy, ship a *new* up migration (N+2)
    that fixes/repairs the mistake introduced in N+1. Think of this as
    "rollback by rolling forward."

-   **Staged rollout** (a.k.a. **expand‑migrate‑contract**)<a id="gloss-staged-rollout"></a>\
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

-   **Operational blast radius**<a id="gloss-operational-blast-radius"></a> The potential impact on
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

## Migration Flow (choose your path)

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

---

## Risk Classification (NEW)

Define a lightweight risk tier early. This dictates required controls, review depth, and rollout rigor.

| Tier | Typical Characteristics | Examples | Mandatory Controls | Optional (Recommend when ↑ tier) |
|------|-------------------------|----------|--------------------|----------------------------------|
| R1 (Low) | Purely additive, small table (<1M rows), reversible, no user-visible semantics change | Add nullable column, create non-unique index | Staged Expand, Working Down, Basic metrics (latency + errors) | Canary flag (if user-facing) |
| R2 (Medium) | Larger surface (10M–500M rows) OR mild performance risk OR partial irreversibility | Add NOT NULL after backfill, dual-write rename, large concurrent index | EMC pattern, Backfill job (idempotent), Kill switch, Progress metrics, Query plan capture pre/post | Shadow reads; Row count & checksum sampling |
| R3 (High/Critical) | Destructive / lossy OR hot path large table OR multi-region replication sensitivity OR irreversible transform | Column drop with data, column split/merge, table repartition, type refactor, cross-shard copy | Formal Forward-Fix plan, Dry-run on prod-sized clone, Explicit Down disabled header, SRE/on-call staffed, Continuous lag monitoring, Checksum coverage (>95% rows or full), Performance guard thresholds, Pre-approved rollback playbook | Traffic replay simulation; Shadow copy + blue/green read switch |

Risk tier selection inputs:
- Data criticality (user-facing, billing, compliance, analytics-only)
- Table size & write QPS
- Reversibility posture (real Down vs forward-fix-only)
- Operational hazard (locks, vacuum amplification, WAL/redo volume)
- Multi-region / replication topology impact

Escalation triggers (auto-bump to higher tier):
- Statement estimated cost or EXPLAIN row estimate > threshold (configurable)
- Requires more than X hours projected backfill time
- Affects > Y% of read queries (based on query stats sampling)
- Any destructive action in same PR as semantic app change

Acceptance summary per tier:
- R1: Reviewed by owning engineer, CI migration pass
- R2: Peer review + performance sanity + basic verification script
- R3: Architecture review + runbook approval + on-call signoff + pre-run simulation evidence

Document chosen tier in migration header for traceability.

---

## Verification & Data Integrity (NEW)

Goal: Prove the migration is correct, complete, and performant before contracting / deleting legacy paths.

Core dimensions:
- Completeness (all intended rows migrated / populated)
- Consistency (referential + value transformations correct)
- Performance (no regression beyond agreed SLO budgets)
- Safety (no unexpected lock amplification / replication lag)

Recommended verification layers:
1. Row count parity (source vs target or old vs new columns)
2. Sampled checksums (stable deterministic hash over key fields)
3. FK & uniqueness integrity
4. Query plan & latency regression diff
5. Drift detection (no new NULLs after backfill freeze point)

Example row count check (Postgres):
```sql
-- For dual-written column migration
SELECT
    (SELECT COUNT(*) FROM users WHERE new_col IS NOT NULL) AS populated,
    (SELECT COUNT(*) FROM users) AS total,
    (SELECT COUNT(*) FROM users WHERE legacy_col IS NOT NULL AND new_col IS NULL) AS remaining_legacy_only;
```

Deterministic checksum sampling (avoid full table scans on huge tables):
```sql
-- Hash deterministic sample (e.g., 0.1% of keys)
WITH sample AS (
    SELECT id, legacy_col, new_col
    FROM users
    WHERE (id % 1000) = 7 -- sampling modulus strategy
)
SELECT
    COUNT(*) AS sampled,
    SUM(CASE WHEN legacy_col_transform(legacy_col) = new_col THEN 1 ELSE 0 END) AS matches,
    SUM(CASE WHEN legacy_col_transform(legacy_col) <> new_col THEN 1 ELSE 0 END) AS mismatches
FROM sample;
```
Note: Implement `legacy_col_transform()` in the application or replace inline if simple.

Checksum strategy (larger sets):
```sql
-- Rolling checksum window for large tables (by id ranges)
SELECT
    floor(id/100000) AS bucket,
    COUNT(*) AS rows,
    sum(mod(hashtext(COALESCE(new_col,'')), 2147483647)) AS checksum
FROM users
GROUP BY 1
ORDER BY 1;
```
Store snapshots pre- and post-migration; diff for anomalies.

FK orphan detection:
```sql
SELECT child.customer_id
FROM invoices child
LEFT JOIN customers c ON c.id = child.customer_id
WHERE c.id IS NULL
LIMIT 50; -- should be zero rows before enforcing FK
```

Uniqueness sanity (prior to adding unique index):
```sql
SELECT new_key, COUNT(*)
FROM staging_table
GROUP BY new_key
HAVING COUNT(*) > 1
LIMIT 50;
```

Query plan capture (EXPLAIN baseline vs new):
```sql
EXPLAIN (ANALYZE, BUFFERS) SELECT ... FROM users WHERE ...;
```
Persist JSON plans so structural diff can be performed (detect plan shape changes, not just timing noise).

Performance guard thresholds:
- p95 read latency increase < +10% sustained over 15 min
- Lock waits < threshold (e.g., < 100ms p95)
- Replication replay lag < X seconds (e.g., 5s) during backfill

Drift monitor query (post-backfill freeze):
```sql
SELECT COUNT(*) FROM users
WHERE legacy_col IS NOT NULL AND new_col IS NULL
    AND updated_at > :backfill_completed_at;
```
Alert if > 0 (indicates missed dual-write path).

Completion criteria gate:
- All remaining_legacy_only = 0
- Drift monitor stable 2 consecutive intervals
- Checksums mismatch ratio < 0.01% (configurable)
- Plan diff acceptable (no switch to seq scan on hot path inadvertently)

Document verification evidence in migration ticket before Contract phase.

---

## Backfill Job Template (NEW)

Goal: Deterministic, restartable, throttled population of new schema elements without harming foreground latency.

Principles:
- Idempotent writes (upsert / compare-and-set)
- Bounded transaction size
- Observable progress & ETA
- Adaptive throttling (respect latency + replication lag budgets)
- Safe abort & resume

Control table example:
```sql
CREATE TABLE IF NOT EXISTS migration_backfill_progress (
    job_name text PRIMARY KEY,
    last_id bigint,
    started_at timestamptz DEFAULT now(),
    updated_at timestamptz DEFAULT now(),
    status text CHECK (status IN ('running','paused','completed','error')),
    total_processed bigint DEFAULT 0,
    notes text
);
```

Pseudocode (language-agnostic):
```
job_name = "users_timezone_backfill"
batch_size = 5000
target_latency_p95_ms = 120
max_replication_lag_s = 5
sleep_base_ms = 250

state = load_progress(job_name) // last_id, status
if state.status in {completed}: exit

loop:
    if cancellation_flag(): mark_paused(); exit

    rows = SELECT id, timezone FROM users
                 WHERE id > state.last_id AND timezone IS NULL
                 ORDER BY id ASC
                 LIMIT batch_size FOR UPDATE SKIP LOCKED;

    if rows empty:
            mark_completed(); break

    begin transaction
        for r in rows:
                UPDATE users SET timezone = 'UTC' WHERE id = r.id AND timezone IS NULL;
    commit

    new_last = max(rows.id)
    processed = len(rows)
    update_progress(job_name, new_last, processed)

    metrics.emit(progress = percent_done_estimate(), eta = compute_eta())

    if current_p95_latency() > target_latency_p95_ms or replication_lag() > max_replication_lag_s:
            sleep(adaptive_backoff())
    else:
            sleep(sleep_base_ms)
```

Key adaptive mechanisms:
- Use `FOR UPDATE SKIP LOCKED` (Postgres) to allow parallel shard workers.
- Maintain moving average throughput to compute ETA: `eta = remaining_rows / rows_per_second`.
- Pause criteria: sustained p95 > threshold for N intervals.

Idempotency test: run last two batches again; zero net changes.

Resume semantics: if job restarts, it reads `last_id` and continues; no race creates duplicate semantics because predicate always filters rows needing work.

---

## Multi-Region & Replication (NEW)

Concerns: replication lag growth, schema divergence timing, global read routing consistency.

Key patterns:
- Stagger DDL: ensure primary applies first; wait until replicas confirm schema (`pg_stat_replication` no longer pending). Avoid rapid successive DDL bursts.
- Large backfills: throttle on replication replay lag, *not* just primary latency.
- Read-switch gating: only route reads that depend on new structures to replicas after validating they possess the structure (feature flag conditioned on replica schema version sync metric).
- Logical replication: maintain ordering of dependent DDL + DML; avoid mixing transactional semantics assumptions if using out-of-order apply in downstream systems.
- Multi-primary / active-active: prefer additive superset phase across regions before any contract; enforce deterministic conflict resolution (timestamp or origin-region precedence) on dual-write fields.

Lag guard example (Postgres):
```sql
SELECT application_name, replay_lag, write_lag, flush_lag
FROM pg_stat_replication;
```
Abort / slow backfill if any `replay_lag > 5s` (tier-dependent threshold).

Shadow copy for high-risk transforms:
- Build new table in region A
- Stream or replicate to region B
- Validate checksums cross-region before flipping global router

Cutover ordering (multi-region read fanout):
1. Expand globally (ensures old code safe)
2. Enable dual-write region by region (canary region first)
3. Complete backfill & verify in canary
4. Roll out dual-write globally
5. After global verification, perform read-switch per region
6. Contract only after *all* regions stable & consistent window elapsed

Replica safety for NOT NULL / constraints:
- Add constraint as `NOT VALID` on primary
- Wait replication
- `VALIDATE CONSTRAINT` during low traffic window (lower lock risk per replica)

Disaster recovery (DR) note:
- Avoid contract steps during an ongoing DR drill or known replica lag incident
- Snapshot DR environment *before* destructive steps (R3 requirement)

---

## Partitioning & Sharding (NEW)

Goals: minimize global impact, leverage natural data boundaries, and ensure balanced progress.

Partitioned tables (single logical DB):
- Prefer altering parent table (inherits to partitions) where engine supports; verify constraint/index propagation semantics.
- Backfill partition-by-partition: choose order by smallest → largest to validate logic early, or by cold → hot to reduce risk first.
- Avoid simultaneous heavy operations on many partitions (amplifies IO + autovacuum churn). Concurrency cap (e.g., 2 partitions at a time for large R3 jobs).

Sharded / distributed environments:
- Maintain a shard orchestration manifest (list of shard DSNs + size + lag metrics).
- Dispatch backfill workers proportionally (weighted by shard size) to keep overall ETA predictable.
- Support partial success bookkeeping: never block entire migration due to a single failing shard— escalate separately.
- Guard central metadata changes (e.g., global unique index emulation) until all shards reached required intermediate state.

Resharding / repartition operations:
- Treat as data copy + verification (shadow copy pattern). Copy → verify (checksums, counts) → atomically swap routing metadata.
- Write traffic during copy: dual-write to old & new until cutover; reconcile late writes with change-data-capture (CDC) tail.

Hotspot mitigation:
- If one partition significantly hotter, apply smaller batch sizes there.
- Consider temporal windows (operate on cold historical partitions during peak traffic windows for hot partitions).

Fallback plan:
- If a single shard/partition lags or errors, allow isolating it (flag skip list) while progressing others— document outstanding debt before Contract phase.

---

## ENUM / Type Evolution (NEW)

Problem: Hard-to-change types (Postgres ENUM) or prematurely rigid domain constraints impede future changes and can cause replica lag spikes when altered.

Guidelines:
- Prefer lookup tables (`status_codes` with FK) or text + CHECK constraint for evolving categorical fields.
- If using ENUM (legacy reasons):
    - Batch add values during low traffic; each `ALTER TYPE ... ADD VALUE` is DDL.
    - Never remove ENUM values— deprecate at application layer and enforce via CHECK if needed.
    - Avoid reordering (not supported) or semantic overloading of old labels.
- For transforming a type (e.g., TEXT → structured JSONB):
    1. Add new column (`details_jsonb`)
    2. Dual-write JSON artifact alongside legacy text
    3. Backfill parse legacy → JSONB with validation
    4. Read-switch & contract

Safer alternative to ENUM for fast-moving domains:
```sql
CREATE TABLE order_status_ref (
    code text PRIMARY KEY,
    active boolean NOT NULL DEFAULT true,
    introduced_at timestamptz NOT NULL DEFAULT now()
);
-- Application: treat inactive codes as legacy; avoid deletes.
```

Validation approach:
- Use CHECK referencing a stable function (but beware function volatility during migration— version via suffix: `validate_order_status_v2()`)

Rolling deprecation:
- Mark code inactive → remove producing logic → eventually remove consumer logic → drop from table (optional) after retention window.

---

## Feature Flag Best Practices (NEW)

Objectives: controlled rollout, fast kill switch, auditability.

Principles:
- Flags must be *fast* (in-memory or cached evaluation); avoid adding network RTT in hot path.
- Deterministic capture: for multi-step jobs, record the flag snapshot used when operation began; store in progress table for traceability.
- Immutable history: log flag state transitions (who, when, old→new) to an audit stream.
- Scoped flags: per-tenant / cohort rollout safer than global big-bang.
- Converge flags: remove within a defined SLA after Contract to reduce config entropy.

Patterns:
- Read-switch flag distinct from write-enable flag (allows enabling writes early, delaying reads until verification passes).
- Canary percentage rollout (random hash of stable key e.g., tenant_id) → ensures consistent assignment.

Example evaluation snippet (conceptual):
```
function isReadFromNewSchema(tenantId):
  base = flags.global.read_switch_new_schema
  if !base.enabled: return false
  if base.mode == "percentage":
      bucket = hash(tenantId) % 100
      return bucket < base.percent
  if base.mode == "allowlist":
      return tenantId in base.tenants
  return base.enabled
```

Kill switch design:
- Toggling flag immediately halts new read-switch behavior without DDL changes.
- Backfill job checks a `paused` flag each loop iteration.

Anti-patterns:
- Combining business semantics & migration gating in a single flag.
- Not documenting expiration date— leads to stale dark config.
- Using request-time random sampling instead of stable hashing (causes inconsistent caching / user experience).

---

## Forward-Fix Header Template (NEW)

Use this header in any migration where Down is disabled (R2/3 destructive or transformative cases).

```sql
-- MIGRATION: 2025-09-15_add_timezone
-- DESCRIPTION: Add timezone column; backfill + enforce NOT NULL later.
-- RISK TIER: R2
-- REVERSIBILITY: forward-fix only after NOT NULL enforcement
-- FORWARD-FIX PLAN:
--   1. Reintroduce legacy behavior gate via flag if errors surge
--   2. Ship follow-up migration to relax NOT NULL if needed
--   3. Optional restore baseline: create shadow column timezone_rollback TEXT NULL
-- OWNER: @team-handle
-- VERIFICATION: row count parity + checksum sample mod 1000 = 7
-- FLAG(S): write_enable_timezone, read_switch_timezone
-- RELATED TICKETS: PROD-1234, TECHDEBT-77
```

Minimum required fields:
- MIGRATION (unique name)
- DESCRIPTION
- RISK TIER
- REVERSIBILITY posture
- FORWARD-FIX PLAN summary
- OWNER handle
- VERIFICATION plan reference
- FLAGS impacting rollout
- RELATED TICKETS / runbook links

---

## Automation & Guardrails (NEW)

Objectives: enforce safety conventions automatically and detect risky patterns early.

Statement protection (Postgres example):
```sql
SET LOCAL lock_timeout = '2s';
SET LOCAL statement_timeout = '30s';
ALTER TABLE users ADD COLUMN timezone TEXT NULL;
```

Index build guard:
```sql
-- Use CONCURRENTLY for large tables
CREATE INDEX CONCURRENTLY IF NOT EXISTS idx_users_timezone ON users(timezone);
```

Migration lint (conceptual rules):
1. Reject `DROP COLUMN` without `-- REVERSIBILITY:` header specifying forward-fix.
2. Warn if large table index lacks `CONCURRENTLY` and table est_rows > threshold.
3. Flag `ALTER TABLE ... SET NOT NULL` when table contains existing NULLs.
4. Flag multiple destructive operations in a single migration file (recommend split).

Sample control table snippet already defined (see Backfill Job Template). Extend with schema versioning:
```sql
CREATE TABLE IF NOT EXISTS schema_version_meta (
    version bigint PRIMARY KEY,
    applied_at timestamptz NOT NULL DEFAULT now(),
    risk_tier text,
    forward_fix boolean DEFAULT false,
    checksum_evidence jsonb
);
```

Automated evidence collection pipeline:
- After migration apply: run verification queries → store hashes / counts in `checksum_evidence`.
- CI step: apply all migrations into ephemeral DB → run linter → run up/down cycle for reversible ones.

Pre-commit Git hook ideas:
```bash
#!/usr/bin/env bash
set -euo pipefail
changed=$(git diff --cached --name-only | grep migrations/ || true)
if [ -n "$changed" ]; then
    python scripts/migration_lint.py $changed || { echo "Migration lint failed"; exit 1; }
fi
```

Runtime dynamic safety:
- Backfill worker refuses to start if existing p95 latency already > safety budget.
- Feature flag gating ensures contract steps only proceed when verification flag is set (distinct from read-switch flag).

Alerting thresholds (example):
- Lock wait p95 > 200ms for 3 consecutive windows.
- Replication lag > 10s (R2) / 5s (R3) sustained 2 mins.
- Backfill throughput stalls (no progress rows in 10 mins while remaining > 0).

---

## Testing & Pre-Prod Strategy (NEW)

Objective: detect correctness and performance regressions *before* production.

Layers:
1. Schema validation: apply all migrations on fresh clone; ensure no ordering / dependency failures.
2. Up/Down cycle (where reversible) to detect drift / irreversibility surprises.
3. Data sample replay: restore anonymized prod snapshot subset (size scaled to risk tier) and run migration end-to-end.
4. Query workload replay: use captured query set (pg_stat_statements top N or slow query log) against pre- and post-migration schema, compare latency & plan shape.
5. Load test backfill: run job with production-like size/time acceleration (scale factor) to measure throughput and resource impact (CPU, IO, WAL volume).
6. Dry-run analyzer: static parse of migration scripts for disallowed patterns.

Plan diff tooling (conceptual):
```
for q in critical_queries:
    plan_before = explain(q, baseline_schema)
    apply_migrations()
    plan_after = explain(q, new_schema)
    diff = compare_plan_structure(plan_before, plan_after)
    flag if diff adds full table scan or removes essential index
```

Performance acceptance gates:
- No critical query latency p95 regression > 10–15% (tier adjustable)
- Backfill projected runtime within maintenance window / SLA
- WAL / binlog generation rate acceptable (no replica saturation predicted)

CDC / event consumers test:
- Validate schema changes do not break serialization contract for downstream systems (e.g., ensure added non-nullable fields have defaults for decoders).

Drift detection rehearsal:
- Intentionally pause mid-backfill; resume; verify idempotent continuation.

Success artifact:
- Attach test run summary (counts, mismatch ratios, plan diffs, throughput) to migration ticket before production approval.

---

## Security & Compliance Notes (NEW)

Goals: avoid data exposure, preserve audit trails, and maintain least privilege during schema evolution.

Data handling:
- Mask / tokenize PII in any shadow copy or temporary export (use approved masking profiles per data class).
- Avoid exporting raw sensitive columns to developer sandboxes; generate synthetic data where possible.

Access control:
- Ensure new tables/columns inherit correct GRANTs— explicitly audit: `
SELECT grantee, privilege_type FROM information_schema.role_table_grants WHERE table_name = 'new_table';`.
- Remove obsolete privileges from deprecated objects during Contract.

Audit continuity:
- If renaming or splitting columns used in audit/event logs, preserve original semantic field in log payload until retention window passes.
- Add version field in emitted events to allow consumers to branch on schema.

Encryption:
- For new sensitive columns, ensure encryption at rest (DB cluster policy) and in-transit unaffected; if application-level encryption, backfill must decrypt + re-encrypt carefully.

Data retention & legal hold:
- Before dropping columns with regulated data, confirm retention / legal hold obligations satisfied. Record approval ID in migration header.

Incident traceability:
- Correlate migration job IDs with security monitoring timeline (store job_name + start/end in audit store) to speed forensic reviews.

---

## Anti-Patterns (NEW)

Common pitfalls that increase risk:

- Single giant `UPDATE` or `DELETE` over entire hot table (causes long locks, bloats WAL) instead of chunked backfill.
- Renaming a column + switching all reads + dropping old column in the same deploy (no rollback seam).
- Adding NOT NULL immediately after adding column without verifying population.
- Dropping index before validating it's unused (lack of query stats review).
- Interleaving destructive DDL with unrelated refactors in one migration file (hard to audit / revert intention).
- Blind ENUM addition for fast-changing business taxonomy.
- Backfill logic embedded in ad-hoc script with no restart / progress tracking.
- Using random per-request sampling for canary (leads to inconsistent cache & user experience) instead of stable hashing.
- Skipping verification evidence and contracting early (can't prove completeness if issues arise later).
- Assuming replica schema instantly updated (race in read-switch causing errors on replicas).

---

## Observability Deep Dive (Expanded)

Key telemetry pillars:
1. Progress & correctness
2. Performance & resource impact
3. Safety (locks, errors, lag)
4. User / business impact (KPIs)

Core metrics (prefix with `migration.` for clarity):
- `migration.backfill.rows_processed_total{job}`
- `migration.backfill.rows_remaining{job}` (derived)
- `migration.backfill.throughput_rows_per_s{job}` (EMA over last N batches)
- `migration.backfill.eta_seconds{job}`
- `migration.drifts.new_nulls{column}` post-freeze
- `migration.verification.mismatch_ratio{job}`
- `db.lag.replication_seconds` (existing infra)
- `db.locks.wait_p95_ms` / `db.locks.blocked_statements`
- `db.query.latency_p95_ms{query_class}` baseline vs current
- `app.errors.rate{service}` (watch for correlated spikes)

ETA formula:
```
remaining = total_target_rows - processed_rows
throughput = smoothed_rows_per_second (EMA)
eta_seconds = remaining / max(throughput, 1)
```

Throughput smoothing (EMA):
```
ema_t = alpha * current_throughput + (1 - alpha) * ema_{t-1}
alpha typically 0.2
```

Index build progress (Postgres):
```sql
SELECT * FROM pg_stat_progress_create_index;
```

Lock inspection:
```sql
SELECT pid, locktype, relation::regclass, mode, granted
FROM pg_locks
WHERE NOT granted;
```

High churn detection (autovacuum pressure): monitor table bloat or autovacuum lag if backfill performs many updates.

Alert recommendations (R3):
- `mismatch_ratio > 0.001` after verification window start
- `eta_seconds` flat (no improvement) for > 3 intervals while `rows_remaining > 0`
- `replication_seconds > threshold` triggers automatic batch size reduction

Dashboards:
- Tile 1: Backfill progress bar (processed / total)
- Tile 2: Latency p95/p99 overlay baseline vs current
- Tile 3: Replication lag & lock waits
- Tile 4: Drift & mismatch ratios
- Tile 5: Error rate & user KPI (e.g., order placements per min)

Sampling strategy for large tables: report progress every N batches or every M seconds to reduce cardinality.

---

## Rollback Playbooks (Clarified)

Define clear actions for each migration phase when aborting.

Phases (EMC): Expand → Migrate (dual-write + backfill) → Read Switch → Contract.

Abort scenarios:
1. During Expand (pre dual-write)
    - Action: Apply Down (if reversible) or drop newly added additive objects.
    - Impact: Minimal; user traffic still on old schema.
2. During Migrate (dual-write active, backfill incomplete)
    - Action: Pause backfill; disable dual-write flag; leave new column/table idle.
    - Do NOT drop new objects— may need them for forward-fix repair.
3. Post Backfill, Pre Read-Switch
    - Action: Similar to above; optionally mark objects deprecated; plan forward-fix if partial contents harmful.
4. After Read Switch (reads primarily new path, old still present for fallback)
    - Action: Flip read flag back to legacy; continue dual-write temporarily; investigate issues.
    - Down often unsafe if new writes already rely on new invariants.
5. After Contract (old path removed / dropped)
    - Action: Forward-Fix only. Reinstate capability via additive re-introduction (shadow column/table) + recompute if possible.

Decision matrix (simplified):
| Phase | Real Down? | Recommended Response |
|-------|-----------|----------------------|
| Expand | Usually yes | Apply Down / revert quickly |
| Migrate | Risky (partial data) | Pause & analyze; keep objects |
| Read Switch | Rarely safe | Revert flag, forward-fix if data divergence |
| Contract | No | Forward-fix, recreate structures |

Operational playbook fields:
- Trigger conditions (errors, latency, mismatch ratio)
- Immediate actions (pause job, disable flags)
- Data capture (collect diagnostics: EXPLAIN, lock state, lag snapshot)
- Escalation contacts (on-call roles)
- Forward-fix outline (if applicable)

Time-bound rollback window:
- Define max interval between Read Switch and Contract (e.g., 48h) to keep fallback feasible.

---

## Tooling Ecosystem Hooks (NEW)

Automate guardrails so human review focuses on semantics, not mechanics.

Suggested components:
- Migration linter (static): enforces headers, forbids dangerous combos, ensures forward-fix annotation.
- Plan regression checker: captures EXPLAIN JSON for critical queries each PR.
- Size estimator: tags migrations with estimated runtime & row touch count.
- Drift detector: scheduled job ensuring schema in all environments matches git-defined migrations state.
- Dependency graph visualizer: surfaces ordering constraints (e.g., cannot drop column still referenced by views).

CI pipeline example stages:
1. Lint migrations (scripts/migration_lint.py)
2. Spin ephemeral DB; apply all migrations
3. (Optional) Run reversible migrations Down then Up to assert safety
4. Run verification harness (sample dataset)
5. Produce HTML report (plans, estimated costs, flagged risks)

Git pre-merge comment bot:
- Summarizes: risk tier, forward-fix posture, objects changed, estimated backfill runtime, required approvals outstanding.

Runtime daemon:
- Watches `schema_version_meta` for newly applied high-risk versions; triggers verification job automatically.

Alert integration:
- Linter emits structured JSON; CI converts to annotations visible inline in code review.

Version drift query (Postgres):
```sql
SELECT version FROM schema_version_meta ORDER BY version DESC LIMIT 1;
```
Compare across environments; mismatch triggers sync alert.

---

## Decision Cheat Cards (NEW)

Quick reference scenarios (default actions):

Add nullable column for future use:
- Tier: R1
- Plan: Expand now, backfill defaults later, optionally enforce NOT NULL after verification.

Rename column (created_at → placed_at):
- Tier: R2 (if widely referenced)
- Plan: Add new column + dual-write + backfill + read-switch + drop later.

Drop unused large index:
- Tier: R1/R2 (check usage stats)
- Plan: Verify no references in query stats; drop during low traffic; monitor plan changes.

Split `full_name` into first/last:
- Tier: R3 (transform)
- Plan: Add new columns + dual-write parse on write + backfill parse existing + read-switch + forward-fix posture.

Add NOT NULL to busy table column:
- Tier: R2
- Plan: Backfill in batches; soft-enforce in app; validate zero violations; then ALTER.

Merge tables (users + user_profiles):
- Tier: R3
- Plan: Create new unified table; dual-write; copy with checksum verification; read-switch; forward-fix only.

Add enum value to fast-changing domain:
- Tier: R2
- Prefer lookup table; if ENUM must be used, schedule low-traffic DDL.

Repartition huge table by month:
- Tier: R3
- Shadow copy partitioned layout; incremental copy; verify; swap routing; forward-fix only.

Introduce computed materialized view for reporting:
- Tier: R1
- Build concurrently; validate row counts; monitor refresh cost.

---

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

---

See also:
- `patterns/templates/forward-fix-template.md`
- `patterns/templates/backfill-job-template.md`
- `patterns/migration-lint-guidelines.md`

These supplemental templates operationalize the guidance above (risk posture, forward-fix headers, backfill mechanics, lint enforcement).
