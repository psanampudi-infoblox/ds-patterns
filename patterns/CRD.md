## Kubernetes Custom Resource Definitions

### 1. What Is a Custom Resource Definition (CRD)?

A Custom Resource Definition (CRD) extends the Kubernetes API surface by registering a new resource type (a new `Kind` under an API `Group` and version). Once a CRD is installed, Kubernetes API servers begin accepting and storing objects (Custom Resources) of that type in etcd, making them first‑class API objects accessible via the same discovery, authentication, authorization (RBAC), admission control, and audit pipelines as built‑in kinds (e.g., `Deployment`, `ConfigMap`).

Distinction:
- **CRD**: The declarative schema/registration (metadata about the new type).
- **Custom Resource (CR)**: An instance (a persisted object) of that type.
- **Controller / Operator**: A reconciliation loop that observes desired state encoded in CRs and drives actual system state toward it.

### 2. Application Layer Pattern or Operator Modeling Mechanism?

CRDs occupy a spectrum:
- Minimal: a typed configuration artifact preferred over ad hoc YAML or external config stores.
- Rich: a domain‑level abstraction fronted by reconciliation controllers (an *Operator*) that encode operational knowledge (installation, orchestration, failover, scaling, upgrades).

Key points:
- A CRD without a controller is structured, validated, versioned configuration.
- A controller without a CRD manipulates only built‑ins; adding a CRD enables an intent‑centric API.
- An Operator is a controller (or a set) plus operational logic leveraging CRDs as its external contract.

Thus: CRDs are a platform extension primitive. They become an application‑layer abstraction only when you intentionally model domain concepts for users or higher‑level automation to consume.

### 3. When to Use a CRD

Adopt a CRD when most of these hold:
1. **Declarative Desired State**: Stakeholders express *what* they want, not *how*.
2. **Reconciliation Semantics**: Continuous converge loop toward spec-defined state (idempotent, eventually consistent).
3. **Kubernetes-Native Integration**: Need RBAC, audit logs, admission policies, `kubectl` tooling, events, and watch semantics.
4. **Lifecycle Management**: Coherent lifecycle (create/update/delete) with finalizers, garbage collection, ownership semantics.
5. **Higher-Level Abstraction**: Hide orchestration complexity behind a single object representing intent.
6. **Multi-Component Coordination**: Converge many underlying resources (built‑ins, external services) into a consistent state.
7. **Status Feedback**: Structured observed state (conditions, phase, metrics) alongside desired state.
8. **Extensibility & Versioning**: Evolvable APIs (e.g., `v1alpha1` → `v1beta1` → `v1`) with per-version conversion.
9. **Event-Driven Integrations**: Other controllers or systems react via watch streams instead of polling bespoke APIs.
10. **Security Boundaries**: Leverage Kubernetes-native authZ to delegate granular CRUD operations on intent objects.

### 4. When Not to Use a CRD

Prefer other mechanisms when:
- **High-Churn Ephemeral Data**: Extremely frequent, transient updates (per-request metrics, sessions) would overload etcd.
- **Large Payloads / Blobs**: Binary or very large structured payloads (> a few MB) do not belong in etcd.
- **Pure Application-Internal State**: Data not requiring Kubernetes-level visibility/policy/multi-actor interaction.
- **Simple Configuration Suffices**: `ConfigMap`, `Secret`, or Helm values already cover the need.
- **Cross-Cluster Global Control Plane**: Semantics must span many clusters with strong consistency guarantees.
- **One-Off Imperative Actions**: Better expressed as direct API/RPC calls unless wrapped intentionally as job-like CRDs.
- **Latency-Critical Mutation Path**: End-to-end latency cannot tolerate reconciliation delay / eventual consistency window.

### 5. Benefits

- **Unified API Surface**: Uniform discovery, RBAC, tooling, admission, auditing.
- **Declarative Convergence**: Reconciliation loop continuously corrects drift; improves reliability.
- **Separation of Concerns**: Spec = intent; Controller = operational logic.
- **Observability**: Standard `status` + `conditions`; watchable events; introspection via `kubectl`.
- **Evolvable Contracts**: Versioned schemas + conversion webhooks.
- **Composability**: Owner references + label selectors integrate with garbage collection and policy engines.
- **Idempotency & Repeatability**: Declarative resets manual snowflake configuration.
- **Loose Coupling**: Clients depend on API shape, not imperative sequences.

### 6. Trade-Offs and Costs

- **Etcd Resource Pressure**: Objects & updates consume storage, watch bandwidth, compaction cycles.
- **Controller Reliability Critical**: Down/misbehaving controller stalls convergence.
- **Complexity Surface**: API versioning & backward compatibility require discipline.
- **Latency / Eventual Consistency**: Lag between spec change and realized state.
- **Expanded Security Footprint**: Larger RBAC matrix; need least privilege discipline.
- **Operational Burden**: Shipping, upgrading, monitoring Operators adds SLO obligations.
- **Risk of Leaky Abstraction**: Exposing implementation details couples clients.
- **Schema Rigidity vs Flexibility**: Too rigid early hampers iteration; too loose invites misuse.
- **Testing Overhead**: Need integration/E2E + conversion tests.

### 7. Reliability Considerations

CRDs enable reliability patterns; they do not guarantee them:
- **Reconciliation Model**: Idempotent operations + crash recovery (state persisted in etcd).
- **Status as Feedback Loop**: Accurate conditions allow automation to detect/remediate failure states.
- **Finalizers**: Orderly teardown prevents orphaned external resources.
- **Backoff & Rate Limiting**: Avoid hot loops; exponential backoff on requeues.
- **Partial Failure Isolation**: Small, bounded CR scopes reduce blast radius.
- **Declarative Drift Correction**: Periodic requeues heal manual or accidental mutations in subordinate resources.
- **Health Signals**: Conditions (`Ready`, `Progressing`, `Degraded`) provide SLO insights.
- **Version Rollout Safety**: Canary/phased adoption with versioned CRDs & conversion.
- **Idempotent Spec Semantics**: Avoid imperatives encoded as spec field toggles.

### 8. Comparison Cheat-Sheet (Conceptual)

| Need | CRD (+Controller) | ConfigMap | External DB / API |
|------|------------------|-----------|-------------------|
| Declarative Desired State with Convergence | Strong | Weak (consumer logic needed) | Varies |
| RBAC + Audit Integration | Native | Native | Custom work |
| Structured Status / Conditions | Yes | No | Custom |
| Watch Semantics / Event Stream | Native | Native (limited semantics) | Custom |
| High Update Throughput Suitability | Moderate | Moderate | High (specialized stores) |
| Versioned API Evolution | Yes | Manual | Custom |
| Orchestration / Multi-Resource Coordination | Strong (controller logic) | None | Custom |
| Latency for Mutation Realization | Reconcile latency | Low (if directly consumed) | Depends |

### 9. Minimal Example Form (Abstracted)

```yaml
apiVersion: group.example.io/v1alpha1
kind: ExampleKind
metadata:
	name: sample
spec:
	# Declarative desired state fields
	paramA: ...
	paramB: ...
status:
	# Set only by controller; reflects observed state
	phase: ...
	conditions:
		- type: Ready
			status: "True"
			reason: ...
			message: ...
			lastTransitionTime: ...
```

Notes:
- `spec` is user/automation authored; treat fields as a contract (avoid mutable semantics that are really commands).
- `status` is controller-authored; never mutated by end users.
- Validation via OpenAPI schema rejects invalid specs early.
- Conditions should be sparse, stable, semantically meaningful; avoid transient boolean proliferation.

### 10. Design Guidelines

1. Start with `v1alpha1`; evolve deliberately.
2. Keep `spec` declarative; no embedded imperative verbs.
3. Reserve `status` for observed/computed data only.
4. Use stable condition types with consistent reason codes.
5. Keep objects small; externalize large lists/blobs.
6. Prefer additive evolution; deprecate before removal; provide conversions.
7. Reference secrets (do not embed credentials).
8. Ensure deterministic reconciliation; each loop converges or surfaces why not.
9. Guard against hot loops with diff checks & backoff.
10. Instrument metrics, structured logs, tracing.
11. Validate early (schema + admission webhooks for invariants beyond OpenAPI expressiveness).
12. Use owner references for subordinate resources.

### 11. Anti-Patterns

- Using CRDs as a generic key/value store.
- Embedding large opaque blobs directly in `spec`.
- Encoding one-shot imperative actions as spec field toggles.
- Overloading a CR type with many unrelated lifecycle phases.
- Polluting `status` with speculative/future desired values.
- Excessive fine-grained CR types creating fragmentation.
- Ignoring versioning discipline (jumping straight to `v1`).

### 12. Summary

CRDs are a foundational extension mechanism for building declarative, reliable, evolvable control planes on top of Kubernetes. Applied judiciously—with clear separation of intent (`spec`) and observation (`status`), disciplined versioning, and robust controller design—they enable higher-level abstractions, strong operational ergonomics, and consistency with platform governance (RBAC, audit, policy). Misused, they introduce unnecessary complexity, etcd load, and fragile APIs. The key is aligning the abstraction with a genuine desired-state domain and committing to its lifecycle.

