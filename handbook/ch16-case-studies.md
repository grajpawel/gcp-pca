# Chapter 16 — Case Studies

## Exam objectives covered

From the official exam guide v6.1 (section 5, row 16):
- **The 4 case studies:** Altostrat Media, Cymbal Retail, EHR Healthcare, KnightMotives Automotive
- Case-study format: 2 per exam (random from the pool of 4), split-screen, 20–30 % of exam questions
- Renewal exam: 1 h, $100, 25 questions, 90–100 % from ONE gen-AI case study (Cymbal Retail or Altostrat Media)
- Scope: hard requirements, existing environment, constraints, risks, architecture decisions, service-by-service mapping, WAF pillar alignment, security/compliance specifics, AI components

---

## What the exam really tests

- **Read the exec statement first.** It contains the constraint keyword: "reliability," "cost," "speed," "compliance," "auditability." That keyword is often the exam-question filter.
- **Constraints > features.** A question asks about a service, but the answer depends on a hidden constraint (e.g., Cymbal must avoid call-centre costs → automate, not Cloud Contact Center AI alone).
- **Integration, not standalone.** No case study uses a single GCP service. Answers chain services: Pub/Sub → Dataflow → BigQuery; Document AI → Gemini → HITL review UI → Firestore.
- **Hybrid is real.** All 4 cases have on-prem or legacy dependencies. Questions test your ability to bridge them: VPN + VPC peering, Cloud SQL + insurer APIs, GKE Enterprise + on-prem Kubernetes.
- **AI is woven in, not bolted on.** EHR is a pre-AI case (it only asks for healthcare-trend insights and predictions), so any AI angle there is a design assumption (e.g., monitoring and explainability of a prediction model). Altostrat's "auditable and explainable AI" technical requirement means you must be able to justify model choices, log inferences and keep humans in the loop.
- **WAF pillar tradeoffs.** Every case-study question has a multi-pillar trade-off. Cost-optimisation often conflicts with performance or reliability; compliance often conflicts with agility. Spot VMs lower cost but raise unavailability risk.
- **Migration and modernisation run in parallel.** Cases describe phased rollouts, not big bangs. Questions test your understanding of risk, rollback, and validation checkpoints.

---

## Core concepts

### 1. How to read a case-study question (30-second skim routine)

**Split-screen exam UI:** Left side = case-study context (it stays available next to each case-study question). Right side = question + options. Still, know each case well beforehand (the four PDFs are public) so you only re-check details instead of reading from scratch, and memorise each exec statement's constraint keyword.

**30-second skim before studying the case:**
1. **Goal:** What does the business want? (e.g., "accelerate AI adoption," "reduce data-centre footprint," "enable dealers")
2. **Constraints:** Hard requirements? SLAs? Compliance? Cost ceiling? (e.g., "≥ 99.9 % availability," "HIPAA," "EU-only data residency")
3. **Existing environment:** What are they running now? (e.g., "GKE on-prem, MySQL, Prometheus, email alerts")
4. **Gotchas:** What will break if you're not careful? (e.g., Cymbal's "MySQL, SQL Server, Redis, MongoDB" = fragmented stack; EHR's "AD + insurer APIs" = federation + hybrid)
5. **Exec statement keyword:** The one or two words that filter 80 % of answers. (Altostrat: exec statement says "Reliability and cost management are our top priorities", plus the technical requirement "auditable and explainable AI"; Cymbal: "reduce call-centre and data-centre costs," "automate"; EHR: "99.9 %," HIPAA; KnightMotives: "monetise data," "security")

---

### 2. Altostrat Media

**Summary:**  
Altostrat is a media company producing podcasts, interviews, documentaries, and news. They want to accelerate gen-AI adoption: personalised recommendations, 24/7 NL chatbots, automated metadata extraction, inappropriate-content detection, and trend insights. They already run GKE (scalability/HA), Cloud Storage for the media library, BigQuery as the warehouse, Cloud Run functions (transcoding, metadata extraction, recommendations), Google Identity plus third-party IdPs, Cloud Monitoring plus open-source Prometheus with email alerts, and some legacy on-prem systems for ingestion and archival that are slated for migration. The executive statement's constraint is **"Reliability and cost management are our top priorities"**; the technical requirements add that AI systems must be **auditable and explainable**, so they must be able to justify why a model made a decision.

| Requirement | Design Decision | GCP Service(s) | WAF Pillar |
|---|---|---|---|
| Personalised recommendations at scale | Vector embeddings + retrieval-augmented generation (RAG) | Gemini (embedding models), Agent Search, BigQuery ML | Performance, Cost optimisation |
| 24/7 NL chatbot (support, discovery) | Conversational agent with grounding; fallback to support queue | Conversational AI / Gemini Enterprise, Agent Runtime, Cloud Pub/Sub (queuing to humans) | Reliability, Performance |
| Metadata extraction (NLP + vision) | Event-driven pipelines (Cloud Run functions already used today); Document AI for documents, multimodal Gemini for audio/video | Document AI, Gemini (multimodal), Cloud Run functions, Dataflow | Performance, Cost optimisation |
| Inappropriate content detection | Vision AI (label detection) + custom fine-tuned Gemini classifier | Vision API, Gemini, custom training on Agent Platform | Security, Reliability |
| Trend insights | BigQuery aggregate queries + Looker dashboards | BigQuery, Looker, Gemini-assisted BI | Performance, Cost optimisation |
| Scalable, HA hybrid infra (GKE + on-prem) | GKE Enterprise with multi-cluster fleet management; Prometheus → Managed Prometheus | GKE Enterprise, Managed Prometheus, Fleet, Cloud Interconnect, Cloud Router | Reliability, Operational excellence |
| Storage cost optimisation (media library) | Cloud Storage Autoclass + lifecycle policies (delete cold archives) | Cloud Storage (Autoclass, Lifecycle rules, retention policies) | Cost optimisation |
| Auditable & explainable AI decisions | Model cards, inference logging, HITL review workflows, audit logs (feature attributions via Explainable AI only for classic tabular/custom models, not for LLM output) | Vertex Model Registry (now Agent Platform Model Registry), BigQuery audit logs, Cloud Logging, custom HITL UI | Security, Compliance |

**Existing-environment gotchas:**
- **Legacy on-prem ingestion:** Altostrat's current ingestion and archival pipelines are on-prem; they are moving to GCP. This means designing a **secure, high-bandwidth hybrid link** (Dedicated Interconnect + Cloud VPN failover) and a phased cutover (test in parallel, validate checksums, only then decommission on-prem).
- **Prometheus to Managed Prometheus:** They use Cloud Monitoring plus open-source Prometheus, with alerts delivered mainly by email. Email-only alerting is weak; move to **Managed Prometheus + Cloud Monitoring alerting policies** (symptom-based, SLO burn rate, on-call notification channels).
- **Kubernetes both on-prem and in the cloud:** The technical requirement is scalable, performant Kubernetes in both places plus modernised CI/CD with central management. **GKE Enterprise** (fleet, multi-cluster management, Config Sync/policy, one control plane view) is the natural fit for the on-prem clusters.
- **Storage cost sprawl:** Media libraries grow fast; without lifecycle policies, old archives bloat costs. **Cloud Storage Autoclass** moves objects to colder classes automatically based on access and moves them back to Standard when read (no retrieval fees), which suits unpredictable access patterns; use lifecycle rules when access is predictable.

**Target architecture (ASCII):**

```
┌─────────────────────────────────────────────────────────────┐
│                      Altostrat Media                         │
├─────────────────────────────────────────────────────────────┤
│                                                               │
│  ON-PREM STUDIO          CLOUD (GCP)                         │
│  ┌──────────────┐        ┌──────────────────────────────┐   │
│  │ Ingestion    │        │ Cloud Storage (Autoclass)    │   │
│  │ (Audio, Video│───────→│ ├ Standard (new/active)      │   │
│  │ archival)    │        │ ├ Nearline/Coldline (idle)   │   │
│  └──────────────┘   IC   │ └ Archive (long-idle)        │   │
│         ▲           VPN  │                              │   │
│         │           ┌────┘                              │   │
│         │           │   ┌──────────────────────────────┐   │
│         └───────────┤   │ Cloud Run functions pipeline │   │
│  (VPN fallback)     │   │ ├ Video/audio transcoding    │   │
│                     └───│ ├ Metadata extraction        │   │
│                         │ ├ (Document AI, Gemini)      │   │
│  ┌─────────────────┐    │ └ Push → Pub/Sub → BigQuery │   │
│  │ Legacy Ingest   │    │                              │   │
│  │ (to be migrated)│────│                              │   │
│  └─────────────────┘    └──────────────────────────────┘   │
│                                                               │
│         ┌────────────────────────────────────────┐           │
│         │ BigQuery (Data Warehouse)               │           │
│         │ ├ Raw events (engagement, metadata)     │           │
│         │ ├ ML tables (embeddings, trends)        │           │
│         │ └ BI layer (Looker dashboards)          │           │
│         └────────────────────────────────────────┘           │
│                          ▲                                    │
│         ┌────────────────┴─────────────────┐                 │
│         │                                  │                 │
│    ┌────▼──────────┐            ┌──────────▼──────┐          │
│    │ Agent Platform│            │ Cloud Run        │          │
│    │ ├ Embeddings  │            │ (Conversational  │          │
│    │ ├ Gemini LLM  │            │  agents, HITL UI)│          │
│    │ └ Fine-tuning │            └──────────────────┘          │
│    │   (content    │                                         │
│    │   policy)     │             ┌──────────────────┐        │
│    └───────────────┘             │ Managed          │        │
│         ▲                         │ Prometheus +     │        │
│         │                         │ Alerting         │        │
│         │ (features)              │ (SLO burn-rate)  │        │
│         │                         └──────────────────┘        │
│  ┌──────┴────────────────────────────────────────────────┐  │
│  │  FRONTEND (Web/Mobile)                               │  │
│  │  ├ Personalised recommendations                       │  │
│  │  ├ Search (Agent Search)                              │  │
│  │  ├ 24/7 chat + NL discovery                           │  │
│  │  └ Content moderation feedback (HITL review)          │  │
│  └──────────────────────────────────────────────────────┘  │
│                                                               │
└─────────────────────────────────────────────────────────────┘
```

**Migration/phasing plan (illustrative design assumption, the case study gives no timeline):**
1. **Phase 1 (Months 1–2):** Set up hybrid connectivity (Dedicated Interconnect + Cloud VPN). Validate throughput and latency for concurrent ingestion.
2. **Phase 2 (Months 3–4):** Migrate ingestion pipelines (Dataflow jobs, Cloud Run functions). Run in parallel with on-prem; compare outputs (checksums, metadata).
3. **Phase 3 (Months 5–6):** Bring on-prem Kubernetes clusters under GKE Enterprise fleet management. Test multi-cluster fleet management. Cutover storage to Cloud Storage + Autoclass.
4. **Phase 4 (Months 7–9):** Deploy Gemini agents (recommendations, chatbot) and fine-tune content-moderation models. Set up HITL review workflows.
5. **Phase 5 (Months 10+):** Decommission on-prem; monitor Managed Prometheus + alerting SLOs.

**Risks & mitigations:**
- **Risk:** Content moderation model gives false positives (blocks good content). **Mitigation:** Use a fine-tuned Gemini classifier with HITL review; log all flagged content to BigQuery audit logs for manual review + feedback loop.
- **Risk:** Hybrid bandwidth saturated during ingestion spikes. **Mitigation:** Use Dataflow backpressure + autoscaling; reserve 30 % bandwidth headroom; failover to Cloud VPN if Interconnect fails.
- **Risk:** BigQuery costs spiral as data grows. **Mitigation:** Partition by ingestion date; cluster by content_id; use BI Engine for hot queries; archive old tables to Cloud Storage.
- **Risk:** Embeddings model update breaks retrieval (old embeddings become stale). **Mitigation:** Version embeddings; re-embed on schedule; build the new index/embeddings side by side (e.g., a new Vector Search index) and switch traffic only after retrieval-quality evaluation.

**Security & compliance:**
- **PII in metadata:** Transcripts may contain user PII (medical info, financial data). Use **Sensitive Data Protection** to scan and mask PII in BigQuery before exposing to models.
- **Model auditability:** Log all model inferences (input, embedding, output, latency, user feedback) to BigQuery. Use **Model Registry** to track versions and approval workflows.
- **Data residency:** If Altostrat has international users, ensure Cloud Storage and BigQuery are in the correct regions. Use **organisation policy** resource-location constraints to enforce.

**AI components:**
- **Gemini models:** Use a Flash-tier Gemini model for high-volume chat (cost-effective) and a Pro-tier model for complex reasoning (recommendations, trend analysis); check the current model list and IDs in Ch08. Consider fine-tuning on an internal content corpus.
- **Agent Builder (ADK):** Build custom agents for domain-specific tasks (playlist curation, show recommendations) using the Agent Development Kit.
- **Document AI:** For extracting structured metadata from show scripts, grant agreements, guest bios.
- **Vector Search / RAG Engine:** Embed show transcripts; retrieve relevant episodes for each query; ground Gemini responses in actual content.
- **HITL review UI:** Build a custom Cloud Run app for content moderators to review flagged items, provide feedback, retrain the classifier.

**Likely question types & worked examples:**

**Q1. Altostrat wants to detect inappropriate content in user-uploaded videos (violence, hate speech) as soon as they're uploaded. They need <2 second latency and <0.1 % false-positive rate. Which architecture should they use?**

A. Run a daily batch job that labels all videos with the Vision API or Video Intelligence API.  
B. Trigger an event-driven pipeline on upload (Cloud Storage event → Pub/Sub → Cloud Run) that sends the video to a multimodal Gemini model tuned on Altostrat's moderation guidelines, quarantines flagged videos, and logs decisions to BigQuery for HITL review.  
C. Use Cloud Armor rules in front of the upload endpoint.  
D. Use open-source Tesseract OCR on-device.

**Answer: B.** The requirements are "as soon as uploaded", Altostrat-specific policy, and auditability. **B** combines:
- Multimodal Gemini, fine-tuned/prompted with Altostrat's moderation guidelines (generic classifiers do not know the company's policy; the latency and false-positive targets must be validated by testing, not assumed)
- Cloud Run (serverless, auto-scales to upload spikes)
- Pub/Sub (decouples upload from moderation; absorbs spikes if moderation lags)
- BigQuery decision log + HITL review (humans correct false positives; feedback improves prompts/tuning and supports the "auditable AI" requirement)

Why others wrong:
- A: A daily batch cannot meet "as soon as uploaded"; pre-trained label/explicit-content detection is also generic rather than policy-specific.
- C: Cloud Armor is a WAF/DDoS service for HTTP traffic; it does not inspect video content.
- D: Tesseract is OCR for text; it cannot judge violence or hate speech in video, and it offers no model serving or feedback loop.

---

**Q2. Altostrat's recommendation engine uses BigQuery ML (BQML) to predict user engagement (watch time per show). Currently, retraining the model nightly takes 4 hours, blocking the recommendation refresh. What should they do?**

A. Switch to Spanner; retraining will be faster.  
B. Use AutoML on Agent Platform, which automatically re-tunes the model every hour.  
C. Decouple training from serving: keep serving the previous model version while the new one trains (scheduled/off-peak retraining), pre-compute feature aggregations (trending, user segments) with Dataflow/BigQuery, and rank with pre-trained embeddings plus those features.  
D. Switch to a commercial recommendation service (Firebase Predictions).

**Answer: C.** The problem is that serving is blocked by a long-running training job. **C** removes the coupling (the refresh no longer waits for retraining), moves repeatable feature engineering out of the training step, and lets the model retrain on a schedule that suits the business.

Why others wrong:
- A: Spanner is an operational database, not a recommendation engine; moving data there does not shorten model training.
- B: AutoML does not retrain "hourly" by itself and would not remove the coupling between training and refresh; it is a model-building service, not a scheduler.
- D: Firebase Predictions was a mobile-analytics feature that Google has shut down; it is not a content-discovery solution. (A managed alternative is the recommendations capability of Agent Search, formerly Vertex AI Search for media. ⚠ verify)

---

**Q3. Altostrat's on-prem ingestion pipeline uploads 10 TB of raw media files daily to Cloud Storage. They want to minimize Interconnect bandwidth costs while ensuring data is available within 4 hours of capture. Which storage strategy should they use?**

A. Cloud Storage Nearline (30-day minimum); compress on-prem before upload.  
B. Dedicated Interconnect (e.g., redundant 10 Gbps circuits) + Cloud Storage with Autoclass enabled on the bucket, so new objects land in Standard and idle objects move to colder classes automatically.  
C. Cloud Storage Standard initially; use Dataflow to sample/compress on upload; use lifecycle rules to move objects to Coldline after 1 day; enable object versioning for a 4-hour restore window.  
D. Use on-prem caching (NAS); upload only metadata to Cloud Storage; defer full media transfer to weekend batch.

**Answer: B.** The constraint is "available within 4 hours." Sizing check: 10 TB is about 80 Tbit; over a 4-hour window that is roughly 5.6 Gbps sustained, so a 10 Gbps circuit (ideally two for redundancy) is enough and 100 Gbps would be over-provisioned. Data ingress to Cloud Storage is not charged, so the cost lever is storage class, not transfer. **B** ensures:
- Standard class = immediate access for fresh content; objects are never "in cold storage" while being actively used
- Dedicated Interconnect = predictable throughput for sustained daily bulk transfer (see Ch03 for Interconnect vs VPN sizing)
- Autoclass = objects not accessed for 30 days move to Nearline, later to colder classes (terminal class configurable), and return to Standard when read, without manual lifecycle tuning
- Net result: on-prem → Cloud Storage Standard (recent, high access) → colder classes (old archives, rare access)

Why others wrong:
- A: Nearline has a 30-day minimum storage duration and retrieval charges, so putting hot, newly ingested files directly in Nearline is a poor cost fit; compressing on-prem only helps if lossless and does not address the cost driver.
- C: Sampling/compression is lossy; the requirement is availability of the *full* media file within 4 hours. Versioning is not a restore-time mechanism, and moving freshly ingested files to Coldline after 1 day would incur early-deletion and retrieval charges if they are still being used.
- D: Keeping media on-prem NAS defeats the migration goal; a weekend batch violates the 4-hour requirement.

---

### 3. Cymbal Retail

**Summary:**  
Cymbal Retail is a fast-growing online retailer with a huge multi-vertical catalog. They want to automate catalog enrichment (attributes, descriptions, images), enable conversational product discovery, and modernise their tech stack. Their existing infrastructure is a mix of on-prem and cloud: MySQL, SQL Server, Redis, MongoDB, Kubernetes clusters, legacy SFTP/ETL batch integrations, a custom web app querying relational databases, and an IVR plus call-centre agents who enter orders manually. The case lists manual, error-prone processes, data silos and hard-to-integrate systems as the problems. The constraint keywords are **"cost"** and **"automate"**: reduce call-centre and data-centre costs.

| Requirement | Design Decision | GCP Service(s) | WAF Pillar |
|---|---|---|---|
| Automate catalog enrichment (attributes, descriptions, images) | Batch pipeline: ingest supplier data → Document AI (extract) → Gemini (generate descriptions) → Imagen (generate/enhance images) → HITL review → Firestore | Document AI, Gemini, Imagen, Dataflow, Cloud Tasks, Firestore, Cloud Run | Cost optimisation, Automation |
| Conversational product discovery (agent + Search) | Conversational agent with grounding in product catalog; Agent Search (the case names Google Discovery AI) for semantic search | Conversational AI, Agent Search, Firestore, Agent Platform | Performance, Reliability |
| Data-silo consolidation (MySQL, SQL Server, Redis, MongoDB) | Lowest-risk path: MySQL → Cloud SQL for MySQL, SQL Server → Cloud SQL for SQL Server, Redis → Memorystore, MongoDB → Firestore (MongoDB compatibility) or a managed MongoDB service; re-platform to AlloyDB (PostgreSQL-compatible) or Spanner only where scale/consistency justify the rewrite; stream everything into BigQuery for analytics | Cloud SQL, AlloyDB, Spanner, Memorystore, Firestore, Database Migration Service, Datastream, BigQuery | Performance, Cost optimisation |
| Call-centre cost reduction | Automate order intake → Conversational Agent + order validation logic; hand escalations to human agents through the contact-centre platform (live-agent handoff) | Conversational AI (Customer Engagement Suite), Firestore (order state), Pub/Sub | Cost optimisation, Automation |
| Human-in-the-loop enrichment review | Build review UI in Cloud Run; moderators approve/reject/edit auto-generated descriptions and images; feedback retrains Gemini classifier | Cloud Run, Firestore, Gemini fine-tuning | Reliability, Compliance |
| Scalable Kubernetes cluster | Migrate to GKE on GCP; use Autopilot for reduced operational burden | GKE (Autopilot), GKE Autoscaler, Workload Identity | Operational excellence |
| Real-time inventory & order visibility | Pub/Sub (order events) → Dataflow (aggregate) → BigQuery (warehouse) + Firestore (real-time reads) | Pub/Sub, Dataflow, BigQuery, Firestore | Performance, Reliability |

**Existing-environment gotchas:**
- **Multi-DB stack:** MySQL, SQL Server, Redis and MongoDB (the case does not say which workload lives where). Data silos, hard to integrate. Migration must map each DB to the right GCP equivalent: Cloud SQL for MySQL / SQL Server for like-for-like moves (no application changes), Memorystore for Redis, Firestore (MongoDB compatibility) for document data. **AlloyDB is PostgreSQL-compatible, not MySQL-compatible**, and **Spanner is not a drop-in SQL Server replacement** (GoogleSQL/PostgreSQL dialect, different schema design, migrated with the Spanner migration tool rather than Database Migration Service). **Database Migration Service** supports MySQL, PostgreSQL and SQL Server to Cloud SQL (and PostgreSQL to AlloyDB).
- **Legacy SFTP/ETL batch integrations:** Suppliers deliver product data through SFTP/ETL batch jobs (frequency and failure visibility are not stated in the case; manual, error-prone processing is). A reasonable design: **Cloud Storage** (suppliers upload to a bucket; SFTP gateway if suppliers cannot change) + **Pub/Sub** (notification on new file) + **Cloud Run functions** (trigger parsing) + **Cloud Tasks** (rate-limited retry of downstream calls).
- **Monitoring tooling:** Existing monitoring is Grafana, Nagios and Elastic (self-managed). Consolidate on **Cloud Monitoring + Cloud Logging + alerting policies** to cut the operations burden.
- **Call-centre inefficiency:** An IVR plus agents who enter orders manually. A conversational agent with **Firestore** (real-time inventory reads) + **Gemini** (understanding customer intent) can handle standard orders end to end and hand only escalations to humans; this directly serves the "reduce call-centre costs" goal (no savings percentage is given in the case).

**Target architecture (ASCII):**

```
┌──────────────────────────────────────────────────────────────┐
│                       Cymbal Retail                           │
├──────────────────────────────────────────────────────────────┤
│                                                                │
│  SUPPLIER ECOSYSTEM        DATA PIPELINE       CUSTOMER APPS  │
│  ┌────────────────┐        ┌──────────────┐                  │
│  │ Supplier data  │        │ Cloud Storage│                  │
│  │ (CSV, JSON)    │───────→│ (staging)    │                  │
│  │ upload to      │   Events│             │                  │
│  │ Cloud Storage  │   via   │ Pub/Sub     │                  │
│  └────────────────┘   Cloud │ notifications                  │
│         ▲             Pub/Sub└──────┬──────┘                  │
│         │                          │                         │
│  ┌──────┴────────────────────────┐ │                         │
│  │ Document AI                   │ │                         │
│  │ (extract structured data from │ │                         │
│  │  supplier PDFs, images)       │ │                         │
│  └──────────────────────────────┘ │                         │
│                                    ▼                         │
│  ┌─────────────────────────────────────────────────────┐     │
│  │ Dataflow (batch + streaming)                        │     │
│  │ ├ Merge product data from suppliers                 │     │
│  │ ├ Call Gemini API to generate descriptions          │     │
│  │ ├ Call Imagen to enhance/generate product images    │     │
│  │ ├ Validate against catalog rules                    │     │
│  │ └ Output → Firestore (raw) + Cloud Tasks (review Q)│     │
│  └────────────┬────────────────────────────────────────┘     │
│               │                                              │
│         ┌─────▼──────────────────────────────────────────┐   │
│         │ Cloud Tasks (HITL review queue)                │   │
│         │ ├ Each task = auto-generated product           │   │
│         │ │   (description, images, attributes)          │   │
│         │ └ Send to review workers (Cloud Run UI)        │   │
│         └─────┬──────────────────────────────────────────┘   │
│               │                                              │
│               ▼                                              │
│         ┌──────────────────────────────┐                    │
│         │ Cloud Run (HITL Review UI)    │                    │
│         │ Moderators:                  │                    │
│         │ - Approve / edit descriptions│                    │
│         │ - Accept / reject images     │                    │
│         │ - Mark attributes correct    │                    │
│         │ Feedback → Firestore         │                    │
│         └──────────────────────────────┘                    │
│                 ▲                                            │
│                 │ Feedback (approve/reject)                 │
│                 │ ↓                                          │
│         ┌──────────────────────────────┐                    │
│         │ Firestore (Product Catalog)   │                    │
│         │ ├ Products (denormalized)     │                    │
│         │ ├ Descriptions (final, after  │                    │
│         │ │  human review)              │                    │
│         │ ├ Images (asset URLs in GCS)  │                    │
│         │ └ Real-time inventory read    │                    │
│         └──────────────────────────────┘                    │
│                 ▲                                            │
│       ┌─────────┴──────────────────────────────┐            │
│       │                                        │            │
│   ┌───▼────────────────┐          ┌────────────▼──────┐    │
│   │ Agent Search       │          │ Conversational     │    │
│   │ (Semantic product  │          │ Agent (Gemini +   │    │
│   │  discovery)        │          │ function calling) │    │
│   └───┬────────────────┘          └────────────┬──────┘    │
│       │                                        │            │
│       │ Product details              Order events (Pub/Sub) │
│       │ (text, embeddings)                    │            │
│       │                                  ┌────▼──────────┐ │
│       │                                  │ Pub/Sub       │ │
│       │                                  │ (order queue) │ │
│       │                                  └────┬──────────┘ │
│       └─────────────────────────────────────┬─┘            │
│                                             │              │
│                 ┌───────────────────────────▼───────┐      │
│                 │ Dataflow (order processing stream)│      │
│                 │ ├ Validate order against inventory│      │
│                 │ ├ Reserve items                   │      │
│                 │ └ Aggregate order metrics         │      │
│                 └───────────────┬───────────────────┘      │
│                                 │                          │
│                         ┌───────▼────────┐                │
│                         │ BigQuery       │                │
│                         │ (order history,│                │
│                         │  analytics)    │                │
│                         └───────────────┘                │
│                                                            │
│  ┌───────────────────────────────────────────────────┐    │
│  │ DATABASES (Consolidated)                          │    │
│  │                                                   │    │
│  │ Cloud SQL for MySQL (user accounts, carts)       │    │
│  │ Cloud SQL for SQL Server / Spanner (orders)      │    │
│  │ Memorystore Redis (sessions, rate limits)        │    │
│  │ Firestore (product catalog, real-time sync)      │    │
│  └───────────────────────────────────────────────────┘    │
│                                                            │
│  ┌────────────────────────────────────────────────────┐   │
│  │ CUSTOMER-FACING (Web + Mobile)                    │   │
│  │                                                   │   │
│  │ ├ Product search (Agent Search)                   │   │
│  │ ├ Conversational shopping (Conversational Agent)  │   │
│  │ ├ Order placement (Cloud Run API)                 │   │
│  │ └ Order tracking (Firestore real-time)            │   │
│  └────────────────────────────────────────────────────┘   │
│                                                            │
└──────────────────────────────────────────────────────────┘
```

**Migration/phasing plan (illustrative design assumption, the case study gives no timeline or volumes):**
1. **Phase 1 (Months 1–2):** Set up Pub/Sub, Dataflow, Document AI pipelines on a sample of 1000 products. Validate enrichment quality. Measure latency, cost.
2. **Phase 2 (Months 3–4):** Build HITL review UI (Cloud Run + Firestore). Enroll 5–10 product managers. Collect feedback; fine-tune Gemini and Imagen prompts.
3. **Phase 3 (Months 5–6):** Run enrichment on the full catalog. Monitor costs (Dataflow, API calls). Set up feedback loop to retrain Gemini classifier.
4. **Phase 4 (Months 7–8):** Migrate databases: MySQL → Cloud SQL for MySQL, SQL Server → Cloud SQL for SQL Server (Spanner/AlloyDB only for workloads that justify a re-platform), Redis → Memorystore, MongoDB → Firestore or a managed MongoDB service. Validate queries; test failover.
5. **Phase 5 (Months 9–10):** Deploy conversational agent + Agent Search. A/B test against legacy search. Monitor conversion rate, call-centre volume.
6. **Phase 6 (Months 11–12):** Decommission legacy SFTP/ETL and the manual order-entry flow, and shrink the on-prem footprint. Measure cost savings against the baseline.

**Risks & mitigations:**
- **Risk:** Gemini generates offensive or inaccurate product descriptions. **Mitigation:** Every auto-generated description requires human review before publication. Log all reviews to BigQuery; track true-positive and false-positive rates; retrain.
- **Risk:** Imagen generates images with copyright violations (e.g., unlicensed background textures). **Mitigation:** Use only Gemini + Imagen for non-photorealistic images (graphics, icons); for photo-realistic images, use Imagen only on Cymbal-owned backgrounds/props.
- **Risk:** Conversational agent fails to understand a customer's intent (e.g., "show me dresses under $50 with pockets"), routes to human agent, defeating the automation goal. **Mitigation:** Log all unresolved intents; cluster similar failures; add function calls (e.g., `get_products(category="dresses", max_price=50, has_pockets=True)`) to Gemini. Monitor agent success rate; retrain on top-N failures.
- **Risk:** A re-platform to Spanner/AlloyDB breaks legacy queries (e.g., T-SQL-specific syntax, stored procedures). **Mitigation:** Prefer like-for-like Cloud SQL moves first; for re-platforms use the migration/assessment tooling (Spanner migration tool, Database Migration Service) plus application-level testing; run parallel validation (old DB vs new DB for a defined period); maintain a query compatibility matrix.
- **Risk:** Real-time inventory sync (Firestore) lags; customer orders items that are out of stock. **Mitigation:** Add a "last_inventory_check_time" timestamp to Firestore; the order-placement function (deterministic code, not the LLM) rejects or re-validates inventory older than a threshold against the system of record.

**Security & compliance:**
- **PII in customer data:** Call-centre conversations, order history contain names, addresses, payment methods. Use **Sensitive Data Protection** to identify and mask PII in BigQuery before analyzing or exposing to models.
- **Payment card data (PCI DSS):** A reasonable design assumption (the case only asks for data security and compliance for customer data and agent interactions): keep card data out of scope by using a PCI-compliant payment provider (hosted payment fields/tokenisation) so only tokens are stored in Firestore/Cloud SQL. **Apigee** can front and secure the order APIs but does not itself tokenise card data. Keep card numbers out of conversational-agent transcripts (Sensitive Data Protection redaction).
- **Customer privacy:** Conversational agent reads customer purchase history to make recommendations. Obtain explicit opt-in; log all access to BigQuery; audit for unauthorised access via **Audit Logs**.
- **Supplier data confidentiality:** Supplier cost data (e.g., wholesale price) is confidential. Ensure only procurement team can read it via **IAM roles** and **VPC Service Controls** (perimeters).

**AI components:**
- **Gemini:** Use a Flash-tier Gemini model for batch product description generation (cheap, fast); a Pro-tier model only where deeper reasoning is needed (check current model names in Ch08).
- **Imagen / Gemini image models:** Use Imagen (check the current version) or Gemini native image generation/editing for product image enhancement and generation (alternate color variants, background change, text overlays).
- **Document AI:** Extract structured data from supplier documents (PDF catalogs, scanned images).
- **Agent Search:** Managed semantic product search over the catalog (the case names Google Discovery AI for product discovery); use Vector Search/RAG Engine only if you need a custom retrieval stack.
- **Conversational Agent:** Accepts customer intent ("Find dresses under $50"), calls Agent Search to retrieve candidates, uses Gemini to rank and explain recommendations, interfaces with Firestore to check inventory and finalise orders.

**Likely question types & worked examples:**

**Q1. Cymbal uses Imagen to auto-generate product images (e.g., clothing in different colors). However, they notice the generated images sometimes show copyrighted background elements (e.g., brand logos, distinctive interiors). They want 99 % of generated images to be legally safe to publish. What should they do?**

A. Use Imagen with a prompt instruction: "no copyrighted materials." This is sufficient.  
B. Post-process every generated image with Vision API; detect logos/text; flag for manual review if confidence > 0.5.  
C. Use Imagen only on Cymbal-owned assets (e.g., white-box studio photos); hire professional photographers for all other images.  
D. Use Imagen to generate images only for non-clothing categories (e.g., furniture, toys); use stock-photo APIs (Unsplash, Pexels) for clothing.

**Answer: B.** The constraint is "99 % legally safe." Imagen is probabilistic; prompts alone don't guarantee copyright compliance. **B** uses Vision API (trademark, logo detection) as a gate:
- Generate image with Imagen
- Check with Vision API for logos, branded content
- Flag if confidence > 0.5 for human review
- Publish only if human approves or confidence < 0.1

This keeps automation high (publish 90 %) while catching risky images (10 % review queue).

Why others wrong:
- A: Prompts are hints, not guarantees; Imagen can hallucinate logos even if you say "no logos."
- C: Hiring professional photographers defeats the cost-reduction goal (Cymbal wants to automate enrichment, not hire photographers).
- D: Too restrictive; Imagen works well for graphics, backgrounds, icons; the issue is only photorealistic images with branded elements.

---

**Q2. Cymbal's conversational agent sometimes fails to understand customer queries (e.g., "pocketed dresses under $50"). They want to measure the agent's success rate, and improve it monthly. What should they do?**

A. Log all customer feedback (thumbs up/down) after each conversation to BigQuery; calculate [approved orders / total orders] weekly.  
B. Measure "false positives" as [orders that were not completed / total orders]; use BigQuery ML to predict incomplete orders; add features to the agent.  
C. Log all agent responses (query, intent, function call, result) to BigQuery; cluster unresolved intents (timeouts, no matches); re-train Gemini on top-N failures; measure [resolved / total] weekly.  
D. Rely only on the platform's built-in conversation analytics dashboard; it tracks conversation success out of the box.

**Answer: C.** The goal is iterative improvement. **C** provides:
- **Logging:** All agent decisions (query, intent, function calls, results) → BigQuery
- **Analysis:** Cluster failures (e.g., "pockets" is a common missing feature), identify top-10 pain points
- **Improvement:** Add or extend tool/function definitions, prompt instructions and few-shot examples for "pockets" and other missing attributes; fine-tune only if prompt/tool changes are not enough (you do not "re-train" Gemini for every failure)
- **Measurement:** Weekly [resolved / total] tells you if changes helped

Why others wrong:
- A: Customer thumbs-up is binary feedback; doesn't tell you *why* the agent failed (intent recognition? function call? result ranking?).
- B: "Incomplete orders" is an outcome metric, not diagnostic; doesn't tell you which specific intents are hard.
- D: Built-in analytics (Conversational Agents, formerly Dialogflow CX) show conversation-level outcomes, but do not by themselves give you the per-tool-call failure clusters and business-outcome joins needed to prioritise fixes; use them alongside your own BigQuery logging. (Dialogflow CX is not deprecated; it was renamed Conversational Agents.)

---

**Q3. Cymbal decides to move its order database from SQL Server to Spanner and runs a large daily analytics scan over order history on the same database. Which statement is the right design guidance?**

A. Spanner read-write transactions take locks, so run the long scan inside a read-write transaction for consistency and cache results in Memorystore.  
B. Run the scan as a read-only transaction (or a stale read with a timestamp bound such as exact or maximum staleness), optionally with Data Boost for large scans; read-only transactions take no locks and do not block order writes.  
C. Spanner does not support transactions. Use Firestore for transactional consistency.  
D. Spanner's SQL is SQL Server T-SQL compatible, so no query or schema rewrite is needed.

**Answer: B.** Spanner is a re-platform from SQL Server, not a drop-in replacement. **B** is the right approach:
- Spanner offers externally consistent (serializable) read-write transactions, which use locking
- Read-only transactions and stale reads see a consistent snapshot at a timestamp without taking locks, so analytics do not contend with concurrent order inserts/updates
- Data Boost runs large scans on independent compute, without affecting the provisioned instance
- (`AS OF SYSTEM TIME` is CockroachDB syntax, not Spanner; Spanner uses timestamp bounds in the API or `SET READ_ONLY_STALENESS`.)

Why others wrong:
- A: A long scan in a read-write transaction would hold locks and contend with order writes; this is exactly what to avoid.
- C: Spanner fully supports ACID transactions, including across regions.
- D: Spanner uses GoogleSQL or the PostgreSQL dialect, not T-SQL; queries, stored procedures and schema design (e.g., primary-key choice to avoid hotspots) need rework. This is why like-for-like moves go to Cloud SQL for SQL Server.

---

### 4. EHR Healthcare

**Summary:**  
EHR Healthcare is a SaaS EHR platform serving multinational medical offices, hospitals, and insurers, with exponential growth. They are replacing colocation facilities (one data-centre lease is expiring) with Google Cloud. They have containerised web apps on Kubernetes, MySQL/SQL Server/Redis/MongoDB databases, and legacy file- and API-based insurer integrations that stay on-prem for now. Users are in Microsoft Active Directory, and monitoring is open-source with email alerts that are often ignored. The constraint keywords are **"≥ 99.9 % availability"** for customer-facing systems and **"regulatory compliance (HIPAA)"**. Note: the case does not mention GDPR; EU residency below is a design assumption for a multinational customer base.

| Requirement | Design Decision | GCP Service(s) | WAF Pillar |
|---|---|---|---|
| ≥ 99.9 % availability (customer-facing) | Regional (multi-zone) GKE clusters behind a global external Application Load Balancer; add a second region only if the SLO or DR targets require it; HA databases (Cloud SQL regional HA, or Spanner / AlloyDB HA where re-platformed) | GKE (Autopilot or Standard, regional), Cloud Load Balancing, Cloud SQL (HA), Spanner / AlloyDB | Reliability |
| HIPAA compliance | Sign the BAA and use only HIPAA-covered services; encryption at rest is on by default (add Cloud KMS CMEK if policy requires key control), encryption in transit (TLS), audit logs (Cloud Audit Logs), least-privilege IAM, Assured Workloads where a compliance regime is enforced | Cloud KMS, Cloud Audit Logs, IAM, Sensitive Data Protection, Assured Workloads | Security, Compliance |
| Hybrid connectivity to legacy insurer integrations that remain on-prem | Dedicated Interconnect (redundant) + HA VPN as backup; secure API gateway (Apigee with mTLS, rate limiting) in front of the APIs | Dedicated Interconnect, Cloud VPN, Apigee, Cloud Armor | Security, Reliability |
| Centralised monitoring & alerting | Cloud Monitoring (custom metrics for SLO burn rate), Cloud Logging (all audit logs), Managed Prometheus, alerting policies (not email) | Cloud Monitoring, Managed Prometheus, Cloud Logging, Alerting policies | Operational excellence |
| Federation with Microsoft AD (workforce users) | Provision AD users/groups into Cloud Identity with Google Cloud Directory Sync (GCDS) and use SAML SSO against AD FS / Entra ID; Workforce Identity Federation is the option when users should not be synced into Cloud Identity | Cloud Identity, GCDS, SAML SSO, Workforce Identity Federation | Security |
| Consolidate RDBMS operations (MySQL, SQL Server) | Cloud SQL (MySQL / SQL Server) for like-for-like managed moves; AlloyDB (PostgreSQL-compatible) or Spanner only if re-platforming for scale or global consistency | Cloud SQL, AlloyDB, Spanner, Cloud SQL Auth Proxy | Reliability, Performance |
| Data residency (design assumption for multinational clients) | Use regional resources (US, EU, etc.) for patient data; organisation policy to enforce location | Cloud Storage, BigQuery, organisation policy (resource location constraint) | Compliance, Security |

**Existing-environment gotchas:**
- **Microsoft AD integration:** EHR users are in Microsoft Active Directory. Moving to GCP means federating identity: **Google Cloud Directory Sync (GCDS)** to provision users/groups into **Cloud Identity** plus **SAML SSO** to the existing IdP (AD FS / Entra ID), or **Workforce Identity Federation** to avoid syncing. (Workload Identity Federation is for workloads/service accounts, not for people.) Validate the authentication chain end-to-end.
- **Insurer integrations:** The case describes legacy file- and API-based insurer integrations that stay on-prem for now. EHR must maintain **hybrid connectivity** (Dedicated Interconnect, with HA VPN as backup) and can put a **secure API gateway** (Apigee with mTLS, rate limiting) in front of the APIs. The link's latency and reliability directly affect onboarding of new insurers and the 99.9 % target.
- **Email alerts are often ignored:** The case states monitoring is open-source with email alerts that are often ignored. Move to **Cloud Monitoring + alerting policies** with on-call channels (e.g., PagerDuty, SMS via the mobile app, webhook, Pub/Sub) and symptom/SLO-based alerts.
- **Multi-DB operational burden:** EHR runs MySQL, SQL Server, Redis, MongoDB. Each requires different backup/restore procedures, expertise, troubleshooting. **Cloud SQL** (MySQL, SQL Server) is the lowest-friction managed target; **AlloyDB** (PostgreSQL-compatible) or **Spanner** require a re-platform; **Memorystore** (Redis) handles caching; **Firestore** (with MongoDB compatibility) can take document data. Migration is complex; a phased approach is mandatory.

**Target architecture (ASCII):**

```
┌──────────────────────────────────────────────────────────────────┐
│                    EHR Healthcare (SaaS)                          │
├──────────────────────────────────────────────────────────────────┤
│                                                                    │
│  ON-PREM (EHR legacy)     HYBRID LINK        GCP (Multi-Region)   │
│  ┌────────────────────┐                      ┌──────────────────┐ │
│  │ Insurer file/API   │  Dedicated Interconnect + VPN Failover │ │
│  │ integrations,      │─────────────────────→│ Apigee (API      │ │
│  │ eligibility checks)│  (Low latency,       │  gateway)        │ │
│  │                    │   99.9 % uptime)     │ ├ Rate limiting  │ │
│  └────────────────────┘                      │ ├ mTLS auth      │ │
│         ▲                                     │ ├ Request log    │ │
│         │                                     │ └ Error handling │ │
│  ┌──────┴─────────────────────────────────→│                  │ │
│  │   Insurer Integration Service            │ Cloud Armor      │ │
│  │   (batches claims, eligibility)          │ (DDoS, WAF)      │ │
│  └──────────────────────────────────────→│                  │ │
│                                             └──────────────────┘ │
│                                                    ↓              │
│  ┌──────────────────────────────────────────────────────────────┐│
│  │ Global HTTP(S) Load Balancer (multi-region failover)         ││
│  │ ├ Route to closest region (us-central1, europe-west1)        ││
│  │ ├ Backend health checks (per-region failover)                ││
│  │ └ SSL termination                                            ││
│  └────┬────────────────┬────────────────────────────────────────┘│
│       │ US REGION      │ EU REGION                              │
│       │ (us-central1)  │ (europe-west1)                         │
│  ┌────▼─────────────┐  ┌────────────────────────────┐           │
│  │ GKE Autopilot    │  │ GKE Autopilot              │           │
│  │ (regional,       │  │ (regional, multi-zone)     │           │
│  │  multi-zone)     │  │                            │           │
│  │ ├ Patient API    │  │ ├ Patient API (replica)    │           │
│  │ ├ Clinic mgmt    │  │ ├ Clinic mgmt (replica)    │           │
│  │ ├ Billing        │  │ ├ Billing (replica)        │           │
│  │ └ Workload ID    │  │ └ Workload ID (replicas)   │           │
│  │  (no SA keys)    │  │  (no SA keys)              │           │
│  └────┬─────────────┘  └────┬───────────────────────┘           │
│       │                     │                                   │
│       ├─────────────────────┼───────────────────────────┐       │
│       │                     │                           │       │
│  ┌────▼──────────────────────────────────────────┐     │       │
│  │ Cloud SQL (MySQL/SQL Server) or AlloyDB      │     │       │
│  │ ├ Primary (us-central1), HA across zones      │     │       │
│  │ │  ├ Patient records, clinic data             │     │       │
│  │ │  ├ User accounts, sessions                  │     │       │
│  │ │  └ Audit trail                              │     │       │
│  │ ├ Cross-region replica (europe-west1; DR/latency) │     │       │
│  │ └ CMEK encryption at rest                     │     │       │
│  └────┬──────────────────────────────────────────┘     │       │
│       │                                                │       │
│  ┌────▼──────────────────────────────────────────┐     │       │
│  │ Spanner (geo-distributed, claims processing) │     │       │
│  │ ├ Claims (immutable, cross-region ACID)       │     │       │
│  │ ├ Eligibility checks (real-time, global      │     │       │
│  │ │  consistency needed)                        │     │       │
│  │ └ CMEK encryption                            │     │       │
│  └─────────────────────────────────────────────┘     │       │
│                                                       │       │
│  ┌────▼──────────────────────────────────────────┐     │       │
│  │ Firestore (semi-structured docs; documents) │     │       │
│  │ ├ Patient documents (scans, notes)           │     │       │
│  │ ├ Real-time sync to multiple regions         │     │       │
│  │ └ CMEK encryption                            │     │       │
│  └────┬──────────────────────────────────────────┘     │       │
│       │                                                │       │
│  ┌────▼──────────────────────────────────────────┐     │       │
│  │ Memorystore (Redis; sessions, caching)       │     │       │
│  │ ├ Session tokens (TTL = 24h)                  │     │       │
│  │ ├ Real-time cache (clinic schedules)          │     │       │
│  │ └ High availability (HA mode)                 │     │       │
│  └────┬──────────────────────────────────────────┘     │       │
│       │                                                │       │
│       └─────────────────────────────────────────────┘       │
│                                                            │
│  ┌───────────────────────────────────────────────────┐     │
│  │ OBSERVABILITY & COMPLIANCE                        │     │
│  │                                                   │     │
│  │ Cloud Monitoring (custom SLO metrics)             │     │
│  │  ├ Patient API latency (target: p99 < 100ms)      │     │
│  │  ├ Clinic search latency (target: p99 < 300ms)    │     │
│  │  └ Claim processing latency (target: 99.9 % < 5s)│     │
│  │                                                   │     │
│  │ Cloud Logging (audit trail)                       │     │
│  │  ├ Who accessed patient data, when, why           │     │
│  │  ├ All API calls (via Apigee logging)             │     │
│  │  └ Access attempt denials (failed auth)           │     │
│  │                                                   │     │
│  │ Alerting Policies (not email; critical only)      │     │
│  │  ├ SLO burn rate > 2 % / 24h                       │     │
│  │  ├ Error rate > 0.1 %                             │     │
│  │  └ Hybrid link latency > 50 ms                     │     │
│  │                                                   │     │
│  │ Cloud Audit Logs (supports HIPAA audit controls)  │     │
│  │  ├ Admin Activity logs (on; 400 days default)      │     │
│  │  ├ Data Access logs (enable for PHI stores; set    │     │
│  │  │  retention by policy, e.g. 365+ days)          │     │
│  │  └ System Event logs (Google-initiated changes)   │     │
│  │                                                   │     │
│  │ Sensitive Data Protection (DLP)                   │     │
│  │  ├ Scan BigQuery datasets for PHI                 │     │
│  │  ├ Mask SSN, MRN, birth date in logs              │     │
│  │  └ Alert on anomalous access patterns             │     │
│  └───────────────────────────────────────────────────┘     │
│                                                            │
│  ┌───────────────────────────────────────────────────┐     │
│  │ IDENTITY (Workforce Federation)                   │     │
│  │                                                   │     │
│  │ Cloud Identity (SAML SSO to AD FS / Entra ID)     │     │
│  │ ├ GCDS: sync AD users/groups → Cloud Identity     │     │
│  │ ├ Enforce MFA (TOTP, WebAuthn)                    │     │
│  │ └ Short session length (policy-defined)           │     │
│  │                                                   │     │
│  │ Workload Identity (service account auth)          │     │
│  │ ├ GKE pods use Workload Identity (no SA keys)     │     │
│  │ ├ Temporary credentials (short-lived)             │     │
│  │ └ Audit trail (who accessed what, when)           │     │
│  └───────────────────────────────────────────────────┘     │
│                                                            │
└──────────────────────────────────────────────────────────────────┘
```

**Migration/phasing plan (illustrative design assumption, the case study gives no timeline):**
1. **Phase 1 (Months 1–2):** Set up the GCP landing zone and regions; set up Dedicated Interconnect + HA VPN back to the on-prem/colocation environment that hosts the legacy insurer integrations. Validate latency and uptime.
2. **Phase 2 (Months 3–4):** Set up Cloud Identity with GCDS + SAML SSO to AD; test the SSO flow (on-prem AD → Cloud Identity → console/GKE access). Validate MFA and session policy.
3. **Phase 3 (Months 5–6):** Migrate databases (MySQL → Cloud SQL for MySQL, SQL Server → Cloud SQL for SQL Server via Database Migration Service; Redis → Memorystore; MongoDB → Firestore or a managed MongoDB service). Run in parallel with on-prem for validation.
4. **Phase 4 (Months 7–8):** Migrate applications (containerised workloads) to GKE. Test failover (zone loss; region loss if a second region is deployed).
5. **Phase 5 (Months 9–10):** Set up Apigee in front of the insurer-facing APIs; keep the legacy file/API insurer integrations on-prem behind the hybrid link until they can be retired. Test error handling and fallback.
6. **Phase 6 (Months 11–12):** Set up Cloud Monitoring + Alerting (replace email alerts with PagerDuty); set up Cloud Audit Logs (HIPAA compliance). Decommission colocation.

**Risks & mitigations:**
- **Risk:** Hybrid link fails; clinic cannot reach insurer eligibility API; patient check-in blocked. **Mitigation:** Set up Cloud VPN failover (if Interconnect fails, reroute via VPN). Implement client-side caching of eligibility checks; allow check-in to proceed with 24-hour-old data if API is down.
- **Risk:** Failover is slow; patient API becomes unavailable for a noticeable period. **Mitigation:** Use the global external Application Load Balancer with tuned health checks (check interval, timeout, unhealthy threshold determine detection time; failover is not instantaneous) and capacity in each region sized to absorb a failed region's traffic. Cloud CDN is not a failover mechanism.
- **Risk:** Audit log retention is insufficient; GCP deletes logs before clinic can retrieve them for compliance audit. **Mitigation:** Configure Cloud Logging to export audit logs to Cloud Storage (Bucket Lock for immutability) or BigQuery for long-term retention (minimum 365 days for PHI).
- **Risk:** Cross-region read replicas (asynchronous, e.g., AlloyDB secondary clusters or Cloud SQL cross-region replicas) lag; a remote clinic reads stale patient data (e.g., outdated medication list). **Mitigation:** Managed replicas offer no guaranteed staleness bound, so monitor replica lag and alert on it, and route clinically critical reads (medications, allergies) to the primary; Spanner offers explicit strong and bounded-staleness reads if a global consistent store is needed.

**Security & compliance:**
- **HIPAA encryption:** PHI is encrypted at rest by default with Google-managed keys and in transit with TLS; HIPAA does not mandate CMEK. Use **CMEK** (Cloud KMS) when EHR wants control over keys (rotation, revocation, separation of duties); a rotation period such as 90 days is a policy choice.
- **Business Associate Agreement (BAA):** Google Cloud offers a BAA that EHR must accept before putting PHI on GCP, and PHI may only be placed in HIPAA-covered services (see Google's HIPAA compliance page); EHR stays responsible for its configuration (shared responsibility).
- **Audit logs (HIPAA audit controls):** Log all access to patient data (Admin Activity logs are always on; **Data Access logs must be enabled** for most services). HIPAA requires retaining required documentation for 6 years; log retention periods are set by EHR's policy and applicable law, not fixed by HIPAA for logs. Use a locked log bucket / Cloud Storage Bucket Lock to prevent deletion.
- **Access controls:** Only users with legitimate clinical need can access patient data. Use IAM roles + Cloud Identity groups. Clinic admins manage group membership. Log all access via Audit Logs.
- **Data residency (design assumption, not in the case):** GDPR restricts transfers of EU personal data outside the EEA unless safeguards exist, so many EU customers require EU-only storage. Use organisation policy (resource-location constraint) to enforce EU locations for data; a regional BigQuery dataset gives the strictest control, though the EU multi-region also stays within the EU.
- **Sensitive Data Protection (DLP):** Scan BigQuery and Cloud Storage for PHI (SSN, MRN, health records); mask or tokenise before exposing to non-clinical staff (e.g., analytics team).

**AI components:**
- **Predictive health insights** (the case asks for healthcare-trend insights and predictions): Use BigQuery ML to predict outcomes (e.g., readmission risk). Train on de-identified historical data. Use feature attributions (BigQuery ML explainability) and require clinician review before action.
- **Document intelligence (optional design assumption):** Use Document AI to extract structured data from patient intake forms, insurance documents.
- **Conversational health assistant (optional design assumption, not in the case):** Use Gemini to answer FAQs (e.g., appointment times, prior-authorisation questions). Use only HIPAA-covered services under the BAA; do not expose PHI in conversation logs.
- **Trend analysis:** Use BigQuery + Looker to identify clinic-level trends (e.g., high readmission rates, common diagnoses). Surface to clinic admins for quality improvement.

**Likely question types & worked examples:**

**Q1. (Assumes an EU residency requirement for EHR's multinational customers; the case itself only names HIPAA.) EHR's EU clinics' patient analytics data currently sits in a BigQuery dataset in the US multi-region, which breaks the EU residency requirement. What should they do? (Select TWO.)**

A. Create BigQuery datasets in an EU location (e.g., the EU multi-region or europe-west1), load the EU data there, and delete it from the US dataset; keep US clinics' data in a US dataset.  
B. Apply the organisation policy resource-location constraint (`gcp.resourceLocations`) to restrict where new resources can be created to the allowed EU (and US) locations.  
C. Use Sensitive Data Protection to mask PHI in the US dataset before serving queries to EU analysts.  
D. Run BigQuery ML in europe-west1; it automatically relocates the training data.

**Answer: A and B.** The constraint is data residency. Dataset location is fixed at creation, so existing data must be moved (A); the organisation policy then prevents anyone from creating resources outside the allowed locations (B):
- A fixes the violation (location cannot be changed in place; copy/re-load, then delete the old data)
- B is the preventive guardrail (a hard constraint on new resources; it does not move existing data)
- Audit logs record resource-creation attempts

Why others wrong:
- C: Masking limits visibility but the data is still stored and processed in the wrong location; residency is about where data is stored and processed.
- D: BigQuery ML runs where the dataset lives; it does not relocate data or stop others from creating datasets in other locations.

---

**Q2. (Hypothetical design question.) EHR runs its patient database on AlloyDB with a cross-region secondary cluster for read traffic. A doctor in a remote clinic reviews a medication list served by the replica that is stale by 3 minutes, and prescribes a conflicting medication, causing an adverse drug event (ADE). How should EHR design to avoid this?**

A. Disable the replicas; all reads go to the primary region. Remote clinics get higher latency but consistent reads.  
B. Keep the replicas, and have doctors acknowledge a banner saying "this data may be up to 10 seconds old" before prescribing.  
C. Migrate the whole system to Spanner, because it offers strong reads everywhere.  
D. Keep replicas for non-critical views, but make clinically critical operations (prescribing, allergy and interaction checks) read from the primary, and monitor replica lag.

**Answer: D.** The constraint is "prevent ADEs due to stale data." **D** is best:
- **Critical reads (prescriptions) must be strong** (read from the primary)
- **Non-critical reads (view-only, past history)** can use replicas
- Application logic: separate paths for strong vs eventual reads
- Replication lag is monitored and alerted on

Why others wrong:
- A: Correct but over-broad: it penalises every remote read, and the primary then carries all read load. Targeting only critical reads gives the same safety.
- B: AlloyDB cross-region replication is asynchronous with no guaranteed staleness bound, and a human acknowledgement is not a safety control.
- C: Spanner can serve strong reads across regions, but a full re-platform is a large cost and risk when routing critical reads to the primary already solves the problem.

---

**Q3. EHR is federating its workforce from on-prem Active Directory to Cloud Identity (SAML). A legacy insurer-integration service that stays on-prem must call Google Cloud APIs directly (for example, publish claims events to Pub/Sub). What should EHR do?**

A. Create a service account in GCP; download a JSON key and store it on the on-prem server.  
B. Set up Workload Identity Federation with the on-prem IdP (e.g., AD FS issuing OIDC/SAML tokens). The service exchanges its token via STS for short-lived Google credentials (impersonating a service account or using the federated principal directly).  
C. Create a Cloud Identity user account for the integration service and have it log in via SAML SSO.  
D. Rely only on the Cloud VPN/Interconnect link plus mutual TLS to an Apigee proxy, with the service calling Pub/Sub directly using its client certificate.

**Answer: B.** The constraint is "on-prem workload authenticates to GCP without long-lived keys." **B** (Workload Identity Federation) is designed for this:
- The on-prem IdP (e.g., AD FS) issues a token for the workload
- The workload exchanges it through the Security Token Service for a short-lived Google credential (typically 1 hour), scoped by IAM
- No long-lived keys to store, rotate or leak

Why others wrong:
- A: Downloaded service account keys are long-lived credentials and an anti-pattern (leak risk, rotation burden); organisations often block creation with the `iam.disableServiceAccountKeyCreation` org policy.
- C: Cloud Identity user accounts are for people (interactive SSO), not for services.
- D: A private link and mTLS secure transport and can authenticate the caller to Apigee, but Google APIs such as Pub/Sub still require Google credentials/IAM identity; a client certificate does not provide one. (Apigee mTLS is a good fit when the service calls an API *behind* Apigee.)

---

### 5. KnightMotives Automotive

**Summary:**  
KnightMotives is a car maker working on autonomous (self-driving) vehicles and selling BEVs, hybrids and ICE vehicles. BEVs have a modern in-vehicle experience; hybrid/ICE models don't, and sales are declining. They want a consistent AI-powered experience across all models within five years, better shopping/buying/service experiences, a reliable build-to-order system (the current online one hurts dealers) and data monetisation. Existing: mostly on-prem plus some clouds, outdated mainframe supply chain, outdated ERP, fragmented vehicle code bases and tech debt, poor plant and rural vehicle connectivity, dealers with no equipment budget. The constraint keywords are **"security"** (past breaches; paramount) and **"data monetisation"**. Also **EU data protection (GDPR)**, including for the autonomous platforms.

| Requirement | Design Decision | GCP Service(s) | WAF Pillar |
|---|---|---|---|
| Consistent AI UX across all vehicle models (old + new hardware) | Modular AI inference service (Cloud Run); lightweight model on-device (LiteRT, formerly TensorFlow Lite) where the hardware allows, cloud inference for heavy models and for legacy head units; OTA updates for older hardware | Cloud Run, Cloud Run functions, LiteRT / TensorFlow Lite, Agent Platform Endpoints | Performance, Reliability |
| Real-time connectivity (plants, dealers, vehicles) | 5G/4G telemetry → Pub/Sub → Dataflow (stream aggregation); Firestore for real-time state; Spanner for transactional consistency | Pub/Sub, Dataflow, Firestore, Spanner, Eventarc | Reliability, Performance |
| Data monetisation (anonymised mobility insights, predictive maintenance) | Pub/Sub → BigQuery (data warehouse); aggregate anonymised telemetry; expose via Apigee (partner API); Sensitive Data Protection to anonymise PII | Pub/Sub, BigQuery, Apigee, Sensitive Data Protection (de-identification), Analytics Hub (data sharing) | Cost optimisation, Security |
| Autonomous-driving development (simulation, testing) | Create simulation environment (virtual vehicle, test scenarios); Dataflow to replay real vehicle telemetry; GKE (GPU node pools) or Batch for large parallel simulation runs; Agent Platform custom training on GPUs/TPUs for driving models (BigQuery ML is not suited to this) | Dataflow, GKE, Batch, Agent Platform custom training, BigQuery (test results) | Reliability, Performance |
| Mainframe/ERP modernisation (gradual, hybrid) | Set up API gateway (Apigee) to abstract legacy systems; Dataflow for ETL (ingest, transform, enrich); Cloud Composer for orchestration; keep mainframe running until replaced | Apigee, Dataflow, Cloud Composer, Application Integration (or custom APIs) | Reliability, Cost optimisation |
| Dealer tooling (sales, inventory, service) | Build user-facing apps in Cloud Run; read inventory from AlloyDB; CRM data in Firestore; use Gemini for recommendations (e.g., "suggest extended warranty for this vehicle") | Cloud Run, AlloyDB, Firestore, Gemini, Looker | Performance, Cost optimisation |
| Vehicle telemetry security (GDPR, past breaches) | Encrypt all telemetry in transit (TLS) and at rest (Cloud KMS CMEK); de-identify personal data (driver ID, precise location, VIN) before analytic storage; audit all access via IAM + Audit Logs | Cloud KMS, Sensitive Data Protection, IAM, Cloud Audit Logs | Security, Compliance |
| EU data residency (GDPR autonomous vehicle data) | Regional resources (europe-west1 for EU vehicles); organisation policy to enforce; BigQuery regional datasets | Cloud Storage, BigQuery (regional), organisation policy | Compliance, Security |

**Existing-environment gotchas:**
- **Mainframe supply chain:** KnightMotives' supply chain runs on an outdated mainframe (the case gives no further detail; assume batch-oriented with limited APIs). Migrating to GCP requires a phased approach: parallel run (old + new), ETL bridging (mainframe → Dataflow → target database), cutover validation. **Apigee** can front the mainframe with a modern API interface (strangler pattern).
- **Fragmented vehicle code bases:** The case cites fragmented vehicle code bases and technical debt. A reasonable design: define a **canonical telemetry schema** (Pub/Sub message format), have each platform map its data to this schema, and aggregate in Dataflow. Use Firestore for real-time state; BigQuery for analytics.
- **Plant and rural connectivity:** The case cites poor plant and rural-vehicle connectivity and needs network upgrades between plants and HQ. Solution: upgrade WAN links (Interconnect/Partner Interconnect/VPN as appropriate) and use **edge processing** (on-vehicle or plant-local) to filter/compress, sending summaries (aggregates, anomalies) via Pub/Sub; buffer data locally when offline.
- **Dealer equipment budget:** Dealers have no equipment budget. Solution: web-based dealer tools (Cloud Run web app) that run on existing devices, avoid device-side heavy lifting.
- **Past security breaches:** KnightMotives has had breaches and security is paramount; the case calls for a security framework, incident response and awareness training. Required on GCP: **encryption everywhere** (CMEK for data at rest, TLS for transit), **identity federation** (OAuth 2.0 / SAML for dealer login, mTLS for vehicle↔cloud), **audit logs** (all API calls, data access), **periodic penetration testing**.

**Target architecture (ASCII):**

```
┌──────────────────────────────────────────────────────────────────┐
│                    KnightMotives Automotive                       │
├──────────────────────────────────────────────────────────────────┤
│                                                                    │
│  VEHICLES (5G/4G/WiFi)         CLOUD (GCP Multi-Region)           │
│  ┌─────────────────────┐                                          │
│  │ In-Vehicle System   │       ┌─────────────────────────────┐    │
│  │ (BEV/Hybrid/ICE)    │       │ Pub/Sub (telemetry ingestion)    │
│  │ ├ GPS, speed,       │   ──→ │ ├ Topic: vehicle-telemetry  │    │
│  │ │  acceleration     │   (5G)│ ├ Topic: driver-behavior    │    │
│  │ ├ Door locks, temp  │   (4G)│ └ Topic: diagnostics        │    │
│  │ ├ Battery % (BEV)   │       │ (ordering key: vehicle_id)  │    │
│  │ ├ ML inference      │       │                             │    │
│  │ │  (TF Lite)        │       └─────────────────────────────┘    │
│  │ │  - Driver assist  │                ▼                        │
│  │ │  - Anomaly detect │    ┌──────────────────────────────┐      │
│  │ ├ Encrypt (TLS,     │    │ Dataflow (stream pipeline)   │      │
│  │ │  mTLS)            │    │ ├ Decode Pub/Sub messages    │      │
│  │ │ - Device identity │    │ ├ De-identify PII (Sensitive │      │
│  │ │ - Cert rotation   │    │ │  Data Protection)           │      │
│  │ └ Fallback mode     │    │ ├ Aggregate (5-min windows)  │      │
│  │   (offline cache)   │    │ ├ Detect anomalies (ML)      │      │
│  └─────────────────────┘    │ └ Sink → BigQuery + Firestore│      │
│           ▲                 │                             │      │
│           │                 └──────────────────────────────┘      │
│  ┌────────┴──────────────────────────────────────────────────┐   │
│  │ Cloud Run (AI Inference Service)                          │   │
│  │ ├ Route requests to on-device model (if available)        │   │
│  │ ├ Fallback to cloud inference (Gemini, Agent Platform EP) │   │
│  │ ├ Log all inferences (audit trail)                        │   │
│  │ └ Encrypt responses (TLS)                                 │   │
│  └────────────────────────────────────────────────────────────┘   │
│                                                                    │
│  ┌────────────────────────────────────────────────────────────┐   │
│  │ BigQuery (Data Warehouse; EU + US regions separated)      │   │
│  │                                                            │   │
│  │ EU Region (europe-west1):                                │   │
│  │  ├ Telemetry (anonymised, PII removed)                    │   │
│  │  ├ Driver behavior (anonymised segments)                  │   │
│  │  ├ Autonomous-vehicle test logs                           │   │
│  │  └ Retention: 3 years (GDPR right to deletion enforced)   │   │
│  │                                                            │   │
│  │ US Region (us-central1):                                 │   │
│  │  ├ Supply-chain data (non-PII)                            │   │
│  │  ├ Sales data (anonymised for trends)                     │   │
│  │  └ Retention: per contract                                │   │
│  └────────────────────────────────────────────────────────────┘   │
│                                                                    │
│  ┌────────────────────────────────────────────────────────────┐   │
│  │ Firestore (Real-Time State; multi-region)                │   │
│  │ ├ Vehicle state (locked, location, fuel, battery)         │   │
│  │ ├ Driver profile (preferences, emergency contacts)        │   │
│  │ └ CMEK encryption; TTL on sensitive fields (6 months)     │   │
│  └────────────────────────────────────────────────────────────┘   │
│                                                                    │
│  ┌────────────────────────────────────────────────────────────┐   │
│  │ Spanner (Transactional; supply chain, sales orders)      │   │
│  │ ├ Supply chain (parts inventory, vendor orders)           │   │
│  │ ├ Sales orders (build-to-order, dealer orders)            │   │
│  │ ├ Service schedules (maintenance, recalls)                │   │
│  │ └ CMEK encryption; audit logs for all writes              │   │
│  └────────────────────────────────────────────────────────────┘   │
│                                                                    │
│  ┌────────────────────────────────────────────────────────────┐   │
│  │ Apigee (API Gateway; dealer API, partner data API)        │   │
│  │ ├ Dealer API (inventory, orders, service scheduling)      │   │
│  │ ├ Partner API (anonymised mobility insights; monetise)    │   │
│  │ ├ OAuth 2.0 (dealer login), mTLS (vehicle auth)           │   │
│  │ ├ Rate limiting, quota management                         │   │
│  │ ├ Request/response logging → BigQuery                     │   │
│  │ └ DLP policies (mask VIN, license plate in responses)     │   │
│  └────────────────────────────────────────────────────────────┘   │
│                                                                    │
│  ┌────────────────────────────────────────────────────────────┐   │
│  │ GKE (Simulation & Development)                            │   │
│  │ ├ Autonomous-driving simulation (CARLA, custom env)       │   │
│  │ ├ Replay real vehicle telemetry (Dataflow ingest)         │   │
│  │ ├ Distributed simulation cluster (auto-scale)             │   │
│  │ └ Store results in Spanner (test case, pass/fail)         │   │
│  └────────────────────────────────────────────────────────────┘   │
│                                                                    │
│  ┌────────────────────────────────────────────────────────────┐   │
│  │ Cloud Composer + Application Integration (Mainframe Bridge)│   │
│  │ ├ Orchestrate supply-chain ETL jobs                       │   │
│  │ ├ Ingest mainframe data (batch, scheduled)                │   │
│  │ ├ Transform & enrich with GCP data                        │   │
│  │ └ Load to Spanner (replace mainframe queries gradually)   │   │
│  └────────────────────────────────────────────────────────────┘   │
│                                                                    │
│  ┌────────────────────────────────────────────────────────────┐   │
│  │ DEALER TOOLS (Cloud Run Web Apps + Mobile)                │   │
│  │ ├ Sales app: inventory search, order placement            │   │
│  │ │  (Gemini recommendations: "Extend warranty? → 80 % → Y")│   │
│  │ ├ Service app: schedule maintenance, track recalls        │   │
│  │ ├ Admin app: manage inventory, view analytics             │   │
│  │ └ Auth: OAuth 2.0 (federated identity, MFA)               │   │
│  └────────────────────────────────────────────────────────────┘   │
│                                                                    │
│  ┌────────────────────────────────────────────────────────────┐   │
│  │ SECURITY & COMPLIANCE (Embedded)                          │   │
│  │ ├ Cloud KMS (CMEK key management; annual rotation)        │   │
│  │ ├ IAM (roles per team: dealer, engineer, analyst, support)│   │
│  │ ├ Audit Logs (Admin Activity, Data Access, System Events) │   │
│  │ │  └ All API calls, data access, privilege changes        │   │
│  │ ├ Cloud Armor (DDoS protection for APIs)                  │   │
│  │ ├ VPC Service Controls (perimeter for sensitive data)     │   │
│  │ ├ Sensitive Data Protection (DLP; scan PII before expose) │   │
│  │ └ Penetration testing (quarterly, external + internal)    │   │
│  └────────────────────────────────────────────────────────────┘   │
│                                                                    │
│  ┌────────────────────────────────────────────────────────────┐   │
│  │ OBSERVABILITY                                              │   │
│  │ ├ Cloud Monitoring (latency, errors, vehicle uptime SLO)  │   │
│  │ ├ Managed Prometheus (custom metrics for autonomous fleet)│   │
│  │ ├ Cloud Logging (structured logs from all components)     │   │
│  │ └ Alerting (SLO burn rate, anomalies, security events)    │   │
│  └────────────────────────────────────────────────────────────┘   │
│                                                                    │
└──────────────────────────────────────────────────────────────────┘
```

**Migration/phasing plan (illustrative design assumption, the case study gives no timeline; the five-year target is for a consistent AI experience across all models):**
1. **Phase 1 (Months 1–3):** Set up Pub/Sub, Dataflow, BigQuery for telemetry ingest. Test with 1000 connected vehicles (sample); measure latency, cost.
2. **Phase 2 (Months 4–6):** Deploy Apigee + dealer API (inventory, order placement). Enroll 20 pilot dealers; measure adoption, feedback.
3. **Phase 3 (Months 7–9):** Set up GKE + simulation environment. Begin autonomous-driving team to test scenarios using replay telemetry.
4. **Phase 4 (Months 10–12):** Migrate supply-chain ETL (mainframe → Cloud Composer + Dataflow → Spanner). Run in parallel; validate data consistency.
5. **Phase 5 (Months 13–15):** Deploy data monetisation API (anonymised insights via Apigee); negotiate partner contracts. Start generating revenue from anonymised telemetry.
6. **Phase 6 (Months 16+):** Gradual vehicle fleet upgrade (firmware/OTA) to newer AI models. Decommission legacy on-prem systems; consolidate on GCP.

**Risks & mitigations:**
- **Risk:** Vehicle data leaked (hacked VIN, location, driver behavior). **Mitigation:** Encrypt all data at rest (CMEK) and in transit (TLS + mTLS vehicle-to-cloud). Audit all access via Cloud Audit Logs. Implement anomaly detection on access patterns (Security Command Center, log-based alerts, BigQuery ML over audit logs).
- **Risk:** Mobile network (5G/4G) unavailable (tunnels, rural areas); vehicle cannot reach cloud. **Mitigation:** Run AI features that must work offline on-device (e.g., LiteRT/TensorFlow Lite models for driver assistance UX); safety-critical vehicle functions (braking, stability control) are never cloud-dependent and run on the vehicle's own certified controllers. Cloud inference is an enhancement/fallback for non-critical features.
- **Risk:** Plant connectivity bottleneck; telemetry backlog causes processing delays. **Mitigation:** Implement edge aggregation (plant-local summarisation); stream only anomalies/aggregates to cloud. Keep raw data local, and move bulk data for later analysis with Storage Transfer Service/Transfer Appliance if needed.
- **Risk:** Mainframe supply-chain migration fails; orders back up. **Mitigation:** Parallel run (old + new) for 3 months; validate all orders match; cutover only after 99.9 % match rate.
- **Risk:** GDPR audit finds EU telemetry exported to US region. **Mitigation:** Use organisation policy (enforce europe-west1 for EU vehicle data); audit logs verify compliance; segregate regional datasets.

**Security & compliance:**
- **Encryption:** All data at rest is encrypted by default; given the breach history, use CMEK in Cloud KMS (HSM protection level if the security team requires FIPS 140-2 Level 3 validated keys; a design choice, not stated in the case); encrypt in transit with TLS (mTLS for vehicle-to-cloud).
- **GDPR (autonomous vehicles):** EU vehicle data (telemetry, driver profiles) stored in EU regions only (design assumption). Implement erasure requests (index personal data by driver ID so it can be located and deleted; Sensitive Data Protection helps discover and de-identify personal data in BigQuery) and retention limits. Log all access via Audit Logs.
- **Incident response:** Use Security Command Center to track vulnerabilities. Penetration testing quarterly. Incident response playbook (detect → isolate → remediate → post-mortem).
- **Supply-chain security:** Artifact Registry (container images), Binary Authorization (only approved images run on GKE), SLSA provenance (track who built and signed code).

**AI components:**
- **On-device inference (LiteRT, formerly TensorFlow Lite):** Lightweight in-vehicle AI features run locally for low latency and offline resilience, where the vehicle hardware allows (legacy hardware may need updates or cloud inference). Safety-critical driving functions are outside this scope.
- **Cloud inference (Gemini + custom models):** Complex reasoning (scenario analysis, personalised recommendations) runs on a suitable Gemini model or a custom model on Agent Platform Endpoints; check current model names in Ch08.
- **Autonomous-driving simulation:** The case requires a development/test simulation environment. Replay real telemetry in a simulator (e.g., open-source CARLA is an assumption, not named in the case) on GPU-enabled GKE or Batch; train driving models with Agent Platform custom training on GPUs/TPUs. Test edge cases without risking real vehicles.
- **Predictive maintenance (design assumption):** BigQuery ML or Agent Platform to predict component failures (engine, battery) from sensor data. Alert dealers in advance; recommend maintenance.
- **Dealer recommendation engine:** Gemini to suggest extended warranty, service packages, or upgrades based on vehicle age, mileage, and driver profile (anonymised).
- **Data monetisation:** Aggregate anonymised mobility insights (traffic patterns, road hazards, weather impacts); expose via Apigee Partner API; license to insurance, navigation, city-planning companies.

**Likely question types & worked examples:**

**Q1. KnightMotives collects 100 million telemetry events/day from 5 million connected vehicles. Each event includes vehicle_id, driver_id, GPS, speed, acceleration. They want to monetise this data by selling anonymised traffic insights to city-planning agencies. Under EU GDPR, data that can still be linked to individuals remains personal data. What should they do?**

A. Use Sensitive Data Protection to mask vehicle_id and driver_id in the event-level data, and publish the event-level dataset.  
B. Aggregate at the road-segment level and publish aggregates, with no minimum group size (e.g., "3 vehicles passed this intersection at 03:00, average speed 45 km/h").  
C. Aggregate by road segment + time window, publish only groups that meet a minimum group size (k-anonymity style threshold, verified with Sensitive Data Protection risk analysis), and suppress small groups; keep event-level data inside the controlled environment.  
D. Publish the full dataset; train a machine-learning model to infer driver attributes and filter the model output.

**Answer: C.** The constraint is "GDPR: prevent re-identification." **C** combines aggregation with a minimum group size:
- Group data by road segment + time window and publish only aggregates (e.g., average speed, counts)
- Suppress or merge groups below the threshold so small groups cannot single out individuals
- Sensitive Data Protection risk analysis can measure k-anonymity/l-diversity of a table before release
- GDPR sets no numeric threshold; the test is whether individuals can still be singled out, linked or inferred. Precise trajectories are notoriously hard to anonymise, so consider stronger techniques (e.g., differential privacy) for fine-grained data, and keep legal review in the loop

Why others wrong:
- A: Masking identifiers is pseudonymisation at best: GPS traces and timestamps can re-identify a driver (e.g., home and work locations), so the data is still personal data.
- B: Aggregation without a minimum group size can still single out individuals when the group is tiny (e.g., one vehicle at 03:00).
- D: Releasing the raw data and "filtering" model output does not anonymise anything; models can also leak training data (membership inference).

---

**Q2. (Hypothetical scenario; the case only says the supply chain runs on an outdated mainframe and calls for gradual legacy modernisation.) KnightMotives' mainframe supply-chain system produces a nightly fixed-width ORDER batch file. They want to modernise gradually without a risky big-bang cutover. What GCP architecture should they use?**

A. Rewrite the whole application in a modern language and cut over in one weekend; shut down the mainframe.  
B. Keep the mainframe running. Land the nightly ORDER file in Cloud Storage, orchestrate with Cloud Composer, parse/transform with Dataflow, load into a modern database (e.g., Spanner or Cloud SQL). Run both systems in parallel, validate consistency, then migrate consumers and cut over in stages.  
C. Use BigQuery Omni to query the mainframe database directly. No migration needed.  
D. Only put an Apigee API facade in front of the mainframe and have all new applications call it, with no data movement.

**Answer: B.** The constraint is "gradual modernisation of legacy while hybrid." **B** is the safe, phased approach:
- **Parallel run:** The mainframe continues (no disruption); Composer + Dataflow ingest and transform the nightly files
- **Validation:** Compare mainframe results to the new system's results; flag mismatches
- **Cutover:** Only after a sustained match rate, switch consumers; rollback to the mainframe stays possible
- (Google also offers mainframe-specific tooling such as Mainframe Connector and Dual Run for this pattern.)

Why others wrong:
- A: A big-bang rewrite is high-risk and contradicts "gradual legacy modernisation."
- C: BigQuery Omni queries data stored in AWS S3 or Azure Blob Storage; it cannot query a mainframe database.
- D: An API facade is a useful part of a strangler-pattern approach (and appears in the table above), but on its own it leaves all data and availability dependent on the mainframe and does not modernise anything.

---

**Q3. KnightMotives' in-vehicle AI features (e.g., a voice assistant and driver-assist UX) must keep working with no connectivity (e.g., in a tunnel or rural area), and a cloud round trip over a mobile network cannot meet their real-time latency target. What is the best design?**

A. Use a cloud-only model (Gemini on Cloud Run); accept degraded performance in tunnels.  
B. Deploy compact on-device models (LiteRT, formerly TensorFlow Lite) for features that must work offline and at low latency, and use cloud inference for heavier or less time-critical features, with OTA model updates.  
C. Use Spanner for local caching; download the model database to the vehicle; run inference locally.  
D. Use Cloud Run batch prediction; pre-compute all predictions and store them in the vehicle (e.g., the next 1000 miles of recommendations).

**Answer: B.** The constraints are "work offline" and "real-time latency." **B** is the right split:
- **Offline/real-time features:** run on-device; no network dependency and no round-trip latency
- **Heavy or non-time-critical features** (large-model reasoning, recommendations): cloud inference when connected, with a degraded or cached mode when not
- **Safety:** Safety-critical vehicle functions are handled by the vehicle's own certified controllers and never depend on the cloud or on these AI features
- **Model lifecycle:** train in the cloud (Agent Platform), push updates over the air; this also helps legacy hardware only where it can run the model

Why others wrong:
- A: Cloud-only fails in tunnels and cannot meet the real-time latency target.
- C: Spanner is a database, not an inference runtime; "caching a model locally" is just on-device inference, which is B.
- D: Inputs are real-time (driver commands, road and traffic conditions), so pre-computing a long horizon of predictions is impractical and stale.

---

## Summary comparison table (all 4 cases)

| | Altostrat Media | Cymbal Retail | EHR Healthcare | KnightMotives Automotive |
|---|---|---|---|---|
| **Industry** | Media, podcasts | E-commerce, retail | Healthcare SaaS | Automotive, autonomous vehicles |
| **Current environment** | GKE, Cloud Storage, BigQuery, Cloud Run functions, some legacy on-prem ingestion/archival | On-prem + cloud: MySQL, SQL Server, Redis, MongoDB, Kubernetes, SFTP/ETL batch, IVR + call centre | Colocation lease expiring; AD users; legacy file/API insurer integrations on-prem | Outdated mainframe supply chain and ERP, fragmented code bases, poor plant and rural connectivity |
| **Top constraint(s)** | Reliability and cost management, auditable and explainable AI | Reduce call-centre and data-centre costs | 99.9 % availability, HIPAA compliance | Security paramount, data monetisation, EU data protection (GDPR) |
| **Signature services** | Cloud Monitoring + Prometheus, Gemini RAG, Autoclass, HITL review | Document AI, Gemini image models, Conversational Agents, Cloud SQL, Dataflow | Regional multi-zone GKE, Cloud SQL/AlloyDB replicas, AD federation (GCDS + SAML / Workforce Identity Federation), Apigee | Pub/Sub telemetry, Dataflow streaming, GKE simulation, Agent Platform custom training, Analytics Hub |
| **Migration phase** | Illustrative: 10–12 months | Illustrative: 12 months | Illustrative: 12 months | Illustrative: 16+ months (mainframe modernisation) |
| **AI pillar** | Gen-AI content generation + NL chat + explainability | Gen-AI catalog enrichment + conversational commerce + HITL review | None in the case (carried-over, pre-AI); predictive health or an assistant would be design assumptions | Autonomous-driving sim + on-device inference + data monetisation |
| **Database design** | BigQuery (warehouse), Cloud Storage (media) | Cloud SQL for MySQL / SQL Server, Firestore (products), Memorystore (Redis); AlloyDB (PostgreSQL-compatible) or Spanner only as a re-platform | Cloud SQL for MySQL / SQL Server (with cross-region replicas) or AlloyDB if re-platforming to PostgreSQL | Spanner (supply chain, sales), Firestore (real-time state), BigQuery (analytics) are reasonable design choices |
| **Hybrid/compliance** | On-prem ingestion → Cloud; storage cost optimisation | Mixed on-prem/cloud estate; SFTP/ETL batch; PCI DSS if card data is in scope (design assumption) | Regional HA, HIPAA (BAA), insurer integration stays on-prem, AD federation | EU GDPR (transfer rules), mainframe bridge, vehicle connectivity |
| **Observability** | Cloud Monitoring + Prometheus, email alerts → alerting policies | Grafana/Nagios/Elastic → Cloud Monitoring/Logging | Open-source monitoring with often-ignored email alerts → Cloud Monitoring + SLO burn rate | Managed Service for Prometheus + custom metrics (design assumption) |
| **Exam keywords to memorise** | "auditable," "explainable," "Autoclass," "HITL," "reliability and cost" | "cost," "automate," "call-centre," "HITL review," "database migration" | "99.9 %," "HIPAA," "failover," "AD federation" | "security," "GDPR," "data monetisation," "edge inference," "mainframe modernisation" |

---

## Well-Architected Framework lens

| WAF Pillar | Altostrat | Cymbal | EHR | KnightMotives |
|---|---|---|---|---|
| **Operational Excellence** | Cloud Monitoring + Prometheus, alerting policies replacing email-only alerts, SLO burn rate alerts, audit logs | Cloud Monitoring/Logging replacing Grafana/Nagios/Elastic where sensible, alerting policies, Dataflow observability | Cloud Monitoring + custom SLO metrics (99.9 %), alerting policies for the often-ignored email alerts, Cloud Logging audit trail | Managed Service for Prometheus + custom metrics for the fleet (design assumption), vehicle uptime dashboards |
| **Security (Privacy & Compliance)** | Inference logging/audit logs for auditable AI, HITL review trails, Sensitive Data Protection where needed | PCI DSS scoping if card data is present (design assumption), Sensitive Data Protection (PII in analytics) | HIPAA (BAA, HIPAA-covered services only, Data Access audit logs enabled; CMEK optional), Sensitive Data Protection (PHI masking) | EU GDPR (transfer rules, Organisation Policy resource locations as a guardrail), encryption (CMEK optional), Sensitive Data Protection (de-identification, risk analysis) |
| **Reliability** | GKE regional clusters, Autoclass (no manual tiering), event-driven Cloud Run functions pipeline with retries | Cloud SQL HA, Memorystore HA, Firestore multi-region, Pub/Sub buffering for order ingestion | Regional multi-zone GKE, Cloud SQL/AlloyDB HA with cross-region replicas for DR, Global external Application Load Balancer with backend health checks | Edge inference (offline operation), Pub/Sub buffering and fan-out, Dataflow autoscaling and backpressure |
| **Performance Optimisation** | BigQuery partitioning/clustering + BI Engine, Vector Search, parallel Cloud Run functions | Batch generation for catalog images, Dataflow streaming, Firestore real-time reads | Cross-region read replicas for read latency (replica lag is not configurable), caching with Memorystore | LiteRT on-device for low latency, cloud inference fallback, Pub/Sub ordering keys per vehicle |
| **Cost Optimisation** | Autoclass, lifecycle policies, BI Engine (query efficiency) | Conversational agents deflecting routine calls (the case goal: lower call-centre and data-centre cost), Dataflow FlexRS for non-urgent batch ETL | Right-size GKE, committed use discounts for steady baseline compute (a multi-region footprint costs more; regional design avoids that) | Pub/Sub + Dataflow autoscaling, BigQuery reservations/editions for predictable analytics |
| **Sustainability** | Autoclass (colder tiers for idle data), on-prem → cloud migration (efficient data centres), low-carbon region choice | Automation (less manual handling), serverless (Cloud Run, Dataflow), exit data centres | Exit the expiring colocation lease; choose low-carbon regions where residency allows | On-device inference (fewer cloud round trips), fewer duplicated data copies |

---

## AI/GenAI angle

**Altostrat:**
- Gemini for recommendations, summarisation, metadata extraction, content moderation (design assumptions; the case lists recommendations and metadata extraction as existing functions).
- Fine-tuning may help for domain-specific content-policy classification; try prompting/grounding first.
- Auditability and explainability (a stated technical requirement): log inputs, outputs and model versions; use HITL review trails; use Agent Platform Explainable AI where the model type supports it, and for generative models rely on grounding with citations and logged reasoning inputs.

**Cymbal:**
- Gemini image models (check the current version) for product image generation, colour variants, background changes, text overlays.
- Document AI for supplier data extraction (PDFs, invoices).
- Gemini for product descriptions, attribute inference, conversational shopping assistant.
- Agent Search for semantic product discovery.
- HITL review critical: humans approve/reject auto-generated content before publication.

**EHR** (carried-over, pre-AI case; everything below is a reasonable design assumption, not case content):
- BigQuery ML for predictive health (e.g., readmission risk) on de-identified data, under the BAA and HIPAA-covered services only.
- Document AI for patient intake forms, insurance documents.
- Conversational health assistant (Gemini) for FAQs; do not expose PHI in logs.
- Trend analysis: Gemini + BigQuery to identify clinic-level patterns.

**KnightMotives:**
- LiteRT (formerly TensorFlow Lite) for on-device inference on features that must work offline with low latency (e.g., voice assistant, driver-assist UX); safety-critical vehicle functions stay on the vehicle's own certified controllers.
- Cloud inference (Gemini) for heavier reasoning when connected (design assumption).
- Autonomous-driving simulation: replay telemetry in a simulator on GKE GPU nodes, train models with Agent Platform custom training (GPUs/TPUs), not BigQuery ML.
- Predictive maintenance: BigQuery ML or Agent Platform models on telemetry (design assumption).
- Dealer recommendations: Gemini for warranty/service suggestions (design assumption).

---

## Case-study hooks

### Reading strategies
1. **Skim the exec statement (30 seconds).** Identify the constraint keyword ("cost," "reliability," "compliance," "security").
2. **Review existing environment.** What does it tell you about the migration path? (e.g., "MySQL + SQL Server" → plan for Cloud SQL for MySQL / SQL Server first, and AlloyDB or Spanner only if re-platforming; "on-prem AD users" → plan for AD federation via GCDS + SAML or Workforce Identity Federation).
3. **Identify gotchas.** Hybrid links, legacy systems, fragmented data, monitoring gaps.
4. **Map requirements to GCP services.** Use decision trees from other chapters (Ch03 networking, Ch04 compute, Ch05 storage/DB, etc.).
5. **Ask: "What would break if we got this wrong?"** That's the exam question.

### Cross-chapter integration

- **Ch01 (Exam strategy):** Case studies are 2 per exam (20–30 % of questions) and are shown split-screen with the questions, so know the cases in advance, skim the exec statement and keep its constraint keyword in mind.
- **Ch02 (HA/DR/SLOs):** EHR's "99.9 %" constraint → design regional multi-zone HA with health checks and cross-region DR, alert on SLO burn rate.
- **Ch03 (Networking):** Altostrat (hybrid connectivity for on-prem ingestion/archival), Cymbal (hybrid connectivity between on-prem and cloud estates, SFTP), EHR (hybrid connectivity to on-prem insurer integrations), KnightMotives (plant and rural connectivity, Pub/Sub fan-out for telemetry).
- **Ch04 (Compute):** Altostrat (GKE, Cloud Run functions), Cymbal (Kubernetes → GKE), EHR (regional multi-zone GKE, a design assumption), KnightMotives (GKE for simulation).
- **Ch05 (Storage/DB):** All 4 cases mix databases. Altostrat (Cloud Storage Autoclass, BigQuery), Cymbal (Cloud SQL, Firestore, Memorystore; AlloyDB/Spanner only if re-platforming), EHR (Cloud SQL/AlloyDB with replicas), KnightMotives (Spanner + Firestore + BigQuery as design choices).
- **Ch06 (Data processing):** Altostrat (Cloud Run functions for transcoding/metadata extraction), Cymbal (Dataflow for ETL replacing SFTP batch, Pub/Sub for orders), EHR (none explicit), KnightMotives (Pub/Sub telemetry → Dataflow).
- **Ch07 (Migration):** All 4 have migration stories. Migration Center for assessment/cost, Database Migration Service for schema and data to Cloud SQL (and PostgreSQL to AlloyDB), Datastream for CDC.
- **Ch08 (AI/Agent Platform):** Altostrat (Gemini + Agent Search + HITL), Cymbal (Gemini image models + Document AI + Conversational Agents + Discovery AI), EHR (BigQuery ML, an assumption), KnightMotives (LiteRT + simulation, Agent Platform custom training).
- **Ch09 (AI infrastructure):** KnightMotives (GPU/TPU training for simulation, on-device inference latency); other cases have no explicit AI-infrastructure angle.
- **Ch10/11 (IAM/Security):** EHR (AD federation via GCDS + SAML or Workforce Identity Federation; BAA and audit logs), KnightMotives (encryption + GDPR + de-identification).
- **Ch13 (CI/CD):** All 4 should have CI/CD pipelines. Exam questions may test Blue/Green or Canary deployments.
- **Ch14 (FinOps):** Altostrat (Autoclass, cost management), Cymbal (call-centre and data-centre cost reduction), EHR (exit colocation, regional vs multi-region cost), KnightMotives (Dataflow autoscaling, BigQuery reservations).
- **Ch15 (Operations/Reliability):** Altostrat (alerting policies beyond email), Cymbal (Grafana/Nagios/Elastic → Cloud Monitoring/Logging), EHR (SLO burn rate; often-ignored email alerts), KnightMotives (Managed Service for Prometheus + custom metrics).

---

## Exam traps & keyword → answer

1. **"Auditable and explainable AI"** → Model/version tracking + inference logging + audit logs + HITL review trails, plus Explainable AI where the model type supports it (not just "use Gemini").
2. **"HIPAA compliance"** → Sign the BAA + use only HIPAA-covered services + enable Data Access audit logs + least-privilege IAM (CMEK is optional, not required; Google encrypts at rest by default).
3. **"99.9 % availability"** → Regional multi-zone HA + health checks + cross-region DR + SLO monitoring (not just "add a replica").
4. **"Cost reduction"** → Automate manual processes (not just "use cheaper service") or consolidate databases.
5. **"GDPR"** → Control where data is stored/processed (Organisation Policy resource-location constraint) and how it is transferred outside the EEA; GDPR restricts transfers rather than strictly requiring EU storage.
6. **"Hybrid connectivity"** → Redundant Dedicated/Partner Interconnect attachments, with HA VPN as a lower-cost backup or starting point (not a single link).
7. **"Offline resilience"** → On-device inference (LiteRT) for features that must work offline (not cloud-only).
8. **"Data monetisation"** → Aggregate with a minimum group size, consider differential privacy for fine-grained data; measure risk with Sensitive Data Protection; pseudonymised data is still personal data.
9. **"Conversational agent"** → Requires grounding (Agent Search/Vector Search + RAG), tool or webhook integration with backend systems (e.g., order APIs), and HITL/human handoff (not just "drop in Gemini").
10. **"Real-time inventory"** → Firestore (not BigQuery, which is batch-oriented).
11. **"Legacy system integration"** → Apigee (API gateway) + Dataflow (ETL) (not direct table-to-table migration).
12. **"Image generation"** → Use a current Google image-generation model (check the current version) with HITL review before publication; do not rely on automated checks alone for brand or copyright issues.
13. **"Stale data in cross-region reads"** → Replicas lag by an unconfigurable amount; send reads that need the latest data to the primary (or use Spanner strong reads / bounded-staleness reads where Spanner is used).
14. **"Model retraining bottleneck"** → Decouple training from serving and feature/data preparation (Dataflow) from model ranking.
15. **"Email alerts ignored"** → Cloud Monitoring alerting policies tied to SLOs, routed to on-call channels (e.g., SMS, PagerDuty, webhook) with sensible thresholds to reduce noise.

---

## Quick-reference cheat table

### Case-study decision tree

| Question | Constraint | Answer | Why | Case |
|---|---|---|---|---|
| "How do we auto-generate product descriptions?" | Cost + speed | Dataflow + Gemini batch + HITL review | Batch is cheap; HITL prevents hallucinations | Cymbal |
| "How do we ensure 99.9 % availability?" | Regional HA + fast failover | Global external Application Load Balancer + regional multi-zone GKE + backend health checks + cross-region DR | Health checks remove unhealthy backends automatically | EHR |
| "How do we reduce call-centre cost?" | Automation required | Conversational Agents (Gemini) integrated with order APIs, with human handoff | Deflects routine calls; humans handle exceptions | Cymbal |
| "How do we keep AI auditable?" | Explainability + compliance | Version tracking + inference logging + audit logs + HITL review (+ Explainable AI where supported) | Every decision is traceable; humans review edge cases | Altostrat |
| "How do we migrate from mainframe?" | Keep mainframe running | Parallel run (mainframe + Cloud Composer + Dataflow), then staged cutover | Low risk; can roll back | KnightMotives |
| "How do we ensure offline AI works?" | Offline + low latency | LiteRT on-device + cloud for heavier features | Device inference works without connectivity; safety-critical functions never depend on the cloud | KnightMotives |
| "How do we comply with GDPR?" | Data location/transfers + anonymisation | Organisation policy (resource-location constraint) + aggregation with a minimum group size for shared data | Limits accidental data placement; aggregation reduces re-identification risk (GDPR sets no numeric threshold) | KnightMotives |
| "How do we reduce storage cost?" | Archive old data | Autoclass (or lifecycle policies when the access pattern is known) | Autoclass moves idle objects to colder classes automatically and has no retrieval or early-deletion fees | Altostrat |
| "How do we unify multi-DB stack?" | Consolidate databases | MySQL → Cloud SQL for MySQL; SQL Server → Cloud SQL for SQL Server; NoSQL → Firestore; cache → Memorystore; AlloyDB (PostgreSQL) or Spanner only if re-platforming | Lift to managed equivalents first; re-platform deliberately | Cymbal, EHR |
| "How do we ensure HIPAA encryption?" | Compliance required | Default Google encryption at rest + TLS in transit; CMEK only if policy demands more key control | Google encrypts at rest by default; HIPAA does not mandate CMEK; the BAA and HIPAA-covered services matter | EHR |

---

## Practice scenarios

**S1. Altostrat Media — Content moderation false positives**

Altostrat's video-moderation Gemini model has a 5 % false-positive rate (flags appropriate content as inappropriate). Every false positive costs Altostrat a complaint + manual review (30 min per review). They want to reduce false positives to <1 %. What should they do?

A. Switch to Vision API (label detection) instead of Gemini; it has lower false-positive rate.  
B. Fine-tune Gemini on Altostrat's historical moderation logs; use HITL review to collect feedback on false positives; retrain quarterly.  
C. Use a threshold-based approach: flag only if confidence > 0.99 (not 0.5); adjust to false-positive rate <1 %.  
D. Use an ensemble: run both Vision API and Gemini; flag only if both agree.

**Answer: B.** Reducing false positives requires domain-specific labelled data. **B** is the standard approach:
- Collect data: log all false positives + human feedback (HITL), which also supports the auditability requirement
- Improve the model with this data: refine the prompt/policy first, then fine-tune Gemini if needed (tells the model what Altostrat's guidelines are)
- Iterate: periodic retraining (the quarterly cadence is illustrative) based on newest feedback
- Measurement: evaluate against a held-out labelled set; track the false-positive rate against the <1 % target

Why others wrong:
- A: Vision API (label detection, SafeSearch) is a pre-trained, general-purpose model with fixed categories; it cannot learn Altostrat's own policy, and it is designed for images rather than full video understanding. Nothing shows it would have a lower false-positive rate here.
- C: Raising confidence threshold reduces false positives but increases false negatives (misses real violations). Doesn't address the root cause (model doesn't know Altostrat's guidelines).
- D: Ensemble is more expensive (2 API calls per decision) and doesn't solve the problem if both models are untrained on Altostrat's guidelines.

---

**S2. Cymbal Retail — Conversational agent fails to understand intent**

Cymbal's conversational agent has been live for 1 month. They see that 20 % of customer queries result in "I don't understand" fallbacks to a human agent. Common failures: "Find dresses with pockets," "Show me sizes XS to M," "Sort by lowest price." How should they diagnose and fix this?

A. Log all unresolved intents to BigQuery; cluster by error type; add function calls to Gemini for top-10 clusters (e.g., `get_products(has_pockets=True)`); measure resolution rate weekly.  
B. Hire more customer-service representatives; the agent is not sophisticated enough.  
C. Replace the agent with a different chatbot product, on the assumption that another platform has pre-built intent matching.  
D. Move to the largest available Gemini model without changing how the agent calls the product catalogue.

**Answer: A.** The failing queries are structured catalogue searches (attribute filters, ranges, sorting), so the gap is tool/function integration and a feedback loop, not model size. **A** is the diagnostic + iterative fix:
- Log all unresolved queries → BigQuery
- Cluster: "pockets" + "sizes" + "price sort" are top 3
- Add function (tool) declarations the agent can call: `get_products(pocket_style, sizes, sort_order)`
- Add example conversations/instructions mapping "dresses with pockets" to the right call
- Measure the weekly resolution rate against a target

Why others wrong:
- B: Hiring more agents works against the case goal of reducing call-centre cost.
- C: Switching products is a large change that does not address the root cause (no structured access to the catalogue). Dialogflow CX (now part of Conversational Agents) is not deprecated and can also call tools/webhooks.
- D: A bigger model does not give the agent a way to filter or sort the catalogue; it costs more and adds latency.

---

**S3. EHR Healthcare — Insurer API latency spike**

EHR's on-prem insurer API (eligibility checks) has a p99 latency of 200 ms. After moving to GCP (Apigee → Cloud Run → backend insurer API via Dedicated Interconnect), p99 latency is now 500 ms. This breaks EHR's clinic-check-in flow (target: <300 ms). What should they investigate?

A. Increase Apigee provisioning (more nodes); scale up Cloud Run autoscaler.  
B. Measure latency on each hop (Apigee, Cloud Run, Interconnect path, insurer endpoint) with tracing and Network Intelligence Center, and check region placement relative to the Interconnect attachment.  
C. Reduce Cloud Run timeout (forces fail-fast); implement client-side caching of eligibility checks (cache locally for 5 minutes).  
D. Add Cloud CDN to cache eligibility responses; set TTL = 5 minutes.

**Answer: B.** Latency increased by 2.5x; likely culprit is network or code. **B** is the diagnostic approach:
- Measure network latency along the path (Network Intelligence Center Performance Dashboard / Connectivity Tests, Cloud Monitoring Interconnect and VLAN attachment metrics, and a test from a VM to the on-prem insurer endpoint). Note that `gcloud compute interconnects describe` shows configuration and state, not latency
- Trace the request path (Cloud Trace and Cloud Logging, including Apigee analytics/trace) to see where time is spent: Apigee policies, Cloud Run, network, or the insurer endpoint
- Check the Interconnect location/region: if Apigee/Cloud Run run in a region far from the Interconnect attachment, the extra distance adds latency; co-locating them helps
- If code or call patterns are the issue: reduce sequential calls to the insurer, reuse connections, and add caching only where the data's freshness rules allow it

Why others wrong:
- A: Scaling Apigee/Cloud Run helps with concurrency, not the latency of individual requests. The problem is not throughput.
- C: Fail-fast doesn't solve latency; it just fails faster. Caching may help in some cases, but it doesn't address the root cause, and eligibility data may be time-sensitive.
- D: Cloud CDN is built for caching static or cacheable web content at the edge; per-patient eligibility responses contain sensitive data and should not be cached publicly. It does not address the root cause.

---

**S4. KnightMotives Automotive — Vehicle telemetry backlog**

(Hypothetical numbers for illustration.) KnightMotives' Pub/Sub → Dataflow telemetry pipeline is falling behind. Roughly 58,000 connected vehicles × 10 events/min ≈ 835 million events/day arrive, but the pipeline is only processing about 500 million events/day, so the Pub/Sub backlog keeps growing and the Firestore dashboard shows increasingly stale data. What should they do first?

A. Let Dataflow autoscale on the Pub/Sub backlog and raise the maximum worker count; find and fix bottlenecks (hot keys, slow sinks such as Firestore write limits).  
B. Use Pub/Sub publisher batching to reduce API calls and increase throughput.  
C. Reduce the event volume only by changing in-vehicle firmware to aggregate before publishing, and leave the pipeline as is.  
D. Switch from Dataflow to self-managed Spark on GKE.

**Answer: A.** Backlog grows when the consumer is slower than the producers, and Pub/Sub buffers the difference. **A** fixes the consumer side with the least change:
- Dataflow streaming autoscaling adds workers when the backlog grows; the cap (max workers) and per-worker resources may be what limits throughput now
- Check for pipeline bottlenecks: skewed keys (e.g., one hot vehicle_id), expensive per-element work, and sink write limits
- Once caught up, the backlog drains because capacity exceeds the arrival rate
- Edge aggregation (summaries + anomalies instead of every raw event) is a good complementary long-term measure to cut volume and cost, but it needs firmware changes and OTA rollout and does not fix the pipeline today, so it is not the first action

Why others wrong:
- B: Publisher batching makes publishing more efficient but does not raise the consumer (Dataflow) throughput that is the bottleneck.
- C: As a sole fix it is slow (firmware rollout), risky, and loses raw data detail; the pipeline stays under-provisioned in the meantime.
- D: Self-managed Spark adds operational burden and is not inherently faster; Dataflow is the managed stream-processing service.

---

**S5. Cymbal Retail — Image generation cost spiral**

(Hypothetical numbers for illustration; check current pricing.) Cymbal uses a Google image-generation model to create product images (say 10 million products × 3 images = 30 million generations per month). At an assumed price of $0.04 per image, that is about $1.2M/month, which exceeds the budget. How do they reduce cost without cutting the feature? ⚠ verify (prices are illustrative, not current list prices)

A. Switch to a self-hosted open-source image model, because it is free.  
B. Always generate only 1 image per product instead of 3.  
C. Mix stock photos, a third-party image model and Google's model for different image types.  
D. Tier generation by product value (3 images for new arrivals and bestsellers, 1 for the long tail), use a lower-cost model variant where quality allows, avoid regenerating unchanged products, and negotiate committed pricing with Google.

**Answer: D.** It cuts volume where it matters least, lowers unit price, and keeps quality where it earns revenue:
- Tier by value: spend generations on products that drive revenue
- Use a cheaper model tier (or lower resolution) for low-value images, after a quality check with HITL review
- Cache and skip regeneration for unchanged products
- Ask Google about committed-spend or volume pricing (a negotiation, not a published self-serve discount)
- Result: cost falls without removing the feature

Why others wrong:
- A: "Free" is wrong: self-hosting needs GPUs/TPUs, engineering and operations, and quality may be lower.
- B: A blanket cut removes the feature value for the products that benefit most from multiple images; D reduces volume selectively.
- C: Mixing multiple vendors adds complexity, licensing and data-handling concerns, and does not clearly reduce cost.

---

**S6. Altostrat Media — Hybrid ingestion failover**

(Hypothetical numbers for illustration.) Altostrat's single Dedicated Interconnect (10 Gbps) fails (fibre cut), so the on-prem ingestion pipeline cannot reach Cloud Storage over it. A backup Cloud VPN exists whose usable throughput is about 3 Gbps. The on-prem site normally ingests close to the full 10 Gbps. How should they respond, and how should they have designed it?

A. Wait for the Interconnect repair (4-hour ETA); ingestion is blocked.  
B. Fail over to the Cloud VPN immediately; on-prem throttles and prioritises ingestion to fit about 3 Gbps (roughly 30 % of normal), queues the rest locally, and drains the queue after recovery. Long term, add redundant Interconnect attachments and more HA VPN tunnels.  
C. Batch the queued data and compress it lossily so it fits the VPN; ship the rest via Transfer Appliance as a backup.  
D. Use Storage Transfer Service for the failover, because it is designed for real-time failover.

**Answer: B.** Degraded service beats an outage, and the design should avoid a single point of failure. **B** is the pragmatic approach:
- Fail over to Cloud VPN (pre-configured failover route via Cloud Router, with BGP)
- Prioritise time-sensitive content; queue older content locally and ingest it when the Interconnect recovers
- Long term: redundant Dedicated Interconnect attachments in separate edge availability domains/locations for the production SLA; HA VPN throughput scales by adding tunnels (each tunnel has a per-tunnel bandwidth limit)
- Result: some data flows, critical content is captured, and no data is lost

Why others wrong:
- A: Waiting leaves content stuck on-prem and risks missing time-sensitive content.
- C: Lossy compression is unacceptable for media masters; Transfer Appliance is for bulk offline migration, not failover (shipping takes days).
- D: Storage Transfer Service moves data in bulk or on a schedule; it is not a network failover mechanism and still needs a working network path.

---

## Retired case studies (not on 2026 exam)

These case studies appeared on older versions of the exam (earlier years; exact dates not verified here). Per current exam-prep sources, the current case pool is the four above (plus a renewal-exam pool), so these should not appear on the 2026 exam. Do NOT spend time studying them. Descriptions below are brief reminders from the older case studies:

- **Mountkirk Games:** Multiplayer mobile gaming backend; global autoscaling; low-latency game state.
- **TerramEarth:** Heavy equipment for mining and agriculture; vehicle telemetry from a connected fleet; legacy system integration.
- **Helicopter Racing League (HRL):** Live race streaming and predictive fan engagement; global audience; video and ML workloads.
- **Dress4Win:** Online fashion/social e-commerce platform moving from on-prem to the cloud.

The EHR Healthcare case carried over from the older pool, so it is still current. The concepts in the retired cases (telemetry, streaming, migration) are not tested as case-specific facts, but the same general services appear in the current cases.

---

## Sources

Official Google Cloud case study PDFs (v6.1, as of exam guide publication):
- [Altostrat Media Case Study](https://services.google.com/fh/files/misc/v6.1_pca_altostrat_media_case_study_english.pdf)
- [Cymbal Retail Case Study](https://services.google.com/fh/files/misc/v6.1_pca_cymbal_retail_case_study_english.pdf)
- [EHR Healthcare Case Study](https://services.google.com/fh/files/misc/v6.1_pca_ehr_healthcare_case_study_english.pdf)
- [KnightMotives Automotive Case Study](https://services.google.com/fh/files/misc/v6.1_pca_knightmotives_automotive_case_study_english.pdf)

Official resources:
- [Google Cloud Professional Cloud Architect Exam Guide v6.1](https://cloud.google.com/learn/certification/guides/professional-cloud-architect)
- [Google Cloud Well-Architected Framework](https://cloud.google.com/architecture/framework)
- [Google Cloud Architecture Center](https://cloud.google.com/architecture)

Community intel:
- gcpstudyhub.com — 2026 exam guide to case studies
- Medium / Cloudbabble — renewal exam write-ups (2026)
- Reddit r/googlecloud — "Passed GCP Professional Cloud Architect (New Syllabus)" threads

Cross-references:
- → Ch01 Exam strategy (how to read case studies split-screen)
- → Ch02 HA/DR/SLOs (regional HA, cross-region DR, replica lag)
- → Ch03 Networking (Dedicated Interconnect, Cloud VPN, Apigee)
- → Ch04 Compute (GKE, GKE Enterprise, Cloud Run functions)
- → Ch05 Storage & Databases (Autoclass, AlloyDB, Spanner, Firestore, BigQuery)
- → Ch06 Data processing (Pub/Sub, Dataflow, Cloud Composer, BigQuery ML)
- → Ch07 Migration (Database Migration Service, Migration Center, Datastream)
- → Ch08 AI/Agent Platform (Gemini, Gemini image models, Document AI, Agent Search, Conversational Agents, fine-tuning, RAG)
- → Ch09 AI Infrastructure (GPUs/TPUs for training and simulation, LiteRT on-device inference, cost/perf optimisation)
- → Ch10 IAM (Workforce Identity Federation, GCDS + SAML, Workload Identity Federation for workloads, service accounts)
- → Ch11 Data Security (CMEK, Sensitive Data Protection, VPC Service Controls, audit logs)
- → Ch12 Compliance (HIPAA BAA, GDPR, data residency, organisation policy)
- → Ch13 CI/CD/IaC (Cloud Deploy, Artifact Registry, deployment strategies)
- → Ch14 FinOps (Autoclass, call-centre and data-centre cost reduction, committed use discounts)
- → Ch15 Operations/Reliability (Managed Prometheus, alerting policies, SLO burn rate, Cloud Logging, Cloud Audit Logs)
