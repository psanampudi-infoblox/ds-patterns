# Asset Ingestion via Kafka → ClickHouse

This document describes how to ingest asset information from multiple
providers and asset types into ClickHouse via Kafka. It covers **Kafka
topic design**, **message schema**, **ClickHouse ingestion patterns**,
and **operational guardrails**.

------------------------------------------------------------------------

## 1. Kafka Topic Strategy

### 1.1 Topic Taxonomy

-   Use namespace by domain:\
    `assets.<provider>.<asset_type>`\
    Examples:
    -   `assets.aws.ec2`\
    -   `assets.aws.s3`\
    -   `assets.gcp.gce`\
    -   `assets.okta.user`

### 1.2 Topic Count

-   **100--300 topics**: reasonable and operationally safe.\
-   Thousands: technically possible, but increases metadata overhead.\
-   Keep **partitions per topic small** (3--6 by default). Only scale
    partitions if throughput demands it.

### 1.3 Keys & Headers

-   **Message key**:

        tenant_id|provider|asset_id

-   **Headers**:

    -   `asset_type`
    -   `provider`
    -   `schema_version`
    -   `op` (`upsert|delete|snapshot`)
    -   `ingest_ts`

### 1.4 Payload

-   Prefer **JSONEachRow** (ClickHouse-friendly).\
-   Include:
    -   `discovery_time` (logical timestamp)
    -   `source_fingerprint` (for idempotence)

### 1.5 Retention & Compaction

-   Two lanes per asset type:
    1.  **Raw lane** (`assets.raw.<provider>.<asset_type>`)
        -   `cleanup.policy=delete`\
        -   Retention: 7--30 days\
        -   Use for replay/debug
    2.  **Current state lane** (`assets.cs.<provider>.<asset_type>`)
        -   `cleanup.policy=compact`\
        -   Keyed by asset ID\
        -   Stores latest asset snapshot

### 1.6 DLQ & Quarantine

-   Global DLQ: `assets.dlq` (invalid JSON, parsing errors).\
-   Quarantine: `assets.quarantine` (valid JSON, failed validation).

------------------------------------------------------------------------

## 2. ClickHouse Ingestion Design

### 2.1 Options for Kafka → ClickHouse

#### Option A: One Kafka Table per Topic

-   Direct mapping: `assets.aws.ec2` → `kafka_assets_aws_ec2`
-   Simple but creates **many tables + MVs**.

Example:

``` sql
CREATE TABLE kafka_assets_aws_ec2 (
  tenant_id String,
  provider String,
  asset_type String,
  asset_id String,
  discovery_time DateTime64(3),
  payload JSON,
  op LowCardinality(String),
  schema_version UInt16
) ENGINE = Kafka
SETTINGS
  kafka_brokers = 'kafka:9092',
  kafka_topic_list = 'assets.aws.ec2',
  kafka_group_name = 'ch.assets',
  kafka_format = 'JSONEachRow',
  kafka_num_consumers = 1;

CREATE TABLE assets_aws_ec2
ENGINE = MergeTree()
ORDER BY (tenant_id, asset_type, asset_id);

CREATE MATERIALIZED VIEW mv_assets_aws_ec2
TO assets_aws_ec2
AS SELECT * FROM kafka_assets_aws_ec2;
```

#### Option B: One Kafka Table with Topic Pattern

-   Single Kafka engine table reads **all asset topics**.\
-   Route data using `_topic` column.\
-   Great for **hundreds of asset types**.

Example:

``` sql
CREATE TABLE kafka_assets_all (
  tenant_id String,
  provider String,
  asset_type String,
  asset_id String,
  discovery_time DateTime64(3),
  payload JSON,
  op LowCardinality(String),
  schema_version UInt16,
  _topic String   -- virtual column
) ENGINE = Kafka
SETTINGS
  kafka_brokers = 'kafka:9092',
  kafka_topics_pattern = 'assets\\.(raw|cs)\\..*',
  kafka_group_name = 'ch.assets',
  kafka_format = 'JSONEachRow',
  kafka_num_consumers = 4;

CREATE TABLE assets_current
ENGINE = ReplacingMergeTree(discovery_time)
ORDER BY (tenant_id, provider, asset_type, asset_id);

CREATE MATERIALIZED VIEW mv_assets_current TO assets_current AS
SELECT * FROM kafka_assets_all
WHERE startsWith(_topic, 'assets.cs.');

CREATE TABLE assets_raw
ENGINE = MergeTree()
ORDER BY (tenant_id, provider, asset_type, asset_id, discovery_time);

CREATE MATERIALIZED VIEW mv_assets_raw TO assets_raw AS
SELECT * FROM kafka_assets_all
WHERE startsWith(_topic, 'assets.raw.');
```

#### Option C: Staging JSON → Typed Tables

-   Land everything in a **wide staging table** with JSON column.\
-   Periodically transform → typed tables.\
-   Best for schema churn and backfills.

------------------------------------------------------------------------

## 3. Table & Materialized View Limits in ClickHouse

-   **No hard limit** on number of tables/MVs.\
-   Practical guidance:
    -   **Hundreds** → fine.\
    -   **Thousands** → possible, but watch memory & startup time.\
    -   **Tens of thousands** → avoid (operational pain).\
-   Each table = background merges, parts, metadata in memory.\
-   Too many MVs on a single Kafka table = duplicated parsing & CPU
    load.\
-   Prefer **few Kafka tables + routing MVs** (Option B) to control
    sprawl.

------------------------------------------------------------------------

## 4. Operational Guardrails

-   **Kafka**
    -   Keep total partitions \<50k unless cluster is tuned.
    -   Monitor controller CPU/memory.
    -   Track consumer lag & DLQ volume.
-   **ClickHouse**
    -   Monitor `system.parts`, `system.merges`.
    -   Use `ReplacingMergeTree` for latest asset state.
    -   Keep raw lane (30--90d) for backfills.
    -   For bulk replays, prefer S3 → ClickHouse directly.

------------------------------------------------------------------------

## 5. Decision Framework

### Create a **new Kafka topic** when:

-   Different retention/cleanup policy is needed.\
-   Different SLA/throughput.\
-   Different access control boundary.\
-   Operational isolation required.

### Create a **new ClickHouse table/MV** when:

-   Different storage engine/merge policy required.\
-   Query patterns differ (raw history vs current state).\
-   Data volume is high enough that combining hurts performance.

Otherwise, **reuse existing tables + add filters**.

------------------------------------------------------------------------

## 6. Summary

-   **Kafka**: hundreds of topics fine; use lanes (`raw`, `cs`) and keep
    partitions modest.\
-   **ClickHouse**: avoid table/MV explosion; prefer one Kafka table
    with topic patterns and routing MVs.\
-   **Always keep a raw lane** for replays and debugging.\
-   **Use compacted lanes** for "latest truth" feeding ClickHouse
    current-state tables.

This balances **operational simplicity** with **data modeling clarity**.

------------------------------------------------------------------------

📌 *This doc is intended as a baseline design for the asset ingestion
pipeline. Teams should adapt retention, partitioning, and table layout
based on provider volume and SLA requirements.*
