# Chapter 13 — CI/CD, DevOps, Infrastructure as Code and APIs

## Exam objectives covered

- **S4.1 Technical processes:** SDLC; CI/CD; troubleshooting / root-cause analysis; testing & validation of software and infrastructure; service catalog & provisioning; disaster recovery.
- **S5.1 Advising dev & ops teams:** application & infrastructure deployment; **Apigee API management** best practices; testing frameworks (load/unit/integration); data & system migration and management tooling; Gemini Cloud Assist.
- **S5.2 Programmatic interaction:** Cloud Shell Editor, Cloud Shell Terminal, Cloud Code, SDKs (gcloud, gsutil, bq); **Cloud emulators** (Bigtable, Spanner, Pub/Sub, Firestore); **Infrastructure as Code / Terraform**; accessing Google APIs best practices; client libraries.

## What the exam really tests

- **CI/CD pipeline design:** choosing Cloud Build triggers, Artifact Registry, Cloud Deploy over home-grown Jenkins; canary + rollback decisions driving SLO/RTO.
- **Deployment strategy trade-offs:** rolling vs blue/green vs canary—when to pick each; testing gates (smoke, integration, load, security SAST/DAST) stopping bad releases; approval workflows.
- **IaC: Terraform state management:** why Cloud Storage + locking + modules beat manual provisioning; drift detection; importing legacy infrastructure; when Config Connector / Infrastructure Manager replace Terraform.
- **API platform selection:** Apigee for monetisation/quotas/policies vs lightweight API Gateway for internal microservices vs Cloud Endpoints for gRPC; rate limiting & spike arrest as bottleneck-prevention.
- **Troubleshooting methodology:** logs → traces → error reporting → change correlation (release bisecting); emulators speeding up local dev-loops.

## Core concepts

### SDLC models & DevOps practices

**Waterfall → Agile → DevOps → SRE.** DevOps emphasises velocity (CI/CD, short release cycles) + reliability (monitoring, runbooks, post-mortems). DORA metrics measure DevOps maturity:
- **Deployment Frequency:** how often code reaches prod (daily, weekly, on-demand). Constraint: testing gate throughput.
- **Lead Time for Changes:** commit → prod. Bottleneck: build, approval, deploy stages.
- **Mean Time to Recovery (MTTR):** detect incident → system stable. Tests & rollback speed drive this.
- **Change Failure Rate (CFR):** % of deploys causing incidents. Determined by test coverage & staging fidelity.

Elite performers (2019 State of DevOps thresholds ⚠ verify): on-demand deploys, lead time under 1 day, recovery under 1 hour, change failure rate 0-15%. Low performers: deploys monthly or less, lead time 1-6 months, change failure rate 46-60%. Newer DORA reports call MTTR "failed deployment recovery time" and add a rework-rate metric.

### CI/CD reference architecture on GCP

```
Source (GitHub/GitLab/Bitbucket via Cloud Build repositories / Developer Connect; Cloud Source Repositories is closed to new customers)
  ↓ [Push to branch / PR / tag]
Cloud Build [Triggers: branch, tag, PR, webhook, Pub/Sub]
  ├─ Build phase (docker build → container image)
  ├─ Unit tests (in-container)
  ├─ Security scan (e.g. SonarQube for code; Trivy / Artifact Analysis for images)
  └─ Push to Artifact Registry [Private container & artifact repo]
     ↓
Cloud Deploy [Delivery pipeline: targets (GKE / GKE Enterprise / Cloud Run; custom targets), promotion gates]
  ├─ Dev → Staging (auto)
  ├─ Staging → Prod (approval + canary/verification)
  ├─ Canary deployment (% traffic split, monitor metrics)
  ├─ Verification gate (SLO checks, integration tests)
  └─ Full rollout / Automatic rollback on failure
```

**Cloud Build** 
- **Triggers:** branch / tag / PR / manual / Pub/Sub / scheduled (via Cloud Scheduler).
- **Private Pools:** Google-managed worker pools (in a Google-owned producer network) that you peer with your VPC via private services access, so builds can reach private resources (private GKE control planes, on-prem via VPN/Interconnect), with no public IPs, static internal ranges and VPC-SC support. They are not GKE clusters you run.
- **Build Provenance:** Cloud Build can generate SLSA build provenance and attestations that Binary Authorization can verify (→ Ch11).
- **Source connections:** GitHub, GitLab, Bitbucket via Cloud Build repositories or Developer Connect; Cloud Source Repositories no longer accepts new customers.
- **Service Account:** runs build steps; needs Artifact Registry push, Cloud Deploy release permissions.

**Cloud Deploy** (exam focus: community reports say it gets more weight in the 2025-2026 exam)
- **Targets (runtimes):** GKE cluster, GKE Enterprise (attached/Anthos) clusters, Cloud Run; plus **custom targets** (you supply render/deploy containers, e.g. for other runtimes).
- **Delivery Pipelines:** define promotion stages (dev → staging → prod) with approval gates; releases are rendered with **Skaffold**.
- **Canary & Verify:** canary strategy (e.g. 10% → 50% → 100%) on GKE or Cloud Run, optional verification job per phase, rollback of a failed rollout; automated canary is supported for GKE and Cloud Run targets.
- **Traffic splitting:** Cloud Run revisions are native; GKE canary uses service-based networking or the Gateway API (Flagger/Istio are alternatives outside Cloud Deploy).
- **Approvals:** IAM permission + console/gcloud approval before promotion (`requireApproval` on a target).

**Artifact Registry** 
- Central repo for container images, Maven/npm/Python/Go/apt/yum packages, Helm charts.
- Private by default (IAM-controlled; supports VPC Service Controls, CMEK, vulnerability scanning via Artifact Analysis, remote and virtual repositories).
- **Replaces Container Registry**: Container Registry was shut down (March 2025); `gcr.io` hosts now run on Artifact Registry repositories.
- Integrates with Cloud Build (automatic push), Cloud Deploy (pull), GKE (image pull with the node service account).

**Skaffold:** open-source CLI (from Google) for fast dev loops (watches local files, rebuilds image, redeploys to a dev cluster, also used by Cloud Code). **Cloud Deploy uses Skaffold** (`skaffold.yaml`) to render and deploy releases.

### Deployment strategies: comparison table

| Strategy | Rollout | Rollback Speed | Traffic Loss | Testing Window | GCP Implementation |
|----------|---------|--------|---|---|---|
| **Recreate** | Stop all, start new | Slow (re-init) | Full | Pre-deploy only | GKE: recreate strategy; Compute Engine: replace MIG |
| **Rolling** | 1 pod/instance at a time | Slow (gradual) | None (if SLO met) | Per-instance | GKE: default; Compute Engine: rolling MIG update |
| **Blue/Green** | Old & new in parallel | Fast (switch LB) | None (switch) | Full stack before cutover | Cloud Run: traffic split (0%→100%); Compute Engine: 2 MIGs + LB |
| **Canary** | % traffic to new version | Very fast (revert) | Minimal (<10%) | SLO monitoring; auto-rollback | Cloud Deploy + verification gate; Cloud Run traffic %; GKE via Flagger |
| **Traffic Splitting / Shadow** | Duplicate to new version, observe | N/A (shadow doesn't serve) | None (read-only) | Production metrics; no user impact | Cloud Run (traffic %, sessions sticky); Compute Engine: LB weights; GKE: Flagger, Istio |
| **Feature Flags** | Same version, feature toggle | Instant | None | Per-feature; decoupled from deploy | App config, Firestore, Redis; not GCP-native, but compatible with any deployment |
| **A/B Testing** | Split audience (user cookie/header) | Instant | None | Conversion, UX metrics | Cloud Run session affinity; Compute Engine: LB headers; analytics layer (BigQuery) |

**Decision:** Recreate for stateless, non-critical (demo apps). Rolling for steady-state. Canary for risky changes (new ML model, pricing logic). Blue/green for full-stack confidence & fast rollback. Shadow for "observe before commit" (e.g. new API version).

### Testing strategies & pipeline placement

| Test Type | Scope | When | Gate? | Tools on GCP |
|-----------|-------|------|-------|------------|
| **Unit** | Function/class | Dev loop, Cloud Build | Yes (fail = no artifact) | JUnit, pytest, Jasmine; run in build step |
| **Integration** | Service + dependencies | Cloud Build post-artifact | Yes | testcontainers, docker-compose in a Cloud Build step |
| **Contract** | API schema compliance | Cloud Build (post-integration) | Yes | Pact, Spring Cloud Contract; prevent downstream breakage |
| **SAST** (Static App Security Test) | Code vulnerabilities (no execution) | Cloud Build step | Yes (failed step blocks the build; Binary Authorization then blocks unattested images at deploy) | SonarQube, OWASP Dependency-Check; Trivy and Artifact Analysis for image/dependency scanning |
| **DAST** (Dynamic App Security Test) | Running app vulnerability scan | Staging (post-deploy) | Soft gate (log, not block) | OWASP ZAP, Burp Suite; Cloud Armor for app-layer attacks |
| **Smoke** | Happy-path sanity check | Prod post-deploy | Soft gate (canary fails = rollback) | Cloud Deploy verification gate; simple GET /health → 200 |
| **Load/Performance** | Latency, throughput, capacity | Staging before prod promotion | Soft gate (alert, may block) | JMeter, Locust (e.g. distributed on GKE), k6; run against staging target |
| **UAT** (User Acceptance) | Business requirements | Staging (stakeholder sign-off) | Yes | Manual + Playwright E2E; Altostrat case study: metadata extraction verification |
| **Chaos / Resilience** | Fault injection (latency, errors) | Staging + periodic prod | Soft gate (observe, improve) | Gremlin, Chaos Mesh; Pod Disruption Budgets in GKE; → Ch15 |
| **Regression** | Old features still work | Cloud Build (after new code) | Yes | Automated test suite; run against previous version baseline |

**Pipeline orchestration:** Typically Cloud Build → Cloud Deploy staging gate (smoke + load tests) → approval → Cloud Deploy canary (verification gate auto-rollback) → full rollout.

### Infrastructure as Code on GCP

**Terraform**
- **State:** typically Cloud Storage bucket (backend "gcs") with locking (DynamoDB alternative not GCP).
- **Modules:** reusable `.tf` files grouped by feature (network, compute, database). Composition > copy-paste.
- **Workspaces:** separate state per environment (dev, staging, prod) in same backend.
- **Drift detection:** `terraform plan` shows reality vs desired (scheduled via Cloud Scheduler → Cloud Run functions / Cloud Build).
- **Import:** `terraform import google_compute_instance.foo projects/my-project/zones/us-central1-a/instances/old-vm` to manage legacy resources.
- **GCP provider:** `google` + `google-beta` (new features before GA).
- **Modules in practice:** Cloud Foundation Toolkit (CFT) fabric blueprints—ready-made Terraform modules for landing zones, VPC, IAM.

**Infrastructure Manager** (Infra Manager; managed Terraform service)
- Google-managed service that runs **Terraform (`.tf`/HCL) configurations** for you: deployments, state and locks managed by Google, previews, revisions, git/Cloud Storage sources, audit trail. It is GCP-resource focused.
- Recommended successor to Deployment Manager; the DM Convert tool helps migrate.

**Config Connector** (Kubernetes Resource Model)
- Kubernetes add-on whose custom resources (CRDs) represent Google Cloud resources (e.g. `SQLInstance`); controllers reconcile them continuously.
- For Kubernetes-native / GitOps teams; Terraform is preferred for multi-cloud and non-Kubernetes workflows.

**Deployment Manager** (deprecated; end of support)
- Python/Jinja/YAML templates, Google-only; Google announced deprecation with end of support around end of 2025 and required migration actions by 31 March 2026, pointing to Infrastructure Manager (Terraform) ⚠ verify exact dates at docs.cloud.google.com/deployment-manager/docs/deprecations.
- Do not choose it for new designs; expect it only as a distractor or as a "migrate to Terraform / Infrastructure Manager" answer.

**Ansible, Chef, Puppet:** less exam-relevant; mentioned for completeness. Use Terraform for infrastructure-at-scale.

### Service catalog & provisioning

**Terraform Modules** (self-service): publish to Terraform Registry or private repo; teams instantiate via `terraform init -upgrade && terraform apply`.

**Service Catalog / Private Catalog** (GCP-native)
- Curated catalog of pre-built solutions (VM templates, app stacks) in GCP console.
- Team members browse and click "Deploy"; solutions are packaged as Terraform configurations ⚠ verify current Service Catalog / Private Catalog status.
- Reduces misconfiguration; audit trail.

**Application Design Center / App Hub** ⚠ verify
- Inventory of services, dependencies, criticality (SLO/RTO/RPO).
- Governance: which teams own which services, compliance mappings.

### API management: comparison table

| Platform | Use Case | Monetisation | Rate Limiting | Analytics | Hybrid | Exam Weight |
|----------|----------|---|---|---|---|---|
| **Apigee** | Public APIs, B2B, monetisation, complex policies | ✅ (quotas, spike arrest, revenue sharing) | ✅ (quota per app, per user) | ✅ (rich analytics, flow debugging) | ✅ (hybrid for on-prem APIs) | High (policies, monetisation, analytics) |
| **API Gateway** | Managed gateway for serverless backends (Cloud Run, Cloud Run functions, App Engine), OpenAPI-defined, API-key/JWT auth | ❌ | Basic quotas (per API key, via OpenAPI extension) | Basic (Cloud Monitoring/Logging) | ❌ | Medium (lightweight, common) |
| **Cloud Endpoints** | OpenAPI or gRPC APIs fronted by an ESPv2/ESP proxy you deploy (Cloud Run, GKE, Compute Engine); auth/JWT | ❌ | Basic quotas via service config | Basic (Monitoring, Trace) | ❌ (proxy can front on-prem backends) | Low-Medium (gRPC focus) |
| **Cloud Run built-in** | Serverless function, light auth, native | ❌ | ✅ (via Cloud Armor, concurrency limits) | Via Cloud Logging | N/A | Medium (implicit for Cloud Run) |

**Apigee deep-dive** (high exam focus)
- **Proxies:** inbound request → policies (transform, auth, rate-limit) → backend.
- **Products:** bundle of APIs; assign quotas (e.g. 1000 requests/day), spike arrest (e.g. 100 req/sec burst).
- **Policies:** Assign Message, Rate Limit, Quota, JWT/OAuth, CORS, XPath/JSONPath extraction, traffic management.
- **Analytics:** request volume, error rates, latency by developer, API, product.
- **Hybrid:** run proxy on on-prem infrastructure; control plane stays on GCP; useful for regulated data (Altostrat on-prem archival).
- **Monetisation:** track usage per app/user, bill external partners.
- **As API→Agent bridge (new):** Model Context Protocol (MCP) servers published via Apigee as REST APIs; Agent Platform consumes them. Not deep-dive; mentioned for context.

### Programmatic interaction & APIs

**Cloud Shell** 
- **Terminal:** bash, gcloud, gsutil, bq (covered in detail).
- **Editor:** built-in web IDE (VS Code-like); edit `.tf`, YAML, scripts.

**Cloud Code** (IDE plugin)
- VS Code / JetBrains IntelliJ integration.
- Deploy to GKE / Cloud Run directly from editor.
- Debug containers in GKE.

**gcloud / gsutil / bq / gcloud storage**
- Exam de-emphasises flag syntax; focus on concepts: `gcloud compute instances create`, `gsutil cp` (now `gcloud storage cp`), `bq query`.
- Service Account auth: `gcloud auth activate-service-account --key-file=...`.

**Emulators** (local dev)
- **Bigtable, Spanner, Pub/Sub, Firestore, Datastore:** run locally in Docker.
- Speeds up dev-loop (no GCP network latency, free, instant reset).

**Client Libraries**
- language-specific SDKs (Python google-cloud-storage, Node.js @google-cloud/pubsub, etc.).
- **Application Default Credentials (ADC):** auto-detect auth (service account, user, compute metadata). No hardcoded keys.

**API best practices**
- **Exponential backoff + jitter:** 2^attempt * (random 0–1) seconds. Prevents thundering herd.
- **Idempotency:** design APIs to be safe if called twice (use request ID, idempotency key).
- **Pagination:** `pageSize=100, pageToken=...` to fetch large result sets.
- **Field masks:** only update/return requested fields (reduces bandwidth, updates only non-null fields).
- **Batching:** group multiple requests (e.g. Pub/Sub publish 100 messages in 1 gRPC call vs 100 calls).
- **Quotas:** per-API per-project (and rate limits per minute); monitor usage, request an increase if needed.
- **API keys vs OAuth vs Service Account:** API key for public APIs (mobile clients, public websites); OAuth for user-delegated access; Service Account for service-to-service.

### Troubleshooting & root-cause analysis (RCA)

**Logs → Traces → Errors → Changes**
1. **Cloud Logging:** aggregate logs from app, GKE, Cloud Run, VMs. Filter by label (env, service, error code).
2. **Cloud Trace:** end-to-end request latency; identify slow microservice hops.
3. **Error Reporting:** deduplicate errors, count, latest stack trace.
4. **Change correlation:** when did incident start? What deployed/changed before? Use audit logs + release history.
5. **Bisect releases:** binary search (deploy commit N-1, N-2, …) to find regression. Cloud Deploy rollback enables this fast.

**Canary + verification gate is RCA insurance:** catches regressions before affecting all users.

## Decision guides

**Choose deployment strategy:**
- **Recreate** ← stateless, non-critical, downtime acceptable
- **Rolling** ← steady-state, HPA handles traffic, zero downtime
- **Blue/Green** ← full confidence before cutover, fast rollback critical (PCI data, healthcare)
- **Canary** ← risky changes, need real user monitoring, auto-rollback desired
- **Feature Flags** ← decouple release from deployment, A/B testing

**CI/CD platform choice:**
- **Cloud Build** ← GCP-first, Artifact Registry native, Cloud Deploy integration
- **Jenkins** ← legacy, multi-cloud, heavy operations cost
- **GitLab CI** ← git-native, but multi-cloud requires self-hosted runners
- **GitHub Actions** ← tight GitHub integration, GCP via `google-github-actions/*` actions

**IaC choice:**
- **Terraform** ← multi-cloud, modules ecosystem, de-facto standard
- **Infrastructure Manager** ← Terraform without running your own state/runner, Google-managed, GCP resources, audit trail
- **Config Connector** ← GKE teams, Kubernetes-native workflow
- **Deployment Manager** ← avoid (deprecated, migrate to Infrastructure Manager / Terraform)

**API platform choice:**
- **Apigee** ← if monetisation, quotas, complex policies, or B2B partner APIs needed
- **API Gateway** ← simple REST, internal microservices, cost-conscious
- **Cloud Endpoints** ← gRPC preferred, low latency, auth-native
- **Cloud Run** ← serverless function, simple auth, no separate API platform needed

## Well-Architected Framework lens

| Pillar | In This Domain |
|--------|---|
| **Operational Excellence** | CI/CD automation reduces toil; monitoring deployment health; DORA metrics; runbooks in Git; post-mortems; shift-left testing (SAST in build, not after deploy). |
| **Security** | SAST/DAST gates; container image scanning (Artifact Analysis); Binary Authorization blocks unsigned images; service account least privilege per build step; audit logs for approvals; API quotas prevent DoS. |
| **Reliability** | Canary + verification gates catch regressions; blue/green for instant rollback; Terraform drift detection prevents config divergence; chaos testing before prod. |
| **Cost Optimization** | Terraform modules reduce duplicate infra; IaC version control avoids pet VMs; Cloud Deploy canary saves re-deploy cost; quotas prevent runaway API calls. |
| **Performance Optimization** | Cloud Deploy traffic splitting (% rollout) monitors real latency; tests (load, smoke) gate releases; field masks reduce API payload; batching lowers request count. |
| **Sustainability** | IaC enables ephemeral envs (spin up for testing, tear down); Terraform modules reduce resource sprawl. |

## AI/GenAI angle

- **AI-driven code generation:** Gemini Cloud Assist suggests tests, IaC modules, CI/CD pipelines. Exam: "use Cloud Assist to draft Terraform for VPC + GKE + monitoring."
- **AI in deployment:** Agent Platform Pipelines orchestrate ML training → model deployment; verify gate checks model accuracy before prod.
- **Testing AI:** custom DAST/SAST for LLM apps (prompt injection, data exfiltration detection); Sensitive Data Protection (DLP) scans prompts + API responses.
- **Monitoring AI:** Trace LLM latency end-to-end (query → embedding → vector DB lookup → generation → response). Alert on model drift (predicted distribution ≠ actual).
- **Governance:** audit logs show who approved ML model deploy; Artifact Registry stores model images alongside app containers.

## Case-study hooks

**Altostrat Media** (podcasts, NLP, on-prem archival)
- **CI/CD:** modernise on-prem ingestion pipeline → Cloud Build triggers on podcast upload; Dataflow processes (transcription, extraction); stores in Cloud Storage.
- **Deployment:** GKE services (recommender, chatbot) via Cloud Deploy canary (verify model accuracy on sample users).
- **IaC:** Terraform modules for GKE cluster, Pub/Sub topics, Dataflow jobs (reproducible across regions for resilience).
- **Testing:** SAST + Trivy for container scanning; DAST for chatbot (prompt injection, unsafe API calls); UAT: metadata extraction verification by subject-matter experts.

**Cymbal Retail** (catalog enrichment, conversational commerce)
- **CI/CD:** attribute/image generation pipelines → Cloud Build; approval gate before applying to catalog (human-in-the-loop, not automatic).
- **Deployment:** Cloud Run services (discovery, recommendation) with traffic splitting; canary detects latency regression.
- **IaC:** Terraform for multi-region setup (US + EU per GDPR); modules for secrets (API keys for supplier data), Cloud SQL, Memorystore.
- **Testing:** integration tests verify supplier data → enriched product (schema + latency); load test discovery API (expected traffic surge during campaign).
- **API management:** Apigee for partner integrations (quotas per partner, analytics per integration).

**EHR Healthcare** (HIPAA, HA, hybrid)
- **CI/CD:** on-prem + GCP: Cloud Build runs tests; Cloud Deploy to staging GKE, approval by compliance officer, then prod.
- **Deployment:** blue/green for patient-facing services (zero downtime critical); canary for new diagnostic features (verify accuracy on sample patients).
- **IaC:** Terraform with CMEK (Cloud KMS) for DB encryption; VPC-SC perimeter for HIPAA compliance.
- **Testing:** UAT with real patient scenarios (anonymized); load test EHR queries under seasonal flu surge.

**KnightMotives Automotive** (autonomous, security, EU data residency)
- **CI/CD:** CI/CD for in-vehicle AI models (inference on car); Cloud Build in EU region (GDPR); security gate (no telemetry exfiltration).
- **Deployment:** canary for ML models (shadow production telemetry, verify accuracy before pushing to vehicles).
- **IaC:** Terraform for EU-only infrastructure (DLP policies, Cloud KMS EU-HSM, Spanner EU region). Modules for dealer tools (API, UI).
- **Testing:** chaos engineering (simulate connectivity loss, GPS spoofing); DAST for API (common automotive attacks: door unlock, telemetry injection).

## Exam traps & keyword → answer

- **"Fastest rollback"** → Blue/Green (instant LB switch), not canary (gradual traffic revert).
- **"Zero downtime + cost-effective"** → Rolling update + HPA (not blue/green with 2x infrastructure).
- **"Monitor before affecting users"** → Canary (% traffic), not feature flag (same code path, harder to debug).
- **"Sensitive data in logs"** → Sensitive Data Protection (DLP) to mask PII/API keys in logs, not just IAM access control.
- **"Terraform state locked"** → Cloud Storage backend + locking prevents concurrent applies, not IAM permissions alone.
- **"Can't modify old container image"** → Use Dockerfile arg or env var override at runtime; rebuild + re-deploy from source, not edit existing image.
- **"API rate-limit to prevent overload"** → Apigee Quota + Spike Arrest policies, API Gateway/Endpoints quotas, Cloud Armor rate limiting at the load balancer, or Cloud Run concurrency/max-instances limits (not IAM policy).
- **"Audit trail for who deployed what"** → Cloud Build release history + Cloud Deploy approval logs (not git blame; git shows author, not approver).
- **"Test this feature on 10% users before full rollout"** → Cloud Run traffic split (%), not feature flag (flag is for same code path; traffic split is different revisions).
- **"SAST must block bad code"** → Binary Authorization + Artifact Analysis (image scanning) + Cloud Build policy gate, not just Sonarqube (which is advisory).
- **"Drift between Terraform desired state & actual GCP"** → `terraform plan` shows drift; fix with `terraform apply` or import legacy changes; not solved by logging alone.
- **"IaC modules for multi-team reuse"** → Terraform modules in registry or git repo, version pinned per team (not shared mutable state).

## Quick-reference cheat table

| Tool/Pattern | Purpose | When to Use | Constraints |
|---|---|---|---|
| **Cloud Build** | CI: build, test, SAST, push artifact | Every commit / PR | Private pools for VPC access/isolation; build timeout configurable (default 60 min, up to 24 h) ⚠ verify |
| **Cloud Deploy** | CD: approval gates, canary, rollback | Post-artifact, before prod | Targets: GKE, GKE Enterprise, Cloud Run, custom targets; uses Skaffold |
| **Artifact Registry** | Central artifact repo (containers, Helm, language packages); replaces Container Registry | Always (push from Cloud Build, pull in Cloud Deploy) | Private by default (IAM); VPC-SC supported |
| **Terraform + Cloud Storage** | IaC with state locking | All infra; modules for reuse | State file is sensitive (DB credentials, keys) |
| **Apigee** | API proxy + policies + monetisation | Public APIs, B2B, quotas/spike arrest | Hybrid mode for on-prem backends |
| **Cloud Endpoints + ESP** | Lightweight API (gRPC, low latency) | Internal microservices, gRPC preferred | Less rich policies than Apigee |
| **Canary (Cloud Deploy)** | Gradual rollout + verification + rollback | Risky changes, need real metrics | Requires app/platform support for % traffic split (GKE, Cloud Run) |
| **Blue/Green** | Instant switch + fast rollback | Full confidence, zero downtime critical | 2x infra cost during transition |
| **Feature Flags** | Decoupled release ≠ deploy | A/B test, gradual feature rollout | App-side burden (feature flag service) |
| **Cloud Shell** | gcloud / gsutil / bq CLI access | GCP admin tasks, scripting | Ephemeral VM; sessions end after an inactivity timeout; only the home directory persists ⚠ verify limits |
| **Terraform Drift Detection** | Detect reality ≠ desired (scheduled) | Prevent pet VMs, config drift | `terraform plan` read-only; `apply` is destructive |
| **Binary Authorization** | Approve only signed container images | Enforce image security policy | Requires container image signing in Cloud Build |
| **API Quotas** | Rate-limit per project / API | All APIs; prevent runaway costs & DoS | Monitor quota usage; request increase if needed |
| **Exponential Backoff + Jitter** | Retry transient failures | API client libs (auto-retry); custom code | Max retries, jitter factor; exam prefers "jitter" |

## Practice scenarios

**Scenario 1: Altostrat release regression caught late**

A new version of the podcast recommendation service deployed to prod at 9 AM. By 11 AM, Customer Support reports users seeing wrong recommendations (old vs new). You have 2 hours to know: was it the new code, the database, or the model? Which tool/strategy prevents this?
- A) Increase Cloud Run replicas and monitor CPU.
- B) Add logging to every function, then reanalyze yesterday's logs.
- C) Require sign-off from ML engineer before any deploy (manual gate).
- D) Deploy to canary (10% traffic), gate on recommendation accuracy metric, auto-rollback if accuracy drops >1%.

**Answer: D.** Canary with verification gate would catch accuracy regression before 11 AM. Option A ignores the actual problem. Option B is reactive (incident already happened). Option C is human latency (why not automate?).

---

**Scenario 2: Cymbal Retail multi-region Terraform**

You manage Terraform for Cymbal's catalog API across US (primary) and EU (GDPR). Currently, all `.tf` files in one directory; US and EU environments have slightly different configs (EU uses Cloud KMS EU-HSM, US uses standard Cloud KMS). Which structure minimizes duplication and enables version pinning per environment?
- A) Terraform modules (one for shared VPC, one for app, one for secrets); root modules per region reference these, pinned by version tag.
- B) Separate git branches per region; merge conflicts managed manually.
- C) Terraform workspaces in same directory; all state in Cloud Storage; separate Cloud KMS backend per workspace.
- D) Copy the whole configuration per region into separate Infrastructure Manager deployments with no shared modules.

**Answer: A.** Modules + root modules per region allow reuse (VPC module is shared) and encapsulation (secrets module can vary KMS per region) + version pinning. Option B doesn't solve duplication. Option C doesn't encapsulate per-region secrets. Option D duplicates configuration per region (no reuse or version pinning).

---

**Scenario 3: EHR patient-facing API Blue/Green cost**

EHR has a blue/green deployment for its patient portal (must be zero-downtime; 99.9% SLA). Blue environment: 50 GKE nodes, $50K/month. Current deployment cycle: 2 weeks, 1 release per cycle. Leadership asks: "Are we wasting money on blue/green?" How do you decide?
- A) Switch to rolling updates (cheaper); accept brief user session drops; add load balancer session affinity to minimize impact.
- B) Calculate the cost of downtime (patients unable to access records × support cost); if downtime cost > blue/green cost, keep blue/green.
- C) Use canary instead; lower infrastructure cost and faster feedback.
- D) Implement feature flags; deploy new code to blue, gate the feature, then toggle (no green environment needed).

**Answer: B.** Blue/green is justified by RTO cost (patient data access critical). Option A increases CFR (session drops = SLA miss). Option C doesn't match "zero-downtime required" (canary assumes user impact acceptable). Option D requires app refactoring + feature flag service (not immediate cost savings).

---

**Scenario 4: Cloud Build private pool for Altostrat on-prem**

Altostrat's podcast ingestion pipeline runs on-prem (sensitive data). They want to build container images in GCP (Artifact Registry) but keep build secrets (on-prem S3 creds, API keys) never leaving on-prem. Standard Cloud Build with gcloud would upload source to GCS and download artifact—both cross the internet. What's the best approach?
- A) Use Cloud Build standard pool; mount on-prem NFS as Cloud Storage FUSE mount; secrets stay local.
- B) Use Jenkins on-prem for builds; push artifact to GCS afterward.
- C) Cloud Build private pool peered with a VPC that is connected to the datacenter over HA VPN or Interconnect; builds reach on-prem resources over private IPs; artifact pushed to Artifact Registry.
- D) Store secrets in Secret Manager; Cloud Build standard pool pulls secrets during build (encrypted in transit via TLS).

**Answer: C.** Private pools are Google-managed workers peered to your VPC, so builds reach on-prem systems privately over the hybrid link (the default pool cannot reach private IPs); no public internet path, and VPC-SC can be applied. Option A is overcomplicated (FUSE adds latency). Option B is extra operational burden (maintain Jenkins). Option D keeps the default pool's public path to on-prem systems (and cannot reach private IPs), so it does not meet the requirement.

---

**Scenario 5: Apigee vs API Gateway for KnightMotives dealer APIs**

KnightMotives needs a dealer API (order status, inventory, service booking) with tiered usage: Tier 1 = 100 req/sec, Tier 2 = 500 req/sec. Dealers by location (EU, US, APAC) with quotas per dealer. Also: 3 partners integrating (payment, CRM, logistics); each needs separate rate limits and billing. Which API platform is the best fit?
- A) API Gateway + Cloud Armor (rate limit by IP).
- B) Cloud Endpoints + custom quotas service.
- C) Cloud Run with concurrency limits + custom middleware.
- D) Apigee (products/quotas per dealer, monetisation reporting, policy composition).

**Answer: D.** Apigee's products (bundle APIs), quotas (per app/user), analytics (by dealer/region), and revenue tracking are built for multi-tenant B2B. Option A doesn't track per-user quotas. Option B is lightweight but requires custom quotas service. Option C has no quotas/analytics native support.

---

**Scenario 6: SAST blocking vs advisory in CI/CD**

You're designing Cymbal's image scanning gate in Cloud Build. Trivy finds 10 "critical" vulnerabilities in a Python base image; Sonarqube finds 5 code defects. The code was written 3 years ago, already in prod, running fine. Do you block the build or make it advisory?
- A) Block on both (zero-risk policy).
- B) Advisory on both; let humans decide.
- C) Block on Trivy (supply chain security); advisory on Sonarqube (code quality). Patch base image in parallel.
- D) Publish metrics only; no blocks.

**Answer: C.** Image vulnerabilities (Trivy) are supply chain risk (must control—binary authorization blocks unsigned images anyway). Code defects (Sonarqube) are technical debt (important but not a release blocker for 3-year-old code working fine). Block image = prevent new risk. Advisory on code = improve over time.

---

**Scenario 7: Terraform module versioning for multi-team reuse**

Altostrat and Cymbal both need the same GKE cluster Terraform module (VPC, GKE, Workload Identity, monitoring). A shared team maintains the module in Git. How do multi-consuming teams safely version-pin?
- A) Shared team publishes versions (v1.0, v1.1, v2.0); each team pins in their Terraform: `source = "git::https://...?ref=v1.0"` or Terraform Registry.
- B) All teams use `ref=main` (always latest); shared team tests thoroughly before merging.
- C) Each team forks the module; maintains own copy (no coordination).
- D) Module in Cloud Storage `.tar.gz`; teams download and check in (Git doesn't track modules).

**Answer: A.** Git tags (semantic versioning) + pinning enable teams to upgrade at their pace (Altostrat on v1.0, Cymbal testing v2.0). Option B = all teams break together if bug introduced. Option C = duplicate maintenance. Option D = no version history, hard to audit.

---

**Scenario 8: Troubleshooting latency spike after release**

EHR's patient portal latency spiked 500ms avg (was 100ms) after 3 AM deploy. Incident commander: "Revert or drill down?" Logs show no errors. What's the fastest RCA path?
- A) Revert immediately (minimize patient impact).
- B) Increase Cloud Run memory (maybe OOM causing GC pause).
- C) Check Cloud Trace for which service hop added latency (identify bottleneck), then decide revert vs fix.
- D) Check Cloud Build logs to see what changed; grep code for database queries.

**Answer: C.** Cloud Trace pinpoints latency (e.g., "new database query takes 300ms on every request"). If it's code, revert is correct. If it's infra (e.g., new query scale exceeding cache), fix forward (add index, tune cache). Option A may be correct but blind (no RCA). Option B is guessing. Option D is slow (need to actually understand the code change).

## Sources

- cloud.google.com/build/docs (Cloud Build triggers, private pools, provenance)
- cloud.google.com/deploy/docs (Cloud Deploy delivery pipelines, targets, canary, verification)
- cloud.google.com/artifact-registry (Artifact Registry, private repos, image scanning)
- cloud.google.com/docs/apigee (Apigee proxies, policies, products, quotas, monetisation, hybrid)
- cloud.google.com/architecture/devops-culture-and-environment (DORA metrics, CI/CD patterns)
- cloud.google.com/docs/terraform (Terraform on GCP, state, modules, importing)
- cloud.google.com/docs/infrastructure-manager (Infrastructure Manager, Terraform-based)
- docs.cloud.google.com/deployment-manager/docs/deprecations (Deployment Manager deprecation)
- cloud.google.com/blog/products/devops-and-google-cloud/5-cloud-deploy-patterns (deployment strategies)
- cloud.google.com/docs/cloud-code (Cloud Code IDE integration)
- cloud.google.com/docs/shell (Cloud Shell editor, terminal)
- cloud.google.com/docs/emulators (Bigtable, Spanner, Pub/Sub, Firestore emulators)
- cloud.google.com/docs/reference/libraries (client libraries, ADC, best practices)
- cloud.google.com/blog/products/api-management (API management, API Gateway vs Apigee decision tree)
