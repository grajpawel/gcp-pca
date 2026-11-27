# Chapter 01 — Exam Overview, Strategy and the Well-Architected Framework

## Exam objectives covered

From exam guide v6.1, section 1.2:
- **Technical requirements:** Google Cloud Well-Architected Framework; HA & fail-over; flexibility of cloud resources; scalability; performance & latency; Gemini Cloud Assist; backup & recovery.

From section 1.1:
- **Business requirements:** functional/non-functional requirements; cost optimization; integration patterns; design decision trade-offs; success measurements (KPI, ROI, metrics); observability.

From section 1.3:
- **Network, storage, compute:** cloud-native networking and storage types; compute choices; mapping workloads to GKE / Cloud Run / Cloud Run functions.

From section S6 (Operational Excellence):
- **Operational excellence pillar** of the WAF; principles & recommendations.

## What the exam really tests

The PCA exam is not a definitions test; it is a **decision test under constraints**. You will see realistic scenarios with explicit constraints ("minimal operational overhead", "must be HIPAA-compliant", "lowest cost", "global consistency") and must pick the architecture that satisfies them.

- **Constraint keywords are signals.** Minimal overhead → managed service. Global consistency → Spanner or Multi-Region Cloud Storage. Lowest cost → Spot VMs or lifecycle policies. Recognise these signals in every question.
- **Questions are about trade-offs.** Each service excels under certain conditions; the exam tests whether you know when to use what. A faster answer is often wrong if it doesn't satisfy the constraint.
- **WAF is everywhere.** Every pillar (reliability, security, cost, performance, sustainability, operational excellence) appears across all 6 sections. Questions do not always explicitly mention WAF, but the answer is usually grounded in one.
- **Case studies are *split-screen* reference material.** You see the business problem on one side and must map it to GCP services on the other. 20–30 % of questions are based on the two case studies; reading the case-study requirements before the exam begins is crucial.
- **Exam strategy matters.** With ~50–60 questions in 2 hours, you have ~2 minutes per question. Mark for review, answer easy ones first, return to hard ones with full context.
- **AI/GenAI is integrated, not separate.** The new 2026 exam expects you to know when to use Gemini models, Agent Platform, Model Garden, and existing AI APIs. Knowing AI is part of knowing **every section**, not a standalone domain.
- **Rebranding is live.** The exam guide uses *new* names (Gemini Enterprise Agent Platform, Agent Platform, Agent Studio, Agent Builder, Model Garden, Agent Runtime). The API and SDK still use old names, but the exam questions use the new terminology.
- **Study time allocation follows weights.** Out of ~90 study hours: S1 (design & plan) ~25 % → ~22 h; S2 (manage & provision) ~17.5 % → ~16 h; S3 (security & compliance) ~17.5 % → ~16 h; S4 (analyze & optimize processes) ~15 % → ~13 h; S5 (manage implementation) ~12.5 % → ~11 h; S6 (solution & operations excellence) ~12.5 % → ~11 h. An 8-week plan frontloads S1, then spirals through all sections.

## Core concepts

### The PCA exam format & logistics

**Standard Exam**
- Duration: 2 hours
- Format: 50–60 multiple-choice and multiple-select questions
- Cost: $200 USD
- Valid for: 2 years
- Languages: English + Japanese
- Recommended background: 3+ years industry experience, 1+ year GCP hands-on
- Passing score: not published; you receive only a pass/fail result (the often-quoted "~70 %" is an unofficial community guess)

**2 Case Studies (20–30 % of questions)**
- 2 randomly selected from a pool of 4 (Altostrat Media, Cymbal Retail, EHR Healthcare, KnightMotives Automotive).
- Shown in a split-screen view (requirements on left, question on right).
- Require mapping business problems → technical solutions.
- Tested extensively in scenario questions; read the case-study digests before the exam.

**Renewal Exam**
- Duration: 1 hour
- 25 questions
- 90–100 % from gen-AI case study (Cymbal Retail or Altostrat Media)
- Cost: $100 USD

**Multiple-Select Gotchas**
- A multi-select question requires **all correct answers** to be checked; partial credit is zero.
- Watch for "which of the following" (implies multiple) vs. "which one of the following" (single).
- In review mode, if uncertain, eliminate obviously wrong options first, then re-read the constraint.

### How to read PCA questions: constraint keywords & elimination heuristics

**Step 1: Identify the constraint(s).**
Most PCA questions hide a single dominant constraint disguised as a scenario. Read the scenario twice; the second time, highlight constraint keywords:

| Keyword | Implies |
|---------|---------|
| "Minimal operational overhead" | Managed service, not self-managed |
| "Globally consistent" | Spanner, Multi-Region Cloud Storage, or datastore with global replication |
| "Lowest cost" | Spot VMs, Standard storage, or lifecycle policies |
| "Immediately available" or "zero RTO" | Hot standby, multi-region failover, or cross-region replication |
| "No code changes" | Fully compatible drop-in replacement (e.g. Cloud SQL for MySQL) |
| "Sub-millisecond" or "ultra-low latency" | Cache (Memorystore) or edge (Cloud CDN) |
| "Audit trail" or "regulatory" | Cloud Audit Logs, Bucket Lock (retention), or VPC Service Controls |
| "HIPAA-compliant" | Assured Workloads, CMEK with Cloud KMS HSM, encryption at rest & in transit, Access Approval |
| "Hybrid" or "on-prem" | Cloud VPN, Interconnect, or GKE Enterprise / Google Distributed Cloud (formerly Anthos) for on-prem Kubernetes |
| "No data egress" | Private Service Connect, Private Google Access, or VPC Service Controls |
| "Scale elastically" | Cloud Run, GKE HPA, Pub/Sub + Dataflow, or MIGs with autoscaling |
| "Distributed training" | AI Hypercomputer, GKE with GPUs/TPUs, or Agent Platform Custom Training |

**Step 2: Eliminate obvious non-starters.**
- If the scenario says "no code changes required" and an option requires custom code, strike it out.
- If the scenario says "minimal operational overhead" and an option requires manual provisioning, strike it out.
- If the scenario mentions GDPR and an option does not support data-residency controls, strike it out.

**Step 3: Narrow to "good enough" candidates.**
Reread the constraint. Is the remaining option addressing it **directly**, or is it a side benefit? Direct answers win.

**Example (real-world inspired):** 
> *Altostrat Media must handle 10x traffic surge during breaking news. Current Cloud Run instances OOM. What is the minimum-operational-overhead solution?*
>
> Options (simplified):
> - A. Migrate the service to GKE Standard and tune HPA ✗ (more operational overhead, does not address the OOM cause)
> - B. Deploy Cloud Run with horizontal pod autoscaling ✗ (HPA is a Kubernetes feature, not Cloud Run's autoscaler)
> - C. Keep the current memory limit and only raise max instances ✗ (every instance still OOMs; the surge is not the root cause)
> - D. Raise the Cloud Run memory limit (e.g. 4Gi), lower per-instance concurrency, and raise max instances ✓ (fixes the OOM and lets the built-in autoscaler absorb the surge; configuration change only)

D is the **direct** answer; it solves the stated constraint (OOM + surge) with the least operational overhead because Cloud Run already autoscales on concurrency and CPU.

### Exam pacing: the 2-minute rule

- **Straightforward questions (20–30 s):** "Which service is used for…?" or "What is the default value…?"
- **Scenario questions (1.5–2.5 min):** Read twice, identify constraint, eliminate, reason through.
- **Hard case-study questions (3–4 min):** These require you to re-read the case digest and map multiple requirements.
- **Triage strategy:** Answer easy ones first, mark hard ones for review, return with fresh eyes. Do not get stuck on one question.

### Case-study split-screen tactics

The exam shows case-study metadata on the left (requirements, systems, constraints) and a single question on the right. 

- **Before the exam starts, memorize each case study's:** business priority, existing systems, hard constraints (e.g., HIPAA, GDPR, on-prem integration), and typical pain points.
- **During the exam, use the left pane as a checklist:** Does this option align with their "minimize operational overhead" goal? Does it integrate with their on-prem legacy system?
- **Multi-part questions:** A question may ask "which service" but the answer requires knowledge of the case's broader context (e.g., Altostrat's requirement for auditable and explainable AI favours options with model versioning, logging and explainability tooling on Agent Platform over an opaque, unmanaged setup).
- **Watch for distractor options that solve a *different* case-study problem.** Only two of the four case studies appear on your exam, but options may still reference requirements that belong to a different case or to none.

## Well-Architected Framework: 6 pillars + AI/ML perspective

Google Cloud's Well-Architected Framework defines 6 pillars. Every architectural decision should be evaluated against all 6, not just one. The exam integrates the WAF into every section.

### The 6 WAF pillars

**1. Operational Excellence**
- *Principle:* Run and monitor systems to deliver business value, and continually improve processes and procedures.
- *Key practices:* Observability (logging, monitoring, tracing, profiling); incident response; chaos engineering; postmortems; automation of runbooks; service catalogs; documentation.
- *GCP angle:* Cloud Observability suite (Monitoring, Logging, Trace, Profiler, Error Reporting); Terraform / Infrastructure Manager for IaC (Deployment Manager is retired); Gemini Cloud Assist for ops troubleshooting.

**2. Security (Privacy & Compliance)**
- *Principle:* Protect information, systems, and assets while delivering business value through risk assessments and mitigation strategies.
- *Key practices:* Principle of least privilege (IAM); encryption at rest and in transit (Cloud KMS, TLS); audit logging (Cloud Audit Logs); data classification and DLP (Sensitive Data Protection); incident response; threat modeling.
- *GCP angle:* IAM roles & deny policies; Cloud KMS (software, HSM, EKM); VPC Service Controls; Confidential Computing; Workload Identity Federation; Organization Policy.

**3. Reliability (Resilience)**
- *Principle:* Ensure a workload performs its intended function correctly and consistently when it's expected to.
- *Key practices:* SLI/SLO/SLA definitions; HA design (multi-region, multi-zone, active-active); backup & DR (RPO/RTO); graceful degradation; health checks; circuit breakers.
- *GCP angle:* Cloud SQL HA (multi-zone) plus cross-region replicas, Spanner multi-region, Multi-Region Cloud Storage, Cloud Load Balancing with health checks, GKE regional clusters, Cloud Run (multi-zone by default; multi-region via a global load balancer).

**4. Performance Optimization**
- *Principle:* Use cloud resources efficiently to meet performance requirements.
- *Key practices:* Latency reduction (caching, CDN, edge); bandwidth optimization; workload-to-resource mapping (Spot vs on-demand, machine types, accelerators); database indexing & partitioning; query optimization.
- *GCP angle:* Cloud CDN; Memorystore (Redis, Memcached); Dataflow autoscaling; BigQuery clustering & partitioning; Machine CPU platform selection; Cloud Trace.

**5. Cost Optimization**
- *Principle:* Run systems that deliver business value at the lowest price point.
- *Key practices:* Resource rightsizing (use Recommender); use discounts (CUD, SUD, Spot, BigQuery editions autoscaling/slot commitments); lifecycle policies for storage; FinOps practices (labeling, budgets, billing export); migration center (TCO).
- *GCP angle:* Compute Engine Spot VMs; Cloud Storage Autoclass; Cloud SQL shared instances; BigQuery editions (autoscaling slots, commitments); Cloud Billing reports; Migration Center cost estimates.

**6. Sustainability**
- *Principle:* Optimize workloads to minimize environmental impact and emissions.
- *Key practices:* Use renewable energy regions; idle resource cleanup; efficient algorithms; consolidation of workloads.
- *GCP angle:* Choose low-carbon regions (Cloud Region Picker shows carbon-free energy %); Carbon Footprint reporting; Active Assist idle-resource recommendations.

### AI/ML perspective of the WAF

The 2026 exam adds an **AI/ML lens** to every pillar:

- **Operational Excellence + AI:** Observability for model performance (data/concept drift, inference latency, hallucinations); automated retraining pipelines (Agent Platform Pipelines); Gemini Cloud Assist for ops troubleshooting; monitoring feature store freshness.
- **Security + AI:** Prompt injection prevention; sensitive data protection (DLP) applied to prompts & responses; VPC Service Controls for Agent Platform; secure model deployment; Model Armor; human review workflows before inference.
- **Reliability + AI:** Model versioning & rollback (Model Registry); A/B testing (traffic splitting on Agent Platform Endpoints); fallback to rule-based systems; SLO burn-rate alerting on inference.
- **Performance + AI:** Model serving latency (online Endpoints vs batch); caching embeddings (Memorystore); GPU/TPU selection for training vs inference; inference gateway patterns.
- **Cost + AI:** Batch inference (cheaper than online); spot GPUs/TPUs for training; model quantization; caching predictions; shared endpoints.
- **Sustainability + AI:** Efficient model architectures (distillation, quantization); batch training in off-peak hours; consolidate inference workloads.

## Decision guides: WAF pillar → typical GCP services

| Pillar | Key Services & Patterns | When to Use |
|--------|-------------------------|----|
| **Operational Excellence** | Cloud Monitoring, Cloud Logging, Cloud Trace, Cloud Profiler, Error Reporting, Gemini Cloud Assist, Terraform, Infrastructure Manager | Centralize observability; automate deployments; enable on-call troubleshooting |
| **Security** | Cloud IAM, Cloud KMS (HSM/EKM), Sensitive Data Protection, VPC Service Controls, Confidential Computing, Workload Identity Federation, Secret Manager, Cloud Audit Logs | Enforce least privilege; encrypt sensitive data; audit access; segregate sensitive workloads |
| **Reliability** | Cloud SQL HA + cross-region replicas, Spanner multi-region, Multi-Region Cloud Storage, Cloud Load Balancer, GKE regional, Cloud Run (multi-region behind a global LB), Backup & DR Service | Meet SLO targets; survive zone/region failures; enable fast, automated failover |
| **Performance** | Cloud CDN, Memorystore (Redis/Memcached), BigQuery clustering/partitioning, Dataflow autoscaling, Cloud Trace, Compute Engine machine families | Reduce latency; optimize database queries; scale elastically under load |
| **Cost Optimization** | Compute Engine Spot, Cloud Storage Autoclass/lifecycle, Cloud SQL shared-core instances, BigQuery editions/slot commitments, Migration Center, Recommender | Right-size resources; use discounts; retire unused assets; plan migrations with ROI |
| **Sustainability** | Low-carbon regions (Region Picker), Carbon Footprint reporting, efficient algorithms, workload consolidation | Reduce carbon emissions; align with ESG goals |

## GCP Fundamentals Primer

### Regions, zones, and multi-region

**Zones:** A zone is a deployment location within a region (e.g., us-central1-a, us-central1-b). Zones are isolated from each other (separate power, cooling, networking). A single-zone outage is possible but rare.

**Regions:** A region is a geographic area containing multiple zones (e.g., us-central1 contains 4 zones). Deploying across zones in a region is called **multi-zone** deployment and provides HA within that region.

**Multi-region:** Deploying across multiple regions (e.g., us-central1 + us-east1) provides even higher availability and disaster recovery. Trade-off: latency to the nearest region and data consistency costs.

**Latency rule of thumb:** 
- Same zone: <1 ms
- Same region, different zones: 1–5 ms
- Different regions: 10–100+ ms (depends on geography)

### Global vs Regional vs Zonal services

| Service Category | Examples | Characteristics | When to Use |
|---|---|---|---|
| **Global Services** | Cloud IAM, Cloud DNS, Cloud Load Balancing (global external Application LB; regional LB variants also exist), Cloud CDN, Cloud Armor, Pub/Sub (global service; message storage region is policy-controlled) | Configured globally; no region/zone selection | Centralized policy, single config point |
| **Regional Services** | Cloud SQL (regional), Cloud Run (deployed in region, auto-scales across zones), Dataflow, Cloud Storage bucket (regional location type) | Deployed in a region; instances auto-distributed across zones in that region | High availability within a region; lower latency to that region |
| **Multi-Region Services** | Cloud Spanner (multi-region config), Multi-Region Cloud Storage, Firestore (multi-region location) | Replicated across regions; strong or eventual consistency options | Strict geo-distribution, data-residency choices within a geography, very low RTO DR |
| **Zonal Services** | Compute Engine VMs and zonal persistent disks (regional PD also exists), GKE cluster in a zone | Tied to a single zone; failure = outage | Lower cost; acceptable for dev/test |

### GCP resource hierarchy & projects

```
Organization (Org Node)
  └─ Folder (optional, nested)
    └─ Project (billing, APIs, quotas)
      ├─ Service A (Cloud Run, GKE, Cloud SQL, etc.)
      ├─ Service B
      └─ ...
```

- **Organization:** Top-level for IAM, organization policies, and billing hierarchy. Requires a Google Workspace or Cloud Identity account as the org admin.
- **Folder:** Grouping mechanism; optional but recommended for large enterprises. Allows hierarchical IAM policies (inherited by child folders and projects).
- **Project:** Billing & quota unit. Every resource belongs to exactly one project. APIs must be enabled per project.
- **Billing Account:** Linked to one or more projects; payment method for services consumed. Shared across projects within an org is common.

### APIs and service enablement

Each service in GCP (Cloud Run, Cloud SQL, Pub/Sub, etc.) has an associated API that must be **enabled** in your project. Enabling an API allows your code to call it; disabling it prevents all calls and is a way to enforce organizational policy.

Common ways to enable:
- `gcloud services enable compute.googleapis.com`
- Cloud Console → APIs & Services → Library → search & enable
- Terraform `google_project_service` resource

### Shared responsibility model

| Layer | Google Responsibility | Your Responsibility |
|-------|----------------------|---------------------|
| **Physical infrastructure** (data center, power, cooling) | ✓ | |
| **Network & routing (backbone, Cloud Backbone)** | ✓ | |
| **OS & kernel patches** | ✓ (managed services); You (Compute Engine VMs) | Managed services only; VMs = you apply patches |
| **Database software** | ✓ (Cloud SQL, Spanner, Firestore); You (self-managed on Compute Engine) | Managed = update via maintenance windows; self-managed = you manage updates |
| **Application code** | | ✓ |
| **Encryption key material** | Shared (CMEK: you manage the key, Google manages the service); (Google-managed: Google only) | CMEK = you own the keys |
| **Access control & authentication** | ✓ (IAM, VPC, firewalls infrastructure) | ✓ (define roles, groups, policies) |
| **Monitoring & alerts** | ✓ (Observability infrastructure) | ✓ (define alerts, respond) |
| **Disaster recovery plan & testing** | ✓ (tooling like Backup & DR Service) | ✓ (design, test, execute) |

**Key takeaway:** Managed services (Cloud Run, Cloud SQL, Spanner) shift more responsibility to Google, lowering operational overhead. Self-managed (Compute Engine, self-hosted databases) give you control but require more effort.

## 8-week realistic study plan

| Week | Focus | Hours | Key Topics | Practice |
|------|-------|-------|-----------|----------|
| 1 | S1 Design & Plan (Pt. 1) | 8–10 | WAF pillars, requirement types, KPI/SLO definition, HA/DR basics, GCP resource model | Read Ch01, Ch02 guides; 5 scenario Q |
| 2 | S1 Design & Plan (Pt. 2) | 8–10 | Network (VPC, peering, routing), storage types, compute options, trade-offs | Read Ch03, Ch04, Ch05; decision trees; 10 Q |
| 3 | S1 Design & Plan (Pt. 3) + S2 Manage | 10–12 | Data processing (Pub/Sub, Dataflow, BigQuery), CI/CD, IaC, provisioning | Read Ch06, Ch07, Ch13; storage/compute tables; 10 Q |
| 4 | S2 + S3 Security & Compliance | 10–12 | IAM, resource hierarchy, encryption, audit logging, DLP, VPC SC, organization policy | Read Ch10, Ch11, Ch12; hands-on: IAM roles exercise; 12 Q |
| 5 | S3 + S4 Optimize & S5 Implement | 10–12 | Cost optimization, FinOps, troubleshooting, testing, API best practices, Gemini Cloud Assist | Read Ch14, Ch13, Ch15; cost-optimization checklist; 12 Q |
| 6 | S6 Operations & Reliability + AI/ML | 10–12 | Observability, SLOs, chaos engineering, AI/ML workloads, Model Garden, Agent Platform basics | Read Ch15, Ch08, Ch09; monitoring exercise; 12 Q |
| 7 | Case Studies Deep-Dive | 10–12 | Study all 4 case studies; map requirements to services; scenario Q from each | Read Ch16; 3 scenario Q per case study = 12 Q |
| 8 | Final Review & Practice | 10–12 | Re-read weak areas, full-length practice exam (50–60 Q), review mistakes, keyword table | Practice test; 60 Q; review traps table; Ch17 cheat sheet |

**Total: ~76–92 hours over 8 weeks = ~10–12 hours/week from week 3 on.**

Adjust up if you have limited GCP hands-on experience; reduce if you have 2+ years GCP production time.

## What to study less vs. more (community intel)

**Study LESS:**
- App Engine (Cloud Run is the preferred answer; App Engine is legacy in many questions).
- Pure lift-and-shift (VMs only); most questions test cloud-native design.
- Specific gcloud CLI flags or syntax; focus on what commands do, not exact flag names.
- Subnet CIDR math and routing minutiae; high-level VPC design is tested more.
- Billing & quotas details; focus on cost optimization *strategy*, not exact price models.

**Study MORE:**
- **Cloud Deploy** (Continuous Delivery; heavily tested in 2026 exam).
- **Cloud Run** (all variants: services, jobs, functions) — the default answer for "managed compute."
- **Application Integration** (serverless workflow orchestration; newer in 2026 syllabus).
- **Testing strategies** (unit, integration, load, smoke, UAT, chaos engineering).
- **VPC Service Controls** (perimeters, bridges, access levels); data exfiltration prevention.
- **Cloud KMS HSM mode** (FIPS 140-2 Level 3 hardware-protected keys; often chosen for compliance, but HIPAA does not mandate it).
- **Conditional IAM policies** (IAM Conditions: access based on resource attributes such as tags, name or type, request time, or access levels).
- **VPC topology** (Shared VPC, Private Service Connect, Private Google Access, VPC peering) — very detailed.
- **Direct VPC egress** (newer alternative to Serverless VPC Access connectors; no connector VMs to manage).
- **AlloyDB** (PostgreSQL-compatible, PITR, enterprise features).
- **Filestore tiers** (Basic HDD/SSD, Zonal [successor of High Scale], Regional, Enterprise).
- **Bigtable schema design** (key design for performance & cost).
- **Migration Center** (assessment tool; TCO/ROI calculations; beats Pricing Calculator for migrations).
- **VMware Engine** (hybrid cloud, on-prem extension).
- **Backup & DR Service** (managed backup, PITR, cross-region).
- **Cloud KMS external key manager (EKM) mode** (key material never leaves your HSM).
- **Model Garden** (200+ models including Gemini, Claude, Llama, DeepSeek).
- **Agent Platform basics** (Agent Studio, Agent Builder, Agent Runtime, Agent Platform Pipelines, ADK).
- **Gemini Cloud Assist** (in design, in ops, in code review).
- **Colab Enterprise** (notebook environment for AI/ML exploration).
- **NotebookLM** (enterprise knowledge assistant; Gemini Enterprise feature).

## Rebrand glossary (2026 naming update)

| **Old Name (Vertex AI Era)** | **New Name (Exam Guide 2026)** | **What Changed** | **SDK/API Impact** |
|---|---|---|---|
| Vertex AI / Vertex AI Platform | **Gemini Enterprise Agent Platform** / **Agent Platform** | Rebrand to emphasize agent-centric design | `aiplatform.googleapis.com` endpoint unchanged; SDK calls still work |
| Vertex AI Studio | **Agent Studio** | UI for prompt engineering & model testing | Same backend |
| Vertex AI Model Garden | **Model Garden** | 200+ models incl. Gemini, Claude, Llama, DeepSeek, Qwen (Models as a Service = MaaS) | Same API |
| Vertex AI Pipelines | **Agent Platform Pipelines** | ML workflow orchestration & automation | Protocol buffers unchanged |
| Vertex AI Search | **Agent Search** (aka "Search") | Semantic search over documents & knowledge | Same service |
| Vertex AI Agent Builder | Part of **Agent Platform** (ADK = Agent Development Kit; Agent Garden) | Open-source framework (ADK) for building agents | ADK is open source; A2A protocol (Agent2Agent) donated to Linux Foundation |
| Agent Engine (Vertex) | **Agent Runtime** | Managed runtime for long-running agents (up to 7 days, sub-second cold starts, custom containers) | Same service as Vertex AI Agent Engine; existing SDK calls keep working |
| Vertex AI Feature Store | **Feature Store** (under Agent Platform) | Feature engineering & serving | Same features |
| Vertex Endpoints | **Agent Platform Endpoints** (online & batch) | Model serving (online = real-time, batch = async) | Same API |
| Vertex Model Registry | **Model Registry** | Version & manage trained models | Same API |
| Vertex Custom Training | **Custom Training** (under Agent Platform) | Run custom Python scripts on infrastructure | Same API |
| Vertex AutoML | **AutoML** (under Agent Platform) | No-code model training | Same API |
| Agentspace | **Gemini Enterprise** | Umbrella product (Gemini Enterprise App = knowledge-worker surface; Workspace Studio = no-code agent builder) | New product branding |
| — | **RAG Engine** | Managed Retrieval-Augmented Generation orchestration (formerly Vertex AI RAG Engine) | Same service under Agent Platform |
| — | **Vector Search** | Managed vector database for embeddings (formerly Vertex AI Vector Search / Matching Engine) | Same service; can serve as a backend for RAG |
| — | **Gemini Cloud Assist** | Generative AI assistant in Cloud Console, Cloud Logging, Cloud Monitoring, BigQuery, etc. | New; available across GCP services |
| — | **Gemini Code Assist** | AI code completion & generation (IDE plugin) | New product line |
| — | **Gemini in Security** | AI-powered threat detection & incident response (in Security Command Center, etc.) | New; being rolled out |
| — | **Gemini 3.x models** | Latest Gemini model series (Pro, Flash, Flash-Lite tiers) | Model IDs keep the same style, e.g. `gemini-3.5-flash`; check the model docs for current IDs |
| — | **NotebookLM** | Enterprise LLM notebook & knowledge assistant (Gemini Enterprise feature) | New product; Gemini-powered |
| — | **Colab Enterprise** | Paid Colab with enterprise features, Google Drive integration, Gemini | New; distinct from free Colab |
| — | **Managed MCP servers** | Protocol servers integrable into Agent Platform (Cloud API Registry, Apigee as bridge) | MCP spec being standardized; managed versions = GCP-hosted |

## Exam traps & keyword → answer

- **"Always use Spanner."** Trap: Spanner is global + strong consistency, but costs noticeably more than Cloud SQL (multi-region configurations are roughly 3x the list price of regional ones). If the question doesn't explicitly require global consistency or multi-row ACID across regions, Cloud SQL is usually correct.
- **"Managed = always better."** Trap: Managed services (Cloud SQL, Firestore) lower operational overhead but may lack fine-grained control. For a scenario requiring custom queries, self-managed (Compute Engine + open-source DB) might be the answer.
- **"Cloud Run can't run long tasks."** Trap: Cloud Run services have a request timeout of up to 60 min; Cloud Run jobs allow task timeouts of up to 168 hours (7 days; 1 hour with GPUs). Cloud Run jobs are often the right answer for batch workloads.
- **"VPC Service Controls blocks everything."** Trap: VPC SC enforces a perimeter around resources; outside the perimeter, access is denied. But you configure what is inside/outside. A question asking "how to allow access from external partners?" likely requires an access bridge, not blocking VPC SC.
- **"Multi-region = multi-region failover."** Trap: Multi-Region Cloud Storage is one bucket replicated; failover is automatic. Multi-zone ≠ multi-region. A question asking "zero-RTO within a single region" needs multi-zone, not multi-region.
- **"Spot VMs = always cheaper."** Trap: Spot VMs are discounted by roughly 60–91 % but can be preempted at any time with no maximum runtime guarantee. Acceptable for batch, CI/CD, ML training, not for user-facing apps. A question saying "cost-optimized but ≥ 99.9 % uptime" rules out pure Spot.
- **"BigQuery is for analytics only."** Trap: BigQuery is also a data warehouse for complex joins and a source for ML (BigQuery ML). A scenario about "building ML models on large datasets" often points to BigQuery ML, not an external ML platform.
- **"AI = Gemini always."** Trap: Not every AI question requires Gemini models. A question about "detecting inappropriate content in images" might be Vision API (pre-built), not Agent Studio. A question about "sentiment analysis of support tickets" might be NL API or CCAI Insights, not Gemini.
- **"Kubernetes = always GKE."** Trap: For on-prem Kubernetes, use GKE Enterprise with Google Distributed Cloud (the Anthos brand has been folded into these). A question saying "standardize Kubernetes across our data center and cloud" likely means GKE Enterprise / Fleet management, not just GKE.
- **"HIPAA = Assured Workloads only."** Trap: Assured Workloads simplifies compliance, but you can meet HIPAA without it using Cloud KMS (HSM), encryption, audit logging, access approval, and BAA. A question asking "which services support HIPAA?" includes Cloud SQL, Cloud Storage, Spanner, etc., not just Assured Workloads.

## Quick-reference cheat table: Global vs Regional vs Zonal services & WAF pillars

| **Service** | **Scope** | **Availability** | **Use When** | **WAF Pillar** |
|---|---|---|---|---|
| **Cloud IAM** | Global | Always HA | Define access globally | Security |
| **Cloud DNS** | Global | Always HA | Resolve names globally | Operational Excellence |
| **Cloud Load Balancing** | Global | Cross-region failover | Route traffic globally with HA | Reliability, Performance |
| **Cloud CDN** | Global | Multi-region cache | Cache content at edge | Performance |
| **Cloud Armor** | Global | Always HA | DDoS & WAF protection | Security |
| **Cloud Run (services)** | Regional | Auto HA across zones | Managed compute, auto-scale | Operational Excellence, Reliability |
| **Cloud Run (jobs)** | Regional | Batch execution | Batch / long-running tasks (task timeout up to 7 days) | Operational Excellence, Cost |
| **Cloud Run functions** | Regional | Event-driven | Lightweight event handlers | Operational Excellence, Cost |
| **GKE (Standard)** | Regional or Zonal | Multi-zone or single-zone | Container orchestration, custom | Reliability (if regional), Performance |
| **GKE Autopilot** | Regional | Multi-zone HA | Managed Kubernetes | Operational Excellence, Reliability |
| **Compute Engine (MIG)** | Regional | Multi-zone HA | Scalable VMs | Reliability, Performance |
| **Compute Engine (VMs)** | Zonal | Single zone | Dev/test, legacy | Cost (no redundancy overhead) |
| **Cloud SQL** | Regional | HA = primary + standby in two zones (automatic failover); single-zone without HA | OLTP, familiar SQL | Reliability (if HA enabled) |
| **Cloud SQL (cross-region replica)** | Cross-region | Async replica; promote for DR (not automatic) | Regional-outage DR, read scaling | Reliability |
| **Cloud Spanner** | Multi-region (config option) | Global, strong consistency | Globally distributed ACID | Reliability, Performance |
| **Cloud Storage (standard bucket)** | Regional | Single region, replicated across zones | OLAP, data lake, archive | Cost, Reliability (within region) |
| **Cloud Storage (multi-region)** | Multi-region | Geo-replicated | HIPAA, GDPR, zero-RTO | Reliability, Security |
| **Cloud SQL Auth Proxy** | Client-side tool | Connects to a regional Cloud SQL instance | Secure client connection | Security |
| **Pub/Sub** | Global service (regional storage policy) | Auto HA across zones | Async messaging | Operational Excellence, Reliability |
| **Firestore (incl. Datastore mode)** | Regional or multi-region (chosen at creation) | Multi-zone (regional) or multi-region | NoSQL, high availability | Reliability |
| **Bigtable** | Regional (cluster in region) | Multi-cluster for DR | Time-series, analytics | Performance |
| **Memorystore (Redis)** | Regional | Multi-zone HA option | Caching, real-time data | Performance |
| **Agent Platform** | Regional | Zones within region | ML workloads, training | Performance, Cost (with spot) |
| **BigQuery** | Regional or multi-region (US/EU) dataset location | Multi-zone within region; cross-region replication/DR available | Analytics warehouse, ML | Performance, Cost |
| **Dataflow** | Regional | Auto-scales across zones | Streaming & batch ETL | Reliability, Performance |

## Practice scenarios

**Scenario 1: Altostrat Media — Surge Handling**

Altostrat Media's podcast transcription service runs on Cloud Run. During breaking news (10x traffic spike), transcription latency increases from 5s to 50s, causing timeouts. The team wants minimal operational changes. Which approach is best?

- A. Manually scale Cloud Run replicas to 1000 max instances; adjust based on monitoring.
- B. Set Cloud Run max instances to 1000 and add a CPU utilization threshold for autoscaling.
- C. Migrate to GKE with HPA (Horizontal Pod Autoscaler) for finer control.
- D. Decouple with Pub/Sub: the API enqueues each transcription job and returns immediately; a Cloud Run worker service consumes the queue and writes results back (async processing).

**Answer:** D

**Why:** Transcription is long-running work done synchronously in the request path, so the surge turns into client timeouts. Queue-based load leveling absorbs the burst, lets workers scale on backlog, and keeps managed services. A is manual and does not fix the synchronous design. B is not how Cloud Run scaling is configured (it already autoscales on concurrency/CPU; raising max instances alone does not remove the timeouts). C is a platform migration, the opposite of minimal change.

---

**Scenario 2: EHR Healthcare — HIPAA Compliance**

EHR Healthcare must migrate MySQL databases to GCP with HIPAA compliance. They require encryption at rest and in transit, audit logging of all access, and external key management (keys never leave their HSM). Which configuration meets these requirements with minimal operational overhead?

- A. Cloud SQL with CMEK backed by Cloud EKM (External Key Manager) keys held in their own HSM; enable Cloud Audit Logs (including Data Access logs).
- B. Cloud SQL with Google-managed encryption keys (default); enable Cloud Audit Logs.
- C. Cloud SQL with CMEK (Cloud KMS software mode); enable Cloud Audit Logs.
- D. Compute Engine + self-managed MySQL with customer-provided HSM.

**Answer:** A

**Why:** The deciding requirement is "keys never leave their HSM", which is what Cloud EKM provides (EKM is a key protection level alongside software and HSM; it is used instead of, not on top of, Cloud HSM keys). HIPAA itself does not mandate CMEK: Google-managed encryption at rest and TLS in transit are on by default and Google signs a BAA. B and C do not meet the external-key requirement; D meets it but at high operational overhead. Cloud Audit Logs covers the access-audit requirement.

---

**Scenario 3: KnightMotives Automotive — Hybrid Connectivity**

KnightMotives has a legacy mainframe on-prem and wants to migrate gradually to GCP. They need >= 99.9% availability for the hybrid link, ~5 Gbps (burst to 10), and data must never route over the internet. Which setup is best?

- A. Cloud VPN with HA configuration (2 tunnels).
- B. Dedicated Interconnect (10 Gbps) with one circuit.
- C. Dedicated Interconnect with two 10 Gbps circuits in different edge availability domains of the same metro.
- D. Partner Interconnect (50 Mbps) + Cloud VPN as primary.

**Answer:** C

**Why:** Dedicated Interconnect gives 10 Gbps circuits (covers the 5–10 Gbps requirement) over a private path that never touches the internet. The 99.9 % SLA requires two circuits in different edge availability domains (99.99 % needs four circuits across two metros). A and D traverse the public internet (IPsec-encrypted, but still internet-routed) and are bandwidth-limited; B is a single circuit with no SLA. Adding a backup Cloud VPN would violate the "never over the internet" constraint.

---

**Scenario 4: Cymbal Retail — Cost Optimization for AI Model Serving**

Cymbal Retail runs product-recommendation models on Agent Platform Endpoints. 60% of traffic is predictable (business hours); 40% is spiky (off-hours, promotions). Model serving latency must stay <100 ms. They want to optimize cost. Which approach is best?

- A. Use a single online Endpoint with a fixed number of replicas sized for peak load.
- B. Use an online Endpoint with autoscaling (min replicas for the predictable baseline, scale out for spikes); use Spot capacity only for training jobs.
- C. Use Batch Endpoints for all predictions (async, cheaper).
- D. Split: on-demand Endpoint for business hours, Spot Endpoint for off-hours; failover via traffic split.

**Answer:** B

**Why:** Autoscaling matches capacity to the spiky 40 % while the baseline replicas (which can be covered by committed use discounts) serve the predictable 60 %, and keeps latency under the target. A pays for peak capacity around the clock. C cannot meet an interactive <100 ms latency target. D puts latency-sensitive serving on preemptible capacity during the spiky period, which risks eviction at exactly the wrong time. Provisioned Throughput is a Gemini (generative model) consumption option, not a setting for custom-model Endpoints.

---

**Scenario 5: When to Use Agent Platform vs. BigQuery ML**

Your team must build "similar product" recommendations for a catalog of 5M items using their text descriptions and images. You want to generate embeddings, find nearest neighbours, and serve results in <50 ms with minimal ML operations. Which platform is best?

- A. BigQuery ML (CREATE MODEL) + BigQuery ML.PREDICT() at serving time.
- B. Agent Platform (Custom Training with TensorFlow) + Agent Platform Endpoints.
- C. Dataflow pipeline (Spark MLlib) → Bigtable → Cloud Run for serving.
- D. Agent Platform Model Garden (pre-trained embeddings) + Vector Search for inference.

**Answer:** D

**Why:** Model Garden provides pre-trained (multimodal) embedding models, so no custom training is needed. Vector Search is a managed approximate-nearest-neighbour index built for low-latency lookups at this scale. Lowest operational overhead and fastest to deploy. A is batch-oriented and not a low-latency serving path; B requires building and operating a custom model; C is a self-built stack. (A collaborative-filtering model trained on user-item interactions would be a different problem, pointing to custom training such as a two-tower model.)

---

**Scenario 6: Compliance — Data Residency & VPC Service Controls**

Your bank (GDPR-regulated) must store EU customer data exclusively in europe-west1 (Belgium). You want to allow authorised engineers in the US to query the data but prevent any data exfiltration outside the approved boundary. Which approach is best?

- A. Store data in europe-west1 Cloud Storage; use firewall rules to block non-EU IPs.
- B. Store data in europe-west1; use the Organization Policy resource-locations constraint to restrict where resources can be created, plus a VPC Service Controls perimeter around the project with access levels for the authorised engineers.
- C. Store data in europe-west1 BigQuery; use Cloud IAM to grant US engineers a query role; no further controls.
- D. Migrate data to Spanner multi-region EU, use Spanner IAM roles to grant US engineers read access.

**Answer:** B

**Why:** The Organization Policy resource-locations constraint enforces that resources are created only in the allowed location (a data-residency control). A VPC Service Controls perimeter with access levels allows controlled access while blocking data exfiltration through Google APIs. This is the defense-in-depth approach: policy + access control + perimeter. A firewall cannot constrain Google API access (and does not govern where data is stored), C relies on IAM alone, and D does not by itself restrict location or exfiltration. Note that residency (where data is stored) is not the same as controlling where it is viewed from; US engineers viewing the data may still need legal review.

---

**Scenario 7: Operational Excellence — Observability Strategy**

Your team runs microservices on GKE with Pub/Sub, Dataflow, and Agent Platform ML pipelines. You need to correlate end-to-end latency (user request → service → message → pipeline → prediction). Which observability stack is best?

- A. Cloud Monitoring + Cloud Logging + Cloud Trace (distributed tracing).
- B. Cloud Monitoring (metrics only) + basic GKE logs.
- C. Prometheus + Grafana (open-source, deployed in GKE).
- D. Cloud Logging + custom dashboards in Looker.

**Answer:** A

**Why:** Cloud Trace is purpose-built for distributed tracing across services; with OpenTelemetry instrumentation and trace-context propagation (including through Pub/Sub message attributes) it correlates a request end to end. Cloud Monitoring shows metrics; Cloud Logging shows logs; Trace connects the latency path. B lacks tracing, C adds self-managed operational burden, and D cannot attribute latency across hops.

---

**Scenario 8: AI/ML Governance — Model Serving & Data Protection**

Altostrat Media uses a Gemini model (via Agent Studio) to summarize podcasts. The system must: (1) never log full podcast content in logs (PII/sensitive data), (2) ensure predictions are auditable (model version, user, when), (3) support human review before publishing. Which setup is best?

- A. Agent Studio → Gemini API → Cloud Logging (standard); rely on Sensitive Data Protection (DLP) to redact logs.
- B. Gemini API → custom logging (only store result hashes); audit via Cloud Audit Logs.
- C. Agent Studio → Agent Platform Endpoints → Sensitive Data Protection on prompt input; Cloud Audit Logs for model calls; manual review UI before publishing.
- D. Migrate to fine-tuned open-source model (LLaMA) on Agent Platform to avoid Gemini's black-box nature.

**Answer:** C

**Why:** Calling Gemini through Agent Platform keeps access under IAM and Cloud Audit Logs (who called the model, when, from where), Sensitive Data Protection can de-identify PII in prompts and logs before they are stored, and a manual review step (custom UI) gates publishing. Pin an explicit model version so outputs are reproducible and auditable. This is the AI governance pattern. A relies on redacting after the fact, B stores hashes only (no meaningful audit of content), and D swaps models without addressing the actual requirements.

## Sources

- Google Cloud Professional Cloud Architect Exam Guide v6.1 (Oct 2025)
- cloud.google.com/architecture/framework (Well-Architected Framework, verified Oct 2025)
- cloud.google.com/learn/certification/cloud-architect (exam overview, verified Oct 2025)
- cloud.google.com/docs (service-specific documentation, continuous)
- community.gcpstudyhub.com (2026 study guide, anecdotal feedback)
- Exam Security Agreement & NDA (confirmed no public question dumps available; scenarios are original)
