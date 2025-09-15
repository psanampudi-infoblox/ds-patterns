# SaaS Operational Resilience Certification (SORC) Maturity Model

This document proposes a layered maturity model for evaluating the
**operational resilience of SaaS platforms**.\
It separates **observability (measurable pulse of the system)** from
**operational processes (human-in-the-loop resilience)**, combining both
into a unified certification framework.

------------------------------------------------------------------------

## Level 1 -- Reactive Monitoring (Foundational)

-   **Observability**
    -   Basic uptime checks (ping/HTTP probes).
    -   Ad-hoc dashboards; no consistent metrics across services.
    -   Kafka, DB, and cache metrics not collected systematically.
-   **Operational Process**
    -   Incident response is ad-hoc, dependent on individual heroics.
    -   Status page updates manual, often delayed.

**Characteristics:** Teams rely on alarms when things are already
broken. Outages are long and painful.

------------------------------------------------------------------------

## Level 2 -- Instrumented Components

-   **Observability**
    -   Core infra (Kafka, DB, Redis) emits metrics (heap, lag,
        latency).
    -   Metrics collected but alerting thresholds crude (CPU \> 80%,
        heap \> 90%).
    -   Dependency mapping is partial/manual.
-   **Operational Process**
    -   Incident response playbooks exist but inconsistently followed.
    -   Change rollouts are staged (e.g., 25% → 50%) but not tied to
        infra health.

**Characteristics:** Better visibility, but alerts generate noise.
Failures still cascade.

------------------------------------------------------------------------

## Level 3 -- Service-Centric Observability

-   **Observability**
    -   Per-service SLOs: latency, error rate, Kafka lag, queue depth.
    -   Automated dependency maps (service A depends on Kafka, DB,
        cache).
    -   Synthetic checks for customer-facing workflows (end-to-end
        health).
-   **Operational Process**
    -   Feature rollout gates tied to observability signals (don't scale
        rollout if Kafka lag spikes).
    -   Status page automation: customer-impact metrics auto-publish.
    -   Regular blameless postmortems institutionalized.

**Characteristics:** Incidents are detected earlier, and contained more
effectively.

------------------------------------------------------------------------

## Level 4 -- Predictive Resilience

-   **Observability**
    -   Capacity models based on historical load and stress tests.
    -   Predictive alerts (GC pause trends, producer count anomalies).
    -   Automated anomaly detection for event throughput, backlog
        growth.
-   **Operational Process**
    -   Chaos drills: practice Kafka broker failure, DB failover, cache
        eviction storms.
    -   Cross-team comms liaisons trained and assigned during SEVs.
    -   Feature rollouts integrated with error budgets and SLO burn
        rates.

**Characteristics:** System anticipates stress before failure; humans
drill responses.

------------------------------------------------------------------------

## Level 5 -- Self-Healing & Continuous Adaptation

-   **Observability**
    -   Fully automated SLO tracking and adaptive thresholds.
    -   Self-healing pipelines (auto-provision broker capacity, restart
        unhealthy producers).
    -   Dependency graph integrated with risk scoring.
-   **Operational Process**
    -   Automated incident comms (status page, customer email,
        stakeholder updates).
    -   Continuous game days across teams, refining both observability
        and process.
    -   Postmortem learnings codified into tooling, not just documents.

**Characteristics:** Failures are brief, self-mitigated, and customer
trust is preserved.

------------------------------------------------------------------------

## Takeaway

-   **Observability = the pulse.** Without it, processes lack grounding.
-   **Processes = the care.** Without them, signals don't translate into
    resilience.
-   Together, they form a **SaaS Operational Resilience Certification
    (SORC)** model.

PagerDuty's Kafka outage would likely place them between **Level 2
(instrumented)** and **Level 3 (service-centric)**.\
They had Kafka metrics, but lacked **producer lifecycle visibility +
rollout gates tied to infra health**, which are hallmarks of Level 3+
maturity.

------------------------------------------------------------------------
