# Chapter 09 — AI Infrastructure and Gemini Assist Tools

## Exam objectives covered

From exam guide v6.1:
- **S2.4 Gemini Enterprise Agent Platform for end-to-end ML workflows:** Agent Platform Pipelines to automate/orchestrate ML lifecycle; preparing for Agent Platform data integration; **AI Hypercomputer** (Cloud Run functions + Agent Platform for ML/AI workloads; GPUs & TPUs in training and serving; optimizing for different consumption models; large-scale AI training).
- **S1.2 Technical requirements:** Gemini Cloud Assist.
- **S5.1 Advising dev & ops teams:** Gemini Cloud Assist.

## What the exam really tests

The exam tests **architectural decisions about AI infrastructure under constraints**, not AI model details:
- **When to choose GPU vs TPU vs CPU** for training, inference, or tuning — based on framework compatibility, latency, scale, cost.
- **Consumption models:** when to use on-demand vs Spot vs Dynamic Workload Scheduler (Flex Start vs Calendar) vs reservations vs committed use discounts.
- **Distributed training patterns:** data vs model parallelism, interconnect requirements, storage for high-throughput training (Cloud Storage FUSE vs Managed Lustre).
- **GKE for AI workloads:** GPU/TPU node pools, Kueue + Dynamic Workload Scheduler for queued provisioning, Cluster Director (managed Slurm/Kubernetes clusters for AI/HPC) for large training clusters, GKE Inference Gateway for model-aware serving.
- **Cloud Run for inference:** GPU support, latency vs cost trade-offs, Cloud Run functions in ML pipelines (event-driven preprocessing).
- **Agent Platform endpoints:** prediction machine types, autoscaling, online vs batch inference, cost optimization.
- **Gemini Cloud Assist:** design investigations, troubleshooting, cost/performance optimization across console, infrastructure operations.
- **Gemini Code Assist:** IDE integration, agent mode, code customization.
- **Responsible AI:** explainability, auditability (Altostrat requirement), human-in-the-loop, data governance for AI, EU AI Act compliance basics.

## Core concepts

### AI Hypercomputer: Integrated stack

**AI Hypercomputer** is Google Cloud's integrated supercomputer system combining:
- **Performance-optimized hardware:** TPUs (Trillium = v6e, v5p, v5e, and Ironwood = TPU7x, now GA), GPUs (A4X Max/A4X with NVIDIA GB300/GB200, A4 with B200, A3 Ultra with H200, A3 Mega/High with H100, A2 with A100, G4 with RTX PRO 6000, G2 with L4), specialized CPUs, plus high-performance networking and storage.
- **Open software & frameworks:** JAX, PyTorch, TensorFlow, vLLM, JetStream, MaxText, Ray, Kubeflow, Flux; XLA compiler for TPU optimization.
- **Flexible consumption:** on-demand, Spot, Committed Use Discounts (CUD), Dynamic Workload Scheduler (DWS), reservations.
- **Unified orchestration:** via Agent Platform endpoints, GKE, Cluster Director (managed Slurm/Kubernetes), Agent Platform batch inference, Cloud Run, Compute Engine.

Designed for the full AI lifecycle: training → fine-tuning → inference (including multi-step reasoning for agentic AI).

### GPU vs TPU vs CPU: Decision criteria

| Factor | GPU (A3/A4/G2/G4) | TPU (Trillium v6e / v5p / v5e / Ironwood) | CPU (General) |
|--------|---|---|---|
| **Best for** | Mixed workloads, PyTorch/TensorFlow/vLLM, inference (long context), custom CUDA code | Large-scale LLM training and serving, TPU-native (JAX), cost-efficient at scale | Preprocessing, small-scale serving, CPU-bound tasks |
| **Framework** | PyTorch, TensorFlow, vLLM, any CUDA-compatible | JAX, PyTorch/XLA, TensorFlow (XLA), vLLM on TPU, JetStream | Any framework |
| **Training: data parallelism** | Good (NVLink within a node, RDMA networking across nodes) | Excellent (dedicated ICI interconnect within a pod; Trillium ICI is 800 GBps per chip) | Not suitable |
| **Training: model parallelism** | Good (scales to 100s–1000s of GPUs with RDMA fabrics) | Excellent (pods of 256 chips for Trillium v6e, 8,960 for v5p, 9,216 for Ironwood) | No |
| **Inference throughput** | Excellent (vLLM, bfloat16/FP8) | Excellent (JetStream/vLLM on Trillium and Ironwood) | Limited |
| **Inference latency** | Low; depends on model, batching and hardware (measure it) | Low; depends on model, batching and hardware (measure it) | Fine for small models |
| **Cost per compute** | Moderate | Often better price/performance at scale for supported frameworks | Low |
| **Availability** | Higher (broad inventory, many regions) | Lower (specialized; limited regions, capacity planning needed) | High |

**Decision tree:**
- Training a large LLM (JAX/MaxText or PyTorch/XLA, very large scale) → **TPU (Trillium v6e, v5p, or Ironwood)**; GPUs (A4/A4X) are the choice when you need CUDA/NVIDIA ecosystem compatibility.
- Fine-tuning with custom code or LoRA → **GPU (A3 Ultra/A4, or G2 L4/G4 for small models)** if using PyTorch/CUDA; **TPU v5e** if JAX.
- Inference (latency-sensitive) → **GPU (L4/G4 for smaller models, A3/A4 for large)** or **TPU** where the model is TPU-optimized; benchmark against your latency SLO.
- Inference (cost-optimized, batch) → **TPU v5e**, **GPU L4**, or **Cloud Run + L4 GPU** for bursty traffic.
- Preprocessing, feature engineering → **CPU instances** (e.g. C3/N2).

### Distributed training

**Data parallelism:** Multiple GPU/TPU chips each process a batch shard; gradients synchronize via all-reduce. Works well up to 100s of chips if synchronization is not the bottleneck.

**Model parallelism:** Model split across chips (e.g., layers or attention heads on different chips). Required when a model does not fit in one accelerator's memory or when batch size is constrained. Requires high interconnect bandwidth (TPU ICI within a pod, e.g. 800 GBps per chip on Trillium v6e; GPU NVLink/RDMA).

**Storage for high-throughput training:**
- **Cloud Storage FUSE:** Mounts buckets as a file system; seamless, with caching options (file cache, parallel downloads) for training data and checkpoints; best when data already lives in Cloud Storage.
- **Managed Lustre:** Google-managed parallel file system (Lustre); the recommended parallel file system for AI training, checkpointing and HPC needing sustained high throughput and low latency.
- **Parallelstore:** Earlier Google-managed parallel file system (DAOS-based) for HPC/AI; current status and availability ⚠ verify, prefer Managed Lustre for new designs.

### GKE for AI/ML workloads

**GPU/TPU node pools:** Create dedicated pools with accelerators; GKE handles provisioning and networking.

**Kueue:** Cloud-native job scheduler; integrates with Dynamic Workload Scheduler for "capacity requests" (wait-for-capacity semantics instead of immediate allocation).

**Dynamic Workload Scheduler (DWS):**
- **Flex Start mode:** Submit a capacity request (number of GPUs/TPUs, duration); DWS provisions when capacity becomes available (GPU requests can wait up to 14 days by default, then expire); the job then runs for up to 7 days and you pay only while it runs, at a discount (up to about 53%). Used on GKE via ProvisioningRequest / Kueue (queued provisioning: all nodes at once, for training) or with flex-start node pools that provision incrementally ⚠ verify exact terms.
- **Calendar mode:** Future reservations in calendar mode: book accelerator capacity for a fixed period starting at a known future date (up to 90 days); predictable start for planned training campaigns.
- **Benefits:** Avoid paying for idle hold-time; with Kueue's MultiKueue you can spread queued jobs across multiple GKE clusters/regions to find capacity; reduced cost.

**Cluster Director:** Managed service for deploying and operating large AI/HPC clusters (managed Slurm, and Kubernetes) with topology-aware placement and health monitoring. It is not a multi-cluster GKE job placement tool (that is MultiKueue).

**GKE Inference Gateway:** Gateway API extension that provides model-aware, load-aware routing for LLM inference requests across model-server pods (e.g. vLLM), including LoRA-adapter-aware routing and request prioritization.

### Cloud Run for inference

**Cloud Run services + GPU:** Serve models via containerized endpoints on NVIDIA L4 GPUs; instances scale to zero (no idle cost). Cold starts are slower than CPU-only services because the model must load (typically seconds to tens of seconds ⚠ verify). Best for low-throughput or bursty inference. Cloud Run has no TPU support.

**Cloud Run jobs:** Batch inference; submit, run, exit; pay per job, no long-running container.

**Cloud Run functions + ML pipelines:** Trigger preprocessing (Cloud Storage upload → Pub/Sub → function → BigQuery) or post-processing. Serverless, event-driven, cost-efficient.

### Agent Platform endpoints (formerly Vertex AI Endpoints)

**Prediction machine types:** n1/n2/t2 CPUs, with optional GPUs/TPUs attached.

**Autoscaling:** Based on CPU/accelerator utilization (or request-based metrics); custom-model endpoints typically keep at least 1 replica (scale-to-zero availability depends on the endpoint type ⚠ verify).

**Online predictions:** Synchronous (request–response); latency-sensitive.

**Batch predictions:** Asynchronous; high throughput, lower cost.

**Cost optimization:**
- Use Spot for non-critical inference (substantial discount, typically up to about 60-91% ⚠ verify; can be preempted).
- Right-size machines (start small, monitor utilization; step up if needed).
- Use Compute Engine rightsizing recommendations.
- For steady traffic, consider committed use discounts or reservations on the base machine/accelerator type.

### Consumption models comparison

| Model | Use Case | Economics | Provisioning | Risk |
|-------|----------|-----------|--------------|------|
| **On-demand (hourly)** | Ad-hoc, unpredictable | Pay-as-you-go; no commitment | Instant | Highest cost per hour |
| **Spot (preemptible)** | Batch, fault-tolerant (training, preprocessing) | Deep discount (typically up to about 60-91% ⚠ verify) | Instant if capacity available | Can be preempted at any time; no availability guarantee |
| **Committed Use Discounts (CUD)** | Steady-state compute (inference, long-running) | 1- or 3-year commitments | Instant | Inflexible; commitment even if unused |
| **Reservations** | Capacity guarantee + flexibility | Billed at on-demand rates for the reserved resources whether used or not (CUDs can discount them) | Instant (if available) | Unused reserved capacity is still billed |
| **DWS Flex Start** | Training + fine-tuning with wait tolerance | Discounted (up to about 53%); pay only while the job runs | Wait for capacity (up to 14 days for GPU requests by default); runs up to 7 days | Delays; GPU/TPU only |
| **DWS Calendar** | Predictable batch jobs and planned training campaigns | Future reservation for a fixed period (up to 90 days) | Planned in advance | Inflexible timing |
| **Gemini Standard / Priority / Flex pay-as-you-go** | Variable AI inference / embedding generation | Charged per input/output token (Flex PayGo ~50% cheaper than Standard but with lower priority; Priority costs more for higher reliability) | None | No reserved capacity; throttling/429s possible under load |
| **Gemini Provisioned Throughput** | High-volume, latency-sensitive inference needing reserved capacity | Fixed price per GSU for a chosen term | Order-based | Pay for reserved capacity even if unused |
| **Gemini Batch inference** | Non-real-time inference (overnight, daily) | Discounted vs real-time (about 50% ⚠ verify) | Results typically within about 24 h ⚠ verify | No immediate response |

### Cost & performance optimization

**Training cost optimization:**
- Use Spot instances for fault-tolerant training with frequent checkpointing (deep discount, can be preempted).
- Overlap compute (GPU/TPU) with I/O (Cloud Storage FUSE or Managed Lustre).
- Use DWS Flex Start + Kueue to avoid idle reservation hold-time.
- Right-size batch size and gradient accumulation; larger batches → fewer synchronization steps.

**Inference cost optimization:**
- Choose accelerators by price/performance for your model (TPUs often win at scale for supported frameworks; benchmark).
- Use Cloud Run (L4 GPU) for bursty inference (scale to zero).
- Use Cloud Run functions for preprocessing (avoid container idle).
- Quantize models (bfloat16, FP8, int8) to fit on cheaper hardware (L4 vs A3).
- Use Spot instances for non-critical inference workloads (agent background tasks).
- For Gemini, pick the consumption model that matches the traffic shape: Provisioned Throughput for steady baseline, Standard/Flex PayGo for variable load, Batch for offline.

**Monitoring for cost:**
- Agent Platform Endpoints: monitor per-replica QPS, latency, cost/prediction.
- GKE: Kueue metrics (queue depth, job wait time); Cluster Director cluster health/utilization.
- Cloud Monitoring: GPU/TPU utilization metrics (high sustained utilization is the goal; set your own target).

### Sustainability in AI

- **Carbon-aware placement:** Choose regions with low grid carbon intensity (Google Cloud region carbon data / Carbon Footprint report) for flexible batch jobs.
- **Efficiency metrics:** Model FLOP utilization (MFU); training efficiency (e.g., tokens/GPU-hour); inference efficiency (latency/Watt).
- **Right-sizing:** Smaller models, quantization and higher accelerator utilization reduce energy per request.

---

## Decision guides

### AI infrastructure architecture: Flowchart

```
Does your workload need ML/AI acceleration?
├─ No → Use CPUs (Compute Engine, GKE CPU node pools, App Engine, Cloud Run)
└─ Yes → Is it training?
   ├─ No (inference/serving) → What's your latency requirement?
   │  ├─ Strict, interactive SLA → GPU (L4/G4 small models; A3/A4 large) or TPU, sized and benchmarked against the SLO
   │  ├─ Moderate → GPU or TPU v5e/Trillium sized for throughput per dollar
   │  └─ Offline/batch → Cloud Run jobs + L4 GPU, batch inference, or TPU v5e
   └─ Yes (training) → What's your model size & framework?
      ├─ Very large LLM (JAX/MaxText or PyTorch/XLA) → TPU (Trillium v6e / v5p / Ironwood, multi-host)
      ├─ Large model (PyTorch/CUDA) → GPU A3 Ultra / A4 / A4X clusters + distributed training
      ├─ Fine-tuning with LoRA/QLoRA → single-node GPU (A3 Mega/Ultra, or L4/G4 for smaller models)
      └─ Experimental (small, variable framework) → GPU L4 (dev/test) or DWS Flex Start (cost-optimal)
```

### Consumption model selection

```
Do you know your resource needs in advance?
├─ No, unpredictable → On-demand (hourly) or Spot (if fault-tolerant)
└─ Yes, predictable → Is it a long-running steady workload?
   ├─ No, training job (hours to days) → DWS Flex Start or Spot instance
   ├─ Yes, inference (weeks/months) → CUD (1/3 yr) or Reservation
   └─ Yes, batch with fixed schedule → DWS Calendar or on-demand
```

### Cloud Run vs Agent Platform endpoints: When to use each

| Aspect | Cloud Run | Agent Platform Endpoints |
|--------|-----------|--------------------------|
| **Cold start** | Present when scaling from zero; GPU instances must load the model (seconds or more ⚠ verify); min instances avoid it | Avoided with warm replicas (min 1) |
| **Scale to zero** | Yes | Generally no for custom-model endpoints (min 1 replica) ⚠ verify |
| **Multi-model serving** | One container per service (you can bundle models yourself) | Supported via model co-hosting on an endpoint ⚠ verify |
| **Batch predictions** | Cloud Run jobs (asynchronous) | Native batch prediction |
| **GPU support** | Yes (NVIDIA L4; no TPUs) | Yes (GPU, TPU) |
| **Cost** | Per execution (bursty = cheap) | Per replica-hour (steady = cheap) |
| **Use case** | Bursty inference, event-driven | High-volume, predictable traffic |

---

## Well-Architected Framework lens

| Pillar | What to do in AI infrastructure | How Gemini Assist helps |
|--------|--------------------------------|------------------------|
| **Operational excellence** | Monitor GPU/TPU utilization; set SLOs for model latency/throughput; automate scaling with Kueue/DWS; document training/serving configs. | Gemini Cloud Assist Investigations help troubleshoot slow or failing workloads from logs, metrics and configuration. |
| **Security** | VPC-isolate GPU clusters; use Workload Identity for ML Ops pipelines; audit model access via Cloud Audit Logs; use Sensitive Data Protection on training data. | Cloud Assist can help review configuration against best practices; security detections come from Security Command Center ⚠ verify. |
| **Reliability** | Use multi-region DWS + Kueue for training; replicate model endpoints across regions; use Pod Disruption Budgets on GKE. | Cloud Assist helps diagnose node/Pod failures and suggests remediation. |
| **Cost optimization** | Right-size GPUs/TPUs; use Spot for training; enable autoscaling on inference endpoints; analyze cost per prediction. | Cloud Assist can help analyse spend and suggest optimizations (design/optimize assistance) ⚠ verify depth. |
| **Performance optimization** | Overlap I/O with compute; batch inference requests; use optimized frameworks (JetStream, vLLM); monitor MFU (model FLOP utilization). | Cloud Assist can help interpret performance signals and suggest changes; it does not replace profiling. |
| **Sustainability** | Use carbon-aware placement; optimize model efficiency (tokens/s per Watt); amortize TPU investment over years. | Use the Carbon Footprint report for emissions data; Cloud Assist is not the carbon reporting tool. |

---

## AI/GenAI angle

- **Agentic AI requires low-latency, high-throughput inference:** Multi-step reasoning (chain-of-thought) increases compute demand at inference time. TPUs (Trillium, Ironwood) with JetStream/vLLM and GPUs with vLLM are optimized for this.
- **Model customization via fine-tuning on AI Hypercomputer:** LoRA fine-tuning on Agent Platform or GKE GPUs; Managed Lustre (or Cloud Storage FUSE with caching) speeds checkpoint save/restore.
- **Responsible AI in infrastructure:** Explainability layers require audit-trail logging; Sensitive Data Protection redacts PII from training logs (Altostrat requirement). Fairness evaluation is done with model evaluation tooling and your own metrics, not by Gemini Cloud Assist.
- **Human-in-the-loop:** Cloud Run functions + Firestore enable user feedback loops (e.g., approve/reject generated content); cost-efficient event-driven architecture.
- **Data governance for AI:** BigQuery + Dataplex (Universal Catalog) tag and track training data lineage ⚠ verify naming; Dataflow preprocesses data at scale (it can use GPUs for some steps ⚠ verify).
- **Gemini Cloud Assist:** AI assistant inside Google Cloud for designing, operating and troubleshooting (Investigations analyse logs, metrics and configuration to propose root causes); some features are free for all users and some require Gemini Code Assist Enterprise ⚠ verify current split.
- **Gemini Code Assist:** IDE/CLI coding assistant with Standard and Enterprise editions (Enterprise adds code customization on private repositories and more Google Cloud integrations) and agent mode for multi-step tasks.
- **EU AI Act basics:** risk-based regulation (prohibited, high-risk, limited-risk, minimal-risk); high-risk systems need risk management, logging, human oversight and documentation; obligation dates and scope have been revised (Digital Omnibus) so check current timelines ⚠ verify.

---

## Case-study hooks

**Altostrat Media:**
- Automated metadata extraction (NLP + vision) → Cloud Run functions + GPU preprocessing (batched).
- Inappropriate-content detection → Use Gemini (supervised tuning on Agent Platform) or a custom/open model fine-tuned on GPUs/TPUs; serve Gemini via the API, or a custom model on Cloud Run + L4 GPU.
- **Auditable & explainable AI** → Use Sensitive Data Protection on prompts; log all model calls to BigQuery; use billing export/labels to track cost per inference (Cloud Assist can help analyse spend).
- Hybrid on-prem ingestion → Transfer data from on-prem into Cloud Storage (Storage Transfer Service / Interconnect), then mount with Cloud Storage FUSE on GKE nodes for the training cluster.

**Cymbal Retail:**
- Catalog enrichment (attribute/image generation) → Gemini batch inference for overnight enrichment; cost-optimized.
- **Human-in-the-loop review:** Cloud Run functions + Firestore for approval workflows; Cloud Monitoring/BigQuery metrics track review latency & cost per approved item.
- Conversational product discovery → Gemini via the API (PayGo or Provisioned Throughput) for agent responses; if self-hosting an open model, Cloud Run + L4 GPU; DWS Flex Start for model fine-tuning at night.

**EHR Healthcare:**
- Patient data preprocessing (de-identification via Sensitive Data Protection) → Cloud Run functions.
- Trend prediction model → Train on GPUs/TPUs (DWS Flex Start or Calendar mode + Kueue for capacity; reservations if guaranteed capacity is needed); serve on Agent Platform endpoints with high availability.
- HIPAA compliance → Sign a BAA, use only covered services; VPC-isolated GPU cluster; Workload Identity for access control; audit logs retained per your policy.

**KnightMotives Automotive:**
- Autonomous-driving simulation at scale → Managed Lustre for high-throughput checkpoint I/O; A4 GPU clusters (or Cluster Director managed Slurm/Kubernetes for large clusters).
- Fleet AI models → Train on TPU or GPU; serve on Agent Platform endpoints (TPU/GPU) or GKE for real-time vehicle telemetry inference (Cloud Run has no TPU support).
- EU data protection (GDPR) → EU regions only (e.g. europe-west1 or europe-west4, subject to accelerator availability) enforced with Org Policy location constraints; VPC-SC perimeters; audit log retention per the customer's requirement.

---

## Exam traps & keyword → answer

1. **"Minimize GPU idle time during training"** → Use Spot instances (with checkpointing, fault-tolerant training) or DWS Flex Start (pay only while the job runs, not for reservation hold-time). Not: standard on-demand reservation (waste on unused hours).

2. **"Inference must meet a strict latency SLA"** → Choose accelerator and serving stack (vLLM / JetStream, batching, regional proximity) by benchmarking against the SLO; do not pick purely by generation name.

3. **"Training a very large model, scale-limited by synchronization/interconnect"** → Pick hardware with a high-bandwidth, topology-aware interconnect (TPU pods with ICI, or GPU clusters with RDMA/NVLink such as A4/A4X) and use model/tensor parallelism; do not rely on slow cross-node networking.

4. **"Choose Kueue when … "** → Multi-job orchestration with priority, preemption, and fair-share quotas. Single job? Just use Compute Engine MIG or GKE job. Kueue adds overhead.

5. **"Cost optimization: idle model endpoints"** → Cloud Run (scale to zero) or Agent Platform with autoscaling to 1 replica + reserved capacity for baseline traffic.

6. **"Batch inference vs real-time API"** → Batch: discounted (about 50% ⚠ verify), asynchronous results (target around 24 h ⚠ verify), good for summarization/enrichment. Real-time: immediate, pay-as-you-go or Provisioned Throughput.

7. **"GKE vs Compute Engine for AI training"** → GKE: complex multi-job orchestration (Kueue), distributed training (all-reduce), fault-tolerance. Compute Engine: simple single-job training, custom scripts.

8. **"Storage bottleneck during training (I/O bound, low GPU utilization)"** → Cloud Storage FUSE (seamless but slower) or Managed Lustre (HPC-grade, parallel I/O). Not: Persistent Disk (IOPS throttling).

9. **"Gemini Cloud Assist investigation: 'why did model inference latency increase?'"** → Gemini Cloud Assist Investigations analyse logs, metrics and configuration, form hypotheses (e.g. saturated replicas, resource limits, recent changes) and suggest next steps; you still validate them.

10. **"Responsible AI, explainability requirement"** → Log all model inputs/outputs to BigQuery; use Sensitive Data Protection to redact PII; audit trail for compliance (Altostrat). Not: just use model confidence scores (insufficient for regulatory).

11. **"Multi-region / multi-cluster training capacity"** → Dynamic Workload Scheduler (Flex Start or Calendar) with Kueue + MultiKueue to queue jobs across GKE clusters/regions; Cluster Director is for managed Slurm/Kubernetes AI clusters, not job placement across GKE clusters.

12. **"Fine-tune a large model, budget-constrained"** → LoRA/QLoRA on a smaller GPU (e.g. L4/G4 for modest models) or DWS Flex Start (discounted, wait for capacity) or Spot (cheapest, can be preempted).

---

## Quick-reference cheat table

### GPU/TPU/CPU comparison (exam essentials)

| Hardware | Generations | Best for | Framework | Peak Throughput | Cost/hr | Availability |
|----------|-------------|----------|-----------|-----------------|---------|--------------|
| **TPU Ironwood (TPU7x)** | 7th gen (GA) | Large-scale training and inference | JAX, PyTorch/XLA, vLLM | 4,614 TFLOPS (FP8), 2,307 (bf16), 192 GiB HBM per chip, 9,216-chip pod | $$$$ | Lower |
| **TPU Trillium (v6e)** | 6th gen | LLM training, large-scale inference | JAX, PyTorch/XLA, TF XLA | 918 TFLOPS (bf16) per chip, 32 GB HBM, 256-chip pod | $$$ | Lower |
| **TPU v5p** | 5th gen (high-perf) | Training, multi-host | JAX, TF XLA | 459 TFLOPS (bf16), pods up to 8,960 chips | $$$ | Low |
| **TPU v5e** | 5th gen (economical) | Cost-optimized training, inference | JAX, TF XLA | 197 TFLOPS (bf16), 256-chip pod ⚠ verify | $$ | Medium |
| **GPU A3 Ultra** | 8× NVIDIA H200 per VM | Mixed workloads, PyTorch training | PyTorch, TF, vLLM | H200 141 GB HBM3e per GPU | $$$ | Medium |
| **GPU A3 Mega / A3 High** | 8× NVIDIA H100 per VM | Training, fine-tuning, inference | PyTorch, TF, vLLM | H100 80 GB per GPU | $$$ | Medium |
| **GPU A4 / A4X** | NVIDIA B200 (A4), GB200 (A4X) | Next-gen training/inference | PyTorch, CUDA, vLLM | Blackwell-generation | $$$$ | Lower |
| **GPU G4** | NVIDIA RTX PRO 6000 | Inference, graphics, small-model training | PyTorch, CUDA | — | $$ | Medium |
| **GPU L4 (G2)** | NVIDIA L4 | Inference, fine-tuning, LoRA | PyTorch, TF, vLLM | 24 GB per GPU | $$ | High |
| **CPU (N2/C3)** | Xeon/AMD/Arm | Preprocessing, small models | Any | — | $ | Very High |

### Consumption model decision table

| Model | Cost/hr | Upfront | Min term | Flexibility | Best for |
|-------|---------|---------|----------|------------|----------|
| On-demand | Base | $0 | None | Full | Ad-hoc, unpredictable |
| Spot | Deep discount (typically up to ~60-91% ⚠ verify) | $0 | None | Can interrupt | Batch, training |
| CUD (1 yr) | Discount varies by resource ⚠ verify | No* | 1 yr (or 3 yr) | Low | Steady inference |
| Reservation | Base (reserved resources billed whether used or not) | No | Chosen period (on-demand reservations can be deleted anytime) ⚠ verify | Medium | Capacity guarantee |
| DWS Flex Start | Discounted (up to ~53%) | $0 | Wait for capacity (up to 14 days default for GPU), run up to 7 days | High (wait) | Training, Kueue jobs |
| DWS Calendar | Future reservation (calendar mode) | $0 | Booked period, up to 90 days | None (fixed time) | Scheduled batch / planned campaigns |

---

## Practice scenarios

### Scenario 1: Training latency vs cost trade-off

**Question:** A fintech startup is training a 70B-parameter LLM for fraud detection on a single A3 Ultra VM (8 H200 GPUs). Training is slow at 42% GPU utilization; profiling shows the GPUs are idle waiting on the input pipeline (reading training shards and writing checkpoints from Cloud Storage). Budget is tight. Which change has the highest ROI?

A) Add more H200 GPUs to the same cluster (A3 Ultra, 16 H200s).
B) Rewrite the training stack for TPUs (Trillium) and use DWS Flex Start.
C) Reduce batch size by half to increase gradient frequency.
D) Move data to Managed Lustre (or enable Cloud Storage FUSE caching) and overlap I/O with compute.

**Answer:** D

**Why D is correct:** The stated bottleneck is I/O, not interconnect: a high-throughput parallel file system (Managed Lustre) or Cloud Storage FUSE with file caching and parallel downloads keeps GPUs fed and speeds checkpointing, which raises utilization without buying hardware. A adds cost without fixing the idle time; B is a costly rewrite aimed at the wrong bottleneck; C worsens utilization.

---

### Scenario 2: Model serving at variable load

**Question:** An e-commerce platform serves real-time product recommendations through the Gemini API (Flash tier). Traffic has a high, predictable daytime baseline with sharp short peaks above it, and a quiet night. On Standard pay-as-you-go they see occasional 429/throttling and latency variance at peaks. Which approach best keeps the latency SLA while controlling cost?

A) Switch to Cloud Run with a GPU (scale to 0) and self-host an open model; accept cold starts.
B) Buy Provisioned Throughput sized for the predictable baseline and let pay-as-you-go absorb the peaks above it.
C) Move to GKE + Kueue, queuing requests; the latency SLA becomes "within 2 s".
D) Move all traffic to Gemini batch inference.

**Answer:** B

**Why B is correct:** Provisioned Throughput (GSUs) reserves capacity for the steady baseline, giving predictable latency and avoiding throttling there; pay-as-you-go handles the spikes without paying for reserved peak capacity around the clock. A changes the model and adds cold-start latency; C breaks a real-time SLA (Kueue is for batch jobs); D is asynchronous and cannot serve real-time recommendations (batch suits only the offline enrichment part).

---

### Scenario 3: Distributed training with data residency

**Question:** A pharmaceutical company trains a proprietary drug-interaction model in europe-west4 (GDPR). Data: 50 TB, stored in a Cloud Storage bucket in the same region. Training runs for 2 weeks on a multi-host accelerator cluster in that region, and GPUs/TPUs are underused while waiting on data and checkpoints. Which architecture minimizes regulatory risk and training time?

A) Stream from Cloud Storage directly with no caching or tuning.
B) Use Managed Lustre in the same region as the parallel file system (import data from the bucket, write checkpoints to it); keep everything in-region.
C) Copy data to local SSD on each host; train offline.
D) Use Dataflow to preprocess data to Bigtable; train from Bigtable.

**Answer:** B

**Why B is correct:** Managed Lustre is the recommended parallel file system for AI training and checkpointing, giving high sustained throughput and low latency from a regional service, and keeping the data in the EU region satisfies residency (confirm accelerator availability in the chosen EU region). Cloud Storage FUSE with caching is a reasonable lighter-weight option but A, as stated (no caching/tuning), is the weakest; C (local SSD) is ephemeral and risks losing data on node failure; D (Bigtable) is not a training-data format store and adds complexity.

---

### Scenario 4: Kueue vs Static GKE allocation

**Question:** A research lab runs 5–10 training jobs weekly (duration: 6–48 hrs each, random schedule). Current setup: GKE cluster, 16 × L4 GPU node pool, reserved capacity. Observed: average utilization 30%. Goal: reduce cost without sacrificing job SLA. Should they adopt Kueue + DWS Flex Start?

A) Yes; Kueue enables job prioritization and preemption, reducing wasted capacity.
B) Yes; DWS Flex Start lets jobs wait for capacity, avoiding idle reservation cost.
C) No; the overhead of Kueue job scheduling is not worth it for 5–10 jobs.
D) No; switch to spot VMs instead.

**Answer:** B (or A+B together)

**Why B is correct:** DWS Flex Start (with Kueue/ProvisioningRequest on GKE) provisions capacity for the job and releases it afterwards; the lab pays only while jobs run, at a discount, rather than for idle reserved capacity. The trade-off is waiting for capacity, acceptable for jobs without hard start times (and each request can run up to 7 days, enough for 6-48 h jobs). At 30% utilization, holding reserved capacity is waste. Kueue adds value (prioritization, fair-share). C underestimates the savings. D (Spot) is cheap but can be preempted mid-job; Flex Start capacity is not preempted once running ⚠ verify.

---

### Scenario 5: Responsible AI and cost auditing

**Question:** Altostrat Media is deploying a Gemini Pro-tier model to summarize podcasts. CTO requires (a) auditability of every summary decision, (b) redaction of PII (guest names, email) from logs, (c) cost per summary tracked. Architect's recommendation?

A) Call Gemini through Agent Platform; enable Cloud Audit Logs, log requests/responses to BigQuery with Sensitive Data Protection de-identification applied, and use labels + billing export for cost.
B) Use Cloud Run + custom Gemini API calls; log to BigQuery with DLP masking.
C) Use Agent Studio (GUI) for all summarization; audit trail via console.
D) Use Firestore + Pub/Sub + Cloud Run; manual audit log per request.

**Answer:** A (B is also workable)

**Why A is correct:** Cloud Audit Logs record who called the API (admin/data-access events, if enabled), but they do not capture prompt or response content, and Sensitive Data Protection is not applied automatically; you must log the content you need to BigQuery yourself and de-identify it with SDP. Cost per summary is derived from billing export plus labels or per-request token metadata. B is valid too (more custom code); C is a prototyping tool, not a production audit design; D is over-engineered.

---

### Scenario 6: Inference gateway for multi-model serving

**Question:** KnightMotives self-hosts several LLMs and LoRA-tuned variants (sales, service, inventory assistants) on a single GKE cluster using vLLM on GPU nodes. Combined peak load is high and requests vary a lot in length. Which architecture best improves latency and GPU utilization?

A) Three Agent Platform endpoints (one per model); client SDK routes by model type.
B) GKE Inference Gateway in front of the model-server pods, using model-aware and load-aware routing.
C) Cloud Run (3 services, one per model); Cloud Load Balancer routes by path.
D) A single Kubernetes ClusterIP Service with round-robin load balancing.

**Answer:** B

**Why B is correct:** GKE Inference Gateway extends the Gateway API with inference-specific routing: it routes by model/LoRA adapter name and by backend load (queue depth, KV-cache utilization), and supports request prioritization, which improves tail latency and GPU utilization versus plain round-robin (D). A moves the workloads off the cluster and scales cost per endpoint; C adds cold starts and per-service overhead for GPU LLM serving. ⚠ verify exact feature list.

---

### Scenario 7: Gemini Cloud Assist for troubleshooting

**Question:** An ML ops engineer notices that a GKE cluster (32 × L4 GPU node pool) running inference has GPU utilization 15% but high latency (p99 > 500 ms). Which first step best uses Gemini Cloud Assist?

A) Run the NVIDIA profiler on every GPU kernel before looking at anything else.
B) Start a Gemini Cloud Assist Investigation scoped to the cluster/workload so it correlates metrics (CPU throttling, evictions), Pod logs and recent configuration changes, then validate its hypotheses.
C) Manually review all Kubernetes logs line by line for OOM errors.
D) Increase replica count immediately.

**Answer:** B

**Why B is correct:** Cloud Assist Investigations gather logs, metrics and configuration and test several hypotheses in parallel, which is an efficient first pass when the symptom (low GPU utilization with high latency) points away from the GPU, for example toward CPU-bound pre/post-processing, throttling or evictions. You still verify the findings. A (profiling) is a later step if the GPU turns out to be the bottleneck; C is slow; D adds cost and may not help if each Pod is CPU-bound.

---

### Scenario 8: EU AI Act and model governance

**Question:** KnightMotives deploys an AI system in the EU that the team has classified as high-risk under the AI Act (for example, an AI model used to score driver risk for fleet insurance; note that vehicle-safety AI is largely governed through sectoral type-approval law). High-risk systems require logging, human oversight and documentation. Which GCP architecture supports this for a model-update/decision workflow?

A) Agent Platform + Cloud Run functions for user approval workflows; Firestore for decision audit trail.
B) Agent Platform Pipelines + BigQuery + Cloud Audit Logs; manual approval step before deployment or decision release.
C) Cloud Run + Firestore + Pub/Sub; custom approval UI + audit logs to BigQuery.
D) Agent Platform Endpoints (batch) + Cloud Composer workflow; approval task in workflow.

**Answer:** B (A, C and D are also workable)

**Why B is correct:** Pipelines orchestrate the workflow with lineage/metadata, BigQuery stores inputs/outputs for traceability, and Cloud Audit Logs record who approved or rejected. A human-approval gate can be added to a pipeline or surrounding workflow (exact pause/approve mechanism ⚠ verify). A and C are valid but more custom; D works with an orchestrator but is not more integrated. Dates and scope of AI Act obligations have been revised, so check current timelines ⚠ verify.

---

## Sources

- [cloud.google.com/tpu](https://cloud.google.com/tpu) — Tensor Processing Units: architecture, generations, pricing.
- [cloud.google.com/blog/products/compute/a3-ultra-with-nvidia-h200-gpus-are-ga](https://cloud.google.com/blog/products/compute/a3-ultra-with-nvidia-h200-gpus-are-ga-on-ai-hypercomputer) — A3 Ultra with H200 GPUs on AI Hypercomputer.
- [cloud.google.com/blog/products/compute/whats-new-with-ai-hypercomputer](https://cloud.google.com/blog/products/compute/whats-new-with-ai-hypercomputer) — AI Hypercomputer, DWS Flex Start, Cluster Director.
- [cloud.google.com/blog/products/compute/introducing-dynamic-workload-scheduler](https://cloud.google.com/blog/products/compute/introducing-dynamic-workload-scheduler) — Dynamic Workload Scheduler Flex/Calendar modes, Kueue integration.
- [docs.cloud.google.com/hub/docs/optimize-gemini](https://docs.cloud.google.com/hub/docs/optimize-gemini) — Gemini Cloud Assist cost optimization.
- [cloud.google.com/blog/topics/google-cloud-next/google-cloud-next-2026-wrap-up](https://cloud.google.com/blog/topics/google-cloud-next/google-cloud-next-2026-wrap-up) — Cloud Next 2026 announcements: Gemini Cloud Assist agents, MCP servers.
- [cloud.google.com/blog/products/ai-machine-learning/deploying-llama4-and-deepseek](https://cloud.google.com/blog/products/ai-machine-learning/deploying-llama4-and-deepseek-on-ai-hypercomputer) — LLM serving recipes (JetStream, vLLM) on TPU/GPU.
- [cloud.google.com/blog/products/containers-kubernetes/using-multikueue-to-provision-global-gpu-resources](https://cloud.google.com/blog/products/containers-kubernetes/using-multikueue-to-provision-global-gpu-resources) — MultiKueue for multi-region GPU provisioning.
- [docs.cloud.google.com/kubernetes-engine/docs/how-to/optimize-gke-gemini](https://docs.cloud.google.com/kubernetes-engine/docs/how-to/optimize-gke-gemini) — GKE optimization with Gemini Cloud Assist.
- [cloud.google.com/products/gemini/pricing](https://cloud.google.com/products/gemini/pricing) — Gemini for Google Cloud pricing and feature tiers.
- [cloud.google.com/blog/products/containers-kubernetes/how-we-built-a-130000-node-gke-cluster](https://cloud.google.com/blog/products/containers-kubernetes/how-we-built-a-130000-node-gke-cluster) — GKE at scale, Kueue for job orchestration.
- [cloud.google.com/products/cluster-director](https://cloud.google.com/products/cluster-director) — Cluster Director (managed Slurm/Kubernetes AI and HPC clusters).
- [cloud.google.com/blog/products/compute/managed-slurm-and-other-cluster-director-enhancements](https://cloud.google.com/blog/products/compute/managed-slurm-and-other-cluster-director-enhancements) — Cluster Director enhancements.
- [cloud.google.com/ai-hypercomputer/docs/create/create-overview](https://cloud.google.com/ai-hypercomputer/docs/create/create-overview) — AI Hypercomputer cluster creation options.
- [cloud.google.com/gemini/docs/codeassist/overview](https://cloud.google.com/gemini/docs/codeassist/overview#editions-overview) — Gemini Code Assist editions.
- [docs.cloud.google.com/gemini-enterprise-agent-platform/models/flex-paygo](https://docs.cloud.google.com/gemini-enterprise-agent-platform/models/flex-paygo) — Flex PayGo; Provisioned Throughput at docs.cloud.google.com/gemini-enterprise-agent-platform/scale/provisioned-throughput.

