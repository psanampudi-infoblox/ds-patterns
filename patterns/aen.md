## Canonical Resource Identifiers (ARN‑like Schemes) — Practical Usage Guide

Canonical (aka globally stable) resource identifiers let a SaaS platform unambiguously reference any asset across regions, accounts / tenants, and internal subsystems. AWS popularized the pattern with ARNs; most modern platforms implement a variant (GCP full resource names, Azure Resource IDs, Stripe IDs, GitHub global node IDs, Kubernetes UIDs, etc.). This guide focuses on pragmatic design and consumption—no math, minimal theory.

### 1. Why They Exist (Practical Needs)
Core drivers:
1. Global Uniqueness: Prevent clashes when aggregating data (logging, billing, analytics) from many partitions/shards/regions.
2. Decoupling & Opacity: Allow internal reshaping (move a resource to a different shard/region) without breaking clients.
3. Authorization & Scoping: Act as the principal subject in IAM / policy evaluation (pattern: policy statements reference identifier patterns / prefixes / segments).
4. Idempotency & Replay Safety: Same identifier reused in retries avoids duplicate creation.
5. Trace Correlation: Single stable key to stitch logs, metrics, traces, events.
6. Federation & Integrations: Third parties hold only the identifier, not internal primary keys.
7. Caching & CDN Keys: Provide cache partition boundaries.
8. Support for Eventual Consistency: Identifier issuance can precede full materialization; later components converge around it.

### 2. Common Solution Shapes
Two broad styles:
- Structured Multi‑Segment (human scannable): e.g. `arn:aws:s3:us-east-1:123456789012:bucket/my-bucket`
- Opaque Token (uniform, fixed-ish length): e.g. `acct_1PAbCdEfGhIJ` (Stripe), `gho_...` (GitHub tokens), `uid` (Kubernetes object UUID)

Selection heuristics:
- Choose structured when policy scoping or wildcard listing by service/region/account is critical.
- Choose opaque when you want freedom to change internal topology, limit client parsing, and keep surface small.
- Mix: Keep an outer structured frame, with an opaque trailing segment for the specific resource instance.

### 3. Comparative Snapshot (Practical Behaviors)
AWS ARN:
- Format: `arn:partition:service:region:account-id:resource` (resource can further embed type/path).
- Pros: Rich scoping, easy for IAM patterns, intuitive segmentation.
- Cons: Client over-parsing leads to brittleness; variations (some services leave region/account blank) cause edge handling.

GCP Full Resource Names:
- Hierarchical path style: `//cloudresourcemanager.googleapis.com/projects/123456/zones/us-central1-a/...`
- Pros: Uniform across APIs; natural for hierarchical traversal.
- Cons: Lengthy; multi-component string operations cost more; not always stable if hierarchical parents change.

Azure Resource IDs:
- `/subscriptions/{subId}/resourceGroups/{rg}/providers/{namespace}/{type}/{name}` (+ nested children).
- Pros: Strong lifecycle grouping (resource group); predictable.
- Cons: Case insensitivity + length constraints create normalization pitfalls.

Stripe IDs:
- Patterned opaque: `cus_`, `acct_`, `pi_` prefixes + short base62-like bodies.
- Pros: Prefix instantly reveals coarse type; remainder opaque—internal freedom.
- Cons: Sorting lexically mixes types; consumers sometimes branch on prefix instead of using metadata.

Kubernetes UIDs:
- Pure opaque UUID assigned server-side.
- Pros: No semantics; safe to move objects across nodes.
- Cons: Policies (RBAC) rely on names & labels rather than the UID, so you juggle two identifiers.

GitHub GraphQL Node IDs:
- Base64 of a type tag + database ID.
- Pros: Single global ID space for relay caching.
- Cons: Decoding temptation (clients shouldn’t rely on internal integer that leaks out after decode).

### 4. DNS Usage (Why Put IDs in DNS?)
Motivations:
1. Multi‑tenant Routing: `tenant-id.region.platform.example.com` directs traffic without central lookup.
2. Isolation & Blast Radius: Each tenant subdomain isolates cookies / caches.
3. Vanity & Self‑Service: Human recognizable environment or tenant handles.
4. Certificate Automation: Wildcard or per‑tenant cert issuance via ACME challenges.

Practical constraints:
- Label Length ≤ 63 chars; full FQDN ≤ 253 chars.
- Lowercase (enforce early) to avoid accidental duplication in caches.
- Avoid characters outside `[a-z0-9-]` → impacts choice of encoding.
- Don’t embed volatile shards/regions if you plan to migrate (treat region as separate label if needed).

Pitfalls:
- High cardinality ephemeral IDs in DNS cause zone bloat & slow propagation.
- Leaking internal semantics (shard numbers, environment names) invites scraping.
- Mixed case Base32 (RFC 4648) vs DNS requirements → must lowercase & maybe strip padding.

### 5. Encoding Scheme Practicalities (No Math, Just Usability)
Goals of an encoding: safety across transport/storage layers, readability, predictable length, optional sortability.

Common choices:
- Base16 (hex): Ubiquitous tooling; doubles length; safe everywhere.
- Base32 (Crockford variant): Human friendly (no ambiguous chars); good for manual transcription; output longer than Base62.
- Base36 / Base62: Compact, widely recognized; may introduce case sensitivity (Base62) which DNS cannot preserve; need normalization.
- Base58 (Bitcoin alphabet): Avoids look-alikes; less library ubiquity; slower implementations.
- Base64 URL-safe: Dense; includes `-` and `_`; padding `=` often trimmed; not DNS-safe if uppercase expectation not controlled.
- Base85 / Z85: Very dense; niche; limited standard library support; readability suffers.
- Custom Crockford+Checksum: Adds error detection; more work; only worth it when human entry is common (activation codes, support flows).

Pragmatic guidance:
- Prefer a widely supported alphabet over marginal density gains (ops tooling & log search matter).
- For DNS labels: stick to lowercase alphanumerics and `-`; consider a Base32 lowercase alphabet or a restricted Base36 with lowercase only.
- If clients never type IDs manually (pure API / machine use), go denser (URL-safe Base64 without padding) but provide canonical form rules.
- Document normalization (case folding, padding rules) explicitly.

### 6. Structured vs Opaque Trade-offs (Usage Perspective)
Structured Pros:
- Easier policy wildcards (e.g., prefix match for all buckets in a region).
- Self‑describing for debugging (“What service/account is this?”).
Structured Cons:
- Clients parse & rely on current segmentation; refactors become breaking.
- Inconsistent optional fields create parsing complexity.

Opaque Pros:
- Maximum internal agility; single comparison operation.
- Smaller (can choose dense encoding) & uniform length aids storage indexes.
Opaque Cons:
- Need side channels (lookup APIs, metadata services) for classification and listing.
- Harder to implement coarse-grained access rules without tagging/attributes.

Hybrid Pattern:
`scheme:region:account:TYPE:opaque-id` — region/account stable, leaf remains opaque. Limit semantic churn to the opaque portion.

### 7. Issuance & Lifecycle (Operational Concerns)
Issuance moments:
- Client-supplied (dangerous: collisions, guessability) – generally avoid unless idempotency key pattern.
- Server pre‑allocation before heavy processing (lets you stream logs & events referencing future state).
- Post‑commit issuance (ensures only fully materialized resources get IDs) – trades off earlier correlation.

Idempotency:
- Provide an optional idempotency key separate from the canonical ID; return existing resource if the same key collides.

Migration & Renames:
- Avoid embedding mutable properties (user’s display name) into identifier.
- If you must present human names, use a separate slug that can change while the canonical ID remains stable.

Deletion & Reuse:
- Default: never reuse — simplifies auditing and event stream replay.
- If storage pressure forces reuse (rare), partition ID space by era or add a version discriminator; document clearly.

### 8. Policy & Authorization Usage
Patterns:
- Prefix / segment pattern matching (ARN style) for coarse scoping.
- Attribute-based evaluation: map canonical ID → metadata (labels/tags) → policy engine.
- Least privilege: encourage referencing exact identifiers; allow wildcards only for automation roles.

Avoid:
- Encoding roles or permission hints inside IDs (e.g., `admin-` prefixes) — treat permissions as orthogonal metadata.

### 9. Observability & Tooling
Log Format:
- Always key logs with `resource_id=` (consistent field name). Avoid multiple synonyms (`id`, `resourceId`, etc.).
- Provide a debug CLI / internal web lookup: `describe <id>` returns core metadata.

Tracing:
- Tag primary spans with identifier; propagate only the identifier, not large serialized metadata.

Metrics:
- Avoid high-cardinality raw identifiers as label values (explodes TS count). Instead, map to coarser dimensions (type, tier, region). Provide ad‑hoc drill tools instead of raw metric labels.

### 10. Backfills, Imports, & External IDs
When ingesting external systems:
- Store external ID separately (`external_ref`), keep internal canonical stable.
- If you must surface external ID to clients, do not conflate it with the canonical; collisions & format drift risk.
- Support an “upsert by external ID” path that internally maps or creates a canonical ID.

### 11. Anti‑Patterns (Field Notes)
- Sequential Integers Exposed Publicly: Predictable enumeration & scraping.
- Parsing Without Contract: Clients regex parse structured IDs; a format change breaks them (publish an EBNF / JSON schema if parsing is expected).
- Dual Encodings: Accepting multiple textual forms (e.g., with/without hyphens) without normalizing → cache misses.
- Hard‑coding Region Into Opaque ID: Later cross‑region replication becomes awkward.
- Overloading for Type Discovery: Using prefix as sole type source; a refactor adding new resource classes breaks consumers — provide type in API payload.

### 12. Checklist (Pragmatic)
Design Time:
- [ ] Decide: structured, opaque, or hybrid (document rationale).
- [ ] Pick encoding with explicit DNS / URL / log considerations.
- [ ] Define single canonical string form + normalization rules.
- [ ] Publish validation contract (regex or BNF) to clients.
- [ ] Ensure generation pathway is horizontally scalable (no single DB sequence bottleneck, if high QPS).
- [ ] Decide reuse policy (default: none) & document.
- [ ] Provide lookup & describe tooling (CLI, internal API).
- [ ] Establish logging field naming standard.
- [ ] Define policy integration pattern (prefix match vs attribute lookup).
- [ ] Write guidance against client-side parsing beyond documented contract.

Operational:
- [ ] Add lint rule / tests ensuring no undisclosed segments creep in.
- [ ] Monitor issuance rate & collision (if probabilistic) or allocator saturation.
- [ ] Track average identifier length; watch for unbounded growth.
- [ ] Verify high-cardinality labels aren’t leaking IDs into cardinality-sensitive metrics.

Developer Experience:
- [ ] Provide sample code (generate, validate, normalize) in main SDK languages.
- [ ] Offer redaction helper (mask all but last N chars for logs in lower environments).

### 13. Glossary (Narrow, Practical)
- Canonical ID: Stable, authoritative identifier for a resource lifecycle.
- Structured ID: Multi-segment textual form exposing coarse attributes.
- Opaque ID: Identifier with no externally interpretable semantics.
- Normalization: Transforming input to canonical form (case fold, trim padding).
- Idempotency Key: Client-supplied token to safely retry create operations.
- Wildcard Scoping: Authorization mechanism allowing pattern (prefix) matches over identifiers.

### 14. Future Enhancements (TODO)
- Add concrete internal example (bloxid) once format is publishable.  // TODO
- Provide example regex patterns for each style.  // TODO
- Add short code snippets for generation & normalization.  // TODO
- Potential cross-link to reliability & modelling docs after approval.  // TODO

---
Summary: Choose a design that balances policy needs, operational agility, and client simplicity. Keep semantics minimal, publish a contract, and enforce normalization early. Resist the urge to leak internal topology; empower tooling instead of over-stuffing the identifier.

