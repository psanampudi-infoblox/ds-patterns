# Salesforce ↔ SaaS: Integration Patterns

This note outlines how SaaS products typically integrate with Salesforce in both directions: what Salesforce calls into a SaaS system, and what a SaaS must send back to Salesforce. It also highlights common auth, APIs, and reliability considerations.

---

## 1) What calls are made by Salesforce to a SaaS?

Two broad categories: synchronous callouts (Salesforce actively calls your SaaS endpoint) and event delivery (Salesforce emits events that your SaaS consumes).

### A. Synchronous callouts from Salesforce

- Apex HTTP callouts (REST/GraphQL/SOAP)
	- Use Named Credentials for auth and endpoint management.
	- Triggered from Apex (Queueable/Future/Batch/Scheduled) or from Triggers/Flows.
	- Limits: governor limits apply (e.g., up to ~100 callouts per transaction; request/response size and timeout caps).
- Flow HTTP Callouts
	- Point-and-click HTTP actions in Flow (great for simple integrations).
	- Use Named Credentials; supports request/response schema.
- External Services (OpenAPI/Swagger)
	- Register your SaaS OpenAPI spec; Salesforce auto-generates invocable actions for Flow/Apex.
- SOAP callouts (WSDL2Apex)
	- Generate strongly-typed Apex from your WSDL when legacy SOAP is required.
- Outbound Messages (legacy)
	- Workflow/Process Builder can POST XML to your SaaS endpoint. Simple but limited and aging.
- Middleware callouts
	- Salesforce → iPaaS (e.g., MuleSoft) → SaaS, to offload orchestration/retries/mapping.

Key design points
- Idempotent SaaS endpoints (dedupe via idempotency keys/request IDs).
- Timeouts/retries/backoff; surface SaveResult-style partial failures.
- Respect per-transaction limits; use async patterns (Platform Events, Queueable) to avoid blocking user transactions.

### B. Events your SaaS can consume from Salesforce

- Platform Events (pub/sub)
	- Your SaaS subscribes via Pub/Sub API or CometD. Durable delivery with replay IDs.
- Change Data Capture (CDC)
	- Row-change events for standard/custom objects; subscribe as above.
- Streaming API (PushTopic/Generic)
	- Older pattern for change notifications; Platform Events/CDC preferred.

Notes
- These aren’t direct HTTP calls into your SaaS; instead, your SaaS maintains a subscriber client and pulls events from Salesforce’s event bus.
- Ensure consumer durability, replay handling, and at-least-once processing.

---

## 2) What updates/notifications must a SaaS initiate towards Salesforce?

This is inbound to Salesforce from your SaaS, typically for data sync, lifecycle events, and user/workflow updates.

### A. Salesforce APIs your SaaS can call

- REST API (sObjects, Query, Search)
	- Create/Update/Upsert/Delete records; use External IDs for upsert and dedupe.
- Composite/Batch/Graph API
	- Fewer round-trips; atomic or best-effort batches; Graph for dependent calls.
- Bulk API v2
	- High-volume async ingest/export; use for nightly syncs and large backfills.
- Pub/Sub API (publish)
	- Publish Platform Events from your SaaS to trigger Salesforce flows/processes.
- Apex REST endpoints (custom controllers)
	- Your Salesforce org exposes a custom REST surface; SaaS POSTs to it for domain logic.

Optional patterns
- Salesforce Connect / External Objects
	- Expose SaaS data via OData or a custom adapter; near real-time reads/writes from Salesforce without copying data.

### B. Authentication from SaaS → Salesforce

- OAuth 2.0 JWT Bearer (server-to-server, recommended)
	- No interactive user step; ideal for backend integrations.
- OAuth 2.0 Web Server flow (user-context)
	- When acting on behalf of a user; requires browser-based consent.
- Username-Password OAuth (discouraged)
	- Use only when others are not possible; rotate credentials and restrict permissions tightly.

Best practices
- Use a dedicated Integration User with least-privilege Profile/Permission Set.
- Store/rotate secrets; enforce IP allowlists where appropriate.
- Honor FLS/CRUD by using the Integration User’s profile as your contract.

### C. What should the SaaS send?

- Data syncs
	- Upsert core objects (Account/Contact/Opportunity/Case/Custom) using External IDs.
	- Maintain reference maps between SaaS IDs and Salesforce IDs.
- Lifecycle notifications
	- Publish Platform Events for important domain transitions (provisioned, deprovisioned, usage spikes, SLA breaches).
	- Optionally call Apex REST endpoints for transactional business rules.
- User/Access updates
	- JIT user provisioning via SCIM (if present) or via REST upserts to User/Permission assignments (with caution).

Reliability
- Implement retries with backoff; handle 429 and concurrency errors.
- Ensure idempotency (External ID upserts, natural keys, or idempotency headers).
- Monitor API limits and event publish failures; alert before hard stops.

---

## 3) Identity and SSO patterns

- SSO: SAML or OpenID Connect to allow users to sign in to your SaaS from Salesforce (or vice versa).
- Provisioning: SCIM 2.0 where available; otherwise API-driven provisioning flows.
- Salesforce Canvas: Embed your SaaS UI inside Salesforce with signed requests and SSO.

---

## 4) Packaging vs. headless integration

- AppExchange managed package
	- Apex classes, Lightning components, custom objects, and setup UI. Best when you need tight in-org experiences.
- Headless/API-only
	- Pure API/event integration with minimal in-org footprint; faster to deploy, fewer upgrade constraints.

Choose based on required UX inside Salesforce, admin experience, and upgrade lifecycle.

---

## 5) Guardrails and ops checklist

- Data modeling: define master for each field, conflict resolution, and merge rules.
- Security: least privilege, audit logs, shield/encryption requirements.
- Limits: governor limits, API rate limits, payload sizes; design for backpressure.
- Observability: trace IDs across SaaS↔SF, dashboards for failures, dead-letter handling.
- Testing: use sandboxes/scratch orgs; seed data; contract tests for APIs and events.

---

## Quick mapping back to your prompts

- What calls are made by SF side to SaaS: Apex/Flow/External Services callouts; events via Platform Events/CDC; optional Outbound Messages.
- What updates/notifications does SaaS initiate towards SF: REST/Composite/Bulk API upserts, publish Platform Events, or call Apex REST endpoints; plus identity/SSO signals.

