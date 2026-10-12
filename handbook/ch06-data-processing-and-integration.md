# Chapter 06 — Data Processing, Analytics and Integration

## Exam objectives covered

From the official exam guide (§1.1, §1.3):
- Data processing solutions and integration patterns with external systems
- Movement and integration of data; messaging; streaming vs. batch
- BigQuery (analytics, querying, partitioning, cost controls, ML)
- Cloud Pub/Sub (messaging)
- Apache Beam and Dataflow (streaming/batch processing)
- Data orchestration (Cloud Composer, Workflows)
- Event-driven architecture (Eventarc, Application Integration)
- Data integration solutions (Datastream, Data Fusion)
- Legacy ETL and SFTP integration patterns
- Data lake/warehouse/lakehouse reference architectures
- Feeding AI/ML with BigQuery data, vector search, RAG preparation

## What the exam really tests

- **Messaging vs. eventing:** when Pub/Sub (async many-to-many messaging, at-least-once by default, optional exactly-once on regional pull subscriptions, up to 31-day topic retention) vs. Eventarc (event routing, GCP-native sources) vs. Cloud Tasks (explicit task queues, rate limiting, delayed/deferred HTTP tasks with retries) vs. Workflows (serverless orchestration of API calls, long-running, supports callbacks for human handoff)
- **Processing engines:** Dataflow (horizontal scaling, Beam, great for stream) vs. Dataproc (now branded Managed Service for Apache Spark; Spark/Hadoop, ephemeral clusters or serverless batches) vs. BigQuery (SQL + ML + storage in one; serverless; pay-per-query) vs. Data Fusion (low-code/no-code pipelines)
- **Publish latency trade-off:** lower latency requires disabling/reducing client-side batching (a known trap: candidates think it's a server setting)
- **BigQuery cost:** partitioning/clustering reduce scan costs; slots vs. on-demand; BI Engine caching; authorized views for shared data
- **Batch vs. streaming:** most analytics workloads are fine as batch; streaming adds cost and operational complexity—choose it only when low-latency (seconds or less) insights are a stated requirement
- **Hybrid/legacy integration:** ETL with SFTP, Cymbal/EHR case study patterns, Data Fusion for no-code, Application Integration for orchestrating legacy APIs
- **Data architecture:** lake (raw storage, schema-on-read, low cost) vs. warehouse (curated, schema-on-write, structured) vs. lakehouse (open table formats such as Apache Iceberg on object storage, queried by BigQuery/Spark; Google now brands BigLake as Google Cloud Lakehouse)
- **Ordering keys:** Pub/Sub with ordering keys delivers messages with the same key in order (publish to one region, enable ordering on the subscription)—use only when order matters, it limits per-key throughput
- **Dead-letter topics:** where messages go after max delivery attempts (5–100); configure a retry policy (exponential backoff) and alert on the dead-letter topic's backlog
- **Exactly-once semantics:** Pub/Sub exactly-once delivery is optional, pull-only and regional; Dataflow gives exactly-once processing inside the pipeline but sinks must be idempotent/transactional for end-to-end guarantees

## Core concepts

### Pub/Sub: Core messaging

**What it is:** Fully managed, globally distributed async messaging service (topic + subscription model). Publisher sends message → topic → multiple subscriptions (pull or push). Unacknowledged messages are kept up to 7 days per subscription (default 7); optional topic message retention up to 31 days enables replay. Message size ≤ 10 MB. Global scale, no provisioning. (Pub/Sub Lite is deprecated and was turned down on 18 March 2026; use Pub/Sub or Managed Service for Apache Kafka.)

**Key configurations:**
- **Push vs. pull:** Push = Pub/Sub delivers to an HTTPS endpoint (Cloud Run, Cloud Run functions, webhook, App Engine) and scales with the endpoint. Pull (incl. StreamingPull) = subscriber requests messages (GKE, Compute Engine, Dataflow); gives flow control and is needed for high throughput and for exactly-once delivery. Both tolerate subscriber downtime (messages are retained until acked or retention expires).
- **Batching & latency:** Publisher client libraries batch messages (count/bytes/delay thresholds, e.g. max latency or delay threshold) to raise throughput; batching adds publish latency because messages wait for the batch. Lower publish latency → reduce the batch delay/size thresholds or disable batching (a client-side setting, not a server setting).
- **Exactly-once delivery:** Optional per-subscription feature, only for **pull** subscriptions (not push), and only effective when subscribers connect to the same region (use a locational endpoint). Default delivery is at-least-once, so make consumers idempotent (e.g., upsert keyed by message ID) for end-to-end guarantees.
- **Ordering keys:** Optional ordering key on messages; with message ordering enabled on the subscription, messages with the same key published in the same region are delivered in order. Reduces throughput per key and a failed message can block that key; use only when order is critical (e.g., per-user account updates).
- **Ack deadline:** Subscriber must ack/nack within the deadline (default 10 s, configurable 10–600 s). No ack = message re-delivered.
- **Dead-letter topic:** Messages that exceed the subscription's max delivery attempts (configurable 5–100) are forwarded to a dead-letter topic (attach a subscription to inspect them); pair with a retry policy (exponential backoff). Monitor backlog on the dead-letter topic to catch processing failures.
- **BigQuery/Cloud Storage subscriptions (Pub/Sub export):** Direct export to BigQuery or Cloud Storage without subscriber code. Ideal for analytics pipeline.
- **Retention & replay:** Enable topic retention (up to 31 days) and/or retain acked messages on a subscription; subscribers can then seek to a timestamp or snapshot to replay for reprocessing.

**When to choose:** Async decoupling; fire-and-forget; high throughput; global scale; low ops overhead; cross-service communication (GKE → Cloud Run functions → BigQuery). NOT for: synchronous request-response (use REST or gRPC); ordering across all messages (ordering is per key only); Kafka-protocol compatibility or Kafka ecosystem needs (use Managed Service for Apache Kafka); a long-term event store (export to BigQuery/Cloud Storage).

**Pub/Sub vs. alternatives:**
| Aspect | Pub/Sub | Kafka/Managed Kafka | Cloud Tasks | Eventarc |
|---|---|---|---|---|
| **Use case** | Async, decoupled, high-volume fan-out | Kafka-compatible apps, log-like streaming, replay | Explicit task queue: delayed/rate-limited HTTP tasks with retries | GCP-native events, webhook routing |
| **Ordering** | Per-key optional | Per-partition strict | None guaranteed | N/A |
| **Retention** | Up to 7 days per subscription; topic retention up to 31 days | Configurable (weeks to months) | Until task done/expired (schedule up to 30 days ahead) | Event delivered via Pub/Sub channel (retry for 24 h by default ⚠ verify) |
| **Delivery** | At-least-once; optional exactly-once (pull, regional) | At-least-once; idempotent producers/transactions | At-least-once (named tasks dedupe) | At-least-once |
| **Latency** | Typically sub-second | Low (ms) | Dispatch-rate controlled | Sub-second to seconds |
| **Ops overhead** | Minimal (serverless) | High (cluster mgmt) | None (serverless) | None (serverless) |
| **Cost** | Pay per msg + storage | Compute + storage | Per task + execution | Per event routed |

### Dataflow: Streaming & batch with Apache Beam

**What it is:** Fully managed service for executing Apache Beam pipelines (unified API for streaming and batch). Automatically scales compute; handles late data, windowing, triggers. Use Beam directly or Dataflow templates (pre-built, parameterized pipelines). Billed per second for vCPU, memory, storage and data processed (Streaming Engine/Shuffle).

**Key concepts:**
- **Beam model:** PCollection (parallel dataset) → ParDo (element-wise transform) → Sink (write). Streaming = unbounded PCollection; batch = bounded. Supports branching (multiple outputs per transform).
- **Windowing & triggers:** Tumbling (fixed time), sliding (overlapping), session (event-driven). Triggers decide when to emit results (on-time, early, late). Handles out-of-order / late-arriving data (replay within allowed lateness window).
- **Autoscaling:** Horizontal (more workers) by default. Backlog/CPU utilization drives scaling. Set max workers (and optionally min); choose machine type. Shuffle (group-by/join) may bottleneck—use Dataflow Shuffle (batch) / Streaming Engine to offload it.
- **Templates:** Parameterized Beam jobs staged for reuse: run without a dev environment or recompiling. Types: Classic and Flex (Flex templates package the pipeline as a container image and are the recommended type); both support batch and streaming; Google also provides ready-made templates (e.g., Pub/Sub to BigQuery). Can be triggered from Cloud Scheduler or Composer.
- **Exactly-once:** Dataflow streaming processes each record effectively once by default (exactly-once mode; at-least-once mode is an optional cheaper setting). End-to-end exactly-once also needs sinks that are idempotent or transactional (e.g., upsert or BigQuery Storage Write API exactly-once, not blind append).
- **Streaming vs. batch:** Streaming = continuously running, low latency, higher cost and operational complexity. Batch = bounded jobs, cheaper, higher latency. Choose streaming for real-time alerts, live dashboards, continuous ingestion; otherwise batch is usually enough.

**When to choose:** ETL at scale; complex transformations (joins, aggregations, windowing); streaming anomaly detection; ML feature engineering; media/gaming analytics. NOT for: simple SQL queries (BigQuery), no-code pipelines (Data Fusion), existing Spark/Hadoop code or Spark ML (Dataproc).

**Integration & cost:** Read from Pub/Sub, Cloud Storage, Bigtable, BigQuery, Kafka; write to BigQuery, Pub/Sub, Datastore, Bigtable. Chain with other services via APIs. Cost optimization: use batch for off-peak, Spot VMs (FlexRS for batch) for non-critical jobs, filter early.

### Dataproc: Managed Spark & Hadoop

**What it is:** Dataproc (Google now brands it Managed Service for Apache Spark) is a fully managed service for Apache Spark, Hadoop, Hive, Flink, Presto/Trino clusters. Ephemeral clusters (billed per second, 1-minute minimum; create, run job, delete). Autoscaling policies. Open-source tools ecosystem.

**Key concepts:**
- **Cluster types:** Single-node (dev/test), multi-node (workers + master), high-availability (3 masters). Optional secondary workers (Spot/preemptible; cheaper, can be interrupted).
- **Serverless:** Serverless for Apache Spark (formerly Dataproc Serverless) runs Spark batches and interactive sessions without managing clusters; autoscaled, billed for resources used.
- **When vs. Dataflow:** Dataproc if you have existing Spark code, PySpark libraries, Hadoop ecosystem, or Spark ML. Dataflow if greenfield, need unified batch+streaming Beam pipelines, or prefer fully serverless scaling.
- **Ephemeral workflows:** Standard pattern = create cluster, submit job, delete cluster (few minutes total). Cost-effective for intermittent workloads.
- **On-cluster caching:** Spark's in-memory cache; use for iterative algorithms, hot data.

**Integration:** Input from GCS, Bigtable, BigQuery, HDFS. Output to same. Dataflow and Dataproc can feed the same BigQuery warehouse.

### BigQuery: Analytics, storage, ML in one

**What it is:** Serverless, petabyte-scale data warehouse. SQL query engine + columnar storage + optional ML (BigQuery ML). Pay per query (on-demand, billed per TiB scanned) or capacity pricing (editions with slots). Multi-region datasets keep data in multiple regions within a location (US/EU); the BigQuery SLA is 99.99% monthly uptime for Enterprise/Enterprise Plus editions ⚠ verify.

**Key configurations:**
- **Storage vs. compute separation:** Storage and compute are billed separately; storage uses lower rates for data unchanged for 90 days (long-term) and can be billed on logical or physical bytes. On-demand compute is billed by bytes scanned. Optimize: partition, cluster, materialized views, BI Engine.
- **Partitioning:** By date/timestamp column, ingestion time or integer range. Reduces bytes scanned per query. Most important for cost; consider requiring a partition filter.
- **Clustering:** Secondary optimization, often combined with partitioning. Cluster by columns frequently used in filters/aggregations, typically with high cardinality (e.g., user_id, customer_id; up to 4 columns, order matters). Within a partition, rows are physically sorted by the clustering columns.
- **Editions and slots:** Pricing models are on-demand (about $6.25/TiB scanned, first 1 TiB per month free) or capacity pricing via editions (Standard, Enterprise, Enterprise Plus) with autoscaling slots billed per slot-hour, plus optional 1-year or 3-year commitments for a discount. Flex slots were retired. Choose based on query volume and predictability; check current prices on the pricing page.
- **BI Engine:** In-memory acceleration for dashboards and BI tools (Looker Studio, Looker, etc.); you reserve memory capacity per project/location, and eligible queries are served from memory. Reduces query latency and bytes billed for accelerated queries.
- **Materialized views:** Pre-aggregated results (auto-refreshed). Queries automatically rewrite to use MV if beneficial. Reduces query cost.
- **BigQuery ML (BQML):** CREATE MODEL statement in SQL. Linear/logistic regression, time series (ARIMA+), clustering (K-means), DNN, XGBoost, matrix factorization, text embedding. No code, no separate ML environment.
- **BigLake (now Google Cloud Lakehouse):** Storage engine/governance layer that lets BigQuery and open-source engines (Spark, Trino) query open-format data (Parquet, Iceberg, etc.) in Cloud Storage (and other clouds) with fine-grained (row/column-level) security. Cost-effective lakehouse.
- **Omni:** Multi-cloud query engine. Query data stored in AWS S3 or Azure Blob Storage from BigQuery without moving it. Billed under BigQuery capacity (editions) or on-demand models for Omni.
- **Authorized views:** Share query results with users who lack access to the underlying tables. For fine-grained control prefer row-level security, column-level security (policy tags) and data masking on the table itself.
- **Streaming ingest:** Use the Storage Write API (the recommended path; supports exactly-once semantics) or Pub/Sub BigQuery subscriptions; data is queryable within seconds. The legacy tabledata.insertAll API is less efficient.
- **Costs:** Partition → cluster → select only needed columns (no `SELECT *`) → materialized views/BI Engine → avoid repeatedly scanning external data. Use `--dry_run` (or the console estimate) to preview bytes scanned; set maximum bytes billed and project/user quotas as guardrails.

**Exam-relevant limits & gotchas:** Table size: effectively unlimited. Partitions per table: 10,000 maximum. Clustering columns: up to 4. Interactive query timeout: 6 hours. Slot capacity and commitments are managed through reservations; on-demand projects share a default cap of about 2,000 slots. Selecting extra columns or `SELECT *` inflates scanned bytes; `LIMIT` does not reduce bytes billed.

**When to choose:** Analytics at scale; SQL-based ML; unified storage + compute; dashboards (Looker integration); data warehouse unification (replace Redshift/Snowflake); lakehouse over open formats (BigLake/Lakehouse). NOT for: OLTP (use Spanner/Cloud SQL/AlloyDB), single-digit-millisecond key lookups (use Bigtable/Memorystore), serving unstructured blobs (use Cloud Storage directly).

### Data orchestration: Cloud Composer, Workflows, Cloud Scheduler, Application Integration

**Cloud Composer (now branded Managed Service for Apache Airflow):** Managed Apache Airflow. DAGs = directed acyclic graphs of tasks. Trigger operators (BQ, Dataflow, Dataproc, GCE), branching, error handling, retries. Python-based (dag.py files). Multi-environment support (dev/staging/prod).
- **When:** Complex multi-service pipelines; hundreds of tasks; conditional logic; time-based scheduling; reusable DAG libraries; monitoring/alerting per task.
- **Overhead:** Environment runs continuously (scheduler, workers, database), so there is a baseline cost even when idle. Overkill for simple schedules.
- **Exam context:** Composer fits nightly ML retraining DAGs and multi-service ETL in the case studies; confirm the specific case-study usage in the official case-study text ⚠ verify.

**Cloud Workflows:** Serverless orchestration (YAML or JSON). Parallel/sequential steps, conditional routing, retries, error handling. Built-in connectors to BigQuery, Cloud Run, Google APIs (HTTP calls, googleapis). Execution duration: long-running (up to one year per execution ⚠ verify).
- **When:** Moderate complexity API orchestration; lower cost; no Airflow expertise needed; rapid iteration.
- **Advantage:** Simpler than Composer; no cluster to manage; fast deployment; per-step pricing (cheap for infrequent workflows); supports callbacks to wait for external events or human approval.
- **Use case:** Payment processing (Stripe integration), approval workflows, ETL orchestration (Pub/Sub → Dataflow → BigQuery).

**Cloud Scheduler:** Fully managed cron job scheduler. Triggers HTTP/S endpoints, Pub/Sub topics or App Engine HTTP targets on a schedule. No workflow logic.
- **When:** Single recurring task (e.g., "run BigQuery query every night" or "trigger Dataflow template hourly").
- **Pricing:** Small free allowance of jobs, then a low per-job monthly price; check the pricing page ⚠ verify.

**Application Integration:** Google Cloud's integration platform as a service (iPaaS). Visual designer (no-code/low-code) with triggers (API, Pub/Sub, scheduler, Salesforce, etc.), 100+ prebuilt connectors (SaaS apps, databases, SAP, etc. ⚠ verify count), data mapping, error handling, manual-approval tasks (human-in-the-loop) and Apigee integration. Built for enterprise app and SaaS integration patterns.
- **When:** Integrating SaaS/enterprise systems and legacy APIs with low-code connectors, business processes with human approval gates, event-driven integrations (can process Pub/Sub messages in order using ordering keys).
- **Exam context:** Cymbal/EHR legacy API integration; human-in-the-loop approvals.

### Data Fusion: Low-code/no-code ETL

**What it is:** Managed service based on the open-source CDAP project. GUI pipeline builder; many pre-built connectors and transformations (databases, cloud services, files) ⚠ verify exact connector count. Pipelines execute on ephemeral Dataproc clusters. Ideal for ops/business analysts without SQL/code skills.

**When to choose:** No-code required; heterogeneous sources (Oracle → Salesforce → GCS); data quality/validation; on-prem integration. NOT for: complex custom transforms or very large-scale/streaming-first workloads (Dataflow), real-time decision making (use Bigtable + application code).

### Datastream: Change Data Capture (CDC)

**What it is:** Serverless CDC and replication service. Streams changes from MySQL, PostgreSQL, Oracle, SQL Server (and others such as MongoDB, Salesforce in preview ⚠ verify) into **BigQuery or Cloud Storage** (the supported destinations). Near real-time replication without custom code. For database-to-database migration/replication into Cloud SQL, AlloyDB or Spanner use Database Migration Service (DMS) instead (or Datastream + Dataflow templates).

**Use:** Database replication into BigQuery, analytics pipeline fed from OLTP source, event-driven architectures on CDC data. Captures insert/update/delete changes and handles schema drift.

### Other services

**Dataplex / Knowledge Catalog:** Unified data governance and metadata. Dataplex Universal Catalog was renamed Knowledge Catalog in April 2026 (API and IAM names unchanged); it covers metadata inventory, lineage, search, data quality and governance across BigQuery, Cloud Storage and other sources. The older Data Catalog service is superseded by it.

**Looker:** BI/dashboarding on BigQuery and other databases. Explores, derived tables, alerts. (see the BI and analytics material elsewhere in the handbook for governance)

**Eventarc:** Event routing service. Receives events from Google Cloud sources (via Cloud Audit Logs and direct events, e.g., Cloud Storage), Pub/Sub topics and third-party sources, and routes to Cloud Run, Workflows, GKE or other HTTP endpoints. Simpler than hand-built Pub/Sub plumbing for GCP-native event sources.

---

## Decision guides

### Choosing a messaging/eventing service

```
Question: What are you decoupling?
├─ Asynchronous producer-consumer (fire-and-forget)
│  └─ Individual HTTP tasks needing rate limiting, delay, retries? → Cloud Tasks
│  └─ High throughput, global scale, fan-out? → Pub/Sub (topics + subscriptions)
│  └─ Ordering required per entity? → Pub/Sub with ordering keys (or Managed Kafka)
│
├─ Event routing from GCP services
│  └─ Native sources (Cloud Storage, Audit Logs)? → Eventarc
│  └─ Custom event source? → Pub/Sub
│
├─ Orchestrating multi-step workflows
│  └─ API/service call orchestration, simple logic? → Workflows
│  └─ Data pipelines with scheduling/dependencies (Airflow DAGs)? → Cloud Composer
│  └─ SaaS/enterprise connectors, human approval? → Application Integration
│
└─ Legacy systems / on-prem integration
   └─ SaaS, ERPs, APIs (low-code)? → Application Integration; data ETL from files/DBs → Data Fusion
```

### Choosing a data processing engine

```
Question: Type & scale of processing?
├─ ETL/streaming pipelines, 100 GB–100 TB
│  └─ Greenfield code, complex transforms? → Dataflow (Beam)
│  └─ Existing PySpark / Hadoop code? → Dataproc (Spark)
│
├─ Structured data, SQL-heavy
│  └─ OLAP analytics at scale? → BigQuery (editions/on-demand)
│  └─ OLTP, real-time inserts? → Spanner / Cloud SQL / AlloyDB (Bigtable for key-value)
│
├─ No-code ETL with prebuilt connectors
│  └─ → Data Fusion
│
└─ Batch jobs, interactive exploration
   └─ Scheduled nightly? → Dataflow Batch or Cloud Scheduler → BigQuery query
   └─ Ad-hoc? → BigQuery (serverless, instant results)
```

### Batch vs. streaming architecture decision

| Scenario | Recommendation |
|---|---|
| Analytics, reporting, BI | Batch (nightly job to BigQuery) |
| Real-time alerts, anomaly detection | Streaming (Pub/Sub → Dataflow → Bigtable or Firestore) |
| Data warehouse sync | Batch ETL or CDC (Datastream) |
| Event-driven workflows | Pub/Sub or Eventarc → Cloud Run / Cloud Run functions |
| Feed features to ML model (inference) | Batch (nightly precomputation) unless online low-latency features are required |
| 24/7 streaming ingestion (e.g., IoT) | Streaming (Pub/Sub → BigQuery or Bigtable) |

---

## Well-Architected Framework lens

| Pillar | In data processing & integration |
|---|---|
| **Operational excellence** | Monitoring: Pub/Sub lag, Dataflow job health, BigQuery slot utilization. Alerting: dead-letter topic backlogs, query timeouts, late data. Logging: audit logs for data access, schema changes. Runbooks for common failures (e.g., Dataflow worker OOM). |
| **Security** | Encryption in transit (VPC-SC for BigQuery), at rest (CMEK). Service accounts with minimal IAM roles. Authorized views, row-level security and column-level security (policy tags) to limit data access. Sensitive Data Protection on Pub/Sub payloads and BigQuery columns. Audit logging of data queries. |
| **Reliability** | Exactly-once processing in Dataflow (with idempotent sinks) or optional Pub/Sub exactly-once delivery (pull, regional). Dead-letter topics for failed messages. Dataflow autoscaling for backlog absorption. BigQuery multi-region datasets for resilience. Idempotent consumers throughout. |
| **Performance** | BigQuery partitioning/clustering, BI Engine cache. Dataflow auto-scaling and worker tuning. Pub/Sub batching trade-offs (throughput vs. latency). Streaming ingest for real-time dashboards. Caching at application layer (Memorystore). |
| **Cost optimization** | BigQuery: partition, cluster, BI Engine. On-demand vs. editions (capacity) decision (high, predictable query volume → capacity pricing). Dataflow: use Spot workers. Dataproc: ephemeral clusters, shut down when done. Pub/Sub: filter messages at source. Move cold data to cheaper Cloud Storage classes via lifecycle rules or Autoclass; BigQuery long-term storage pricing applies automatically after 90 days. |
| **Sustainability** | Batch queries at off-peak (cheaper, greener). Avoid unnecessary cross-cloud data movement (Omni queries data in place). Ephemeral clusters over always-on. Dataflow: choose region close to data source. |

---

## AI/GenAI angle

- **Feature engineering for LLMs:** BigQuery tables → Dataflow pipelines to clean/normalize → feature vectors/embeddings → store in AlloyDB pgvector or BigQuery Vector Search. RAG documents: extract from databases (Datastream + BigQuery → vector embeddings → Agent Platform Vector Search or BigQuery vector search).
- **Real-time feature serving:** Dataflow streams → Firestore, Bigtable or Memorystore for instant lookup during inference. Latency-critical.
- **Data prep and privacy:** Datastream CDC on production DBs → Dataflow for data masking/de-identification (Sensitive Data Protection) → BigQuery for training data (remove PII). Reduces the risk of leaking sensitive data through models.
- **Multi-modal ingestion:** Ingest images/videos from on-prem systems → Cloud Storage → BigQuery object tables (via BigLake/Lakehouse) → Gemini for tagging/classification.
- **Workflow orchestration for ML:** Workflows / Composer (Managed Service for Apache Airflow) to chain: BigQuery feature prep → Agent Platform training → Endpoints deployment → prediction pipeline in Dataflow.
- **Batch predictions:** Scheduled Dataflow job: read features from BigQuery, call Agent Platform Endpoints (batch), write predictions back to BigQuery. Cost-effective.
- **Continuous training:** Cloud Composer DAG: pull recent data (Datastream CDC) → BigQuery → retrain model → swap Endpoints → log metrics in Cloud Monitoring.

---

## Case-study hooks

**Altostrat Media (podcasts, recommendations):**
- Metadata extraction (transcripts, summaries) → Pub/Sub ingestion → Dataflow → BigQuery warehouse → Looker dashboards.
- Inappropriate-content detection → Dataflow + Sensitive Data Protection.
- Real-time recommendations → Pub/Sub → Cloud Run (call Agent Platform endpoint) → user feed.
- Hybrid ingestion (on-prem legacy archival, cloud streaming) → Application Integration connectors or Storage Transfer Service for the archive, Pub/Sub for streaming.

**Cymbal Retail (catalog enrichment, conversational commerce):**
- Catalog ingestion (supplier SFTP/APIs) → Application Integration → Data Fusion (no-code transform) or Dataflow → BigQuery.
- Image generation pipeline (Color variants, BG removal) → Cloud Run jobs triggered by Pub/Sub.
- Product discovery index → BigQuery Vector Search (embeddings) ← Dataflow feature pipeline.
- Human-in-the-loop: Dataflow outputs to Firestore/GCS; review UI reads from there; approval → triggers next stage in Composer DAG.
- Order pipeline: real-time Pub/Sub (order events) → Bigtable for instant inventory checks.

**EHR Healthcare (insurer integrations):**
- Legacy insurer APIs (file-based, SFTP) → Application Integration (or Cloud Scheduler-triggered jobs for nightly sync).
- Analytics on operational databases without load on production → Datastream CDC (MySQL/PostgreSQL → BigQuery). For database replication/migration to Cloud SQL, AlloyDB or Spanner use Database Migration Service, not Datastream.
- Compliance (HIPAA) → Pub/Sub with CMEK, VPC Service Controls, Sensitive Data Protection on data pipelines.
- Real-time provider alerts → Pub/Sub → Cloud Run functions → alert dashboard in Looker.

**KnightMotives Automotive:**
- Vehicle telemetry (millions of data points/s) → Pub/Sub → Dataflow → BigQuery (time-series bucketing).
- Supply chain analytics → Datastream (supported relational sources only; mainframes need other tooling) + Data Fusion (legacy ERP sources) → BigQuery → Looker for plant managers.
- Autonomous-driving simulation data (massive, unstructured video) → Cloud Storage → BigQuery object tables / BigLake for data scientists.
- Dealer tools (real-time inventory, recommendations) → Firestore (low-latency backend) ← batch Dataflow jobs writing latest inventory state nightly.

---

## Exam traps & keyword → answer

1. **"Lower Pub/Sub publish latency"** → Reduce or disable client-side publisher batching (client library batch settings). NOT a server setting; NOT scaling topics.

2. **"Exactly-once message delivery"** → Pub/Sub default is at-least-once. Exactly-once delivery is an optional subscription setting, available for pull subscriptions only and within a single region. Still make processing idempotent for end-to-end guarantees.

3. **"Dead-letter messages"** → Configure a dead-letter topic with max delivery attempts (5–100) and alert on its backlog (Cloud Monitoring). Rising backlog = processing failures; investigate Dataflow/function logs.

4. **"Message ordering in Pub/Sub"** → Optional. Use ordering keys (messages with the same key published in one region are delivered in order) and enable message ordering on the subscription. Default = no ordering; ordering limits throughput per key.

5. **"Dataflow vs. Dataproc"** → Dataflow for Beam pipelines / greenfield / unified batch and streaming. Dataproc (Managed Service for Apache Spark) for existing Spark/PySpark/Hadoop jobs and ecosystem tools, or lift-and-shift of on-prem Hadoop.

6. **"BigQuery cost optimization"** → Partition (with required partition filter) + cluster, select only needed columns, materialized views, BI Engine for dashboards; set maximum bytes billed; pick on-demand vs. editions by query volume.

7. **"Streaming vs. batch"** → Choose streaming only when the requirement states low-latency (seconds) insight; otherwise batch is cheaper and simpler. Streaming adds operational complexity (autoscaling, windowing, late data, cost).

8. **"Cloud Composer vs. Workflows"** → Workflows for serverless orchestration of API/service calls; Composer (Managed Service for Apache Airflow) for data pipelines expressed as Airflow DAGs with scheduling, dependencies, backfills and a large operator ecosystem.

9. **"Pub/Sub retention window"** → Unacked messages are kept up to 7 days; topic retention can be raised to 31 days and acked-message retention enables replay (seek to timestamp or snapshot). Retention is billed as storage.

10. **"BigQuery external query over Cloud Storage"** → External tables read files in place (slower, limited optimization, no storage-side clustering benefit). BigLake tables (Google Cloud Lakehouse) add fine-grained security and metadata caching for better performance.

11. **"Data Fusion with relational sources"** → No-code connectors and visual pipelines; lower ops overhead than custom Dataflow, but complex logic or very large streaming workloads fit Dataflow better. For CDC use Datastream.

12. **"HIPAA/PCI-DSS + data pipeline"** → VPC Service Controls perimeter around Pub/Sub/BigQuery, CMEK, Cloud Audit Logs (including Data Access logs), column-level security / row-level security / authorized views for shared PII, Sensitive Data Protection on payloads.

---

## Quick-reference cheat table

Messaging & eventing comparison (choose by requirement):

| Service | Throughput | Latency | Ordering | Retention | Use |
|---|---|---|---|---|---|
| **Pub/Sub** | Very high, autoscaling | Typically sub-second | Per-key opt. | Up to 7 d (subscription); topic up to 31 d | Async decoupling, fan-out |
| **Kafka / Managed Kafka** | High | Low (ms) | Strict per-partition | Configurable | Kafka-compatible apps, replay, complex topology |
| **Cloud Tasks** | Dispatch-rate limited | Controlled by queue | None guaranteed | Until task done | Deferred, delayed, rate-limited HTTP tasks |
| **Eventarc** | Event-driven | Sub-second to seconds | N/A | N/A (routes events) | GCP-native event routing |
| **Workflows** | Per-step | Any (orchestrator) | Step order | N/A | Serverless orchestration of API calls |
| **Cloud Composer** | Scheduled DAGs | Any (DAG) | DAG order | N/A | Airflow data pipelines, dependencies, backfills |
| **Application Integration** | Low–med | Varies | Workflow order | N/A | SaaS/enterprise connectors, human approval, legacy |

Data processing engines comparison:

| Engine | Type | Scale | Code | Latency | Ops Overhead | Cost |
|---|---|---|---|---|---|---|
| **Dataflow (Beam)** | Batch/Stream | Large, autoscaled | Python/Java/Go | Seconds (streaming) to hours (batch) | Low–medium (serverless) | Per-second vCPU, memory, storage |
| **Dataproc (Managed Service for Apache Spark)** | Batch/Stream/ML | Large, cluster-sized | PySpark/Scala/SQL | Minutes to hours | Medium–high (clusters; low with serverless) | Per-second cluster resources |
| **BigQuery** | SQL / ML | PB | SQL / BQML | Seconds to minutes | Minimal | Per TiB scanned or slot-hours (editions) |
| **Data Fusion** | ETL | Moderate | No-code / simple | Minutes to hours | Low (GUI) | Instance-hours + Dataproc run costs |
| **Cloud Run jobs** | Batch | Small–medium | Container | Minutes | Minimal | Per vCPU/memory-second |

---

## Practice scenarios

**Scenario 1:** A retail company ingests order events from 500+ physical stores (HTTPS POST) in real time. They need a recommendations ML model to score each event and fraud anomalies flagged within a few seconds, with minimal operations. Which architecture best balances latency and operational simplicity?

A) Pub/Sub (with push subscriptions) → Cloud Run functions (call Agent Platform Endpoints for recommendations + fraud check) → Firestore (store latest score)
B) Cloud Tasks (scheduled every 50 ms) → Dataflow → BigQuery
C) Dataproc cluster (Spark Streaming) → self-managed Kafka → custom consumer application
D) BigQuery (real-time ingestion + clustering) → SQL queries on a schedule

**Answer: A**
- **Why A:** Pub/Sub with push delivers events in near real time to serverless Cloud Run functions, which scale automatically and call Agent Platform Endpoints. Firestore stores the latest score for fast retrieval. No cluster overhead (serverless).
- **Why not B:** Cloud Tasks is not a streaming ingestion mechanism; it dispatches individual deferred/rate-limited HTTP tasks. Scheduling a Dataflow job every 50 ms makes no sense.
- **Why not C:** Dataproc requires cluster provisioning and management, and self-managed Kafka adds more ops; overkill for this use case.
- **Why not D:** Scheduled SQL queries are batch analytics, not per-event scoring; latency is far above the requirement.

**Scenario 2:** A healthcare org must replicate a 50 GB MySQL database (running on Cloud SQL) to BigQuery for analytics, and they need CDC (change data capture) to keep it synchronized continuously. Their data includes patient records (HIPAA). Which service and security config?

A) Datastream → BigQuery with CMEK + column-level security (policy tags) or authorized views to protect PII columns
B) Cloud SQL backup → BigQuery external table + Sensitive Data Protection scan
C) Dataflow custom pipeline + Pub/Sub topics (one per table)
D) MySQL replication to replica instance → BigQuery Data Transfer Service

**Answer: A**
- **Why A:** Datastream is purpose-built, serverless CDC with near real-time replication into BigQuery (Cloud SQL for MySQL is a supported source). CMEK encrypts at rest. Policy tags or authorized views restrict PII columns (SSN, medical history), and audit logs record access. (Compliance also depends on a signed BAA and following Google's HIPAA guidance.)
- **Why not B:** Backup → external table is a one-time snapshot, not continuous CDC.
- **Why not C:** Custom Pub/Sub + Dataflow replication is complex to build and maintain (ordering, deduplication, schema changes); Datastream does this natively.
- **Why not D:** A replica instance plus export is not continuous CDC into BigQuery, adds manual ops, and BigQuery Data Transfer Service does not read MySQL replicas.

**Scenario 3:** A media company processes 2 TB of transcoded video metadata daily, performs NLP sentiment analysis using a custom Python library and model, generates recommended content lists, and stores results in BigQuery for dashboards. Metadata comes from Cloud Storage (batch-ingested nightly). Latency tolerance: 4 hours. Budget: minimize compute cost and operations. Which processing engine?

A) Dataflow (batch, Beam) running the custom Python NLP step → write to BigQuery partitioned table
B) BigQuery ML SQL only, with no custom code
C) Cloud Composer + always-on Dataproc cluster (Spark ML) → GCS → BigQuery
D) Data Fusion → BigQuery (no-code, predefined transforms only)

**Answer: A**
- **Why A:** Dataflow batch jobs are serverless and cost-efficient for nightly processing, autoscale horizontally for 2 TB, and can run custom Python code per element. A 4-hour latency tolerance is easily met.
- **Why not B:** BigQuery ML can use built-in or remote models, but it cannot run an arbitrary custom Python library inside SQL; it is the better choice only if a built-in or remote (e.g., Gemini or Natural Language API) model fits the use case.
- **Why not C:** Composer plus a Dataproc cluster means higher ops overhead and cost (always-on environment and cluster); an existing Spark investment is not part of the requirements.
- **Why not D:** Data Fusion is no-code and less flexible for custom NLP logic.

**Scenario 4:** An order-processing system decouples fulfillment from payment. Orders come in via REST API (thousands/sec), trigger payment processing (Stripe API), and if payment succeeds, create a fulfillment task. If payment fails, retry with exponential backoff. Which service orchestrates this?

A) Pub/Sub → Cloud Run (call Stripe API) → Firestore (track state) → Retry via Cloud Tasks
B) Workflows → Stripe API call (HTTP step) → conditional step → Firestore write → configured retry policy
C) Cloud Scheduler (cron) → Dataflow (process batch)
D) Eventarc → Cloud Run → BigQuery (log events)

**Answer: B**
- **Why B:** Workflows is designed for orchestrating HTTP/API calls and Google Cloud services: HTTP steps (no container needed to call Stripe), conditional logic, and retry policies with exponential backoff. It keeps execution state per order. YAML or JSON definition.
- **Why not A:** Could work, but Pub/Sub + Cloud Tasks + Firestore means hand-built state management and retry logic; error-prone compared with a managed orchestrator.
- **Why not C:** Cloud Scheduler is cron; doesn't handle per-order orchestration. Dataflow is batch; not designed for individual order processing.
- **Why not D:** Eventarc is for event routing from GCP sources (Cloud Storage, Pub/Sub), not external APIs like Stripe.

**Scenario 5:** A company wants to build a customer data platform (CDP): ingest clickstream events (10 k/sec) from website, join with CRM data (MySQL), perform audience segmentation, and deliver segments to email marketing tool (webhook). Latency tolerance: up to 1 hour. Prefer low ops overhead. Which components?

A) Pub/Sub (clickstream) → Dataflow → BigQuery, with the CRM table exported manually once a week as CSV and segments sent to the email tool by an engineer-run script
B) Kafka on self-managed VMs → custom consumer → Elasticsearch → webhook caller
C) Pub/Sub with a BigQuery subscription (clickstream) + Datastream (CRM MySQL CDC) → BigQuery segmentation (scheduled SQL) → Application Integration (webhook to email tool)
D) Cloud Composer (DAG) → BigQuery ML segmentation → Dataflow → email webhook, with clickstream sent to the DAG by HTTP POST

**Answer: C**
- **Why C:** Pub/Sub handles 10 k events/s and writes straight to BigQuery without custom code; Datastream keeps the CRM data continuously replicated into BigQuery (fresh joins within the 1-hour tolerance); segmentation is plain SQL; Application Integration (or Workflows) delivers the webhooks with retries. Fully managed and low ops.
- **Why not A:** Weekly manual CRM exports violate the freshness requirement and the manual script is not low-ops.
- **Why not B:** Self-managed Kafka and Elasticsearch require cluster operations and are not a segmentation platform.
- **Why not D:** Airflow DAG runs are not an event-ingestion path for 10 k/s clickstream data; over-engineered and fragile.

**Scenario 6:** Your organization has a petabyte-scale data lake in GCS (images, JSON logs, Parquet tables mixed). Data scientists want to query and perform SQL analytics without copying/converting formats. Compliance: encrypt with customer-managed keys (CMEK). Which service?

A) BigLake tables (Google Cloud Lakehouse) over the Cloud Storage data, plus BigQuery object tables for images, with CMEK on the buckets and BigQuery resources, queried in place from BigQuery
B) Load every file into BigQuery native tables, converting images and logs to rows, then delete the originals
C) Nightly Dataflow job converting all data to CSV and loading it into Cloud SQL
D) Move the data lake to Bigtable and query it with the HBase API

**Answer: A**
- **Why A:** BigLake tables let BigQuery query open-format files (Parquet, JSON, etc.) in Cloud Storage in place, with fine-grained security; object tables expose unstructured data such as images to SQL/ML. No data movement; CMEK is applied to the buckets and BigQuery resources. On-demand queries are billed by bytes scanned (or covered by capacity).
- **Why not B:** Copying and converting a petabyte-scale lake contradicts the "without copying/converting" requirement and cannot represent images as rows.
- **Why not C:** Cloud SQL is not designed for petabyte-scale analytics and the nightly conversion defeats the requirement.
- **Why not D:** Bigtable is a key-value/wide-column store, not a SQL analytics engine for a data lake.

---

## Sources

- [Google Cloud Pub/Sub documentation](https://cloud.google.com/pubsub/docs)
- [Apache Beam programming guide](https://beam.apache.org/documentation/programming-guide/)
- [Google Cloud Dataflow documentation](https://cloud.google.com/dataflow/docs)
- [Google Cloud Dataproc documentation](https://cloud.google.com/dataproc/docs)
- [Google Cloud BigQuery documentation](https://cloud.google.com/bigquery/docs)
- [BigQuery cost optimization guide](https://cloud.google.com/bigquery/docs/best-practices-costs)
- [Google Cloud Datastream documentation](https://cloud.google.com/datastream/docs)
- [Google Cloud Data Fusion documentation](https://cloud.google.com/data-fusion/docs)
- [Google Cloud Composer (Managed Service for Apache Airflow) documentation](https://cloud.google.com/composer/docs)
- [Pub/Sub exactly-once delivery](https://cloud.google.com/pubsub/docs/exactly-once-delivery)
- [BigQuery pricing](https://cloud.google.com/bigquery/pricing)
- [Google Cloud Workflows documentation](https://cloud.google.com/workflows/docs)
- [Google Cloud Application Integration documentation](https://cloud.google.com/application-integration/docs)
- [Professional Cloud Architect exam guide v6.1](https://cloud.google.com/certification/guides/professional-cloud-architect)
- [Google Cloud Well-Architected Framework](https://cloud.google.com/architecture/framework)
