# Backfill Job Template

Reliable, idempotent, observable job pattern for migrating or populating data.

## Goals
- Idempotent (safe retries)
- Chunked & throttled (protect p95 latency, replication lag)
- Progress + ETA visibility
- Kill switch & safe pause/resume

## Control Table
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

## Pseudocode Skeleton
```pseudo
const JOB = "<job_name>"
const BATCH_SIZE = 5000
const TARGET_P95_MS = 120
const MAX_REPL_LAG_S = 5
const SLEEP_BASE_MS = 250

state = loadProgress(JOB) or initProgress(JOB)
if state.status == 'completed': exit

loop:
  if killSwitchEnabled(JOB): markPaused(JOB); exit

  rows = select id, <fields>
         from <table>
         where id > state.last_id and <predicate_for_rows_needing_work>
         order by id asc
         limit BATCH_SIZE
         for update skip locked

  if rows.empty(): markCompleted(JOB); break

  begin
    for r in rows:
      performTransformation(r)
  commit

  new_last = maxId(rows)
  updateProgress(JOB, new_last, rows.count)

  metrics.emit(rows_processed=rows.count, last_id=new_last,
               throughput=currentThroughput(), eta=estimateEta())

  if currentP95Latency() > TARGET_P95_MS or replicationLag() > MAX_REPL_LAG_S:
     sleep(adaptiveBackoff())
  else:
     sleep(SLEEP_BASE_MS)
```

## Adaptive Backoff Example
```pseudo
function adaptiveBackoff():
  over = currentP95Latency() / TARGET_P95_MS
  return clamp(SLEEP_BASE_MS * over, 250, 5000)
```

## Idempotency Strategies
- Use predicates like `WHERE target_col IS NULL` or version columns.
- Upserts (`INSERT ... ON CONFLICT DO UPDATE WHERE`) for merge operations.
- Store derived hashes; skip if hash unchanged.

## Verification Hooks
- Per N batches: compute checksum sample (mod id space)
- Emit mismatch ratio metric

## Pause / Resume
- Pause: set status='paused'; loop exits after current batch commit.
- Resume: set status='running'; job picks up from `last_id`.

## Metrics (suggested)
- migration.backfill.rows_processed_total{job}
- migration.backfill.throughput_rows_per_s{job}
- migration.backfill.eta_seconds{job}
- migration.backfill.retries_total{job}
- migration.backfill.drifts{job}

## Completion Criteria
- Remaining predicate returns zero rows in 2 consecutive scans
- Drift counter stable (no new rows needing work after freeze time)
- Mismatch ratio below threshold (e.g., <0.01%)

## Common Pitfalls
- Large transactions causing bloat / lock contention
- Ignoring replication lag leading to replica query timeouts
- Lack of restart metadata causing duplicate processing or gaps

## Checklist
- [ ] Predicate excludes already-processed rows
- [ ] Upsert or compare-and-set prevents double writes
- [ ] Kill switch integration
- [ ] Metrics emitted
- [ ] Verification queries documented
- [ ] Throughput & ETA visible
