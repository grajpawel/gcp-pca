# Chapter 08 — AI on Google Cloud: Agent Platform, Gemini and Model Garden

## Exam objectives covered

From the 2026 exam guide (v6.1):
- **1.3 Network, storage, compute:** "Google Cloud AI/ML solutions (Gemini LLMs, Agent Builder, Model Garden, Gemini models, AI Hypercomputer)"
- **2.4 (NEW) Gemini Enterprise Agent Platform for end-to-end ML workflows:** "Agent Platform Pipelines to automate/orchestrate ML lifecycle; preparing for Agent Platform data integration; AI Hypercomputer (Cloud Run functions + Agent Platform for ML/AI workloads; GPUs & TPUs in training and serving; optimizing for different consumption models; large-scale AI training)"
- **2.5 (NEW) Prebuilt solutions or APIs with Agent Platform:** "Differentiate Google AI APIs (Search, Conversation, Vision, Image, Video, Audio); integrate Gemini Enterprise features (AI Agents, NotebookLM); integrate AI models from Model Garden"

## What the exam really tests

- **Model selection logic:** Which Gemini model tier (Pro, Flash, Flash-Lite; thinking is a configurable capability, not a separate model; multimodal) for latency/cost/context constraints; when 1M tokens matter; open models (Gemma) vs proprietary
- **Agent Platform vs discrete services:** When to build multi-step agents vs single-API calls; Agent Runtime (formerly Agent Engine) for long-running ops; Sessions / Memory Bank for stateful interactions
- **RAG decision tree:** RAG Engine vs Vector Search vs Agent Search vs grounding with Google Search vs BigQuery vector; hyperparameter sensitivity (retrieval quality vs cost)
- **Pipelines for automation:** Agent Platform Pipelines (Kubeflow/TFX-based) for ML lifecycle (data → training → evaluation → deployment); metadata lineage; Feature Store integration
- **Pre-built API placement:** Vision, Speech, Document AI, Translation, NL API for domain-specific tasks; Imagen/Veo for generative media; CCAI/Conversation for customer interactions
- **Endpoints & deployment:** Online vs batch prediction; autoscaling; custom containers; custom prediction routines; cost (Provisioned Throughput vs pay-as-you-go tiers vs batch, for Gemini models)
- **Responsible AI & governance:** Prompt injection in agents; data exfiltration; Model Armor; Sensitive Data Protection on inputs; audit trails for agent decisions

## Core concepts

### Gemini Enterprise Agent Platform (formerly Vertex AI)

Gemini Enterprise Agent Platform (April 2026 rebrand from Vertex AI) is Google's end-to-end platform for building, deploying, scaling, and governing agents and ML systems. Key name changes:
- **Vertex AI Studio** → **Agent Studio** (prompt design, tuning, testing)
- **Agent Engine** → **Agent Runtime** (managed runtime for agents; long-running ops up to 7 days; sub-second cold starts; custom containers)
- **Vertex AI Pipelines** → **Agent Platform Pipelines** (orchestration, scheduling, lineage, metadata)
- **Sessions / Memory Bank** → **Agent Platform Sessions / Agent Platform Memory Bank** (stateful conversation context, multi-turn interactions)
- **Model Garden** (renamed from Vertex AI Model Garden; access to 200+ first-party, partner, and open-source models)
- **Vertex AI Workbench** → **Agent Platform Workbench** (still a separate, customizable JupyterLab service; **Colab Enterprise** is the managed, collaborative notebook option, part of Agent Platform and embedded in BigQuery)
- **Vertex AI Search** → **Agent Search**; Vertex AI Feature Store / Model Registry / AutoML → Agent Platform Feature Store / Model Registry / AutoML

### Gemini model family & selection

| Model | Context | Latency | Cost | Key Use Cases |
|-------|---------|---------|------|---|
| **Gemini Pro tier** (version numbers change often: check the current model list) | up to 1M tokens | highest | $$$$ | Complex reasoning, multi-step tasks, highest accuracy |
| **Gemini Flash tier** | up to 1M tokens | low | $$ | Fast general-purpose, balanced quality/speed; agent backbone, coding |
| **Gemini Flash-Lite tier** | up to 1M tokens | lowest | $ | High-volume, latency- and cost-sensitive traffic: classification, simple QA, routing |
| **Thinking (capability, not a separate model)** | per model | adds latency and output tokens | adds cost | Gemini 2.5/3.x models reason before answering; thinking budget/level is configurable per request (more thinking = more latency and cost) |
| **Gemini 2.5 Pro / Flash / Flash-Lite (legacy)** | 1M tokens | varies | varies | Scheduled for retirement in Oct 2026 per the model-lifecycle page; migrate to 3.x |
| **Gemini Live API** | — | real-time (streaming) | — | Real-time voice/video agent conversations |
| **Gemini Omni Flash (preview)** | multimodal in (text, image, audio, video) | — | — | Video generation and conversational video editing; Veo 3.1 remains for scene extension / last-frame control |
| **Imagen / Gemini image models (Nano Banana) / Veo 3.1** | N/A | seconds to minutes | $$$ | Image generation and editing, video synthesis (Veo 2 and Veo 3.0 endpoints retired or retiring) |

Latency and price tiers are relative; exact figures change often and are not exam material (check the pricing page).

**Decision logic:**
- **Reasoning-heavy (multi-step planning, math, code generation):** Pro tier, or Flash with a higher thinking budget
- **Agent backbone (low-latency loops, repeated calls):** Flash or Flash-Lite
- **Latency critical, simple tasks (classification, routing):** Flash-Lite
- **Context-sensitive (10k–1M tokens):** use Flash or Pro depending on reasoning complexity
- **Multimodal (vision + audio in agents):** Flash or Pro tier (natively multimodal); Live API for real-time voice; Omni for video generation

### Model Garden

**Model Garden** (formerly Vertex AI Model Garden) is a hub for 200+ models:
- **First-party:** Gemini, Imagen, Veo, Lyria (music), embeddings, Chirp (speech models in Speech-to-Text / Text-to-Speech)
- **Partners (MaaS managed APIs):** Anthropic Claude, Mistral, xAI Grok, Meta models, and more
- **Open-source / open-weight:** Gemma, Llama, DeepSeek, Hugging Face models, and more

**Deployment options in Model Garden:**
1. **MaaS (Models as a Service):** Serverless managed API for partner and open models; no infrastructure to provision; pay-per-use; requests still go to Agent Platform endpoints
2. **Agent Platform Endpoints (one-click):** Managed online endpoints; autoscaling; custom containers (Artifact Registry); custom prediction routines
3. **Self-deploy:** Bring your own infrastructure (GKE, Cloud Run, Compute Engine)

**Cost/Performance tradeoff:**
- **MaaS:** Zero ops overhead; ideal for proof-of-concept, variable load
- **Endpoint (self-deployed open model):** You pay for the provisioned accelerator nodes while deployed; lower per-request cost at sustained high utilisation; best for production with predictable QPS or when you need custom weights/containers
- **Self-deploy:** Full control, highest operational burden

### Agent Studio & Agent Builder suite

**Agent Studio** (formerly Vertex AI Studio) is the low-code visual workspace for discovering models, prompt design, tuning, testing and building agents without writing code.

**Agent Builder** (now part of Agent Platform) is the set of tools for building agents; three development paths:
- **Agent Studio (low-code):** Visual workspace for rapid prototyping and business-centric agents
- **Agent Development Kit (ADK):** Open-source, code-first framework (Python, Go and others) for complex multi-agent orchestration with granular control over logic and tools
- **Managed Agents API (managed code):** Config-driven, REST-first API for building autonomous agents in a managed sandbox
- **Agent Garden:** A library of prebuilt agent samples/templates that accelerate common patterns (not a drag-and-drop builder)
- **A2A Protocol:** Open Agent2Agent standard (donated to the Linux Foundation) for agents to delegate tasks to other agents; complements MCP (agent-to-tool)
- **Govern pillar:** Agent Registry (central catalogue of agents and MCP servers), Agent Identity (unique SPIFFE-style ID usable in IAM), Agent Gateway (enforcement point for agent traffic, with Model Armor inspection)

### Agent Runtime (formerly Agent Engine)

Managed runtime for long-lived agents:
- **Long-running operations:** Up to 7 days (vs Cloud Run services: 60 min request timeout)
- **Sub-second cold starts**; deploy agents built with ADK or other frameworks, including custom containers (from Artifact Registry)
- **Stateless or stateful:** Pair with Sessions/Memory Bank for multi-turn conversations
- **Ideal for:** Async workflows, background jobs, 24/7 orchestration tasks

### Agent Platform Sessions & Memory Bank

**Agent Platform Sessions** (formerly Vertex AI Agent Builder Sessions):
- Short-term memory: keeps the history of interactions between a user and an agent **within a single conversation**
- Provides the context for the ongoing interaction and is a source for generating long-term memory

**Agent Platform Memory Bank**:
- Long-term memory: **extracts, stores and retrieves personalised information about a user across multiple sessions**
- Enables personalisation and cross-session continuity (e.g. remembers preferences from last week's chat)
- Exam shortcut: Sessions = this conversation; Memory Bank = across conversations

### RAG & Grounding options

| Option | Latency | Accuracy | Setup | Cost |
|--------|---------|----------|-------|------|
| **RAG Engine** | Managed orchestration (retrieve, rerank, generate) | High (controllable sources) | Managed framework: ingest from Cloud Storage/Drive, chunking, embedding; RAG-managed vector DB (Spanner-backed) or bring your own (e.g. Vector Search) | RAG-managed DB + embeddings + LLM calls |
| **Vector Search** | Very low (ANN at large scale) | Medium–high (embedding/index quality) | You generate embeddings and build/manage the index; a building block for RAG | Index storage and serving |
| **Agent Search** (formerly Vertex AI Search) | Low | Medium–high (Google-quality search over your data) | Fully managed search/answer engine over websites, unstructured and structured data, with connectors | Per query + indexed storage |
| **Grounding with Google Search** | Adds a web search call | Medium (public web) | Enable as a tool on the Gemini call; no data store to build | Billed per grounded prompt/query beyond free quota ⚠ verify |
| **BigQuery Vector Search** | Moderate (analytical) | High (structured data already in BigQuery) | SQL (VECTOR_SEARCH) over embeddings in BigQuery | BigQuery compute pricing |
| **AlloyDB Vector** (pgvector / ScaNN index) | Low | High (structured, transactional, real-time) | Managed PostgreSQL | Instance pricing |

**Decision flow:**
- **Private knowledge base, strict data governance:** RAG Engine with internal document sources (or Agent Search for a ready-made enterprise search experience)
- **Very low-latency retrieval at very large vector scale / custom retrieval pipeline:** Vector Search (you own embeddings and index)
- **Broad web context, low setup:** Google Search Grounding
- **Analytical queries (product catalog, financial data):** BigQuery Vector + SQL grounding
- **Real-time updates, transactional consistency:** AlloyDB pgvector

### Function calling & MCP servers

**Function calling:**
- Gemini models can decide to invoke agent functions instead of generating text
- The model only returns the function name and arguments; **your application (or the agent runtime) executes the function** and returns the result to the model
- Asynchronous function calling in Live API (preview ⚠ verify): lets the model keep talking while a tool runs

**Model Context Protocol (MCP) servers** (open standard; GA status of individual Google MCP servers varies ⚠ verify):
- Standardized bridge between agents and external tools (APIs, databases, services)
- Google Cloud offers managed MCP servers for its services; Agent Registry (and Cloud API Registry / Apigee) catalogue MCP servers and tools for agents
- Apigee can expose REST APIs as MCP tools; agents consume them through a single governed interface
- MCP = agent-to-tool; A2A = agent-to-agent delegation (different layers)

### Agent Platform Pipelines

Orchestrate end-to-end ML workflows (training, evaluation, deployment, monitoring):
- **Framework:** Kubeflow Pipelines or TensorFlow Extended (TFX)
- **Execution:** Serverless; managed scaling; no cluster provisioning
- **Scheduling:** Cron-based or event-triggered
- **Lineage & metadata:** Track data → model → evaluation → endpoint deployments
- **Integration points:** BigQuery / Cloud Storage / Dataflow (data), Feature Store (features), Model Registry (versioning), Endpoints (deployment); prebuilt components via the Google Cloud Pipeline Components SDK
- **Runs on:** Pipelines defined with the KFP (or TFX) SDK, compiled and executed by the managed service (you do not run a Kubeflow cluster)

**Typical pipeline DAG:**
```
Data (BigQuery/Cloud Storage) 
  → Preprocessing (Dataflow/custom container) 
  → Feature Store (managed features) 
  → Training (Agent Platform custom training job or custom container) 
  → Evaluation (metrics, bias detection) 
  → Conditional approval (human-in-the-loop) 
  → Model Registry (register best run) 
  → Endpoint deployment (online/batch serving)
```

### Feature Store, Model Registry, Endpoints

**Feature Store:**
- Centralized feature engineering and versioning
- Real-time (online) and batch (offline) serving
- **Current Feature Store uses BigQuery as the offline store** (register BigQuery feature groups/features; no copy/import to a separate offline store) and serves online from a managed online store
- **Legacy (V1) Feature Store and its "Optimized online serving" are deprecated**; use the BigQuery-based Feature Store for new work
- Data consistency across training & serving (avoids training-serving skew)
- Integration with Pipelines for automated feature refresh

**Model Registry:**
- Version and document all trained models
- Metadata: hyperparameters, training data, evaluation metrics, lineage
- Aliases (prod, staging, canary); promote versions
- Model cards for transparency (input/output schema, performance splits, limitations)

**Agent Platform Endpoints:**
- **Online (real-time):** Synchronous predictions; autoscaling (0–100+ replicas); traffic splitting (canary, A/B)
- **Batch:** Asynchronous, cost-effective for large datasets; Cloud Storage → predictions → Cloud Storage
- **Custom containers:** Any framework (TensorFlow, PyTorch, scikit-learn, custom logic)
- **Custom prediction routines:** Pre/post-processing logic (e.g., image resizing, feature normalization) in handler code
- **Provisioned Throughput (GA, Gemini and supported partner models only):** Reserved capacity bought in GSUs (generative AI scale units) for a term; predictable latency/throughput; it does NOT apply to custom-trained model endpoints (those bill for the provisioned nodes)

### Custom training & AutoML

**Custom training:**
- Bring your code (Python, custom container with Dockerfile)
- Support for distributed training (data parallelism, model parallelism)
- Hyperparameter tuning: grid search, Bayesian optimization
- GPU/TPU support (→ Ch09)

**AutoML:**
- Minimal code; Google-managed training
- Image, video and tabular data (Text AutoML objectives have been retired; for text use Gemini prompting/tuning) ⚠ verify exact current AutoML data types
- Automatic model/architecture search and hyperparameter tuning
- Lower operational overhead; less control

### Pre-built Google AI APIs

Differentiated from Agent Platform; typically used within agents for specific domains:

| Service | Input | Output | Use Case | Latency |
|---------|-------|--------|----------|---------|
| **Vision API** | Image/PDF | Labels, OCR, safe-search, logo detection | Content moderation, document extraction | Online (sync) or async batch |
| **Video Intelligence** | Video | Scene labels, object tracking, shot detection | Video indexing, content understanding | Mostly async/long-running |
| **Speech-to-Text (STT)** | Audio | Transcription; speaker diarization | Voicebot, accessibility | Streaming or batch |
| **Text-to-Speech (TTS)** | Text | Audio stream | Voicebot, accessibility, podcast generation | Near real-time |
| **Translation API** | Text + language pair | Translated text | Multilingual agents, international workflows | Near real-time |
| **Document AI** | Document (PDF, image) | Structured extraction (invoices, contracts, forms) | Process automation, compliance | Online or batch processing |
| **Natural Language API** | Text | Entity extraction, sentiment, syntax | Content analysis, intent detection | Near real-time |
| **Conversational Agents / Contact Center AI (CCAI)** | User utterance | Intent, entities, response routing | Customer service agents, IVR replacement | Real-time conversation |
| **Search for commerce (formerly Discovery AI / Retail Search)** ⚠ verify current name | Query + catalog | Ranked results, personalisation | Conversational commerce, product discovery | Real-time |

**Generative media models (Imagen / Veo / Chirp):**
- **Imagen / Gemini image models (Nano Banana):** Image generation and editing; text-to-image, image-to-image
- **Veo 3.1:** Text/image-to-video generation with native audio (Veo 2.0 and 3.0 endpoints are deprecated)
- **Chirp 3:** Speech models for STT and high-definition TTS voices; multilingual

### Gemini Enterprise: AI Agents, NotebookLM, Workspace Studio

**Gemini Enterprise** is the umbrella knowledge-worker product (Agentspace was rebranded to Gemini Enterprise); it is a separate product from the Agent Platform developer/ops stack:

**Gemini Enterprise App:**
- Front door for enterprise AI; AI agent and Gemini access for every employee
- Integrated with organizational data (connectors to Workspace, Microsoft 365, Salesforce etc.; federated search across enterprise apps) ⚠ verify connector list
- Web and mobile access

**AI Agents in Gemini Enterprise:**
- Pre-built agents (e.g. research-style agents) plus a no-code agent designer for business users ⚠ verify current agent names
- Custom agents built with ADK on Agent Platform can be registered and surfaced to employees
- Admin controls and data-protection features (access controls, CMEK, VPC-SC support; check current docs) ⚠ verify
- Integration with Google Workspace (Docs, Sheets, Gmail)

**NotebookLM Enterprise** (newer docs may call it "Gemini Notebook Enterprise" ⚠ verify):
- AI-powered research notebook grounded in the sources you upload; summaries, Q&A, audio overviews
- Handles documents, web pages, videos and transcripts as sources
- Enterprise version adds Google Cloud IAM/identity and admin controls (exact audit-log and residency features ⚠ verify)

**Workspace Studio** (no-code agent builder for Workspace) ⚠ verify exact scope and relation to Gemini Enterprise:
- Agent construction for business users without code
- Integration with Workspace apps (Gmail, Drive)

**Colab Enterprise:**
- Managed, collaborative Jupyter-style notebooks on GCP (separate from Agent Platform Workbench; docs tell you to choose between them)
- Built-in Gemini assistance in notebooks
- Serverless runtimes (no cluster provisioning)
- Integration with BigQuery and Cloud Storage (also reachable from BigQuery Studio)

### BigQuery ML (BQML)

Create and deploy ML models with SQL; no Python/TensorFlow required:
- **Supported model types:** Linear/logistic regression, time-series forecasting, clustering, deep neural networks, boosted trees (XGBoost), matrix factorization, plus remote models over Gemini/Agent Platform endpoints
- **Training data:** Native BigQuery tables
- **Evaluation:** Built-in metrics (precision, recall, RMSE, AUC)
- **Deployment:** Predict in SQL (`ML.PREDICT`); many model types can be registered to Model Registry or deployed to Endpoints ⚠ verify per model type
- **Cost:** BigQuery compute pricing (on-demand or slots) for BQML jobs; remote/hosted models add their own costs ⚠ verify

### Hyperparameter tuning & evaluation

Agent Platform supports distributed hyperparameter tuning for custom training jobs:
- **Search strategies:** Grid search (exhaustive), random search (efficient), Bayesian optimization (smart)
- **Tuning metrics:** Validation loss, AUC, custom metrics exported from training code
- **Parallel trials:** Run multiple trials in parallel across GPUs/TPUs (limits are configurable and quota-bound)
- **Early stopping:** Halt trials with poor early-stage metrics to save costs
- **Conditional hyperparameters:** Change tuning space based on other hyperparameter values

After training, Agent Platform Pipelines can run evaluation jobs:
- **Metrics:** Precision, recall, F1, AUC, RMSE, MAE; confusion matrices; ROC curves
- **Fairness evaluation:** Equalized odds, demographic parity (via model evaluation slices or custom code) ⚠ verify built-in tooling
- **Bias detection:** Slice evaluation (e.g., accuracy by age group, gender)
- **Model Cards:** Automated documentation of model performance splits, limitations, recommendations

In production (Endpoints):
- **Prediction drift detection:** Compare live predictions to training distribution; alert if divergence
- **Feature skew detection:** Compare live feature values to training; indicate data quality issues
- **Evaluation metrics dashboard:** CloudMonitoring dashboard for latency, error rate, throughput

### Custom containers & prediction routines

Agent Platform supports Dockerfile-based custom containers:
- **Training:** Container is invoked by training job; reads training data from Cloud Storage, BigQuery; writes model to Cloud Storage
- **Serving (Endpoint):** Container runs an HTTP server exposing configurable prediction and health routes (you declare them when uploading the model; the same container serves online and batch requests); loads model artifacts from Cloud Storage
- **Benefit:** Use any framework (TensorFlow, PyTorch, scikit-learn, R, Java), custom pre/post-processing logic
- **Best practice:** Keep images small; push to Artifact Registry

## Decision guides

### When to build a multi-step agent vs. single-API call?

```
Agent (Agent Runtime) if:
  ✓ Multi-turn, stateful (user context > 1 query)
  ✓ Conditional logic (if X then call Tool A, else Tool B)
  ✓ Long-running operations (minutes to days; Agent Runtime supports up to 7 days)
  ✓ Parallel function calling (async tool invocation)
  ✓ Human-in-the-loop approval gates

Single API call (Models API) if:
  ✓ Stateless request-response
  ✓ Lowest latency critical
  ✓ Simple task (translate, summarize, classify)
```

### Model selection (Gemini variant)

```
Lowest latency / lowest cost, simple task (classify, route, extract)?
  → Flash-Lite tier

General-purpose, agent backbone, good price/performance?
  → Flash tier

Accuracy > latency, complex multi-step reasoning?
  → Pro tier (latest GA: Gemini 3.1 Pro) with a higher thinking level/budget

Deep reasoning (math, proofs, complex code)?
  → Pro (or Flash) with thinking enabled; thinking is a setting, not a separate model

Real-time voice/video conversation?
  → Live API (Gemini live models)

Video generation / editing?
  → Veo 3.1 or Gemini Omni Flash (preview)

Already using open model or multi-model cost optimization?
  → Model Garden (Llama, Gemma, Claude, Grok)
```

### Endpoints deployment strategy

```
Proof-of-concept, variable/spiky load, no infra to manage?
  → Gemini pay-as-you-go (Standard PayGo) or MaaS for partner/open models

Production Gemini with steady, predictable high volume and latency/throughput guarantees?
  → Provisioned Throughput (GSUs) as baseline, PayGo for overflow

Self-hosted open or custom model with steady traffic?
  → Agent Platform Endpoint (dedicated nodes, autoscaling)

Batch reporting (latency-insensitive)?
  → Batch inference (discounted vs real-time; results typically within about 24 h ⚠ verify)

Custom pre/post-processing logic?
  → Custom container endpoint + custom prediction routine

Strict data residency or custom infrastructure?
  → Self-deploy GKE or Cloud Run
```

### RAG vs. Grounding decision

```
Private knowledge base (internal docs, policies)?
  → RAG Engine (managed, auto-chunking)

Low-latency retrieval over a very large vector index you manage (embedding pipeline → index)?
  → Vector Search

Public web context, low operational overhead?
  → Grounding with Google Search (billed per grounded query beyond free quota ⚠ verify)

Structured data (product catalog, financials)?
  → BigQuery Vector with SQL grounding

Real-time updates, transactional consistency?
  → AlloyDB pgvector
```

## Well-Architected Framework lens

| Pillar | What to do in Agent Platform |
|--------|---|
| **Operational Excellence** | Use Agent Platform Pipelines with lineage/metadata; CloudLogging for agent decision traces; set SLOs on inference latency & availability |
| **Security** | Enable VPC Service Controls for Agent Platform perimeters; use Sensitive Data Protection (redact PII in prompts); Model Armor for prompt injection defense; audit function-calling via Cloud Audit Logs |
| **Reliability** | Design agents with fallback policies (WHEN_IDLE for graceful degradation); Endpoints with autoscaling; multi-region endpoints via traffic splitting |
| **Performance** | Choose Flash-Lite for low-latency paths; use context caching ⚠ verify GA status; batch predictions for non-real-time tasks; leverage TPUs in training (→ Ch09) |
| **Cost Optimization** | Use Flash or Flash-Lite instead of Pro when possible; Provisioned Throughput for steady high-volume Gemini traffic (PayGo for spiky); batch predictions instead of real-time; right-size embedding dimensions in Vector Search |
| **Sustainability** | Prefer Endpoints (managed scaling) over self-managed GKE for inference; use batch jobs for non-urgent workloads; monitor token/compute usage per agent |

## AI/GenAI angle

This entire chapter is AI/ML-focused, so the angle is intrinsic. Key considerations:

- **Responsible AI by design:** Agents inherit model biases; use model cards, evaluation splits (demographic parity, fairness metrics), and human-in-the-loop approval
- **Prompt injection & data exfiltration:** Agents expose function-calling; validate inputs (Sensitive Data Protection), constrain function schemas, audit logs
- **Model transparency & auditability:** Log agent decisions (function calls, reasoning traces); BQML and Model Cards for explainability; prepare audit trails for compliance
- **Cost of reasoning:** Thinking tokens are billed as output tokens, so high thinking budgets raise cost and latency; use only when required; profile agent decision trees to avoid unnecessary LLM calls
- **Latency for agent loops:** Flash-tier models suit low-latency agent loops; use Flash-Lite for classification tasks within agents
- **Data residency in RAG/grounding:** Ensure knowledge base (Vector Search, BigQuery, AlloyDB) is in compliant regions; residency comes from region choice, regional endpoints and Org Policy location constraints; VPC-SC mitigates exfiltration but does not enforce location

## Case-study hooks

### Altostrat Media
- **Agent:** 24/7 conversational AI for podcast/content discovery (Agent Runtime + Memory Bank for session state)
- **RAG:** Ground agent responses in content library (Altostrat catalog via Vector Search or RAG Engine)
- **Pre-built APIs:** Speech-to-Text (podcast transcription), Vision API (metadata extraction from thumbnails), NL API (sentiment of user feedback), Translation API (multilingual support)
- **Pipelines:** Automate transcription → metadata extraction → RAG indexing via an Agent Platform Pipeline (KFP)
- **Model:** Gemini Flash backbone for low-latency responses; Flash-Lite for quick classification (appropriate-content detection)

### Cymbal Retail
- **Agent:** Conversational commerce with product discovery (Gemini + Search for commerce / Agent Search; Agent Runtime for stateful shopping sessions)
- **RAG:** Product catalog enrichment via Agent Platform (catalog data → Vector Search → agent grounding for recommendations)
- **Pre-built APIs:** Imagen / Gemini image models for dynamic product image generation (color variants, background), Vision API for catalog quality checks, NL API for attribute extraction
- **Pipelines:** ETL (Dataflow) → Feature Store (product attributes, pricing) → Endpoints (recommendation model) with human-in-the-loop approval
- **Model:** Gemini Flash for conversational layer; custom container endpoint for recommendation model

### EHR Healthcare
- **Agent:** Clinical decision support or patient intake agent (Gemini Pro for reasoning accuracy; compliance logging)
- **RAG:** Ground agent in HIPAA-compliant knowledge base (medical guidelines, formulary); VPC-SC perimeter around Agent Platform
- **Pipelines:** Data pipeline (EHR sources → BigQuery ML) for patient risk prediction
- **Model:** Gemini Pro (reasoning over patient data); sensitivity to accuracy (healthcare domain)

### KnightMotives Automotive
- **Agent:** In-vehicle AI assistant (real-time, low-latency; cloud Gemini Flash/Flash-Lite over cellular/5G, with an on-device open model such as Gemma for offline fallback ⚠ verify)
- **RAG:** Ground agent in vehicle knowledge base (diagnostics, service history via AlloyDB pgvector for real-time sync)
- **Pre-built APIs:** Speech-to-Text for voice commands; Text-to-Speech for responses; Vision API for damage assessment photos
- **Pipelines:** Fleet telemetry → BigQuery ML (predictive maintenance)
- **Model:** Gemini Flash-Lite (latency-critical, bandwidth-constrained); Pro with thinking for backend analytics (offline)

## Exam traps & keyword → answer

1. **"Multi-turn conversation, stateful"** → Agent Platform Sessions (within a conversation) / Memory Bank (across conversations), not one-off Model.generate() calls
2. **"Lowest latency / lowest cost, simple QA or classification"** → Flash-Lite tier (not Flash or Pro)
3. **"Private knowledge base, no public web"** → RAG Engine or Vector Search, NOT Google Search Grounding
4. **"Batch predictions, non-real-time load"** → Use Batch Endpoint (cheaper), not online Endpoint
5. **"Agent needs to call multiple tools conditionally"** → Function calling, NOT prompt injection
6. **"Audit agent decisions, compliance"** → Cloud Audit Logs + Model Cards, NOT just application logs
7. **"Custom pre-processing (image resize, feature norm) at inference"** → Custom prediction routine on Endpoint container, NOT preprocessing in training
8. **"Long-running async job, up to 7 days"** → Agent Runtime, NOT Cloud Run services (60 min max)
9. **"1M token context, retrieve relevant subset"** → RAG Engine or Vector Search, NOT storing full 1M in prompt (inefficient)
10. **"Generative media (images, videos, audio)"** → Imagen / Gemini image models, Veo 3.1 (and Omni Flash preview), Chirp 3 / Gemini TTS; do NOT assume Gemini is text-only (Gemini image/audio models generate media)
11. **"Prevent prompt injection from user input"** → Model Armor (prompt/response screening) + Sensitive Data Protection + function schema validation + least-privilege tools, NOT just model instruction tuning
12. **"Model selection table in multi-cloud decision"** → Gemini plus partner/open models (Claude, Mistral, Llama, Gemma) all available via Model Garden / MaaS; choose by task, cost and governance

## Quick-reference cheat table

| Component | Old Name | New Name (2026) | Latency | Cost | Best For |
|-----------|----------|-----------------|---------|------|----------|
| **Platform** | Vertex AI | Agent Platform | — | — | End-to-end ML + agents |
| **Prompt tuning UI** | Vertex AI Studio | Agent Studio | — | Free | Prompt engineering |
| **Agent framework** | Agent Builder / ADK | ADK (code-first) + Agent Studio (low-code); Agent Garden = sample library | — | Ops only | Multi-step agents |
| **Agent runtime** | Agent Engine | Agent Runtime | Sub-second cold starts | On-demand | Long-running, up to 7-day ops |
| **Conversation state** | Agent Engine Sessions | Agent Platform Sessions | — | Included ⚠ verify | History within one conversation |
| **Long-term memory** | Memory Bank | Agent Platform Memory Bank | — | Usage-based ⚠ verify | Facts/preferences across sessions |
| **Orchestration** | Vertex AI Pipelines | Agent Platform Pipelines | Serverless | Compute + ops | ML lifecycle automation |
| **Model zoo** | Vertex AI Model Garden | Model Garden | Varies | MaaS pay-per-use | 200+ models |
| **Inference** | Endpoint (Vertex AI) | Agent Platform Endpoint | Online: low; batch: async | Node-hours (custom) / PT or PayGo (Gemini) | Online/batch serving |
| **RAG+retrieval** | Vertex AI RAG Engine | RAG Engine | Moderate | Per usage ⚠ verify | Managed RAG |
| **Vector search** | Matching Engine | Vector Search | Low | Index storage + serving | Fast semantic retrieval |
| **Enterprise search** | Vertex AI Search | Agent Search | Low–moderate | Per query ⚠ verify | Managed search/RAG over enterprise data |
| **Gemini Pro tier** | Gemini Advanced (consumer) | Gemini 3.1 Pro (latest GA) | Higher | $$$$ | Deep reasoning |
| **Gemini Flash tier** | — | Gemini Flash | Low | $$ | Agent backbone, balance |
| **Gemini Flash-Lite tier** | — | Gemini Flash-Lite | Lowest | $ | Low-latency, high-volume tasks |
| **Thinking** | (2.5 thinking models) | Capability of Gemini 2.5/3.x (not a model) | Adds latency | Adds output tokens | Complex reasoning |
| **Live API** | (preview 2025) | Gemini Live API | Real-time streaming | Usage-based ⚠ verify | Real-time voice agents |
| **Feature versioning** | Feature Store (Legacy/V1, deprecated) | Feature Store (BigQuery offline store) | Online serving managed | Managed | Training/serving consistency |
| **Model versioning** | Model Registry | Model Registry | — | Metadata | Promote, alias, audit |
| **Model documentation** | Model Card | Model Card | — | Free | Transparency, evaluation splits |
| **Batch serving** | Batch predict | Batch prediction / batch inference | Async | Discounted ⚠ verify | Non-real-time workloads |
| **Custom inference** | Custom code handler | Custom prediction routine | — | Compute | Pre/post-processing |
| **AutoML** | Vertex AI AutoML | Agent Platform AutoML (Text retired) | Varies | Lower ops cost | Minimal-code training |
| **Speech output** | Text-to-Speech API | TTS / Chirp 3 HD voices | Near real-time | Per character | Voicebot, accessibility |
| **Video understanding** | Video Intelligence API | Video Intelligence | Mostly async | Per minute | Content labeling, tracking |
| **Document parsing** | Document AI | Document AI | Online or batch | Per page | Invoice, contract extraction |
| **Enterprise knowledge-worker assistant** | Agentspace | Gemini Enterprise (NotebookLM Enterprise is a separate research notebook) | — | Subscription ⚠ verify | Employee search, agents, research |
| **Managed notebooks** | Vertex AI Workbench | Agent Platform Workbench (still exists) vs Colab Enterprise (separate, collaborative) | — | Compute | Notebook-based development |

## Practice scenarios

**Scenario 1: Multi-turn customer support agent**

A customer-service team wants a 24/7 chatbot that remembers customer history (order status, returns, preferences) across sessions. The bot should handle 1000 concurrent users, respond in <1 second, and escalate unresolved issues to a human agent. Which architecture do you recommend?

- A) Agent Runtime + Agent Platform Sessions (Memory Bank for customer context) + function calling for order lookup
- B) Stateless Gemini Flash API calls + Cloud Logging for history
- C) Cloud Run service with Gemini integration; store session state in Memorystore
- D) NotebookLM Enterprise with manual context passing

**Answer:** **A**. Agent Platform Sessions keeps the history of the current conversation; Memory Bank provides long-term memory of customer facts/preferences across sessions (the "across sessions" requirement); Agent Runtime is the managed, scalable host; function calling enables order lookups. Option B loses context between turns; C requires custom session management; D is for document research, not agent runtime.

---

**Scenario 2: Image-heavy e-commerce catalog enrichment**

Cymbal Retail has 500k product images and wants to auto-generate titles, descriptions, and color-variant images for fast onboarding. Expected output: metadata in BigQuery, images in Cloud Storage. Latency: batch (12–24 hours acceptable). Which services combo?

- A) Imagen in a real-time online request path + Vision API in parallel
- B) Cloud Run function triggered per image; call Vision + Imagen APIs sequentially
- C) Agent Platform Pipeline (Dataflow for splitting images) → Vision API → BigQuery → Imagen / Gemini image model → Cloud Storage, with human approval gate
- D) BQML image classifier + Imagen, no orchestration

**Answer:** **C**. Pipelines enable orchestration, lineage, and human-in-the-loop approval. Dataflow scales to 500k images; the pipeline DAG can parallelize Vision API calls and image generation. Option A ignores the batch nature (and offers no approval step); B lacks orchestration, retry and lineage; D doesn't include a managed generation workflow.

---

**Scenario 3: RAG for internal policy agent**

KnightMotives HR wants an agent that answers employee questions about benefits, leave policies, and compliance rules. The policy documents change monthly; <2 sec response latency required. Policies are confidential (not public web). Which RAG approach?

- A) Google Search Grounding + Gemini Agent
- B) Vector Search with manual embedding pipeline; agent queries index
- C) BigQuery SQL for policy lookup; no vector embeddings
- D) RAG Engine with documents uploaded to managed pipeline; monthly refresh via Agent Platform Pipelines

**Answer:** **D**. RAG Engine is managed (no infrastructure to run), handles chunking/embedding/retrieval, and supports monthly refresh via Pipelines. (Agent Search is an equally valid managed alternative; it is not offered here.) Private docs rule out Google Search grounding (A). Option B requires a custom embedding pipeline (more ops). C is for structured data, not free-form policy text.

---

**Scenario 4: Model selection for vehicle diagnostics**

An in-vehicle system needs to classify diagnostic error codes with very low latency over unreliable cellular, and must keep working offline (no cloud round-trip). Which model and deployment?

- A) Gemini Pro tier (best accuracy, cloud-based)
- B) A small open model such as Gemma, quantized and deployed on-device, with cloud Gemini Flash-Lite for non-critical enrichment
- C) Gemini Pro with a high thinking budget
- D) Gemini Omni Flash for multimodal input

**Answer:** **B**. Offline capability requires the model to run on the device, and the hosted Gemini models (including Flash-Lite) are cloud APIs, so an open on-device model (Gemma family ⚠ verify device-class support) is the fit; cloud Flash-Lite is the cheap/fast tier for what can tolerate connectivity. Pro (A, C) is cloud-only and over-specced for classification; Omni (D) targets video generation/editing, not error codes.

---

**Scenario 5: Endpoint cost optimization**

Altostrat Media runs a Gemini-powered summarisation feature with a steady, high, around-the-clock request volume plus occasional spikes. They pay Standard pay-as-you-go today and need predictable latency during peaks. What is the best consumption model?

- A) Stay on Standard pay-as-you-go only; no commitment
- B) Batch inference for all traffic; cheapest
- C) Buy Provisioned Throughput for the custom recommendation model endpoint
- D) Provisioned Throughput sized for the steady baseline, with pay-as-you-go absorbing spikes above it

**Answer:** **D**. Provisioned Throughput (GSUs, GA) reserves capacity for predictable latency and cost at steady high volume; overflow beyond reserved capacity can use pay-as-you-go. A offers no capacity guarantee at peaks; B removes real-time responses; C is wrong because Provisioned Throughput applies to Gemini/supported partner models, not custom-trained model endpoints (those bill for node-hours). The break-even point depends on current pricing, so compare using the pricing calculator.

---

**Scenario 6: Responsible AI in agent function calling**

An agent can call functions to look up customer PII (name, SSN, account balance) and reply to users. How do you prevent prompt injection attacks where a user tricks the agent into exposing sensitive data?

- A) Add guardrail prompts ("Do not share PII")
- B) Disable function calling; answer all questions from prompt
- C) Restrict function schemas and tool permissions (least privilege, only non-sensitive fields); screen prompts/responses with Model Armor; use Sensitive Data Protection to redact PII; validate user input
- D) Use a private LLM with custom safety training

**Answer:** **C**. Defense in depth: least-privilege tool schemas limit what can be returned (A is insufficient); Model Armor screens for prompt injection/jailbreaks and sensitive data in prompts and responses; SDP redacts PII; input validation reduces attack surface. Option A is ineffective (prompt-level guardrails bypass via jailbreaks); B removes utility; D is not practical on exam.

---

**Scenario 7: Geographically distributed agents**

EHR Healthcare needs a multi-region HIPAA-compliant system. Agent Platform is deployed in us-central1, but users are in us-east1, europe-west1 and europe-central2 (EU patient data must stay in the EU). How to minimize latency and ensure data residency?

- A) Deploy regional Agent Runtime/endpoints per geography (EU data in EU regions) behind a global load balancer routing by user location; enforce location with Organization Policy resource-location constraints and regional endpoints; VPC Service Controls for exfiltration protection
- B) Replicate Agent Runtime in each region; use Cloud CDN
- C) Single Agent Runtime in us-central1; use Cloud Armor geo-routing
- D) On-premises private agent runtime

**Answer:** **A**. Regional deployments keep EU data and processing in EU regions and put compute near users; Org Policy location constraints and regional endpoints are what enforce residency, while VPC-SC prevents data exfiltration (it does not itself control where data is stored). CDN (B) does not apply to agent state. Armor (C) is a security product, not a latency/residency tool. On-prem (D) gives up the managed service for no stated requirement. For HIPAA, a BAA covering the services used is also required.

---

**Scenario 8: Choosing between Model Garden and custom training**

A Cymbal partner wants to deploy a proprietary recommendation model. They have 1TB of labeled data, custom loss function (not standard sklearn), and need monthly retraining. Model Garden doesn't have their algorithm. Should they:

- A) Use AutoML; Google handles retraining
- B) Custom training container (PyTorch) + Pipelines for monthly retraining; Endpoints for serving
- C) BQML (simpler, no custom code)
- D) Llama fine-tuning via Model Garden

**Answer:** **B**. Custom training with Pipelines is the only option for non-standard algorithms and monthly automation. AutoML (A) does not let you supply a custom loss function or custom algorithm. BQML (C) is limited to supported model types. Model Garden Llama (D) is for LLMs, not recommendation models.

## GenAI reference architectures & patterns

### RAG chatbot architecture

**Pattern:** Private knowledge base → retrieval + grounding → Gemini response + function calling

```
User query 
  → Agent Runtime (Memory Bank for session context) 
  → RAG Engine or Vector Search (retrieve relevant docs) 
  → Grounding with doc snippets in system prompt
  → Gemini Flash (generate response, select function to call)
  → Function calling (lookup order, call support API, store in knowledge base)
  → Cloud Logging (audit decision, sources used)
  → Response to user
```

**Cost optimization:**
- Use the Flash tier (billed per input/output token; check the current pricing page)
- Retrieval adds cost per query (RAG Engine/Vector Search serving and embedding); retrieve fewer, better chunks
- Function call overhead is minimal if schemas are precise
- Context caching can cut repeated-prefix cost ⚠ verify

**Tradeoff:** Managed RAG Engine is faster to build and operate; a self-managed Vector Search index gives more control over latency and scale. The choice depends on freshness and data governance requirements.

### Content moderation pipeline

**Pattern:** Multimodal input (image + text) → Vision + NL API → Gemini → human review

```
User uploads content (image + caption) 
  → Parallel:
     - Vision API (detect violence, adult, unsafe)
     - NL API (extract entities, sentiment, toxicity)
  → If Vision confidence is very high: auto-reject (threshold tuned by the team)
  → Else: Gemini Flash (summarize violations, recommend action)
  → Human reviewer (UI with approve/reject/edit)
  → Decision logged to BigQuery
  → Audit trail in Cloud Audit Logs
```

**Cost optimization:**
- Vision/NL API are priced per unit/item (see pricing pages)
- Call Gemini only for low-confidence cases (skip the LLM for obvious cases)
- Use asynchronous/batch Vision processing for bulk backlogs

**Latency:** Near real-time per item; batch mode acceptable for non-real-time feeds.

### Catalog enrichment with human-in-the-loop

**Pattern (Cymbal Retail):** Supplier data → LLM generation → Human approval → BigQuery → Search index

```
Supplier product data (CSV: SKU, raw description, images) 
  → Dataflow ETL (parse, dedup, validate)
  → Gemini Pro (extract structured attributes, generate category-specific descriptions)
  → Imagen / Gemini image model (generate color variants, backgrounds, lifestyle images)
  → Human reviewer UI (custom UI or Workspace-based review workflow)
     - Approve auto-generated descriptions
     - Modify attributes
     - Select best variant images
  → BigQuery (write approved data)
  → Vector Search (index descriptions for product discovery)
  → Search for commerce / Agent Search (serve ranked results to agent) ⚠ verify product naming
```

**Cost:** Token cost of Pro generation + per-image generation cost + human review labor; use batch processing and generate only the variants you need.

**Latency:** Non-urgent; batch via Pipelines, overnight scheduling acceptable.

### Conversational commerce agent

**Pattern:** User NL query → Agent → Product search (Search for commerce / Agent Search) + Inventory Check → Personalized offer

```
Customer: "Show me red running shoes under $150"
  → Agent Runtime (hosts the agent; Sessions hold the current conversation, Memory Bank holds long-term preferences)
  → Gemini Flash (extract intent: color=red, category=shoes, price<$150)
  → Product search (search catalog: "red running shoes", rank by popularity + relevance)
  → Real-time inventory check (AlloyDB: stock level, warehouse location)
  → Feature Store (retrieve customer segment, discount eligibility)
  → Gemini Flash (personalize explanation: "You qualify for 10% member discount")
  → Function calling: add to cart, apply discount code, check shipping estimate
  → Response: ranked products + offer + delivery date
  → Session logged for feedback loop (A/B test descriptions)
```

**Cost:** Flash token cost, product-search query cost, Feature Store lookup (managed); keep it low by using Flash rather than Pro in the loop.

**Latency:** Interactive (around a second or less is the design goal); critical for user experience (agent loops with Flash not Pro).

### Summarization + podcast generation pipeline

**Pattern:** Long-form content → Gemini Pro → Podcast script → TTS → Distribution

```
Altostrat: Podcast transcript (1–2 hours, ~100k tokens)
  → Gemini Pro tier, e.g. 3.1 Pro (multi-step extraction)
     - Generate outline (themes, speakers, key insights)
     - Create podcast script (conversational, 5–10 min)
     - Extract quotable clips (highlight best moments)
  → Text-to-Speech (Chirp 3 HD voices or Gemini TTS) to synthesize the script
  → Cloud Run function / Transcoder API (assemble audio, add intro/outro music, chapter markers)
  → Cloud Storage (publish to platform)
  → Cloud Logging (track user engagement: duration, skip points, replays)
```

**Cost:** Pro token cost plus TTS per-character cost; batch processing keeps it predictable.

**Latency:** Batch acceptable; overnight processing via Pipelines.

### Anomaly detection with explainability

**Pattern:** Time-series data → AutoML / Custom training → Threshold alert → Model Cards + audit

```
Telemetry stream (vehicle, manufacturing equipment, network metrics)
  → Feature Store (aggregate over 15-min windows)
  → Agent Platform Pipelines
     - Training: Custom container (isolation forest, ARIMA)
     - Evaluation: Slice by equipment type, time-of-day
     - Register: Model Registry with Model Card
       (accuracy by sensor, latency sensitivity, known edge cases)
  → Endpoint (real-time scoring, online serving)
  → Anomaly detected: score > threshold
     - Log to BigQuery
     - Trigger alert (Pub/Sub → Cloud Run function → Slack)
     - Store raw input + model prediction for forensics
  → Post-incident: Gemini Pro (summarize anomaly root cause based on logs)
```

**Cost:** Training compute (on-demand GPU/CPU), Endpoint node-hours, minimal logging.

**Key:** Model Cards for auditability; enables compliance & debugging.

### Data integration strategy for Agent Platform

**Feature Store + Pipelines + Endpoints:**
1. **Offline (batch):** BigQuery is the offline store of the current Feature Store (daily compute, cost-optimized)
2. **Online (real-time):** Feature Store online store, fed from BigQuery (Dataflow can stream fresh data into BigQuery) for low-latency serving at inference
3. **Consistency:** Feature Store ensures train/serve feature parity (avoids skew)
4. **Monitoring:** Feature drift alerts (distribution shift); retraining triggered automatically

**Grounding data sources:**
- **Public web:** Grounding with Google Search (managed; billed per grounded query beyond free quota ⚠ verify)
- **Private docs:** RAG Engine + Cloud Storage (auto-chunking, embedding)
- **Structured data:** BigQuery Vector + SQL queries (product catalog, policies)
- **Real-time:** AlloyDB pgvector or Memorystore Vector (low-latency, transactional)

### Cost / Latency / Quality tradeoff matrix

| Scenario | Model | Retrieval | Latency | Cost |
|----------|-------|-----------|---------|------|
| Simple QA, low accuracy bar | Flash-Lite | None | Lowest | $ |
| Agent backbone, stateless | Flash | Context caching ⚠ verify | Low | $$ |
| Multi-step reasoning | Pro | Vector Search | Higher | $$$$ |
| Deep math / code | Pro with high thinking level | RAG Engine | Highest (thinking time) | $$$$$ |
| Batch analysis, no latency | Any | Batch inference | Async (hours) | $ |
| Real-time with freshness | Flash | Dataflow stream | Low–moderate | $$$ |

**Decision:** Balance on latency SLA × accuracy × cost constraint. Most exam scenarios optimized with Flash + targeted retrieval.

## Sources

- https://docs.cloud.google.com/gemini-enterprise-agent-platform/release-notes (April 2026 rebrand naming, Agent Runtime features)
- https://cloud.google.com/blog/topics/google-cloud-next/welcome-to-google-cloud-next26 (Gemini Enterprise Agent Platform announcement)
- https://cloud.google.com/blog/topics/developers-practitioners/io26-news-for-agent-developers-on-google-cloud (Agent platform architecture layers)
- https://docs.cloud.google.com/gemini-enterprise-agent-platform/build/vector-search/overview (Vector Search, RAG Engine, Pipelines integration)
- Exam guide v6.1 (2026 official syllabus, sections 1.3, 2.4, 2.5)
- https://docs.cloud.google.com/gemini-enterprise-agent-platform/models/google-models (current Gemini models, Omni preview, retirements; QA-checked Oct 2026)
- https://docs.cloud.google.com/gemini-enterprise-agent-platform/models/partner-models/use-partner-models (MaaS partner models)
- https://docs.cloud.google.com/gemini-enterprise-agent-platform/scale/provisioned-throughput (Provisioned Throughput, GSUs)
- https://docs.cloud.google.com/gemini-enterprise-agent-platform/models/flex-paygo (Flex PayGo)
- https://docs.cloud.google.com/gemini-enterprise-agent-platform/agents/overview (Build / Scale / Govern / Optimize, Sessions vs Memory Bank)
- https://docs.cloud.google.com/gemini-enterprise-agent-platform/build/runtime (Agent Runtime, 7-day operations)
- https://docs.cloud.google.com/gemini-enterprise-agent-platform/notebooks/overview (Workbench vs Colab Enterprise)
