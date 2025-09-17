# Project
This is a log of various patterns, techniques gathered for handling distributed services, microservices, SaaS etc.

We will log the tidbits under patterns directory.

---

## 2025-08-19

- Beautified `patterns/fail-fast.md`: normalized headings (H1/H2), fixed list formatting, consolidated duplicate content, and added a concise conclusion/callout. No semantic changes beyond minor clarity edits.
- Added `patterns/SalesForceAndSaaS.md` content: covered SF→SaaS callouts (Apex/Flow/External Services, events), SaaS→SF updates (REST/Composite/Bulk, Platform Events, Apex REST), auth (OAuth), identity/SSO, packaging choices, and ops guardrails.
 - Added Glossary to `patterns/data-migrations.md` with definitions for Up/Down, forward‑fix, EMC, dual‑write, backfill, online DDL, idempotency, tenancy models, and guardrail terms.

## 2025-09-14

- Created comprehensive introduction and foundational sections for `patterns/CRD.md` (purpose, usage criteria, anti-criteria, benefits, trade-offs, reliability considerations, comparison table, example form, design guidelines, anti-patterns, summary). No concrete domain examples included per requirement. Avoided cross-links; kept industry-standard terminology.
- Potential future enhancements (not yet done):
	- Add glossary (Conditions, Finalizer, Conversion Webhook, Informer, Workqueue, Controller Runtime, Operator)
	- Add a section on API evolution workflow (conversion strategy, storage version migration)
	- Add guidance on multi-tenancy & namespace scoping strategies
	- Provide a checklist for launching a new CRD to `beta`/`GA`
	- Discuss observability patterns (metrics naming, event patterns, condition taxonomy)
	- Add references to reliability pattern doc once cross-linking approved.

## 2025-09-15

- Authored full modelling tutorial in `patterns/modelling.md` (previously only a placeholder). Covered: model taxonomy, starting approaches comparison, DDD slice (glossary, aggregates, invariants), mapping to persistence, integration boundaries + ACL, projections, versioning/evolution, heuristics, workflow checklist, example mini domain (Order), tool/artefact selection guide, extensive FAQ (20 Q&A), cheat sheet, adjacent topics pointers.
- Added TODO comment for potential future code sample (Order aggregate implementation) to illustrate theory with a concrete example.
- Intent: Provide pragmatic, non-dogmatic guidance; emphasize separation of model concerns and evolutionary practices.
- Next enhancement ideas:
	- Add a diagram (bounded contexts & flow) — requires decision on diagram tooling (Mermaid vs external).
	- Provide concrete code sample: Value Object (Money), Aggregate (Order), Repository interface, projection builder.
	- Add pitfalls section examples with before/after refactor snippets.
	- Cross-link to `database-migration` patterns for schema evolution details.
	- Possibly integrate a “model review checklist” template as a separate markdown artifact.

	## 2025-09-16

	- Replaced stub in `patterns/aen.md` with comprehensive practical guide on canonical resource identifiers (ARN-like schemes). Focus areas: purpose, design goals, structured vs opaque trade-offs, comparative survey (AWS/GCP/Azure/Stripe/K8s/GitHub), DNS usage rationale & constraints, encoding scheme practical guidance (no math), issuance & lifecycle, policy integration, observability patterns, external ID handling, anti-patterns, pragmatic checklist, glossary, and future TODO placeholders (regex examples, internal bloxid example, code snippets, cross-links pending approval). Intentionally omitted entropy/math per request.


