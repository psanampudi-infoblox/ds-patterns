# On-Behalf-Of (OBO) Token Minting in Multi‑Service SaaS Platforms  
## A Comprehensive Overview and How the Infoblox Implementation Compares

## 1. Executive Summary
On-Behalf-Of (OBO) token minting is a controlled delegation mechanism used in distributed SaaS platforms to allow one trusted actor (service, integration, edge component, host agent) to obtain a new token that represents (fully or partially) another principal (human user, system user, host, external entity) for a specific downstream call. It solves the problem of propagating authenticated identity context across service boundaries without over-exposing original credentials. The Infoblox endpoint `/v2/session/svc/obo_mint` implements a practical variant of this model tailored to multi-tenant identity constructs (Accounts, CSP IDs, Salesforce Account IDs) and multiple impersonation origins (envoy, user, host).

---

## 2. Terminology
- Principal: Entity performing an action (user, service, host agent).
- Delegation: Authorized act of allowing one principal to act for another.
- Impersonation vs Delegation:
  - Impersonation: Act fully as the target (often indistinguishable).
  - Delegation: Act with a constrained subset of the target’s authority.
- OBO Token: A freshly minted token whose subject or effective subject represents a target identity, derived from an existing authenticated caller context.
- IdP: Identity Provider (internal or external).
- Tenant / Account: Logical customer or organizational boundary.
- CSP ID / SFDC Account ID: Additional Infoblox tenant identifiers (Cloud Service Provider affinity / Salesforce correlation).

---

## 3. Problem Space in Multi‑Service SaaS
In a mature SaaS:
- Numerous microservices call each other.
- Edge/API gateways terminate user/API keys and forward enriched context.
- 3rd-party integrations (e.g., partner systems) trigger workflows.
- On‑prem or host agents phone home and must perform limited actions (telemetry, config pull, key rotation).
- External SaaS APIs may need to call back into the platform with delegated authority.

Challenges addressed by OBO:
1. Propagation of identity without forwarding original long-lived credentials.
2. Least privilege scoping for downstream calls.
3. Auditability: Clear chain of “A acted on behalf of B”.
4. Multi-tenant isolation: Ensure correct tenant/account binding.
5. Compliance: Avoid sharing end-user secrets with internal subsystems.

---

## 4. Core Concept of OBO
A trusted caller presents:
- Its own authenticated token (or API key / session context).
- A request specifying a target subject (e.g., user email, host identifier).
The OBO service:
1. Validates caller’s eligibility to mint.
2. Resolves target identity & tenant.
3. Applies policy (allowed target? allowed scope? TTL cap?).
4. Mints a short‑lived, scoped token representing “caller acting for target” or “target (delegated)”.
5. Emits audit trail (who, for whom, why, when).

---

## 5. Industry Standard Patterns
| Pattern | Standards / Ecosystem | Description |
|--------|-----------------------|-------------|
| OAuth 2.0 OBO (a.k.a. token exchange subset) | RFC 6749 (base), RFC 8693 (Token Exchange) | Client exchanges a user token for a token usable to call a downstream API. |
| Azure AD OBO Flow | Microsoft Identity Platform | API A receives user token, exchanges for new token to call API B. |
| STS Token Federation | AWS STS AssumeRole / AssumeRoleWithWebIdentity | Short-lived credentials representing delegated authority. |
| Google Service Account Domain-Wide Delegation | Google Workspace | Service impersonates user with pre-authorized scopes. |
| SPIFFE/SPIRE + Impersonation Gateways | Cloud-native zero-trust | Service identity asserts limited act-for semantics via workload identities. |

Key shared themes: explicit token exchange, policy gating, scope reduction, short TTL, auditability.

---

## 6. Typical Scope Boundaries
An OBO framework usually defines:
- Who can mint (service accounts with a specific role).
- Who can be targeted (subset of users, hosts belonging to same tenant).
- What scopes/claims can be carried forward (reduced vs original token).
- Max TTL (e.g., 5–15 min).
- Replay & revocation considerations (nonce / jti / introspection).
- Distinction between “full impersonation” vs “delegated acting-for” (claim layering like `sub` vs `act` (RFC 8693), or custom `obo_origin`).

---

## 7. Threat Model & Security Considerations
Threats:
- Elevation: Caller mints token for a privileged admin user.
- Lateral movement across tenants (improper account resolution).
- Token replay (long TTL or no jti uniqueness).
- Shadow identity (minting for non-existent user to confuse audit).
- Insufficient audit (no link to original caller).
Mitigations:
- Explicit allowlists/role checks.
- Enforce target resolution within same tenant boundary.
- TTL cap + jti uniqueness + signing key rotation.
- Deny minting for “not-found” identities (unless purposeful “prospective identity” use case).
- Structured audit log linking `requester_id`, `target_id`, `obo_type`, correlation ID.

---

## 8. Typical Implementation Architecture
1. Entry Endpoint: `/token/obo` (HTTP/GRPC).
2. Authentication Layer: Validates caller’s token.
3. Policy Engine: Evaluates (caller_role, target_identity, tenant match, scopes).
4. Identity Resolution: Lookup user/host/service in directory/DB.
5. Claim Construction:
   - Standard: `iss`, `aud`, `iat`, `exp`, `sub` (target or composite)
   - Delegation chain (e.g., `act` claim per RFC 8693) or custom `obo_chain`.
   - Tenant claims (e.g., `acct_id`, `acct_csp_id`, `acct_sfdc_id`).
6. Signing: JWT / opaque reference token (with introspection).
7. Audit Log + Metrics.
8. Response: Token + metadata (expiry, subject, limited scopes).

Sequence (textual):
Caller Token -> OBO Service: (target descriptor) -> Validate -> Resolve -> Policy -> Mint -> Return shortened JWT.

---

## 9. Infoblox Implementation (Observed)
Endpoint: `/v2/session/svc/obo_mint` (server-side Sessions service).

Helper functions (from `sessions_obo_mint.go`):
- `oboMintTypeEnvoy`
- `oboMintTypeUser`
- `oboMintTypeHost`
(Primary handler `SessionOBOMint` not shown here, but implied.)

### 9.1 Supported id_types (inferred)
- envoy: `email.<address>`
- user: `email.<address>` (+ optional `account_id` or `account_csp_id` disambiguation)
- host: `ophid.<opaque_onprem_host_id>` OR `host_id.<uuid>`

### 9.2 Identity Resolution Logic Highlights
- Envoy:
  - If email user not found, constructs a sentinel `UserORM{Id:"not-found", Email:<email>}` and still proceeds after also resolving account by Salesforce Account ID (SFDC).
  - Returns NOT_FOUND if account absent.
- User:
  - Requires resolving user by email. If user belongs to multiple accounts, requires explicit `account_id` or `account_csp_id`.
  - Enforces matching of provided account context with user’s memberships.
- Host:
  - Resolves an on-prem host either by internal ID or OPHID via external CSP API helpers (`cspGetOnpremHost`, `getCspHostByField`).
  - Maps host API key back to a User (host-associated user) and then obtains primary account.

### 9.3 Tenant Correlation
- Multiple identifiers: internal `account_id`, `csp_id` (int), `sfdc_account_id` (string). This is richer than many generic systems which usually rely on a single tenant claim.

### 9.4 Notable Special Handling
- Sentinel “not-found” user for envoy flows diverges from stricter industry practice (most systems would fail early). Possible rationale:
  - Allow minting a token for a provisional identity (e.g., pre-registration flows, analytics tagging, inbound edges).
  - Risk: Downstream services must treat “not-found” carefully to avoid privilege leakage.
- Flexible account disambiguation (ID or CSP ID) supports transitional or multi-source-account identity domains.

### 9.5 Likely Claim Model (Inferred)
Not shown, but typical for this pattern:
- `sub`: either the target user or a host-labeled principal.
- Additional claims: `account_id`, `csp_id`, `sfdc_account_id`, `obo_type`, maybe `origin`.
- Potential need for chain claim (e.g., `act` or `obo_parent_sub`)—unclear if implemented.

### 9.6 Strengths
- Clear separation of logic per target type (envoy/user/host) aids auditing and future policy branching.
- Strict validation of email and presence of host IDs.
- Disambiguation enforcement for multi-account users reduces accidental cross-tenant delegation.

### 9.7 Potential Gaps (From Provided Snippet)
- No visible explicit max TTL enforcement in snippet (may exist elsewhere).
- No explicit audit emission shown (logger usage only).
- “not-found” user path may create ambiguous audit semantics unless downstream sanitized.
- No visible scope reduction logic (e.g., filtering roles / permissions) in the helper layer.

---

## 10. Comparison Table

| Dimension | Industry Ideal | Infoblox Observed Behavior |
|-----------|----------------|----------------------------|
| Delegation Standard | RFC 8693 style token exchange (optional) | Custom GRPC/HTTP endpoint `/session/svc/obo_mint` |
| Chain Attribution | `act` claim or similar | Not visible (unknown) |
| Tenant Identification | Single tenant claim | Multiple: account_id, csp_id, sfdc_account_id |
| Target Resolution Failure | Hard fail | Envoy path allows sentinel user |
| Multi-Account Users | Must specify context | Enforced when >1 accounts |
| Host Delegation | Sometimes separate device identity | Host -> API key -> user -> account |
| Scope Narrowing | Common (aud/scope shrink) | Not visible in snippet |
| Audit | Structured event bus/log | Logger only (snippet) |
| TTL Cap | Short-lived (5–15 min) | Not visible (assumed in main handler) |
| Policy Engine | Central PDP / ABAC | Embedded logic in service layer |

---

## 11. Deviations / Special Takes
1. Sentinel “not-found” user for envoy: Uncommon; treat with care.
2. Triple tenant identity (account_id / csp_id / sfdc_account_id) affords flexibility but increases complexity for policy & drift.
3. Host path ties hardware/agent identity back to a user via API key—some ecosystems treat device identity as 1st-class principal with its own claim set; here it's indirectly coupled.
4. Lack (in snippet) of explicit “delegation chain” claims may reduce downstream ability to differentiate original vs effective principal unless injected elsewhere.
5. Potential absence of explicit scope claims (roles, entitlements) in OBO minting path (not shown) may rely on downstream lookups—can increase latency and complexity.

---

## 12. Recommendations / Potential Enhancements
| Area | Recommendation | Rationale |
|------|----------------|-----------|
| Chain Attribution | Add `act` (actor) claim per RFC 8693 or custom `obo_actor` | Clear auditing & security analytics. |
| Sentinel User | Return explicit error or mark claim `provisional=true` | Prevent misuse or confusion. |
| Policy Centralization | Externalize rules (OPA, Cedar, Rego, or internal PDP) | Future agility & consistency. |
| Scope Minimization | Recompute / project minimal entitlements for OBO tokens | Principle of least privilege. |
| TTL & Constraints | Enforce and surface `max_ttl`, reject longer | Limits replay window. |
| Audit Events | Emit structured event (JSON) with correlation ID | Compliance & incident response. |
| Revocation Strategy | Maintain short-lived + jti in a bloom/cache for early revocation windows | Further risk reduction. |
| Host Identity | Consider first-class “host” principal claim separate from user | Better attribution surfaces. |
| Multi-Tenant Consistency | Normalize to one canonical tenant claim + alias mapping | Simplifies policy evaluation. |
| Security Tests | Add fuzz & negative OBO tests (multi-account mismatch, sentinel user) | Hardens boundary conditions. |

---

## 13. Textual Sequence Examples

### 13.1 User OBO Flow
1. Service A (trusted) holds internal service token.
2. Calls `/v2/session/svc/obo_mint` with: `id_type=user`, `id=email.alice@example.com`, optional `account_id`.
3. Service validates caller’s permission -> resolves user -> resolves account -> builds claims -> signs OBO token.
4. Returns OBO token to Service A.
5. Service A calls Service B with new token; B treats actions as Alice (delegated).

### 13.2 Host OBO Flow
1. Host agent identified via OPHID.
2. Backend resolves host -> finds associated API key -> maps to host user -> resolves primary account.
3. Mints host-scope OBO token (short TTL).
4. Host uses OBO token for restricted configuration pull.

### 13.3 Envoy Flow (Edge Email)
1. Edge component needs to perform a pre-user-binding operation (e.g., pre-provision or invite).
2. Envoy sends `email.<prospective_user>`; user absent.
3. System (currently) produces token with sentinel user object.
4. Downstream service must interpret sentinel semantics properly.

---

## 14. References
- RFC 6749: OAuth 2.0 Authorization Framework
- RFC 8693: OAuth 2.0 Token Exchange
- Azure AD On-Behalf-Of Flow Documentation
- AWS STS AssumeRole / Web Identity Federation
- Google Workspace Domain-Wide Delegation
- NIST SP 800-204 (Microservices Security Considerations)
- SPIFFE/SPIRE Identity Specifications

---

## 15. Summary
Infoblox’s `/v2/session/svc/obo_mint` embodies a pragmatic OBO pattern adapted to rich tenant identity constructs and multiple origin types. Core alignment with industry concepts (identity resolution, delegation, new token minting) is present. Main divergences are the provisional (not-found) user handling and (from the visible snippet) lack of explicit chain/scope claims. Tightening policy centralization, chain attribution, scope minimization, and audit rigor would further align the implementation with leading zero-trust and regulated SaaS practices.

---

## 16. Fast Checklist (Operational)
- Enforce max TTL? (Verify)
- Add actor/chain claim
- Evaluate sentinel user necessity
- Centralize policy decisions
- Emit structured audit events
- Introduce revocation hints (jti)
- Publish contract: accepted id_type formats and error taxonomy

---

Prepared for: Internal engineering & architecture review  
Document purpose: Align implementation with industry best practices while highlighting unique tenancy and identity resolution considerations.