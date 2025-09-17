# Forward-Fix Migration Template

Use this template when a migration cannot safely provide a true Down path (irreversible or high-risk destructive / transformative change).

```sql
-- MIGRATION: <timestamp_or_seq>_<short_name>
-- DESCRIPTION: <one line>
-- RISK TIER: R1|R2|R3
-- REVERSIBILITY: forward-fix only | reversible pre-contract | reversible fully
-- FORWARD-FIX PLAN:
--   1. <step if issue occurs>
--   2. <follow-up migration / remediation>
--   3. <data restoration strategy (snapshot / shadow / recompute)>
-- OWNER: @team-handle
-- VERIFICATION: <row counts + checksum + plan diff refs>
-- FLAGS: <flag_list>
-- RELATED TICKETS: <JIRA/PRD links>
-- APPROVALS: <arch-review id / security if needed>
-- DEPENDENCIES: <other migration ids>
-- CREATED_AT: <ISO timestamp>
```

## Sections Guidance

- MIGRATION: consistent naming (e.g., `20250915_add_timezone_column`).
- REVERSIBILITY: be explicit; if only forward-fix, justify briefly.
- FORWARD-FIX PLAN: concrete, minimal steps (avoid vague phrases like "revert").
- VERIFICATION: reference evidence artifacts (dashboards, logs, stored procedures output).
- FLAGS: list both write and read switch flags distinctly.
- APPROVALS: include required sign-offs for R3 (SRE, security, data governance).

## Example

```sql
-- MIGRATION: 20250915_split_full_name
-- DESCRIPTION: Introduce first_name / last_name replacing full_name.
-- RISK TIER: R3
-- REVERSIBILITY: forward-fix only once reads switched
-- FORWARD-FIX PLAN:
--   1. If downstream breakage: re-enable legacy read fallback flag
--   2. Ship N+1 migration adding computed column full_name_computed
--   3. Rehydrate analytics export using first/last snapshot
-- OWNER: @identity-platform
-- VERIFICATION: checksum sample mod 1000, drift monitor zero after 2 intervals
-- FLAGS: write_split_name, read_split_name
-- RELATED TICKETS: PROD-4567, DATA-112
-- APPROVALS: ARCH-REVIEW-22, SRE-SHIFT-0915
-- DEPENDENCIES: 20250910_add_name_columns
-- CREATED_AT: 2025-09-15T09:12:00Z
```
