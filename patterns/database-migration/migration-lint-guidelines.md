# Migration Lint Guidelines

Automated checks to enforce safety, consistency, and traceability for database schema/data migrations.

## Objectives
- Detect destructive or high-risk operations lacking explicit forward-fix posture
- Enforce presence of required metadata headers
- Encourage performance-safe patterns (online index builds, chunking strategies)
- Provide early feedback in CI before human review

## Scope
Files under `migrations/` (adjust path as needed). Applies to SQL or DSL-based migration files.

## Required Header Fields (Regex enforced)
- `-- MIGRATION:` unique identifier
- `-- DESCRIPTION:` non-empty
- `-- RISK TIER:` R1|R2|R3
- `-- REVERSIBILITY:` includes `forward-fix` or `reversible`
- For forward-fix only: `-- FORWARD-FIX PLAN:` line present

## Rule Set
| ID | Rule | Severity | Description | Auto-Fix |
|----|------|----------|-------------|----------|
| R001 | Missing required header fields | error | Reject migration lacking metadata | no |
| R002 | DROP COLUMN without forward-fix posture | error | Must state irreversibility | no |
| R003 | ALTER TABLE ... SET NOT NULL while nulls exist (detected via optional pre-check script) | warn | Suggest backfill first | no |
| R004 | Large index without CONCURRENTLY (Postgres) | error | Requires concurrent build | no |
| R005 | Multiple destructive ops in single file | warn | Recommend splitting | no |
| R006 | No verification plan mention in header | warn | Encourage evidence | no |
| R007 | Enum alteration present | info | Suggest lookup table alternative | no |
| R008 | Missing risk tier for destructive op | error | Must classify | no |
| R009 | Hardcoded long statement timeout absence for R3 | warn | Suggest guard | no |
| R010 | Unqualified wildcard SELECT in backfill script | warn | Encourage explicit columns | no |

## Heuristics
- Large table threshold configurable (e.g., > 10M rows) pulled from stats metadata file.
- Detect destructive ops via regex: `DROP (COLUMN|TABLE)` | `ALTER TABLE .* (SET DATA TYPE|TYPE)`.
- Dual-write expectation: if rename pattern detected (new column added similar to legacy), enforce presence of plan comment `dual-write`.

## Example Lint Output (JSON)
```json
{
  "file": "migrations/20250915_split_full_name.sql",
  "issues": [
    {"rule": "R004", "severity": "error", "message": "Index on users(email) missing CONCURRENTLY"},
    {"rule": "R006", "severity": "warn", "message": "Header lacks VERIFICATION reference"}
  ]
}
```

## CI Integration
1. Collect changed migration files from diff
2. Run linter script (Python/Go) producing JSON
3. Fail build if any `error` severity present
4. Post summary as PR comment (GitHub/GitLab API)

## Optional Pre-Check Script
- Connect to staging clone, sample null counts for columns targeted by NOT NULL additions; emit warnings if >0

## Suggested Configuration File (`migration_lint.yml`)
```yaml
table_size_threshold: 10000000
require_verification_header: true
timeout_required_risk_tiers: ["R3"]
blocked_statements:
  - "ALTER TABLE .* SET UNLOGGED"
allow_patterns:
  - "CREATE INDEX CONCURRENTLY"
```

## Future Enhancements
- Automatic plan diff capture for annotated queries
- Check CDC compatibility (new NOT NULL columns with default)
- Estimate WAL volume impact heuristically

## References
See `data-migrations.md` sections: Risk Classification, Automation & Guardrails, Forward-Fix Header Template.
