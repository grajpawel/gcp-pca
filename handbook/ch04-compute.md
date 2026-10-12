# Chapter 04 — Compute

## Exam objectives covered

From the 2026 exam guide (v6.1):
- **1.3 — Network, storage, compute:** mapping compute to GKE / Cloud Run / Cloud Run functions; compute choices (spot VMs, custom machine types, specialized workloads).
- **2.3 — Compute systems:** provisioning; volatility (spot vs standard); cloud-native network config for compute (GCE, GKE, serverless networking, Google Cloud VMware Engine); infrastructure orchestration, resource configuration, patch management; container orchestration; serverless.

## What the exam really tests

- **GCE vs GKE vs Cloud Run decision drivers** under latency, scaling, cost, and operational overhead constraints.
- **Autoscaling signals & trade-offs** (HPA vs VPA vs Cluster Autoscaler vs node auto-provisioning; when Spot saves money vs when it breaks workload SLA).
- **Machine family selection** (general, compute, memory, accelerator optimised) and custom machine types for cost & performance.
- **Stateless vs stateful workloads** (MIGs for stateless; GKE StatefulSets / persistent disks for stateful).
- **GKE Autopilot vs Standard** — which removes operational burden.
- **Cloud Run concurrency, min instances, traffic splitting** — common misreads (e.g., min instances are billed, at a reduced idle rate under request-based billing).
- **VMware Engine, Bare Metal** — when you cannot migrate (regulatory, app incompatibility).
- **Cloud Run vs Cloud Run functions** — execution model, invocation style, cold-start, duration limits.
- **Patch management & automation** (OS Config, startup scripts, Golden Images, release channels for GKE).

## Core concepts

### Compute Engine (GCE)

**What it is:** IaaS virtual machines on Google's infrastructure. You manage VM lifecycle, OS, software, networking, security. Automatic live migration preserves uptime during maintenance.

**Machine families:**
- **General purpose** (E2, N2, N2D, N4, C4, C4A (Arm/Axion), C4D, C3, T2D, N1): balanced CPU/memory for web servers, app servers, development. E2 = cost-optimized; N4/C4 = newer-generation price-performance.
- **Compute optimized** (C2, C2D, H3, H4D): high CPU per core, for batch, HPC, scientific computing, gaming.
- **Memory optimized** (M1, M2, M3, M4): high memory-to-vCPU for in-memory databases (SAP HANA), analytics.
- **Storage optimized** (Z3): high local-SSD throughput/capacity for scale-out databases and analytics.
- **Accelerator optimized** (A2, A3, A4, G2, G4 and TPU machines): GPUs (and TPUs) attached (see → Ch09).

Custom machine types let you choose vCPU count and memory within per-family ranges (vCPUs generally in even increments; limited GB per vCPU, extended memory costs extra), on supported series (e.g., N1, N2, N2D, E2, N4); they are priced slightly above equivalent predefined types. Exam: use custom types when no predefined shape fits (avoid paying for unused vCPU or memory); also consider rightsizing recommendations.

**Volatility models:**
- **Standard (on-demand):** always available, consistent billing (per-second, min 1 min).
- **Spot VMs** (successor to preemptible): roughly 60–91 % cheaper, can be preempted at any time with a best-effort shutdown period of up to 30 seconds (preemption notice, optionally longer in preview). **No maximum runtime** (no 24 h cap) and no SLA; workload must tolerate interruption. (Legacy "preemptible" VMs are still supported, same price, but capped at 24 h; Spot is the recommended option.)
- **Committed Use Discounts (CUDs):** 1- or 3-year commitment → roughly 25–70 % off depending on type (resource-based vs spend-based flexible CUDs; → Ch14). Not tied to specific instances.
- **Dynamic Workload Scheduler** (Flex-start / Calendar mode) and **reservations**: for scarce GPU/TPU capacity (→ Ch09). (Do not confuse with BigQuery "Flex Slots", which is a BigQuery concept.)

**Instance groups & Managed Instance Groups (MIGs):**
- **Stateless MIGs:** all VMs identical, interchangeable; auto-heal broken instances; scale based on CPU/custom metrics.
- **Stateful MIGs:** preserve per-VM state (e.g., boot disk snapshots, attached disks); rolling updates; richer. Used for databases, caches needing persistent data.
- **Regional MIGs:** replicate across multiple zones in a region (HA, load balancing built-in).
- **Instance templates:** read-only blueprints define machine type, image, disks, network, startup scripts. MIGs are created from templates.

**Autoscaling:**
- **CPU utilization, load-balancing serving capacity, Cloud Monitoring metrics (e.g., Pub/Sub queue depth as a custom/external metric), schedules and predictive autoscaling** drive scale-out/scale-in.
- **Scaling targets:** min/max instance count, initialization (cooldown) period, scale-in controls.
- Exam trap: "Which metric?" → Read constraint (latency, cost, throughput) to pick the right signal.

**Health checks & autohealing:**
- HTTP, HTTPS, HTTP/2, TCP, SSL and gRPC health checks probe instances periodically (e.g., every 5-10 s; unhealthy threshold of N consecutive failures).
- Autohealing (application-based health check on the MIG) recreates unhealthy instances; a separate health check on the load balancer decides traffic routing.
- Regional MIGs + autohealing + autoscaling = resilient, self-healing stateless tier.

**Sole-tenant nodes:** book entire physical servers (no other customers' VMs). Licensing, compliance, performance isolation.

**Confidential VMs:** CPU-encrypted VMs; data encrypted in memory during execution. For regulated workloads (PII, HIPAA). GCE boots and runs them transparently; small performance penalty.

**Live migration:** during host maintenance Compute Engine migrates the running VM to another host with minimal disruption; preserves IP, storage, network. Not available for Spot/preemptible VMs (they are preempted instead), most GPU/accelerator VMs (terminate or restart on maintenance) and some Confidential VM types ⚠ verify; sole-tenant nodes have configurable maintenance policies. Exam: "Zero-downtime maintenance" → live migration (default `MIGRATE`), unless workload incompatible.

**OS Config / VM Manager & patch management:**
- OS Config agent auto-applies OS-level patches (Linux, Windows) on schedule.
- Golden Images: pre-built VM images with OS, middleware, app code baked in; faster boot, consistency.
- Startup scripts (via metadata) run code at instance launch (install packages, configure, run services).

### Google Kubernetes Engine (GKE)

**What it is:** Managed Kubernetes on GCP. GCP owns control plane; you manage node pools, workloads, networking, security.

**Autopilot vs Standard:**

| Aspect | Autopilot | Standard |
|--------|-----------|----------|
| Control plane | Managed by Google | Managed by Google |
| Nodes | Fully managed (auto-provisioned, auto-upgraded) | You add node pools; you manage versions |
| Ops overhead | Minimal (patch mgmt, scaling built-in) | Moderate (manual node lifecycle) |
| Cost transparency | Billed per vCPU/GB (and GPU) requested by running pods (plus cluster fee); no cost for idle node capacity | Per-node (VM) billing, including unused capacity |
| Security posture | Hardened defaults; VPC-native; no node SSH/privileged pods | Flexible; must configure |
| Scaling | Automatic node provisioning (compute classes) | Node pools + Cluster Autoscaler / node auto-provisioning |
| When to use | Managed, low-ops workloads (Google's recommended default) | Full control: custom node config/OS, privileged or DaemonSet-heavy workloads, specific machine types |

Note: Autopilot mode can also be enabled per workload in Standard clusters via compute classes ⚠ verify; Autopilot supports GPUs and Spot via compute classes.

**Node pools:**
- Cluster has a default pool + optional additional pools (e.g., one for general, one GPU-accelerated for ML).
- Each pool: machine type, zone distribution (zonal vs regional), disk, network tags, labels.
- Pools can have different autoscaling policies.

**Autoscaling orchestrators:**
- **HPA (Horizontal Pod Autoscaler):** scales pods (replicas) based on CPU, memory, custom/external metrics (e.g., request rate). Reacts within about a minute.
- **VPA (Vertical Pod Autoscaler):** recommends and optionally applies CPU/memory *requests* per container (modes: Off/recommendation, Initial, Auto; Auto traditionally evicts/restarts pods, in-place resize is newer). Slower convergence; for right-sizing. **Do not run HPA and VPA on the same CPU/memory metric for the same workload** (they conflict); use VPA in recommendation mode, or HPA on custom/external metrics.
- **Cluster Autoscaler (CA):** scales *nodes* (VMs) when pods are pending or nodes are underutilized. Works with node pools.
- **Node Auto-provisioning (NAP):** GKE auto-creates pools with right machine types for pending pods. Less operational friction.
- Typical pattern: NAP (or CA) + HPA (or VPA); each layer solves different bottleneck.

**Cluster topology:**
- **Zonal cluster:** control plane & nodes in one zone. Single point of failure.
- **Regional cluster:** control plane replicated across zones; nodes spread across multiple zones. HA by default. Exam preference: regional for prod.
- **Private clusters:** nodes have no public IPs (Cloud NAT provides internet egress); control-plane access via authorized networks, bastion/IAP or DNS-based endpoint. Network-isolated.

**GKE Enterprise (Multi-cluster / Fleet):**
- Fleets group multiple GKE clusters (on-prem, cloud, other clouds) for unified management (a fleet is a logical grouping, not a single control plane).
- **Fleet features:** centralized policy and config (Config Sync, Policy Controller), multi-cluster Gateway/Ingress, centralized logging/monitoring, multi-cluster services, Cloud Service Mesh.
- **Anthos was rebranded GKE Enterprise** (premium edition); older "Anthos" wording may still appear.
- Exam context: "Multi-region HA for apps" → GKE multi-cluster + Fleet.

**Workload Identity:**
- Officially **Workload Identity Federation for GKE**: pods get GCP identity from their Kubernetes service account, either directly as an IAM principal (preferred) or by impersonating a GCP service account; not the node's (VM) service account.
- Tokens are short-lived, issued via the cluster's workload identity pool; no keys or JSON files on nodes.
- Exam: "Least-privilege pod-to-GCP access" → Workload Identity + narrow IAM roles.

**Gateway API & ingress:**
- **Ingress:** expose HTTP(S) services to the internet; traffic routing, SSL termination.
- **Gateway API (newer):** multi-cluster routing, advanced traffic management, closer to industry standards.
- Typical: Service + Ingress/Gateway → Cloud Load Balancer (external Application LB); Service type LoadBalancer → passthrough Network LB; pod/node egress to the internet via Cloud NAT (→ Ch03).

**GKE monitoring & health:**
- GKE Monitoring integrates with Cloud Monitoring; dashboards for cluster health, node pool status, pod metrics, workload performance.
- GKE observability dashboards and recommended alerts cover control-plane, node, pod and workload health (→ Ch15). ⚠ verify exact panel names

**Stateful workloads on GKE:**
- **StatefulSets:** ordered pod identity, stable network names, persistent storage per replica.
- **Persistent Volumes (PVs):** backed by Persistent Disk / Hyperdisk (block, zonal or regional), Filestore (shared file), Cloud Storage FUSE etc.
- Typical: database (MySQL, PostgreSQL, MongoDB) via StatefulSet + PV; data survives pod restarts.

**Release channels & upgrades:**
- **Rapid, Regular, Stable (and Extended) channels:** trade new-feature availability against version maturity (Rapid = newest versions first, Stable = most validated).
- GKE auto-upgrades control planes and nodes; control with maintenance windows/exclusions.
- Exam: "Minimise upgrade disruption" → release channel + maintenance windows/exclusions + surge (or blue-green) node upgrades + Pod Disruption Budgets.

### Cloud Run

**What it is:** Fully managed, serverless compute. Deploy container images or source code; auto-scale from 0 to 1000s of instances. No infrastructure management.

**Services vs jobs:**
- **Services:** long-running, request-driven (HTTP). Auto-scale per request load. Examples: web APIs, backends.
- **Jobs:** batch workloads, event-triggered or scheduled; run to completion. Examples: data processing, report generation.

**Concurrency & scaling:**
- **Concurrency:** requests per container instance. Default 80. Increase for I/O-bound workloads → cheaper (fewer instances).
- **Min instances:** instances kept warm even with no traffic (default 0 = scale to zero). Reduces cold-start latency. Billed: under request-based billing idle min instances are charged at a reduced idle rate; under instance-based billing you pay for the full instance lifetime.
- Exam trap: min instances *are* billable (misleads students expecting "true serverless = zero cost idle"). Max concurrency per instance is 1000.

**Traffic splitting & revisions:**
- Each deployment is a new revision. Route % traffic to each revision (e.g., 90 % to v2, 10 % to v1 for canary).
- Simplifies canary deployments, A/B testing.

**GPUs on Cloud Run:**
- Attach an NVIDIA GPU (L4 or RTX PRO 6000 Blackwell) to a service or job instance; enables inference and batch/light-training jobs. Billed per instance (GPU + CPU + memory, instance-based billing); can scale to zero; instances start in roughly 5 s. No TPUs.
- Exam: "Real-time image inference" → Cloud Run + GPU, or GKE with GPU node pool.

**Direct VPC egress** (→ Ch03): Cloud Run can send outbound traffic into your VPC (to on-prem, private databases) with no connector to manage; the older Serverless VPC Access connector is still supported. Static outbound IP = VPC egress + Cloud NAT.

**Cloud Run vs GKE decision:**

| Factor | Cloud Run | GKE |
|--------|-----------|-----|
| Cold start | Typically sub-second to a few seconds (depends on image/app); avoid with min instances | Seconds for a pod on an existing node; minutes if a new node must be provisioned |
| Min latency | 0 (no idle) | Overhead of containers/orchestration |
| Ops overhead | None (serverless) | Moderate–high (cluster, upgrades, security) |
| Pricing model | Request-based (per request + CPU/memory while handling requests) or instance-based billing | Per node (Standard) or per pod resource request (Autopilot), plus cluster fee |
| Concurrency control | Built-in (config, max 1000/instance) | Manual (via pod limits, HPA) |
| GPU/TPU | GPUs only (L4, RTX PRO 6000); no TPU | Full support (GPUs and TPUs; training, inference) |
| Stateful workloads | Not recommended (short-lived) | Ideal (StatefulSets, persistent storage) |
| Custom kernel/networking | Not available | Full control |

**Choosing:** Cloud Run for HTTP APIs, microservices, event handlers (latency-insensitive). GKE for stateful, long-running, complex networking, on-prem hybrid.

### Cloud Run Functions

**What it is:** Function-as-a-Service (FaaS). Deploy a single function; GCP handles auto-scaling, versioning, routing.

**Execution model:**
- Upload function code (Node.js, Python, Go, Java, etc.).
- Triggered by: HTTP request, Pub/Sub message, Cloud Storage event, Firestore write, scheduled (Cloud Scheduler).
- **Cold-start latency:** 2nd-gen functions run on Cloud Run, so cold starts are comparable to Cloud Run (min instances apply too).
- **Duration limit:** 2nd gen: HTTP functions up to 60 min, event-driven functions up to 9 min. 1st gen: 9 min. Longer → Cloud Run jobs/services.
- **Context.** Functions are the fastest path for simple event handlers (you ship source, not a container); Cloud Run for full API servers/containers.

**Pricing:** 2nd gen follows Cloud Run pricing (request/instance-based); no cost at zero instances.

**Gen1 vs Gen2:**
- **Gen1:** older platform; concurrency 1 per instance, 8 GB/2 vCPU max, 9 min timeout.
- **Gen2** (Cloud Run-based, the default): concurrency up to 1000 per instance, up to 16 GiB/4 vCPU, longer HTTP timeout (60 min), traffic splitting between versions, Eventarc triggers.

### App Engine

**Status on the 2026 exam:** deprioritised (Cloud Run preferred). Not covered here in depth. Exists in two flavors:
- **Standard:** sandbox environment, auto-scaling to 0, cheap, limited customisation.
- **Flexible:** Docker containers on VMs, SSH access, custom runtimes, slower scaling; does not scale to zero (at least one instance).
- **Exam shorthand:** rarely tested; if it appears, treat as legacy. Prefer Cloud Run (modern serverless) or GKE (orchestration).

### Cloud Tasks

**What it is:** Managed task queue. Your code enqueues tasks; Cloud Tasks dispatches each as an HTTP request (to Cloud Run, Cloud Run functions or any HTTP endpoint) or App Engine request, asynchronously, with retries and rate limiting. (Pub/Sub is a separate service for event fan-out.)

**Use case:** decoupling heavy backend work from frontend requests (e.g., send email, process video after form submission).
- Retries with configurable exponential backoff; no native dead-letter queue (Pub/Sub has DLQs).
- Task-name deduplication, dispatch rate limits and max concurrency, scheduled delivery.
- See → Ch06 for orchestration & queues.

### Batch

**What it is:** Managed batch computing for large-scale, parallelizable workloads. Define job = container(s) + resource needs; Batch provisions, schedules, scales.

**Typical use:** ML training, scientific simulations, data processing pipelines.
- Task groups = parallelism (many tasks across VMs it provisions for you).
- Runs on Compute Engine VMs, including Spot VMs (cheap batch) and GPUs.
- Monitoring, retry, autoscaling.

### Google Cloud VMware Engine

**What it is:** Managed VMware vSphere / vSAN / NSX clusters on GCP. Run VMware-based apps (vSphere VMs) natively on GCP.

**When:** an existing on-prem VMware (vSphere) estate must move quickly with no re-platforming (same tools, same VM images, same operational model), e.g., data-centre exit. Windows, SAP or Oracle do NOT by themselves require VMware Engine; they also run on Compute Engine.
- Private connectivity to your VPC and Google services (via private services access / VPC peering).
- Hybrid connectivity (VPN, Interconnect) to on-prem VMware; VMware HCX for migration.
- Licensing: new license-included VMware capacity ended 15 Oct 2025, so new nodes need bring-your-own VMware (VCF) subscriptions.

**Trade-off:** dedicated bare-metal nodes (costly, minimum cluster size), less cloud-native than Compute Engine/GKE, but full VMware ecosystem.

### Bare Metal Solution

**What it is:** Dedicated physical servers in Google-managed regional extensions, connected to Google Cloud at low latency (no hypervisor layer). Mainly for specialised workloads that cannot be virtualised, e.g., Oracle Database with licensing/hardware certification requirements. ⚠ verify current product naming/positioning

**When:** Performance, licensing, or app requirements preclude virtualisation.

## Decision guides

### Compute decision tree (flowchart-style)

```
START: I need to run code/workloads on GCP
  ├─ "Existing VMware (vSphere) estate to move as-is, or non-virtualisable/Oracle-licensed hardware?"
  │  ├─ YES → VMware Engine OR Bare Metal Solution
  │  └─ NO ↓
  ├─ "Is it a simple event handler or short-lived function?"
  │  ├─ YES → Cloud Run Functions
  │  └─ NO ↓
  ├─ "Is it HTTP API, microservice, or event-driven, containerised, requests ≤ 60 min?"
  │  ├─ YES → Cloud Run (Services)
  │  └─ NO ↓
  ├─ "Is it stateful, needs complex networking, or long-running daemon?"
  │  ├─ YES → GKE (or GCE if not containerised or needs OS-level control, e.g., Windows)
  │  └─ NO ↓
  ├─ "Simple stateless app, VM-based, no Kubernetes skills?"
  │  ├─ YES → GCE + MIG
  │  └─ NO → GKE Autopilot (default); GKE Standard if node-level control needed
  └─ For batch jobs, ML training → Batch or GKE Job resource
```

### Autoscaling decision table

| Scenario | Choose | Why |
|----------|--------|-----|
| Pod CPU spikes, need more replicas fast | HPA (metric: CPU %) | Reacts within about a minute; scales replicas within one cluster |
| Containers over-provisioned (CPU request >> actual use) | VPA | Right-sizes requests; saves cost/improves density |
| Pending pods, nodes at capacity | Cluster Autoscaler or NAP | Adds VMs to cluster; NAP simpler |
| VM CPU high, instance group idle → need more VMs | GCE MIG autoscaling | Scales pool by metric (CPU, custom) |
| Custom metric (e.g., Pub/Sub queue depth) | GCE MIG or GKE HPA | Both support custom metrics via Monitoring |

## Well-Architected Framework lens

| WAF Pillar | Compute pattern | What to do |
|-----------|------------------|-----------|
| **Reliability** | Multi-zone, redundancy, health checks | GKE regional cluster + MIGs with autohealing; Cloud Run inherently replicated |
| **Performance** | Low-latency decision | Cloud Run with min instances (avoids cold starts); GKE pods on warm nodes; GCE flexible (C/H-series for compute-bound) |
| **Cost Optimization** | Right-size, use discounts | Custom machine types; Spot VMs (stateless batch); CUDs (steady-state); Cloud Run for bursty; GKE Autopilot for variable |
| **Security** | Least privilege, encryption, compliance | Workload Identity (GKE); Confidential VMs; OS Config patch automation; sole-tenant nodes (compliance) |
| **Operational Excellence** | Automation, observability | GKE Autopilot (minimal ops); OS Config (patches); Cloud Monitoring dashboards; startup scripts (automation) |
| **Sustainability** | Efficient resource use | Spot VMs (use spare capacity); autoscaling and scale-to-zero (right-size); VPA (density); lower-carbon regions |

## AI/GenAI angle

- **Model training:** AI Hypercomputer stacks use GKE + Batch + GPUs/TPUs. → Ch09.
- **Inference:** Cloud Run + GPU for lightweight models; GKE + GPU node pools for production serving.
- **Model serving:** Agent Platform endpoints are the managed option (Gemini and custom models; custom containers from Artifact Registry); self-managed serving on GKE or Cloud Run when you need full control.
- **Agent Platform Pipelines:** orchestrate preprocessing (GCE/GKE) → model training (Batch) → inference (Cloud Run/GKE). See → Ch08.
- **Prompt deployment risk:** containerise AI workloads (Cloud Run, GKE) with input validation, rate limiting, DLP scanning. → Ch11.

## Case-study hooks

**Altostrat Media:**
- Transcoding, metadata extraction: Cloud Run functions (event-driven on Cloud Storage uploads) or GKE batch jobs.
- Scalable recommendation engine: GKE with stateless services + BigQuery lookups; Cloud Run for APIs.
- 24/7 chatbot: Cloud Run services auto-scale to 0 when idle (cost); custom concurrency tuning.

**Cymbal Retail:**
- Catalog enrichment pipeline: Batch (parallelise attribute generation, image processing); GKE StatefulSet for long-lived cache.
- Product discovery APIs: Cloud Run services with traffic splitting (canary new models).
- On-prem SKU ingestion: Cloud Run (Direct VPC egress) or a GKE private cluster, reaching on-prem over Cloud VPN/Interconnect (no public IP for the database).

**EHR Healthcare:**
- Stateful patient-data services: GKE with StatefulSets + persistent disks; strict isolation per tenant (Workload Identity, RBAC).
- 99.9 % availability: GKE regional cluster, multi-zone node pools, health checks, autohealing.
- Insurer integrations (legacy APIs): Cloud Run functions enqueuing messages to on-prem; Cloud Tasks for retries.

**KnightMotives Automotive:**
- Vehicle AI inference (real-time, low-latency): GKE with GPU node pools; Cloud Run with GPU for lighter models; emphasis on edge deployment (minimal latency).
- Simulation environment: GKE for containerised workloads; Batch for large training jobs.
- Dealer tools (web APIs): Cloud Run or GKE; traffic splitting for feature rollout.

## Exam traps & keyword → answer

1. **"Minimal operational overhead, auto-scaling to 0"** → Cloud Run (not GCE MIGs, not GKE without explicit scaling to 0).
2. **"Stateful workload with persistent data"** → GKE StatefulSet + PersistentVolume (not Cloud Run, not MIGs without disk snapshots).
3. **"Low cold-start latency for APIs"** → Cloud Run with min instances (and startup CPU boost); 2nd-gen Cloud Run functions run on the same platform, so they are not inherently slower. GKE only if pods stay warm on existing nodes.
4. **"Spot VMs interrupted → need immediate failover"** → Spot is for fault-tolerant, stateless (e.g., batch); use standard VMs / HA clusters for critical.
5. **"Scale 10,000 pods"** → GKE with HPA (per-pod), not GCE MIGs (VM-level scaling less granular).
6. **"Right-size over-provisioned containers"** → VPA (not HPA, not Cluster Autoscaler).
7. **"Windows Server / SQL Server / SAP"** → Compute Engine VMs (Windows images, BYOL on sole-tenant if needed; SAP-certified M-series). VMware Engine only if the stem says an existing VMware estate must move as-is; Bare Metal Solution only for specialised (e.g., Oracle) hardware needs.
8. **"Custom machine type to cut 30 % cost"** → Feasible; adjust vCPU/memory to workload profile (e.g., memory-bound app → more memory, fewer vCPU).
9. **"Live migration = zero downtime"** → Effectively true for standard GCE VMs (default `MIGRATE`); not available for Spot/preemptible, GPU VMs and some Confidential VM types ⚠ verify (they terminate/restart on maintenance).
10. **"Golden image for faster boot"** → Correct; bake OS + middleware into custom image; reduces startup-script time.
11. **"OS Config patches → Confidential VM?"** → OS Config/VM Manager works on Confidential VMs like any other VM; Confidential computing adds encryption of data in use (memory), while data at rest is encrypted by default on every disk.
12. **"GKE Autopilot for minimal ops; GKE Standard for control"** → Autopilot removes node lifecycle; Standard lets you choose machine type, disk, networking details.
13. **"Min instances on Cloud Run = free warm-up"** → No; min instances stay warm (service never scales below them) and are billed (reduced idle rate under request-based billing). Default 0 = scale to zero.
14. **"Revision traffic split = blue/green"** → Yes, simplified; split revision traffic, no service restart.
15. **"VPA and HPA together?"** → Not on the same CPU/memory metric (they fight). Use VPA in recommendation mode, or HPA on custom/external metrics with VPA on resources.

## Quick-reference cheat table

| Service | Scale to 0 | Min latency | Ops burden | Pricing | Stateful | Concurrency | GPU/TPU | Best for |
|---------|-----------|------------|-----------|---------|----------|-------------|---------|----------|
| **Cloud Run** | Yes | Cold start ~seconds (none with min instances) | None | Request- or instance-based | No | Built-in (default 80, max 1000) | GPU (L4/RTX PRO 6000) | HTTP APIs, microservices, event handlers |
| **Cloud Run Functions** | Yes | Like Cloud Run (2nd gen) | None | Cloud Run pricing (2nd gen) | No | Up to 1000 (2nd gen); 1 (1st gen) | No | Simple event triggers, webhooks |
| **GKE Autopilot** | No* | Seconds (warm node) to minutes (new node) | Minimal | Per pod vCPU + memory requested (+ cluster fee) | Yes | HPA/pod-level (you configure) | Yes (via GPU requests/compute classes) | Containerised, variable workloads, managed |
| **GKE Standard** | No* | Seconds (warm node) to minutes (new node) | Moderate | Per node (even idle) | Yes | Full control | Yes | Heterogeneous, custom networking, on-prem hybrid |
| **GCE + MIG** | No | Tens of seconds (VM boot) | Moderate–High | Per VM second | Yes (disks/snapshots) | N/A (VM-level; MIG autoscaling) | Yes (A/G series) | Stateless, cost-optimised, simple scaling |
| **Batch** | Yes | Minutes | Low | Per vCPU + GB-sec | No | Parallelism via task groups | Yes | Batch jobs, ML training, simulations |
| **VMware Engine** | No | N/A (dedicated nodes, provisioned in hours) | High | Per node (BYOL VMware licensing for new capacity) | Yes | Native vSphere | Limited ⚠ verify | Existing VMware estates moved as-is |
| **App Engine** | Yes (Standard) | Seconds | Low | Per request or per instance | Limited | Auto | No | Legacy apps; not for new builds (Cloud Run preferred) |

*The GKE control plane is always running (cluster management fee, with a free tier), so a cluster never fully "scales to zero". Autopilot bills per pod, so with zero pods you pay only the cluster fee; Standard node pools can autoscale down to zero nodes but system pods usually keep at least one node.

## Practice scenarios

**Q1: Scenario — Stateless web microservice, highly variable traffic (0–10,000 requests/min), cost-critical, must scale to near-zero idle.**
- Likely design: Cloud Run service, concurrency 100, no min instances.
- Automates: scaling up on traffic spike, scaling to 0 when idle. Why: a cold start of about a second is acceptable for web (not real-time). Cost per request optimised; no idle fees.
- Why *not* GKE? Standard clusters keep nodes running (cost even when idle) and add cluster-ops overhead; Autopilot is workable but heavier than needed here.
- Why *not* GCE MIG? Must manage instance lifecycle; higher ops burden.

**Answer: A)** Cloud Run with autoscaling and 0 min instances.

---

**Q2: Scenario — Multi-tenant SaaS application. Each tenant has persistent data (PostgreSQL), needs <100 ms latency to DB, strict network isolation, ≥99.9 % uptime.**
- Design: GKE Standard, regional cluster (multi-zone HA), StatefulSet for PostgreSQL + PersistentVolume (persistent disks). Network policies isolate tenants. Workload Identity + IAM roles per tenant service account.
- Why: Stateful + persistent storage ruled out Cloud Run. Regional + multi-zone HA meets SLA. StatefulSet preserves pod identity & storage.
- Why *not* GKE Autopilot? Autopilot also supports StatefulSets, PVs and network policies; pick Standard only because the stem implies node-level control (machine type, local SSD, privileged workloads).
- Caveat: in a real design a managed database (Cloud SQL/AlloyDB with regional HA) is normally preferred over self-hosted PostgreSQL on GKE; the in-cluster DB is the answer here only because the question is constrained to compute choices and per-tenant co-location.

**Answer: C)** GKE Standard, regional, StatefulSet + persistent disk, Workload Identity for isolation.

---

**Q3: Scenario — Daily batch ETL job (3 AM, processes 100 GB of data, ~10 hours, parallelizable, bursty, used once a day).**
- Design: Cloud Batch (or Dataflow). Define job with task groups (parallelism factor). Use Spot VMs (up to 90 % savings on VMs). Schedule via Cloud Scheduler.
- Why: Batch workload (to completion, not continuous). Spot acceptable (fault-tolerant, long-running doesn't mind interruption; Batch auto-restarts). Minimal ops (managed orchestration).
- Why *not* GKE? Overkill; a Standard cluster's nodes cost money while idle ~14 h/day unless carefully scaled down. Batch provisions and removes VMs per job.
- Why *not* GCE MIG? Would need to manually orchestrate parallel tasks; Batch does it.

**Answer: B)** Cloud Batch with Spot VMs, scheduled via Cloud Scheduler.

---

**Q4: Scenario — Serving TensorFlow models for image inference (50 ms latency SLA, peak 1000 req/sec, GPU-accelerated).**
- Design: Cloud Run + GPU, or GKE + GPU node pool + model-serving container.
- Cloud Run: simpler, auto-scales, no ops; GPU instances start in roughly 5 s, so set min instances to protect the 50 ms SLA from cold starts. Concurrency tunable (max 1000 per instance), but a GPU model usually needs a much lower value.
- GKE: more control, can co-locate multiple models, complex ingress, Cloud Service Mesh for traffic management.
- Likely answer: Cloud Run + GPU (simpler, managed).
- Why *not* App Engine? No GPU support. Why *not* Cloud Run Functions? Not designed for high-throughput inference.

**Answer: A)** Cloud Run with GPU, concurrency tuned for request rate.

---

**Q5: Scenario — Complex multi-service system: API gateway (stateless), data-processing sidecars (stateful, persistent cache), event handlers. Needs centralized logging, multi-region failover, on-prem integrations.**
- Design: GKE multi-cluster (Fleet). Cluster 1 (us-central1), Cluster 2 (europe-west1). StatefulSet for cache (regional affinity). Cloud Run for stateless APIs if traffic is spiky; GKE Services for sidecars.
- Logging: Cloud Logging (centralized across both clusters via Fleet integration).
- On-prem: GKE private clusters with Interconnect, or Cloud VPN, into VPC with on-prem connectivity.
- Why GKE multi-cluster? Provides geographic HA, reduces latency (each region has a cluster), unifies management.
- Why *not* single GKE cluster? Single region is not true HA; multi-cluster is standard for SaaS.

**Answer: C)** GKE multi-cluster (Fleet) in 2 regions, StatefulSet + cache, Cloud Logging aggregation, on-prem Interconnect.

---

**Q6: Scenario — Company must move a legacy 32-bit Windows app (runs on 64-bit Windows Server) to GCP, no time to refactor. Per-core BYOL licensing; no VMware in the current estate. ≥99.99 % uptime target.**
- Design: Compute Engine Windows Server VMs (lift-and-shift), on sole-tenant nodes if the BYOL terms require dedicated hardware, deployed as instances in at least two zones behind a load balancer (stateful: regional persistent disks or application-level replication).
- Why: no VMware estate to preserve, so VMware Engine adds cost and dedicated-node minimums for nothing. Windows runs natively on Compute Engine; GKE/Cloud Run are not suitable for a legacy Windows desktop-style app (GKE supports Windows node pools but needs containerisation).
- Compute Engine SLA is 99.99 % for instances spread across multiple zones, which fits the target.
- Sole-tenant nodes are zonal, so HA comes from running instances in multiple zones, not from the nodes themselves.
- If the stem instead said "currently runs on on-prem VMware vSphere", the answer would flip to VMware Engine.

**Answer: D)** Compute Engine Windows VMs (sole-tenant if BYOL requires), across two or more zones.

---

**Q7: Scenario — Team wants to adopt Kubernetes, but small workload (50 pods), uncertain traffic. Minimal DevOps staff (1 person). Prefer low ops overhead.**
- Design: GKE Autopilot. Nodes auto-provisioned, auto-upgraded, patches applied automatically. Pod RBAC and Workload Identity baked in.
- Why: Removes node lifecycle management (biggest ops burden). Auto-provisioning means don't over-spec clusters.
- Why *not* GKE Standard? Would require 1 person to manage node pools, upgrades, security patches. Autopilot is purpose-built for this.
- Why *not* Cloud Run? Only if workloads are HTTP-only; if they require long-running daemons or batch processing, GKE Autopilot is better.

**Answer: A)** GKE Autopilot.

---

**Q8: Scenario — Over-provisioned Kubernetes cluster. Pods request 4 vCPU / 8 GB memory, but actual usage is ~1 vCPU / 2 GB. No single fix (can't rewrite app). Cost is the constraint.**
- Design: VPA (Vertical Pod Autoscaler) with auto-scaling mode. VPA observes actual usage, adjusts *requests*, triggers pod restart. Cluster Autoscaler then right-sizes node pool (fewer, smaller nodes needed).
- Why: HPA scales *replicas*, not requests. HPA alone won't save cost here (cost is idle, over-provisioned space, not #replicas). VPA identifies over-provisioning, adjusts requests, enables tighter packing.
- Follow-up: also consider Cluster Autoscaler to delete now-idle nodes.

**Answer: C)** Deploy VPA in auto-recommendation + auto-approve mode, observe request adjustments, then scale down nodes via Cluster Autoscaler.

## Sources

- [Google Cloud Compute Engine documentation](https://cloud.google.com/compute/docs)
- [GKE Architecture and design](https://cloud.google.com/kubernetes-engine/docs/concepts)
- [Cloud Run documentation](https://cloud.google.com/run/docs)
- [Cloud Run Functions documentation](https://cloud.google.com/functions/docs)
- [Batch documentation](https://cloud.google.com/batch/docs)
- [VMware Engine documentation](https://cloud.google.com/vmware-engine/docs)
- [GCP Well-Architected Framework — Compute lens](https://cloud.google.com/architecture/well-architected-framework)
- [GCP Exam Guide v6.1 (official)](https://cloud.google.com/learn/certification/cloud-architect)
- [Autoscaling on GCP: when to use HPA vs VPA](https://cloud.google.com/kubernetes-engine/docs/concepts/horizontalpodautoscaler)
- [Cloud Run request timeout](https://docs.cloud.google.com/run/docs/configuring/request-timeout)
- [Cloud Run task timeout](https://docs.cloud.google.com/run/docs/configuring/task-timeout)
- [Cloud Run GPUs](https://docs.cloud.google.com/run/docs/configuring/services/gpu)
- [Cloud Run minimum instances](https://docs.cloud.google.com/run/docs/configuring/min-instances)
- [Cloud Run billing settings](https://docs.cloud.google.com/run/docs/configuring/billing-settings)
- [Cloud Run functions comparison (1st vs 2nd gen)](https://docs.cloud.google.com/run/docs/functions/comparison)
- [Cloud Run functions quotas](https://docs.cloud.google.com/functions/quotas)
- [Spot VMs](https://docs.cloud.google.com/compute/docs/instances/spot)
- [Accelerator-optimized machines](https://docs.cloud.google.com/compute/docs/accelerator-optimized-machines)
- [VMware Engine release notes](https://docs.cloud.google.com/vmware-engine/docs/release-notes)

---

## Extended considerations

### GCE machine family trade-offs in detail

**E2 (general, budget):** shared-core (0.25–1 vCPU) up to 32 vCPU, up to 128 GB memory. Lowest cost; no sustained-use discounts (CUDs still apply). Trade-off: lowest performance per vCPU. Exam context: use for dev/test, low-traffic services. Not for CPU-intensive batch.

**N2 (general, mainstream):** 2–128 vCPU, up to 864 GB (limits vary by CPU platform and type) ⚠ verify. Sweet spot for web servers, app servers, databases. Newer generation and more powerful than N1/E2. Standard recommendation.

**C2 (compute-optimized):** up to 60 vCPU, 240 GB memory. High CPU-to-memory ratio. For scientific computing, batch processing, gaming servers. Cost-per-vCPU higher than N2, justified when CPU is the bottleneck.

**M2 (memory-optimized):** up to 416 vCPU and about 11.7 TB memory (M-series generally for SAP HANA and large in-memory databases) ⚠ verify exact sizes. Extreme memory, low cost/GB. Exam: managed caching layer → Memorystore (no VM needed); self-managed in-memory DB (SAP HANA) → M-series GCE.

**Exam pattern:** Question states workload constraint (e.g., "16 vCPU, 64 GB, I/O-heavy"). Eliminate machines that don't fit. Then choose by cost/perf: E2 (budget), N2 (balanced), C2 (CPU bound), M2 (memory bound), A2/A3 (GPU workloads).

### Preemptible VMs and Spot history

**Preemptible (legacy, ≤ 24 h lifetime, 30 s notice):** older Spot model. Exam may reference; if it does, note Spot VMs supersede them.

**Spot VMs (current, no max runtime, up to 30 s shutdown period):** can run indefinitely (no 24 h cap), heavily discounted, may be preempted at any time when capacity is needed. Standard for stateless batch, MIGs for fault-tolerant workloads.

**Dynamic Workload Scheduler / reservations:** for scarce GPU/TPU capacity, flex-start or calendar-mode requests (and future reservations) obtain capacity for a time window. This is about availability, not a discount. (BigQuery "Flex Slots" is a different, retired product.)

Exam context: Spot + Batch = cheap ML training; Spot + MIG = cost-optimised web tier (if fault-tolerant); standard VMs for critical / HA.

### GKE networking integrations

**Workload Identity Federation (WIF):** external OIDC providers (e.g., GitHub, AWS) → GCP service accounts. For hybrid CI/CD, cross-cloud workloads. Not just GKE, but GKE's use case is straightforward: GitHub Actions workflow assumes GCP service account without static keys.

**Network policies:** restrict pod-to-pod communication (like firewall rules between pods). Combined with Workload Identity, achieves "least-privilege networking + auth."

**Cloud Service Mesh (optional; Istio-based, formerly Anthos Service Mesh):** advanced routing, mutual TLS, canary deployments, observability. Not mandatory (Kubernetes Services / Ingress sufficient for simple routing). Exam: "Advanced traffic management / mTLS between services" → Cloud Service Mesh; simple routing → built-in Services.

### Multi-cluster patterns

**Fleet management:** manage n clusters (different regions, on-prem, other clouds) as one logical fleet from a single Google Cloud console/API.
- Centralized config: Config Sync (formerly Anthos Config Management), a GKE Enterprise feature.
- Multi-cluster Ingress / multi-cluster Gateway: expose services across clusters, route users to the closest healthy cluster.
- Centralized Logging: Cloud Logging aggregates logs from all clusters.
- Policy enforcement: roll out RBAC and Pod Security Standards / Policy Controller constraints across fleets (Pod Security Policies were removed in Kubernetes 1.25).

**Anthos (legacy term):** the former multi-cluster, multi-cloud platform (GKE on-prem, GKE on AWS/Azure, + Anthos Service Mesh), now branded **GKE Enterprise**, with fleets as the grouping mechanism.

### Cloud Run advanced patterns

**Concurrency tuning:** default 80 req/instance. If service is I/O-bound (e.g., waits for DB), increase to 500+; reduce instance count, save cost. If CPU-bound, keep low (e.g., 10–20); more parallelism via more instances.

**Min instances trade-off:** every min instance is billed even with no traffic (reduced idle rate under request-based billing). Use for latency-critical APIs where cold starts break the SLA; avoid for batch and spike-driven workloads.

**Direct VPC egress + outbound static IP:** route all egress through the VPC (not Google-managed public IPs). Enables private database access, on-prem connectivity, outbound firewall rules. Setup: Service → Direct VPC egress (or Serverless VPC Access connector) with all traffic routed to the VPC → Cloud NAT with a reserved static IP. (→ Ch03 for VPC details.)

### Patch management at scale

**Baseline:** OS Config agent (VM Manager) on every Compute Engine VM (GKE nodes are patched through GKE node auto-upgrades instead). Enforces:
- Security patch policies (e.g., "apply all critical patches Tue 2 AM").
- Config drift remediation (e.g., enforce sysctl settings, iptables rules).
- Compliance reporting (show which VMs are out of patch SLA).

**Golden images:** pre-bake patches, middleware, app code into custom image. Reduces startup time (skip "apt-get install" in the startup script). CI/CD pipeline: Packer (or Image Builder) builds image, tests, publishes to a Compute Engine image family (container images, by contrast, go to Artifact Registry). MIGs boot new instances from the image via the instance template.

**GKE release channels:** Rapid/Regular/Stable auto-upgrade schedule. Each channel receives new Kubernetes versions at a different pace. Choose based on risk appetite: Stable = most validated, lower risk; Rapid = newest features but higher risk.

### Batch computing example walkthrough

Architecture: data in Cloud Storage → Batch job → output to BigQuery.
1. Create Batch job definition: container image (from Artifact Registry), resource needs (vCPU, memory, GPU), task group (e.g., 100 parallel tasks).
2. Each task reads a partition of input data, processes (e.g., runs ML inference, transforms), writes output.
3. Batch auto-provisions VMs (Spot, if specified), schedules tasks, auto-scales based on queue depth.
4. On completion, tears down VMs (cost-efficient).
5. Monitoring: Cloud Logging + Cloud Monitoring for job status, task failures, resource usage.

### VMware Engine networking

**Hybrid link:** on-prem VMware and VMware Engine connect over Cloud VPN/Interconnect; VMware HCX migrates VMs and can extend L2 networks between sites.

**Private connectivity:** the VMware Engine private cloud has its own network, peered to your VPC (private services access style peering) with routes to on-prem (Interconnect, VPN). Google services (Cloud Storage, BigQuery, Cloud SQL) are reached privately from there via private connectivity ⚠ verify exact mechanism (peering vs Private Service Connect).

**Licensing:** VMware Engine nodes are dedicated bare-metal hosts (no sole-tenant node needed). New license-included VCF capacity ended 15 Oct 2025, so new nodes use bring-your-own VMware (VCF) subscriptions.

