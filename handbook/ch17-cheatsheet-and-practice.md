# Chapter 17 — Cheat Sheet: Comparisons, Decision Trees and Practice Questions

> The last-week reference. Order: comparison tables (databases, storage, compute, deployment, messaging, DR, migration, load balancers, hybrid and VPC connectivity, encryption, IAM, governance, observability, IaC and CI/CD, pricing), then the AI service picker, decision trees, keyword-to-answer lookup, numbers to remember, the top 50 traps, 50 scenario questions with answers, and the final-week checklist. Each comparison ends with a "Pick it when" line of trigger keywords. The owning chapter is shown in brackets, for example (Ch05).

## Database comparison

### Relational: Cloud SQL vs AlloyDB vs Spanner vs Bare Metal/Oracle

| Service | Scale | Consistency | Latency | HA / Multi-Region | Cost Model | Best For |
|---------|-------|-------------|---------|-------------------|-----------|----------|
| **Cloud SQL** | Vertical scale; storage up to ~64 TB per instance | Strong (single primary) | Low ms | Regional HA (synchronous standby, automatic failover) + read replicas / cross-region replicas | Per instance + storage | Traditional OLTP (MySQL, PostgreSQL, SQL Server), easy lift-and-shift |
| **AlloyDB** | Vertical scale + read pools; storage auto-scales (up to ~128 TB) | Strong (primary); read pools replicate with very low lag | Low ms | Regional HA (99.99 % SLA incl. maintenance) + cross-region replication | Per vCPU + memory + storage | PostgreSQL-compatible HTAP (OLTP + columnar engine for real-time analytics), demanding PostgreSQL workloads |
| **Spanner** | Horizontal scale, effectively unlimited (nodes / processing units) | Strong globally (external consistency, serializable) | Low ms for regional reads; multi-region writes cost cross-region round trips | Regional 99.99 %, multi-region 99.999 % SLA, automatic failover, no downtime | Per node / processing unit + storage | Global consistency (finance, inventory), horizontal scale, distributed transactions |
| **Bare Metal Solution / Oracle Database@Google Cloud** | Customer-sized Oracle hardware | Strong | Low (Interconnect-attached) | Customer-managed (Oracle RAC / Data Guard) | Per server + BYOL Oracle licence | Legacy Oracle (and SAP) that cannot be refactored; zero-code-change Oracle move |

**Pick it when:** Cloud SQL = "minimal ops, standard RDBMS"; AlloyDB = "need analytics + speed"; Spanner = "global consistency is non-negotiable"; Bare Metal = "license lock-in or vendor requirement".

### NoSQL and in-memory: Bigtable vs Firestore vs Memorystore vs MongoDB-compatible options

| Service | Model | Scale | Consistency | Latency | HA / Multi-Region | Cost Model | Best For |
|---------|-------|-------|-------------|---------|-------------------|-----------|----------|
| **Bigtable** | Wide-column (time-series) | Billions rows, petabyte | Strong within a single cluster; eventual across replicated clusters | Single-digit ms | Multi-cluster async replication | Per-node + storage | Time-series, IoT, analytics at massive scale, HBase-compatible |
| **Firestore** | Document (NoSQL) | Millions docs, GB–TB | Strong (all reads and queries) | Low ms–tens of ms | Regional 99.99 %, multi-region 99.999 % SLA | Per read/write/delete + storage | Real-time mobile/web, user profiles, presence, sub-second updates |
| **Memorystore (Redis / Valkey / Memcached)** | In-memory cache | GB–TB (cluster mode) | Per-primary; replicas async | Sub-millisecond | Standard tier / cluster with replicas and automatic failover | Per GB provisioned | Session cache, rate limiting, counters, real-time leaderboards |
| **MongoDB-compatible (Atlas on GCP)** | Document | Millions–Billions | Configurable (strong/eventual) | <10 ms | Multi-region replica sets | Per operation / provisioned | Existing MongoDB apps, flexible schema, community ecosystem |

**Pick it when:** Bigtable = "massive time-series or analytics scale"; Firestore = "real-time, document-oriented, mobile-first"; Memorystore = "ultra-low latency cache"; MongoDB = "vendor lock-in or ecosystem".

### Analytics: BigQuery vs Spanner vs AlloyDB columnar vs Bigtable

| Service | Model | Scale | Consistency | Query Type | Latency | Cost Model | Best For |
|---------|-------|-------|-------------|-----------|---------|-----------|----------|
| **BigQuery** | Columnar data warehouse | Petabyte+ | Strong (OLAP, not for row-level OLTP) | SQL, analytics, ML | 10 sec (batch) / sub-sec (BI Engine) | Per-scan ($ / GB) or slots | Analytics, BI, ML feature engineering, ad-hoc queries |
| **Spanner** | Distributed RDBMS (columnar indexes optional) | Exabyte | Strong globally | SQL (OLTP + BI) | <10 ms OLTP, <100 ms BI | Per node + RPC | Global BI with transactional consistency; complex joins |
| **AlloyDB columnar** | RDBMS with columnar engine | Up to ~128 TB per cluster | Strong | SQL (OLTP + BI) | Low-ms OLTP, fast analytics on columnar data | Per vCPU | HTAP (real-time analytics + OLTP without ETL); PostgreSQL ecosystem |
| **Bigtable** | Wide-column (time-series focused) | Petabyte | Strong in one cluster; eventual across clusters | Key-value / row-range scans, no SQL joins | Single-digit ms | Per node | High-throughput serving of time-series; heavy analytics via Dataflow / BigQuery federation |

**Pick it when:** BigQuery = "large batch BI, cost-per-scan model fits"; Spanner = "need transactional consistency + global BI"; AlloyDB = "OLTP + BI in one system"; Bigtable = "raw scale for analytics compute".

### Database picker: keyword → service

| Keyword / Constraint | Answer Service | Reasoning |
|---|---|---|
| "Global consistency" or "distributed transactions" | Spanner | Only GCP DB with serializable isolation across regions |
| "Easy lift-and-shift MySQL/PostgreSQL" | Cloud SQL | Minimal schema changes, native replication, managed HA |
| "Real-time mobile app, sub-second sync" | Firestore | Document model, strong consistency per-doc, real-time listeners |
| "Massive time-series (billions of points)" | Bigtable | Row-key design optimized for sequential scans, petabyte scale |
| "OLTP + analytics in one" | AlloyDB (or Spanner if global) | AlloyDB: columnar engine speeds analytics, read pools scale out; Spanner: distributed |
| "Ultra-low latency cache (< 1 ms)" | Memorystore (Redis) | In-memory, not disk-backed |
| "Need to query data already in BigQuery" | BigQuery | Avoid moving data out; use SQL or BQML |
| "Oracle migration, zero code changes" | Bare Metal Solution / Oracle Database@Google Cloud | Runs Oracle as-is. Moving to AlloyDB / Cloud SQL PostgreSQL (DMS + conversion workspace) or Spanner needs schema and code conversion |
| "Document store, flexible schema" | Firestore | Schemaless, strongly consistent, supports transactions |

---

## Storage comparison

### Cloud Storage classes and minimum storage durations and location types

| Class | Min. Duration | Use Case | Location Types | Cost |
|-------|---|---|---|---|
| **Standard** | None | Frequent access, hot data, real-time analytics | Region, Dual-region, Multi-region | Highest per-GB (no minimum), lowest retrieval |
| **Nearline** | 30 days | Infrequent (log archival after 30d, backup snapshots) | Region, Dual-region, Multi-region | Moderate per-GB, moderate retrieval |
| **Coldline** | 90 days | Cold archives, compliance archives, rare access | Region, Dual-region, Multi-region | Lower per-GB, high retrieval |
| **Archive** | 365 days | Long-term legal holds, rarely accessed, compliance | Region, Dual-region, Multi-region | Lowest per-GB, highest retrieval |
| **Autoclass** | None for transitions (no retrieval or early-deletion fees) | Unknown access patterns; auto-moves objects Standard→Nearline→Coldline (→Archive if enabled) and back to Standard on access | Region, Dual-region, Multi-region | Optimal (no manual rules; small management fee) |

**Pick it when:** Standard = "hot data, BI queries, frequent access"; Nearline = "after 30 days of non-access"; Archive = "compliance holds, 7+ year retention"; Autoclass = "uncertain access pattern, prefer simplicity".

### Block storage: Persistent Disk types vs Hyperdisk vs Local SSD

| Type | Durability | Latency | Throughput | Snapshots | Use Case | Cost |
|---|---|---|---|---|---|---|
| **Persistent Disk (pd-standard)** | Durable, replicated (zonal; regional PD optional) | Highest (HDD) | Lowest; scales with size | Yes | Large sequential / low-IOPS workloads, boot disks | Low |
| **Persistent Disk (pd-balanced)** | Durable, replicated | Low ms (SSD) | Moderate; scales with size | Yes | General-purpose, default choice | Low–Medium |
| **Persistent Disk (pd-ssd)** | Durable, replicated | Low ms (SSD) | Higher IOPS per GB than pd-balanced | Yes | High-IOPS databases, transactional | Medium |
| **Persistent Disk (pd-extreme)** | Durable, replicated | Low | Provisioned IOPS (legacy; Hyperdisk Extreme is the successor) | Yes | Legacy high-IOPS (Oracle, SAP) | High |
| **Hyperdisk Balanced** | Durable, replicated | Low | IOPS and throughput provisioned independently of size | Yes | Modern general / database workloads, newer machine series (C3, N4, etc.) | Medium–High |
| **Hyperdisk Extreme** | Durable, replicated | Lowest of the network disks | Highest provisioned IOPS | Yes | Mission-critical OLTP, large Oracle / SAP HANA | Very High |
| **Local SSD** | Ephemeral (data lost when the VM stops or is deleted; survives live migration) | Lowest (physically attached NVMe) | Very high | No snapshots | Scratch, caching, temporary data | Medium (per 375 GB unit) |

**Pick it when:** pd-balanced = "default for most workloads"; pd-ssd/Hyperdisk = "high-IOPS database"; Local SSD = "temporary cache, don't need persistence"; Hyperdisk Extreme = "SAP/Oracle/premium OLTP".

### File and parallel storage: Filestore tiers vs NetApp Volumes vs Managed Lustre vs Parallelstore vs Cloud Storage FUSE

| Service | Protocol | Use Case | Tier / HA | Latency | Scale | Cost |
|---------|----------|----------|-----------|---------|-------|------|
| **Filestore (Basic)** | NFSv3 / v4.1 | Dev/test shared FS | Zonal (99.9 % SLA) | <5 ms | Multi-TB | Low |
| **Filestore (Regional)** | NFSv3 / v4.1 | Production shared FS (GKE, ML, batch) | Multi-zone HA (99.99 % SLA; older docs say 99.95 %) | Low ms | Multi-TB | Medium |
| **Filestore (Enterprise)** | NFSv3 / v4.1 | High-availability, snapshots, advanced (regional-class tier, GKE CSI) | Multi-zone HA (same SLA as Regional) + snapshots | Low ms | Multi-TB | Medium–High |
| **NetApp Volumes** | NFS / CIFS / SMB | Advanced data mgmt (snapshots, clones, dedup) | Regional HA | <5 ms | Multi-TB | High |
| **Managed Lustre** | POSIX | High-performance HPC, parallel ML training | Zonal / Regional | <1 ms | 100+ TB | Medium (consumption-based) |
| **Parallelstore** | POSIX (DAOS-based) | Ultra-high-throughput AI/ML training scratch | Zonal | Sub-ms | Up to ~100 TiB per instance ⚠ verify | Premium (HPC-tier) |
| **Cloud Storage FUSE** | POSIX mount | Data lakes via filesystem interface | Multi-region (GCS backend) | <100 ms | Petabyte (GCS) | Per-GCS cost |

**Pick it when:** Filestore Regional = "managed NFS, GKE/ML shared FS"; NetApp = "advanced data services, vendor ecosystem"; Managed Lustre = "HPC, distributed training"; Parallelstore = "very high-throughput AI scratch space"; Cloud Storage FUSE = "lake-scale data access as POSIX".

---

## Compute comparison

### Compute option matrix: Compute Engine, GKE Autopilot, GKE Standard, Cloud Run, Cloud Run functions, Cloud Run jobs, App Engine, Batch, VMware Engine, Bare Metal

| Service | Scale to Zero | Cold Start | Ops Burden | Pricing | Stateful Support | Concurrency Model | GPU / TPU | Best For |
|---------|---|---|---|---|---|---|---|---|
| **Compute Engine + MIG** | No | ~5 s boot | Moderate–High | Per VM·second (idle costs) | Yes | Manual (MIG policy) | Yes (A/L series) | Stateless/batch, complex networking, VMs as primary |
| **GKE Autopilot** | Pods only (cluster fee remains; no node management) | 10–30 s (pod scheduling may wait for a new node) | Minimal (Google manages nodes) | Per pod vCPU + memory + storage requested | Yes | HPA / VPA / KEDA | Yes (via compute classes) | Containerised, variable workloads, managed K8s |
| **GKE Standard** | Node pools can scale to 0 (not the last/default pool); cluster fee remains | 10–30 s | Moderate | Per node (even idle) | Yes | Full control (node pools, CA, NAP) | Yes | Heterogeneous, custom node config, DaemonSets, privileged workloads |
| **Cloud Run** | Yes | <1 s | None (serverless) | Per request + duration | No | Built-in, tunable concurrency | GPU (inference) | HTTP APIs, microservices, event handlers, cost-sensitive |
| **Cloud Run functions** | Yes | Sub-second to seconds | None | Same as Cloud Run (per request + duration) | No | Default 1 request per instance (configurable on 2nd gen) | No | Simple event triggers, webhooks, glue code |
| **Cloud Run jobs** | Yes | Seconds | Minimal | Per vCPU·second + memory | No | Parallelism via tasks | Yes (GPU) | Run-to-completion batch processing, scheduled / one-off tasks |
| **App Engine** | Yes (Standard) | Seconds | Low | Per request or per instance | Limited | Auto | No | Static sites, low-traffic apps (use Cloud Run instead) |
| **Batch** | Yes (async) | Minutes (queue-based) | Low | Per vCPU + GB·second | No | Parallelism via task groups, retries | Yes | Large-scale batch jobs, ML training, simulations, HPC |
| **VMware Engine** | No | N/A (always-on dedicated bare-metal nodes) | Moderate (Google runs hardware and vSphere stack; you run VMs) | Per node (hourly or commitment) | Yes | Native vSphere | GPU nodes available | Rehost VMware workloads without re-platforming, data-centre exit |
| **Bare Metal Solution** | No | N/A (provisioned hardware) | Very High | Per server (term-based) | Yes | Bare hardware | No | Oracle and other licensed / specialised workloads that cannot be virtualised |

**Pick it when:** Cloud Run = "serverless, variable load"; Batch = "massive parallel jobs"; GKE Autopilot = "containers, low-ops"; Compute Engine = "long-running, complex"; VMware Engine = "lift-and-shift VMware".

### Machine families and purchasing: general, compute-, memory-, accelerator-optimised; Spot vs standard vs sole-tenant

| Family / Option | vCPU | Memory | Use Case | Volatility | Cost |
|---|---|---|---|---|---|
| **General (E2, N2, N2D)** | 2–96 | 0.5–7.2 GB/vCPU | Web, app, dev, balanced | Standard / Spot / CUD | Low (baseline) |
| **Compute-optimized (C2, C2D)** | 4–96 | 0.25–1.75 GB/vCPU | Batch, gaming, scientific | Standard / Spot / CUD | Medium (high-CPU) |
| **Memory-optimized (M-series, e.g. M3)** | Up to hundreds of vCPUs | Highest memory per vCPU (well above 14 GB/vCPU) | In-memory databases (SAP HANA), large analytics | Standard / CUD (Spot generally not offered) | High (high-memory) |
| **Accelerator-optimized (A2, A3, A4, G2 / G4)** | 12 to 200+ | Large; attached GPUs (TPUs are separate Cloud TPU VMs) | ML training, inference, GPU workloads | Standard / Spot / reservations / DWS | High (GPU premium) |
| **Spot VMs** | Any family (except some, e.g. M-series) | Any tier | Non-critical, tolerant of eviction | Up to ~60–91 % cheaper, 30 s preemption notice, no max runtime | Very Low |
| **Standard (on-demand)** | Any family | Any tier | Always available, predictable | Always available, per-second billing | Baseline |
| **Sole-tenant nodes** | Entire server | Entire server | BYOL licensing (Windows, SQL Server, Oracle) tied to physical cores, isolation, compliance | Standard / CUD | ~10 % sole-tenancy premium on the node price |

**Pick it when:** General = "default web/app"; Compute = "batch heavy"; Memory = "databases, caching"; Spot = "fault-tolerant, save up to ~90 %"; Sole-tenant = "BYOL SQL Server / Oracle".

### Autoscaling: MIG autoscaler vs HPA vs VPA vs Cluster Autoscaler vs Cloud Run concurrency

| Scaler | What it scales | Signal | Latency | Best For | Trap |
|--------|---|---|---|---|---|
| **MIG Autoscaler** | Number of VMs | CPU %, custom metrics, HTTP LB | 1–3 min | Stateless fleets (web tier) | Can't distinguish between real load and false alarm |
| **HPA (K8s)** | Pod replicas | CPU %, memory, custom metrics (QPS, latency) | 15–30 s | Kubernetes workloads, sub-minute spikes | Needs CPU requests set correctly or scales wildly |
| **VPA (K8s)** | Pod resource *requests* | Historical CPU/memory usage | Hours (slow) | Right-sizing pods, gradual optimization | Restarts pods; not for fast autoscaling |
| **Cluster Autoscaler (CA)** | Number of nodes in pool | Pending pods, node utilization | 1–3 min | Provision new nodes as pods can't fit | Works best with HPA (pods request resources, CA provisions nodes) |
| **Node Auto-provisioning (NAP)** | Automatically creates pools | Pending pod resource requests | 2–5 min | Heterogeneous workloads (don't pre-define pools) | Less control than manual pools + CA |
| **Cloud Run concurrency** | Container instances | Concurrent requests per instance (configurable) | <1 s | Serverless request-driven scaling | Max concurrency too low → request queuing; too high → OOM |

**Pick it when:** MIG = "VM fleet"; HPA = "fast pod scaling on K8s"; VPA = "gradual right-sizing"; CA = "node-level bottleneck"; Cloud Run = "request-driven autoscale".

---

## Deployment strategies

### Strategy comparison: recreate, rolling, blue-green, canary, A/B, shadow — downtime, rollback speed, cost, risk, needs

| Strategy | Downtime | Rollback Speed | Cost During | Risk Level | Exam Keyword | Use Case |
|----------|----------|---|---|---|---|---|
| **Recreate** (kill old, start new) | Yes (full downtime) | Slow (redeploy old) | Lowest (no parallel infra) | High (user impact) | "Downtime acceptable" | Batch jobs, internal tools, maintenance windows |
| **Rolling** (gradual instance replacement) | No (canary-like, implicit) | Medium (depends on pod health) | Same as running | Medium (staggered, can detect issues) | "Gradual rollout," "no downtime" | Standard Kubernetes, stateless services, GKE default |
| **Blue-green** (instant switch) | No (instant switchover) | Very fast (flip switch) | High (2x infra during transition) | Low (test everything before switch) | "Zero downtime," "instant rollback" | Critical services, high confidence, full stack tested |
| **Canary** (% traffic to new, auto-rollback) | No | Fast (abort traffic if metrics bad) | Slightly higher (parallel versions) | Low (detects regressions early) | "Gradual rollout," "auto-rollback," "metrics gate" | Risky releases, real-world load testing, production-first validation |
| **A/B** (user segment split, compare) | No | N/A (different versions coexist) | Higher (both versions running) | Low (user segment impact, not rollback) | "Experimentation," "user split," "compare" | Feature flags, experiment validation, marketing tests |
| **Shadow** (mirror traffic, no user impact) | No | N/A (old version serves, new shadowed) | Higher (duplicate infra + processing) | Very low (shadow results discarded) | "Test in production," "real traffic pattern" | Risky rewrites, data pipeline validation, new backend testing |

**Pick it when:** Recreate = "maintenance, batch"; Rolling = "default K8s"; Blue-green = "critical, need instant rollback"; Canary = "risky release, need metrics gate"; A/B = "feature experiment"; Shadow = "validate rewrite with real traffic".

### How each strategy is done per platform

#### Compute Engine + MIG rolling update

- **Mechanism**: MIG template update → instances replaced in waves (e.g., 20 % per minute).
- **Config**: max surge (how many new instances before old removed), max unavailable (downtime tolerance).
- **Rollback**: manual (revert to the previous template and trigger a new rolling update). Autohealing only recreates unhealthy VMs; it does not roll back a release.
- **Downtime**: zero if max-unavailable = 0, max-surge > 0.

#### GKE Rolling (default)

- **Mechanism**: Deployment rolling update → old Pods terminated, new Pods scheduled, health checks (readiness probes).
- **Config**: maxSurge (extra Pods allowed), maxUnavailable (Pods that can be down during update).
- **Rollback**: `kubectl rollout undo` (instant rollback to previous ReplicaSet).
- **Downtime**: zero if maxUnavailable = 0.

#### Cloud Run Traffic Splitting (Blue-Green / Canary)

- **Mechanism**: Deploy new revision, then manually or via Cloud Deploy adjust % traffic (0 → 50 % → 100 %).
- **Cloud Deploy integration**: canary strategy with percentage phases and optional verification (and automated rollback via automation rules).
- **Rollback**: instant (redirect traffic back to old revision).
- **Downtime**: zero.

#### Cloud Deploy Canary (multi-platform)

- **Targets**: GKE (and attached clusters), Cloud Run; other platforms (for example Compute Engine MIGs) only through custom targets.
- **Mechanism**: 
  - Deploy new version to the canary phase (e.g., 10 % of traffic or pods).
  - Optional verify job (tests) runs against the canary.
  - Advance manually or automatically through the percentages (e.g., 25 → 50 → 100 %).
  - If verification fails → roll back (manual, or automatic via automation rules).
- **Rollback**: manual, or automatic when verification fails.
- **Downtime**: zero.

#### App Engine Traffic Splitting

- **Mechanism**: Deploy new version, then adjust traffic split (% to old vs new).
- **Config**: split by IP address, cookie or randomly; the "gradual migration" option shifts traffic to the new version over time.
- **Rollback**: traffic re-routed to previous stable version (if live).
- **Downtime**: zero.

**Pick platform strategy when:** Compute Engine MIG = "VM-heavy"; GKE = "Kubernetes rollout"; Cloud Run = "serverless traffic split"; Cloud Deploy = "CD-first, multi-platform gates".

---

## Messaging and data processing

### Messaging and eventing: Pub/Sub vs Pub/Sub Lite vs Eventarc vs Cloud Tasks vs Cloud Scheduler vs Workflows vs Application Integration

| Service | Throughput | Latency | Ordering | Retention | Scaling | Best For | Exam Keyword |
|---------|-----------|---------|----------|-----------|---------|----------|---|
| **Pub/Sub** | Very high (millions msgs/s) | ~100 ms–1 s typical | Per ordering key (opt-in) | Subscription default 7 days (up to 31 days with topic retention) | Auto (global service, no partitions to manage) | Async decoupling, fire-and-forget, event fanout, streaming ingest | "Decouple producers/consumers" |
| **Pub/Sub Lite** | Discontinued (shut down 18 Mar 2026; migrate to Pub/Sub) | n/a | n/a | n/a | n/a | Do not choose in new designs | Appears only in old material |
| **Eventarc** | High | Sub-second to seconds | N/A | N/A (event router; delivers via Pub/Sub transport) | Auto | GCP-native event routing (Audit Log, Pub/Sub, direct events → Cloud Run, GKE, Workflows) | "GCP events, event-driven" |
| **Cloud Tasks** | Low–med | Asynchronous; supports scheduled delay | N/A | Until completed or expired | Auto (queues) | Deferred / delayed HTTP callbacks, rate-limited dispatch, retries to a target | "Delayed execution," "rate limiting" |
| **Cloud Scheduler** | Low (scheduled) | At scheduled time | N/A | None (scheduler, not queue) | Fixed schedule | Cron jobs, periodic tasks, maintenance windows | "Cron, scheduled trigger" |
| **Cloud Workflows** | Low | Any (orchestrator) | Sequential steps, parallel branches | Execution state kept for a limited period | Per workflow invocation | Serverless multi-step orchestration of HTTP / Google APIs, callbacks (human approval), long waits | "Orchestration, approval gates" |
| **Application Integration** | Low–med | Varies | Workflow order | Configurable | Per integration | No-code / low-code integrations, 100+ connectors (SAP, Salesforce, on-prem), human approval tasks | "Connectors, legacy integration, human step" |

**Pick it when:** Pub/Sub = "fire-and-forget"; Eventarc = "GCP-native events"; Cloud Tasks = "delayed or rate-limited"; Scheduler = "cron"; Workflows = "simple orchestration"; Application Integration = "connector-heavy, low-code integration".

### Processing engines: Dataflow vs Dataproc vs Dataproc Serverless vs BigQuery vs Data Fusion vs Cloud Composer

| Engine | Type | Scale | Code | Latency | Ops Overhead | Cost | Best For |
|--------|------|-------|------|---------|-------------|------|----------|
| **Dataflow** | Batch & Stream (Beam) | 100 GB–PB | Python / Java / Go (Apache Beam) | 100 ms (stream) / hrs (batch) | Medium (autoscaling, managed) | $ / (vCPU·h + bytes input) | Unified batch/stream, complex transformations, windowing |
| **Dataproc** | Batch & Stream (Spark, Hadoop) | 100 GB–PB | PySpark / Scala / HQL | 10 min–hrs (batch) | High (cluster management, sizing, scheduling) | $ / (vCPU·h) + cluster time | Large Spark jobs, data science teams, Hadoop ecosystem |
| **Dataproc Serverless** (now Serverless for Apache Spark) | Batch (Spark) | 100 GB–PB | PySpark / Scala | 10 min–hrs | Minimal (no cluster mgmt) | $ / (vCPU·s) | Spark without cluster ops overhead |
| **BigQuery** | SQL analytics & ML | PB | SQL / BQML | 10 s–min (batch) / sub-sec (BI Engine) | Minimal | $ / (bytes scanned) or $/slot·h | Analytics, BI, ML, ad-hoc queries, fast feedback |
| **Data Fusion** | ETL (no-code/low-code) | 100 GB–TB | Graphical / simple logic | 10 min–hrs | Low (GUI, Dataflow backend) | $ / (compute hours) | Visual ETL, business users, quick pipelines |
| **Cloud Composer** | Orchestration (DAG) | Hundreds of tasks | Airflow DAG (Python) | Any (orchestrator sync) | Low (Composer manages) | $ / (Composer environment + compute) | Complex workflows, 100+ tasks, multi-step dependencies |

**Pick it when:** Dataflow = "unified batch/stream, Beam"; Dataproc = "large Spark ecosystem"; BigQuery = "analytics + speed"; Data Fusion = "no-code ETL"; Composer = "100+ task DAGs".

### Batch vs streaming decision table

| Constraint | Batch | Stream | GCP Tool |
|-----------|-------|--------|----------|
| Data arrives continuously | ❌ | ✅ | Pub/Sub → Dataflow / Spark Streaming |
| Need results in minutes | ✅ | ✅ | BigQuery batch, Dataflow window |
| Need results in <500 ms | ❌ | ✅ | Pub/Sub → Cloud Run functions or Dataflow |
| Reconciliation / replay needed | ✅ | ✅ (with audit log) | Dataflow (Beam) with state, Pub/Sub message retention + seek/replay |
| Complex aggregations (joins, windows) | ✅ | ✅ | Dataflow, Spark Streaming |
| Cost-optimized for occasional queries | ✅ | ❌ | BigQuery batch, on-demand |
| Must preserve event order | ❌ (unless by key) | ✅ | Pub/Sub per-key ordering, Dataflow |

**Pick it when:** Batch = "scheduled, large bulk data, cost-optimized"; Stream = "continuous data, low-latency insights required".

---

## Disaster recovery patterns

### DR pattern comparison: backup and restore, pilot light, warm standby, hot/active-active — RTO, RPO, cost, complexity, GCP implementation

| Pattern | Backup Interval | Failover Time (RTO) | Data Loss (RPO) | Cost | Complexity | GCP Implementation | Use Case |
|---------|---|---|---|---|---|---|---|
| **Backup & Restore** (Cold) | Daily–weekly | 8–24 hours | Hours–days | Lowest | Low | Cloud Storage (Archive) + Cloud SQL snapshots, restore from Cloud Storage | Non-critical, acceptable downtime |
| **Pilot Light** | Continuous (async) | 10–60 min | <5 min | Low–moderate | Medium | Cloud SQL async replica (read-only standby) + Datastream | Moderate SLA (99.5–99.9 %), secondary region |
| **Warm Standby** | Continuous (async) | 1–5 min | <1 min | Moderate–high | Medium–high | Scaled-down copy of the stack in a second region (cross-region Cloud SQL / AlloyDB replica, small MIG or GKE cluster behind a global LB), scale up on failover | High SLA (99.9–99.95 %), frequent failover drills |
| **Hot / Active-Active** | Sync (quorum) or multi-writer | Seconds or less | ≈0 for synchronous stores (Spanner multi-region, Firestore multi-region); small but non-zero for async-replicated stores (Bigtable, Cloud Storage) | High–very high | High | Global LB + full stacks in 2+ regions; Spanner multi-region, Firestore multi-region, Bigtable multi-cluster routing | Critical (99.95–99.999 %), near-zero downtime tolerance |

**Pick it when:** Backup = "non-critical, cost-first"; Pilot Light = "moderate SLA, cost-conscious"; Warm = "high SLA, acceptable brief downtime"; Hot = "critical, zero-downtime requirement".

### Per-service HA and DR features

| Service | HA option (single region) | Multi-region DR | RTO | RPO | SLA (multi-region) |
|---------|---|---|---|---|---|
| **Cloud SQL** | Regional HA (synchronous standby in another zone, automatic failover, roughly a minute or two) | Cross-region read replica promoted manually (Enterprise Plus: advanced DR with switchover) | Minutes (HA failover is automatic; cross-region promotion is manual) | ≈0 for zonal HA (sync); cross-region replica = async lag | 99.95 % (Enterprise HA), 99.99 % (Enterprise Plus) |
| **Spanner** | Replicas in 3 zones of a region (implicit) | Multi-region configuration (synchronous quorum, automatic failover) | ≈0 (automatic) | 0 (sync writes) | 99.99 % regional, 99.999 % multi-region |
| **Bigtable** | Single cluster is zonal (99.9 %) | Multi-cluster replication (async, eventually consistent) with multi-cluster routing | Seconds–minutes (automatic with multi-cluster routing) | Seconds–minutes (async replication) | 99.99 % (replicated, <3 regions), 99.999 % (multi-cluster routing, 3+ regions) |
| **Firestore** | Regional (replicated across zones) | Multi-region (synchronous, strongly consistent) | ≈0 (automatic) | 0 | 99.99 % regional, 99.999 % multi-region |
| **Cloud Storage** | Regional (data redundant across zones) | Dual-region or Multi-region (async replication between regions; default replication typically within ~1 h, turbo replication 15 min on dual-region) | Automatic (single bucket name keeps working) | Non-zero by default (async); 15 min with turbo replication | Standard class: 99.9 % regional, 99.95 % dual/multi-region |
| **GKE** | Regional cluster (control plane and nodes across 3 zones) | Multiple clusters in 2+ regions + multi-cluster Gateway / Ingress with Cloud Service Mesh (formerly Traffic Director) | Seconds (pod rescheduling) / minutes (regional shift) | Seconds (stateless), depends on storage tier for persistent volumes | 99.95 % (regional control plane), 99.5 % (zonal) |
| **Compute Engine** | Regional MIG (VMs across zones) | Global LB + MIGs in multiple regions | Seconds (LB redirect) | Seconds (stateless) | 99.99 % (instances across multiple zones), 99.9 % (single instance) |
| **Backup and DR Service** (managed) | Snapshots / PITR | Cross-region backup, cross-project recovery | Custom (restore time) | Custom (backup frequency) | Depends on RTO/RPO SLA defined |

**Pick it when:** Cloud SQL HA = "single-region high availability"; Spanner = "multi-region zero-downtime"; Cloud Storage dual/multi = "geo-redundancy"; Backup and DR = "managed recovery service".

---

## Migration tools and strategies

### The 6 Rs: rehost, replatform, refactor, repurchase, retire, retain — effort, benefit, typical tool

| R | Definition | Example | Time-to-Cloud | Cost-to-Move | Ongoing OpEx | Typical Tool | Exam Keyword |
|---|---|---|---|---|---|---|---|
| **Rehost** | Move VM as-is, no code changes. | On-prem app → Compute Engine Windows VM | Fast (days) | Low | High (BYOL, larger VMs) | Migrate to Virtual Machines | "Lift-and-shift," "minimal changes," "urgent" |
| **Replatform** | Move to managed services, minimal code changes. | MySQL on-prem → Cloud SQL; .NET IIS → Cloud Run | Medium (weeks) | Low–medium | Medium | Database Migration Service, Cloud Deploy | "Managed services," "app refactor minimal" |
| **Refactor** | Rewrite for cloud-native (microservices, serverless). | Monolith → GKE + Pub/Sub; batch → Dataflow | Slow (months) | High (engineering) | Medium–low (autoscaling) | Cloud Build, Cloud Deploy, GKE | "Cloud-native," "microservices," "long-term" |
| **Repurchase** | Switch to cloud SaaS. | Custom ERP → SAP on Google Cloud | Medium–long | High (license + migration) | Very high (SaaS subscription) | SaaS vendor + migration partner | "Replace with SaaS," "vendor standard" |
| **Retire** | Turn off; data archived/deleted. | Legacy reporting system → BigQuery + Looker | Immediate | Zero | Zero | Cloud Storage (Archive), document archive | "Decommission," "no longer used" |
| **Retain** | Keep on-prem (hybrid/coexistence). | Mainframe, legacy systems not yet ready. | None (stays) | None | On-prem (high) | Cloud Interconnect (for hybrid link) | "Can't move," "data sovereignty," "long-term hybrid" |

**Pick it when:** Rehost = "urgent timeline"; Replatform = "managed services fit"; Refactor = "long-term modernization"; Repurchase = "strategic SaaS switch"; Retire = "redundant systems"; Retain = "regulatory/zero-downtime blocker".

### Migration tool picker: Migration Center, Migrate to Virtual Machines, Database Migration Service, Datastream, Storage Transfer Service, Transfer Appliance, VMware Engine, BigQuery Data Transfer

| Tool | Purpose | Data Size | Scope | Effort | Exam Keyword |
|------|---------|-----------|-------|--------|---|
| **Migration Center** | Discover, assess, plan 6 Rs, TCO estimate, wave planning | Any | VM inventory, dependencies, cost projection | Low (agents + analysis) | "Assessment," "TCO," "recommendations," "cost estimate" |
| **Migrate to Virtual Machines** | VM migration (replication, test-clone, cutover) from VMware / AWS / Azure to Compute Engine | Any (replication-based) | VMs → Compute Engine | Low–medium | "Lift-and-shift," "VM migration," "test-clone" |
| **Database Migration Service** | DB migration with continuous replication (homogeneous and heterogeneous, e.g. Oracle → PostgreSQL with conversion workspace), low-downtime cutover | Terabyte | Cloud SQL, AlloyDB (not Spanner / Firestore) | Medium | "Continuous replication," "schema conversion," "minimal-downtime cutover" |
| **Datastream** | CDC (change data capture), real-time ELT to BigQuery | Continuous stream | BigQuery, Cloud Storage | Low–medium | "Real-time sync," "CDC," "BigQuery load" |
| **Storage Transfer Service** | Managed online data copy from on-prem (agents) / AWS / Azure / other clouds / HTTP to Cloud Storage; scheduled and incremental | Any size (bandwidth-bound) | Cloud Storage | Low | "Online transfer," "S3 → GCS," "scheduled / recurring copy" |
| **Transfer Appliance** | Offline physical appliance (TA40 = 40 TB, TA300 = 300 TB usable), shipped data import | Online upload would take more than about a week | Cloud Storage | Medium (shipping delay) | "Tens or hundreds of TB," "slow link (< 10 Mbps)," "offline" |
| **Cloud Interconnect** | Hybrid networking, on-prem ↔ GCP high-throughput | N/A (networking) | Networking | Medium–high | "10–100 Gbps," "hybrid," "on-prem link," "low latency" |
| **VMware Engine** | Lift entire VMware private cloud (vSphere, NSX, HCX). | Millions GB (entire DC) | VMware cluster → GCP | Medium | "VMware," "vCloud," "HCX," "entire cluster" |
| **BigQuery Data Transfer** | Scheduled ETL (not raw data movement) | Any | Recurring data pipelines → BigQuery | Low | "Scheduled ETL," "recurring," "pipeline" |

**Pick it when:** Migration Center = "start here (assess first)"; Migrate to Virtual Machines = "VMs"; Database Migration Service = "databases"; Storage Transfer Service = "online data"; Transfer Appliance = "upload would take over a week, offline"; VMware Engine = "VMware estate"; BigQuery Data Transfer = "recurring pipelines".

### Online vs offline data transfer (size/bandwidth rule of thumb as stated in Ch07)

**Formula (ideal link, decimal units, 100 % utilisation):** hours = data in GB x 8,000 / bandwidth in Mbps / 3,600. Real links rarely sustain more than 60-80 % of nominal bandwidth, so add margin.

| Data size | Bandwidth | Ideal transfer time | Typical choice |
|---|---|---|---|
| 50 GB | 100 Mbps | about 1.1 hours | Online (Storage Transfer Service or gcloud storage) |
| 500 GB | 100 Mbps | about 11 hours | Online |
| 5 TB | 100 Mbps | about 111 hours (4.6 days) | Online if the link can be dedicated, otherwise offline |
| 5 TB | 1 Gbps | about 11 hours | Online |
| 100 TB | 1 Gbps | about 222 hours (9.3 days) | Over a week at ideal speed: Transfer Appliance, unless the link is dedicated and the delay is acceptable |
| 100 TB | 100 Mbps | about 2,222 hours (93 days) | Offline (Transfer Appliance) |
| 1 PB | 1 Gbps | about 2,222 hours (93 days) | Offline (Transfer Appliance, possibly several) |

**Rule of thumb:** work out the transfer time first. If it is days or weeks, or the link is shared with production traffic, use Transfer Appliance (offline); otherwise use Storage Transfer Service (online). Ch07 gives Google's rule of thumb: consider Transfer Appliance when the online upload would take more than about a week.

**Pick online when:** the calculated transfer time is acceptable (hours to a few days) on a link you can dedicate.
**Pick offline when:** the online upload would take more than about a week, or the link is slow or shared (for example < 10 Mbps).

## Load balancer comparison

### Load balancer types

| Type | Scope | Layer | Protocol | External | Proxy | Multi-region | Best for |
|------|-------|-------|----------|----------|-------|------|----------|
| **HTTP(S) LB** | Global | L7 | HTTP/HTTPS | Yes | Yes | Yes | Web apps, REST APIs, host/path-based routing, WAF |
| **SSL Proxy LB** | Global | L4 | TLS (non-HTTP) | Yes | Yes | Yes | SMTP, POP3, global HA for non-HTTP TLS |
| **TCP Proxy LB** | Global | L4 | TCP (non-HTTP) | Yes | Yes | Yes | MQTT, gaming, global HA non-HTTP protocols |
| **Network LB (external)** | Regional | L4 | TCP/UDP | Yes | No (pass-through) | No | Ultra-low latency, high throughput, non-HTTP |
| **Internal Network LB** | Regional | L4 | TCP/UDP | No (VPC-only) | No (pass-through) | No (single-region) | VPC-internal services, minimal latency |
| **Internal HTTP(S) LB** | Regional (a cross-region internal Application LB also exists) | L7 | HTTP/HTTPS | No (VPC-only) | Yes | Regional by default | VPC-internal microservices, path/host routing |

**Network Service Tier:** the global external Application LB and the global external proxy Network LB (TCP/SSL proxy) require Premium Tier; Standard Tier is limited to regional external LBs. Internal LBs are not tied to the tiers. Cloud Armor attaches to external Application LBs and proxy Network LBs (and to passthrough Network LBs via network edge security policies).

**Pick it when:** Global + HTTP/HTTPS → HTTP(S) LB; Global + non-HTTP → TCP/SSL Proxy; Regional + lowest-latency non-HTTP → Network LB; VPC-internal + L7 → Internal HTTP(S) LB; VPC-internal + lowest-latency → Internal Network LB; in-line third-party appliance (NGFW, IDS/IPS) → internal passthrough Network LB used as a next hop (there is no separate "Gateway LB" type on Google Cloud).

### Load balancer decision picker

| Requirement | Decision Path |
|---|---|
| **Is app global (multi-region)?** | Yes → Investigate HTTP(S), SSL Proxy, TCP Proxy. No → regional options below. |
| **Does app need Layer 7 routing (Host, Path, TLS termination)?** | Yes → HTTP(S) LB (external) or Internal HTTP(S) LB. No → Layer 4. |
| **Is protocol HTTP/HTTPS?** | Yes → HTTP(S) LB. No → Layer 4 (TCP Proxy, SSL Proxy, Network LB). |
| **Is target internal (VPC-only) or external (internet)?** | Internal → Internal Network LB (fastest) or Internal HTTP(S) LB (if L7 needed). External → External LB. |
| **Is latency critical (<10 ms end-to-end) with non-HTTP?** | Yes → Network LB (pass-through, ultra-low). No or HTTP → HTTP(S) LB. |

---

## Hybrid connectivity comparison

### Cloud VPN, Interconnect, Cross-Cloud comparison

| Aspect | Cloud VPN (HA) | Dedicated Interconnect | Partner Interconnect | Cross-Cloud Interconnect | Direct Peering / Carrier Peering |
|--------|---|---|---|---|---|
| **Bandwidth** | Up to 3 Gbps per tunnel (scale out with more tunnels) | 10 Gbps or 100 Gbps circuits (up to 8 x 10 G or 2 x 100 G per bundle) | 50 Mbps–50 Gbps per VLAN attachment (partner-dependent) | 10 Gbps or 100 Gbps | N/A (peering, public Google services only) |
| **SLA / Availability** | 99.99 % (HA VPN with both interfaces / 2+ tunnels to redundant peer) | 99.9 % (2 connections, one metro); 99.99 % (4 connections, 2 metros) | 99.9 % (2 attachments, one metro); 99.99 % (4 attachments, 2 metros) | Same 99.9 / 99.99 % topologies as Interconnect | No SLA |
| **Encryption** | IPsec (encrypted by default) | Not encrypted by default; MACsec available, or HA VPN over Interconnect | Not encrypted by default; HA VPN over Interconnect for IPsec | MACsec available | Not encrypted |
| **Latency** | Internet-dependent, variable | Low and predictable (set by distance to the colocation facility) | Low and predictable (adds partner hop) | Low and predictable | Depends on peering location |
| **Setup time** | Days (DNS, IPsec config) | Weeks (fiber provisioning) | Days–weeks (via partner) | Weeks (partner + fiber) | Weeks–months |
| **Cost model** | Per GB egress + per-tunnel hour fee | Per Gbps; highest per-service at scale | Middle ground; per Gbps via partner | Premium; per-Gbps + setup | Per-peer; negotiated rates |
| **Best for** | Hybrid (branch office, moderate bandwidth <10 Gbps), low capex | Enterprise hybrid (high throughput, mission-critical), latency-sensitive (autonomous vehicles, scientific computing) | Enterprises without direct fiber access, geographic flexibility | Regulated multi-cloud, consistent connectivity across clouds | Direct peer-to-peer exchange, high-volume peering |
| **NOT for** | High throughput (>10 Gbps), latency-critical | Small branch office, low-bandwidth, cost-sensitive, no presence in a colocation facility | Not available in region/country | Clouds other than the supported providers (AWS, Azure, Oracle, Alibaba) | Reaching VPC private IPs or anything needing an SLA (Google recommends Interconnect) |

**Pick it when:** Branch office + VPN cheaply → Cloud VPN; Enterprise hybrid + high perf/throughput → Dedicated Interconnect; No colocation presence or need under 10 Gbps → Partner Interconnect; Private link to another cloud (AWS / Azure / OCI / Alibaba) → Cross-Cloud Interconnect; Public Google services (e.g. Workspace) over a peering link → Direct/Carrier Peering.

### Interconnect topology for 99.9 % vs 99.99 % availability

| Availability Target | Architecture | Redundancy | Config |
|---|---|---|---|
| **99.9 %** | 2 Interconnect connections (or 2 VLAN attachments) in the same metro | Two different edge availability domains of one metro | 2× Dedicated Interconnect connections (or 2× Partner VLAN attachments) in different edge availability domains; Cloud Router in the region. A single connection has no SLA |
| **99.99 %** | 4 Interconnect connections (or 4 VLAN attachments) across 2 metros | Two connections in each of two metros, each pair in different edge availability domains | Two Cloud Routers (one per region), VPC in global dynamic routing mode; same layout for Dedicated and Partner. HA VPN alone also reaches 99.99 % (two interfaces, 2+ tunnels to a redundant on-prem peer) |

### Hybrid connectivity option picker

| Use Case | Recommended | Reasoning |
|---|---|---|
| **Branch office, <10 Mbps, cost-sensitive** | Cloud VPN (HA) | Cheapest, acceptable for low-bandwidth WAN |
| **Regional HQ, 100 Mbps–1 Gbps, moderate latency tolerance** | Cloud VPN (HA) or Partner Interconnect | VPN simpler; Partner Interconnect if partner available + lower latency critical |
| **Headquarters, >1 Gbps, low and predictable latency, mission-critical** | Dedicated Interconnect (4 connections, 2 metros for 99.99 %) | Highest throughput and private path; Partner Interconnect can also reach 99.99 % with the same topology |
| **Multiple clouds (AWS / Azure / OCI), private high-bandwidth link** | Cross-Cloud Interconnect | Google-provisioned link to the other cloud; MACsec available; traffic stays off the internet |
| **No colocation presence, need up to 50 Gbps per attachment** | Partner Interconnect | Service provider provisions; 99.99 % needs 4 attachments across 2 metros |
| **Reach public Google services (e.g. Workspace) from an existing peering presence** | Direct/Carrier Peering | Public IPs only, no SLA; not for private VPC connectivity |

---

## VPC and service connectivity comparison

### VPC Network Peering vs Shared VPC vs Network Connectivity Center vs Private Service Connect vs Private Services Access

| Method | Scope | Transitive? | Overlapping CIDR? | Centralized Admin? | Setup Effort | Best for |
|---|---|---|---|---|---|---|
| **VPC Network Peering** | 2 VPCs (same or diff org) | No (direct only) | No; CIDRs must differ | No; each project configures | Minutes (mutual) | Federation, multi-team, partner integrations, low-touch |
| **Shared VPC** | Host + Service projects (same org) | N/A (one VPC shared by many projects) | No; all in host VPC | Yes (host project admin) | Hours (hierarchy + IAM) | Large org, centralized security, network audit, cost control |
| **Network Connectivity Center** | Hub + Spokes (VPC spokes, hybrid spokes: VPN / Interconnect / SD-WAN) | Yes between spokes (via the hub's route exchange) | No for VPC spokes (subnet ranges must not overlap) | Yes (hub-centric) | Hours (hub setup) | Multi-VPC + on-prem hub-and-spoke routing, site-to-site transfer over Google's backbone |
| **Private Service Connect** | Endpoint ↔ Google APIs or a published service (service attachment) | N/A (point-to-point, one direction: consumer to producer) | Yes (consumer and producer IP ranges may overlap) | Producer publishes, consumer controls its endpoint | Minutes (endpoint creation) | Private access to Google APIs or third-party / internal published services without peering, with overlapping IPs, from on-prem too |
| **Private Services Access (PSA)** | VPC ↔ Google-managed service producer VPC (Cloud SQL, Memorystore, AlloyDB, etc.) | No (VPC peering underneath; non-transitive unless custom routes are exported) | No (reserved allocated range must not overlap) | No | Hours (allocate range + create connection) | Cloud SQL / Memorystore / AlloyDB private IP; PSC is preferred where the service supports it |
| **Cloud NAT** | Outbound VPC traffic | N/A (NAT only, no unsolicited inbound) | N/A (masquerades source) | Per-VPC router | Hours (Cloud Router + NAT) | Outbound internet for VMs / GKE nodes without external IPs, egress IP control (not for on-prem traffic) |
| **Private Google Access (PGA)** | VPC subnet → Google APIs (Cloud Storage, Pub/Sub) | N/A (egress only) | N/A (not peering) | Per-subnet enable | Minutes (subnet setting) | VMs without external IPs reach Google APIs; use PSC endpoints when you need private IPs for APIs, on-prem access, or VPC-SC bundles |
| **Serverless VPC Access Connector** | Cloud Run/Functions ↔ VPC | N/A (appliance bridge) | N/A (connector in VPC) | Per-connector config | Hours (connector provisioning, extra VMs) | Older serverless-to-VPC path; Direct VPC egress is the preferred option for Cloud Run |
| **Direct VPC Egress** | Cloud Run/Functions native VPC routing | N/A (egress via VPC) | N/A (uses VPC CIDR) | Per-service config | Minutes (service setting) | Modern serverless, VPC networking, Cloud NAT egress, hybrid connectivity |

**Pick it when:** Federation + low admin → VPC Peering; Centralized org + many projects → Shared VPC; Multi-VPC + on-prem hub → NCC; Private access to Google APIs or a published service with overlapping IPs → Private Service Connect; Managed-service private IP (Cloud SQL etc.) → PSA (or PSC where supported); Older serverless connector → migrate to Direct VPC egress; Cloud Run needing VPC → Direct VPC egress; Outbound internet from private VMs → Cloud NAT.

### Firewall layers

| Layer | Scope | Enforcement | When to use |
|---|---|---|---|
| **Hierarchical Firewall Policies** | Organization or Folder | Mandatory allow + deny; inherited by all VPCs below | Org-wide mandatory rules: block external DB ports, enforce TLS, deny public IPs |
| **Network Firewall Policies** | Network (global or regional policy attached to a VPC) | Policy-based rules with secure tags, FQDN, geo-location and threat-intelligence match; replace legacy VPC firewall rules | Grouping many rules and applying them to several VPCs; tag-based and FQDN rules |
| **VPC Firewall Rules** | Project/VPC | Allow/Deny per rule; match on CIDR, port, protocol, tags, SAs | App-level fine-grained control: service-account-based, tag-based rules |
| **Network Policies (Kubernetes)** | GKE cluster pods | Pod-level ACLs (east-west + north-south) | Pod isolation in multi-tenant clusters; Calico or GKE Dataplane V2 |
| **Cloud Armor** | External Application LB (also proxy Network LBs) | WAF + DDoS at Google edge (before VPC) | HTTP/HTTPS threats, geo-blocking, rate-limiting, Adaptive Protection |
| **Cloud IDS / Cloud NGFW** | VPC (east-west) / Appliance | Signature-based threat detection / next-gen stateful inspection | Detect lateral threats (IDS), replace legacy firewall appliances (NGFW) |

### DNS options

| Type | Scope | Use | Cost |
|---|---|---|---|
| **Cloud DNS (Public)** | Internet-facing domains | Authoritative DNS for your domains; integrates with GCP services | ~$0.20/zone/month |
| **Cloud DNS (Private)** | VPC-scoped | Internal DNS; VPC sees internal IPs; can be shared across VPCs | Same |
| **Cloud DNS (Forwarding)** | VPC → on-prem | Forward queries to on-prem DNS servers | Same |
| **Cloud DNS (Peering)** | VPC A ↔ VPC B DNS | Resolve names from peer VPC without peering entire networks | Same |

---

## Encryption and key management comparison

### Google-managed vs CMEK vs Cloud HSM vs Cloud EKM vs CSEK vs client-side

| Encryption Type | Who Holds Key | Key Rotation | Audit Trail | Compliance Fit | When to Use |
|---|---|---|---|---|---|
| **Google-managed (default)** | Google | Automatic (Google-managed) | Minimal | Non-regulated, cost priority | Non-sensitive data, no compliance mandate |
| **CMEK (Cloud KMS, software)** | Customer-controlled key in Cloud KMS (Google hosts it) | Configurable automatic rotation (e.g. 90 days) or manual | Full (key use logged in Cloud Audit Logs; can disable / destroy key) | Policy requires customer control of keys, key audit, crypto-shredding | "Customer controls key," key audit needed; works with most Google services |
| **CMEK (Cloud KMS, HSM mode)** | Customer-controlled key in Cloud HSM (FIPS 140-2 Level 3) | Configurable automatic or manual | Full + HSM attestation | Regimes requiring HSM-protected keys | "HSM required"; same CMEK integration, protection level = HSM |
| **CMEK with Cloud KMS Autokey** | Customer-controlled; keys created on demand by Autokey in a central key project | Same as Cloud KMS | Full | Many projects, want CMEK without manual key-ring / IAM setup | "Automate CMEK provisioning," "separation of duties between key admins and developers" |
| **Cloud EKM (External Key Manager)** | Customer (key stays in external key manager outside Google) | Customer-managed | Full + external audit trail | On-prem mandate, multi-cloud, key never in Google | "Key must never touch Google infrastructure," external key manager requirement |
| **CSEK (Customer-Supplied Encryption Key)** | Customer (supplied with each API request; Google does not store it) | Customer-managed | Limited | Supported ONLY for Cloud Storage and Compute Engine (disks, images); lose key = lose data | "Supply my own key per request" for GCS / GCE only |
| **Client-side (application-level)** | Customer (app/client) | Customer-managed | Customer-managed | Highest control (beyond GCP) | Encrypted blob storage, app owns encryption/decryption |

**Pick it when:** Non-sensitive, low cost → Google-managed; Customer-controlled key + audit → CMEK (software KMS); HSM-protected key → CMEK HSM; Many projects, automate CMEK → Autokey; Key held outside Google → EKM; Supply key per request (GCS / GCE only) → CSEK; App-level encryption → Client-side.

### Secrets and sensitive-data tools

| Tool | Purpose | Who Manages | Rotation | Exam Keyword |
|---|---|---|---|---|
| **Secret Manager** | Store and version API keys, DB passwords, OAuth tokens | Customer | Rotation schedule publishes a Pub/Sub notification; your function creates the new version (Secret Manager does not rotate the backend credential itself) | "Rotate secrets every 90 days", "credential management", "API key rotation" |
| **Cloud KMS** | Encrypt data at rest; manage encryption keys | Customer | Configurable automatic schedule (e.g. 90 days) or on-demand | "Customer-managed encryption key", "audit key access", "key versioning", "transparent encryption" |
| **Sensitive Data Protection (DLP)** | Find, redact, mask PII (credit cards, SSN, email, phone) in data | Customer (inspection + de-id rules) | N/A (inspection jobs, discovery scans or API calls you configure) | "Detect PII", "redact sensitive data", "format-preserving encryption", "mask credit cards" |
| **Certificate Manager** | Manage SSL/TLS certificates | Customer or Google-managed (auto-renewal) | Automatic (Google-managed cert) or manual (BYO cert) | "Certificate renewal", "self-signed cert lifecycle", "managed SSL" |

### Perimeter and data-exfiltration controls

| Control | Layer | Blocks | When to use |
|---|---|---|---|
| **IAM (identity-based)** | API level | Who can call an API (based on role) | Baseline; not sufficient for data exfiltration |
| **VPC Firewall Rules** | Network level (packet) | Network traffic by CIDR/port/protocol | Prevent unauthorized network access; does not understand APIs |
| **VPC Service Controls** | API level (perimeter) | API calls in/out of perimeter; blocks data export even if IAM allows | Prevent insider exfiltration, rogue code; block egress to external APIs |
| **Organization Policy** | Resource configuration | What resources can be created (e.g., public buckets, unencrypted disks) | Enforce compliance constraints (e.g., no external public IPs) |
| **Private Google Access / PSC** | Egress control | Routes internal IP to Google APIs (no public internet) | Reduce surface area; private API access |
| **Access Context Manager (Context-Aware Access)** | Device + network posture | Access based on device encryption, OS version, location, MFA | BeyondCorp zero-trust (granular conditional access) |
| **Confidential Computing** | Data-in-use | Encrypts memory + vCPU while workload runs | Process trade secrets, genomics, financial models (highest privacy) |

**Pick it when:** Identity → IAM; Network perimeter → VPC Firewall; API-level exfiltration → VPC-SC; Compliance constraint → Org Policy; Google API privacy → Private Google Access/PSC; Device trust + location → Context-Aware Access; Process secrets → Confidential Computing.

### Securing AI controls (from Ch11)

| Control | Purpose | When to use | Cost / Overhead |
|---|---|---|---|
| **Model Armor** | Screen prompts and model responses: prompt injection / jailbreak, harmful content, malicious URLs, sensitive data (via Sensitive Data Protection). Access control to models stays with IAM | Any LLM / agent app, including models outside Google; apply via templates and floor settings | Separately priced; integrates with Security Command Center ⚠ verify pricing |
| **Sensitive Data Protection on prompts** | Inspect / de-identify user input before sending to Gemini APIs; mask PII, block card numbers | Any app calling Gemini APIs with user data | Priced by data volume inspected |
| **CMEK for AI models** | Encrypt trained models, datasets and endpoints at rest with your own key | Regulated training (HIPAA, GDPR) | Standard Cloud KMS key and operation charges |
| **VPC-SC for Agent Platform** | Prevent data exfiltration from Gemini/Agent Platform APIs | Prevent insider exfiltration; data residency enforcement | No direct cost; perimeter management |

---

## IAM and identity comparison

### Role types and policy types

| Item | Type / Policy | Scope | When to use | Example |
|---|---|---|---|---|
| **Basic role** | Viewer, Editor, Owner (legacy, very broad) | Project | Avoid in production; use predefined or custom roles | N/A (legacy) |
| **Predefined role** | Google-curated (e.g., roles/compute.instanceAdmin.v1) | Service-specific | Most common; standard job functions | `roles/bigquery.dataEditor`, `roles/storage.admin` |
| **Custom role** | User-defined bundle of permissions | Org, folder, or project | Exact least-privilege fit | `roles/custom.deploymentEngineer` (deploy functions, no IAM changes) |
| **Allow policy** | Grant role to principal on resource | Org/Folder/Project/Resource | Enable access; default policy type | `Group: data-team@company.com → roles/bigquery.dataEditor on project X` |
| **Deny policy** | Forbid specific permissions for principals (evaluated before allow, always wins) | Org/Folder/Project | Guardrails that must hold regardless of grants (with exceptions listed) | `Deny compute.instances.delete for everyone except the platform-admins group` |
| **Organization Policy** | Enforce constraints on resource configuration (not role-based) | Org/Folder/Project | Prevent creation of non-compliant resources | `storage.uniformBucketLevelAccess = true` (enforce uniform bucket-level access, no object ACLs) |
| **Principal Access Boundary (PAB)** | Limit the resources a set of principals can use, even if allowed by an allow policy (ceiling) | Applied to principal sets (e.g. groups, workforce pool, Workspace domain) via a binding; org-level | Contain a population of principals to specific projects / folders | `Contractor principal set can only access resources inside folder X` even if granted broader roles elsewhere |
| **IAM Condition** | Attribute-based conditions on a role binding: request time, resource name / type / tags, access levels | Role grant | Temporary access, tag-based access, context-aware permissions | `Role valid only until 2026-12-31 and only on resources tagged env=dev` |

**Pick it when:** Job function → Predefined role; Exact fit required → Custom role; Block action → Deny policy; Compliance constraint → Org Policy; Multi-tenant isolation → PAB; Temporary access → IAM Condition.

### Identity options for humans and workloads

| Identity Type | Lifespan | Auth Method | When to use | Exam Keyword |
|---|---|---|---|---|
| **Cloud Identity (users)** | Persistent | Email + password + MFA | All GCP users; centralized directory | "Centralized user management", "SSO", "MFA enforcement" |
| **Google Workspace (users)** | Persistent | Gmail + Workspace apps + SSO | Orgs with Gmail/Docs/Sheets/Drive | "Email + productivity", "unified collaboration" |
| **Workforce Identity Federation** | Federated | External IdP (Okta, Microsoft Entra ID) token via OIDC / SAML | Contractors, partners with their own IdP | "External users", "partner SSO", "external IdP integration" |
| **Workload Identity Federation (external)** | Short-lived (1 h) | OIDC token (GitHub, AWS, on-prem) | Apps outside GCP; CI/CD pipelines (GitHub Actions → GCP) | "GitHub Actions deploy to GCP without keys", "cross-cloud apps", "OIDC token exchange" |
| **Service accounts** | Persistent | Key (legacy), Workload Identity, Impersonation | Apps on VMs/GKE, Cloud Run, functions | "Non-interactive identity", "app auth", "least-privilege SA per workload" |
| **Service account impersonation** | Short-lived (1 h) | User/SA impersonates another SA | User acting on behalf of SA; temporary access | "Temporary cross-team access", "audit trail of impersonation" |
| **GKE Workload Identity** | Short-lived (pod lifetime) | Kubernetes SA ↔ GCP SA binding + signed JWT | GKE pods needing GCP APIs | "Pod authenticates to GCP without keys", "automatic credential refresh" |

**Pick it when:** Humans → Cloud Identity or Workspace; External contractors → Workforce WIF; External apps/CI/CD → Workload Identity Federation; Non-interactive workload → Service account (with Workload Identity or impersonation); Temporary elevation → Service account impersonation.

### Remote access options

| Option | Setup | Audit Trail | Cost | Best for |
|---|---|---|---|---|
| **IAP (Identity-Aware Proxy)** | IAP TCP forwarding for SSH/RDP (user needs IAP-secured Tunnel User role; firewall allows 35.235.240.0/20) or IAP-protected web apps | Built-in Cloud Audit Logs (user identity + timestamp) | No extra IAP charge | Modern GCP-native; VMs without external IPs; internal web apps; no bastion or VPN |
| **OS Login** | Enable on project/instance; Linux access granted through IAM roles (roles/compute.osLogin, osAdminLogin); SSH keys tied to the Google identity | Centralised in Cloud Audit Logs; optional 2-step verification | No extra charge | Centralised, IAM-managed Linux access and key lifecycle (works together with IAP) |
| **Bastion Host** | Jump VM in VPC; user SSH to bastion, then target | Manual (logs in bastion); not audit-friendly | A VM to run and patch | Legacy; prefer IAP or OS Login |
| **Cloud VPN / Interconnect** | Site-to-site connectivity between on-prem network and VPC (not a per-user client VPN) | Network-level (limited user detail) | Tunnel / attachment hours + egress | Hybrid; on-prem users reach VPC private IPs from the corporate network |
| **Context-Aware Access (BeyondCorp)** | Chrome Enterprise + device posture checking | Device + network + MFA; risk-based auth | Chrome Enterprise Premium | Zero-trust; device compliance + location required before SSH |

**Pick it when:** Cloud-only + no keys → IAP; Centralized SSH key mgmt + MFA → OS Login; Hybrid on-prem user → Cloud VPN/Interconnect; Zero-trust device check → Context-Aware Access.

### Resource hierarchy inheritance rules

| Scenario | Inheritance Rule | Example |
|---|---|---|
| **Viewer at Org, no role at Project** | Inheritance downward; user keeps Viewer at project | Org: User = Viewer → Project: User inherits Viewer |
| **Editor at Project, Viewer at Org** | Additive; user has both Editor (project) + Viewer (org) = Editor | User gets max of both (superset) |
| **Deny policy at Org, Allow at Project** | Deny wins (deny overrides allow) | Org: Deny compute.instances.delete for all → Project: Allow overridden → Action blocked |
| **Org Policy constraint at Org** | Inherited by all child folders/projects by default; a lower level can merge with or replace it only if someone holds the Org Policy Administrator role there (use "enforced" at the top and restrict who can override) | Org Policy: enforce public access prevention → applies to all child projects unless explicitly overridden |

---

## Compliance and governance tools

### Governance tool picker

| Tool | Purpose | When to use | Output |
|---|---|---|---|
| **Organization Policy** | Enforce compliance constraints (resource location, disable APIs, require encryption) | Prevent non-compliant resource creation | Blocks violating API calls; audit log of denied attempts |
| **Assured Workloads** | Managed compliance control packages (e.g. FedRAMP Moderate/High, CJIS, ITAR, IL4/IL5, EU data sovereignty, HIPAA) enforcing data location, personnel and key controls | Regulated workloads; auto-applies org policy constraints, location and support-personnel controls | Compliance monitoring; violation alerts |
| **Security Command Center** | Unified security posture + threat detection + findings | Aggregate vulnerabilities, misconfigurations, active threats | Findings list (severity + remediation); advanced threat detection in Premium / Enterprise tiers |
| **Cloud Audit Logs** | Track all API calls (admin, data, system, policy-denied) | Compliance audit trail; incident investigation | Queryable logs (30–400 day retention); integration with SIEM |
| **Access Transparency** | Logs of Google's internal access to customer data (for support, debugging) | Regulated workloads; see when Google staff access your content (pair with Access Approval to require approval first); needs a qualifying support plan | Access Transparency log entries in Cloud Logging |
| **Policy Intelligence** | Recommender (unused roles), Policy Troubleshooter (permission debug), Analyzer (blast radius) | Least-privilege audits, permission troubleshooting, policy testing | Recommendations, permission simulation results, access graph |
| **Cloud Asset Inventory** | Real-time snapshot of all resources + configs + relationships | Governance inventory, compliance reporting, change tracking | Asset list + metadata (exportable to BigQuery) |
| **Cloud Logging sinks + buckets** | Route logs to Cloud Storage, BigQuery, Pub/Sub or another log bucket for long-term archive | Compliance retention (e.g., 7-year financial audit logs), BI querying | Logs routed to sinks; queryable, immutable (with Bucket Lock) |

### Audit log types

| Log type | Content | Default | Retention | Chargeable | When to enable |
|---|---|---|---|---|---|
| **Admin Activity** | IAM changes, resource create/delete, API calls that modify config | ON (cannot disable) | 400 days (_Required bucket) | No | Always on; free |
| **Data Access** | Reads of data and metadata, user-driven writes (Cloud Storage, Spanner, Bigtable, etc.) | OFF by default (except BigQuery, always on); enable per service | 30 days in _Default bucket (configurable) | Yes (log ingestion and storage charges apply) | Regulatory or audit need to see who read data; enable selectively |
| **System Event** | Google-initiated system actions that modify resources (e.g. live migration) | ON (cannot disable) | 400 days (_Required bucket) | No | Understand Google-initiated changes (informational) |
| **Policy Denied** | Requests denied by VPC Service Controls or other security policy | ON by default (can be excluded) | 30 days in _Default bucket | Yes (ingestion into _Default) | Security investigations; track blocked attempts |

---

## Observability comparison

### Cloud Monitoring vs Cloud Logging vs Cloud Trace vs Cloud Profiler vs Error Reporting vs Managed Prometheus vs Cloud Audit Logs

| Tool | Signal Type | Scope | Cost Model | Best for |
|---|---|---|---|---|
| **Cloud Monitoring** | Metrics (time-series) | All GCP + on-prem (via Ops Agent) | Google Cloud system metrics free; custom / external / Prometheus metrics charged by volume ingested | SLO monitoring, infrastructure dashboards, alerting |
| **Cloud Logging** | Logs (structured or unstructured) | All GCP services | ~$0.50/GB ingested (after free tier) | Audit, debugging, compliance retention, app logs |
| **Cloud Trace** | Traces (distributed request latency) | Apps instrumented with OpenTelemetry | Free tier, then charged per span ingested | Latency breakdown per hop, RCA of slow requests |
| **Cloud Profiler** | Profiles (CPU/memory hotspots) | Compute Engine, GKE, Cloud Run | Free | Identify CPU-bound / memory-hungry code |
| **Error Reporting** | Error signals (app exceptions) | App logs (ingested from Cloud Logging) | Free | Deduplicate errors, track trends, group by stack trace |
| **Managed Prometheus** | Metrics (OSS Prometheus format) | Kubernetes (GKE) + on-prem | Charged per sample ingested | Prometheus-native monitoring (custom instrumentation) |
| **Cloud Audit Logs** | Audit events (IAM, config, policy denials) | All GCP | Admin Activity / System Event free; Data Access charged as log ingestion and storage | Compliance audit trail, security investigation |
| **Gemini Cloud Assist** | AI-driven insights, anomaly detection | Monitoring data + logs + traces | Pricing per Google Cloud terms ⚠ verify | Troubleshooting recommendations, anomaly alerts (AI-powered) |

### Alert types and SLO concepts

| Alert Type | Trigger | When to use | Pitfall |
|---|---|---|---|
| **Metric threshold** | CPU > 80 %, disk > 90 %, latency > 500 ms | Infrastructure health | Alert fatigue if threshold not SLO-aligned |
| **Log-based metric** | Count(5xx errors) > 10/min | Error rate trending | Dedup similar errors first to avoid noise |
| **SLO burn-rate alert** | Error budget burn ≥ 10× (user-facing SLO breach imminent) | Symptom-based alerting | Requires SLO setup first; less false positives than threshold |
| **Uptime check** | GET /health endpoint → 200 OK | External availability | May pass while internal latency high; not comprehensive |
| **Absence alert** | No log entry in 5 min | Detect stalled jobs / missing heartbeat | False positives if job is legitimately idle |
| **Custom (PromQL / ratio query)** | User-defined query (e.g., error_count / request_count > 0.01) | Complex logic (ratios, composite metrics) | Query syntax errors silently never fire; test the query in Metrics Explorer |

**SLO Concepts:**
- **SLI (Service Level Indicator):** Measured metric (e.g., % of requests < 200 ms, availability %).
- **SLO (Service Level Objective):** Target for SLI (e.g., 99.9 % availability).
- **SLA (Service Level Agreement):** Contractual commitment + penalty (e.g., "99.99 % uptime or refund").
- **Error budget:** 1 − SLO = budget to spend. E.g., SLO 99.9 % → 43.2 min downtime/month; if burned early, scale back deploys.
- **Burn-rate alert:** Measures if you're on track to exhaust error budget. E.g., a 10× burn rate on a 30-day SLO window exhausts the whole budget in 3 days (urgent); 1× burns it exactly at the end of the window.

### Which signal answers which question

| Question | Signal(s) to check |
|---|---|
| "Is my app slow?" | **Latency metric** (p50, p95, p99) + Cloud Trace (breakdown by service hop) |
| "Why is it slow?" | **Traces** (request path + latency per hop), **Profiler** (CPU/memory hotspots) |
| "Are users getting errors?" | **Error rate metric** (% 5xx) + Error Reporting (deduplicated exceptions) |
| "Which errors?" | **Cloud Logging** (filter by error code, service, timestamp) + stack traces |
| "Did this deploy break anything?" | **SLO burn-rate alert** (user-facing), **change correlation** (release notes + metric drop) |
| "What changed?" | **Cloud Audit Logs** (who deployed, when) + **GKE event logs** (pod restart, version change) |
| "Is my database healthy?" | **Metrics** (query latency, connection count, replication lag) + **Logs** (slow query log) |

---

## IaC and CI/CD comparison

### Terraform vs Config Connector vs Infrastructure Manager vs Deployment Manager vs gcloud scripts vs Helm

| Tool | Language | State Management | Learning Curve | Multi-cloud | When to use |
|---|---|---|---|---|---|
| **Terraform** | HCL | Cloud Storage backend (gcs) + locking | Moderate | Yes (AWS, Azure, GCP, K8s) | Gold standard; most flexible; enterprise standard |
| **Config Connector** | Kubernetes Custom Resources (YAML) | Kubernetes etcd | Moderate | No (GCP only) | GKE-native teams; sync K8s ↔ GCP resources |
| **Infrastructure Manager** | Terraform (HCL) configs run as a managed Google service (uses Cloud Build) | State stored by the service (Cloud Storage) | Low if you know Terraform | Deploys Google Cloud resources only | Teams that want Terraform without running their own pipeline / state backend; successor to Deployment Manager |
| **Deployment Manager (end of support 31 Mar 2026)** | Python/Jinja/YAML templates | Google-managed | Low | No (GCP only) | Retired: migrate to Infrastructure Manager or Terraform; never pick for new work |
| **gcloud / gsutil scripts** | Bash/Python + CLI | N/A (imperative, no state) | Low | No (GCP only) | One-off ops, scripting; not for production infrastructure |
| **Helm** | YAML (charts) + templating | Kubernetes release versions | Moderate | Yes (any K8s) | Kubernetes app package manager; templates app configs |

**Pick it when:** Multi-cloud or enterprise → Terraform; GKE-only → Config Connector; Terraform as a managed Google service (Deployment Manager replacement) → Infrastructure Manager; App package mgmt → Helm; One-offs → gcloud scripts.

### Cloud Build vs Cloud Deploy vs Artifact Registry vs Binary Authorization vs Cloud Source / Developer Connect vs Skaffold

| Tool | Purpose | Triggers | Output | Exam Weight |
|---|---|---|---|---|
| **Cloud Build** | CI: compile, test, SAST, build artifact (container, Maven, npm) | Branch, tag, PR, manual, Pub/Sub, schedule | Artifact to registry | High (core CI tool) |
| **Cloud Deploy** | CD: approval gates, canary, auto-rollback, traffic splitting | Artifact ready (Cloud Build post-push); manual trigger | Deployment to target (GKE, Cloud Run; other platforms via custom targets) | High (core CD tool; emphasised by recent test-takers) |
| **Artifact Registry** | Central artifact repo (containers, Maven, npm, Python, Docker, Helm) | Pushed by Cloud Build | Stored artifact (versioned, scannable) | High (core repository) |
| **Binary Authorization** | Policy enforcement: only signed images deployable to GKE / Cloud Run | Image push to registry | Deploy approval/denial | Medium-High (supply-chain security) |
| **Cloud Source Repositories** | Git repository on GCP | Code push (same as any Git) | Webhooks to Cloud Build | Low (closed to new customers; use GitHub / GitLab with Developer Connect or Cloud Build triggers) |
| **Developer Connect** | GitHub / GitLab / Bitbucket integration | PR, push, tag | Webhook to Cloud Build | Low (newer; unified Git integration) |
| **Skaffold** | Dev loop: file watch, rebuild, redeploy to dev cluster | File saves locally | Updated container in dev cluster | Low (dev tool, not exam-heavy) |

### API management: Apigee vs API Gateway vs Cloud Endpoints vs Application Integration

| Platform | Use Case | Monetisation | Rate Limiting | Analytics | Hybrid | Best for |
|---|---|---|---|---|---|---|
| **Apigee** | Public APIs, B2B, partner ecosystems | ✅ (quotas, spike arrest, revenue sharing) | ✅ (sophisticated: per-app, per-user, per-product) | ✅ (rich: flow debugging, latency breakdown, error analysis) | ✅ (hybrid for on-prem APIs) | B2B monetisation, complex policies, ecosystem |
| **API Gateway** | Simple REST APIs in front of serverless backends (Cloud Run, Cloud Run functions, App Engine), OpenAPI-defined | ❌ | Basic quotas (OpenAPI quota config) | Basic (Cloud Monitoring / Logging) | ❌ | Lightweight, quick deployment, API-key / JWT auth for serverless backends |
| **Cloud Endpoints + ESP** | gRPC + REST, auth / JWT / API keys via the ESP / ESPv2 proxy | ❌ | Basic quotas | Basic (Monitoring) | ✅ (ESP container can run on GKE, Compute Engine, on-prem) | gRPC-first, internal microservices |
| **Application Integration** | Not an API gateway: low-code integration workflows with connectors, triggers (API, Pub/Sub, schedule) | ❌ | N/A | Execution logs | Connectors reach on-prem / SaaS | Process integration, connector-heavy flows |
| **Model Context Protocol (MCP) via Apigee** | Expose existing APIs as MCP tools for agents, with Apigee security and governance | N/A | ✅ (Apigee policies) | ✅ (Apigee analytics) | Follows the Apigee deployment | Agents consuming enterprise APIs |

**Pick it when:** B2B + monetisation → Apigee; Simple REST + quick → API Gateway; gRPC + low-latency → Cloud Endpoints; Connector-based integration (not an API proxy) → Application Integration; Agent integration → Apigee (MCP bridge).

---

## Pricing and discount options

### On-demand vs Sustained Use Discounts vs Committed Use Discounts vs Spot VMs vs Reservations vs Flex / DWS

| Model | Cost Reduction | Upfront | Min Term | Flexibility | Best for |
|---|---|---|---|---|---|
| **On-demand (pay-as-you-go)** | 0 % | $0 | None | Full | Ad-hoc, unpredictable, spike handling |
| **Sustained Use Discounts (SUD)** | Automatic, up to ~20–30 % for a full month of use | $0 | None | Full (automatic) | Only older Compute Engine families (e.g. N1, N2, N2D, C2, M1, M2; GKE Standard nodes on them). No SUD on E2, C3, N4 etc., and none on Cloud SQL |
| **Committed Use Discounts (CUD), 1-year spend-based (flexible)** | ~28 % off for most general-purpose / compute families (Compute Engine docs) | Billed monthly, no upfront | 1 yr | Flexible across eligible machine series and regions; other services (Cloud SQL, Cloud Run, Spanner, etc.) have their own spend-based CUDs | Steady baseline spend that may change shape |
| **Committed Use Discounts (CUD), 3-year spend-based (flexible)** | ~46 % off for most general-purpose / compute families (Compute Engine docs) | Billed monthly, no upfront | 3 yrs | Flexible across eligible machine series and regions | Long-term baseline, flexible shape |
| **Committed Use Discounts (CUD), resource-based (1-year)** | Roughly similar to or slightly above flexible ⚠ verify exact % | Billed monthly, no upfront | 1 yr | Rigid (specific machine type / vCPU / memory amount in a region) | Predictable compute (vCPU, memory) |
| **Committed Use Discounts (CUD), resource-based (3-year)** | Highest for Compute Engine (above flexible 46 %; up to ~63 % for memory-optimised M-series) | Billed monthly, no upfront | 3 yrs | Rigid (specific resources in a region) | Maximum savings, predictable steady state |
| **Spot VMs** | Roughly 60–91 % off | $0 | None (interruptible) | Can be preempted at any time with a 30 s notice; no 24 h max runtime | Batch jobs, training, fault-tolerant workloads, cost-sensitive |
| **Reservations (capacity)** | No discount by themselves (billed at on-demand until a CUD is attached) | No upfront | Until deleted (future-dated reservations possible) | Medium (can resize) | Guaranteed capacity (not cost reduction) |
| **Dynamic Workload Scheduler (DWS) Flex Start** | Discounted vs on-demand (Spot-like pricing) ⚠ verify exact % | $0 | None (request queues until capacity is found; runs up to 7 days) | High | AI/batch jobs with flexible start that need all-or-nothing GPU capacity |
| **Dynamic Workload Scheduler (DWS) Calendar mode** | Capacity assurance for a fixed window (no commitment-style discount) ⚠ verify | $0 upfront | Fixed reservation window (days to ~3 months) | None (fixed time) | Training runs that must start at an exact time on guaranteed accelerators |

**Pick it when:** Unpredictable → On-demand; Steady baseline across changing VM shapes → flexible (spend-based) CUD; Fixed steady resources → resource-based CUD; Fault-tolerant batch → Spot VMs; Flexible-start AI jobs → DWS Flex Start; Guaranteed accelerator window → DWS Calendar; Capacity guarantee → Reservation.

### FinOps tools and cost optimization per service family

| Category | Tool / Method | Use Case |
|---|---|---|
| **Cost visibility** | Billing export to BigQuery | Query costs by label, service, project (custom dashboards) |
| **Cost visibility** | Cloud Billing reports / Cost Table | Pre-built cost breakdowns by service, resource, time period |
| **Budgets & Alerts** | Billing Budgets + alerts | Email/Pub/Sub alert when spend crosses a threshold; budgets only notify, so capping spend needs a Pub/Sub-triggered function that acts (e.g. disables billing) |
| **Anomaly detection** | Billing anomaly alerts | Auto-detect cost spikes; unusual patterns |
| **Rightsizing** | Recommender (Compute sizing) | Under-utilised VMs → downsize recommendations |
| **Storage tiering** | Autoclass + lifecycle policies | Auto-tier cold data to Archive; reduce per-GB cost |
| **Network optimization** | Premium vs Standard tier, CDN, Private Service Connect | Premium tier: lower latency, higher cost; Private endpoints: reduce egress |
| **Labels & allocation** | Resource labels + billing export | Split cost across teams/projects/environments |
| **Quotas** | Service quotas / quota overrides (not Org Policy) | Cap resource consumption per project to limit runaway spending |
| **Carbon Footprint** | Carbon Footprint report (Console, export to BigQuery) | Track gross CO₂e emissions per project, service, region (sustainability reporting) |

---

## AI service picker

### Prebuilt AI APIs vs Gemini API vs Custom Model

| Option | What | Effort | Data Needed | When |
|--------|------|--------|-------------|------|
| **Pre-built APIs** (Vision, Video, Speech, Document AI, Discovery AI, Conversation) | Single-task, domain-specific models | Low (API call, format input) | Any (image, audio, video, text) | Off-the-shelf task (invoice extraction, video labeling, product discovery); quick POC; minimal ML expertise |
| **Gemini API** (Gemini 3.x Pro / Flash / Flash-Lite, thinking models) | General-purpose, multi-modal LLM; text generation, reasoning, function calling | Low–Medium (prompt eng, grounding setup) | None (zero-shot) or domain docs (RAG) | Conversational, summarization, code generation, agent backbone; needs reasoning or context-awareness; pick Flash-Lite/Flash when latency and cost matter most |
| **Agent Platform fine-tuning** (Gemini models) | Adapt a Gemini model to domain language/style (supervised tuning) | Medium (curate labelled examples, tune) | Curated labelled examples (hundreds to thousands) | Custom terminology, domain-specific tone, high-accuracy classification; try prompting/RAG first |
| **Custom training** (PyTorch/TensorFlow/JAX on Agent Platform) | Full model build; any architecture | High (data prep, hyperparameter tuning, monitoring) | Large labelled dataset | Competitive differentiator; unique loss function; proprietary data; avoid if a pre-built API or Gemini already solves the problem |
| **Agent Platform AutoML** (image/text/tabular/video) | Google-managed training; minimal code | Low–Medium (data upload, wait) | Labelled dataset in a supported format | Quick baseline with no ML engineering; less control; avoid if custom logic needed |

**Pick it when:** Pre-built if task is narrow (OCR, face detect); Gemini API if conversational/reasoning/agent backbone; fine-tuning if domain language critical; AutoML if rapid prototyping; custom training if scale/uniqueness demands it.

### Agent Platform components (formerly Vertex AI)

| Component | Purpose | Latency | Setup | Cost |
|-----------|---------|---------|-------|------|
| **Model Garden** | Access 200+ models (Gemini, Claude, Llama, DeepSeek, Imagen, Veo); MaaS (serverless API) or self-deployed endpoints | Varies by model/serving option | Browse, deploy, or pay-per-request | MaaS: pay-per-token; Endpoint: compute + ops; Provisioned Throughput: fixed $/month |
| **Agent Studio** | No-code prompt design, tuning, testing (formerly Vertex AI Studio) | — | Point-and-click | Studio itself is a console tool; model calls billed per usage |
| **Agent Builder / ADK** | Multi-step agent framework; open-source Agent Development Kit (ADK) or low-code builder; A2A protocol | — | Code (ADK) or low-code builder | Ops + Agent Platform compute |
| **Agent Runtime** (formerly Agent Engine) | Managed runtime for agents; long-running operations up to 7 days; sub-second cold starts | Sub-second cold start | Custom container or Python agent code | Usage-based compute |
| **RAG Engine** | Managed RAG orchestration; chunking, embedding, retrieval | — | Documents uploaded; auto-indexed | Embedding/vector store + model usage |
| **Vector Search** (formerly Matching Engine) | Managed, low-latency approximate-nearest-neighbour search on embeddings you generate | Low (ms-level at scale) | Create index from embeddings + deploy to an index endpoint | Index serving nodes + storage |
| **Agent Search** (formerly Vertex AI Search) | Managed enterprise search/RAG over websites, structured and unstructured data; grounding | — | Configure data stores | Per query / edition-based |
| **Agent Platform Pipelines** (formerly Vertex AI Pipelines) | ML lifecycle orchestration; Kubeflow/TFX-based; serverless | Varies (task-dependent) | Define DAG in Python/YAML | Compute per task |
| **Feature Store** | Centralized feature management; BigQuery-based offline store + online serving | Real-time + batch | Register BigQuery feature tables/views | Managed serving ops + BigQuery |
| **Model Registry** | Version, alias, promote trained models; metadata | — | Register after training | Managed storage |
| **Endpoints** (online & batch) | Serve any model; autoscaling; traffic split | Online: real-time; Batch: asynchronous (not latency-sensitive) | Deploy container or model artifact | Node-hour based (online) or per job (batch); Provisioned Throughput for Gemini |
| **Experiments** | Track training runs; hyperparameter comparison; A/B testing | — | Built into training jobs | Included in training cost |
| **Colab Enterprise** | Managed Jupyter; Gemini code assist built-in; instant compute | Fast startup | Notebooks + GCP auth | Runtime compute billing |
| **Workbench** | Managed JupyterLab notebook instances (more control over the VM than Colab Enterprise) | — | Launch from console | VM hourly rate |

**Pick it when:** Model Garden for model access; Agent Studio for prompt tuning; ADK for custom agent logic; Agent Runtime for long-ops; RAG Engine for managed knowledge bases; Vector Search for low-latency vector retrieval; Pipelines for ML automation; Endpoints for production serving; Colab Enterprise for development.

### Gemini Enterprise vs NotebookLM vs Gemini Code Assist vs Gemini Cloud Assist

| Product | Audience | Purpose | Data Residency | Audit |
|---------|----------|---------|-----------------|-------|
| **Gemini Enterprise** | Knowledge workers (all employees) | AI-powered copilot; org-wide access to Gemini + AI agents; integrated with Workspace (Gmail, Drive, Docs) | Configurable per region; DLP integration | Cloud Audit Logs; org policy controls |
| **Gemini Enterprise App** | End users | Mobile + web surface; AI agents; search org data | Org-level residency policy | Audit Logs; session logging |
| **AI Agents in Gemini Enterprise** | Business users | Pre-built templates (research, analysis, customer support); no code; human handoff | Private knowledge + web grounding | Audit Logs; agent decision traces |
| **Workspace Studio** | Citizen developers | No-code agent builder within Gemini Enterprise; drag-and-drop; Workspace integration; human approval | Org-level | Audit Logs |
| **NotebookLM Enterprise** | Researchers, analysts, knowledge workers | Long-form document analysis; auto-generate outlines, summaries, Q&A, podcast; collaborative | Per-user or org-level | Audit Logs; document versioning |
| **Gemini Code Assist** | Software engineers | IDE plugin (VS Code, JetBrains, Cloud Shell Editor); code completion, generation, explanation; GitHub code review | Prompts/code sent to Google for inference; not used to train models for paid editions ⚠ verify | Cloud Audit Logs for admin/API activity; Standard vs Enterprise editions (Enterprise adds private-codebase customization) ⚠ verify |
| **Gemini Cloud Assist** | Architects, ops engineers, SREs | Design assistance, troubleshooting, cost and security insights; built into Cloud Console | Processed by Google; not used to train models on customer data ⚠ verify | Cloud Audit Logs |

**Pick it when:** Gemini Enterprise for org-wide AI knowledge work; NotebookLM for document research/synthesis; Gemini Code Assist for development velocity; Gemini Cloud Assist for ops/architecture decisions; Workspace Studio for rapid agent prototyping.

### AI infrastructure picker

| Choice | Criteria | Options | Trade-off |
|--------|----------|---------|-----------|
| **Compute Hardware** | Large-scale LLM training, large-scale inference, cost-optimized, latency-sensitive inference | **TPU** (Trillium / Ironwood; best throughput/cost at large scale, JAX/PyTorch-XLA) / **TPU v5e** (cost-optimized) / **GPU A3 Ultra / A4** (flexible, CUDA/PyTorch ecosystem) / **GPU L4** (inference, light fine-tuning) / **CPU** (preprocessing) | TPUs: narrower software ecosystem, capacity must be planned; GPUs: broadest framework support, high power/cost at scale |
| **Serving Platform** | Real-time low-latency, batch non-real-time, bursty traffic, scale-to-zero | **Agent Platform Endpoints** (managed, autoscaling) / **GKE** (multi-model, custom orchestration) / **Cloud Run** (serverless, scale-to-zero, GPU supported) / **Cloud Run functions** (pre/post-processing) | Endpoints: opinionated; GKE: ops overhead; Cloud Run: cold starts when scaled to zero |
| **Consumption** | Predictable steady-state, ad-hoc unpredictable, batch wait-tolerant, strict SLA guarantees | **On-demand** / **Spot** (roughly 60–91 % off, preemptible) / **Committed Use Discount** (flexible or resource-based, 1 or 3 yr) / **Reservations** (capacity assurance) / **DWS Flex Start** (queued, wait-for-capacity) / **DWS Calendar** (guaranteed future window) / **Provisioned Throughput** (reserved Gemini capacity) / **Batch API** (discounted, asynchronous) | On-demand: expensive; Spot: can be preempted; CUD: commitment; DWS Flex: wait time; Provisioned: fixed spend |

**Pick it when:** Large-scale LLM training → TPU or GPU cluster (A3 Ultra/A4); PyTorch/CUDA-centric fine-tuning → GPU; small inference needing low latency → L4 GPU or TPU; async batch inference → Batch prediction / DWS Flex; steady inference → CUD or Provisioned Throughput; bursty/scale-to-zero → Cloud Run (GPU); fault-tolerant training → Spot; wait-tolerant jobs → DWS Flex Start.

### RAG option comparison

| Option | Retrieval Latency | Accuracy (relevance) | Setup | Integration | Cost |
|--------|-------------------|----------------------|-------|-------------|------|
| **RAG Engine** (managed) | Managed orchestration (seconds incl. generation) | Depends on chunking/embedding config | Managed; upload docs | Gemini API, Agent Platform agents | Embedding + vector store + model usage |
| **Vector Search** (managed index) | Low (ms-level ANN at scale) | Depends on embedding quality | Generate embeddings; create and deploy index | Custom agent code or RAG Engine backend | Index serving nodes + storage |
| **Agent Search** (managed, formerly Vertex AI Search) | Managed | Good out of the box (Google-quality search, ranking) | Configure data stores (websites, docs, structured data) | Gemini API, Agent Platform agents | Per query / edition |
| **Google Search Grounding** | Adds a search call per prompt | Public web only; up-to-date | Enable on the request; no index to build | Gemini API | Billed per grounded prompt beyond free quota ⚠ verify |
| **BigQuery Vector Search + SQL** | Moderate | High for structured/analytical data | Ingest data into BigQuery; embeddings column + vector index | SQL or Agent Platform | BigQuery compute (slots/on-demand) |
| **AlloyDB / Cloud SQL pgvector** | Low | High (transactional consistency, SQL filters) | PostgreSQL with pgvector (AlloyDB adds ScaNN index) | SQL driver or custom agent code | Managed DB pricing |
| **Self-run open-source vector DB** (e.g. Chroma) | Low (local) | Variable | Embedded in agent, or separate service | In-process or via API | Compute only; you operate it |

**Pick it when:** RAG Engine for a managed RAG pipeline with citations; Vector Search for custom, high-QPS low-latency retrieval; Agent Search for enterprise/website search with minimal build; BigQuery for analytics + semantic search; AlloyDB/Cloud SQL pgvector for transactional data + vectors; Google Search grounding for public-web freshness; open-source vector DB for dev/small scale.

### Responsible AI and Securing AI control picker

| Control | Pillar | Option | When | Notes |
|---------|--------|--------|------|-------|
| **Input validation** | Security | Model Armor (prompt-injection/jailbreak screening) + Sensitive Data Protection (DLP) pre-flight + function schema constraints | Prevent prompt injection; detect PII in user input | Custom regex too fragile; DLP detects PII infoTypes; Model Armor screens prompts and responses |
| **Prompt hardening** | Security | System instruction tuning; avoid dynamic concatenation | Reduce injection surface; use function calling instead of free-text concatenation | Don't rely on instruction alone; layer with DLP |
| **Output filtering** | Security | Sensitive Data Protection redact/inspect post-response | Prevent data exfiltration in model output | Latency-sensitive applications: pre-cache DLP policies |
| **Data residency** | Compliance | Org Policy resource-location constraint + VPC-SC perimeter on Agent Platform | GDPR/data sovereignty | Gemini API honors constraint; fine-tuning/training data stays in region |
| **Audit trail** | Compliance | Cloud Audit Logs (Gemini API calls logged) + Model Cards + Session logs | Explainability for regulators, ML governance | Agent Platform Sessions log all interactions |
| **Human-in-the-loop** | Governance | Agent Platform Pipelines conditional approval; Workspace Studio manual review gate | High-risk decisions (hiring, medical, legal) | Humans review before deployment, not just monitoring |
| **Fairness metrics** | AI governance | Fairness Indicators, What-If Tool, slice evaluation in Pipelines | Detect demographic bias before production | Compute during training; monitor during serving |
| **Model explainability** | Governance | Agent Platform Explainable AI (feature attributions: Sampled Shapley, Integrated Gradients, XRAI) + Model Cards | Understand feature importance; regulatory requirement | Applies to custom/AutoML models; Model Cards document intended use and limits |
| **Evaluation & red-teaming** | Security | Gen AI evaluation service / evaluation jobs in Pipelines; internal red-team exercises; Model Armor | Adversarial robustness; prompt injection resistance | Continuous; not one-time pre-launch |

**Pick it when:** Sensitive data in prompts → DLP pre-flight; function calling critical → constrain schema; regulatory audit → Cloud Audit Logs; fairness required → What-If + slice evaluation; explainability → Explainable AI + Model Cards; production agents → human-in-loop + red-teaming + Model Armor.

---

## Decision trees

### Compute choice

- **What are you running?**
  - Needs OS/kernel control, custom licensing, lift-and-shift, special hardware? → **Compute Engine** ✓
  - Containers that need the Kubernetes API/ecosystem (operators, DaemonSets, service mesh, multi-cluster)? → **GKE** (Autopilot by default; Standard for node-level control) ✓
  - Stateless container/HTTP service, bursty, scale-to-zero, no cluster to manage? → **Cloud Run** ✓
  - Small single-purpose event-driven function? → **Cloud Run functions** ✓
  - Run-to-completion container task? → **Cloud Run jobs** ✓
  - Large queued HPC/batch with scheduling and retries on VMs? → **Batch** ✓
  - Model/agent inference needing GPUs/TPUs? → **Agent Platform Endpoints**, **GKE** or **Cloud Run with GPU** ✓
  - Managed runtime for agents (long-running operations up to 7 days)? → **Agent Runtime** ✓

### Storage & database choice

- **Structured relational data, ACID critical?**
  - Yes: → **Scale & consistency?**
    - Single-region (HA but not multi-region)? → **Cloud SQL** ✓
    - Multi-region strong consistency? → **Cloud Spanner** ✓
    - PostgreSQL compatibility + ML features (pgvector)? → **AlloyDB** ✓
  - No: → Type of data?
    - Document/JSON flexibility, real-time, mobile? → **Firestore** ✓
    - Wide-column time-series (IoT, telemetry)? → **Bigtable** ✓
    - In-memory cache, atomic counters? → **Memorystore (Redis/Valkey)** ✓
    - Unstructured (media, logs, archives)? → **Cloud Storage** ✓
      - Cost tiering and/or WORM retention needed? → **Autoclass or lifecycle rules + retention policy (Bucket Lock)** ✓
    - Data warehouse, analytics at scale? → **BigQuery** ✓

### Load balancer choice

- **Protocol & layer?**
  - HTTP/HTTPS (Layer 7, URL/host/header routing)? → **Application Load Balancer**
    - External, global anycast (Premium Tier), Cloud CDN/Armor/IAP? → **Global external Application LB** ✓
    - External but single region / data residency / Standard Tier? → **Regional external Application LB** ✓
    - Internal clients only (VPC/hybrid)? → **Internal Application LB** (regional or cross-region) ✓
  - TCP/SSL (Layer 4 proxy, non-HTTP)? → **Proxy Network Load Balancer** (global external, regional external or internal) ✓
  - TCP/UDP/ESP/ICMP, client IP preserved, direct server return? → **Passthrough Network Load Balancer** (external or internal) ✓
  - DNS-based routing only? → **Cloud DNS routing policies** (not a load balancer) ✓

### Hybrid connectivity choice

- **Throughput & latency sensitivity?**
  - Quick/low-cost, encrypted over the internet, up to a few Gbps (per-tunnel limit ~3 Gbps), 99.99 % with HA VPN? → **HA VPN** ✓
  - 10 or 100 Gbps circuits, private path, you can reach a Google colo? → **Dedicated Interconnect** ✓
  - Cannot reach a Google colo, or need 50 Mbps–50 Gbps via a provider? → **Partner Interconnect** ✓
  - Multi-cloud (GCP ↔ AWS, Azure, Oracle) private link? → **Cross-Cloud Interconnect** ✓
  - BGP dynamic routing for any of the above? → **Cloud Router** (required) ✓
  - Many sites/VPCs in hub-and-spoke or site-to-site transfer over Google's backbone? → **Network Connectivity Center** ✓

### Disaster recovery pattern choice

- **RTO requirement (max downtime)?**
  - Many hours to days acceptable, minimal budget? → **Backup & restore (cold DR)** ✓
  - Tens of minutes to a few hours, core data replicated, compute built on failover? → **Pilot light** ✓
  - Minutes, scaled-down stack already running in a second region? → **Warm standby** ✓
  - Near-zero RTO/RPO, full stacks serving in 2+ regions? → **Hot / active-active** ✓

### Messaging & event processing choice

- **What is the interaction pattern?**
  - Decouple producers/consumers, fan-out, streaming ingest? → **Pub/Sub** (at-least-once by default; exactly-once for pull subscriptions; ordering keys; dead-letter topics; replay via seek) ✓
  - Route events from Google services/SaaS/Pub/Sub to Cloud Run, GKE, Workflows? → **Eventarc** ✓
  - Dispatch individual async tasks to an HTTP target with rate limits and retries? → **Cloud Tasks** (asynchronous only, not for synchronous lookups) ✓
  - Time-based trigger (cron)? → **Cloud Scheduler** ✓
  - Orchestrate multi-step API/service calls with branching and retries? → **Workflows** ✓
  - Batch ETL DAGs on a schedule with dependencies? → **Cloud Composer (Apache Airflow)** ✓
  - Real-time streaming transforms/analytics? → **Dataflow (Apache Beam)**, or a **Pub/Sub BigQuery subscription** for direct load ✓

### Migration approach (6 Rs + tool)

- **Workload type & risk tolerance?**
  - Rehost (lift-and-shift)? → **Migrate to Virtual Machines** (or **VMware Engine** for VMware workloads) ✓
  - Replatform (light modernization)? → **Database Migration Service (MySQL/PostgreSQL)** or **Datastream** ✓
  - Refactor (containerize)? → **Cloud Build + Artifact Registry + GKE** ✓
  - Repurchase (SaaS alternative)? → Evaluate on Google Cloud Marketplace ✓
  - Retire (deprecate)? → Plan decommission, data archive to Cloud Storage ✓
  - Retain (on-prem stays)? → **Hybrid connectivity (VPN/Interconnect)**; use **Workload Identity Federation** for keyless access ✓
  - **Assessment & TCO?** → **Migration Center** (not Pricing Calculator; it estimates costs + phased plan) ✓

### AI approach (prebuilt API vs Gemini vs RAG/agent vs custom training)

- **Data & use case?**
  - Off-the-shelf task (OCR, face detection, translation)? → **Pre-built API** ✓
  - Conversational, reasoning, multi-step, needs context? → **Gemini API + RAG** ✓
    - Lowest latency/cost, simple tasks? → **Gemini Flash-Lite / Flash** ✓
    - Complex reasoning? → **Gemini Pro (thinking)** ✓
  - Multi-step agent, tool calling, orchestration? → **Agent Platform (ADK + Agent Runtime, models from Model Garden) with a Gemini backbone** ✓
  - Domain-specific language, custom accuracy that prompting/RAG cannot reach? → **Supervised fine-tuning, then custom training if needed** ✓
  - Knowledge base, private docs? → **RAG Engine or Vector Search** ✓

### AI hardware choice (training vs inference, scale vs cost)

- **Workload phase?**
  - Large-scale distributed LLM training? → **TPU (Trillium/Ironwood) or GPU clusters (A3 Ultra/A4)** ✓
  - Fine-tuning or custom training with a CUDA/PyTorch stack? → **GPU (A3 Ultra / A4 / L4)** ✓
  - Latency-critical inference? → **GPU (L4 and up) or TPU** ✓
  - Cost-optimized or bursty inference? → **TPU v5e, or Cloud Run with GPU (scale-to-zero)** ✓
  - Preprocessing, feature engineering? → **CPU (general-purpose / C-series)** ✓
  - Fault-tolerant or wait-tolerant jobs? → **Spot** (roughly 60–91 % off, can be preempted) or **DWS Flex Start** (queued) ✓

### Identity & access choice (human vs workload, cross-cloud)

- **Principal type?**
  - End users (humans)? → **Cloud Identity + SAML/OIDC federation** ✓
    - Microsoft Active Directory? → **Cloud Identity + GCDS or Workforce WIF** ✓
    - Organizational users, SSO? → **Cloud Identity + SAML idP (Okta, Ping, etc.)** ✓
  - Services & applications (workloads)? → **Service accounts** ✓
    - GCP-to-GCP (incl. cross-project)? → **Attached service account (no keys) or service account impersonation**; avoid downloaded keys ✓
    - Cross-cloud (AWS, Azure)? → **Workload Identity Federation** (exchange AWS/Azure token for GCP SA) ✓
    - GKE Pod? → **GKE Workload Identity** (OIDC-based, no keys) ✓
  - Secure remote access (jump hosts, bastion)? → **IAP (Identity-Aware Proxy)** ✓
  - Machine learning model serving? → **Service account attached to Endpoint; DLP pre/post-processing** ✓

---

## Keyword to answer lookup

### Design and Planning

| If the question says | Think | Not |
|---|---|---|
| "Minimize operational overhead" | Managed services (Cloud SQL HA, Cloud Spanner, Agent Platform Pipelines, Cloud Run), not self-managed (GKE custom scaling, custom Kubernetes). | GKE, Cloud Build custom scripts, VMs with manual patch mgmt. |
| "99.9% or 99.99% availability required" | Multi-region failover (Cloud Spanner, Cloud Load Balancer), HA per tier (Cloud SQL HA, Spanner replicas), not single-region. | Single-region with replicas; backup-restore only. |
| "Global, strongly consistent relational data" | Cloud Spanner multi-region (99.999% SLA), external consistency. | Cloud SQL (regional HA only), eventual consistency. |
| "Cost-optimized data warehouse" | BigQuery (on-demand vs capacity-based editions with slot autoscaling, partitioning/clustering, automatic long-term storage pricing). | Self-managed warehouse cluster; over-provisioned fixed capacity. |
| "Real-time inventory across continents" | Cloud Spanner (low-latency reads via read replicas), Firestore (real-time sync), not polling Cloud SQL. | Cloud SQL with batch sync; polling Bigtable. |
| "Data residency (GDPR, HIPAA)" | Org Policy location constraint, VPC Service Controls perimeter, single-region storage, not cross-region replication by default. | Cross-region replication enabled by default; IAM IP restrictions only. |
| "Compliance audit, every API call logged" | Cloud Audit Logs (Admin, Data Access, System Event), Data Access logging on Cloud Storage/BigQuery, not just application logs. | Application-level logging only; Cloud Monitoring metrics. |
| "Build fast POC, minimal ML expertise" | Pre-built APIs (Vision, Document AI, Conversation API) or Gemini API, not custom training. | Custom PyTorch pipeline, Dataflow preprocessing, feature engineering. |
| "Conversational AI, multi-turn stateful" | Agent Platform Sessions + Memory Bank; Gemini API with structured context, not single-request predictions. | Agent Studio (no-code prompting); one-off Gemini.generate() calls. |
| "Sub-second cold-start agent" | Agent Runtime (managed agent runtime with sub-second cold starts, custom containers). | Cloud Batch, scheduled Dataflow jobs. |
| "Inference on-demand, bursty, zero idle cost" | Cloud Run services (scale to zero) + L4 GPU, Batch Endpoint, not always-on Endpoints. | Always-on dedicated endpoints (minimum spend); GKE with reserved nodes. |
| "High-performance ML training, multi-node" | TPU Trillium or A3 Ultra GPU (distributed training, allreduce), DWS Flex Start for Kueue orchestration, not single-machine. | Compute Engine n2-standard, Cloud Batch without GPU. |

### Networking and Compute

| If the question says | Think | Not |
|---|---|---|
| "Lowest possible latency between services" | Same region and zone (compact placement policy), not cross-region. | Cross-region calls; routing via on-prem over VPN/Interconnect. |
| "No public IPs, all private" | Private IPs only on VMs, Private Google Access / Private Service Connect for Google APIs, Cloud NAT for outbound-only internet. | External IPs on VMs; public endpoints. |
| "Hybrid on-prem + cloud, 10+ Gbps, low latency" | Dedicated or Partner Interconnect with Cloud Router (BGP); a VPN tunnel is capped near 3 Gbps and rides the internet. | Direct Peering (public Google services only, not private VPC traffic). |
| "Multi-cluster management, auto-discovery" | GKE Enterprise fleets (multi-cluster services/gateway, Config Sync), not per-cluster configs. | Separate Kubernetes clusters, manual endpoint mapping. |
| "Automatic failover based on health" | Load balancer health checks and backend failover; Cloud DNS routing policies with health checks for DNS-level failover. | Manual detection and cutover. |
| "Container registry, secure, compliance-audited" | Artifact Registry (GCP-native, IAM, vulnerability scanning, retention policies), not Docker Hub or self-hosted. | Cloud Storage + Docker images; public registries. |
| "Zero-trust, every request authenticated" | Identity-Aware Proxy (IAP) on load balancers, VPC Service Controls, not IP allowlists. | IP address whitelisting (Cloud Armor); SSL/TLS only. |

### Storage, Data and Analytics

| If the question says | Think | Not |
|---|---|---|
| "Wide-column, low-latency time-series (IoT, metrics)" | Bigtable (row-key = timestamp, device ID), not Cloud SQL. | Cloud SQL; Firestore (not optimized for sequential time-series). |
| "Append-only audit log, immutable" | Cloud Storage with Bucket Lock (retention policy prevents deletion), not Cloud SQL. | Cloud SQL with triggers; Firestore with retention rules. |
| "Data archival, minimum 7-year retention, cost-optimized" | Cloud Storage Archive class (or lifecycle/Autoclass) + retention policy locked with Bucket Lock, not active queries. | BigQuery (expensive for archival); Cloud SQL (not designed for archives). |
| "Real-time collaborative documents, mobile sync" | Firestore (real-time listeners, offline-first), not Cloud SQL. | Cloud SQL + polling client; Bigtable (no mobile SDKs). |
| "Scalable data pipeline, 1M+ events/sec" | Pub/Sub (autoscaling) → Dataflow (streaming) → BigQuery, not Cloud SQL or manual ETL. | Cloud SQL with batch inserts; Cloud Composer (scheduled); Dataproc (batch only). |
| "Fast analytics on multi-billion-row data lake" | BigQuery (OLAP, columnar, no schema change required), not OLTP DB. | Cloud SQL; Spanner (optimized for transactions, not analytics). |
| "Backup & point-in-time restore" | Cloud SQL automated backups + point-in-time recovery (within the log-retention window) + replicas, not BigQuery (versioning via snapshots). | Snapshots only (not true backups); manual exports. |
| "Feature engineering for ML, consistent train/serve" | Feature Store (online + batch serving), not BigQuery View. | BigQuery views (no real-time serving); hand-rolled code. |
| "Real-time, low-latency vector similarity search at high QPS" | Vector Search (managed ANN index) or AlloyDB pgvector (when vectors sit beside transactional data). | RAG Engine alone (an orchestration layer); BigQuery vector search (analytics-oriented). |
| "Private knowledge base for agents, strict governance" | RAG Engine or Agent Search over your own data stores, not Google Search Grounding (public web). | Google Search Grounding (public web); Vector Search (self-managed embeddings). |

### Security and Compliance

| If the question says | Think | Not |
|---|---|---|
| "Encrypt keys, rotate quarterly, audit every use" | Cloud KMS (HSM mode for compliance, Autokey for ease), log all key operations in Audit Logs. | Google-managed keys (no rotation control); in-app encryption (no key lifecycle). |
| **"Least privilege, deny by default"** | IAM roles (predefined or custom), Deny policies, resource hierarchy (org/folder/project), not allow-all then subtract. | One large "Editor" role; IP allowlists. |
| "Prevent data exfiltration to public internet" | VPC Service Controls (perimeter on Agent Platform, Cloud Storage), Private Service Connect, no public endpoints. | Cloud NAT (still egress, just NATed); IP whitelists. |
| "Regulatory audit, every data access logged" | Cloud Audit Logs (Data Access logs on BigQuery/Cloud Storage), Bucket Lock audit trail, not application logs. | Application log only; single Cloud Logging sink. |
| "PII protection, redaction in logs" | Sensitive Data Protection to inspect/de-identify before or after logging, log exclusion filters to drop fields/entries, and tight IAM on log buckets/views. | Manual redaction; storing PII in application configs. |
| "Enforce that services use customer-managed keys" | CMEK (Autokey for automation) plus Org Policy `constraints/gcp.restrictNonCmekServices` (and `gcp.restrictCmekCryptoKeyProjects` to limit key projects). | CSEK (only Cloud Storage and Compute Engine); Google-managed keys default. |
| "Secure model serving, no data exfiltration from inference" | VPC-SC perimeter on Agent Platform Endpoint, Custom prediction routine with DLP, private endpoints only. | Public Endpoint; inference logging to external sinks. |
| "Human-in-the-loop approval before AI model deployment" | Agent Platform Pipelines conditional approval gate, Workspace Studio manual review, not just CI/CD gates. | Automatic deploy after training; unit tests only. |
| **"Prompt injection from user input, prevent"** | Model Armor screening of prompts/responses, function schema constraints, least-privilege tool credentials; add Sensitive Data Protection for PII; NOT just instruction tuning. | System instruction: "never reveal secrets" (injectable via prompt). |
| **"Cross-cloud secret sharing (AWS → GCP)"** | Workload Identity Federation (exchange AWS STS token for GCP service account), NOT application-managed secrets. | Shared credential files; multi-cloud IAM federation. |

### Operations, DevOps and Cost

| If the question says | Think | Not |
|---|---|---|
| "Reduce CapEx, move to OpEx" | Cloud (pay-as-you-go, no hardware purchase) vs on-prem; CUD for steady workloads, Spot for batch. | Buying hardware; over-provisioning for peak. |
| "Cost-optimize compute for ML training" | Spot VMs (deep discount, can be preempted), DWS Flex Start (wait-tolerant), not always on-demand. | Always-on reservations for bursty jobs; on-demand only. |
| "Immediate CI/CD, short deployment windows" | Cloud Build + Cloud Deploy (managed delivery pipeline with approvals, canary, rollback) for GKE/Cloud Run, not manual steps. | Ad-hoc kubectl apply; Deployment Manager (end of support 31 Mar 2026); script-based. |
| "Canary deployment, SLO-driven rollback" | Cloud Deploy canary strategy + Cloud Monitoring SLO burn-rate alerts to drive rollback, not blue-green (instant switch). | Blue-green (binary switch); rolling updates (no SLO feedback). |
| "Infrastructure-as-code, versioned, auditable" | Terraform (Infrastructure Manager for managed runs) with Git (GitHub/GitLab/Developer Connect), Config Connector for Kubernetes-style management of GCP resources, not imperative gcloud. | gcloud scripts; CloudFormation (AWS-only); manual console. |
| "Prevent config drift, enforce policy" | Org Policy constraints (enforcement + deny policies), Config Connector (desired state), not CIS Benchmarks alone. | CIS checks (audit only); manual compliance tracking. |
| "Monitor and alert on SLO burn" | Symptom-based alerts (error rate, latency) + SLO burn-rate calculation, not metric thresholds. | Static thresholds ("alert if CPU > 70%"); anomaly detection. |
| "Reduce observability data, lower logging costs" | Log exclusion filters (drop non-critical logs at ingest), BigQuery log sinks for analysis, not collecting everything. | Cloud Logging all traffic; no filtering; Stackdriver archives. |
| "Fastest incident response, root-cause analysis" | Cloud Trace + Logging + Error Reporting correlation, Cloud Profiler for code hot spots, NOT just dashboards. | Cloud Monitoring dashboards; post-mortem analysis only. |
| "Right-size infrastructure, auto-scale based on load" | Compute Engine rightsizing recommendations, VPA/HPA/Cluster Autoscaler (GKE), MIG autoscaler, not manual capacity planning. | Static sizing; manual VM additions. |

### AI and GenAI

| If the question says | Think | Not |
|---|---|---|
| "Agent with external tool access, function calling" | Gemini function calling + schema validation, MCP servers, NOT separate LLM + if-then logic. | Hard-coded if-else routing; LLM → parse output → call tool. |
| "Agent needs to remember conversation state" | Agent Platform Sessions + Memory Bank (or ADK session services), NOT prompt concatenation. | Prepend history to every request (context explosion, cost). |
| "RAG with strict data governance, audit trail" | RAG Engine (managed source control) or Vector Search with DLP, NOT Google Search Grounding. | Public web search; no source tracking. |
| **"Explainability, audit trail, regulators ask 'why did AI decide X?'"** | Model Cards + Cloud Audit Logs (Gemini API calls + inputs) + What-If Tool, NOT just model metrics. | Metrics (precision, recall); no audit logs. |
| "Low-latency inference, simple tasks" | Gemini Flash-Lite / Flash (smallest, fastest, cheapest tiers), NOT Gemini Pro. | Gemini Pro; custom model in TensorFlow. |
| "Complex reasoning, multi-step planning" | Gemini Pro (thinking), NOT Flash-Lite (fast but less capable). | Flash-Lite; pre-built API (single-task). |
| "Custom domain language, high accuracy" | Supervised fine-tuning (after prompting/RAG fall short) or custom training, NOT off-the-shelf zero-shot Gemini. | Zero-shot Gemini (generic); pre-built API. |
| "Unbiased model, demographic parity" | Fairness Indicators + slice evaluation in Pipelines, What-If Tool, NOT accuracy metrics alone. | Train-test accuracy; single overall AUC. |
| "Generative images/video from text" | Imagen (images) / Veo (video) from Model Garden, or a Gemini model with image output, NOT a text-only model. | Text-only LLM; self-hosted Stable Diffusion (more ops). |
| **"Human review before AI output published"** | Human-approval step in the workflow (Pipelines approval gate, Workspace Studio review step, application-level approval), NOT just logging. | Logs show decision; humans not in critical path. |

---

## Numbers to remember

### Availability & SLA tiers

| Service / Tier | SLA | RTO | RPO | Notes | Source |
|---|---|---|---|---|---|
| Cloud SQL HA (regional, Enterprise edition) | 99.95 % | Automatic failover, typically about a minute or two (not an SLA) | ≈0 (synchronous regional disk) | Automatic zonal failover | Ch05 |
| Cloud Spanner (multi-region) | 99.999 % | Automatic, seconds | 0 (synchronous replication) | Regional configuration is 99.99 % | Ch05 |
| Bigtable (replicated, multi-cluster routing) | 99.99 % | Automatic with multi-cluster routing | Small, non-zero (async replication) | Eventual consistency between clusters | Ch05 |
| Firestore (multi-region) | 99.999 % | Automatic | ≈0 | Strong consistency; single-region location is 99.99 % | Ch05 |
| Firestore in Datastore mode (formerly Cloud Datastore) | Same as Firestore (location-dependent) | Automatic | ≈0 | Datastore was merged into Firestore | Ch05 |
| Cloud Storage (Standard) | 99.9 % regional / 99.95 % dual- and multi-region | N/A | Async cross-region replication (turbo replication targets 15 min on dual-region) | Regional = zone-redundant; dual/multi-region = geo-redundant | Ch05 |
| Filestore (Regional/Enterprise) | 99.99 % (older docs 99.95 %) | Automatic zonal failover | ≈0 (synchronous across zones) | Basic/Zonal tiers are 99.9 % | Ch05 |
| Cloud Load Balancing | 99.99 % | — | — | Global + regional, multi-zone | Ch03 |

### Storage class & minimum durations

| Class | Min Duration | Access Cost | Latency | Use | Source |
|---|---|---|---|---|---|
| Standard | 0 days | Lowest | <10 ms | Frequent access, analytics, data lake | Ch05 |
| Nearline | 30 days | Low | <10 ms | Infrequent (1/month), backup | Ch05 |
| Coldline | 90 days | Medium | <10 ms | Rare access (1/quarter), archival | Ch05 |
| Archive | 365 days | Highest | Milliseconds (same as other classes) | Compliance archive, <1/year | Ch05 |
| Autoclass | No early-deletion charge for moves between classes | Optimized | Milliseconds | Automatic class transition based on access | Ch05 |

### Compute & networking limits

| Resource | Limit | Notes | Source |
|---|---|---|---|
| VPN throughput | Up to ~3 Gbps per tunnel | Add tunnels (ECMP) for more; HA VPN SLA 99.99 % | Ch03 |
| Dedicated Interconnect | 10 Gbps or 100 Gbps per circuit | Bundle several circuits for more | Ch03 |
| Partner Interconnect | 50 Mbps–50 Gbps per VLAN attachment | Provider-delivered | Ch03 |
| Cloud SQL storage | Up to ~64 TB per instance | MySQL/PostgreSQL/SQL Server | Ch05 |
| Cloud Spanner storage | Scales with nodes/processing units | Storage limit per node exists; multi-region adds replicas and cost | Ch05 |
| Bigtable cluster size | Add nodes for linear throughput scaling | Storage and throughput scale per node | Ch05 |
| Cloud Run service request timeout | Up to 60 minutes (default 5) | Jobs run much longer per task | Ch04 |
| Agent Runtime timeout | 7 days | Long-running agents | Ch08 |
| BigQuery dataset location | Regional or multi-region | Immutable after creation | Ch05 |
| Pub/Sub throughput | Scales automatically; limited by project quotas rather than a fixed per-topic cap | Message retention up to 31 days on topic (7 days default on subscription) | Ch06 |

### KMS & encryption rotation

| Configuration | Rotation | Audit | Notes | Source |
|---|---|---|---|---|
| Google default encryption | Automatic, managed by Google | Not customer-visible | Default; no user control | Ch11 |
| Customer-managed keys (CMEK, software) | Configurable automatic schedule (e.g. 90 days) or manual | Key operations logged in Cloud Audit Logs | Customer control, can disable/destroy key | Ch11 |
| Cloud KMS HSM keys | Configurable automatic schedule or manual | Key operations logged; keys in FIPS 140-2 Level 3 HSMs | Regulatory requirement for HSM-protected keys | Ch11 |
| Key Access Justifications (with Cloud EKM) | N/A | Each external key request carries a justification you can approve/deny | Needs EKM; strongest governance | Ch11 |

### IAM & organization policy limits

| Item | Limit | Notes | Source |
|---|---|---|---|
| Principals per allow policy | ~1,500 (of which a limited number can be Google groups) ⚠ verify | Prefer groups over individual users | Ch10 |
| Service account keys (per SA) | Max 10 user-managed keys | Best practice: no keys (use attached SAs, impersonation, WIF) | Ch10 |
| Organization policies | Inherited org → folder → project; child can override unless parent enforces | Boolean and list constraints | Ch10 |
| IAM deny policies | Separate from Org Policy; evaluated before allow policies | Deny wins over allow | Ch10 |
| Cloud Identity groups | Nested groups supported | Use for role assignment | Ch10 |

### AI/ML infrastructure

| Component | Specification | Cost/hr | Availability | Source |
|---|---|---|---|---|
| TPU (Trillium / Ironwood) | Google custom accelerators for large-scale training and serving (JAX, PyTorch/XLA) | Priced per chip-hour; see pricing page | Capacity must be planned | Ch09 |
| GPU A3 Ultra / A4 | NVIDIA H200 / B200 8-GPU VMs for large training and inference | Highest GPU tiers | Reserve or DWS for capacity | Ch09 |
| GPU L4 | 24 GB VRAM; inference and light fine-tuning | Lower | Broad availability | Ch09 |
| Gemini model tiers | Pro (reasoning, slowest, costliest) > Flash (balanced) > Flash-Lite (fastest, cheapest) | Per-token; Provisioned Throughput for reserved capacity | Pay-as-you-go may be throttled (shared quota) | Ch08 |
| RAG Engine | Managed orchestration; latency depends on retrieval + generation | Embeddings + vector store + model usage | Managed | Ch08 |
| Vector Search | Low-latency ANN retrieval | Index serving nodes + storage | Managed (you manage embeddings) | Ch08 |
| Agent Platform Endpoint (online) | Real-time serving, autoscaling | Node-hours or Provisioned Throughput | Autoscaled | Ch08 |
| Agent Platform batch prediction | Asynchronous, not latency-sensitive | Per job | Cost-optimized | Ch08 |
| DWS Flex Start | Queued until capacity is found (wait time varies) | Discounted (Spot-like) | Best effort | Ch09 |

### DR & RTO/RPO targets

| Pattern | RTO | RPO | Complexity | Cost | Source |
|---|---|---|---|---|---|
| Backup & restore (cold) | Hours to a day or more | Last backup (hours to a day) | Low | Lowest | Ch02 |
| Pilot light | Tens of minutes to a few hours | Replication lag (seconds to minutes) | Medium | Low–moderate | Ch02 |
| Warm standby | Minutes | Replication lag (seconds) | High | Moderate–high | Ch02 |
| Hot / active-active | Seconds or near zero | ≈0 with synchronous stores | Very high | Very high | Ch02 |

### Migration & assessment

| Tool | Cost | Effort | Output | Source |
|---|---|---|---|---|
| Migration Center | No charge for the discovery/assessment tooling ⚠ verify | Discovery then planning | TCO estimate, target recommendations, migration groups | Ch07 |
| Migrate to Virtual Machines | No charge for the service itself; you pay for the resources used | Per wave | VMs in Compute Engine, network integration | Ch07 |
| Database Migration Service | Free for homogeneous migrations; heterogeneous may be billed ⚠ verify | Per database | Cloud SQL and AlloyDB targets (check current list) | Ch07 |
| Storage Transfer Service | Per-GB for some sources; free for Google-to-Google | Depends on data size and bandwidth | Data in Cloud Storage | Ch07 |

### Exam facts

| Item | Value | Notes |
|---|---|---|
| **Duration** | 2 hours | Standard exam |
| **Questions** | 50–60 multiple-choice/multiple-select | ~2–3 min per question |
| **Cost** | $200 | Exam fee |
| **Validity** | 2 years | Renewal exam required after |
| **Renewal exam** | 1 hour, 25 questions, $100 | 90–100% from one gen-AI case study |
| **Case studies** | 2 random from pool of 4 | ~20–30% of questions |
| **Case study pool** | Altostrat Media, Cymbal Retail, EHR Healthcare, KnightMotives Automotive | Official PDFs on Google Cloud |
| **Section S1 weight** | ~25% | Design & planning |
| **Section S2 weight** | ~17.5% | Manage & provision |
| **Section S3 weight** | ~17.5% | Security & compliance |
| **Section S4 weight** | ~15% | Analyze & optimize |
| **Section S5 weight** | ~12.5% | Manage implementation |
| **Section S6 weight** | ~12.5% | Operations excellence |
| **Passing score (est.)** | Not published (community guesses ~70 %) | Pass/fail is reported; no score cutoff is disclosed ⚠ verify |
| **Recommended prep** | 3+ years industry, 1+ year GCP | Official exam guide recommends this background |
| **Community overlap estimate** | ~60 % classic architecture, ~40 % newer services and AI | Per community write-ups of the new syllabus (v6.1) ⚠ verify |

---

## Top 50 exam traps

| # | Trap | Why it is wrong | Right answer | Chapter |
|---|---|---|---|---|
| 1 | "Always use Spanner." | Spanner costs substantially more than Cloud SQL for small workloads; overkill if question doesn't require global consistency or multi-row ACID across regions. | Choose Spanner only for global ACID + multi-region consistency; else Cloud SQL. | Ch05 |
| 2 | "Managed = always better." | Managed services reduce ops overhead but sacrifice fine-grained control. For custom queries or edge-case workloads, self-managed may be correct. | Evaluate trade-off: ops burden vs. control. | Ch01 |
| 3 | "Cloud Run can't run long tasks." | Cloud Run service requests time out after at most 60 min, but Cloud Run jobs run to completion with much longer task timeouts (up to 7 days ⚠ verify). | Use Cloud Run jobs (or Batch) for long run-to-completion workloads. | Ch01 |
| 4 | "VPC Service Controls blocks everything." | VPC-SC enforces an API-level perimeter, but it's configurable. A question about "allow access from external partners" may require an access bridge, not blocking. | Configure what is inside/outside VPC-SC; use access levels/bridges. | Ch01 |
| 5 | "Multi-region = multi-region failover." | Multi-region Cloud Storage is automatic failover in one bucket. Multi-zone ≠ multi-region. A "zero-RTO within single region" needs multi-zone, not multi-region. | Match failover scope to question: single-region (multi-zone) vs cross-region (multi-region). | Ch02 |
| 6 | "Spot VMs = always cheaper." | Spot VMs are roughly 60–91% cheaper but can be preempted at any time (30 s notice). Unacceptable for HA. A question saying "≥99.9% uptime" rules out pure Spot. | Reserve Spot for batch, CI/CD, ML training only; not user-facing. | Ch02 |
| 7 | "BigQuery is for analytics only." | BigQuery is also a data warehouse for complex joins and ML (BigQuery ML). A "building ML models on large datasets" often points to BigQuery ML, not Agent Platform. | BigQuery ML is valid for model training on big data. | Ch01 |
| 8 | "AI = Gemini always." | Not every AI question requires Gemini. "Detecting inappropriate content in images" = Vision API (pre-built); "sentiment analysis" = NL API or CCAI Insights, not Gemini. | Match AI task to pre-built API first; Gemini for generative text/reasoning. | Ch08 |
| 9 | "Kubernetes = always GKE." | For Kubernetes across on-prem and other clouds, Anthos was rebranded GKE Enterprise (GKE on VMware/bare metal and attached clusters, managed as a fleet). | GKE Enterprise for hybrid/multi-cloud; plain GKE for cloud-only. | Ch01 |
| 10 | "HIPAA = Assured Workloads only." | You can meet HIPAA without Assured Workloads using Cloud KMS (HSM), encryption, audit logging, access approval, and BAA. | Assured Workloads simplifies HIPAA but isn't the only path. | Ch01 |
| 11 | "99.9% availability with zonal GKE." | Zonal cluster = all nodes in one zone; failure = entire outage. | Use regional GKE (nodes auto-spread to 3 zones). | Ch02 |
| 12 | "RPO of 1 hour" implies daily backup. | Daily snapshots miss 23 hours of transactions; RPO = replication lag, not backup frequency. | Use hourly snapshots or continuous replication (Cloud SQL async replica, Datastream). | Ch02 |
| 13 | "Multi-region Spanner = twice the cost." | Multi-region Spanner configurations carry more replicas (including witness/read-write regions) so cost is well above a regional instance; there is no simple 2× rule. | Price the chosen multi-region configuration with the calculator; use it only when the requirement justifies it. | Ch02 |
| 14 | "DR tested annually." | Untested DR never works. RTO/RPO are theoretical. | Execute runbook monthly; measure actual RTO/RPO; update based on findings. | Ch02 |
| 15 | "External API SLA 99% acceptable for 99.9% SLA." | Composite SLA = your SLA × their SLA = 99.9% × 99% = 98.9%. | Negotiate better SLA, decouple with async, or accept lower composite SLA. | Ch02 |
| 16 | "Horizontal scaling solves latency." | More instances increase throughput, not per-request latency. | Optimize queries, cache, use CDN, multi-region for latency. | Ch02 |
| 17 | "Warm standby region with 0 RPO." | Only hot (sync replication) guarantees 0 RPO; warm (async) has lag (~<1 min). | RPO = replication lag; cold/warm/hot determine acceptable lag. | Ch02 |
| 18 | "DLP scanning not needed in standby." | Standby replica contains same sensitive data as production. | Apply DLP rules to standby or segregate access. | Ch02 |
| 19 | "Minimize ops overhead" → Cloud VPN. | Cloud VPN is correct (managed, no fiber). Interconnect (requires fiber/coordination) is higher ops overhead. | Cloud VPN for low-ops hybrid; Interconnect for high-throughput. | Ch03 |
| 20 | "Pod-to-pod traffic filtering in GKE" → VPC Firewall Rules. | Firewall rules see node IPs, not pod IPs. | Use Kubernetes Network Policies for pod-level filtering. | Ch03 |
| 21 | "Secure Cloud Run to on-prem DB" → Serverless VPC Access. | Serverless VPC Access is the older connector-based option (extra connector VMs to size and pay for); both reach on-prem over hybrid connectivity. | Prefer Direct VPC egress for Cloud Run (no connector); keep connectors where Direct egress isn't supported. | Ch03 |
| 22 | "Firewall rule applies org-wide immediately" → VPC Firewall Rules. | VPC Firewall Rules are project-scoped. | Use Hierarchical Firewall Policy for org-wide enforcement. | Ch03 |
| 23 | "Cost-effective hybrid <100 Mbps" → Dedicated Interconnect. | Dedicated Interconnect is too expensive for low bandwidth (recurring). | Use Cloud VPN for <100 Mbps non-critical. | Ch03 |
| 24 | "Multi-region global Anycast IP" → Internal LB. | Internal LB is regional only. | Use Global HTTP(S) LB or Global SSL/TCP Proxy LB. | Ch03 |
| 25 | "Minimal ops overhead, auto-scale to 0" → Compute Engine MIGs. | MIGs don't scale to 0; Cloud Run does. | Use Cloud Run for auto-scale-to-zero APIs. | Ch04 |
| 26 | "Stateful workload persistent data" → Cloud Run. | Cloud Run is stateless. | Use GKE StatefulSet + PersistentVolume. | Ch04 |
| 27 | "Sub-second cold-start latency" → GKE. | GKE pods need scheduled nodes (new nodes take much longer to provision); Cloud Run is built for fast, scale-to-zero starts (min instances remove cold starts). | Use Cloud Run. | Ch04 |
| 28 | "Scale 10,000 pods" → Compute Engine MIGs. | MIG scaling is VM-level (coarse-grained). | Use GKE with HPA (pod-level scaling). | Ch04 |
| 29 | "Windows Server/SQL Server" → GKE. | GKE supports Windows node pools, but with limitations; full Windows Server/SQL Server needs are simpler on VMs. | Use Compute Engine (or VMware Engine / Bare Metal); Cloud SQL for SQL Server for the database. | Ch04 |
| 30 | "Sub-millisecond cache latency" → Cloud SQL read replicas. | Read replicas have ~1–10 ms network latency. | Use Memorystore (Redis) for <1 ms. | Ch05 |
| 31 | "Hotspotting in Bigtable" → Add more shards. | Hotspotting is bad row-key design. | Use hash prefix or reverse chronological row keys. | Ch05 |
| 32 | "Compliance archive + immutable" → Cloud Storage lifecycle rule. | Lifecycle rules can be deleted; not truly immutable. | Use Cloud Storage Archive + Bucket Lock (WORM). | Ch05 |
| 33 | "Shared filesystem for GKE" → Cloud Storage. | Cloud Storage has no POSIX mount (except FUSE, which is slower). | Use Filestore NFS. | Ch05 |
| 34 | "Restore to point-in-time" → Cloud SQL snapshots. | Snapshots are discrete points, not continuous. | Use Cloud SQL point-in-time recovery (within the transaction-log retention window). | Ch05 |
| 35 | "Lower Pub/Sub publish latency" → Increase throughput or scale topics. | Publish latency is often client-side batching delay. | Tune publisher batch settings (smaller delay/threshold) before adding topics. | Ch06 |
| 36 | "Exactly-once message delivery" → Pub/Sub guarantee. | Pub/Sub is at-least-once by default; exactly-once delivery exists only for pull subscriptions (regional, with limits). | Enable exactly-once on a pull subscription, and still make consumers idempotent. | Ch06 |
| 37 | "Dataflow vs. Dataproc" → Spark everywhere. | Dataflow is Beam/greenfield; Dataproc is Spark/on-prem legacy. | Choose Dataflow for new pipelines; Dataproc for existing Spark/Hadoop. | Ch06 |
| 38 | "Streaming vs. batch" → Always stream for freshness. | Streaming costs more and is harder to operate; batch is cheaper and simpler when freshness requirements are relaxed. | Batch by default; streaming only when the freshness requirement demands it. | Ch06 |
| 39 | "Fast to cloud" → Low cost to operate. | Rehost is fastest but OpEx stays high (large VMs, licensing). | Evaluate long-term TCO, not just migration speed. | Ch07 |
| 40 | "Licensing BYOL" → Shared VMs. | BYOL on shared VMs is against licensing agreements. | Use sole-tenant nodes for BYOL. | Ch07 |
| 41 | "Migration Center ≠ Pricing Calculator." | Pricing Calculator is generic SKU pricing. | Migration Center recommends target and estimates *for that target*. | Ch07 |
| 42 | "Multi-turn conversation, stateful" → Model.generate() one-off. | One-off calls lose conversation state. | Use Agent Platform Sessions or Memory Bank. | Ch08 |
| 43 | "Sub-100 ms latency, simple QA" → Gemini Pro. | Gemini Pro is the slowest, costliest tier, built for reasoning. | Use Gemini Flash-Lite or Flash for simple, latency-sensitive QA. | Ch08 |
| 44 | "Private knowledge base" → Google Search Grounding. | Google Search is public web only. | Use RAG Engine or Vector Search for private data. | Ch08 |
| 45 | "Minimize GPU idle time during training" → Standard on-demand reservation. | Wastes money on unused reservation hours. | Use Spot instances or DWS Flex Start (pay only for job duration). | Ch09 |
| 46 | "Inference <100 ms SLA" → TPU v5e. | TPU v5e is the cost/throughput-optimised TPU; latency depends on model size, batching and serving stack, so hardware alone doesn't guarantee a latency SLA. | Size the accelerator (L4+ GPU or newer TPU) and serving stack against measured latency. | Ch09 |
| 47 | "Default service account best." | Default service account is shared; hard to audit. | Create custom service account per workload. | Ch10 |
| 48 | "Revoke role at child if parent granted it." → Use IAM alone. | IAM policies are additive; revoking at child doesn't override parent. | Use Deny policy to override; or move resource to folder without parent policy. | Ch10 |
| 49 | "VPC-SC is a firewall." | VPC-SC is API-level; cannot block network traffic. | VPC-SC blocks API calls; Firewall blocks packets. Use both for defense-in-depth. | Ch11 |
| 50 | "Data must stay in EU" → Just store in EU region. | IAM IP conditions or VPC firewall insufficient for residency. | Org Policy `gcp.resourceLocations` + VPC-SC EU perimeter + CMEK in EU. | Ch12 |

---

## Fifty practice scenarios

### Section 1: Design and planning

**Q1. Your company stores EU customer data and must comply with GDPR. Data never leaves the EU, even if misconfigured. Which controls are MOST effective?**

- A) Org Policy `gcp.resourceLocations` constraint (EU value groups) + CMEK in EU + VPC-SC EU perimeter.
- B) Enable CMEK in europe-west1 only.
- C) IAM IP-address conditions restricting EU IP ranges.
- D) Cloud Armor rate-limiting on BigQuery API.

**Answer: A** - Org Policy blocks creation of non-EU resources at API layer. CMEK ensures EU-held encryption key. VPC-SC perimeter prevents export to non-EU services. B alone insufficient (key security ≠ residency). C is network layer. D is rate-limiting.

**Why not the others:** B: CMEK is encryption; doesn't enforce residency. C: Network conditions don't enforce data residency at creation time. D: Rate-limiting is unrelated to data residency.

---

**Q2. You are designing a global e-commerce platform serving millions of users with <100 ms p99 latency, multi-region HA (99.95% SLA), and cost-conscious. Which compute + storage combo is most suitable?**

- A) Zonal GKE in single region + Cloud SQL HA replica; Cloud Storage in single region.
- B) Cloud Run services in a single region + Memorystore (Redis) multi-zone; BigQuery as the system of record for orders and inventory.
- C) Global HTTP(S) LB + multi-region GKE (Autopilot, 3 regions) + Spanner (multi-region); Firestore for session store.
- D) Compute Engine MIGs (3 regions, manual failover) + Cloud SQL cross-region replicas; Cloud Storage multi-region.

**Answer: C** - The global external Application LB (anycast, Premium Tier) routes each user to the nearest healthy region. Spanner multi-region gives a 99.999% SLA with strong consistency for globally distributed orders/inventory. Firestore serves session data with multi-region durability. Autopilot removes node-level ops burden.

**Why not the others:** A: Zonal GKE = single point of failure; can't meet 99.95%. B: Single-region Cloud Run is not multi-region HA, and BigQuery is an analytics warehouse, not a transactional system of record. D: Manual failover gives an unpredictable RTO and cross-region Cloud SQL replicas are asynchronous, so it cannot meet the SLA reliably.

---

**Q3. A media company wants to auto-generate podcast transcripts, extract metadata (speaker names, topics), and detect harmful content in audio streams. They process 10,000 hours/month, have a budget constraint, and want batch processing. Which architecture minimizes cost?**

- A) Cloud Run service listening to Pub/Sub, invoking Speech-to-Text API real-time for each audio file.
- B) Batch pipeline: Cloud Storage → Speech-to-Text batch recognition (with speaker diarization) → Gemini Flash batch prediction on the transcripts (topics, entities, harmful-content classification) → BigQuery; orchestrated by Workflows or Cloud Composer.
- C) Agent Platform Pipelines custom training on audio; deploy to Endpoints for real-time inference.
- D) Gemini Pro online calls on each raw audio file, issued synchronously from a Cloud Run service as files arrive.

**Answer: B** - Prebuilt/pre-trained models do the work with no training: Speech-to-Text batch recognition is the cheaper path for non-urgent audio, and Gemini batch prediction (discounted vs online) extracts metadata and classifies harmful content. BigQuery stores results cheaply. Budget-friendly for high-volume batch.

**Why not the others:** A: Real-time (streaming) Speech-to-Text for 10k hours/month costs more than batch/dynamic-batch recognition and the stem asks for batch. C: Custom training is unnecessary; pre-built APIs and Gemini suffice. D: Synchronous online Pro calls cost more and add no value for latency-tolerant batch work.

---

**Q4. You are planning a migration from on-prem mainframe (3 payment processing systems, 1 inventory system) to GCP. Mainframe runs 24/7, can't be offline, and must continue for 18 months post-cloud-go-live for regulatory hold. Which migration strategy minimizes risk and cost?**

- A) Lift all 4 to Compute Engine VMs, test 3 months, hard cutover, retire mainframe.
- B) Rehost payment systems to Cloud Run (assume stateless); refactor inventory to Firestore; retire mainframe immediately.
- C) Replatform payment systems to Cloud SQL, inventory to BigQuery, run in parallel 6 months only.
- D) Parallel-run all 4 (mainframe + GCP) for 18 months using Dataflow for reconciliation, cutover inventory first (lowest risk), then payment systems after 12 months.

**Answer: D** - Parallel run minimizes cutover risk (can rollback). Dataflow handles reconciliation. Phased cutover (inventory first, payment systems later) lowers blast radius. 18-month hold is satisfied. Mainframe stays as safety net.

**Why not the others:** A: Hard cutover is risky if bugs found post-go-live. B: Payment processing isn't stateless; refactoring under time pressure is risky. C: 6 months insufficient for regulatory hold requirement (needs 18).

---

**Q5. Your startup is building a conversational chatbot for a retail website (Cymbal Retail scenario). Users ask product questions, and the bot should recommend items + handle orders (function calls to inventory system). Accuracy and low latency (about 1 sec) are critical. Which design is correct?**

- A) Gemini Pro + hand-coded prompt + no grounding → call Apigee for inventory lookup.
- B) Gemini Flash + RAG Engine (grounded on product catalog) + function calling to a synchronous inventory API (Cloud Run behind Apigee) and asynchronous order submission via Cloud Tasks or Pub/Sub + HITL review queue for high-value orders.
- C) Agent Platform custom training on Cymbal chat history; deploy to Batch Endpoint (overnight inference).
- D) NL API sentiment analysis → Cloud Workflows (if positive, offer discount) → Agent Platform inference.

**Answer: B** - Flash is the latency-friendly Gemini tier. RAG Engine grounds answers on product data (reduces hallucinations). Function calling uses a synchronous API for inventory lookups and a queue only for the asynchronous order step. HITL review prevents expensive mistakes. Balanced accuracy + latency.

**Why not the others:** A: No grounding = hallucinated product details, and Pro is slower; Apigee only fronts the API. C: Batch Endpoint is overnight (slow); doesn't meet <1 sec SLA. D: NL API is for sentiment, not conversational; Workflows is orchestration, not inference.

---

**Q6. A financial services firm wants to ensure 99.95% availability for their HIPAA-compliant patient database serving 10 million queries/day. They need zero data loss (RPO 0) and strongly consistent relational data across regions. Which database + replication strategy is correct?**

- A) Spanner (multi-region configuration); all reads/writes strongly consistent; CMEK enabled.
- B) Cloud SQL (regional HA) in us-central1; read replicas in us-east1 and europe-west1; accept eventual consistency for reads.
- C) Cloud SQL (regional HA) with a cross-region read replica that is promoted manually after a regional disaster.
- D) Bigtable (multi-cluster) with async replication; Cloud SQL for audit logs.

**Answer: A** - Spanner multi-region provides a 99.999% SLA, synchronous replication (RPO 0) and strong consistency for relational data. CMEK supports customer key control for HIPAA workloads (also sign the BAA and use HIPAA-covered services). Single source of truth; no eventual-consistency complexity.

**Why not the others:** B: Cross-region read replicas are asynchronous (possible data loss on failover) and reads are stale. C: Promotion is manual and the replica lags the primary, so RPO is not zero. D: Bigtable replication is async/eventually consistent and it is not relational (not compliant with zero-data-loss requirement).

---

**Q7. A retail company processes 100 GB of order data daily (structured, CSV files in Cloud Storage). They want to load into BigQuery daily, enrich with ML predictions (upsell recommendations), and export results to a CDP. Latency must be <1 hour end-to-end. Which pipeline is most cost-effective?**

- A) Dataproc (Spark) cluster spinning up daily → apply pre-trained Agent Platform Model Garden model → BigQuery sink.
- B) Cloud Composer DAG: Cloud Storage → Dataflow → BigQuery ML inference → Cloud Workflows export.
- C) Scheduled BigQuery load job from Cloud Storage → BigQuery ML prediction in a scheduled query → export results (EXPORT DATA to Cloud Storage or Pub/Sub) to the CDP, orchestrated by Workflows or Cloud Scheduler.
- D) Cloud Functions (CSV trigger) → transform in-memory → BigQuery insert → call Agent Platform batch endpoint.

**Answer: C** - CSV files already in Cloud Storage can be loaded with BigQuery load jobs (no charge for the load itself), and BigQuery ML runs inside the warehouse (no custom endpoint, no data movement). Serverless and simple; easily within 1 hour.

**Why not the others:** A: A Dataproc cluster is extra infrastructure to start and manage for a simple load. B: Cloud Composer and Dataflow add cost and moving parts the pipeline does not need. D: Cloud Functions inserting 100 GB row by row is not idiomatic; batch load jobs are cheaper and faster.

---

**Q8. You are designing a disaster recovery strategy for a mission-critical ecommerce platform. Current RTO = 1 hour, RPO = 15 minutes. Acceptable downtime = 10 min/year. Which DR pattern + tooling is correct?**

- A) Cold backup (daily snapshots to Cloud Storage Archive) + manual restore runbook.
- B) Warm standby (Cloud SQL async read replica in secondary region) + Application Load Balancer + health checks; DNS failover on-demand.
- C) Pilot light (minimal CloudSQL replica, powered-off 95% of time; spin up on failover).
- D) Hot active-active (Spanner multi-region, sync replication) + Global Load Balancer; auto-failover on health check failure.

**Answer: D** - Spanner multi-region + global LB gives automatic health-check failover (seconds) and RPO ≈0 (synchronous replication). 10 min downtime/year is roughly 99.998% availability; only a hot, active-active design with automatic failover can credibly meet it. D is correct.

**Why not the others:** A: Cold backup/restore takes hours to a day (exceeds the 1-hr RTO). B: Warm standby with an async replica has non-zero RPO and manual/on-demand DNS failover, so it cannot reliably stay within 10 min/year. C: Pilot light needs manual activation and a build-out step (tens of minutes to hours).

---

**Q9. A healthcare SaaS (EHR) serving hospitals across US and EU needs consistent, low-latency reads and writes (<100 ms) for hospitals on both continents, and each hospital must see only its own data. Current architecture: single Cloud SQL in us-central1. Which architecture is the best fit?**

- A) Cloud SQL cross-region read replica in europe-west1; IAM policies per hospital.
- B) Spanner with a multi-region configuration spanning US and EU + fine-grained access control (database roles) per hospital.
- C) Firestore (multi-region) with per-hospital document partitioning + IAM + Cloud Armor.
- D) Cloud SQL HA (single region) + Cloud VPN to EU clinics; local caching with Memorystore EU.

**Answer: B** - Spanner serves strongly consistent reads and writes from both continents (synchronous replication, RPO 0), and fine-grained access control restricts each hospital's role to its own rows/columns. Single source of truth; low operational complexity. (Check data-residency rules for EU patient data before choosing a cross-continent configuration.)

**Why not the others:** A: The EU replica is read-only (writes still cross the Atlantic to the US primary), and IAM does not enforce per-row isolation. C: Firestore is a document store (not a relational fit for the existing Cloud SQL schema), and Cloud Armor does not isolate tenant data. D: A single region means EU users pay trans-Atlantic round trips on every query; caching does not help writes.

---

**Q10. You're consulting for KnightMotives (autonomous vehicle co.). They need consistent AI UX across vehicle models (legacy + new), with onboard inference for safety (no cloud fallback). Connectivity is unreliable in rural areas. What is the architecture?**

- A) All AI models in cloud (Agent Platform) only; offline cache on vehicles using a LiteRT (TensorFlow Lite) fallback.
- B) Edge TPU deployment on vehicle hardware + large Gemini models run through Agent Runtime on the vehicle.
- C) Compact on-device models (LiteRT/TensorFlow Lite) for safety-critical functions + cloud Agent Platform for non-critical (infotainment, analytics).
- D) Cloud Run endpoints with edge caching; critical models replicated to on-device SQLite.

**Answer: C** - Safety-critical inference (steering, braking) must run on-device (no latency/connectivity risk). Non-critical inference (suggestions, analytics) runs in the cloud. Hybrid approach balances safety + features + cost.

**Why not the others:** A: Cloud-primary design depends on connectivity; unsafe for safety-critical functions. B: Agent Runtime is a managed cloud service, not an on-device runtime, and large Gemini models do not fit vehicle memory/power budgets. D: SQLite not designed for model serving; edge caching insufficient for continuous decision-making.

---

### Section 2: Infrastructure

**Q11. Your company runs 50 microservices on GKE Standard (3-zone cluster, 300 nodes). Developers request: high autoscaling granularity, cost optimization, and automated right-sizing and node scaling, while the platform team still needs privileged DaemonSets and custom node images/kernel settings. What should you deploy?**

- A) GKE Autopilot (replaces Standard cluster; managed nodes, automatic scaling).
- B) Existing Standard + migrate to Compute Engine MIGs + manual instance groups.
- C) GKE Standard + switch to Spot VMs (save 80%) + add Kueue for job prioritization.
- D) Existing Standard + add Vertical Pod Autoscaler (VPA) for right-sizing + Cluster Autoscaler for node scaling.

**Answer: D** - VPA right-sizes pods (request tuning); Cluster Autoscaler adds/removes nodes. Both work with Standard cluster, giving full control + cost optimization without replatforming.

**Why not the others:** A: Autopilot manages nodes for you but restricts privileged workloads and custom node images/kernel settings, which the stem requires. B: MIGs are less granular than Kubernetes autoscaling. C: Spot is fault-tolerant only; not suitable for persistent services (Kueue is for batch).

---

**Q12. You're migrating a stateful monolithic Java app (500 GB database, 10,000 concurrent users) from on-prem to GCP. The app uses external file shares (NFS) for temp storage. Which compute + storage combo is most appropriate?**

- A) Compute Engine VMs in a managed instance group (e.g., n2-standard-8) for the app (rehost) + Cloud SQL for the 500 GB database (managed replatform) + Filestore for the NFS temp storage; load-balanced by a load balancer.
- B) Cloud Run + Cloud Storage FUSE mount (stateless app, object storage).
- C) GKE StatefulSet (1 pod, PersistentVolume) + Filestore Regional (shared NFS); Compute Engine VM for legacy support.
- D) Cloud SQL (migrate 500 GB) + Cloud Run (stateless) + Cloud Storage (replace NFS).

**Answer: A** - The monolith is stateful and expects NFS, so it needs Compute Engine (not Cloud Run). Filestore provides the familiar NFS share; Cloud SQL takes over database operations with minimal change. Minimal refactoring.

**Why not the others:** B: Cloud Run is stateless; can't run the monolith directly. C: A single StatefulSet pod for 10k users is a SPOF and does not scale. D: Re-architecting the monolith to Cloud Run (stateless) is high-risk and time-consuming.

---

**Q13. Your batch ML training pipeline (200 TB dataset, 48 hrs training, GPUs required) runs monthly. The training code already checkpoints and can resume after interruption, cost is critical, and the team does not want to run a cluster. Which infrastructure is most cost-effective?**

- A) Compute Engine MIG (n1-highmem-96 + 8 A100 GPUs) reserved for 1 year; run 1 month/year, idle 11 months.
- B) Cloud Batch API + Spot VMs + container image with training code; serverless orchestration.
- C) Compute Engine Spot VMs (8 A100s) + checkpointing every 10 min; 30-second preemption notice; you script the restart on preemption.
- D) GKE cluster (Spot + On-demand mix); Kueue for job queuing; DWS Flex Start for GPU provision on-demand.

**Answer: B** - Batch provisions Spot VMs (roughly 60–91% cheaper) with no cluster to manage, and its retry policy re-runs interrupted tasks (the job resumes from its checkpoints). Ideal for a monthly run packaged as a container.

**Why not the others:** A: A 1-year reservation/commitment sits idle 11 months = wasteful. C: You build the restart/orchestration logic yourself. D: Needs a GKE cluster plus Kueue for a single monthly job; more to operate (DWS Flex Start is a reasonable alternative when GPU capacity is the issue).

---

**Q14. Your company's microservices (30 services, gRPC, variable load 10–1000 RPS) run on GKE Standard. Several services rely on Kubernetes-specific features (DaemonSets, a service mesh). You want a more managed setup, less operational burden, and faster deployment. Which is the best next step?**

- A) Rewrite all 30 services to run on Cloud Run services, with Memorystore for distributed caching.
- B) Move to GKE Enterprise multi-cloud (AWS + GCP).
- C) Migrate the cluster to GKE Autopilot (Google-managed nodes), keeping the Kubernetes workloads as they are.
- D) Move back to Compute Engine VMs for some services and Cloud Run for the rest.

**Answer: C** - Autopilot removes node lifecycle management (less ops burden) while keeping the existing Kubernetes workloads and features. Lower risk than a full refactor.

**Why not the others:** A: Cloud Run has no DaemonSets/mesh equivalents, so this is a re-architecture, not a migration. B: Multi-cloud adds complexity with no stated requirement. D: Mixing Compute Engine back in increases, not reduces, operational burden.

---

**Q15. A customer requires a hybrid setup: 50 on-prem application servers (Linux), about 100 Mbps of traffic, latency tolerance of tens of milliseconds with no strict jitter requirement, cost-optimized. Which hybrid connectivity is appropriate?**

- A) Dedicated Interconnect (10 Gbps) with a colocation cross-connect.
- B) Partner Interconnect via local carrier + Cloud Router; on-prem termination at carrier PoP.
- C) Cross-Cloud Interconnect (GCP ↔ AWS); on-prem apps live on AWS EC2.
- D) HA VPN + Cloud Router (BGP); the on-prem site uses a standard IPsec/IKE VPN gateway.

**Answer: D** - HA VPN is cost-effective for ~100 Mbps (each tunnel supports up to ~3 Gbps) and offers a 99.99% SLA when both interfaces are configured correctly. Cloud Router exchanges routes dynamically. No circuit provisioning. Note Cloud VPN rides the public internet, so latency is not guaranteed; a strict latency/jitter requirement would point to Interconnect.

**Why not the others:** A: Dedicated Interconnect is overkill and expensive for 100 Mbps. B: Partner Interconnect costs more than VPN for this volume (consider it only when latency or SLA needs it). C: Cross-Cloud Interconnect links Google Cloud to another cloud and does not serve on-prem servers.

---

**Q16. Your data pipeline ingests 1 TB/day of IoT sensor data, needs sub-second latency for alerts, and must scale to 10,000 sensors. Which messaging + storage combo is best?**

- A) Pub/Sub topic (ordering key = sensor ID where per-sensor order matters) → Dataflow (windowed aggregation) → BigQuery (analytics) + Firestore (real-time alerts).
- B) Cloud Tasks (one task per sensor event) → Cloud Functions → Firestore (real-time).
- C) Kafka (managed Confluent Cloud) → BigQuery; Memorystore for alerting cache.
- D) Cloud Storage (hourly batch files) → Cloud Composer (daily ingestion) → BigQuery.

**Answer: A** - Pub/Sub is the managed, autoscaling ingestion layer for high-volume streams. Dataflow processes windowed aggregations with low latency. BigQuery for analytics; Firestore for real-time alert queries. A standard IoT streaming architecture.

**Why not the others:** B: Cloud Tasks dispatches individual asynchronous tasks with rate control; it is not a high-volume streaming ingestion service. C: Running Kafka adds cost and operations that Pub/Sub already covers natively on Google Cloud. D: Batch (hourly) defeats sub-second latency requirement.

---

**Q17. You manage 5 GCP projects (dev, staging, prod, data, shared-infra). Security team wants centralized logging, cost tracking, and unified IAM policy enforcement. What is the organization structure?**

- A) Flat: all 5 projects under one org, with shared folder for common policies.
- B) Hierarchical: org → Folder (by environment: Dev, Staging, Prod) → Folder (by team: data, infra) → projects.
- C) Separate orgs per environment (org-dev, org-prod) with VPC peering for cross-org communication.
- D) No org; manage 5 projects independently with custom scripts for sync.

**Answer: B** - Hierarchical allows org-level logging (all projects) + env-specific policies (Dev permissive, Prod strict) + team-level budgets. Scales well; centralized governance.

**Why not the others:** A: Flat with shared folder lacks isolation. C: Multiple orgs complicate cross-org resources + billing. D: No org = no central control (non-compliance risk).

---

**Q18. Your company wants to standardize container images across 200 teams, enforce supply chain security (no unsigned images deploy), and scan for vulnerabilities automatically. Which tool combo is correct?**

- A) Docker Hub + GitHub Actions + manual image inspection + deploy to GCE.
- B) Artifact Registry + Cloud Build + Cloud Deploy only (no scanning or signing).
- C) Container Registry (shut down) + gcloud builds + custom scanning scripts.
- D) Artifact Registry (central) + Cloud Build (build provenance/attestations) + Binary Authorization (signed/attested images only) + Artifact Analysis (vulnerability scanning).

**Answer: D** - Artifact Registry centralizes images. Cloud Build produces provenance/attestations. Binary Authorization enforces the signature/attestation policy at deploy time. Artifact Analysis scans for vulnerabilities automatically. Covers the supply chain end to end.

**Why not the others:** A: External registries + manual inspection don't scale. B: Lacks vulnerability scanning + signature enforcement. C: Container Registry has been shut down (replaced by Artifact Registry); custom scripts are not auditable.

---

**Q19. You're designing a disaster recovery setup for a Cloud Run application (stateless, 10 reqs/sec average). RTO = 1 hour, RPO = 15 min. Which strategy is most cost-effective?**

- A) Primary Cloud Run (us-central1) + Backup code repo in git; on outage, deploy to us-west1 (manual).
- B) Cloud Run service (us-central1) + Cloud Storage backup of code (snapshot every 15 min) + runbook for redeploy.
- C) Cloud Run service (us-central1) + global external Application LB with a second Cloud Run service deployed in us-west1 (scales to zero, near-zero idle cost) as failover backend; state in a database with cross-region replication.
- D) Cloud Run in one region, with Cloud Deploy used as the failover mechanism between regions.

**Answer: C** - Cloud Run is already multi-zone HA within a region. A pre-deployed second-region service scales to zero, so a warm failover target costs almost nothing while the load balancer's health checks fail over automatically, comfortably within the 1-hour RTO. The 15-minute RPO applies to the data store, which needs cross-region replication or backups.

**Why not the others:** A and B: Redeploying by hand from git/backups depends on a human runbook and needs the second region provisioned during the outage; backing up stateless code does nothing for RPO (the data does). D: Cloud Deploy is a delivery tool; it does not provide runtime failover.

---

**Q20. Your company has 1000 developers accessing 500 GCP projects. You want to grant access per team (Data, Backend, Frontend) without listing individual users. Best practice?**

- A) Create Cloud Identity Groups (data-team@company.com, backend-team@company.com); add team members as groups; grant roles to groups.
- B) Create IAM roles per team; grant role to each developer individually (tedious but explicit).
- C) Use OS Login for all developers; SSH into GCE VMs using OS Login credentials; control access via IAM.
- D) Use Workforce Identity Federation (contractors + employees) with external IdP.

**Answer: A** - Groups decouple user management from IAM. Add/remove developers from group; role stays attached to group. Scales to 1000 users without duplicate role grants.

**Why not the others:** B: 1000 individual grants = unscalable. C: OS Login is for SSH; doesn't solve API/console access. D: WIF is for external identities (contractors); internal employees use Cloud Identity + groups.

---

**Q21. You want to reduce network egress costs (currently $10k/month) for a high-traffic web app (Global LB → multi-region GKE backends). Where should you focus?**

- A) Migrate to Cloud Run (serverless) → auto-scale → reduced traffic.
- B) Reduce HTTP response size (compression, caching headers) → Cloud CDN + local caching layers.
- C) Switch to Compute Engine + Network LB (cheaper than HTTP(S) LB).
- D) Use Premium Tier networking instead of Standard (reduce hops, lower egress cost).

**Answer: B** - Cloud CDN serves cacheable content from Google's edge locations, reducing origin traffic and egress (CDN cache egress is also priced lower than origin egress). Compression reduces response size. Caching headers reduce redundant requests. Direct impact on egress costs.

**Why not the others:** A: Cloud Run doesn't reduce egress costs (still charged per GB out). C: A Network LB doesn't reduce egress (same traffic out of Google Cloud) and loses L7 features. D: Premium Tier is the higher-priced tier (Standard Tier has lower per-GB egress cost but a regional scope and public-internet routing), so it increases cost.

---

### Section 3: Security and compliance

**Q22. A fintech company stores credit card data (PAN, CVV) in a "payment" project separate from analytics projects. They want to ensure card data never reaches analytics (regulatory block). What is the control?**

- A) IAM policies (deny BigQuery.dataAdmin in payment project).
- B) Cloud Storage + Cloud KMS (encrypt card data; require decryption key for export).
- C) VPC Service Controls (payment project in perimeter; analytics outside) + egress rule blocking external APIs.
- D) Data Loss Prevention (DLP) discovery + redaction on export.

**Answer: C** - VPC-SC perimeter enforces API-level boundary. Egress rule inside perimeter blocks calls to analytics BigQuery. Even if developer tries to export, VPC-SC blocks at API layer (not network layer).

**Why not the others:** A: IAM is additive (allows); doesn't block determined developer. B: Encryption is data protection, not exfiltration prevention. D: DLP redaction is last-mile; doesn't prevent access to unredacted data.

---

**Q23. A healthcare system wants to prove to auditors that no one accessed patient X's records without authorization (HIPAA audit trail). Current logging: Cloud Logging with 30-day retention. What change is needed?**

- A) Enable Admin Activity logs (already on); add Data Access logs; export to Cloud Storage with Bucket Lock (immutable).
- B) Enable Cloud Audit Logs sink to BigQuery; query for patient X access.
- C) Use Sensitive Data Protection (DLP) to mask PII before logging.
- D) Use Cloud Identity audit logs only.

**Answer: A** - Admin Activity logs show who granted permissions; Data Access logs show who read data. Bucket Lock makes logs immutable (audit requirement). 7+ year retention possible.

**Why not the others:** B: BigQuery retention can be modified/deleted (not immutable). C: DLP redaction prevents analysis; you need original logs for audit. D: Cloud Identity logs are for identity events, not data access.

---

**Q24. Your company must comply with the EU AI Act (obligations for high-risk AI systems). You're building a loan-approval AI (credit scoring is a high-risk use case). Which set of controls best supports compliance?**

- A) Use only open-source models (GPL license guarantees compliance).
- B) Encrypt data in transit only (HTTPS); compliance is encryption.
- C) Use Google's public Gemini models; Google is compliant, so you are.
- D) Implement human-in-the-loop review, audit logs (all inference + decision rationale), model card documentation, and test for bias.

**Answer: D** - Human-in-the-loop (humans review borderline cases), audit logs (decision trail), model cards (transparency), bias testing (fairness) are core EU AI Act requirements for high-risk systems.

**Why not the others:** A: License compliance ≠ AI governance. B: Encryption is data protection, not AI governance. C: Google's compliance ≠ customer's compliance; customer responsible for their use case.

---

**Q25. You have a Cloud SQL database with customer data in `us-central1`. A developer accidentally exports the database to a Cloud Storage bucket in a project outside your organization. Which control prevents this class of incident in future?**

- A) Enforce the Org Policy constraint `constraints/storage.publicAccessPrevention`; it stops buckets from being made public.
- B) Enable Cloud DLP to scan Cloud Storage; redact PII in public files.
- C) Use VPC Service Controls perimeter (Block export APIs outside perimeter).
- D) Revoke developer's access; rotate database credentials.

**Answer: C** - VPC-SC perimeter prevents Cloud SQL export to Cloud Storage outside perimeter (API-level block). More efficient than post-hoc DLP scanning.

**Why not the others:** A: Public access prevention stops public exposure of a bucket, but does not stop an export to a bucket outside the organization. B: DLP is detective (after the data is already out); VPC-SC is preventive. D: Revoking one developer's access and rotating credentials is a response to this incident, not a control that blocks the export path.

---

**Q26. Your company handles EU customer genetic data (extreme high-risk under GDPR). Storage location, encryption key management, and audit trails must be customer-controlled. Which GCP service supports this?**

- A) Assured Workloads / Sovereign Controls for the EU (EU data residency, Access Transparency/Approval) + Cloud EKM (keys held in an external key manager outside Google) + Cloud Audit Logs.
- B) Org Policy resource-location constraint to EU regions, with Google-default encryption keys.
- C) Cloud Storage + client-side encryption (data encrypted before upload).
- D) BigQuery + Sensitive Data Protection + Confidential Computing.

**Answer: A** - Sovereign/Assured Workloads controls enforce EU residency and restrict/record Google personnel access; Cloud EKM keeps the encryption keys outside Google under customer control, with key use auditable. Meets the residency, key-control and audit requirements together.

**Why not the others:** B: Location constraint gives residency but Google-managed keys give no key control. C: Client-side encryption gives key control but pushes encryption/key handling into every application and blocks server-side processing and analytics. D: Sensitive Data Protection and Confidential Computing address detection and data-in-use protection, not key custody or residency.

---

**Q27. You want to implement end-to-end encryption for Pub/Sub messages (producers encrypt before publish, subscribers decrypt). Which approach is secure + GCP-native?**

- A) Use Pub/Sub encryption (automatic, built-in CMEK).
- B) Client-side encryption (produce → encrypt with Cloud KMS CMEK key → publish; subscribe → decrypt with KMS).
- C) TLS in transit (Pub/Sub uses TLS 1.2+; encryption is automatic).
- D) Use Secret Manager to store encryption keys; manually encrypt/decrypt in app.

**Answer: B** - Client-side (envelope) encryption with a Cloud KMS key means only producers and subscribers can read the payload; Google never sees plaintext, and KMS audit logs show who called decrypt.

**Why not the others:** A: Pub/Sub CMEK encrypts at rest, but the Pub/Sub service itself decrypts the message, so it is not end-to-end. C: TLS is transport layer; doesn't protect message at rest on broker. D: Secret Manager is for static secrets, not per-message encryption.

---

**Q28. Your company's ML model (trained on customer data) must be auditable: show which customers' data was used, whether their consent was obtained, and any model retraining. What is the architecture?**

- A) Log all training data to BigQuery with customer ID, with no link between rows and model versions.
- B) Agent Platform Pipelines for reproducible runs + Cloud Audit Logs only, with no catalog or consent metadata.
- C) TensorFlow SavedModel with embedded metadata; store in Cloud Storage.
- D) Model Registry (versions linked to training datasets) + Dataplex Universal Catalog lineage and metadata (including consent/purpose tags on datasets) + Cloud Audit Logs (who trained and when).

**Answer: D** - Model Registry ties each model version to its training data; the catalog's lineage and tags show dataset ownership, consent/purpose and processing history; Cloud Audit Logs show who trained and when. Together they form a full audit trail.

**Why not the others:** A: Raw data copies are not linked to model versions and add a second copy of customer data. B: Pipelines give reproducibility but nothing records consent or dataset ownership. C: Metadata embedded in an artifact does not scale across datasets/versions and is not centrally queryable.

---

**Q29. You're adopting Confidential Computing to protect sensitive workloads (healthcare analytics). Which GCP resources support Confidential Computing?**

- A) Compute Engine Confidential VMs (data encrypted in memory) + Persistent Disk CMEK.
- B) GKE Confidential Nodes (nodes use Confidential VMs) + Cloud KMS CMEK for pod secrets.
- C) Cloud Run services with a "confidential execution" setting that encrypts the container's memory.
- D) A and B only.

**Answer: D** - Confidential VMs encrypt data in memory (data in use). GKE Confidential Nodes run pods on Confidential VMs. Cloud Run has no Confidential Computing mode ⚠ verify against current docs.

**Why not the others:** A and B alone are each correct but incomplete; C describes a Cloud Run capability that does not exist as stated.

---

**Q30. A SaaS vendor must prove to customers that only authorized staff access their data. Staff includes developers (on-call rotation), DBAs (schema maintenance), and auditors (read-only). What IAM + governance setup minimizes access?**

- A) Three predefined roles (compute.admin, cloudsql.admin, logging.viewer); grant to staff users directly.
- B) Use OS Login for all; audit SSH logs; Workload Identity for apps.
- C) Create custom roles (dev-oncall, dba-maintenance, auditor-readonly) with minimum permissions; grant to Cloud Identity Groups; use Principal Access Boundary (PAB) to limit scope per customer.
- D) Enforce Deny Policies (deny all by default; allow specific actions per role).

**Answer: C** - Custom roles reduce permissions; groups scale to 100s of staff. PAB sandboxes staff access per customer (customer A DBAs only see customer A data). Maximum least-privilege.

**Why not the others:** A: Predefined roles too broad (admin). B: OS Login is for SSH; doesn't control API access. D: Deny Policies are too blunt (deny all breaks normal work); allow-based (roles) is better.

---

**Q31. Your company is migrating 10 TB of PII-heavy data from on-prem to Cloud Storage. You want to ensure sensitive data is detected + redacted during migration. Which tool is most appropriate?**

- A) Dataflow pipeline with DLP transform (redact PII during ingestion → Cloud Storage).
- B) Cloud DLP Discovery job (scan Cloud Storage post-upload for PII patterns).
- C) Tag PII fields in the source system and rely on those tags to protect the data after migration.
- D) Cloud KMS encryption (encrypt everything; compliance satisfied).

**Answer: A** - Dataflow pipeline with DLP transform catches PII during ingestion (not after). Redaction is applied on-the-fly. Prevents storing plain PII.

**Why not the others:** B: Post-upload discovery is reactive; PII already stored. C: Source-side tags do not detect or redact PII in the files being copied. D: Encryption protects, but doesn't redact (still stored encrypted).

---

**Q32. A US federal agency processes controlled unclassified information (CUI) and must meet FedRAMP High. Which Google Cloud deployment fits?**

- A) Standard Google Cloud regions + EU Sovereign Cloud offerings.
- B) Cross-Cloud Interconnect to another cloud provider's government region.
- C) Bare Metal Solution in a standard region, linked on-prem via Cloud Interconnect.
- D) Assured Workloads with the FedRAMP High compliance regime (US data residency, personnel and service restrictions, CMEK/EKM for key control).

**Answer: D** - Assured Workloads applies the FedRAMP High control package (US residency, restricted support personnel, supported services only) to a folder, with customer-controlled keys on top. (Truly classified Secret/Top Secret workloads need Google Distributed Cloud air-gapped, not public regions.)

**Why not the others:** A: EU sovereign offerings target EU requirements, not FedRAMP. B: Cross-Cloud Interconnect is private connectivity, not a compliance regime. C: Bare Metal Solution in a standard region does not by itself apply the FedRAMP High control package.

---

**Q33. You want to prevent developers from accidentally committing API keys to git. Which control is most effective?**

- A) Use Git commit signing (GPG) + Cloud KMS key; review all commits before merge.
- B) Use Secret Manager for all secrets; scan git history with `git-secrets` pre-commit hook.
- C) Binary Authorization + a Cloud Build scanning step (block the build if secrets are found in code).
- D) Educate developers; rely on self-discipline.

**Answer: B** - Secret Manager (external secrets, not in code) + pre-commit hook (scan local git before push) catches secrets early. Layered defense.

**Why not the others:** A: Commit signing verifies author, not secret content. C: SAST in build is late (already committed). D: Self-discipline alone fails at scale.

---

**Q34. Your company's web app logs customer transactions (amounts, card-type). For debugging, developers need logs; for analytics, data team needs summary stats (no PII). How do you set this up?**

- A) Log transactions plaintext; use Sensitive Data Protection to redact PII for analytics users (authorized views).
- B) Log transactions with CSEK (client-side encryption) per request; share decryption key only with finance team.
- C) Use DLP format-preserving encryption (FPE) on sensitive fields; log encrypted values; authorized view shows stats only.
- D) Separate logs: dev sees plaintext; analytics sees redacted-only queries via BigQuery authorized view.

**Answer: D** - Separate access paths: developers read the full logs (restricted by IAM), while analytics queries run through BigQuery authorized views (or column-level security) that expose only aggregates and non-PII columns. Balanced access.

**Why not the others:** A: Redacting after the fact still leaves plaintext PII stored in the logs. B: CSEK applies only to Cloud Storage and Compute Engine, so per-request CSEK for log entries is not an option. C: Format-preserving encryption adds key management and still needs an access layer; the authorized view is simpler.

---

**Q35. You need to deploy a machine learning model that processes customer images (faces, license plates, medical scans). Your compliance team requires human review of model outputs before action. Which pattern is correct?**

- A) Agent Platform Batch Endpoint → auto-save results to BigQuery (no human review).
- B) Agent Platform Online Endpoint → Cloud Run function (routes high-confidence to approval queue, low-confidence to auto-approve).
- C) Agent Platform Online Endpoint → Firestore (stores predictions) + human-review queue (Pub/Sub feeding a review web app) + recorded approval before any action is taken.
- D) Custom training on Agent Platform + model deployment to GKE (full control).

**Answer: C** - Online Endpoint for responsive inference. Firestore stores predictions. A review queue routes every output to humans. The recorded approval step documents sign-off (compliance trail).

**Why not the others:** A: Batch has latency; no human review loop. B: Auto-approve low-confidence is risky (what's "low"?). D: Custom training unnecessary; pre-built models + HITL sufficient.

---

### Section 4: Process and cost

**Q36. Your company's GCP bill is $50k/month. You want to understand where money goes and implement cost governance. What's the first step?**

- A) Use Pricing Calculator to estimate each service's cost.
- B) Enable Cloud Billing export to BigQuery; analyze by service, project, label.
- C) Right-size all VMs manually using the Recommender (VM rightsizing).
- D) Purchase 3-year CUDs across all services to "lock in savings."

**Answer: B** - Billing export to BigQuery is the foundation for analysis (by label, service, project). Without visibility, you can't optimize. Recommender is next step; CUDs without visibility risk wasting money.

**Why not the others:** A: Pricing Calculator is pre-deployment estimate, not post-deployment analysis. C: Recommender is tactical (VM right-sizing); doesn't address app architecture. D: CUDs without visibility risk stranded capacity.

---

**Q37. A development team wants to reduce time-to-deployment. Currently: 2 hours from code commit to prod (manual testing + approval). Cloud Deploy is recommended. What else is needed?**

- A) Cloud Deploy canary strategy + automated deployment verification (tests run after each phase) with automatic rollback on failure; approval only for the production promotion.
- B) Cloud Deploy with an instant-switch release + rollback only if users complain.
- C) Cloud Deploy rolling update + manual approval step (keep existing rigor).
- D) Cloud Deploy + remove all approval gates (move fast).

**Answer: A** - Canary phases + automated verification + auto-rollback replace manual testing with repeatable checks, so releases are both faster and safer.

**Why not the others:** B: Instant-switch releases offer fast rollback but no automated testing and detection relies on user complaints. C: Manual approval bottleneck remains. D: No gates = risk of bad deploys reaching customers.

---

**Q38. Your team manages 5 GCP projects (dev, staging, prod, data, infra). Each project has different cost drivers: prod is compute-heavy, data is storage-heavy. How do you allocate FinOps responsibility?**

- A) Single finance team monitors all 5 projects; approves budget per project monthly.
- B) Automate costs: apply Recommender suggestions and Spot VMs everywhere without review.
- C) Assume Google Cloud optimizes costs automatically; no team involvement needed.
- D) Team leads own their project budgets; set labels (cost-center, app); track via Billing BigQuery; finance audits quarterly.

**Answer: D** - Distributed accountability (team leads + labels) enables fast optimization. Quarterly finance audit prevents drift. Recommender suggestions are reviewed by the owning team before being applied.

**Why not the others:** A: Single team bottleneck; doesn't scale. B: Blindly applying rightsizing and Spot can break apps (need testing). C: Recommender only suggests changes; someone must review and apply them, and nothing else is automatic.

---

**Q39. You're planning a 3-year GCP migration for a $10M CapEx-heavy on-prem infrastructure (data center lease + hardware). Cloud estimates $2M OpEx/year. How do you justify it to finance?**

- A) Compare the $10M CapEx with 3 years of cloud OpEx ($2M/yr × 3) only, ignoring on-prem running costs; cloud wins.
- B) Use Migration Center's cost estimate (vs. Pricing Calculator) + quantify avoided costs (power, cooling, maintenance).
- C) Focus on speed-to-market benefits; cloud enables 6-month faster feature launch (worth $5M in new revenue).
- D) Assume cloud is cheaper; request approval without financial analysis.

**Answer: B** - Migration Center provides accurate target-specific cost estimates (vs. Pricing Calculator's generic SKU pricing). Quantifying avoided OpEx (power, cooling, staff) makes ROI clear.

**Why not the others:** A: Ignores on-prem running costs (power, cooling, staff, hardware refresh), so the TCO comparison is incomplete. C: Speed-to-market is speculative (hard to quantify). D: No analysis = risky approval.

---

**Q40. Your microservices platform has 20 services, each with custom alerting (email, Slack, PagerDuty). Alert fatigue is high (thousands/day). What's the root cause + fix?**

- A) Increase alerting threshold (ignore low-severity alerts).
- B) Reduce monitoring granularity (fewer metrics = fewer alerts).
- C) Switch to SLO burn-rate alerting (alert only if SLO at risk, not on every metric spike).
- D) Mute alerts during business hours (ops team responds faster).

**Answer: C** - SLO burn-rate alerts are symptom-based (user-facing impact); metric alerts are noisy (CPU spike ≠ user impact). Reduce alert count + improve relevance. Pair with on-call rotations for escalation.

**Why not the others:** A: Raises threshold but doesn't fix root cause. B: Reduced granularity hides real problems. D: Muting is avoidance, not fixing.

---

### Section 5: Implementation

**Q41. You're implementing GitOps (desired state in git, automated deployment) for many GKE clusters fed by 50 repos. Which Google-managed approach centralizes deployment policy and reverts accidental manual changes?**

- A) Config Sync (GKE Enterprise) reconciling each cluster from git continuously, with Policy Controller enforcing fleet-wide policy.
- B) Imperative `gcloud`/`kubectl apply` scripts triggered manually per repo.
- C) A third-party SaaS IaC tool with drift detection that you must integrate yourself.
- D) A self-hosted open-source GitOps controller that you install and operate on every cluster.

**Answer: A** - Config Sync is a managed GitOps controller that continuously reconciles cluster state to the repo (reverting drift/manual edits), and Policy Controller applies policy fleet-wide. Native to Google Cloud.

**Why not the others:** B: Imperative and manual; no continuous reconciliation, so drift is not reverted. C: Infrastructure-focused tooling you integrate yourself; not an in-cluster reconciler with fleet policy. D: Works (e.g. Argo CD) but is not Google-managed; you own installation, upgrades and operations.

---

**Q42. Your team uses Terraform (50+ modules, shared across 10 projects). State files contain sensitive data (API keys, passwords). You want the state to stay inside your Google Cloud organization. What's the safest way to manage Terraform state?**

- A) Store state in Cloud Storage with public read access (easier for team collaboration).
- B) A third-party hosted remote backend outside your organization (managed locking, encryption by default).
- C) Store state in git repo (version control + audit trail).
- D) The `gcs` remote backend (built-in state locking, object versioning) on a private bucket with uniform bucket-level access, least-privilege IAM and CMEK; keep secrets out of state where possible.

**Answer: D** - The native `gcs` backend gives remote state, locking and versioning, and you control encryption keys and access with IAM/CMEK inside your own organization.

**Why not the others:** A: Public state = exposed secrets. B: Can work, but state (with secrets) leaves your org and your IAM/CMEK controls. C: State in git exposes secrets in history and has no locking.

---

**Q43. You're building a CI/CD pipeline for a microservice (deployed to GKE via Cloud Deploy). You want to ensure only signed container images deploy. Which sequence is correct?**

- A) Cloud Build → build container → push to Artifact Registry → Cloud Deploy → GKE deploy (no signing).
- B) Cloud Build → build → push to Artifact Registry → human review (manual signing) → Cloud Deploy.
- C) Cloud Build → build + sign (cosign + KMS key) → push signed image and attestation to Artifact Registry → Cloud Deploy → GKE, where the Binary Authorization policy verifies the attestation at admission.
- D) Build locally → push container → GKE deploy (no Cloud Build).

**Answer: C** - Cloud Build signs images with KMS key (provenance/SLSA attestation). Artifact Registry stores signed image + attestation. Binary Authorization enforces signature policy at GKE deploy time. Full supply chain security.

**Why not the others:** A: No signature = can deploy unsigned images. B: Manual signing is slow + not auditable. D: Local build bypasses CI pipeline.

---

**Q44. You're adopting event-driven architecture (multiple services listening to Pub/Sub topics). Tracing cross-service calls is difficult. What observability setup helps?**

- A) Cloud Logging + grep for request ID across logs.
- B) Cloud Trace (distributed tracing with OpenTelemetry instrumentation) + correlation IDs in Pub/Sub metadata + Cloud Monitoring dashboards.
- C) Prometheus (custom instrumentation) + ELK stack (external).
- D) Add print() statements with timestamps to debug (simplest).

**Answer: B** - Cloud Trace shows request flow across services (parent-child spans). OpenTelemetry auto-instruments. Correlation IDs in Pub/Sub metadata link async messages. End-to-end visibility.

**Why not the others:** A: grep is manual + slow. C: Prometheus alone doesn't show traces; ELK is external (not GCP-native). D: Logging is debugging after-the-fact.

---

### Section 6: Operations

**Q45. Your production GKE cluster (1000 pods, 300 nodes) has intermittent latency spikes (p99 latency 200 ms → 1 sec). Where do you investigate first?**

- A) Cloud Trace (distributed tracing to find slow service) + Cloud Profiler (CPU hotspots within that service).
- B) Increase node count (assumes scaling solves latency).
- C) Cloud Monitoring dashboards (CPU, memory, network graphs).
- D) Restart all pods (often fixes transient issues).

**Answer: A** - Cloud Trace shows which service is slow; Cloud Profiler shows why (CPU hotspot, lock contention, etc.). Most direct path to root cause.

**Why not the others:** B: Scaling won't help if bottleneck is single service. C: Dashboards show symptoms, not root cause. D: Restart is avoidance; problem returns.

---

**Q46. You're launching a Gemini-based feature on Agent Platform that will see 100 RPS avg, 1000 RPS peak, with many repeated queries. Cost and latency are concerns. Which design is optimal?**

- A) Provisioned Throughput sized for the 1000 RPS peak, running all the time; cost is high but capacity is guaranteed.
- B) Pay-as-you-go Gemini + a Cloud Run HTTP cache in front, with a second endpoint as fallback.
- C) Self-host an open model on GKE with GPU nodes sized for peak; scale manually.
- D) Pay-as-you-go (per-token) Gemini + Memorystore (Redis) caching of results for repeated queries to cut model calls and latency.

**Answer: D** - Pay-as-you-go has no idle cost, and a cache absorbs repeated requests, reducing both spend and latency. Handle peaks with quota planning (and Provisioned Throughput only if guaranteed capacity becomes a requirement).

**Why not the others:** A: Capacity sized for peak is paid for even when traffic is low; it suits steady, predictable high load. B: A fallback endpoint does not reduce cost, and HTTP caching is a poor fit for POST-style prompts. C: GPU nodes sized for peak mean high fixed cost plus operational burden.

---

**Q47. Your microservices run on GKE; some pods are "noisy neighbors" (hogging CPU, affecting other pods' latency). How do you isolate them?**

- A) Pod Priority + Pod Disruption Budget (prioritize critical pods).
- B) Resource Requests/Limits in pod spec (ensure each pod gets guaranteed resources).
- C) Separate node pools (noisy pods in dedicated pool, critical pods in another).
- D) Increase node count (more resources for everyone).

**Answer: B** - Requests guarantee resource allocation (QoS); Limits cap usage. Pod won't starve others if Requests/Limits are correct. Cheapest solution.

**Why not the others:** A: Priority/PDB handle eviction, not resource isolation. C: Separate pools works but costs more. D: More nodes don't isolate; noisy pod still hogs its node's CPU.

---

**Q48. Your company has a 24/7 on-call rotation (4 engineers, each on-call 1 week/month). Alert fatigue is high; on-calls are burnout-prone. How do you improve?**

- A) SLO burn-rate alerting + incident response runbooks + blameless postmortems + automated remediation of known low-risk issues.
- B) Add more on-call engineers (dilutes responsibility, doesn't fix alerts).
- C) Mute low-priority alerts.
- D) Remove on-call rotation (move to best-effort SLA).

**Answer: A** - SLO burn-rate reduces noise (only real problems alert). Runbooks enable faster resolution. Postmortems reduce recurrence. Automated remediation removes toil for well-understood issues. Sustainable on-call.

**Why not the others:** B: More engineers = more cost, same alert fatigue. C: Muting ignores problems. D: Removing on-call abandons reliability.

---

### Case-study scenarios

**Q49. Altostrat Media scenario: They want to automatically detect and remove harmful content (hateful speech, violence) from user-generated podcasts before publishing. Latency constraint: must complete within podcast's editing window (~30 min). What architecture?**

- A) Real-time Gemini content moderation (analyze stream while recording).
- B) Agent Platform custom training on hateful-content dataset (long training, ongoing).
- C) Batch pipeline per podcast (Cloud Run jobs or Dataflow): Speech-to-Text transcript → text moderation (Natural Language API or Gemini with safety settings) + explicit-content detection on video frames → flagged items to a human review queue.
- D) Manually review all podcasts (no automation).

**Answer: C** - A per-podcast batch pipeline fits a ~30-minute window. Speech-to-Text, moderation and image/video analysis are pre-built (no training). Humans review flagged content (HITL). Balanced cost + accuracy.

**Why not the others:** A: Live analysis while recording is not required by a 30-minute window and is costlier. B: Custom training adds weeks; pre-built APIs sufficient. D: Manual doesn't scale.

---

**Q50. Cymbal Retail scenario: They want conversational product discovery (user asks "show me blue winter coats under $100 with fast shipping"). The bot must call inventory API, recommend SKUs, and handle order placement. Constraint: responses must feel interactive (about 1 second to first token, streamed), multi-turn conversation. What design?**

- A) Custom agent (Agent Builder) + fine-tune Gemini on Cymbal support conversations.
- B) Gemini Flash + grounding on the product catalog (vector search / RAG) + function calling to the synchronous inventory API (via Cloud Run) and the order API + Agent Platform Sessions (conversation memory).
- C) Apigee + Cloud Run functions (stateful conversation endpoint); no LLM, pure business logic.
- D) BigQuery SQL (query product table) + Cloud Workflows (orchestrate APIs) + no LLM.

**Answer: B** - Gemini Flash (fast) + RAG grounds on product data (accurate recommendations). Function calling integrates inventory/order systems through synchronous APIs (Cloud Tasks is asynchronous and would not return stock results to the conversation). Agent Platform Sessions maintain conversation state across turns. Streaming keeps perceived latency low.

**Why not the others:** A: Fine-tuning adds latency + cost. C: No LLM misses conversational understanding. D: Pure SQL is rigid; can't handle natural language.

---

## Final week checklist

### Day 7 (1 week before exam)

- **Chapters to review:** Ch02 (RTO/RPO, HA), Ch05 (DB decision tree), Ch08 (Gemini models), Ch13 (Cloud Deploy, Terraform)
- **Task:** Read "Exam traps" from all 16 chapters; quiz yourself on top 30 traps (write answers, don't peek).
- **FinOps task:** Run `gcloud compute instances list --format='table[box](MACHINE_TYPE, ZONE, STATUS)' | wc -l` across all projects; identify idle instances (0% CPU for 7 days via Cloud Monitoring).
- **Security task:** Enable Org Policy constraint (`compute.requireShieldedVm: true`) on a test project; verify it blocks Compute Engine instance creation without Shielded VM features.

### Day 6

- **Chapters:** Ch03 (VPC connectivity), Ch04 (compute decision tree), Ch07 (migration 6 Rs), Ch14 (TCO, FinOps).
- **Decision trees:** Memorize: compute choice (latency vs cost vs control), database choice (consistency vs scale vs cost), DR pattern (RTO vs RPO vs cost).
- **Practice:** 10 exam-style scenario questions (Q1–Q10 from this chapter) with timer (2 min per question).
- **Cloud Build task:** Create a simple Cloud Build trigger on a GitHub repo; deploy to Cloud Run; test canary traffic split in Cloud Deploy.

### Day 5

- **Chapters:** Ch06 (Pub/Sub, Dataflow, BigQuery), Ch09 (GPU/TPU, Kueue), Ch10 (IAM, groups), Ch15 (SLO, alerting).
- **Case studies deep-dive:** Read ch16; map each case study's requirements to GCP services (Altostrat = media + gen-AI; Cymbal = ecommerce + conversational; EHR = healthcare + HIPAA; KnightMotives = automotive + hybrid + autonomous).
- **Migration task:** Use Migration Center to assess a test workload (create dummy on-prem data in Cloud Storage); see recommended target + cost estimate.

### Day 4

- **Chapters:** Ch01 (exam format, WAF, eliminating traps), Ch11 (security, encryption, VPC-SC), Ch12 (compliance, GDPR, HIPAA).
- **Mock exam:** 30-question practice test with 60-min timer. Mix sections: 5 design (Ch02), 4 compute (Ch04), 4 DB (Ch05), 3 networking (Ch03), 3 security (Ch11), 3 compliance (Ch12), 2 FinOps (Ch14), 2 case studies (Ch16), 1 AI (Ch08).
- **Score:** Aim for ≥27/30 (90 %).
- **Review:** Wrong answers; check traps table (this chapter) for root-cause analysis.

### Day 3

- **Chapters:** Ch04 (GKE Autopilot vs Standard, Cloud Run), Ch06 (Dataflow vs Dataproc), Ch13 (Cloud Deploy canary).
- **Hands-on labs:** 1. Create GKE Autopilot cluster; deploy sample app. 2. Create Cloud Run service; set min instances + traffic split. 3. Deploy via Cloud Deploy (manual approval); test rollback.
- **Case-study practice:** 2 scenario questions (Q49, Q50) under a 4-min timer (2 min per question).

### Day 2

- **Chapters:** Quick skim: Ch02 (DR), Ch05 (storage), Ch07 (migration), Ch08 (AI), Ch09 (infra).
- **Weak areas:** Re-read sections where you scored <80% on mock exam.
- **Keyword review:** Scan "Exam traps" table (this chapter); for each trap, close the book and explain *why* it's wrong (30 sec per trap).
- **Sleep well:** 8 hours minimum.

### The night before

**Logistics (30 min)**
- Confirm exam start time (arrive 15 min early if in-person; test webcam + ID if online).
- Have water bottle + light snack (fruit, protein bar).
- Phone off; no notes/books at exam center.
- Know password + pin for proctor sign-in (if remote).

**Final memory (30 min)**
- Skim "Last-minute memory list" (below).
- Do NOT cram new material; review only.
- Visualize yourself reading a question, eliminating options, and arriving at the right answer (confidence boost).

### Exam day strategy

**Pacing: ~2 min per question** (50–60 questions in 120 min = 2 hours)

1. **Read question + scenario completely** (first 20 sec). Identify the constraint (cost, latency, compliance, etc.).
2. **Eliminate obviously wrong answers** (A/B/C/D) using constraint keywords (e.g., "zero downtime" → eliminate manual failover; "lowest cost" → eliminate premium options).
3. **For case-study questions (split-screen):** Use case study summary (architecture, requirements) as backdrop. Don't memorize entire case study; refer to the split-screen if unsure.
4. **Flag for return** (if unsure): Mark question, continue to end of section. Return if time permits.
5. **Multi-select ("Select TWO"):** 
   - Eliminate 1–2 obviously wrong answers first.
   - Pick 2 of remaining 2–3 (if only 2 left, both are likely correct).
   - If all 4 seem plausible, re-read question's exact requirement.

**Constraint-keyword method:**
- "**Minimal ops overhead**" → Managed (Cloud Run, Cloud SQL, Autopilot), avoid self-managed.
- "**Lowest cost**" → Spot VMs, Batch API, batch endpoints, BigQuery (pay-per-scan).
- "**Highest availability (99.95%+)**" → Multi-region failover (Spanner, Global LB), active-active, not single-zone.
- "**Sub-millisecond latency**" → Cache (Memorystore), co-locate, avoid network hops.
- "**Regulatory compliance (GDPR, HIPAA)**" → CMEK, Audit logs, Org Policy (location), VPC-SC, BAA.
- "**Hybrid / on-prem link**" → Cloud VPN, Interconnect, Network Connectivity Center.
- "**Zero downtime**" → Multi-region, health checks, automated failover (not manual).
- "**Batch / offline-friendly**" → Cloud Batch, Dataflow, Pub/Sub, BigQuery (not real-time APIs).

---

### Last-minute memory list (25 facts)

1. **Spanner SLA 99.999%** (multi-region); Cloud SQL Enterprise HA 99.95 % (Enterprise Plus 99.99 %).
2. **Spot VM ~60–91% discount**, 30 s preemption notice, can be reclaimed any time; use for fault-tolerant batch only, not production.
3. **Org Policy** (constraints on what can be configured); **IAM** (who can do what). Both needed for governance and least privilege.
4. **Pub/Sub at-least-once by default**; exactly-once delivery exists for pull subscriptions, otherwise use idempotent consumers or Dataflow dedup.
5. **Cloud SQL point-in-time recovery** replays transaction logs within the log-retention window; snapshots/backups are discrete points.
6. **RPO = replication lag** (cold/warm/hot); RTO = failover time.
7. **VPC-SC blocks API calls** (not network); Firewall blocks packets.
8. **Spend-based CUDs 28 % (1-yr) / 46 % (3-yr)**; Spot VMs ~60–91 % (fault-tolerant only).
9. **Dataflow** (Beam, greenfield); **Dataproc** (Spark, legacy, PySpark).
10. **Cloud Run scales to zero** (cold starts; set min instances for latency); **GKE** pods need warm nodes, and new nodes take time to provision.
11. **Model Garden**: Google Gemini plus partner (e.g. Claude) and open models; **Agent Platform** for orchestration.
12. **Gemini model tiers**: Flash-Lite (fastest/cheapest) < Flash < Pro (most capable, slowest); no fixed latency numbers to memorize.
13. **Agent Platform Sessions** (conversation memory); **RAG Engine** (private knowledge).
14. **Binary Authorization** (signed images only); **Artifact Analysis** (CVE scanning).
15. **Cloud Build** (CI; add scanning steps); **Cloud Deploy** (CD: approvals, canary, verification, rollback).
16. **Terraform state in Cloud Storage + locking** (not git); Terraform Cloud (remote, managed).
17. **SLO burn-rate alerts** (symptom); Metric alerts (noisy).
18. **Assured Workloads** (compliance regimes such as FedRAMP High, CJIS, IL4); **Sovereign Controls / Sovereign Cloud** (EU sovereignty).
19. **Sensitive Data Protection discovery** (find PII at rest); **SDP de-identification in Dataflow** (redact on ingestion).
20. **Confidential VMs** (data in memory encrypted); **Confidential GKE nodes** (pods on Confidential VMs).
21. **Cloud KMS CMEK** (customer controls key in Google); **EKM** (key held in an external key manager outside Google).
22. **Memorystore** (in-memory, sub-millisecond reads); **Cloud SQL read replicas** offload reads but are disk-backed.
23. **Cloud Trace** (distributed tracing); **Cloud Profiler** (CPU hotspots); **Cloud Monitoring** (dashboards, alerts).
24. **AlloyDB columnar engine** (HTAP: transactions + analytics); **BigQuery** (OLAP, not transactional).
25. **Migration Center** (workload assessment + TCO); **Pricing Calculator** (generic SKU pricing).

---

### Can you explain this in 30 seconds? — Self-test checklist

- [ ] RTO vs RPO (and why cold/warm/hot matter)
- [ ] Spanner multi-region: why 99.999 % SLA + zero data loss + strong consistency
- [ ] Cloud Run vs GKE: latency, ops burden, statefulness
- [ ] Org Policy location constraint (residency) vs VPC-SC (exfiltration): why both
- [ ] Pub/Sub + Dataflow: why Dataflow not Cloud Run functions for batch
- [ ] CMEK vs EKM: where key lives, compliance implications
- [ ] Memorystore vs Cloud SQL replicas: in-memory cache vs read scaling
- [ ] Gemini Flash vs Pro: latency trade-offs
- [ ] RAG Engine vs Vector Search: when each is used
- [ ] Agent Platform Sessions: why needed for multi-turn conversation
- [ ] Canary deployment: automated verification + auto-rollback
- [ ] Cost traps: Spot VMs, CUD vs SUD, Recommender
- [ ] HIPAA: CMEK + audit logs + BAA (not just encryption)
- [ ] GDPR: Org Policy location + VPC-SC + CMEK (not just region)
- [ ] Binary Authorization: signed images only, supply chain security
- [ ] Cloud Build → Artifact Registry → Cloud Deploy: CI/CD pipeline
- [ ] SLO burn-rate: alert only if user-facing impact (not metric spikes)
- [ ] Horizontal vs vertical scaling: latency vs throughput
- [ ] Dataflow vs Dataproc: when each (Beam vs Spark)
- [ ] GKE Autopilot vs Standard: managed nodes vs custom control
- [ ] Cloud SQL HA: automatic failover to the standby (zero data loss, short but non-zero RTO)
- [ ] Filestore: shared NFS for GKE + Compute Engine
- [ ] Cloud Armor: DDoS + WAF at edge
- [ ] IAP: internal apps; Application-level auth for public
- [ ] Terraform state: Cloud Storage + locking (or Terraform Cloud)
- [ ] VPA vs HPA: right-sizing vs scaling out
- [ ] Cold/Pilot Light/Warm/Hot DR: RTO/RPO trade-offs
- [ ] Spot VMs: fault-tolerant batch only
- [ ] Cluster Autoscaler: scales nodes; HPA scales pods
- [ ] Denial Policies: override allows (if any deny, blocked)

---

*End of Chapter 17 Fragment*
