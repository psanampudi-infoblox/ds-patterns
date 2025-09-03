# All about database views

A practical guide to using SQL views effectively: what they are, when to use them, when to prefer application code, and how different databases implement them.

## TL;DR
- Use a standard view to DRY up repeatable joins/filters and expose a stable read contract.
- Use a materialized view when the query is expensive, read often, and the data can be slightly stale.
- Don’t use views for logic that spans services, performs side effects, or needs retries/feature flags—do that in application code.

## What is a view?
- A saved query that presents a virtual table to consumers.
- Encapsulates joins, filters, and computed columns under a stable name.
- Helps with consistency, readability, and permissioning.

### Standard view
- Definition: `CREATE VIEW name AS SELECT ...`
- Virtual: no storage of result rows; always reflects current base table state.
- Performance: relies on optimizer to push predicates into base tables.
- Indexing: you index base tables, not the view.
- Updatability: some views can be updatable; many require INSTEAD OF triggers.

### Materialized view
- Definition (PostgreSQL/Oracle): `CREATE MATERIALIZED VIEW name AS SELECT ...`
- Physical: stores result rows; requires refresh to update.
- Indexing: can index the materialized view to speed lookups.
- Staleness: must refresh on a schedule or by event.

## Cross‑DB cheat sheet

- PostgreSQL
  - Standard views: `CREATE VIEW v AS SELECT ...`.
  - Materialized: `CREATE MATERIALIZED VIEW mv AS SELECT ...`.
  - Refresh: `REFRESH MATERIALIZED VIEW mv;` or `REFRESH MATERIALIZED VIEW CONCURRENTLY mv;` (needs a unique index on the MV).
  - Security: `CREATE VIEW ... WITH (security_barrier=true)` to prevent predicate reordering that could bypass RLS; `WITH LOCAL CHECK OPTION` for updatable views.

- Oracle
  - Standard views + materialized views with rich refresh options (fast/complete, on commit/ demand).
  - Consider materialized view logs for incremental refresh.

- SQL Server
  - Indexed views are the closest analog to materialized views; require schema and session settings; persisted index enforces refresh.

- MySQL/MariaDB
  - No native materialized views; simulate with tables + triggers/jobs.

## When views make sense
- You repeat the same joins/filters across queries—define once, reuse everywhere (DRY).
- You want a stable contract that shields consumers from base table changes.
- You need DB‑level security/permission control (grant SELECT on the view, not tables).
- You want predicate pushdown and planner optimizations handled in the DB.
- Reporting/analytics read patterns over operational tables.

## When to prefer application code
- The logic spans multiple services/DBs or needs external API calls.
- You need side effects (create/update downstream artifacts), retries, or backoff.
- Feature flags, per‑tenant behavior, or complex conditionals are required.
- You need strong audit trails, idempotency semantics, or workflows.

## Performance considerations
- Predicate pushdown: write views so filters can push into base tables (avoid wrapping with functions that block pushdown).
- Avoid over‑nesting: deep stacks of views can confuse optimizers and humans.
- Materialized view refresh:
  - Postgres: `REFRESH MATERIALIZED VIEW CONCURRENTLY mv;` to avoid blocking reads; requires a unique index.
  - Schedule refresh (cron/job) or trigger by events; monitor runtime.
  - Incremental refresh is not native in Postgres—simulate with change tables or use extensions.

## Security and governance
- Row‑level security (RLS): ensure the view doesn’t accidentally bypass RLS; use `security_barrier` and test.
- Least privilege: grant SELECT on views to limit exposure.
- Check‑option: `WITH [LOCAL] CHECK OPTION` on updatable views to enforce view predicates on writes.
- Ownership chaining: be aware of definer’s rights vs invoker’s rights behavior in your DB.

## Updatable views (quick notes)
- Some simple views (single table, no aggregates) are updatable.
- Others need INSTEAD OF triggers to map writes to base tables.
- Be explicit about which columns are writable to avoid surprises.

## Practical patterns
- Contract view: expose `v_domain_model` as a stable API; evolve base tables independently.
- Compatibility view: keep `v_old_shape` for legacy consumers during migrations.
- Security view: mask PII or filter rows per role.
- Aggregation view: centralize business metrics; materialize if heavy.

## Example snippets (PostgreSQL)

Standard view
```sql
CREATE VIEW app.v_orders_enriched AS
SELECT o.id,
       o.account_id,
       o.created_at,
       c.name   AS customer_name,
       SUM(oi.qty * oi.price) AS total_amount
FROM app.orders o
JOIN app.order_items oi ON oi.order_id = o.id
JOIN crm.customers c     ON c.id = o.customer_id
GROUP BY o.id, o.account_id, o.created_at, c.name;
```

Materialized view + refresh
```sql
CREATE MATERIALIZED VIEW app.mv_monthly_revenue AS
SELECT date_trunc('month', created_at) AS month,
       account_id,
       SUM(total_amount) AS revenue
FROM app.v_orders_enriched
GROUP BY 1, 2;

CREATE UNIQUE INDEX ON app.mv_monthly_revenue (month, account_id);
-- Later
REFRESH MATERIALIZED VIEW CONCURRENTLY app.mv_monthly_revenue;
```

Updatable view with check option
```sql
CREATE VIEW app.v_active_users AS
SELECT id, email, is_active
FROM app.users
WHERE is_active = true
WITH LOCAL CHECK OPTION;
```

## Decision guide
- Is the logic pure SQL over one DB? Use a view (or materialize if heavy).
- Does it require external systems/side effects? Do it in application code.
- Is staleness acceptable and performance critical? Materialize.
- Do you need a stable read contract or security boundary? View.

## Testing and observability
- EXPLAIN/EXPLAIN ANALYZE the view query paths.
- Add alerts for materialized view refresh failures and latency.
- Track usage: which clients read which views and with what predicates.

## Naming and versioning
- Prefix/view schema: `v_` for views, `mv_` for materialized; or separate schemas (e.g., `read_api.*`).
- Version when changing contracts: `v_orders_enriched_v2` with a deprecation period.

---
Use views to push read composition and security closer to the data; use code for orchestration, cross‑service logic, and side effects. Combine both intentionally for a maintainable, performant system.
