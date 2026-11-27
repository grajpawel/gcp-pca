# Chapter 07 — Migration

## Exam objectives covered

From the PCA exam guide (v6.1):
- **1.4 Migration plan (docs + diagrams):** integrate with existing systems; assess & migrate systems/data (Migration Center); migration methodologies, workload testing, network planning, dependency planning; software licence & financial impact.
- **1.5 Future improvements:** cloud/technology improvements; evolution of business needs; cloud-first design.
- **5.1 Advising dev & ops teams:** data & system migration and management tooling.

## What the exam really tests

- **6 Rs decision tree**: knowing when rehost (lift-and-shift), replatform (managed versions), refactor (containerization), repurchase (SaaS), retire, or retain wins—driven by effort, cost, downtime, complexity.
- **Migration Center vs Pricing Calculator**: Migration Center now preferred for TCO because it assesses in-place workloads, factors licensing/BYOL, produces cost estimates *and* recommendations (not just raw pricing).
- **Hybrid & phased migration**: assessing coexistence periods, network planning, dependency mapping, cutover strategies, rollback readiness—realistic constraints, not one-shot moves.
- **Database migration pathways**: homogeneous (same engine) vs heterogeneous (e.g. Oracle → Cloud SQL Postgres) — DMS handles continuous replication, schema conversion challenges, validation.
- **Data transfer decision by size × bandwidth**: online (STS / gcloud storage) when the upload fits in about a week, offline Transfer Appliance when it would take longer (Google's own rule of thumb); formula: hours = GB x 8000 / Mbps / 3600 (account for realistic link utilisation, backup windows).
- **Licensing & financial impact**: BYOL implications (sole-tenant VMs, Windows/SQL Server per-core), double-running costs during cutover, CapEx→OpEx shift impact on TCO, case-study urgency (Cymbal cost-reduction, EHR compliance-driven).
- **Modernization during migration**: don't just rehost—refactor monoliths to containers (Migrate to Containers), upgrade legacy DBs (on-prem MySQL → AlloyDB/Cloud SQL, on-prem SQL Server → Cloud SQL or AlloyDB, MongoDB → Firestore/Spanner), replatform to serverless where applicable.

## Core concepts

### Google Cloud's migration framework (phases)

Google's 4-phase approach, embedded in tools & advice:

| Phase | Goal | Tools | Key decisions |
|-------|------|-------|---------------|
| **Assess** | Discover workloads, measure (CPU, RAM, storage, costs). Identify candidates (easy to hard). | Migration Center (discovery client / imports → inventory, assessment, TCO reports), Pricing Calculator for ad-hoc SKU estimates | Which workloads migrate first (proof-of-concept candidates)? BYOL licensing? Current OpEx vs cloud OpEx? |
| **Plan** | Design target architecture (landing zone), network topology, cutover sequence, rollback. | Migration Center (asset groups, target recommendations, dependency info), Migrate to Virtual Machines (readiness/fit assessment), dependency mapping tools | Phased vs big-bang? Coexistence period duration? Network interconnect type & bandwidth? |
| **Deploy** | Execute migration (rehost, replatform, refactor), validate, cutover. | Migrate to Virtual Machines (VMs), Database Migration Service, Datastream, Storage Transfer Service, Transfer Appliance, Migrate to Containers (for modernization). | Cutover window length? Test environment? Parallel run (double-running costs)? |
| **Optimize** | Tune workloads (rightsizing, licensing, cleanup). Establish landing zone (organizational & security). | Active Assist (Recommender), Cost Management tools, Cloud Monitoring, org-policy framework. | CUDs vs Spot? Shared VPC model? Identity federation from on-prem AD? |

**Landing zone**: a foundational GCP organization setup (resource hierarchy, IAM, networking, monitoring, billing controls) that supports repeatable, secure, cost-managed deployments. Also called "organizational foundation" or "Google Cloud Foundation."

### 6 Rs: rehost, replatform, refactor, repurchase, retire, retain

Each R represents a migration strategy with different effort, cost, downtime, and modern-architecture tradeoffs:

| R | Definition | Example | Time-to-cloud | Cost-to-move | Ongoing OpEx | Best for | Exam trap |
|---|---|---|---|---|---|---|---|
| **Rehost** (Lift-and-shift) | Move VM as-is, no code/config changes. Runs on Compute Engine. | On-prem .NET Framework app → Compute Engine Windows VM | Fast (days) | Low | High (larger VMs, licensing) | Urgent timelines, unmodified legacy apps, vendor lock-in risks, compliance hold. | "Fastest to cloud" ≠ "cheapest to operate." |
| **Replatform** (Lift-improve-and-shift) | Move to managed services (Cloud SQL for RDBMS, AlloyDB for perf-critical, Cloud Run for HTTP services). Minimal code changes. | MySQL on EC2 → Cloud SQL; containerised .NET (Core) service → Cloud Run (classic .NET Framework on IIS needs Windows VMs/GKE Windows nodes, not Cloud Run) | Medium (weeks) | Low–medium | Medium (less admin, license discounts) | Apps that benefit from managed-service features (HA, scaling, backups) without refactor. | Doesn't always require code refactor. |
| **Refactor** (Re-architect) | Rewrite for cloud-native (microservices, serverless, containers). | Monolith → GKE + Pub/Sub; custom batch → Dataflow | Slow (months) | High | Medium–low (native scaling, no VMs) | Apps with heavy batch, streaming, legacy tech debt, team wants modernization. | "Most modern" ≠ "necessary." ROI over 2+ years. |
| **Repurchase** (Re-license) | Switch to cloud-native SaaS. | Custom CRM/HR system → Salesforce or Workday (SaaS) | Medium | High (SaaS license) | High (SaaS) | Strategic vendor lock-in acceptable; SaaS feature-set > on-prem. | High annual cost commitment. Requires change management. |
| **Retire** (Decommission) | Turn off (data archived/deleted). | Legacy reporting system replaced by BigQuery + Looker | Immediate | Zero | Zero | Redundant apps, obsolete data pipelines, regulatory-hold expires. | Data retention + compliance must confirm it's OK to retire. |
| **Retain** (Revisit later) | Keep on-prem or in colocation long-term (not cloud). | Active Directory, mainframe (too risky/costly to migrate). | N/A | N/A | On-prem (high) | Regulatory blockers, zero-downtime non-negotiable, team unwilling to change. | Contradicts "cloud-first"; usually hybrid reality. |

**Exam logic**: "How much effort? How much downtime? How much cost?" If effort is acceptable and cost is lower → rehost or replatform. If team has bandwidth and app is modernization-critical → refactor. If legacy app must run unchanged → rehost. If only 20 % usage → retire.

### Migration Center: discovery, assessment, TCO, recommendations

Google Cloud's unified migration platform (discovery, assessment, planning; named in the exam guide):

**What it does:**
- **Discovery**: the Migration Center discovery client (or imports from other tools/CSV) scans VMs/servers and collects inventory (OS, CPU, RAM, storage, network, running processes, dependencies).
- **Groups**: logical groupings of workloads (e.g., "ERP tier," "web tier").
- **Assessment / TCO reports**: compares as-is costs with estimated Google Cloud costs under different migration preferences (e.g. Compute Engine multi-tenant vs sole-tenant, VMware Engine); reports export to CSV, Excel or Slides.
- **Recommendations**: suggests target product and machine series (servers) or target database service (databases), and estimated monthly cost.
- **Dependencies**: maps inter-VM communication, external integrations (e.g., on-prem DB, SaaS).
- **Waves**: you use the groups and dependency data to sequence phased migration (pilot → wave 1, 2, 3…); the wave plan itself is your own artefact.

**Why preferred for migration TCO**: Unlike Pricing Calculator (which computes prices for pre-specified SKUs), Migration Center *discovers* workloads, factors licensing (BYOL rules), and produces *migration-specific recommendations*. Answers: "Should I rehost or replatform VM X? What machine type? What's the cost impact over 3 years?"

### Migrate for Compute (VMs): runbook, BYOL, cutover, benchmarking

**Migrate to Virtual Machines** (current name; older names "Migrate for Compute Engine" / "Migrate for GCE" / Velostrata are the same product line; the exam guide and docs use the current name):

- **Workflow**: connect a Migrate Connector (VMware vCenter source) or cloud source (AWS/Azure), onboard the VM, replicate disks (full copy then incremental sync), create a test clone, validate in Google Cloud, cutover.
- **BYOL (Bring Your Own License)**: Compute Engine supports PAYG and BYOL licences; Migrate to Virtual Machines assigns a default licence type per OS and you can override it (PAYG or BYOL) when the OS supports both.
  - **Sole-tenant nodes**: Windows Server BYOL requires sole-tenant nodes (physical-server affinity); SQL Server and other vendor BYOL depend on the vendor's own mobility terms (e.g. Microsoft Software Assurance / License Mobility) ⚠ verify. Migrated VMs can be placed on sole-tenant nodes via node affinity labels (node group must exist first).
  - **Cost impact**: BYOL saves licence cost, but sole-tenant nodes carry a premium on vCPU/memory and you pay for the whole node. Net win only if the licence saving is large; Migration Center TCO reports compare multi-tenant PAYG vs sole-tenant BYOL.
- **Cutover strategies**:
  - **Test clone**: migrate to VM, test in GCP without shutting source. Validate networking, app behavior, database connectivity. Low-risk.
  - **Parallel run** (double-running): keep source live, switch traffic after cutover. Highest cost (double infrastructure), but minimal downtime & instant rollback.
  - **Big-bang cutover**: source stops, migrate, target starts. Fastest prep, longest downtime window.
- **Benchmarking**: post-migration, measure application latency, throughput, resource utilization. Compare to baseline. Verify rightsizing assumptions (did we pick the correct machine type?).

### Database Migration Service (DMS): homogeneous vs heterogeneous, continuous

**Database Migration Service**: managed, serverless migration of relational databases from on-prem or other clouds into Cloud SQL and AlloyDB (one-time or continuous jobs).

- **Homogeneous**: source & target same engine (MySQL 5.7 → Cloud SQL MySQL 8.0; SQL Server 2019 → Cloud SQL SQL Server 2022). Schema compatible, mostly DDL/DML passthrough. **Faster, lower risk.**
- **Heterogeneous**: source engine ≠ target engine (Oracle → Cloud SQL for PostgreSQL or AlloyDB; SQL Server → Cloud SQL for PostgreSQL or AlloyDB). Uses a conversion workspace (Gemini-assisted conversion available) and requires **schema mapping** (Oracle sequences → Postgres SERIAL, etc.), possible application-logic rewrites (stored procedures, data types). **Slower, higher risk, often paired with refactoring.**
- **Continuous replication**: DMS creates initial snapshot, then streams updates (binlog, transaction logs) to target in near-real-time. Enables low-downtime/zero-downtime cutover (source & target briefly synced, then switch).
- **Supported engines**: homogeneous MySQL → Cloud SQL for MySQL, PostgreSQL → Cloud SQL for PostgreSQL / AlloyDB, SQL Server → Cloud SQL for SQL Server; heterogeneous Oracle → Cloud SQL for PostgreSQL / AlloyDB and SQL Server → Cloud SQL for PostgreSQL / AlloyDB. DMS does **not** migrate into Spanner, Firestore or Bigtable (those use other tools, e.g. Spanner migration tool, Dataflow templates).
- **Exam focus**: "How to migrate Oracle on-prem to GCP?" → If modernization desired: Oracle → Cloud SQL for PostgreSQL or AlloyDB (heterogeneous, conversion workspace). If the Oracle licence/features must be kept unchanged: Oracle Database@Google Cloud or Bare Metal Solution (not Cloud SQL, which has no Oracle engine) ⚠ verify product naming. "How to minimize downtime?" → continuous replication.

### Datastream

**Datastream**: CDC (change data capture) service. Captures inserts/updates/deletes from source DB, streams to target (BigQuery, Cloud Storage, or other consumer).

- **Use case**: real-time data warehouse population (ELT instead of batch ETL), trickle-feed migrations (incremental data sync), audit logging.
- **Source support**: Oracle, MySQL, PostgreSQL, SQL Server (plus AlloyDB for PostgreSQL ⚠ verify).
- **Compared to DMS**: DMS is *database-to-database* replication (full schema + data); Datastream is *change stream* (useful for analytics pipelines, not full DB migration). Often used *alongside* DMS (DMS handles schema+data, Datastream streams ongoing CDC for reporting/BI).

### Data transfer: online vs offline, decision tree

Transferring data *to* GCP (on-prem data center or branch office to Cloud Storage).

**Online transfer** (over internet or dedicated link):
- **Storage Transfer Service (STS)**: managed transfer into Cloud Storage from Amazon S3, Azure Blob/other clouds, HTTP/URL lists, other buckets, and on-prem POSIX/NFS file systems (via transfer agents). Handles retries, checksums, scheduling, bandwidth limits.
- **gcloud storage (or legacy gsutil) CLI**: lightweight, scripted, incremental sync; `gcloud storage` is the current tool.
- **When**: the upload completes in an acceptable time (Google's guidance: if it would take more than about a week, consider Transfer Appliance).
- **Cost**: ingress into Google Cloud is free; you pay for your own link (ISP / Interconnect) and for egress charged by the *source* cloud. Check current STS pricing (agent-based and cloud-to-cloud transfers are not always free) ⚠ verify.

**Offline transfer** (physical appliance):
- **Transfer Appliance**: rugged appliance in two sizes, **TA40 (up to 40 TB usable) and TA300 (up to 300 TB usable)**, shipped to your site, loaded with data (encrypted), shipped back to Google, which imports it into Cloud Storage. Several appliances can be used in parallel for larger datasets. (Older 100 TB / 480 TB models are not current.)
- **When**: Google's criteria: existing Google Cloud customer, appliance available in your location, and the upload would take **more than one week** over the network. In practice that means tens of TB or more on typical links, or any size on a slow link; there is no hard "100 TB break-even".
- **Cost**: priced per appliance/rental period plus shipping (see pricing page; no invented figures here). Ingress to Cloud Storage is free, so the comparison is appliance cost vs time and link cost, not vs egress fees. Docs example: 300 TB at 100 Mbps takes about 9 months online vs under 25 days with the appliance plus about 10 business days for data to appear in Cloud Storage.

**Formula (decision)**: 
```
Transfer time (hours) = Data size (GB) x 8000 / Bandwidth (Mbps) / 3600   (assumes 100 % link utilisation; real life is slower)
Example: 5 TB, 10 Mbps link
         = 5000 x 8000 / 10 / 3600 ≈ 1,111 hours ≈ 46 days (offline better)
Example: 50 GB, 100 Mbps link
         = 50 x 8000 / 100 / 3600 ≈ 1.1 hours (online fine)
```

**BigQuery Data Transfer Service**: separate product that automates scheduled, managed loads *into BigQuery* from SaaS apps (e.g. Google Ads, YouTube), Cloud Storage, Amazon S3, Azure Blob and data warehouses such as Teradata and Amazon Redshift. Not a general file/object migration tool and not CDC (that is Datastream).

### Google Cloud VMware Engine: lift-and-shift VMware to cloud

**Google Cloud VMware Engine**: managed VMware on GCP (vSphere, vSAN, NSX). Allows lifting entire VMware private clouds to GCP without rehosting to Compute Engine VMs.

- **Use case**: data-center VMware migration, long-term hybrid (on-prem VMware + GCP VMware + GCP-native services).
- **Connectivity / migration**: VMware HCX (Hybrid Cloud Extension) is installed in the private cloud (HCX Enterprise features included at no extra cost): bulk, vMotion / Replication Assisted vMotion migration and L2 network extension, so IP addresses can be kept and downtime is minimal.
- **Licensing**: VMware Engine runs the VMware Cloud Foundation (VCF) stack (ESXi, vCenter, vSAN, NSX, HCX). Historically licences could be bundled in the service (via CUDs) or BYOL. Following Broadcom's change to a "bring your own subscription" model for hyperscalers, Google stopped selling new VCF-licence-included nodes after 15 Oct 2025: new capacity needs a VCF subscription bought from Broadcom plus the BYOL option of the service. Existing licence-included commitments run to term.
- **Cost**: bare-metal node-based pricing (you pay per ESXi node, with 1/3-year CUDs), not per VM; justified when you want to keep VMware tooling/skills, move fast without re-platforming, or have a large VMware estate. A follow-on step is to modernise VMs to native services later.

### Hybrid & phased migration: coexistence, network planning, dependency mapping

**Hybrid migration** (on-prem + cloud coexist during transition):
- **Coexistence period**: often 3–12 months. Source remains live, apps gradually migrate.
- **Network link**: Cloud Interconnect (Dedicated or Partner, up to 100 Gbps) preferred over HA VPN (3 Gbps per tunnel, runs over the public internet so latency varies) for high-throughput replication or frequent DB sync.
- **Shared DNS/Active Directory**: federate on-prem AD to Cloud Identity (Google Cloud Directory Sync for users/groups + SAML SSO with AD FS/Entra ID, or Workforce Identity Federation for console/API access without syncing). Workload Identity Federation is for workloads/service accounts, not staff SSO. Enables hybrid SSO.

**Dependency mapping**:
- Identify tier dependencies (frontend → app → database), external integrations (EDI partners, SaaS, APIs).
- **Tools**: Application Dependency Mapping (part of Migration Center discovery), network packet analysis.
- **Outcome**: groups for ordered cutover waves. Migrate DB first, then app tier, then frontend.

**Testing & PoC**:
- Pilot wave: one low-risk, representative workload. Proves tooling, networking, team readiness.
- Failover test: simulate link failure, ensure failback works.
- Load test: does app perform on cloud machine type? Latency to cloud DB acceptable?

**Rollback readiness**:
- Snapshots of source (before cutover).
- Reverse replication (cloud → on-prem) must be set up separately with native database tooling; DMS itself replicates source → destination only ⚠ verify.
- RTO/RPO agreements: if cloud instance fails, how fast back to on-prem (minutes → need hot-standby; hours → snapshots OK).

### Cutover strategies & validation

- **Dry-run**: migrate to test VM, validate in staging environment. No production traffic.
- **Canary cutover**: send small % of traffic to cloud first (load balancer traffic splitting). Validate errors, performance.
- **Feature flag cutover**: toggle backend endpoint in code (on-prem ↔ cloud) without infrastructure change. Fast rollback.
- **Acceptance testing**: business users test cloud instance for functional parity, compliance (data privacy handling, audit logs).

### Software licensing & financial impact

**Licensing considerations**:
- **BYOL**: Windows Server, SQL Server, Oracle—licenses are often tied to physical cores/servers or to vendor mobility rules. Transferring to cloud VMs may *not* be permitted (check vendor terms). GCP sole-tenant nodes provide the physical-server affinity that Windows Server BYOL needs.
- **Per-core vs per-seat**: SQL Server pricing depends on core count. Larger cloud machine → higher license cost. Rightsizing (and custom core counts / constrained vCPU) is crucial. Cloud SQL / managed services are licence-included (no BYOL).
- **Sole-tenant nodes**: dedicated hardware (no multi-tenant sharing). Enables BYOL but carries a premium on vCPU/memory (see current pricing) and you pay for the whole node. Break-even if licence saving > the premium plus any unused node capacity.
- **Double-running costs**: parallel run (source + cloud) for cutover window = 2× infrastructure cost for days/weeks.

**Financial impact**:
- **CapEx → OpEx**: on-prem capital investment (servers, networking, real-estate) → cloud monthly bills. Impacts accounting (OpEx expensed, CapEx capitalized → depreciation). TCO changes over 3–5 years (cloud: higher early OpEx, lower later; on-prem: high CapEx, stable OpEx).
- **Licensing savings**: BYOL → cloud may reduce vendor costs if vendor offers cloud discounts. Or increases if licensing per-core & cloud machine is larger.
- **Admin headcount**: reduced ops staff (cloud-managed services handle patching, backups). Salary savings—large for enterprises migrating 100+ VMs.

**Case-study angles**:
- **Cymbal Retail**: "reduce call-centre and data-centre costs" → migrate redundant on-prem infra, consolidate to cloud. Eliminate duplicate MySQL/SQL Server licenses.
- **EHR Healthcare**: "reduce infra admin cost" + "≥ 99.9 % availability" → cloud-managed services (Cloud SQL HA, GKE Autopilot) eliminate on-prem admin, improve reliability.

### Modernization during migration

**Migrate to Containers** (formerly Migrate for Anthos / Migrate for Containers): modernize workloads running in VMs (e.g. Linux Java/Python apps, some Windows IIS apps) → containers on GKE (or Cloud Run).
- Analyzes the source VM, generates container artifacts (Dockerfile, deployment YAML), builds the image, deploys to GKE.
- Less risky than full refactor; preserves app logic; gains container scaling benefits.

**Database modernization paths**:
- On-prem MySQL 5.7 → Cloud SQL for MySQL (homogeneous, fast). AlloyDB is PostgreSQL-compatible only, so MySQL → AlloyDB is a heterogeneous refactor (schema/app changes), not a drop-in.
- On-prem SQL Server → Cloud SQL SQL Server (similar version, minimal changes) or Cloud SQL PostgreSQL (heterogeneous refactor, schema mapping).
- On-prem MongoDB → Firestore (serverless, real-time sync) or Spanner (distributed, ACID for EHR multi-region compliance).
- **Cymbal case study**: MySQL, SQL Server, Redis, MongoDB on-prem. Modernization path: MySQL → Cloud SQL for MySQL (homogeneous; AlloyDB only if willing to convert to PostgreSQL), SQL Server → Cloud SQL for SQL Server (or AlloyDB after heterogeneous conversion), MongoDB → Firestore (real-time product discovery), Redis → Memorystore (same semantics, cloud-managed).

### Future improvements (1.5): cloud-first design, evolving needs, enablement

- **Cloud-first design**: new applications built on cloud-native (Pub/Sub, Dataflow, BigQuery, Firestore) from the start, avoiding migration debt.
- **Evolving business needs**: migration not one-time; continuous repatriation (cloud → on-prem if business requirements change) possible. Hybrid is the new normal; architecture must support workload mobility.
- **Enablement & advocacy**: upskilling teams (GCP certifications, architecture workshops), establishing centers of excellence (CoE), sponsoring cloud-first culture shift.

## Decision guides

**When to choose each R**:
```
START: "Migrate workload X to GCP?"
├─ "Timeline urgent (< 1 month)?"
│  ├─ YES → REHOST (Compute Engine VM). Quick, minimal code changes.
│  └─ NO
│     └─ "Is workload modernization strategic?"
│        ├─ YES (long-term roadmap) → REFACTOR (GKE, Dataflow, serverless). Invest months.
│        └─ NO (just move) → REPLATFORM. Evaluate: SQL Server → Cloud SQL? App → Cloud Run? Weeks to months.
├─ "Is workload redundant?"
│  ├─ YES → RETIRE. Zero cost, compliance OK?
│  └─ NO
├─ "Is vendor lock-in OK?" (e.g., replace with SaaS)
│  ├─ YES → REPURCHASE (Salesforce, SAP on GCP, etc.). High license cost but fewer ops headaches.
│  └─ NO → above paths.
├─ "Regulatory blockers (data sovereignty, ≥ 99.99 % availability)?"
│  └─ YES → RETAIN (hybrid, coexist on-prem + cloud). Will test: can link support seamless failover?

OUTCOME: Recommended path + effort estimate + cost delta + risk (downtime, data loss, complexity).
```

**Data transfer decision**:
```
START: "Move data to GCS?"
├─ Compute transfer time = GB x 8000 / Mbps / 3600 (use realistic link utilisation)
├─ Fits in about 1 week (or a dedicated Interconnect is already in place)
│  └─ → STS / gcloud storage (online, scheduled, checksummed)
├─ Would take more than ~1 week
│  └─ → Transfer Appliance (offline; TA40 = 40 TB, TA300 = 300 TB usable; use several for more)
└─ Source is another cloud (S3 / Azure) or recurring sync
   └─ → STS (cloud-to-cloud); ingress to Google Cloud is free, the source cloud charges egress.
```

## Well-Architected Framework lens

| Pillar | What to do in migration | How to verify |
|--------|---|---|
| **Operational excellence** | Clear runbook (cutover steps, rollback procedures). Dependency mapping + testing. Landing zone (org, monitoring, alerts ready). Post-migration: RCA on issues. | Cutover < expected time? Alerts fire correctly? Team confidence in rollback? |
| **Security & compliance** | Assess on-prem controls; ensure cloud landing zone has equivalent (IAM, encryption, audit logs). Data residency requirements enforced (org-policy constraints). Network segmentation (VPC peering, firewalls). Credential migration (AD → Cloud Identity). | Security review of cloud config pre-cutover? Compliance audit post-migration? |
| **Reliability** | HA for critical systems (Cloud SQL HA, multi-region if needed). Test failover. DR plan (backup frequency, RTO/RPO). Chaos test: simulate region failure. | Does cloud app tolerate VM restart? DB failover? Link failure? |
| **Performance & cost** | Rightsize instances (benchmarking). Choose appropriate service (rehost → GCE, replatform → Run/SQL, refactor → GKE). Monitor cloud vs on-prem performance. Cost monitoring from day 1 (label, budget alerts). | App latency improved or same post-migration? Monthly cloud cost < projected? |
| **Sustainability** | Google data-center fleet efficiency (PUE about 1.1) is typically better than the industry average on-prem (roughly 1.5+) ⚠ verify; also lower-carbon regions (Carbon Footprint report) and higher utilisation. Migration → net reduction in operational energy. | Calculate on-prem power consumption vs cloud equivalent. Highlight sustainability win to stakeholders. |

## AI/GenAI angle

- **Anomaly detection in migration**: use BigQuery ML or Gemini Enterprise Agent Platform (formerly Vertex AI) time-series forecasting to predict cutover issues (based on pre-migration performance metrics, flagging outlier workloads).
- **Generative summaries**: Gemini Cloud Assist (and Gemini-assisted code conversion in Database Migration Service / Migration Center) can summarize migration readiness assessments, generate data-transfer runbooks, explain licensing implications to non-technical stakeholders.
- **Auto-remediation**: Agent Platform Pipelines orchestrate post-migration validation (connectivity tests, latency checks, cost optimization recommendations).
- **Skill elevation**: train teams using NotebookLM (knowledge articles on GCP services, BYOL rules, cutover procedures).

## Case-study hooks

### Altostrat Media
- **Hook**: on-prem ingestion/archival ("legacy on-prem to be migrated"); hybrid connectivity for high-throughput media ingestion.
- **Migration path**: Assess on-prem ingestion pipeline (batch ETL, storage) → Replatform to Dataflow + Cloud Storage (managed pipelines, cost savings, scalability). Test with pilot (one media stream). Use Cloud Interconnect (Dedicated/Partner) for the fast, reliable ingestion link; for the historical archive use Transfer Appliance or STS depending on the weeks-of-upload test. HCX not needed unless the source is VMware (then consider VMware Engine).

### Cymbal Retail
- **Hook**: "reduce call-centre and data-centre costs" + "mix on-prem + cloud" MySQL, SQL Server, Redis, MongoDB.
- **Migration path**: Assessment via Migration Center (on-prem inventory → cloud cost estimate). Phased: (1) databases first (MySQL → Cloud SQL for MySQL, SQL Server → Cloud SQL for SQL Server, MongoDB → Firestore, Redis → Memorystore). (2) Apps rehost on GCE or replatform to Cloud Run. (3) Retire on-prem data center over 6 months. Cutover: parallel run (weeks), then switch. Monitor costs; verify licensing (per-core SQL Server).

### EHR Healthcare
- **Hook**: "≥ 99.9 % availability" + "HIPAA compliance" + "reduce infra admin cost" + hybrid (insurers still on-prem, not migrating).
- **Migration path**: Hybrid. Containerized apps (Kubernetes on-prem → GKE in cloud) → migrate in waves (test env first, then prod). Database: MySQL, MS SQL Server, Redis → Cloud SQL, Memorystore. Use Interconnect (Dedicated, 10/100 Gbps) for low-latency hybrid queries. Use Cloud SQL HA (regional, synchronous standby; 99.95 % SLA Enterprise edition, 99.99 % Enterprise Plus) and add a cross-region read replica for DR (replicas are DR, not part of the HA SLA). Network planning: private connectivity (no internet exposure for patient data). AD federate to Cloud Identity (SSO for staff). Ensure audit logs (Cloud Logging) meet HIPAA 6-year retention. Compliance: BAA signed, Assured Workloads / org-policy location constraints for data residency and compliance controls ⚠ verify exact HIPAA control package.

### KnightMotives Automotive
- **Hook**: "on-prem + some clouds" fragmented, supply-chain mainframe to modernize, dealer connectivity unreliable, GDPR compliance.
- **Migration path**: Hybrid cloud strategy. Phase 1: modernize on-prem + GCP connectivity (upgrade plant networks → direct link to HQ/cloud). Phase 2: rehost on-prem VMs → Compute Engine (parallel-run pilot, test failover). Phase 3: databases (legacy ERP → SAP on Google Cloud or managed PostgreSQL for new systems). Modernization: dealer tools (sales CRM, inventory) → Cloud Run (serverless, dealer devices low-bandwidth friendly, Gemini-powered chat for support). Data platform: anonymize telemetry (GDPR-compliant, secure model deployment with VPC-SC). Continuous: lift autonomous-driving simulation to cloud (AI Hypercomputer for training).

## Exam traps & keyword → answer

- **"Fast to cloud" ≠ "low cost to operate."** Rehost is fastest but OpEx stays high (large VMs, licensing per-core). Look for *long-term TCO* question.
- **"Zero downtime" → continuous replication + traffic switch.** Parallel run not required if DMS replication is fast enough.
- **Windows Server BYOL → sole-tenant nodes.** Not shared (multi-tenant) VMs; Migration Center only allows BYOL with sole-tenant for Windows. Managed services (Cloud SQL) are licence-included, no BYOL.
- **Transfer Appliance when the online upload would take more than about a week** (TA40 = 40 TB, TA300 = 300 TB). Small data online is faster. Always calculate transfer time; ingress is free, so egress fees are not the argument.
- **Migration Center ≠ Pricing Calculator.** Migration Center assesses workloads, recommends target, estimates cost for *that* target. Pricing Calculator generic SKU pricing.
- **Network planning critical.** If case-study says "high-freq DB queries on-prem ↔ cloud," Cloud Interconnect (Dedicated 10/100 Gbps, or Partner) better than VPN: private path, predictable lower latency and higher throughput than VPN over the public internet.
- **Dependency mapping first.** Can't cutover randomly. Frontend depends on app tier, which depends on DB. Migrate DB → app → frontend order.
- **VMware Engine ≠ Compute Engine VMs.** VMware Engine lifts entire vSphere clusters (HCX migration, NSX, vSAN; VCF subscription now BYO from Broadcom for new capacity). Compute Engine is IaaS VMs. If question says "lift VMware vCloud to GCP" → VMware Engine, not individual Compute Engine rehosting.
- **Datastream ≠ DMS.** Datastream: change-data-capture → analytics (BigQuery). DMS: database-to-database replication. Both can coexist (DMS for cutover, Datastream for ongoing sync).
- **Cutover window ≠ migration duration.** Full data copy may take weeks (offline Transfer Appliance). Cutover (flip traffic) is hours/days. Separate metrics.

## Quick-reference cheat table

| **Tool / Strategy** | **Best for** | **Effort** | **Downtime** | **Cost to execute** | **OpEx impact** | **Exam keyword** |
|---|---|---|---|---|---|---|
| **Rehost (Compute Engine)** | Urgent, unmodified legacy | Low | Low–medium | Low | High | "Minimal changes," "fastest" |
| **Replatform (Cloud Run, Cloud SQL)** | Web apps, RDBMS, moderate modernization | Medium | Low | Low–medium | Medium | "Managed services," "app refactor minimal" |
| **Refactor (GKE, Dataflow, serverless)** | Monoliths, batch/streaming, team bandwidth | High | Low | High | Medium–low | "Cloud-native," "microservices," "long-term" |
| **Repurchase (SaaS)** | Strategic vendor switch | Medium–high | Medium | High (SaaS license) | High | "Replace with SaaS," "vendor standard" |
| **Retire** | Redundant, EOL | Low | Immediate | Zero | Zero | "Decommission," "no longer used" |
| **Retain (hybrid)** | Regulatory blockers, zero-downtime non-negotiable | Low (as-is) | None (stays) | Zero | High (on-prem) | "Can't move," "mainframe," "data sovereignty" |
| **Migration Center** | Assess & plan all strategies | Medium | N/A | Low (free/cheap discovery) | N/A | "Assessment," "recommendation," "TCO," "cost estimate" |
| **Migrate to Virtual Machines** (ex Migrate for Compute Engine) | VM migration, BYOL, test-clone | Low–medium | Low–medium | Low | High | "Lift-and-shift," "cutover," "BYOL" |
| **Database Migration Service** | DB replication (homo/hetero), zero-downtime | Medium | Low–zero | Low–medium | N/A | "Continuous replication," "schema mapping," "cutover" |
| **Datastream** | CDC, real-time ELT to BigQuery | Low–medium | N/A (change-capture) | Low | N/A | "Real-time sync," "BigQuery load," "CDC" |
| **Storage Transfer Service** | Online data copy (upload fits in about a week) | Low | N/A | Low | N/A | "Online transfer," "bandwidth available," "S3/Azure to Cloud Storage" |
| **Transfer Appliance** | Offline data move (TA40 / TA300; upload would take > 1 week) | Medium (shipping delay) | N/A | Per-appliance rental + shipping | N/A | "Tens/hundreds of TB," "slow link," "offline" |
| **Cloud Interconnect** | Hybrid networking, high-throughput | Medium–high | N/A | Medium (recurring) | N/A | "10–100 Gbps," "low latency," "hybrid," "on-prem link" |
| **VMware Engine** | Lift entire VMware estate (vSphere, NSX, HCX) | Medium | Low (HCX) | High (premium) | Very high | "VMware," "vCloud," "HCX," "entire cluster" |
| **Migrate to Containers** | VM workload → GKE/Cloud Run, less risky than full refactor | Medium | Low | Low–medium | Medium | "Container," "Docker," "GKE" |

## Practice scenarios

**Scenario 1**: Your company migrates a 50-VM on-prem SQL Server farm to GCP. Timeline: 6 months. Business says "≥ 99.95 % availability post-migration, and we want costs down 30 %." 3 VMs are mission-critical (ERP). What's the recommended strategy?

- A) Replatform all SQL Server instances to Cloud SQL for PostgreSQL (heterogeneous conversion); rehost app VMs on Compute Engine; retire on-prem data center immediately; expect 50 % cost savings.
- B) Retire 30 VMs deemed redundant; rehost 15 on Compute Engine; refactor ERP to microservices on GKE (risky, timeline too tight).
- C) Retain everything on-prem; migrate non-critical data to cloud analytics (BigQuery) only; minimize disruption.
- D) Rehost the 50 VMs on Compute Engine with Migrate to Virtual Machines (sole-tenant nodes only where BYOL licences are needed); move the 3 mission-critical ERP databases to Cloud SQL for SQL Server HA via Database Migration Service; use Cloud Interconnect for the hybrid link during coexistence (6 months); test failover weekly.

**Answer: D.** Rehost ensures fast timeline (urgent apps). BYOL licensing on sole-tenant nodes only where the licence saving justifies the premium. Cloud SQL for SQL Server HA gives a 99.95 % SLA (Enterprise edition; meets target) and DMS keeps cutover downtime low. Interconnect handles frequent ERP queries (on-prem app → cloud DB). Coexistence & testing reduce risk. *Why not A:* SQL Server schema/stored procedures → Cloud SQL PostgreSQL is risky refactor (timeline too tight). *Why not B:* Refactor during tight timeline = risk. *Why not C:* No progress; on-prem costs stay high.

---

**Scenario 2**: You're assessing a 1 PB data lake (Hadoop HDFS on-prem) for cloud migration. Network bandwidth from data center to internet: 1 Mbps. Which data-transfer approach minimizes cost and time?

- A) Use Storage Transfer Service with parallelization; budget 3 months transfer time.
- B) Order Transfer Appliances (TA300, 300 TB usable each; about 4 units, loaded and shipped in rotation).
- C) Establish a Cloud Interconnect 100 Gbps; reduce the transfer to days; accept recurring port cost and provisioning lead time.
- D) Retain 80 % on-prem (Hadoop); migrate 20 % to BigQuery incrementally via STS.

**Answer: B.** 1 PB = 1,000,000 GB. At 1 Mbps: 1,000,000 x 8000 / 1 / 3600 ≈ 2.2 million hours, i.e. roughly 250 years (1 Mbps is only ~10 GB/day), so any online option over that link is unfeasible. Transfer Appliance TA300 holds up to 300 TB usable, so 1 PB needs about 4 appliances, rotated/parallel; the offline route takes weeks rather than centuries. *Why not A:* STS over 1 Mbps would take centuries, not 3 months. *Why not C:* a 100 Gbps Interconnect is technically fast but needs a long provisioning lead time, a colocation/port presence and recurring cost for a one-off move, and the on-prem HDFS source must also sustain that throughput; Transfer Appliance is the standard answer for a bulk one-off with a tiny link (note ingress to Google Cloud is free, so egress fees are not a factor). *Why not D:* Leaves 80 % on-prem; doesn't address the "full migration" goal.

---

**Scenario 3**: EHR Healthcare is migrating MySQL + Redis on-prem to cloud. Data: 500 GB MySQL, 50 GB Redis. Timeline: 4 months. Network: 10 Mbps ISP link (during business hours only; nights available for heavy sync). Cutover planned for weekend 6-hour window. What's the migration data strategy?

- A) STS both MySQL + Redis online to Cloud Storage; use Datastream for continuous MySQL sync (post-cutover); manual Redis refresh at cutover.
- B) Take a full mysqldump and a Redis snapshot at cutover, copy both to Cloud Storage over the 10 Mbps link during the 6-hour window, then restore them.
- C) Use Database Migration Service for MySQL (continuous replication); manually dump Redis to .rdb file, STS to GCS, restore to Memorystore at cutover.
- D) Migrate only MySQL with Database Migration Service and leave Redis on-prem permanently, reached from Google Cloud over Cloud VPN.

**Answer: C.** DMS handles continuous MySQL replication (homogeneous, low-risk, near-zero downtime). Redis is key-value (not a "database" in DMS sense); dump .rdb file (point-in-time snapshot) to Cloud Storage via STS or gcloud storage (50 GB at 10 Mbps ≈ 50 x 8000 / 10 / 3600 ≈ 11 hours of link time, so schedule it on a night before cutover; the 500 GB MySQL initial load is ≈ 111 hours of link time, spread over nights during the 4 months), then import into Memorystore at cutover (minutes to tens of minutes). 6-hour window: MySQL final sync (seconds to minutes if replication has caught up), Redis import, validation (~1 hour). Fits comfortably, provided Redis data is a cache or can tolerate the snapshot gap. *Why not A:* STS is for object storage (GCS), not direct DB replication. Datastream is change-capture (useful for BigQuery, not for cutover sync). *Why not B:* at 10 Mbps, about 550 GB takes roughly 122 hours of link time (550 x 8000 / 10 / 3600), far more than the 6-hour window; the bulk copy has to happen beforehand, with replication keeping it current. *Why not D:* leaves Redis on-prem, so the migration is incomplete and every cache call crosses the VPN, adding latency.

---

**Scenario 4**: You're using Migration Center to assess 200 on-prem VMs for cloud readiness. Assessment shows: 150 rehost-ready, 30 refactor candidates (legacy Python, high batch load), 20 "complex" (custom hardware dependencies, proprietary protocols). Business says "Get 80 % migrated in 12 months." Which approach achieves this?

- A) Migrate all 150 rehost-ready in wave 1 (months 1–6), refactor 30 in wave 2 (months 7–12), retain 20 on-prem indefinitely.
- B) Migrate 50 rehost-ready as PoC (months 1–3), then full 150 (months 4–9), defer 50 "complex" VMs, retry assessment in 6 months.
- C) Replatform all 200 to Cloud Run, BigQuery, etc. (one-size-fits-all modernization).
- D) Retire 50 % deemed "redundant" (no evidence given), migrate remaining 100 rehost-ready, retain 50 complex.

**Answer: A.** 150 + 30 = 180 VMs = 90 % cloud-bound (exceeds 80 % goal). Phased: rehost (fast, low-risk) first, then refactor (invest team effort later). 20 complex → retain (regulatory, custom hardware, can't migrate without massive refactor). Timeline realistic: rehost 150 in 6 months (~25 VMs/month, one pilot wave + 3–4 prod waves). Refactor 30 in next 6 months (higher effort, ~5–6 per month). *Why not B:* it only migrates the 150 rehost-ready VMs (75 % of 200), which misses the 80 % target, and it defers the 30 refactor candidates that A can still deliver. A pilot wave inside wave 1 is still good practice. *Why not C:* Cloud Run is wrong for most rehost-ready (VMs aren't HTTP services). *Why not D:* "Retire 50 %" without evidence is reckless (data loss risk).

---

**Scenario 5**: A financial-services company plans BYOL Windows Server + SQL Server (hundreds of cores, expensive licenses) to GCP. CFO asks: "What's the total-cost impact over 3 years?" Which factors matter most?

- A) Just compare on-prem annual license cost vs GCP instance costs; if GCP higher per core, we've lost.
- B) Only consider GCP compute cost; licensing is vendor problem.
- C) Sole-tenant node cost (premium over shared VMs, plus paying for whole nodes) + reduced license cost (if BYOL allows cloud discount) + eliminated data-center real-estate cost + staff overhead reduction (cloud-managed updates, backups). Then compare to on-prem (license + salary + power + real-estate).
- D) Assume BYOL doesn't work on GCP; plan to repurchase licenses (cloud-native alternatives).

**Answer: C.** TCO is multidimensional: license cost, infrastructure cost, staffing (ops salary), facilities (real-estate, power). BYOL on sole-tenant nodes *may* reduce per-core license cost (vendor discount for cloud) or keep same (vendor doesn't discount). Sole-tenant premium (~$X/month/instance) likely offset by: (1) eliminated data-center real-estate ($Y/mo), (2) reduced FTE ops staff (cloud-managed, fewer on-call incidents), (3) potential energy/cooling cost reduction (Google fleet PUE ≈ 1.1 vs typical on-prem 1.5+ ⚠ verify). 3-year TCO requires summing all. *Why not A:* Ignores staffing, real-estate, managed-service benefits. *Why not B:* Licensing is often majority cost for financial services (strict compliance requirements, per-core pricing high). *Why not D:* BYOL works on GCP; check vendor terms, not all prohibit it.

---

**Scenario 6**: Altostrat Media is migrating on-prem media ingestion pipeline (batch ETL, daily, 500 GB data per day) to cloud. Current: Bash scripts, NFS storage, on-prem VMs. Target architecture must: (1) auto-scale; (2) cost-optimized; (3) low ops overhead; (4) support future real-time streaming. What's the recommended migration path?

- A) Rehost: keep Bash scripts, run on Compute Engine VMs (GCE autoscaling group), NFS → Cloud Filestore (same paradigm, minimal code change).
- B) Refactor: Bash → Dataflow pipeline (managed, auto-scaling, fault-tolerant, BigQuery sink for downstream analytics). Cloud Storage for raw media. Future: swap batch → streaming (Dataflow Streaming API).
- C) Replatform: Bash → Python, containerize, deploy to GKE (manual scheduling). Store in Cloud Storage (cheaper, durable, no NFS).
- D) Repurchase: switch to Palantir / Apache Airflow Cloud offering; they handle scaling.

**Answer: B.** Batch ETL is Dataflow's bread-and-butter. Dataflow (managed beam) auto-scales (parallelizes 500 GB ingestion), requires zero VM management, integrates Cloud Storage + BigQuery seamlessly. Cost-optimized (pay per second, autoshuts down). Bash script complexity → Python Beam (mature, large community). *Future-ready*: Dataflow supports batch & streaming (same code, just toggle API), enabling real-time when Altostrat needs it. *Why not A:* Rehost adds VM management burden (GCE autoscaling needs tuning). "Low ops overhead" lost. *Why not C:* GKE adds cluster ops (node pools, upgrades). More complex than Dataflow for batch-only pipeline. *Why not D:* Repurchase (third-party SaaS) introduces vendor lock-in, higher cost for simple batch job. Overkill.

## Sources

- **cloud.google.com/docs/migrations**: Migration guides, reference architectures.
- **docs.cloud.google.com/transfer-appliance/docs/4.0/overview** and **/specifications**: Transfer Appliance TA40/TA300, "more than one week" criterion.
- **cloud.google.com/database-migration/docs/supported-databases**: Database Migration Service supported sources/destinations.
- **docs.cloud.google.com/migrate/virtual-machines**: Migrate to Virtual Machines (licensing, sole-tenancy).
- **docs.cloud.google.com/migration-center/docs/discovery-and-assessment-overview** and **/generate-tco-report**: Migration Center assessment and TCO reports.
- **cloud.google.com/blog/products/compute/broadcom-vcf-licensing-changes-for-vmware-engine**: VMware Engine VCF licensing change (Oct 2025).
- **cloud.google.com/architecture/hybrid-cloud-patterns-for-gke**: Hybrid + multicloud best practices.
- **cloud.google.com/architecture/cloud-vmware-engine-architecture**: VMware Engine, HCX.
- **cloud.google.com/products/migrate**: Migrate to Virtual Machines, Migrate to Containers.
- **Official GCP PCA exam guide v6.1** (cloud.google.com/learn/certification/cloud-architect).
