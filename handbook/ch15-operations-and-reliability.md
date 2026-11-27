# Chapter 15 — Operations Excellence and Reliability

## Exam objectives covered

- **6.1** Principles & recommendations of the operational excellence pillar of the **Google Cloud Well-Architected Framework** (continuously improve & adapt, ensure operational readiness & performance via CloudOps, manage incidents & problems, manage & optimise cloud resources, automate & manage change; AI/ML perspective operations)
- **6.2** Google Cloud Observability: **Cloud Monitoring** (metrics scopes, dashboards, uptime checks, SLO monitoring, alerting policies); **Cloud Logging** (log types, buckets, sinks, exclusions, Log Analytics, log-based metrics, retention); **Cloud Trace**; **Cloud Profiler**; **Error Reporting**; **Managed Service for Prometheus & OpenTelemetry** (Altostrat uses Prometheus); **GKE observability** (health/performance monitoring — new 2026 exam focus); **Agent Platform / model monitoring** & LLM observability (tokens, latency, quality, safety)
- **6.3** Deployment & release management → Ch13 cross-ref; release channels, canary, rollback
- **6.4** Supporting deployed solutions: support plans, escalation, runbooks, Personalized Service Health, Service Health, **Gemini Cloud Assist** investigations
- **6.5** Evaluating quality-control measures: reviews, automated gates, SLO gating, policy-as-code
- **6.6** Reliability in production: chaos engineering, penetration testing (Google policy: no need to notify Google for your own resources but must comply with the AUP and ToS), load testing (tools, patterns), DR/game days, capacity planning, quotas; reliability patterns (graceful degradation, retries with backoff, circuit breakers, load shedding, multi-region)

**Alerting strategies:** symptom-based vs cause-based; SLO burn-rate alerts; notification channels; avoiding alert fatigue (EHR case study — "alerts ignored"; Altostrat case study — email alerts). **SRE practices:** SLI/SLO/SLA; error budgets; toil; blameless postmortems; incident management roles; on-call.

## What the exam really tests

- **Observability design**: Choose the right monitoring tool (metric vs log vs trace vs profile); define SLOs that align with business SLAs; route alerts to the right team.
- **Alert strategy**: symptom-based (what users see) vs cause-based (infrastructure detail); understand SLO burn-rate alerts to prevent alert fatigue; recognize when on-call fatigue is a reliability risk.
- **Incident response**: postmortem culture; roles (incident commander, comms, technical lead); blameless feedback loops.
- **Reliability testing**: distinguish load / stress / soak / chaos / pen-test / DR drill by purpose; pen-testing your own Google Cloud resources does NOT require notifying Google, but you must follow the Acceptable Use Policy and Terms of Service and only affect your own projects.
- **Deployment confidence**: canary / blue-green / rolling; SLO gates block bad releases; rollback strategy.
- **AI/ML ops**: LLM token & latency monitoring; model drift detection; prompt-injection prevention; safety rails monitoring.
- **Operational excellence culture**: automation reduces toil; continuous improvement via data-driven insights; cross-team collaboration.

## Core concepts

### Well-Architected Framework: Operational Excellence Pillar

The operational excellence pillar emphasizes designing, implementing, and managing cloud solutions for **continuous improvement and value delivery**. Core principles:

1. **Ensure operational readiness**: readiness reviews, operational runbooks, health monitoring, alerting on symptoms (not just metrics).
2. **Manage incidents and problems**: postmortem culture, root-cause analysis (RCA), incident management roles, on-call rotations.
3. **Manage and optimize cloud resources**: capacity planning, quotas, cost optimization, auto-scaling, right-sizing.
4. **Automate and manage change**: CI/CD gates, infrastructure-as-code (→ Ch13), policy-as-code, approval workflows.
5. **Continuously improve**: data-driven decision-making, feedback loops, experiment culture, growth mindset.
6. **Embrace CloudOps**: orchestration, automation, and observability as the foundation (cross-cutting practice behind principle 1; the guide groups it as "operational readiness and performance using CloudOps").

**AI/ML perspective**: Monitor model performance, token/latency metrics, prompt-injection attempts, safety scores, and data quality. Use Human-in-the-loop workflows for content moderation and model evaluation.

### Google Cloud Observability: The Four Signals

Google Cloud Observability comprises four signal types; no single tool covers all:

| Signal | Tools | Use case | Example |
|--------|-------|----------|---------|
| **Metrics** | Cloud Monitoring, Managed Service for Prometheus, Grafana | Time-series numerical data; trending & alerting | CPU %, QPS, latency p95, memory usage, error rate |
| **Logs** | Cloud Logging, Log Analytics, Dataflow, BigQuery | Detailed event records, debugging, compliance audit trail | request ID, user ID, operation detail, stack trace |
| **Traces** | Cloud Trace, OpenTelemetry, Jaeger | Request flow across services; latency breakdown | "request took 200ms: gRPC call 150ms, DB 40ms, cache 10ms" |
| **Profiles** | Cloud Profiler, pprof, Grafana Pyroscope | CPU & memory hotspots over time | "function X consumes 60% of CPU in prod; optimize sorting logic" |

**Error Reporting** is a dedicated error-aggregation service fed by logs and the API: it groups exceptions/crashes, trends them and can notify.

### Cloud Monitoring

Core components:

- **Metrics Scope**: defined by a *scoping project* (one scope per project) that can view metrics from up to 375 monitored projects by default. Use a central scoping project for multi-project observability.
- **Dashboards**: custom visualizations pinning metrics, logs, uptime checks, SLO charts. Answer "is the system healthy right now?"
- **Uptime Checks**: external synthetic checks (HTTP/HTTPS/TCP) from multiple regions (at least 3 checkers; private uptime checks reach internal IPs via Service Directory). Alert if endpoint unreachable.
- **SLO Monitoring**: track error budget burn rate. Two types:
  - **Request-based**: measure successful vs total requests.
  - **Window-based**: measure time windows when service met SLO threshold.
- **Alerting Policies**: fire when a metric/log condition occurs; route to notification channels (email, SMS, PagerDuty, Slack, webhook).

**Alert fatigue trap**: Too many alerts → alerts ignored (EHR case: "alerts often ignored"). **Solution**: symptom-based alerting (alert on SLO burn, user-facing latency) not every tiny CPU spike.

### Cloud Logging

- **Log types**: audit logs (Admin Activity, Data Access, System Event, Policy Denied), platform logs, and user-written (application/agent) logs. Built-in buckets: `_Required` (Admin Activity + System Event, 400 days, not configurable) and `_Default` (30 days by default).
- **Log Buckets**: store logs in a chosen region; configurable retention 1 day to 3650 days (10 years). Ingestion is billed per GiB; retention beyond the default 30 days is billed as storage.
- **Log Sinks**: route logs to BigQuery, Cloud Storage, Pub/Sub, another Logging bucket or project (third-party tools such as Datadog/Splunk are fed via Pub/Sub). Exclusion filters are per sink: excluding logs from the `_Default` sink stops them being stored there, but other sinks still receive them.
- **Log Analytics**: SQL querying of log buckets upgraded for Log Analytics; a linked BigQuery dataset lets you join logs with other data without exporting.
- **Log-based Metrics**: create metrics from log entries (e.g., count 500 errors per minute).
- **Retention & Compliance**: a *locked* log bucket (retention lock) prevents deletion and shortening of retention; Bucket Lock is the Cloud Storage equivalent for log sinks. `_Default` logs are deleted after 30 days unless retention is extended; Data Access audit logs are off by default (except BigQuery).

### Cloud Trace, Cloud Profiler, Error Reporting

- **Cloud Trace**: distributed tracing shows request latency breakdown across services. OpenTelemetry auto-instrumentation. Answer: "which service made my request slow?"
- **Cloud Profiler**: continuous, low-overhead statistical sampling of CPU & memory hotspots in production. Answer: "which function burns most CPU?"
- **Error Reporting**: groups crash logs by type, trends them, sends alerts. Dedups similar exceptions.

### Managed Service for Prometheus & OpenTelemetry

- **Managed Prometheus** (Cloud Monitoring native): managed or self-deployed collection scrapes Prometheus endpoints; data is stored in Google's Monarch backend, queried with PromQL (e.g., from Grafana) and retained 24 months.
- **OpenTelemetry**: vendor-neutral instrumentation standard (traces, metrics, logs). Google Cloud SDKs emit OTel; route to Cloud Trace / Cloud Monitoring.
- **Altostrat case study**: uses Prometheus metrics + Cloud Logging; expect to recommend Managed Service for Prometheus (PodMonitoring on GKE) so existing PromQL/Grafana dashboards keep working while data lands in Cloud Monitoring.

### GKE Observability (2026 Exam Focus)

- **Cluster health**: node readiness, pod resource requests vs limits, control-plane component health (via metrics).
- **Workload performance**: GKE Monitoring (GKE native dashboards); resource usage by namespace, deployment, pod.
- **Managed collection**: GKE can auto-configure Prometheus scraping, ship logs to Cloud Logging.
- **Health checks**: Kubernetes liveness/readiness probes → reconcile with Cloud Monitoring status; misaligned probes cause cascading failures.

### Agent Platform / LLM Observability

- **Token metrics**: input tokens, output tokens, total cost; alert on token wastage (bad chunking, redundant calls).
- **Latency**: time to first token, total generation time; SLO: p95 latency ≤ X ms.
- **Quality**: user satisfaction scores, BLEU/ROUGE (if comparing to reference); human-in-the-loop feedback loop.
- **Safety**: prompt-injection rate, PII detection on outputs, sentiment drift on responses, refusal rate (how often model refuses unsafe requests).
- **Cost tracking**: log API calls + token counts to BigQuery (or Log Analytics); chart cost trends in Looker Studio or Cloud Monitoring.

---

## Decision guides

### Choosing an observability signal

```
Is the question about "the system is responding slowly"?
  → Traces (end-to-end latency breakdown) OR metrics (SLI = latency p95)
  
Is it about "we have many errors" (but what kind)?
  → Logs + Error Reporting (crash grouping)
  
Is it about "process X is eating resources"?
  → Profiles (CPU/memory hotspots)
  
Is it about "did we meet our SLA?"
  → Metrics + SLO monitoring (error budget burn)
  
Is it about "compliance: did X event happen?"
  → Logs (audit trail, immutable with a locked log bucket / Bucket Lock)
```

### Choosing an alerting strategy

| Scenario | Alert type | Trap |
|----------|-----------|------|
| **SLO burn rate high (e.g., ≥ 10× over 1 h; illustrative)** | SLO burn-rate alert (symptom-based) | Alerting on every CPU spike (cause-based) → alert fatigue |
| **Endpoint unreachable for 2+ min** | Uptime check + page oncall (external view) | Uptime check passes but internal latency is high |
| **User-facing latency p95 > threshold** | Metric threshold on latency (symptom) | Alerting on total request count (not a symptom) |
| **10+ 5xx errors in 1 min** | Log-based metric + threshold | Alerting per individual error (noise) |
| **Model outputs contain PII** | Log-based metric + analysis (safety check) | Missing safety monitoring on gen-AI outputs |

**Multi-channel routing**: critical→ page on-call; warning→email+Slack; debug→logs only.

### Deciding on reliability testing

| Test type | Purpose | Trigger | Risk level |
|-----------|---------|---------|-----------|
| **Load test** | Verify capacity at X QPS; identify breaking point | Before release; before capacity forecast | Low (realistic traffic) |
| **Stress test** | Push beyond expected load; find limits | Quarterly capacity planning | Low (in lab/staging) |
| **Soak test** | Run at normal load for extended time; detect leaks | Before production release | Low (no limit on time) |
| **Chaos engineering** | Inject failure (kill pod, network partition, latency spike); verify resilience | Incident response drills; reliability review | Medium (need runbook) |
| **Penetration test** | Security: find exploitable vulnerabilities | Before high-security release; annually | High (test only your own resources; follow AUP/ToS; no notice to Google needed) |
| **DR drill** | Failover to standby region; verify RTO/RPO | Quarterly; after architecture change | Medium (downtime window planned) |

⚠ **Pen-testing rule**: Google does not require you to notify it before penetration-testing your own Google Cloud projects, but you must follow the Acceptable Use Policy and Terms of Service and ensure tests only affect your projects (not other customers or Google infrastructure). No DoS attacks or attacks on other customers' resources. Report vulnerabilities found in Google products via the Vulnerability Reward Program.

---

## Well-Architected Framework lens

| Pillar | Application in Operations |
|--------|--------------------------|
| **Operational Excellence** | Automate observability collection; define SLOs that align with business KPIs; blameless postmortems; runbooks; continuous improvement culture. |
| **Security** | Encrypt logs at rest/in transit; locked log buckets for compliance logs; fine-grained access (logging.viewer ≠ logging.admin); audit who accessed sensitive logs. |
| **Reliability** | SLO-driven alerting prevents alert fatigue; chaos testing ensures graceful degradation; multi-region failover tested via DR drills. |
| **Performance Optimization** | Profiler finds CPU/memory hotspots; trace latency breakdown; optimize hot path (before cache, before scaling up). |
| **Cost Optimization** | Log exclusion filters (don't ship debug logs to Cloud Logging); BigQuery BI Engine for cheap dashboard acceleration; Prometheus scrape interval tuning. |
| **Sustainability** | Monitor energy/carbon footprint; right-size resources (avoid over-provisioning); idle resource detection via monitoring. |

---

## AI/GenAI angle

- **Model monitoring**: LLMs drift (output quality degrades over time, user satisfaction drops). Monitor token counts, response times, safety scores.
- **Prompt injection detection**: log + analyze prompts for injection patterns; alert if suspicious prompt pattern detected; add to blocklist.
- **Cost control**: token-counting + budgets. Gen-AI can be expensive (long context windows, repeated API calls). Monitor token/$ ratio; alert on token budget overage.
- **Human-in-the-loop feedback**: log user feedback on model outputs; feed back to training pipeline or fine-tuning loop.
- **LLM safety observability**: track refusal rate (model refuses unsafe requests), sentiment drift, factuality scores; integrate with content moderation (→ Ch11 SDP & Model Armor).
- **Altostrat case study**: transcoding, metadata extraction, summarization, chatbot are all LLM-based; need cost tracking + safety monitoring on chatbot outputs (inappropriate content filtering).

---

## Case-study hooks

**Altostrat Media**: 
- *Problem*: manual operational workflows; email alerts ignored or missed.
- *Solution*: Cloud Monitoring SLO burn-rate alerts → PagerDuty (not email); Prometheus metrics on GKE; Cloud Logging aggregates hybrid (on-prem + GCP).
- *Reliability*: chaos test the media ingestion pipeline (network failure, service restart); verify graceful degradation (buffer to disk if Cloud Storage unavailable).
- *Safety*: inappropriate-content detector (vision + NLP model) → log detections + safety metrics; alert if FP/FN rate climbs.

**Cymbal Retail**: 
- *Problem*: no visibility into legacy SFTP/ETL; no alerting on enrichment pipeline failures.
- *Solution*: Cloud Logging + Log Analytics for batch job monitoring; canary deployment for new catalog enrichment logic (measure quality gate: attribute accuracy ≥ 95%); load test product discovery API before Black Friday.
- *Human-in-the-loop*: log model outputs for human review; metrics on review queue length, approval rate.

**EHR Healthcare**: 
- *Problem*: "alerts often ignored" (operations fatigue); no visibility into hybrid insurer integrations.
- *Solution*: SLO-based alerting (≥99.9% availability on customer-facing APIs); symptom alerts only (user latency p95, error rate); incident commander role for page on-call; blameless postmortems after outages.
- *Compliance*: audit logs in locked log buckets; Data Access logs for PHI queries (must be enabled); retention per policy (HIPAA documentation retention is 6 years).

**KnightMotives Automotive**: 
- *Problem*: on-prem connectivity unreliable; no unified observability.
- *Solution*: uptime checks on Hybrid Connectivity health; distributed tracing on vehicle API calls (on-prem → GCP → external); chaos test dealership app failover to DR region.
- *Autonomous driving*: simulator generates synthetic traffic; Cloud Profiler identifies bottlenecks in ML inference code; cost tracking on TPU training runs.

---

## Exam traps & keyword → answer

1. **"Email alerts are being ignored"** → **SLO burn-rate alerts** (symptom-based, fewer alerts) + **page on-call** for critical; switch from email to on-call phone integration (PagerDuty, webhook/Pub/Sub integrations).

2. **"How do we measure if the service is working?"** → **SLI** (Service Level Indicator = measurable metric like error rate ≤ 1%, latency p95 ≤ 100ms); **SLO** (target: 99.9 % of requests meet SLI); **SLA** (legal/business commitment with consequences such as service credits; set the SLO stricter than the SLA).

3. **"We want to release new code safely"** → **Canary** deployment (route 5% traffic to new version, measure SLO; if OK, route to 100%); **SLO gate** (block rollout if error budget exhausted); **automated rollback** on breached SLO.

4. **"Penetration test our app without notifying Google"** → ✅ **No notification to Google is needed for your own projects/resources**, provided you follow the AUP and ToS. ❌ **NOT OK**: attacking Google infrastructure or other customers' resources, DoS, or tests that affect anyone but you.

5. **"How do we know which service made the request slow?"** → **Cloud Trace** (distributed tracing, latency breakdown per service); **Cloud Profiler** (CPU/memory hotspots within a service).

6. **"Logs are growing; how do we control cost?"** → **Exclusion filters** (don't store debug logs); **log-based metrics** (count errors, don't store full logs); shorter **retention** for noisy buckets or route to a cheaper Cloud Storage sink. (Locked buckets/Bucket Lock protect compliance logs; they are not a cost control.)

7. **"Model outputs contain customer PII; what do we monitor?"** → **Sensitive Data Protection** (DLP) inspection of model outputs/logs (and Model Armor filtering, → Ch11); **log-based alerting** if detection rate climbs; **Human-in-the-loop** review of risky outputs.

8. **"Is node CPU at 90% a problem?"** → Depends: ✅ **If SLO is being met** (latency/errors OK) = no problem; ❌ **If latency p95 breached** = yes, scale out. **Metric alone is not a symptom.**

9. **"Chaos engineering means crashing production"** → ❌ Wrong. It is controlled experimentation with a hypothesis, limited blast radius and a kill switch: start in **staging** (or with mirrored/dark traffic), then graduate to limited production experiments; use **runbooks** to guide recovery. **Document hypotheses** before test.

10. **"Load test showed 95th percentile latency = 500ms; SLO is p99 ≤ 200ms"** → **Fails SLO** (p99 is always ≥ p95, so p99 ≥ 500ms > 200ms); capacity plan for lower QPS or add caching/sharding to reduce latency.

---

## Quick-reference cheat table

| Tool | Signal type | Scope | Pricing | Best for |
|------|-------------|-------|---------|----------|
| **Cloud Monitoring** | Metrics | All GCP resources | Per MiB ingested for chargeable (non-Google-Cloud) metrics; Google Cloud metrics free; free monthly allotment | SLO monitoring, dashboards, alerting |
| **Cloud Logging** | Logs | All services | Per GiB ingested (~$0.50/GiB after free monthly allotment); storage beyond default retention | Audit, debugging, compliance retention |
| **Cloud Trace** | Traces | Apps with OTel/instrumentation | Per million spans ingested after a free monthly allotment (spans auto-generated by Cloud Run/App Engine/Cloud Run functions are free) | Latency breakdown, RCA |
| **Cloud Profiler** | Profiles | Compute Engine, GKE, Cloud Run | Free | CPU/memory hotspots |
| **Error Reporting** | Error signals | App exceptions, logs | Free | Crash grouping, trends |
| **Managed Prometheus** | Metrics (OSS format) | Prometheus-compatible | Per sample ingested | Kubernetes, custom instrumentation |
| **Cloud Logging + BigQuery sink** | Logs (queryable) | BI/analytics queries | BigQuery storage pricing | Log analytics, trending |

| Alert type | Trigger | When to use | Pitfall |
|-----------|---------|------------|--------|
| **Metric threshold** | CPU > 80 % (example) | Infrastructure detail | Alert fatigue; not user-facing |
| **Log-based metric + threshold** | Count(5xx errors) > 10/min | Error rate trending | Dedup similar errors first |
| **SLO burn-rate alert** | Error budget burn above a threshold over a lookback window (SRE Workbook: 14.4× over 1 h and 6× over 6 h to page; 1× over 3 days to ticket) | User-facing SLO breach | Requires SLO setup first |
| **Uptime check** | GET endpoint → 200 OK | External availability | May pass while internal latency high |
| **Absence alert** | No log entries in 5 min | Detect stalled jobs | False positives if job is truly idle |

---

## Practice scenarios

**Scenario 1**: You run a REST API serving 10,000 RPS. SLO is 99.9% of requests ≤ 100ms. Yesterday, error rate spiked to 5%, but your monitoring dashboard showed no alerts. Why?

- A) Cloud Monitoring was disabled
- B) Uptime check passed because endpoint returned 200 OK
- C) BigQuery sink was misconfigured
- D) Your alerting rules only checked CPU %, not error rate (symptom-based alerts missing)

**Answer: D**. The trap: you were alerting on infrastructure (CPU) not symptoms (errors, latency). SLO-based alerting would have fired immediately. Why others wrong: (A) would disable all dashboards; (B) uptime check measures availability, not error rate; (C) sink doesn't affect real-time alerting.

---

**Scenario 2**: Your GKE cluster runs Altostrat's metadata extraction pipeline. A developer asks to run a penetration test on the Pub/Sub topic to find injection vectors. What do you say?

- A) "Get written permission from Google Cloud first"
- B) "Use only staging Pub/Sub topics, not production"
- C) "OK, it's your resource; you don't need to notify Google, as long as you follow the Acceptable Use Policy and ToS"
- D) "Pen-testing Pub/Sub is never allowed on Google Cloud"

**Answer: C**. Google's penetration testing policy does not require you to notify Google for your own resources; you must follow the AUP and ToS and only affect your own projects. Constraints: (B) is overly restrictive as a rule (staging is good practice, but production is permitted if you own it); no DoS, no access to other customers' resources. Why others wrong: (A) not required for your own resources; (D) false.

---

**Scenario 3**: Your Cymbal Retail product-enrichment pipeline processes 50,000 items/hour. After a code deploy, latency grew from 2ms to 50ms per item. You ran a load test and it showed normal throughput. Where do you look first?

- A) Cloud Trace (latency breakdown across services in the pipeline)
- B) Cloud Monitoring CPU % metric
- C) Cloud Profiler (CPU/memory hotspots in the code)
- D) Increase the number of replicas

**Answer: A**. Cloud Trace shows you which service is slow (enrichment service vs. storage write vs. image processing). Profiler helps once you know which service. Why others wrong: (B) CPU may look fine even if a lock contention issue exists; (D) scaling helps only if the bottleneck is capacity, not a code bug.

---

**Scenario 4**: EHR's incident commander runs a postmortem on a 30-min database outage. A team member says, "The on-call engineer should have caught this earlier." What is the most important item for the postmortem to address?

- A) Fire the on-call engineer for missing alerts
- B) Identify the root cause and design preventive controls (better alerting, automation, chaos testing); frame as "we, as a team"
- C) Blame the on-prem legacy system for the outage
- D) Skip the postmortem and move to fire-fighting

**Answer: B**. Blameless postmortems focus on **systemic improvements**, not blame. The goal is to surface why alerts failed (were they too noisy? Did on-call not see them?), design better ones, and automate recovery. Why others wrong: (A) is blame culture, not learning; (C) doesn't fix the problem; (D) repeats the cycle.

---

**Scenario 5**: You want to validate that KnightMotives' autonomous vehicle API can handle a regional failover to DR (RTO 15 min, RPO 1 hour). Which test do you run?

- A) Load test with 2× expected QPS in staging
- B) Stress test pushing 10× capacity on-prem
- C) Chaos engineering: inject 100ms latency on the gRPC call
- D) DR drill: fail over to DR region, measure time to recovery and data loss

**Answer: D**. A DR drill explicitly tests failover mechanics and validates RTO/RPO. Why others wrong: (A) load/stress test capacity, not failover; (B) on-prem capacity doesn't test GCP DR; (C) chaos test resilience, not failover readiness.

---

**Scenario 6**: An LLM-powered chatbot at Altostrat outputs a summary containing a user's phone number. Your log-based monitoring didn't catch it. What signal should you add?

- A) Cloud Profiler (CPU usage)
- B) Cloud Trace (request latency)
- C) Sensitive Data Protection (DLP) analyzing model outputs for PII
- D) Error Reporting (crash grouping)

**Answer: C**. DLP scans outputs for PII patterns; log-based alerts trigger if detection rate exceeds threshold. Why others wrong: (A)/(B) measure performance, not safety; (D) groups crashes, not PII leaks.

---

**Scenario 7**: Your team's error budget for a 99.9% SLO is 43.2 minutes/month (0.1% × 30 days). Current burn rate is 20 minutes/week. At this rate, when will the error budget be exhausted?

- A) In ~2 weeks
- B) In ~1 month (no action needed)
- C) Never (burn rate is sustainable)
- D) The SLO is unachievable

**Answer: A**. 43.2 min budget / (20 min/week) ≈ 2.16 weeks. Action: halt feature rollout, focus on reliability fixes. Why others wrong: (B) ignores burn rate; (C) wrong math; (D) SLO is fine, execution is the issue.

---

**Scenario 8**: You deploy a new recommendation model to 10% of Cymbal's traffic (canary). Metrics look good for 1 hour. What's the next step?

- A) Roll back (model may degrade after 1 hour)
- B) Proceed to 50% traffic; set SLO gate (block full rollout if error rate > 1% or latency p95 > 200ms)
- C) Roll out to 100% immediately
- D) A/B test against old model forever

**Answer: B**. Canary → measure on SLO → gate-check → proceed to higher traffic. Why others wrong: (A) premature rollback; (C) skips gradual validation (risky); (D) eventually you commit or rollback.

---

## Sources

- https://cloud.google.com/architecture/framework/operational-excellence (Well-Architected Framework)
- https://cloud.google.com/monitoring/kubernetes-engine (GKE Observability)
- https://cloud.google.com/logging/docs (Cloud Logging)
- https://cloud.google.com/trace/docs (Cloud Trace)
- https://cloud.google.com/profiler/docs (Cloud Profiler)
- https://cloud.google.com/stackdriver/docs/error-reporting (Error Reporting)
- https://cloud.google.com/architecture/adopting-cloud-native-architecture#sre-practices (SRE on Google Cloud)
- https://support.google.com/cloud/answer/6262505 (Cloud Security FAQ: penetration testing)
- https://cloud.google.com/products/observability/pricing (Observability pricing)
- https://docs.cloud.google.com/stackdriver/docs/observability/scopes (metrics scopes)


---

## Extended concepts and SRE foundations

### SLI, SLO, and SLA in depth

The exam heavily tests understanding the distinction and alignment between these three metrics, especially in real-world scenarios:

- **SLI (Service Level Indicator)**: a measurable aspect of service quality. Examples:
  - Availability: percentage of requests that return 2xx/3xx status
  - Latency: percentage of requests completed within X ms (e.g., p99 ≤ 100ms)
  - Error rate: percentage of requests that do not return 5xx
  - Throughput: requests (or bytes) processed per second

  A good SLI is **user-centric** (what they observe), not infrastructure-centric (CPU %, disk I/O).

- **SLO (Service Level Objective)**: a target for the SLI. Examples:
  - Availability SLO: 99.9% (= 43.2 min downtime/month allowed)
  - Latency SLO: p99 ≤ 100ms for 99.5% of requests
  - Error rate SLO: ≤ 0.1% 5xx errors

  SLOs drive alerting (burn-rate alerts trigger when error budget is consumed too fast).

- **SLA (Service Level Agreement)**: a legal/contractual commitment to the customer. If service falls below the SLA (e.g., monthly uptime), the customer typically receives service credits or other remedies. Keep the internal SLO stricter than the SLA so you react before the SLA is breached.

**Error budget**: an SLO of 99.9% availability has a budget of 0.1% downtime/month. If you consume it all in week 1 (due to bugs), you must halt new feature rollouts until the budget resets. **Error budget is not a buffer to waste—it's a forcing function for prioritizing reliability over velocity.**

### Toil and Burnout

Toil is manual, repetitive operational work that doesn't add long-term value. Examples: manually restarting crashed services, running the same Kubernetes upgrade steps per cluster, copy-pasting log entries into a ticket. **High toil = on-call burnout = poor reliability culture.**

Automation strategy:
1. **Identify toil**: ask on-call engineer "what do you do every day that you'd automate if you could?"
2. **Automate it**: runbook → script → cloud function → CI/CD gate → auto-remediation.
3. **Measure toil reduction**: track engineer time freed per month; reinvest in reliability work (chaos testing, observability).

Altostrat case study: manual transcoding jobs → Cloud Run function with Pub/Sub trigger (eliminates manual scheduling toil).

### Incident Management and Postmortems

Example incident severity levels (organisation-defined and illustrative, not a Google standard; times and escalation paths are examples):

| Severity | Impact | Response Time | Escalation | Example |
|----------|--------|----------------|-----------|---------|
| **P1 (Critical)** | Complete service outage; all users affected; revenue impact | < 15 min page on-call | VP Engineering, CEO notified | EHR: all customer APIs down |
| **P2 (High)** | Partial degradation; some users; significant impact | < 1 hour page on-call | Engineering lead, on-call manager | Altostrat: chatbot latency 10x normal |
| **P3 (Medium)** | Limited scope; workaround exists | < 4 hours email + Slack | Team lead | Cymbal: search filter missing for 1 vertical |
| **P4 (Low)** | Minor UX issue; no workaround needed | < 1 day | Ticket in backlog | KnightMotives: sort order wrong on test vehicle list |

**Incident roles**:
- **Incident Commander (IC)**: owns communication, timeline, decision escalation (not necessarily the technical expert).
- **Technical / Operations Lead**: investigates root cause, coordinates fix (Google SRE calls this the Operations Lead).
- **Communications Lead**: updates stakeholders, writes postmortem summary.
- **On-Call**: pages appropriate specialist(s); hands off to IC.

**Blameless postmortem template**:
1. **Timeline**: what happened and when (UTC timestamps).
2. **Impact**: number of users, duration, revenue loss (if applicable).
3. **Root cause**: not "engineer made a typo" but "our review process missed this type of error, and our tests don't cover it."
4. **Contributing factors**: why did it take 2 hours to detect? (alerts were too noisy, on-call missed them). Why did recovery take 30 min? (no runbook, manual steps needed).
5. **Action items**:
   - Immediate: urgent fixes to prevent recurrence.
   - Short-term: monitoring gaps, runbook automation.
   - Long-term: architecture change, testing improvement.
6. **Lessons learned**: cultural (share knowledge, no blame) and technical.

The goal is **continuous improvement**, not accountability. Blame culture → engineers hide outages → system degrades.

### Monitoring Multi-Region and Hybrid Workloads

KnightMotives and EHR have hybrid (on-prem + GCP) and multi-region requirements. Observability design:

- **Metrics aggregation**: use folder-level metrics scope to collect from all projects (on-prem Prometheus → Cloud Monitoring via Managed Prometheus; GCP native metrics).
- **Distributed tracing**: instrument both on-prem (OpenTelemetry) and GCP services; send traces to Cloud Trace backend.
- **Log centralization**: on-prem syslog → Cloud Logging (via OpenTelemetry Collector/BindPlane or the Cloud Logging API; the Ops Agent covers VMs; the legacy Logging agent is deprecated ⚠ verify); query unified logs in Log Analytics.
- **Alert routing**: critical→ page on-call (phone integration via PagerDuty); warning→ email+Slack; multi-region alert → de-dup (don't page twice if issue is in one region).

### Reliability Patterns

Google Cloud Well-Architected Framework reliability pillar complements operational excellence. Key patterns:

1. **Graceful degradation**: when a service fails (e.g., recommendation engine), serve stale results instead of 500 error. **SLO still breached** (if SLO defined as "fresh results"), but **user experience better**.

2. **Retries with exponential backoff**: transient failures (network glitch) retry with jitter (avoid thundering herd). Example: `retry_delay = min(max_backoff, base_delay * 2^attempt + jitter)`.

3. **Circuit breaker**: if service calls downstream API and gets 5 consecutive 5xx errors, **stop calling it** (fast-fail) for 30 sec, then retry. Prevents cascading failure.

4. **Load shedding**: when queue grows beyond threshold, reject new requests with 429 (Too Many Requests) instead of queueing forever (which delays response for all).

5. **Multi-region active-active**: traffic routed to nearest region via Cloud Load Balancer (global LB); if region fails, traffic reroutes automatically once health checks mark backends unhealthy (seconds, not instant). RTO ≈ 0, RPO ≈ 0 if databases are replicated in real-time (Spanner multi-region).

6. **Bulkhead isolation**: partition resources by customer (Cymbal has many retailers); a bad retailer's runaway job doesn't starve others. Example: Kubernetes namespaces + resource quotas.

---

## Observability Best Practices

### Log Aggregation and Querying

Logs are overwhelming; without filtering, you drown in noise. Best practices:

1. **Exclusion filters first**: drop debug logs, health-check pings, CDN cache hits.
2. **Structured logging**: emit JSON (not plain text) with fields: `{"severity": "ERROR", "request_id": "abc123", "user_id": "user456", "error_code": 42}`.
3. **Consistent labels**: use the same key names across services (not `user_id` in service A and `uid` in service B).
4. **Log-based metrics**: instead of storing every log entry, count error types: `count(severity == "ERROR" AND error_code == 42)`.
5. **Locked buckets for compliance**: if logs are evidence (HIPAA, SOC 2), use a locked log bucket (or Bucket Lock on a Cloud Storage sink) to prevent deletion; set retention to your policy/regulatory requirement.

### Alert Design: Symptom vs Cause

The exam tests your ability to distinguish these:

- **Symptom-based alert** (what users see): error rate > 1%, latency p95 > 100ms, availability < 99.9%.
  - **Pro**: users care; directly tied to SLO.
  - **Con**: may be slow to detect (if you measure latency at the end of the request).

- **Cause-based alert** (infrastructure detail): CPU > 80%, memory > 90%, disk nearly full, replication lag > 5 sec.
  - **Pro**: early warning; may allow preemptive scaling.
  - **Con**: false alarms; CPU at 85% is fine if SLO is met.

**Best practice**: alert on **symptoms first**; use cause-based alerts only to **explain** the symptom (e.g., "latency breached because replication lag spiked").

### Uptime Checks and Synthetic Monitoring

Uptime checks measure **external availability** (what users see). Checks run from multiple regions (USA, Europe, South America, Asia Pacific); you choose which.

- **HTTP/HTTPS check**: GET or POST to an endpoint, expect a 2xx response by default. Can match response content.
- **TCP check**: establish a connection to a port.
- (Private uptime checks use the same protocols against internal IPs via Service Directory.)

Advantages over log-based alerts:
- Detects outages a user would see (endpoint unreachable) **before** logs show errors.
- Measures latency from user perspective (includes network hops, CDN).

Trap: **uptime check passes but internal services fail**. Example: Cloud CDN returns 200 OK with stale cached page, but origin server is down. Mitigation: uptime check + internal health check (e.g., Kubernetes liveness probe).

### Policy-as-Code for Observability

Enforce observability standards across teams:
- Every service must export SLI metrics (latency, error rate, availability).
- Every API must have an SLO defined.
- Every critical service must have runbooks.
- Every on-call engineer must receive pages only if SLO is breached (not noisy infrastructure alerts).

Use **Policy Controller** (Kubernetes) or **Cloud Asset Inventory** + custom Cloud Functions to audit compliance.

---

## Testing Strategies in Depth

### Load Testing

Tools: Apache JMeter, Locust, Vegeta, Gatling. Cloud Run supports running load tests (scales dynamically). Best practices:

- **Realistic traffic model**: mix read/write at the ratio you see in production. Don't test only writes.
- **Ramp-up**: start low (100 RPS), increase linearly over 5 minutes to avoid warm-up effects.
- **Steady state**: hold at target (10,000 RPS) for 10+ minutes; measure p50, p95, p99 latency.
- **Ramp-down**: reduce load linearly; measure if system recovers.
- **Measure**: latency, throughput, error rate, resource utilization (CPU, memory, network).
- **Result**: "Service can handle 10,000 RPS with p99 latency ≤ 100ms; beyond 12,000 RPS, latency exceeds SLO."

Exam scenario: "Load test showed p99 latency = 500ms at 5,000 RPS; SLO is p99 ≤ 200ms at 10,000 RPS." → **You cannot scale to 10,000 RPS without violating SLO; need architecture change (caching, sharding, faster DB).**

### Chaos Engineering in Production (Safely)

**Controlled chaos**: start in staging, then run limited experiments in production because staging is a poor proxy for production traffic, failure modes, and scale. Rules:

1. **Announce** to stakeholders (ops team, product, customers) that you're running a test.
2. **Limit blast radius**: affect 5% of traffic first; have a manual off-switch (circuit breaker to disable the chaos injection).
3. **Run during business hours** (so you can observe and react quickly).
4. **Document hypothesis**: "We hypothesize that if a replica dies, traffic fails over to remaining replicas within 100ms. We'll kill 1 out of 5 replicas and measure error rate."
5. **Rollback plan**: if error rate breaches SLO, disable chaos injection immediately.

Tools: Gremlin, LitmusChaos, Chaos Toolkit, or custom Cloud Functions (e.g., randomly add 500ms latency to 1% of requests).

**Exam trap**: "Chaos engineering = crashing prod and hoping." **Wrong.** It's testing resilience with a controlled plan and a kill switch.

---

## Operational Excellence in AI/ML

New 2026 exam focus. LLM observability is different from traditional services:

1. **Token cost**: tokens are the billable unit; every call costs money (unlike request metrics). Monitor:
   - Input tokens per request (does your prompt engineering waste tokens?).
   - Output tokens generated (is model verbose?).
   - Cost per inference (total tokens × rate).
   - Cost trend (is it growing with usage or exploding due to a bug?).

2. **Latency variability**: LLM latency is inherently variable (first token latency, generation time). SLOs must account for this:
   - Set targets from your measured baseline (numbers are workload-specific, e.g., p50 and p99 total latency).
   - Track p99, not just p95: the tail is long.
   - Time to first token (user experience metric) for streaming responses.

3. **Quality drift**: model outputs may degrade over time (data distribution shift, API version changes). Monitor:
   - User satisfaction scores (thumbs up/down on responses).
   - Fact-checking score (if comparing to golden dataset).
   - Hallucination rate (groundedness/factuality evaluations, LLM-as-judge, or human review samples). BLEU/ROUGE measure overlap with a reference text, not hallucination.

4. **Safety monitoring**: specific to LLMs.
   - Prompt injection attempts (SQL injection but on prompts): count suspicious patterns.
   - PII leakage: DLP scan outputs; alert if leakage detected.
   - Refusal rate: model too conservative (refusing safe requests) or not conservative enough (accepting unsafe requests).
   - Sentiment: response sentiment drift (responses becoming hostile?).

5. **Model evaluation gates**: before deploying a new LLM model version to production:
   - A/B test on shadow traffic (send copy of prod traffic to new model, compare outputs).
   - Quality gate: new model's score on your evaluation set (task-appropriate metric, e.g., groundedness or pairwise win rate) >= baseline minus an agreed tolerance.
   - Safety gate: PII leakage rate <= old model's rate.
   - Cost gate: new model's token cost within an agreed tolerance of the old model's cost.

6. **Human-in-the-loop**: log model outputs for human review; feed feedback back to model fine-tuning. Altostrat's chatbot example: flag outputs where user gave low satisfaction; human reviewer determines if it's a hallucination or legitimate edge case; use feedback to improve retrieval or prompt.

Altostrat case study: GKE runs metadata extraction (vision + NLP), transcoding (audio), summarization (LLM), and chatbot (conversational AI). Each has different observability needs. Metadata extraction: F1 score (precision/recall), not just latency. Transcoding: error rate, throughput. Summarization: ROUGE score. Chatbot: token cost, latency, safety (inappropriate content).

---

## Deployment and Release Management (Integration with Ch13)

The exam covers deployment strategy, rollback mechanics, and SLO gates as part of operations excellence. (→ Ch13 for CI/CD pipelines, Cloud Deploy, Skaffold.)

### Release Strategies

| Strategy | Rollout | Rollback | Risk | SLO gate fit |
|----------|---------|----------|------|--------------|
| **Rolling** | Gradually replace old pods with new (1 at a time or % per minute) | Restart old pods; no data loss | Medium (transient 5xx if new version broken) | Yes; measure SLO during rollout |
| **Blue-Green** | Old (blue) and new (green) run in parallel; switch traffic at once | Switch traffic back to blue | Low if verified; high if not tested | Yes; test green; gate before switch |
| **Canary** | Route 5% to new, 95% to old for 1 hour; gate check (SLO OK?); increase to 100% | Disable new version; traffic goes back to old | Low (only 5% affected initially) | Yes; most SLO-friendly |
| **Recreate** | Kill all old pods, start new ones | Restart old pods (slow) | **Highest** (downtime) | No; may breach SLO during switchover |

**SLO gates (pattern; Cloud Deploy provides `verify` jobs and automations for promote/rollback, so an SLO check is implemented in a verification container or custom automation)**:
Deploy new version.
Measure SLI (error rate, latency) for 5 minutes.
If error_rate > SLO threshold, STOP (don't proceed to next stage).
If OK, automatically proceed to wider rollout.
If gate fails, auto-rollback.

### Rollback and Recovery

Rollback speeds vary (relative, not guaranteed numbers):

- **Canary**: shift traffic weight back to the old version (near-instant); users see old version.
- **Blue-Green**: switch LB traffic back to blue (seconds, once blue is still running).
- **Rolling**: running old+new pods; roll back the Deployment so old pods are restored (minutes).
- **Data-dependent rollback**: if new code broke the DB, rollback may be a long migration (hours). This is a **long-tail failure mode**; must be tested in DR drills.

**Exam tip**: Automatic rollback based on SLO is safer than manual. If you find yourself manually rolling back every deployment, automation is missing.
