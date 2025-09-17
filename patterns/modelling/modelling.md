## Modelling Tutorial: From Data Models to Domain Models & Practical DDD

> Goal: Give a pragmatic mental model and a repeatable workflow for modelling in real-world systems. Focus on clarity, alignment, and evolution over dogma.

### 1. Why Modelling Matters (30‑second pitch)
Poor models calcify. Good models let you change direction without multi-quarter refactors. Modelling is about:
- Shared language between business + engineering.
- Making invariants explicit (so code + data enforce them).
- Deliberately separating concerns: domain intent vs storage, vs integration contracts, vs analytics.

### 2. A Landscape of Model Types
Not all “models” are the same. Conflating them creates coupling and pain.

| Model Type | Primary Audience | Optimized For | Change Frequency | Typical Artefacts |
|------------|------------------|---------------|------------------|-------------------|
| Domain Model | Domain + engineers | Behavior + invariants | Moderate | Entities, Value Objects, Domain Events |
| Data/Persistence Model | DB engineers, backend | Efficient storage + query | Low–Moderate | Tables, Columns, Indexes |
| Integration / Contract Model | External teams, clients | Stability + compatibility | Low | Protobufs, OpenAPI, XSD, JSON Schema |
| API Representation (DTO/View) | Client app devs | Convenience + decoupling | Moderate | REST/GraphQL resource shapes |
| Analytical / Reporting Model | Data & analytics | Aggregation + historical trends | Slow | Star/Snowflake schemas, Views |
| Projection / Read Model | App + UX | Fast query for a use case | Regeneratable | Materialized views, denormalized tables |

Practical rule: Each boundary (domain ↔ integration ↔ persistence ↔ read) is an opportunity to translate intentionally. Accidental reuse bleeds concerns.

### 3. Core Distinctions (Cheat Sheet)
- Entity: Has identity across time (Order, Customer).
- Value Object: Defined only by its attributes, immutable (Money, DateRange, Coordinates).
- Aggregate: Consistency boundary; cluster of entities/values with a root enforcing invariants.
- Domain Service: Domain logic that does not fit naturally on a single Entity/Value Object.
- Application Service: Orchestrates use cases, coordinates domain objects + side-effects.
- Repository: Collection-like abstraction for retrieving/storing aggregates (not ad-hoc queries dump).

### 4. Approaches to “Starting” Modelling
You may begin from different angles; know what you gain and what you defer.

| Starting Style | What You Do First | Strengths | Risks / Blind Spots |
|----------------|-------------------|-----------|---------------------|
| Contract-First (proto/OpenAPI/XSD) | Define external messages/resources | Early API alignment; parallelism | Locks premature abstractions; domain emerges late |
| Domain-First | Ubiquitous Language, contexts, aggregates | Strong invariants; expressive code | Can feel “abstract” to stakeholders early |
| Data-First (ERD) | Tables/relationships | Clear persistence cost model | Data shape ossifies behavior decisions |
| Event-First (Event Storming) | Domain events timeline | Discovers hidden workflows | Over-producing events; analysis paralysis |
| UI/Workflow-First | Screens / user journeys | Rapid validation of UX value | Domain invariants bolted on later |

Pragmatic hybrid: Explore domain language + key events, sketch 1–2 aggregates, then shape initial contracts—iterate.

### 5. Minimal DDD Slice (What to Actually Do)
1. Collect terms → build a Ubiquitous Language glossary.
2. Identify Bounded Contexts: Where does language meaning shift?
3. For each context, define main aggregates + invariants (what must be true atomically?).
4. Draft external seams (protobuf/OpenAPI) only for crossing-context or external integration.
5. Map aggregates to persistence (don’t force 1:1 with tables). Flatten value objects where pragmatic.
6. Identify initial domain events (OrderPlaced, PaymentCaptured) if they unlock workflows or integration.
7. Create one vertical slice (API → App Service → Domain → Repo → DB) to validate layering.

### 6. Aggregates & Invariants
Guidelines:
- Keep aggregates small; single transaction boundary.
- Invariant examples: “Total must equal sum of lines”; “Cannot ship unpaid Order”.
- If two invariants require cross-aggregate coordination → use a domain/service or a process manager (saga) eventually consistent.
- Avoid chatty multi-aggregate transactions (hint you modelled boundaries wrong or need eventual consistency).

### 7. Mapping Domain to Persistence
Patterns:
- Value Object flattening: `Money(amount, currency)` → columns `amount_decimal`, `currency_code`.
- Aggregate root foreign keys: Child tables carry root FK; enforce via unique + constraint combos.
- Soft vs hard deletes: Domain may say “Cancelled”; DB may keep row with state vs physically removing.
- Denormalize for projections, NOT inside the core write model (unless justified by invariant locality).

Anti-patterns:
- Letting ORM auto-generated entities become the “domain model” (anemic, leaky persistence concepts).
- Exposing internal IDs from persistence as public contract prematurely.

### 8. Integration & Anti-Corruption Layer (ACL)
When external system concepts don’t align with your domain (e.g., CRM “Account” vs your “Customer”), use translators.
- ACL shields domain from churn and weird semantics.
- Avoid sprinkling conversion logic across services—centralize boundary translation.

### 9. Projections / Read Models
Use when:
- Read query complexity or latency threatens core aggregate tables.
- Different shape needed (e.g., dashboard, search index).
- You require pre-computed joins or aggregations.

Design:
- Treat them as disposable—rebuild from events or canonical state.
- Keep transformation deterministic; log version to manage rebuilds.

### 10. Evolution & Versioning
Contracts (proto/JSON): additive first (add optional fields), never recycle field numbers/keys.
Persistence: expand→migrate data→contract (drop obsolete). Use idempotent migrations.
Events: prefer immutable semantics; add new event types rather than mutating old meaning.
Feature toggles + dual-writes help smooth transitions.

### 11. Modelling Heuristics & Smell Detectors
Smells:
- “Utility” domain classes with only getters/setters.
- Large aggregate updated by >70% of unrelated use cases.
- Frequent need for distributed transactions across aggregates.
- API resource names bleeding storage terms (e.g., `tbl_customer`).

Heuristics:
- Name things in business language; if you can’t, you might not understand the domain.
- Delay modeling rarely used edge cases; keep a parking lot.
- Prefer explicit Value Objects over primitive obsession (clarity + invariants).

### 12. Practical Workflow Checklist
Run this for each new capability:
- [ ] Terms captured & clarified
- [ ] Bounded context impact known
- [ ] Aggregate invariants listed
- [ ] Contract diff reviewed (if external)
- [ ] Persistence impact sized (migration? index?)
- [ ] Projection need? (if yes, rebuild strategy documented)
- [ ] Versioning / compatibility considered
- [ ] Observability hooks (log events/metrics) defined

### 13. Example Mini Domain (Order Service)
Glossary (excerpt):
- Order: Customer intent to purchase; transitions: Created → Paid → Shipped → Completed/Cancelled.
- OrderLine: Value object capturing SKU, quantity, price snapshot.
- Money: Value object (amount, currency) enforcing non-negative + currency rules.
- Events: `OrderPlaced`, `PaymentCaptured`, `OrderShipped`.

Sample invariant: Cannot transition to Shipped unless Paid; cannot modify lines after Paid.

### 14. Tool & Artefact Selection Guide
| Situation | Favors | Rationale |
|-----------|--------|-----------|
| Public stable external API | Protobuf/OpenAPI first | Consumer alignment & versioning discipline |
| Rapid internal iteration | Domain-first + code-first | Less upfront contract drag |
| Complex workflow discovery | Event storming | Visual sequence reveals gaps |
| Data warehouse alignment | ERD / dimensional | Optimize analytics performance |
| Multi-team schema negotiation | Contract-first | Parallel development |

### 15. FAQ

Q: If I start by defining `.proto` files, what modelling style is this?
A: Contract-first integration modelling. You’re shaping external/API messages, not yet the internal domain model.

Q: If I start my development using XSDs?
A: Also contract/schema-first—XML flavor. Emphasizes external structure stability over domain behavior early.

Q: Is an ER diagram the same as a domain model?
A: No. ERD optimizes storage relationships; domain model encodes behavior + invariants.

Q: Can my domain entities map 1:1 to tables?
A: Sometimes, but forcing it creates leakage (persistence constraints shaping domain). Accept divergence where behavior wins.

Q: Difference between API resource model and domain entity?
A: Resource model is a contract optimized for clients + version stability. Domain entity is internal, focused on invariants and behavior.

Q: When should I define events first?
A: When workflows are unclear or cross multiple stakeholders; events expose business narrative & timing.

Q: How do I identify a Value Object?
A: Equality by attributes, immutable, side-effect free, can be replaced wholesale without identity change.

Q: When to split an aggregate?
A: High contention, unrelated invariants, or need for separate lifecycle/versioning.

Q: Do I need DDD for a CRUD admin tool?
A: Usually no—cost outweighs value; keep it simple.

Q: Is using an ORM equal to having a domain model?
A: No. ORM entities often become anemic bags of state. Behavior + invariants must be explicit.

Q: Safe protobuf evolution rules?
A: Only add optional/`reserved` removed fields; never reuse numbers; avoid changing field meaning.

Q: Can projections become my source of truth?
A: They’re derivative. Keep canonical write model; treat projections as rebuildable caches.

Q: Prevent integration models polluting domain?
A: Use ACLs + translation objects; never pass raw contract DTOs deep into domain.

Q: Should every microservice map to a bounded context?
A: Not strictly. Context defines boundaries; services are a deployment detail.

Q: Is CQRS required for DDD?
A: No—only adopt when read/write requirements diverge materially.

Q: Zero-downtime aggregate schema migration approach?
A: Expand (add nullable/optional), dual write/read adaptation, backfill, contract (remove old).

Q: Domain events vs audit log?
A: Domain events = business facts; audit logs = operational trace/compliance.

Q: When is a rich model overkill?
A: Low complexity, minimal invariants, stable domain. Use straightforward CRUD.

Q: Start contract-first then refine domain later—ok?
A: Yes, but set explicit revisit checkpoints to avoid fossilized abstractions.

Q: How do I know modelling worked?
A: Changes localize; invariants obvious; adding features doesn’t trigger cross-cutting rewrites; fewer semantic bugs.

### 16. Quick Reference Cheat Sheet
DO:
- Isolate bounded contexts
- Use Value Objects for clarity
- Keep aggregates small & cohesive
- Translate at integration boundaries
- Evolve schemas additively first

AVOID:
- Letting persistence/entity framework dictate domain
- Designing only from data access convenience
- Over-segmenting into micro-aggregates with no invariants
- Leaking external contract shapes internally

### 17. Adjacent Topics (Pointers)
- Event Sourcing vs state storage (choose when audit/history first-class).
- CQRS for scaling read shape divergence.
- Hexagonal Architecture for pluggable infrastructure.
- Data Mesh: organizational data ownership—complements bounded contexts.

---
Feedback welcome: tighten, expand examples, or add a deeper dive on any section.

<!-- TODO: Potential future enhancement: add a small code sample folder illustrating the Order aggregate & repository pattern. -->
