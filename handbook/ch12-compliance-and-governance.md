# Chapter 12 — Compliance and Governance

## Exam objectives covered

From exam guide v6.1, section 3.2 — Compliance and governance:
- Understand legislation and regulations: health-record privacy, children's privacy, data privacy, ownership, data sovereignty
- Commercial compliance: card data (PCI DSS), personally identifiable information (PII)
- Industry certifications: SOC 2, ISO 27001
- Audits and audit logging mechanisms
- Shared responsibility model and governance frameworks

## What the exam really tests

- **Data residency vs sovereignty vs localisation:** recognising which GCP technical controls (Org Policy resource location, VPC-SC, Assured Workloads, CMEK/EKM, regional resource selection) enforce each requirement
- **Regulation identification:** matching a business scenario to applicable laws (HIPAA/HITECH + BAA, GDPR, CCPA/CPRA, COPPA, PCI DSS, SOX, FedRAMP, ISO 27001)
- **Audit log strategy:** choosing among Admin Activity, Data Access, System Event, and Policy Denied logs; log sinks (BigQuery/Storage/Pub/Sub); retention and Bucket Lock for immutability
- **Shared responsibility model:** understanding what Google manages vs what you must configure (encryption keys, access policies, audit logs, data classification)
- **PII handling:** data classification, DLP, SDP discovery/de-identification, data minimisation, right to erasure
- **Compliance as code:** Org Policy constraints, Security Command Center posture scanning, Terraform policy validation
- **Regulated-workload reference design:** selecting controls for HIPAA, PCI, or EU-only deployment
- **AI governance basics:** EU AI Act, explainability/auditability, data used for training, prompt logging, customer data protection guarantees

## Core concepts

### Shared responsibility and shared fate model

Google Cloud and customers share responsibility for security and compliance. The boundary shifts by service:

- **Infrastructure (Google manages):** physical security, hardware, hypervisor, BIOS
- **Google-managed services (Google manages):** OS patches, security updates, service SLAs, encryption in transit (TLS 1.2+), default encryption at rest
- **Customer configures:** identity and access (IAM), data classification and encryption keys (CMEK/EKM), network isolation (VPC-SC, org policy), audit logging, data retention/deletion
- **Shared control:** encryption (Google provides algorithm; customer chooses managed vs CMEK vs EKM), configuration management (customer defines policy; Google enforces)

**Key exam trap:** A regulation says "data must not leave the EU." Google cannot guarantee this by itself—you must enable Org Policy location constraints, use regional resources, and enable VPC-SC with EU-only perimeters.

### Regulations and compliance landscape

| Regulation | Protects | Key GCP controls | Use case |
|---|---|---|---|
| **HIPAA/HITECH + BAA** | Patient health records (EHR) | Sign Google's BAA and use only BAA-covered services; optionally Assured Workloads (Healthcare and Life Sciences data boundary packages), CMEK, audit logs (Data Access enabled), VPC-SC, org policy (region constraint), Cloud KMS HSM | Healthcare SaaS, medical imaging, patient records |
| **GDPR** | EU resident personal data (any company storing EU data) | Org Policy (resource location = europe-*), VPC-SC (EU perimeter), CMEK (data controller owns keys), Access Transparency + Access Approval, DLP scanning, automated deletion workflows (erasure requests without undue delay, within one month) | EU operations, KnightMotives (GDPR + autonomous data) |
| **CCPA/CPRA** | California resident data; right to know/delete/opt-out | DLP, data classification, deletion workflows, audit logs, consumer-facing privacy portal | California customers, Altostrat (if US presence) |
| **COPPA** | Children <13 years; no tracking, parental consent | Age-verification controls, minimal data collection, restricted marketing, audit logs | Ed-tech, children's apps |
| **PCI DSS** | Credit card data (PAN, CVV) | Tokenisation (offload card processing to payment gateway), network segmentation (separate VPCs/projects plus VPC-SC and firewall policies), encryption in transit/at rest (CMEK), audit logs (network access, key usage); never log PAN/CVV | E-commerce, payment SaaS (Cymbal Retail) |
| **SOX** | Financial records integrity; internal controls, change audit | Immutable audit logs (Cloud Logging + Bucket Lock), access controls, segregation of duties (IAM deny policies), change management logs | Financial services, Altostrat Finance (media company with revenue) |
| **FedRAMP** | US federal government data | Assured Workloads (FedRAMP Moderate / High control packages; also IL2/IL4/IL5, CJIS, ITAR), CMEK, Access Transparency + Access Approval, continuous monitoring | US gov agencies, gov contractors |
| **ISO 27001** | Information security management system (ISMS) | GCP certifications (Google holds ISO 27001 for its operations), customer must layer own controls (access, encryption, incident response, awareness training), aligned policies | Enterprise SaaS, EHR Healthcare (regulated SaaS) |
| **SOC 1/SOC 2/SOC 3** | Controls for confidentiality, integrity, availability | Google publishes SOC 2 Type II (annual audit of Google's controls); customer must document own controls and controls verification | SaaS providers, Cymbal Retail (financial reporting; data handling audits) |

### Data residency, sovereignty, and localisation

These terms are often conflated but differ legally and technically:

- **Data residency:** where data is stored (jurisdiction). Enforced by: Org Policy `constraints/gcp.resourceLocations` (restrict to regions or value groups such as `in:eu-locations`; applies to supported services and only to new resources), VPC-SC perimeters (prevent export), CMEK (ensure key in same region or HSM in specific country)
- **Data sovereignty:** data is subject to the laws of that jurisdiction (GDPR for EU data, CCPA for California, DPDP Act for India). Requires residency + legal review; not purely technical
- **Data localisation (colloquial):** overlaps both; often used interchangeably with residency in vendor marketing

**Technical controls:**
- **Org Policy resource location constraints** (least flexible): block creation of resources outside specified locations; enforced at API level for supported services; does not move or delete existing resources
- **VPC Service Controls perimeters:** prevent data exfiltration across perimeter boundary; blocks ingress/egress, not creation (resource can be global but VPC-SC stops data leaving the perimeter)
- **CMEK + Cloud KMS (regional HSM):** keep encryption keys in a chosen location (e.g., a Cloud KMS key in `europe-west3` is stored in Frankfurt, Germany; `europe-west1` is Belgium); **EKM (External Key Management)** pushes key material outside GCP entirely (on-prem or third-party vault)
- **Assured Workloads:** bundles controls (org policy location restrictions, personnel/support access controls, Access Transparency/Approval, CMEK requirements) for regulated workloads in specific regions
- **Google Cloud Sovereign Cloud (2024+):** a family of offerings: **Data Boundary** (via Assured Workloads, with optional Sovereign Controls by partners such as T-Systems or S3NS in the EU), **Dedicated** (partner-operated cloud regions) and **Air-gapped** (Google Distributed Cloud); verify current availability on the Google Cloud Sovereign Cloud pages ⚠ verify

### Assured Workloads and compliance packages

Assured Workloads is a managed control framework that creates a compliant "workload" folder and applies a **control package**: data location (resource-location org policy), personnel/support access controls (e.g. support staff who are US persons or EU-based), Access Transparency/Access Approval, and a restricted list of supported services and CMEK/VPC-SC requirements.

**Control packages available (examples; the list changes, check the docs):**
- **US government and regulated:** FedRAMP Moderate, FedRAMP High, DoD Impact Levels IL2/IL4/IL5, CJIS, IRS 1075, ITAR, US Regions and Support
- **Healthcare:** US Data Boundary for Healthcare and Life Sciences (with or without Support); in-scope services must be on Google's HIPAA BAA and HITRUST pages. Note: HIPAA itself does not require Assured Workloads, the BAA is what matters
- **EU:** EU Regions and Support, EU Data Boundary (with Support / with Access Justifications), optional Sovereign Controls by partners
- **Other jurisdictions:** UK, Israel, Saudi Arabia (KSA), Canada, Australia and others
- There is **no PCI DSS or SOC 2 control package**: PCI scope is reduced by design (tokenisation, segmentation) and SOC 2 is an audit report, not a technical boundary

Assured Workloads enforces the constraints via Org Policy and **monitors for violations** (compliance monitoring / drift), reducing manual compliance work. Not every Google Cloud service is supported in every package.

### Audit logging and log strategy

GCP Cloud Logging tracks four log types:

| Log type | What | When enabled | Use | Retention |
|---|---|---|---|---|
| **Admin Activity** | IAM changes, resource creation/deletion, API calls that modify config | Always on, cannot disable, free | Who did what to infrastructure | 400 days (_Required bucket) |
| **Data Access** | Reads of sensitive data (Cloud Storage object reads, Firestore reads, Spanner reads, etc.) | Disabled by default (except BigQuery, where it is always on); enable per service/resource | Audit data access for compliance (HIPAA, PCI); track who accessed customer data | 30 days in the _Default bucket (configurable 1-3650 days) |
| **System Event** | Google-initiated administrative actions that modify configuration (e.g. live migration) | Always on, cannot disable | Understand system behavior | 400 days (_Required bucket) |
| **Policy Denied** | Requests denied by a security policy (VPC-SC, org policy) | Generated by default (can be excluded from sinks, not disabled) | Track denied access attempts; security investigations | 30 days in the _Default bucket |

Admin Activity and System Event logs are free and stored in the non-configurable **_Required** bucket (400 days); Data Access and Policy Denied logs go to **_Default** (30 days unless you change it) and ingestion is charged.

**Log sinks strategy:**
- **Cloud Logging log buckets:** quick queries; _Required 400 days (fixed), _Default 30 days, custom buckets 1-3650 days; individual entries cannot be edited or deleted, and a **locked** log bucket cannot have its retention reduced or be deleted (use for immutability)
- **Cloud Storage bucket (Bucket Lock):** long-term archive; Bucket Lock (WORM) prevents deletion for compliance audits (SOX, HIPAA require immutability)
- **BigQuery:** query audit logs + correlate with business metrics; supports BI dashboards
- **Pub/Sub:** stream logs to external SIEM (Splunk, Datadog) for real-time alerting; off-loads GCP retention cost
- **Aggregated sinks at org level:** one sink capturing all projects' logs; reduces per-project configuration

**Enable Data Access logs** by adding `auditConfigs` (log types DATA_READ, DATA_WRITE, ADMIN_READ) to the IAM policy, or in the console under IAM & Admin > Audit Logs; **route logs** with a sink:
```
gcloud projects get-iam-policy PROJ > policy.yaml   # add auditConfigs, then:
gcloud projects set-iam-policy PROJ policy.yaml
gcloud logging sinks create audit-to-bq bigquery.googleapis.com/projects/PROJ/datasets/AUDIT_LOGS \
  --log-filter='logName:"cloudaudit.googleapis.com"'
```

### PII handling, data classification, and privacy by design

**Sensitive Data Protection (SDP):**
- **Discovery:** scan Cloud Storage, BigQuery, Databases for PII patterns (email, phone, credit card via regex/ML models)
- **De-identification:** remove/mask/pseudonymise PII before sharing with analytics teams; preserves utility (e.g. date-of-birth → age-decade) while protecting identity
- **Classification:** auto-tag PII; integrate with data access policies (restrict "High Sensitivity" to data custodian roles only)

**Data Loss Prevention (DLP):**
- Policy-based: define rules ("do not export credit card numbers")
- Content inspection: scan payloads (API responses, email body, logs) for sensitive data
- Actions: alert, redact, block

**Privacy by design practices:**
- **Data minimisation:** collect only what's needed (don't store full phone number if you only need area code)
- **Purpose limitation:** use data only for stated purpose; disclose to customers
- **Retention:** delete customer data when no longer needed; automate deletion (e.g. Dataflow job running nightly)
- **Right to erasure (GDPR Article 17):** on customer request, delete personal data without undue delay (generally within one month); requires inventory of where data lives
- **Consent & transparency:** privacy notice, explicit opt-in for non-essential processing (e.g. marketing)

**Data classification tagging (Dataplex / Data Catalog):**
- Tag schema: `sensitivity=high`, `pii_type=email`, `retention_class=annual`
- Governs access (via IAM conditions), logging level, encryption strength, replication scope

### Compliance as code

Shift compliance left: codify governance in version control.

**Org Policy as code:**
```yaml
# policy.yaml (applied via Terraform or gcloud)
resource "google_org_policy_policy" "enforce_region" {
  name   = "projects/PROJECT_ID/policies/gcp.resourceLocations"
  parent = "projects/PROJECT_ID"
  spec {
    rules {
      values {
        allowed_values = ["in:eu-locations"]
      }
    }
  }
}
```

**Security Command Center (SCC) compliance tooling:**
- **Compliance Manager** (newer, in SCC): define frameworks and cloud controls, deploy them, monitor for drift and gather audit evidence; **Audit Manager** generates audit-ready evidence reports for assured compliance; the SCC Compliance dashboard/reports map findings to standards (CIS, PCI DSS, NIST, ISO 27001 and others) ⚠ verify exact framework list and tier
- **Security Posture** (SCC Premium/Enterprise): define posture as code (org policies, Security Health Analytics detectors) and detect drift
- Automated checks such as "is a bucket publicly accessible?" or "is a firewall open to 0.0.0.0/0?"; only controls that map to findings can be automated
- Shift-left: validate Terraform plans against policy before apply (Terraform validator / SCC IaC validation, OPA, Sentinel). Forseti is deprecated, do not choose it

**Terraform policy validation:**
- Sentinel (HashiCorp) or OPA/Rego (open-source): pre-apply checks on Terraform plan
- Example: "deny Cloud SQL without CMEK encryption"
- Prevents deployment of non-compliant infrastructure

### Regulated-workload reference designs

**HIPAA landing zone on GCP:**
1. Org setup: sign Google's BAA; optionally Assured Workloads Healthcare and Life Sciences data boundary (applies location/support-access constraints)
2. Network: VPC-SC perimeter (no egress to non-regulated services)
3. Compute: GKE or Cloud Run (both HIPAA-eligible) with Workload Identity
4. Data: Cloud SQL/Spanner (CMEK), Cloud Storage (Bucket Lock, versioning), BigQuery (separate BI project, isolated via VPC-SC)
5. Access: IAM roles + org policy + Access Transparency (logs of Google staff access) and Access Approval (you approve Google staff access); Key Access Justifications (with Cloud EKM) gives a justification for each external key request
6. Audit: Data Access logs → Cloud Storage with Bucket Lock; monthly reports to auditors
7. BAA (Business Associate Agreement): Google and customer sign BAA; customer owns data, Google acts as a business associate (processor)

**PCI DSS segmentation:**
- **PCI scope:** payment processing (tokenizer, gateway, settlement database) in isolated VPC-SC perimeter, limited ingress, no egress to public internet
- **Out of scope:** e-commerce app frontend (internet-facing), inventory DB, analytics (separate projects with own VPC-SC) — card data never touches these
- **Network:** payment gateway in dedicated project; app projects call via Private Service Connect (PSC) on non-routable IP; audit logs in separate Storage bucket

**EU-only deployment (GDPR, ePrivacy):**
1. Org Policy: `gcp.resourceLocations` = `in:eu-locations` (value group) or explicit EU regions such as `europe-west1` (Belgium), `europe-west3` (Frankfurt), `europe-west4` (Netherlands); note some `europe-*` regions (e.g. `europe-west2` London) are not in the EU
2. CMEK: keys in EU region (Cloud KMS)
3. Data residence check: Org Policy prevents accidental global replication
4. Vendor: Google's EU controller datasheet + contract amendment (Standard Contractual Clauses for data transfers outside EU)
5. No AI training on customer data: ensure contract language and SDP covers prompts sent to AI services

### AI governance and compliance

**EU AI Act basics (2024+):**
- **Timeline:** in force Aug 2024; prohibitions applied Feb 2025; general-purpose AI obligations Aug 2025; the 2026 Digital Omnibus on AI deferred the high-risk obligations to Dec 2027 (Annex III stand-alone systems) and Aug 2028 (AI embedded in regulated products) ⚠ verify exact dates
- **High-risk systems** (e.g. employment/hiring, education, creditworthiness, law enforcement, critical infrastructure; AI safety components in regulated products such as vehicles fall under the Annex I regime): need risk management, data governance, logging/audit trail, transparency, human oversight
- **Prohibited practices:** subliminal/manipulative techniques, social scoring, exploiting vulnerabilities, emotion recognition in workplaces and education, untargeted facial-image scraping
- **Transparency obligation:** disclose when a user interacts with an AI system and label AI-generated content
- **Audit trail:** log model inputs, outputs, decisions for investigation

**GCP AI governance practices:**
- **Data used for training:** Google does not use customer data to train its foundation models (Gemini) without permission; verify in contract ("Customer data is not used to improve products")
- **Explainability/auditability (e.g. Altostrat use case):**
  - Log prompts and responses explicitly (e.g. request-response logging for Agent Platform endpoints to BigQuery, or application-level logging); Cloud Audit Logs record API calls and who made them, not prompt content
  - Model cards: document training data, limitations, accuracy on subgroups
  - Feature attribution: explain why model made a decision
  - Responsible AI: bias detection, fairness reporting
- **Sensitive Data Protection on prompts:** before sending data to Gemini APIs, use SDP to mask PII, prevent card numbers, etc.
- **Secure model deployment:** Confidential Computing (encrypted models in memory), VPC-SC for model API calls, RBAC on model access
- **Human-in-the-loop:** for high-stakes decisions (content moderation, loan approval, hiring recommendations), require human review before action
- **Regulated vs unregulated AI:** healthcare AI (diagnosis) may require FDA approval; hiring AI requires EEOC transparency; not GCP's job but customer's responsibility

## Decision guides

**Choose Assured Workloads when:**
- A supported regime applies (FedRAMP, IL4/IL5, CJIS, ITAR, EU/UK/other data boundary with sovereign controls, healthcare data boundary)
- You need data location plus personnel/support-access controls enforced as a package
- Compliance team wants centralized control enforcement and violation monitoring
- Cost of non-compliance high (fines, liability)

**Avoid Assured Workloads if:**
- No regulatory mandate that maps to a package (e.g. plain PCI DSS or HIPAA: BAA + org policy + VPC-SC + audit logs may suffice); lower-risk data (marketing website, internal tools)
- Required services are not supported in the package; some packages carry extra cost, check pricing

**Enable Data Access audit logs when:**
- Business handles customer PII, health, financial, or card data
- Auditors ask "who accessed customer data?"
- Compliance frameworks (HIPAA, PCI, SOC 2) require it

**Avoid Data Access logs if:**
- Cost-sensitive; logs cost ~$0.50/GB ingested; high-volume APIs (> 10M reads/day) add up
- Data not sensitive (public landing page, cache metrics)

**Choose VPC-SC for data residency when:**
- Data must not leave a region (GDPR, HIPAA)
- Need to prevent lateral movement to untrusted services
- Org Policy alone insufficient (resource created but data could be accessed externally)

**Choose CMEK/EKM when:**
- Customer owns encryption keys (regulatory requirement or risk appetite)
- Audit trail needed for key access (who decrypted data)
- Multi-tenant SaaS (each tenant's key managed separately)

**Choose Org Policy location constraints when:**
- Fastest way to enforce region residency (blocks resource creation outside region)
- All workloads need same region (no exceptions)

## Well-Architected Framework lens

| Pillar | In compliance & governance | How |
|---|---|---|
| **Security & Privacy** | Foundational | Data classification, access control (IAM + org policy), audit logging, encryption (CMEK), PII masking (DLP/SDP) prevent breaches; reduce attack surface (VPC-SC); key lifecycle (Cloud KMS) |
| **Reliability** | Supporting | Audit logs immutable (Bucket Lock) ensure post-incident investigation; compliance violations trigger incidents; SLOs on log ingestion/retention |
| **Operational Excellence** | Enabling | Compliance as code (Org Policy, Terraform, SCC Posture) automates checks, reduces manual overhead; CloudLogging dashboards visualize compliance metrics |
| **Cost Optimization** | Managed trade-off | Some Assured Workloads packages, VPC-SC operations effort and Data Access log ingestion add cost; trade against breach/fine risk |
| **Performance** | Indirect | Compliance controls (VPC-SC, CMEK) add latency; design for perf-first, then layer controls (e.g. cache outside VPC-SC when possible) |
| **Sustainability** | Not primary | Compliance infra (VPC-SC appliances, log storage, key management) consume energy; offset by consolidating workloads, retiring legacy systems |

## AI/GenAI angle

- **Prompt logging & audit trail:** When using Gemini APIs or Agent Platform, enable Data Access audit logs (who called which model) and add explicit request-response logging if you must retain prompt/response content; critical for audit trail and incident investigation
- **PII in prompts:** Use SDP to scan and mask PII before sending to Gemini APIs; prevents accidental customer data leakage to Google's logs or model training
- **Model transparency:** Document which models used, training data provenance, limitations; Altostrat requirement for explainable recommendations and content moderation
- **Responsible AI governance:** Bias testing, fairness reports, human review for high-stakes AI (content removal, pricing, targeting); ensure alignment with EU AI Act for high-risk systems
- **Data not used for training:** Verify Google's commitment—customer data (including prompts to private models) not used to train shared foundation models; review contract/DPA
- **Secure AI deployment:** Deploy agent/model API endpoints in VPC-SC perimeter; restrict access via Assured Workloads; audit model-based decisions
- **Chain of custody for AI decisions:** Log decision inputs, model version, confidence score, human override; required for HIPAA (patient recommendations), PCI (fraud flagging), hiring (anti-discrimination)

## Case-study hooks

**Altostrat Media:**
- Needs explainable AI for content recommendations (EU AI Act if Altostrat expands to EU); log all recommendation logic + prompts
- Compliance: if EU users, GDPR applies → use EU region + Org Policy constraint + CMEK in EU
- Inappropriate content detection: audit that AI model doesn't bias against protected groups; Responsible AI reporting
- Podcasts/interviews: PII in transcripts (guest names, locations); use SDP to mask before analytics

**Cymbal Retail:**
- PCI DSS for checkout: payment gateway isolated (VPC-SC), tokenised card data, audit logs for payment access
- CCPA if California customers: capture consent, enable data deletion (right to erasure), audit logs for data access
- Human-in-the-loop: AI attribute generation approved by human before publishing (Cymbal requirement); track approvals in audit logs

**EHR Healthcare:**
- HIPAA mandatory: BAA with Google + BAA-covered services only (optionally Assured Workloads healthcare data boundary), Data Access logs on for PHI services
- 99.9% availability requirement: audit logs to Cloud Storage + Bucket Lock (immutable) for compliance; no log loss on failures
- Insurer integrations on-prem: hybrid connectivity (Interconnect) with network logs (Cloud VPC Flow Logs) showing who accessed insurer data

**KnightMotives Automotive:**
- GDPR (EU vehicles, drivers): EU-only resource location via Org Policy, CMEK in EU, no data export for training
- Autonomous vehicle data: sensitive (safety, biometrics, location); AI safety components are regulated under the EU AI Act via the vehicle type-approval regime; explainability required; log all vehicle-data access + ML decisions
- Data monetisation (dealers/partners): classify data, use org policy to restrict data sharing, audit access, ensure consent/contracts

## Exam traps and keyword → answer

- **"Data must stay in EU. What control?"** → NOT just IAM IP conditions or VPC firewall. → Org Policy `gcp.resourceLocations` + VPC-SC perimeter + CMEK in EU region. Also: verify contract (SCCs for transfers outside EU).
- **"Audit shows someone accessed customer health data. How do we know?"** → Enable Data Access audit logs → query Cloud Logging for resource reads by principal → join with IAM logs to see who had access.
- **"HIPAA audit wants proof logs aren't tampered."** → Bucket Lock (WORM—Write-Once-Read-Many) on Cloud Storage bucket holding audit logs; cannot delete, not even object owner.
- **"Cost is high. How to reduce audit log cost?"** → Disable or narrow Data Access logs (exempted principals, exclusion filters) if not required by compliance → route to Cloud Storage via sinks (cheaper than BigQuery, though less queryable) and add exclusions on the _Default sink so logs are not also stored there; Pub/Sub sink for an external SIEM. Admin Activity logs are free and cannot be disabled.
- **"PCI DSS says card data can't touch analytics. How to segment?"** → Payment processing in separate project with VPC-SC perimeter → app calls via PSC on non-routable IP → analytics in separate project outside perimeter.
- **"AI model trained on customer data—is that allowed?"** → Google does not use customer data (including prompts) to train public Gemini models. Review contract DPA. If customer wants custom training (e.g. fine-tune), must be separate, explicitly consented model.
- **"Compliance Manager vs Audit Manager vs Cloud Audit Logs?"** → Cloud Audit Logs = record of who did what. Compliance Manager (in SCC) = configure frameworks/controls, monitor continuously, detect drift. Audit Manager = generates audit-ready evidence/reports of conformance for auditors. Access Transparency = logs of Google staff access to your data; Access Approval = you approve that access first.
- **"How to delete customer data on GDPR right-to-erasure request?"** → Data spread across BigQuery, Cloud Storage, databases. Use data-mapping tool (Data Catalog / Dataplex) to find all instances → trigger deletion workflow → verify deletion and record it (GDPR: without undue delay, generally within one month).
- **"Can org policy be bypassed by a developer?"** → Not by ordinary developers or project owners: Org Policy is enforced at the API layer and only principals with Org Policy Administrator (`roles/orgpolicy.policyAdmin`) can change or override it at a lower level. It is not retroactive for existing resources and the resource-location constraint covers supported services only.
- **"SOC 2 Type II—does Google pass?"** → Google (the company) is SOC 2 Type II audited. Customer must document own controls and verify with own auditor; Google's SOC 2 does not automatically grant customer SOC 2 compliance.

## Quick-reference cheat table

| Regulation | Scope | Residency | Encryption | Audit Logs | Access Control | Key GCP services |
|---|---|---|---|---|---|---|
| **HIPAA/HITECH** | Patient health data (US) | No fixed residency rule (US regions are common practice) | Encrypted by default; CMEK optional but common | Data Access logs on for PHI services (audit controls) | BAA (Google is a business associate); role-based | BAA-covered services, Cloud KMS, VPC-SC, optionally Assured Workloads healthcare boundary |
| **GDPR** | EU resident data | EU regions only | CMEK (strong rec.) | Data Access; access/export logs | Consent proof, right to erasure, DPA/SCCs | Org Policy, VPC-SC, SDP, Data Catalog, BigQuery |
| **CCPA/CPRA** | California resident data (incl. outside CA) | US (state residency not enforced, but data residency good practice) | Recommended (mitigates liability) | Data access; deletion logs | Consumer access/delete/opt-out API | DLP, SDP, Dataplex, BigQuery export |
| **COPPA** | <13-year-olds (US) | US regions | CMEK | Parental consent logs | Age verification, no tracking/marketing | Cloud Logging, IAM conditions, AppEngine Auth |
| **PCI DSS** | Credit card data (PAN, CVV, etc.) | Scoped: payment processing only | CMEK; TLS in transit | Access logs (card data touches) | Network segregation (VPC-SC), tokenisation (offload to PSP) | VPC-SC, Private Service Connect, Cloud KMS, Firewall |
| **SOX** | Financial records | US (SEC regulated entities) | CMEK (best practice) | Immutable logs (Bucket Lock) | Segregation of duties, change audit | Cloud Storage with Bucket Lock, Cloud Logging, IAM policies |
| **FedRAMP** | US federal govt data | Approved US data centers only | CMEK; EKM option | All logs; FedRAMP-specific logging | Government access controls | Assured Workloads (FedRAMP Moderate/High), Access Transparency, Access Approval |
| **ISO 27001** | Info security ISMS (all industries) | Customer configures | Recommended (ISMS pillar) | Compliance logs | Least privilege, incident response | Security Command Center, Compliance Manager, Terraform/Org Policy, training |
| **SOC 2 Type II** | Service org controls (all industries) | Not primary | Recommended | SoD audit, change logs, uptime | Role-based, access logs, incident response | Cloud Logging, Cloud Monitoring, audit logs, Cloud Build (change log) |

## Practice scenarios

**Scenario 1:** Your company stores EU customer data in BigQuery and must comply with GDPR. You want to ensure data never leaves the EU, even if an employee misconfigures replication. Which combination of controls is MOST effective?

- A) Org Policy `gcp.resourceLocations` constraint (`in:eu-locations`) + CMEK in EU + VPC-SC perimeter
- B) Enable CMEK and store keys in europe-west1
- C) IAM IP-address-based conditions restricting EU IP ranges
- D) Cloud Armor rate-limiting on BigQuery API

**Answer: A.** Org Policy blocks creation of non-EU resources at API layer (cannot accidentally replicate globally). CMEK ensures data encrypted with EU-held key. VPC-SC perimeter prevents export to non-EU services. B alone insufficient (CMEK is key security, not residency). C is network layer, not data residency. D is rate-limiting, not data locality.

---

**Scenario 2:** A healthcare SaaS company must prove to auditors that patient data access has not been tampered with. The audit happens 90 days after go-live. What is the BEST strategy?

- A) Rely on Cloud Logging's default _Default bucket retention
- B) Export logs to BigQuery for fast queries and HA replication
- C) Stream Data Access logs to Cloud Storage bucket with Bucket Lock (WORM) + configure immutability retention
- D) Keep logs in Cloud Logging console and print monthly screenshots

**Answer: C.** Bucket Lock (WORM) retention policy prevents objects from being deleted or overwritten until retention expires, even by bucket owners, satisfying the auditor's tamper-evidence requirement (a locked Cloud Logging log bucket is an alternative). A: Data Access logs land in _Default, which keeps 30 days by default, so they would already be gone at day 90 (only Admin Activity/System Event keep 400 days in _Required), and it is not locked. B: BigQuery tables/datasets can be modified or deleted; no WORM guarantee. D: Printouts not tamper-evident and impractical for 90-day audit.

---

**Scenario 3:** PCI DSS regulation states payment-card data (PAN) cannot be processed on the same database as inventory or customer profiles. You run an e-commerce platform. What design minimises PCI scope?

- A) Single Cloud SQL instance with separate tables; IAM roles restrict card-table access
- B) Tokenise cards at edge (external payment gateway) → store only token in Cloud SQL; payment processing handled by payment provider's VPC-SC-isolated project
- C) Encrypt card data with customer-managed keys and use Cloud DLP redaction on all queries
- D) Run payments on Cloud Run with custom encryption; all other services on GKE

**Answer: B.** Tokenisation is the gold-standard PCI strategy: card data never touches your DB, shrinking PCI scope dramatically (your systems then fall under a much lighter assessment, not zero obligations). Option A: single instance violates network segmentation (scope remains high). C: encryption + DLP helps, but data still in your DB (scope includes entire instance). D: separation by compute type doesn't satisfy PCI (data can still flow between).

---

**Scenario 4:** You enable Data Access audit logs for Cloud Storage in a project. After 2 weeks, monthly log costs triple. What is the root cause and a mitigation?

- A) Data Access logs capture the full object contents; disable Data Access logs to save money
- B) Bucket Lock is duplicating logs; remove Bucket Lock
- C) Cloud Logging is charging per-log-line; compress logs manually
- D) High-volume read workload (e.g. batch jobs or dashboards reading millions of objects) logging every object access; use log exclusion filter or exempted principals, or move that workload to a separate project outside audit scope

**Answer: D.** Data Access logs are charged for ingestion (Cloud Logging list price is about $0.50/GiB ⚠ verify current price). High-volume readers can generate millions of entries/day. Solution: exclude low-risk read patterns (e.g. a batch service account) via an exclusion filter or exempted principals in the audit config, or move that workload to a non-regulated project (if HIPAA/PCI doesn't require auditing it). A: disabling loses audit trail. B: Bucket Lock doesn't cost extra or increase volume. C: Logs cannot be compressed at API layer.

---

**Scenario 5:** Your org policy currently restricts compute resources to `us-central1` only. A team wants to deploy a disaster-recovery instance in `us-east1` for HA. What is the BEST path without violating compliance?

- A) Override org policy for the team's project (grant exception via policy amendment)
- B) Create a separate folder for the DR project and set a different org policy on that folder
- C) Request policy amendment to include `us-east1` in allowed regions, with business justification
- D) Use VPC peering from `us-central1` to `us-east1` instead of actual DR in `us-east1`

**Answer: C.** Org Policy should be amended at the org/folder level based on business need. B is valid (hierarchy allows different policies per folder), but C is cleaner if both projects should be compliant. A: exceptions are not recommended (org policy should be unchangeable except by policy admin). D: peering doesn't create DR in `us-east1` (still single region failure point in `us-central1`).

---

**Scenario 6:** An AI company uses Gemini API to generate metadata for sensitive documents (legal contracts with PII redacted). The legal team asks: can Google use our prompts to train Gemini? What is the answer + required verification step?

- A) No, Google does not use customer data for training public models; verify contract DPA and "customer data commitment"
- B) Yes, Google trains on all API calls; use SDP to mask prompts before sending
- C) Depends on model; Flash models are trained on customer data, Pro models are not
- D) Google stores prompts in Agent Platform logs for 30 days, then deletes; no training risk

**Answer: A.** Google's standard commitment: customer data (including prompts to private Gemini APIs) is NOT used to train shared foundation models. Verify in contract DPA or Google's "Customer Data Terms." B: SDP masks sensitive data in documents, but incorrect premise (Google doesn't use prompts). C: incorrect, both models follow same data policy. D: logs are separate from training (deletion doesn't guarantee no training happened during retention window).

---

**Scenario 7 (Multi-select):** Which of the following are enforced by Org Policy and NOT bypassed by project-level IAM?

- A) Users in group X can edit Cloud Storage buckets
- B) Only service accounts with label `prod` can access Cloud KMS keys
- C) Instances must have OS Login enabled
- D) Compute resources can only be created in europe-* regions

**Answer: D, C.** Org Policy is enforced at the API layer; project owner cannot override. D: resource location constraint blocks non-EU resource creation (non-bypassable). C: OS Login constraint enforced by Compute API. A: IAM role (settable at project level) can be overridden by org admin; not org policy. B: IAM conditions (label-based) are IAM, not org policy (can be modified by project owner).

## Sources

- [cloud.google.com/docs/security/best-practices-for-enterprise-organizations](https://cloud.google.com/docs/security/best-practices-for-enterprise-organizations) — Org Policy, shared responsibility
- [cloud.google.com/architecture/compliance-and-governance-on-google-cloud](https://cloud.google.com/architecture/compliance-and-governance-on-google-cloud) — HIPAA/GDPR/PCI reference architectures
- [cloud.google.com/architecture/assured-workloads](https://cloud.google.com/architecture/assured-workloads) — Assured Workloads control packages
- [cloud.google.com/docs/compliance/hipaa](https://cloud.google.com/docs/compliance/hipaa) — HIPAA on GCP, BAA
- [cloud.google.com/docs/compliance/gdpr](https://cloud.google.com/docs/compliance/gdpr) — GDPR data residency, DPA
- [cloud.google.com/docs/compliance/pci-dss](https://cloud.google.com/docs/compliance/pci-dss) — PCI DSS, tokenisation, VPC-SC for scope reduction
- [cloud.google.com/architecture/audit-logs-best-practices](https://cloud.google.com/architecture/audit-logs-best-practices) — Admin Activity, Data Access, audit log sinks
- [cloud.google.com/docs/security/sensitive-data-protection](https://cloud.google.com/docs/security/sensitive-data-protection) — DLP, SDP, data classification
- [cloud.google.com/docs/security/encryption-in-transit](https://cloud.google.com/docs/security/encryption-in-transit) — encryption standards, Cloud KMS, CMEK
- [cloud.google.com/docs/security/vpc-service-controls](https://cloud.google.com/docs/security/vpc-service-controls) — VPC-SC for data residency, perimeters
- [cloud.google.com/docs/security/cloud-audit-logs](https://cloud.google.com/docs/security/cloud-audit-logs) — Cloud Audit Logs architecture
- [cloud.google.com/architecture/responsible-ai](https://cloud.google.com/architecture/responsible-ai) — Model cards, explainability, bias mitigation
- [cloud.google.com/responsible-ai](https://cloud.google.com/responsible-ai) — Google's Responsible AI principles, Gemini data commitment
- Exam guide v6.1 (sections 3.2, 4.2) — regulatory scope, compliance audit
- ISO 27001:2022 (international standard; GCP is certified)
- SOC 2 Type II (Google publishes annual audit)
- EU AI Act (Regulation EU 2024/1689, in force Aug 2024, phased application; amended by Digital Omnibus on AI, Regulation EU 2026/1744)
- GDPR Articles 5 (principles), 17 (right to erasure), 28 (Data Processor/Controller agreements)
- HIPAA/HITECH (US 45 CFR § 164, § 13400)
- PCI DSS v4.0 (2024 update; tokenisation, scope reduction)
