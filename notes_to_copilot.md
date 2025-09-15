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
