# Chapter 02 — Requirements, High Availability and Disaster Recovery Design

## Exam objectives covered

- **1.1 Business requirements:** functional and non-functional requirements extraction; business continuity plan; cost optimization; supporting application design; integration patterns with external systems; movement of data; design decision trade-offs; workload disposition (build, buy, modify, deprecate); success measurements (KPI, ROI, metrics); security & compliance; observability.
- **1.2 Technical requirements:** HA & fail-over; scalability; performance & latency; backup & recovery.
- **4.1 Technical processes (partial):** disaster recovery.
- **4.2 Business processes (partial):** business continuity.

## What the exam really tests

- **Requirement extraction under ambiguity:** identify hidden functional vs non-functional constraints from narrative scenarios; map business pain to technical solutions (e.g., "99.9 % availability" → multi-zone/regional HA is normally enough; multi-region is for region-failure DR or 99.99 %+ targets).
- **SLA arithmetic:** calculate composite availability for serial/parallel dependencies; understand why an availability target is not the same as an RTO and why RPO and RTO are independent.
- **DR pattern fit:** when backup-restore (cold) vs pilot light vs warm standby vs active-active; cost-recovery trade-offs; testing and runbooks.
- **HA per tier:** regional MIG ≠ zonal; Cloud SQL HA vs Spanner multi-region; stateless services vs persistent state failover; load balancer role in failover.
- **Scalability vs availability trade-offs:** horizontal scales HA but adds complexity; vertical has cost ceiling; autoscaling requires observability feedback loops.
- **Build/buy/modify/deprecate decisions:** when to greenfield vs partner vs migrate; cost-benefit of modernising legacy components; sunsetting timeline constraints.
- **Integration failure modes:** API retry/circuit-breaker patterns; async (queue) + idempotency vs sync blocking; external SLA as constraint on your SLA.
- **Business continuity realism:** plans must be tested; Gemini Cloud Assist role in automation and troubleshooting; RTO/RPO targets must match budget and recovery infrastructure.

## Core concepts

### Functional vs non-functional requirements

**Functional requirements:** what the system *does*. Examples:
- Accept user payments via Visa/Mastercard/PayPal.
- Generate PDF invoices monthly.
- Query inventory in <200 ms.
- Sync with external accounting system (weekly batch).

**Non-functional requirements:** how well it must do it. Examples:
- Availability: ≥99.9 % uptime (SLA).
- Latency: p99 ≤ 100 ms for search queries.
- Throughput: ≥10,000 concurrent users.
- Consistency: eventual within 5 seconds (product catalog updates).
- Data retention: e.g. HIPAA requires retaining compliance documentation for 6 years (retention of medical records themselves is set by state law).
- Disaster recovery: RPO ≤ 1 hour, RTO ≤ 4 hours.

**Exam trap:** a scenario mentioning "24/7 support chatbot" is *functional*; "must serve 1M requests/day with <50 ms latency" is *non-functional*. The question often hides requirements in narrative; extract both.

### KPIs, ROI, and success metrics

Business leaders measure success via:

| Metric | Definition | GCP angle |
|---|---|---|
| KPI | Key Performance Indicator (business-visible: revenue, customer churn, uptime %). | Query in Cloud Monitoring; SLO burn-rate alerts notify business impact. |
| ROI | Return on Investment: (Benefit − Cost) / Cost over a period. | TCO via Migration Center; OpEx (GCP bills) vs CapEx (on-prem). |
| Availability/SLA | Customer-facing % uptime. | Cloud SLA monitoring; composite for multi-service architectures. |
| Latency (p50/p99) | Response time percentiles. | Cloud Trace, Monitoring; BigQuery percentile queries. |
| Cost per transaction | OpEx / transaction count. | BigQuery cost analysis + Monitoring custom metrics. |
| Time-to-market | Weeks to launch feature. | CI/CD + Cloud Deploy speed (rolling/canary/blue-green). |

**Exam angle:** scenarios supply numbers ("reduce costs 40 %", "improve uptime from 95 % to 99.9 %"). Your architecture must quantify trade-off (e.g., an illustrative "active-active DR costs substantially more but cuts RTO from 1 day to minutes").

### SLI, SLO, SLA

- **SLI (Service Level Indicator):** a measurable metric. E.g., "request latency p99 < 100 ms", "successful requests / total requests".
- **SLO (Service Level Objective):** a target for the SLI. E.g., "SLI ≥ 99.9 % monthly".
- **SLA (Service Level Agreement):** customer-facing promise (often with financial penalty for breach). E.g., "Cloud SQL Enterprise edition HA: 99.95 % monthly uptime SLA" (Enterprise Plus: 99.99 %).

**Composite SLOs:** if your app requires both Cloud SQL (99.95 % SLA) and Cloud Run (99.95 % SLA), and they are *serial* (both must work), the composite SLA is 0.9995 × 0.9995 ≈ **99.90 %**. If a request can succeed via *either* of two redundant backends (parallel), SLA improves: 1 − (1 − 0.9995)² ≈ **99.99998 %** (in theory; real systems are limited by shared dependencies such as the load balancer or a common database).

| Availability (%) | Annual downtime | Monthly downtime | Comment |
|---|---|---|---|
| 99.0 | 3.65 days | 7.2 hours | Single data center. |
| 99.9 | 8.76 hours | 43 min | Multi-zone HA typically sufficient. |
| 99.95 | 4.38 hours | 21.6 min | Multi-zone HA with managed services (e.g. Cloud SQL Enterprise HA, Cloud Run). |
| 99.99 | 52.6 min | 4.3 min | Regional/multi-zone with strong automation, or multi-region for region-failure tolerance. |
| 99.999 | 5.26 min | 26 sec | Extreme (e.g., Spanner multi-region SLA, Tier-1 telcos). |

### RTO and RPO

- **RTO (Recovery Time Objective):** how long it takes to recover from failure and resume normal operations. E.g., "1 hour RTO" = restored and in production ≤1 h after failure detected.
- **RPO (Recovery Point Objective):** maximum acceptable *data loss*. E.g., "1-hour RPO" = you accept losing up to 1 hour of transactions if you fail over.

**Relationship:** RPO and RTO are independent targets (one is data loss, the other is downtime); neither is derived from the other. If RPO = 1 hour and RTO = 4 hours:
- Database must have a recovery point ≤1 h old (hourly backups or continuous log shipping / PITR, or a replica).
- Recovery procedure and infrastructure must be tested to recover in ≤4 h.
- **Daily backup-restore does not fit** (RPO up to 24 h); hourly backups with a rehearsed restore, a **pilot light** (cross-region replica) or **warm standby** do, at different cost.

### Disaster Recovery patterns

| Pattern | Backup cadence | Failover time | Data loss | Cost | GCP implementation | When to use |
|---|---|---|---|---|---|---|
| **Backup & Restore (Cold)** | Daily/weekly snapshots | 8–24 hours | Hours to days | Lowest | Cloud Storage + scheduled Cloud SQL backups (or Backup and DR Service); recovery via manual restore. | Non-critical workloads, acceptable downtime ≤24 h, tight budget. |
| **Pilot Light** | Near-real-time replication | 10 min – 1 hour (indicative) | Minutes to hours | Low | Cloud SQL cross-region read replica (promote on failure) with minimal compute pre-provisioned in the DR region (IaC to scale it out); switch DNS/LB on failure. | Moderate RPO/RTO, cost-conscious. |
| **Warm Standby** | Continuous replication (async) | Minutes (indicative) | Seconds to minutes (replication lag) | Moderate | Scaled-down but running stack in a second region: Cloud SQL cross-region replica (promote) or Cloud SQL Enterprise Plus advanced DR; MIG/GKE/Cloud Run already deployed; failover via global LB or DNS. | SLA ≥99.9 %; active-passive architecture acceptable. |
| **Hot/Active-Active** | Synchronous replication + distributed consensus | Seconds or none | ≈0 | Highest | Spanner multi-region (synchronous quorum across regions); multi-region Firestore; multi-region MIGs / Cloud Run behind a global load balancer. | SLA ≥99.99 %+; zero-downtime expected by users. |

**Key trade-offs:**
- Cold (backup-restore): test restore regularly (RTO often longer in practice than promised).
- Pilot light: RPO depends on replication lag; must verify lag (Cloud SQL replication lag monitoring).
- Warm standby: standby infra must be sized; automatic failover adds complexity (DNS propagation ≠ instant, app-level health checks).
- Hot: synchronous replication adds write latency and cost; Spanner multi-region configurations (two read-write regions plus a witness region) cost roughly 3× a regional configuration at list price. Note that Spanner *regional* configs already span 3 zones (99.99 % SLA); multi-region raises the SLA to 99.999 %.

### High-availability design per tier

#### Compute

| Tier | Zonal HA? | Multi-zone / Regional HA | Multi-region HA |
|---|---|---|---|
| **Compute Engine (GCE)** | No (single VM = single point of failure). | **Managed Instance Group (MIG) regional** (VMs spread across up to 3 zones in one region; auto-healing, rolling updates). | MIGs in multiple regions + load balancer (Global HTTP(S) LB or Cloud Load Balancing). |
| **GKE Standard** | Single-zone (zonal) cluster = control plane and nodes in one zone; a zone outage = cluster outage. | **Regional GKE** (control plane replicated across 3 zones; nodes spread across zones, by default 3). | **Multi-cluster Ingress / Gateway** (GKE Enterprise fleet) across clusters in different regions + a replicated data tier (e.g. Spanner). |
| **GKE Autopilot** | Not the default; Autopilot clusters are regional by default. | Regional by default (nodes auto-distributed, no node admin overhead). | Multi-cluster via GKE Fleet + multi-cluster Ingress/Gateway. |
| **Cloud Run** | Fully managed; automatically runs across multiple zones within a region. | Same (regional service; zone HA is built in). | Deploy the service in multiple regions behind a global external Application Load Balancer (serverless NEGs); the LB fails over between regions. |
| **Cloud Run functions** | Fully managed; multi-zone within a region. | Same as Cloud Run (single-region auto-HA). | Multi-region deployment behind a global LB + custom failover. |

#### Storage & Databases

| Service | HA mode | RTO/RPO | Failover |
|---|---|---|---|
| **Cloud SQL** | HA configuration (primary + standby in a second zone of the same region; synchronous replication to regional disk). | RTO typically about a minute or two (⚠ verify); RPO ≈0. | Automatic failover (zonal only; does not protect against a region outage). |
| **Cloud SQL** (cross-region) | Cross-region read replica (async); Enterprise Plus adds advanced DR (replica failover/switchover). | RTO minutes to tens of minutes (promotion + app switch); RPO = replication lag (typically seconds to minutes). | Promotion is a manual or scripted action; application-driven (e.g., switch connection string). |
| **AlloyDB** | Regional HA (primary + standby in different zones; 99.99 % SLA including maintenance); cross-region replicas for DR. | Zonal failover typically under a minute; RPO ≈0 within the region. | Automatic zonal failover; cross-region promote is manual. |
| **Cloud Spanner** | Regional config (3 zones, 99.99 %) or multi-region config (read-write regions + witness region, synchronous quorum, 99.999 %). | RPO = 0 (no data loss); RTO ≈ 0 (transparent failover, no manual step). | Automatic failover (leader re-election within the quorum). |
| **Firestore** | Regional (replicated across zones) / multi-region (replicated across regions). | Regional SLA 99.99 %; multi-region 99.999 %. | Automatic (managed by Firestore). |
| **Bigtable** | A cluster is zonal; HA/DR comes from replication across clusters (same or different regions). | Single cluster = no zone-failure tolerance; replicated clusters = async (eventual consistency). | Multi-cluster routing app profiles fail over automatically; single-cluster routing needs a manual switch. |
| **Cloud Storage** | Multi-region and dual-region buckets are geo-redundant (any storage class); regional buckets are zone-redundant. | Data survives a full region failure for multi-/dual-region; replication is async (turbo replication on dual-region targets 15 min RPO). | Automatic; the bucket endpoint does not change. |

#### Networking & Load Balancing

- **Global external Application Load Balancer (HTTP(S)):** routes traffic across regions; health checks per backend (regional MIG). Failed region automatically drains. Failover = seconds (health-check interval + route update).
- **Cloud CDN:** caches content at edge; on origin failure, returns stale content or falls back to other origins.
- **Cloud DNS:** globally distributed; no single point of failure. If your DNS provider fails, app is unreachable (use Google Cloud DNS).
- **VPC / Shared VPC / VPC peering:** a VPC network is global (subnets are regional), so workloads in different regions communicate over Google's backbone using internal IPs; use Interconnect or Cloud VPN (HA VPN) for hybrid connectivity.

### Scalability patterns

**Horizontal scaling:** add more instances (GCE VMs in MIG, GKE Nodes, Cloud Run containers). Pros: no ceiling; naturally supports HA (spread replicas). Cons: stateless required or distributed state (Firestore, Memorystore), adds orchestration complexity.

**Vertical scaling:** bigger machine (GCE e2-standard-8 → e2-standard-32). Pros: simple, no refactor. Cons: single point of failure, cost ceiling, eventual latency plateau.

**Autoscaling triggers:**
- CPU utilization (e.g., >70 % → add instance).
- Memory (less common; requires app instrumentation).
- Custom metric (e.g., Pub/Sub queue depth; Cloud Tasks queue length).
- Request rate (e.g., >1000 req/s → add replica).

**GCP autoscaling services:**
- **Compute Engine:** MIG autoscaler (based on CPU/custom metrics).
- **GKE:** Horizontal Pod Autoscaler (HPA, based on CPU/memory), Vertical Pod Autoscaler (VPA, recommend bigger Pods), Cluster Autoscaler (CA, add Nodes).
- **Cloud Run:** automatic (driven by request concurrency and CPU utilization); tune concurrency, min/max instances.
- **App Engine:** automatic; config via `app.yaml` autoscaling settings.

**Queue-based load leveling:** decouple spiky requests via Pub/Sub / Cloud Tasks. Producer publishes to queue; consumer (worker) scales based on queue depth. Decouples producer from consumer; absorbs traffic bursts.

### Performance and latency design

**Latency sources:**
- Network round-trip (RTT): <1 ms within zone, ~1–5 ms across zones in a region, ~50–150 ms intercontinental.
- Database query (order of magnitude): sub-millisecond to ~1 ms for a Memorystore cache hit, single-digit to tens of ms for an indexed Cloud SQL/Spanner point read, much more for scans or cross-region synchronous writes.
- Application processing: depends on code.
- Load balancer path: <1 ms.

**Optimization strategies:**
- **Caching (Memorystore Redis/Memcached):** cache frequently accessed data (user profiles, product catalog). TTL = 1–24 hours depending on consistency tolerance.
- **CDN (Cloud CDN):** cache static content and cacheable API responses at edge. Cache-Control headers + origin must support.
- **Database read replicas:** scale read-heavy workloads (reporting, analytics). Eventual consistency acceptable? Use an async read replica (Cloud SQL) or Spanner stale reads; strong consistency everywhere? Spanner.
- **Batch + async:** don't wait for every operation. Use Dataflow for ETL, Cloud Tasks for delayed work, Pub/Sub for fanout.
- **Geolocation routing:** serve users from nearest region (Global LB + regional backends).

**Latency vs cost trade-off:** multi-region HA costs a multiple of single-region HA (e.g. Spanner multi-region ≈3× regional at list price). Latency SLO and user geography drive architecture: users spread across continents with a tight latency SLO → multi-region serving; users concentrated in one geography → regional + cache.

### Design trade-off matrix

| Driver | Low Cost | High Availability | High Performance | Low Complexity |
|---|---|---|---|---|
| **Compute** | Single GCE VM (cheapest). | Regional MIG + Global LB (HA). | Multi-region MIG + Global LB (HA + low latency). | Cloud Run (serverless, no scaling config). |
| **Database** | Zonal Cloud SQL (no HA). | Cloud SQL Regional HA. | Spanner multi-region (global, low latency, sync). | Firestore (managed, easy). |
| **Storage** | Regional Cloud Storage (Standard). | Dual-/multi-region Cloud Storage (geo-redundant). | Multi-region + CDN (global cached reads). | Cloud Storage (handles everything). |
| **Integration** | Batch ETL (nightly). | Event-driven (Pub/Sub, Eventarc). | Real-time streaming (Dataflow). | Cloud Run functions (managed). |

**Exam pattern:** "lowest cost + 99.9 % HA" → regional services + caching, no multi-region. "Globally distributed users + strong consistency + 99.999 %" → Spanner multi-region (expensive); "99.99 % in one geography" → regional HA with managed services (e.g. Spanner regional, Cloud SQL Enterprise Plus, AlloyDB) is often enough. "Fast time-to-market" → Cloud Run + Cloud SQL (managed, less ops overhead).

### Build, buy, modify, deprecate (workload disposition)

For each application tier, decide:

| Decision | Rationale | GCP fit | Example |
|---|---|---|---|
| **Build** | Custom, competitive advantage, no suitable vendor. | Use Cloud Build (CI/CD), GKE (containers), AlloyDB (custom schema). | Custom ML model for product recommendations (Altostrat). |
| **Buy** | Standard functionality, fast time-to-market, low opex. | SaaS or marketplace products (e.g. Looker for BI) or managed services instead of self-run software; integrate via APIs. | ERP (SAP, Oracle → lift-and-shift or Cloud Migration). |
| **Modify** | Vendor product + custom extensions (hybrid). | Apigee (API gateway + custom policies), Cloud Run (wrap legacy with modern API). | Add payments layer to legacy e-commerce (Cymbal Retail). |
| **Deprecate** | Legacy system; plan retirement. | Migrate or archive the data first (Database Migration Service, Datastream, Cloud Storage Archive class), then set a sunset date. | Turn off on-prem data center by Q4 2027; migrate to Cloud Spanner. |

**Exam angle:** scenario has legacy mainframe inventory system; "which best reduces cost and team burden?" → Buy (SaaS) often wins; "needs custom analytics on sensitive data" → Modify (wrap + custom layer); "sunset timeline 18 months" → Deprecate (plan migration path + parallel run).

### Business continuity plan (BCP)

A BCP answers:
1. **Scenarios:** which events trigger the plan? (data centre failure, ransomware, key staff unavailable, regional outage, vendor SaaS downtime).
2. **Detection:** how do we know a failure occurred? (Monitoring alerting, manual report, SLA breach).
3. **Notification:** who gets called? (incident commander, ops team, business stakeholders, customers).
4. **Runbook:** step-by-step recovery. Who executes? (automation + human). Dependencies on what?
5. **Testing:** monthly/quarterly drill. Measure actual RTO/RPO (not theoretical). Update runbook from findings.
6. **Communication:** customer comms (status page, email), internal (war room, incident post-mortem).

**Gemini Cloud Assist role:** an AI assistant in the Google Cloud console that helps with designing and deploying (architecture suggestions, IaC generation), troubleshooting (investigations that analyse logs, metrics and config to propose likely root causes and next steps) and optimizing workloads. Treat it as an accelerator for runbooks and design reviews, not a replacement for tested procedures.

### Integration patterns with external systems

| Pattern | Mechanism | When | Pros | Cons | GCP implementation |
|---|---|---|---|---|---|
| **Sync API** | App calls external API, waits for response. | Real-time data (rate lookup, payment auth). | Simple, immediate. | Slow if external API slow; cascading failure (external SLA becomes yours). | Cloud Run → external API (add circuit-breaker + retry logic). |
| **Async messaging** | App publishes event to queue; external system consumes. | Data sync, order processing. | Decoupled; external latency doesn't block app. | Eventual consistency; dead-letter queue (DLQ) for failures. | Pub/Sub (push subscription or consumer) or Cloud Tasks (HTTP push with retries) to call the external system. |
| **Batch ETL** | Periodic (nightly) data export, external import. | Reporting, data warehouse sync. | Economical; low complexity. | High latency (RPO = 1 day); no real-time. | Cloud Dataflow or Dataproc for ETL; BigQuery export to GCS. |
| **Webhook** | External system calls your API on events. | 3rd-party SaaS notifications (payment confirmed, inventory updated). | Immediate notification. | External SLA is critical; idempotency key needed (retries). | Cloud Run + Eventarc listener; idempotency via an idempotency key stored with a transactional create-if-absent (Firestore/Cloud SQL). |
| **File transfer** | SFTP, GCS bucket, or EDI interchange. | Legacy integrations, batch data. | Works with isolated systems. | Slow; no visibility; error-prone. | Cloud Storage buckets + Storage Transfer Service (managed file movement), Datastream (CDC for databases). |

**Exam pattern:** "external payment API adds 500 ms latency; customer complaints about slow checkout." → Don't call it in request path; use async (queue payment, return immediately, retry on failure). "Reconciliation with vendor inventory every hour." → Batch ETL at :00 (Dataflow), not sync API.

## Decision guides

### When to choose each DR pattern

→ **Backup & Restore (Cold):** Non-critical workloads (internal tools, dev/test environments), acceptable downtime ≤24 h, tight budget. RTO 8–24 h, RPO 24 h.

→ **Pilot Light:** Moderate criticality (SLA 99.5–99.9 %), budget-conscious. Data replicated near-real-time (Cloud SQL cross-region replica, Spanner/Bigtable replication); compute in the standby region minimal or created on demand via IaC. Indicative RTO 10–60 min, RPO minutes.

→ **Warm Standby:** High criticality (SLA 99.9–99.95 %), business justified cost. Standby region running replicas, health checks active, failover via DNS or load balancer. Indicative RTO of minutes, RPO = replication lag (seconds to minutes).

→ **Hot / Active-Active:** Critical systems (financial, healthcare, 99.99+ % SLA) or new greenfield apps with strong consistency requirements. Spanner multi-region, multi-region Firestore, or multi-region serving tiers behind a global load balancer. RTO near zero, RPO ≈0 for synchronously replicated data.

### When to scale horizontally vs vertically

→ **Horizontal:** stateless services (Cloud Run, containers, API servers), natural HA benefit, workload bursty, team comfortable with orchestration. Limit: cost grows linearly.

→ **Vertical:** tightly coupled system (legacy monolith), database tier (more RAM = faster queries), team minimal ops knowledge. Limit: eventual latency plateau, single-machine cost ceiling, still no HA.

→ **Hybrid:** database tier vertical (bigger machine), API tier horizontal (MIG). Cache (Memorystore) horizontal for read scaling.

### Choosing a database for your scenario

→ See **Ch05 (databases)** for detailed decision tree. Quick version: **OLTP (transactional)** → Cloud SQL (relational), Cloud Spanner (global, HA), Firestore (NoSQL, managed). **OLAP (analytical)** → BigQuery. **NoSQL key-value** → Firestore, Bigtable. **Time-series** → Bigtable, BigQuery + clustering. **Real-time streaming** → Bigtable + Dataflow. **Strong consistency + global HA** → Cloud Spanner.

### Integration pattern decision tree

→ Does the remote system need real-time data? → YES: Sync API (with circuit-breaker) or async Pub/Sub + webhook. NO: Batch ETL (nightly). Is the remote API often slow or down? → Use async + Pub/Sub (decouple). Must data be transactionally consistent? → Sync API or distributed transaction (complex; prefer async + idempotency keys). Can you tolerate duplicate messages (e.g., payment twice)? → Pub/Sub with idempotent consumer. Data sensitivity high? → Encrypt in transit (TLS), use service-to-service auth (Workload Identity, IAP).

## Well-Architected Framework lens

| Pillar | Application to this chapter |
|---|---|
| **Operational Excellence** | BCP + runbooks tested regularly; SLO dashboards; incident communication playbook; Gemini Cloud Assist for design review and runbook drafting. |
| **Security & Compliance** | DR infrastructure must meet same security posture as production (encryption at rest/in transit, IAM, audit logs); sensitive data in replicas handled correctly; integration auth (API keys in Secret Manager). |
| **Reliability** | SLI/SLO/SLA targets drive architecture (HA design); RTO/RPO targets justify multi-region; design for graceful degradation (circuit-breaker, retry backoff). |
| **Performance Optimization** | Latency SLO (p99 <100 ms) → multi-region or caching; throughput SLO (10k concurrent) → horizontal scaling. Measure via Cloud Trace / Monitoring. |
| **Cost Optimization** | Trade-off: low-cost single-region + acceptable downtime vs high-cost multi-region HA. Right-size instances (Recommender); use Spot for non-critical workloads. |
| **Sustainability** | Multi-region = higher energy cost; prefer single-region with HA (Cloud SQL Regional HA) if RPO/RTO allow. Consolidate workloads onto fewer, larger machines if sustainable. |

## AI/GenAI angle

- **Gemini Cloud Assist:** helps with design review and architecture suggestions (including help spotting single points of failure and generating Terraform for failover infrastructure) and incident response (investigations that analyse logs/metrics and suggest likely root causes and next steps). Review its output; it does not replace tested DR procedures.
- **LLM-driven testing:** use Gemini to draft chaos-engineering test cases or review a BCP runbook for ambiguities.
- **Predictive scaling:** Compute Engine MIGs have built-in predictive autoscaling; for custom demand forecasting use BigQuery ML or Agent Platform models and feed the result into scaling decisions.
- **Sensitive data in replicas:** a DR replica holds the same data as production, so it needs the same controls (CMEK keys available in the DR region, IAM, VPC Service Controls, audit logs, data-residency constraints). Do not mask the standby copy; masking would break restore. Use Sensitive Data Protection to de-identify data that flows into AI prompts, logs or analytics copies.

## Case-study hooks

**Altostrat Media:**
- HA: 24/7 support chatbot (Cloud Run + regional MIG for redundancy; multi-region failover via Global LB).
- Scalability: metadata extraction (Cloud Run functions) scales horizontally; queue-based (Pub/Sub) decoupling for transcoding workers.
- Integration: legacy on-prem ingestion → Datastream (near-real-time DB replication); build runbook for hybrid failover.
- RTO/RPO: SLA ≥99.9 % → Cloud SQL Regional HA + Cloud Run multi-region.

**Cymbal Retail:**
- Requirements: "reduce data-centre costs" → migrate to cloud; "improve discoverability" → Agent Search (or Vector Search + Gemini embeddings).
- HA: e-commerce platform (illustrative target SLA ≥99.95 %) → regional MIG/GKE/Cloud Run + Cloud SQL HA, with a cross-region replica if region-level DR is required.
- Integration: legacy SFTP/ETL batch supplier feeds → Cloud Storage landing bucket + Storage Transfer Service / batch ETL (Dataflow); Datastream for database CDC.
- Human-in-the-loop: AI enrichment (Gemini for image generation) → manual review UI (Cloud Run) → database update (idempotent).

**EHR Healthcare:**
- Hard requirement: 99.9 % availability, HIPAA compliance → Cloud SQL Regional HA (sync replica) in same region; encryption (Cloud KMS), audit logs.
- Hybrid: insurers on-prem (legacy) → Cloud Interconnect (low-latency, HA) or Cloud VPN.
- Disaster recovery: 2nd region for compliance (HIPAA allows; data residency in US) → warm standby (Cloud SQL cross-region replica + pre-deployed scaled-down stack).
- Runbook: failover to 2nd region; update DNS; smoke test; customer notification (SLA 4-hour RTO).

**KnightMotives Automotive:**
- Workload disposition: legacy mainframe (supply chain) → Deprecate (plan 18-month migration to cloud ERP); custom autonomous-driving simulation → Build (on GKE, multi-region for resilience); vehicle telematics → Buy (partner platform) + integrate via Pub/Sub.
- HA: real-time vehicle data (connected vehicles) → regional Bigtable + multi-region Dataflow processing.
- Security & DR: GDPR (EU data residency) → separate GCP project for EU data; Assured Workloads + Sovereign Cloud (if needed); backup to secondary EU region.

## Exam traps and keyword → answer

1. **"99.9 % availability with zonal GKE"** → ❌ Zonal cluster = all nodes in one zone; failure = entire outage. ✅ Use **regional GKE** (nodes auto-spread to 3 zones).

2. **"RPO of 1 hour" implies daily backup schedule** → ❌ Daily snapshots miss 23 hours of transactions. ✅ Hourly snapshots or continuous replication / PITR (Cloud SQL read replica, binary-log based point-in-time recovery).

3. **"Multi-region Spanner = twice the cost"** → ⚠ Closer to ~3× the list price of a regional configuration (multi-region configs have two read-write regions plus a witness region; replica count and storage price drive the cost). A regional Spanner config is already 3-zone and 99.99 %; pay for multi-region only for 99.999 % or multi-region reads/writes.

4. **"DR is documented but never tested"** → ❌ Untested disaster recovery never works. ✅ Test the runbook regularly and after major changes (measure actual RTO/RPO; update based on findings).

5. **"External API SLA 99 % is acceptable for our 99.9 % SLA"** → ❌ Composite SLA = yours × theirs; 99.9 % × 99 % = 98.9 %. ✅ Either negotiate better SLA, use async (decouple), or accept your SLA drops to 98.9 %.

6. **"Horizontal scaling solves latency"** → ⚠ More instances don't reduce per-request latency; they increase *throughput*. For latency: optimize queries, cache, use CDN, multi-region. Horizontal scaling helps if bottleneck is request *queueing* (backlog).

7. **"Vertical scaling (bigger VM) = infinite scalability"** → ❌ Single machine has CPU/memory ceiling; single point of failure. ✅ Vertical for database tier; horizontal for stateless app tier.

8. **"Warm standby region with 0 RPO"** → ❌ Only hot (sync replication) guarantees 0 RPO; warm (async) has lag. ✅ RPO = replication lag (typically <1 min for Cloud SQL async).

9. **"Build vs Buy is purely cost-driven"** → ❌ Time-to-market, strategic value, and team skills matter. "Buy" (vendor SaaS) often wins on speed and opex; "Build" wins on customization. "Deprecate" is cost-neutral but reduces maintenance burden.

10. **"The standby region needs fewer security controls"** → ❌ A DR replica contains the same sensitive data as production. ✅ Apply the same controls (CMEK keys available in the DR region, IAM, VPC Service Controls, audit logs, residency constraints); do not mask the replica or restore would be useless.

11. **"Pilot light = pilot can serve traffic"** → ❌ Pilot light is a *standby* with minimal infrastructure (data replicated, compute minimal or created on demand). ✅ When activated, compute must be scaled out and traffic rerouted (DNS/LB change), so standby ramp-up takes time.

12. **"Composite availability = SLA1 + SLA2 for independent services"** → ❌ Addition is wrong. ✅ Serial (both must work): 0.999 × 0.999 = 0.998001 (≈99.8 %); Parallel (either): 1 − (1−0.999)² = 0.999999 (≈99.9999 %).

## Quick-reference cheat table

### Disaster Recovery Patterns Comparison

| Pattern | Backup interval | Failover time | Data loss | Cost | GCP tools | Use case |
|---|---|---|---|---|---|---|
| Cold | Daily–weekly | 8–24 hours | Hours–days | Lowest | Cloud SQL snapshots, GCS archive | Non-critical, acceptable downtime. |
| Pilot Light | Continuous replication | 10–60 min (indicative) | Minutes (replication lag) | Low–moderate | Cloud SQL cross-region replica, IaC for DR compute | Moderate SLA (99.5–99.9 %). |
| Warm Standby | Continuous (async) | Minutes (indicative) | Seconds–minutes (lag) | Moderate–high | Cloud SQL Regional HA + cross-region replica, pre-deployed scaled-down stack | High SLA (99.9–99.95 %). |
| Hot / Active-Active | Sync (quorum) | Seconds or none | ≈0 | High–very high | Spanner multi-region, Firestore multi-region, global LB over multi-region serving | Critical (99.99–99.999 %), zero downtime. |

### Availability & Downtime Table

| SLA (%) | Annual downtime | Monthly downtime | Weekly downtime |
|---|---|---|---|
| 99.0 | 3.65 d | 7.2 h | 1.7 h |
| 99.5 | 1.83 d | 3.6 h | 50 min |
| 99.9 | 8.76 h | 43 min | 10 min |
| 99.95 | 4.38 h | 21.6 min | 5 min |
| 99.99 | 52.6 min | 4.3 min | 60 sec |
| 99.999 | 5.26 min | 26 sec | 6 sec |

### HA Design per Tier

| Tier | No HA | Regional HA | Multi-region HA |
|---|---|---|---|
| **Compute** | Single GCE VM | MIG (regional, ≥2 zones) | MIGs + Global LB |
| **GKE** | Single-zone | **Regional (default ≥3 zones)** | Multi-cluster Ingress |
| **Cloud Run** | N/A (always multi-zone within its region) | Built in | Multi-region deployment + global LB |
| **Database (SQL)** | Zonal (no HA) | Regional HA (sync replica in second zone) | Cross-region replica (async, promote on failure) |
| **Spanner** | N/A | Regional config (3 zones, 99.99 %) | Multi-region config (sync quorum, auto-failover, 99.999 %) |
| **Storage** | Single-zone/zonal disks | Regional bucket (zone-redundant) | Dual-/multi-region bucket (geo-redundant) + CDN |

## Practice scenarios

### Scenario 1: E-commerce with 99.95 % SLA requirement

Your company runs an online store on Compute Engine (3 VMs in a single zone). You receive a SLA requirement: "99.95 % monthly uptime". The current architecture is a single-zone deployment. Which combination of changes is necessary to meet the SLA?

- A) Upgrade VMs to larger instances and enable Compute Engine live migration.
- B) Move to a regional Managed Instance Group (MIG) + Global HTTP(S) Load Balancer.
- C) Move to Cloud Run (serverless) in a single region.
- D) Deploy Spanner database and upgrade network to Dedicated Interconnect.

**Answer: B.** Explanation: 99.95 % SLA requires high-availability architecture (zone redundancy). Regional MIG + Global LB provides: (1) ≥2 zones within a region (auto-healing, rolling updates), (2) global load balancer with health checks (automatic failover across zones), (3) managed service (no VM patching burden). Cloud Run (C) is multi-zone HA, but it means re-platforming the application, and the end-to-end SLA still depends on the database tier; the question asks for the change needed to fix the single-zone design. Spanner and Interconnect (D) address the data tier and hybrid connectivity, not the single-zone compute weakness; regional HA is sufficient for 99.95 %.

### Scenario 2: DR for regulated healthcare data

Your HIPAA-regulated EHR system uses Cloud SQL (zonal, no replication). Requirements: a single-zone failure must be tolerated automatically with no data loss, and after the loss of a whole region the service must be restored within 4 hours, accepting up to 1 hour of data loss. Which configuration fits at reasonable cost?

- A) Daily automated backups to Cloud Storage; cross-region manual restore on failure.
- B) Cloud SQL without HA, plus a cross-region read replica to promote on failure.
- C) Cloud SQL HA (standby in a second zone, synchronous) plus a cross-region read replica to promote in a regional disaster.
- D) Spanner multi-region (two read-write regions plus a witness region, synchronous); automatic failover.

**Answer: C.** Explanation: Zone-level protection needs HA (automatic failover, RPO ≈0 via the synchronous standby); region-level DR with RTO 4 h / RPO 1 h is met by a cross-region async replica (RPO = replication lag, promotion within minutes to tens of minutes), which is a pilot-light/warm-standby style design. Daily backups (A) give an RPO of up to 24 hours. B has no automatic zonal failover (a zone failure would need a manual promotion and risks data loss). Spanner (D) exceeds the requirements and means migrating a MySQL/PostgreSQL workload; cost is not justified.

### Scenario 3: Latency-sensitive personalized recommendation engine

Your Gen-AI recommendation engine reads user embeddings + product catalog data on every request; median read latency is 200 ms, and you have a P99 latency SLO of <100 ms. Which optimization is most effective?

- A) Migrate from Cloud SQL to Spanner (global database).
- B) Deploy horizontal Dataflow pipeline to pre-compute all embeddings hourly.
- C) Upgrade Cloud SQL machines to larger compute (vertical scaling).
- D) Use Memorystore (Redis) for user embeddings cache; embed-refresh every 6 hours.

**Answer: D.** Explanation: The bottleneck is latency (200 ms → 100 ms SLO), not throughput. Spanner (A) is a scale/consistency answer, not a latency fix (synchronous multi-region writes add latency; it is no faster than a cache for hot reads). Dataflow (B) helps with batch pre-compute but doesn't cache the results in fast memory. Vertical scaling (C) improves single-query latency somewhat but plateaus. **Memorystore caching (D)** is most effective: embed queries hit Redis (sub-millisecond), reducing 200 ms to <5 ms. 6-hour TTL is acceptable for recommendations (stale is ok; freshness trade-off). This directly targets the latency bottleneck.

### Scenario 4: Integrating third-party inventory API

Your e-commerce app queries a vendor's inventory API (latency 500 ms, SLA 99 %). Your app SLA target is 99.9 %. Synchronously calling the API on every request breaks your SLA. How should you refactor?

- A) Async: publish inventory-check events to Pub/Sub; worker polls API; update Firestore; request queries Firestore (eventual consistency, <1 sec).
- B) Cache API responses in Memorystore with 1-hour TTL; serve stale inventory if API unavailable.
- C) Add circuit-breaker + exponential backoff retry; keep sync API in request path.
- D) Negotiate 99.9 % SLA with vendor.

**Answer: A.** Explanation: Sync API (500 ms + 99 % SLA) on every request → composite SLA drops to 99.9 % × 99 % = 98.91 % (below target). Caching (B) works but risks stale inventory. Retry (C) improves reliability but doesn't reduce composite latency or SLA. Async (A) decouples inventory-check from request path: request returns immediately, inventory updates asynchronously; Firestore query is <1 sec (no external dependency). The request path no longer depends on the vendor's 99 % SLA, so your own SLA target becomes achievable. Negotiating SLA (D) is unrealistic.

### Scenario 5: Build vs Buy: AI-powered image moderation

Your platform allows user-uploaded images. You must moderate (filter inappropriate content). Build your own ML model (8–12 weeks, team of 2), or use a pre-built Google API such as Cloud Vision API SafeSearch detection (or Gemini multimodal with safety filters)? Business constraint: "market in 6 weeks; moderation quality of the pre-built service is acceptable after a pilot".

- A) **Build:** train custom model on your data; potentially higher accuracy, but 12 weeks (miss market).
- B) **Buy:** pre-built Vision API SafeSearch; about 1 week integration; accuracy acceptable per the pilot.
- C) **Modify:** Vision API for first-pass moderation plus a custom model for edge cases (6–8 weeks); best of both.
- D) **Deprecate:** don't moderate; accept legal risk.

**Answer: B.** Explanation: Market timeline (6 weeks) is the hard constraint. Build (A) misses deadline. Buy (B) meets the timeline with acceptable quality. Modify (C) adds weeks of integration and ML work; 6–8 weeks is still risky. Deprecate (D) is not an option (legal risk). **Buy** wins on time-to-market.

### Scenario 6: Disaster recovery testing cadence

Your company has a BCP with RTO 2 hours and RPO 1 hour for the critical ERP system. The runbook was last tested 18 months ago; it mentions a retired data centre. During a disaster simulation, the restore took 6 hours (exceeding RTO). What is the primary issue?

- A) Recovery infrastructure (standby region) is undersized; needs bigger VMs.
- B) Database too large; restore is slow.
- C) Incorrect RTO/RPO targets (should be 8 hours / 4 hours).
- D) Runbook was out of date; procedures don't match current infrastructure.

**Answer: D.** Explanation: **BCP/DR plans must be tested regularly** (e.g. at least quarterly and after major infrastructure changes). An 18-month-old runbook is stale (infra changes, retired data centre referenced). Actual RTO (6 hours) vs target (2 hours) = gap. Root cause is outdated procedure. Fix: test the runbook on a regular schedule; measure actual RTO/RPO; update based on findings. Undersizing (A) is possible, but the scenario states runbook was untested, not infrastructure. Database size (B) is a symptom, not the root cause; would still fail with outdated runbook. Targets (C) should match business need, not be adjusted down because testing fails.

## Sources

- cloud.google.com/architecture/framework (Well-Architected Framework, all pillars)
- cloud.google.com/docs/enterprise/best-practices-for-running-cost-effective-kubernetes
- cloud.google.com/architecture/disaster-recovery (DR patterns, RTO/RPO, testing)
- cloud.google.com/docs/cloud-sql/high-availability-overview
- cloud.google.com/docs/cloud-sql/replication (async/sync, failover)
- cloud.google.com/architecture/reducing-latency-by-using-cloud-cdn-and-traffic-director
- cloud.google.com/spanner/docs/instance-configurations (regional, dual-region and multi-region configurations; replica layout)
- cloud.google.com/spanner/pricing (regional vs multi-region price ratio)
- cloud.google.com/docs/gke/concepts/regional-clusters (regional GKE HA)
- cloud.google.com/docs/cloud-run/quickstarts/build-and-deploy (Cloud Run HA model)
- Official exam guide v6.1, section 1.1, 1.2, 4.1, 4.2
