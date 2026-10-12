# Chapter 11 — Data Security, Controls and Securing AI

## Exam objectives covered

From GCP PCA exam guide v6.1, section 3.1 (part 2) — Designing for security and compliance:

- **Data security:** encryption at rest (Google-managed, CMEK, CSEK, EKM) and in transit (TLS, ALTS); envelope encryption.
- **Cloud KMS:** key rings, keys, versions; protection levels (software, HSM, External); Cloud HSM; EKM; Autokey; rotation; import; key destruction schedule; CMEK org policy constraints; which services support CMEK; separation of key admin vs data user.
- **Secret Manager:** versions; rotation; comparison with KMS and environment variables.
- **Sensitive Data Protection (DLP):** inspect, de-identify, tokenise, format-preserving encryption; risk analysis; discovery.
- **VPC Service Controls:** perimeters; access levels; ingress/egress rules; bridges; dry-run mode; what it protects vs IAM vs firewall; supported services including Agent Platform.
- **Security Command Center:** tiers; findings; posture and Security Health Analytics; threat detection; Gemini in Security.
- **Cloud Armor:** WAF; DDoS; rate limiting; Adaptive Protection; edge policies.
- **Cloud IDS, Cloud NGFW, hierarchical firewall policies** (→ Ch03 for networking detail).
- **Confidential Computing:** VMs; GKE nodes; Confidential Space.
- **Shielded VMs.**
- **Software supply chain security:** Artifact Registry; Artifact Analysis vulnerability scanning; Binary Authorization; Cloud Build provenance and SLSA; Assured Open Source Software; SBOM; GKE policy enforcement; signing.
- **Securing AI:** Model Armor (prompt injection, jailbreak, harmful content, PII/SDP integration, templates, floor settings); Sensitive Data Protection on prompts/responses; secure model deployment (private endpoints, PSC, VPC-SC around Agent Platform, CMEK for models/datasets, least-privilege agent identities, model provenance); OWASP LLM top risks (data exfiltration, poisoning); Gemini in Security (verify current).
- **Incident response & forensics basics.**
- **Zero Trust / BeyondCorp summary** (→ Ch10).

---

## What the exam really tests

- **Encryption trade-offs:** when to use Google-managed vs CMEK vs CSEK vs EKM (complexity, cost, key control, compliance, shared-tenant risk).
- **KMS as a gate:** which GCP services support CMEK (not all do); what "bring your own key" means; rotation policy + key versioning.
- **Secrets vs keys:** when Secret Manager is the right tool vs KMS vs hardcoding in config (exam **trap:** mixing them).
- **VPC Service Controls as a perimeter:** blocks exfiltration even if IAM is misconfigured; can isolate Agent Platform; ingress/egress rules (bridges are legacy) for controlled cross-perimeter access.
- **DLP in data pipelines:** detect PII before it leaves the organization; de-identify vs tokenization vs FPE (format-preserving encryption).
- **Supply chain integrity:** attestations (SLSA provenance, Build L3), signed images, vulnerability scanning in CI/CD.
- **AI-specific risks:** prompt injection attacks, data poisoning, jailbreaks; Model Armor policies; SDP on prompts/responses; secure Agent Platform deployment.
- **Compliance as architecture:** Sensitive Workloads + VPC-SC + CMEK + audit logs = regulated industry design.

---

## Core concepts

### Encryption at rest

#### Google-managed (default)

All customer data at rest is encrypted by default (AES-256) with keys managed by Google using envelope encryption (data encryption keys wrapped by key encryption keys in Google's internal KMS); transparent to the customer and at no extra cost. It is adequate for most workloads, including many regulated ones; you move to CMEK/EKM only when a requirement demands customer control over keys (rotation, revocation, audit, location).

**When to choose:** no regulatory/contractual mandate for customer key control; cost and operational simplicity.

#### CMEK (Customer-Managed Encryption Key)

Customer creates and manages keys in **Cloud KMS**. Google services request a key from KMS to encrypt/decrypt. Customer can rotate, disable, or destroy keys. Supports key versioning.

**When to choose:** HIPAA, GDPR, SOX, or other "customer controls keys" requirement; need for audit of who accessed key; want to rotate on schedule.

**Cost:** you pay Cloud KMS charges (per active key version per month, which differs by protection level, plus per-operation charges); there is no extra charge from the integrated service. Check the Cloud KMS pricing page for current numbers. Rotating a key is free.

**Limitations:** not every Google Cloud service supports CMEK, but most data services do (BigQuery, Cloud Storage, Cloud SQL, Spanner, Bigtable, Firestore, Compute Engine disks, GKE, Dataflow, Dataproc, Cloud Run, Agent Platform and many more); check the "CMEK-integrated services" list. Org policy constraints `constraints/gcp.restrictNonCmekServices` (require CMEK for listed services) and `constraints/gcp.restrictCmekCryptoKeyProjects` (limit which projects may supply keys) enforce CMEK use.

#### CSEK (Customer-Supplied Encryption Key)

Customer provides the key (AES-256) with each API call (e.g., Cloud Storage request headers / gcloud `--csek-key-file` for Compute Engine). Google uses it in memory for the operation and does not store it persistently; customer responsible for key management and if the key is lost the data is unrecoverable.

**When to choose:** niche cases where the key must be supplied by the client per request and never persisted by Google.

**Limitations:** only **Cloud Storage** and **Compute Engine** (persistent disks, images, snapshots) support CSEK; no Cloud KMS integration, no automatic rotation; not supported for BigQuery, Cloud SQL, etc. Google is steering customers to CMEK/EKM instead.

#### EKM (External Key Manager)

Cloud EKM: the key material lives **outside** Google Cloud in a supported external key manager (partner HSM/key-management service such as Thales, Fortanix, etc., on-premises or in another cloud) while you use it through Cloud KMS (CMEK integration). Google calls the external manager over the internet or, with "EKM via VPC", over a private VPC/hybrid connection. Highest control (you can revoke access at the external manager; combined with Key Access Justifications you see why each use occurred); requires integration setup.

**When to choose:** on-premises HSM mandate; key must never be stored in Google; data sovereignty / "hold your own key" requirements.

**Limitations:** availability depends on the external key manager (if it is unreachable the data cannot be decrypted); added latency; only some services/keys support EKM; higher cost and operational effort. It does not require Assured Workloads (although they are often combined).

### Cloud KMS

**Architecture:**

- **Key Rings:** logical container (scoped to region or global). No encryption performed at this level; it's an organizational unit.
- **Keys:** primary resource. Scoped to a key ring. Multiple **versions** can exist (e.g., key version 1, 2, 3).
- **Key Versions:** the actual cryptographic material. Each version has a state such as `ENABLED`, `DISABLED`, `DESTROY_SCHEDULED` or `DESTROYED`. One enabled version of a symmetric key is designated the **primary** version (used for new encryption); other enabled versions can still decrypt.

**Protection Levels:**

| Level | Where key material lives | Use Case |
|---|---|---|
| **Software** | Cloud KMS software (FIPS 140-2 Level 1) | Standard CMEK; lowest cost |
| **HSM** | Cloud HSM: Google-managed, FIPS 140-2 Level 3 validated HSMs | Compliance requiring hardware-protected keys (e.g., FedRAMP High-type needs) |
| **External (EKM)** | External key manager outside Google (on-prem or partner) | Key must never be stored in Google; strict sovereignty |

Latency and price rise from software to HSM to EKM (see the pricing page for current numbers).

**Key Features:**

- **Rotation:** automatic rotation is **not on by default** for keys you create: you set a rotation period (min 1 day; e.g., 90 days) or rotate on demand (rotation is free; keys created through Autokey get a default rotation schedule). Rotation creates a new primary version; **existing data is not re-encrypted** with it, so old enabled versions are still needed to decrypt data encrypted before the rotation (re-encrypt if you need to retire old versions).
- **Autokey:** automatically creates and manages CMEK keys and key rings on demand when developers create supported resources (via key handles), following your standard key policy (e.g., per location and resource type in a project). Reduces manual key-per-service overhead while keeping a separation of duties between key admins and resource creators. Enabled per folder by configuring an Autokey key project (and granting the Cloud KMS service agent the needed roles); supports a defined list of services.
- **Import:** bring existing keys into Cloud KMS; set import job, supply key material in encrypted envelope.
- **Destruction Schedule:** destroying a key version first schedules destruction: the version stays in `DESTROY_SCHEDULED` for a delay (default 30 days, configurable per key at creation) during which it can be restored; after that the material is permanently destroyed (crypto-shredding).

**Separation of Duties:**

- `roles/cloudkms.admin`: manage key rings and keys (create, rotate, set rotation, destroy) but cannot encrypt/decrypt.
- `roles/cloudkms.cryptoKeyEncrypterDecrypter`: use a key to encrypt/decrypt (data user / service agent of the integrated service).
- Also `roles/cloudkms.cryptoKeyEncrypter` / `cryptoKeyDecrypter`, `cloudkms.viewer`, `cloudkms.publicKeyViewer`.

⚠ Exam trap: IAM bindings can be set on the key, key ring, project, folder or organization, and are inherited downward. Best practice is to grant `cryptoKeyEncrypterDecrypter` on the specific key (or key ring), not at project level, and to keep key admins separate from key users (ideally KMS keys in a separate project from the data).

### Secret Manager

Stores **secrets** (API keys, DB passwords, certificates, OAuth client secrets, etc.; payload up to 64 KiB). Replication policy: automatic (Google chooses) or user-managed (chosen regions); regional secrets are also available. Versioning (create new version; versions can be disabled/destroyed; apps usually read `latest` or pin a version). Access via IAM (`roles/secretmanager.secretAccessor`), all access audit-logged; optional CMEK.

**Rotation:**

- Manual: create new version, update application, then disable/destroy old version.
- Scheduled: a rotation schedule on the secret makes Secret Manager **publish a notification to a Pub/Sub topic**; you subscribe (e.g., Cloud Run function) to generate the new credential, update the target system and add a new secret version. Secret Manager does not rotate the credential itself.

**vs KMS:**

| Aspect | KMS | Secret Manager |
|---|---|---|
| **Purpose** | Encrypt data at rest; key lifecycle | Store small secrets (API keys, passwords) |
| **Access** | Via API request to KMS; key never leaves service | Via API; secret contents returned to caller |
| **Rotation** | Key version rolls; old data still decryptable | Versioning; typically requires app restart or reload |
| **Cost** | Per key version per month + operations (varies by protection level) | Per active secret version per month + access operations (cents) |
| **Common trap** | Using KMS as a secret store | Hardcoding secrets in code (use Secret Manager) |

**Best practice:** Secret Manager for secrets; KMS for transparent encryption of data at rest.

### Sensitive Data Protection (DLP)

Sensitive Data Protection (SDP) is the current name of what used to be Cloud DLP (the API is still the "Cloud Data Loss Prevention API", `dlp.googleapis.com`, so you will see both names). It scans text, structured and unstructured data (BigQuery tables, Cloud Storage files, Datastore, images via OCR) for sensitive patterns (PII, credit cards, SSN, email, etc.) using built-in and custom infoTypes.

**Capabilities:**

- **Inspect:** find sensitive data; report findings with confidence scores.
- **De-identify:** replace sensitive values (e.g., masking, hashing, redaction).
  - *Masking:* replace with character(s) (e.g., `***-**-1234` for SSN or `4532-XXXX-XXXX-1234`); irreversible.
  - *Tokenization (pseudonymization):* replace with a token that keeps referential integrity; deterministic encryption (AES-SIV) is reversible, hashing is not.
  - *Format-preserving encryption (FPE, FFX mode):* encrypt with a Cloud KMS-wrapped key so the ciphertext keeps the same alphabet and length as the original (e.g., a 16-digit card number becomes another 16-digit number); reversible with the key.
  - *Bucketing:* group into ranges (e.g., age → `20-30` range).
- **Risk Analysis:** re-identify risk (how many quasi-identifiers needed to re-identify a record?).
- **Discovery:** scan all Cloud Storage buckets / BigQuery projects to find datasets containing PII.

**Integration:**

- Called via the Cloud Data Loss Prevention API (SDP).
- Often integrated into Dataflow pipelines, BigQuery transformations, or Cloud Functions.
- SDP on prompts/responses: before sending user input to Gemini, inspect for PII; before returning model output, de-identify.

### VPC Service Controls

**Perimeter:** virtual boundary around a set of GCP services (e.g., BigQuery, Cloud Storage, Agent Platform). Enforces **egress rules** (what data leaves the perimeter) and **ingress rules** (who can access resources in the perimeter).

**Key properties:**

- Works at the API level (not network packet level).
- **Blocks data exfiltration even if IAM says "allowed"** (defense in depth).
- **Access Levels** (Access Context Manager): conditions on the client (IP range, region, device policy, identity). Requests from outside the perimeter that satisfy an access level referenced in an ingress rule or the perimeter's access-level list are allowed in.
- **Ingress/Egress Rules:** fine-grained rules for API calls crossing the perimeter boundary (identity, source, service/method, target resources). By default, traffic across the boundary is denied in both directions; rules open specific exceptions (e.g., allow a specific service account to write to a bucket in another perimeter, or allow exports to a named external project).
- **Perimeter bridges:** legacy mechanism for allowing communication between projects in different perimeters; ingress/egress rules are the preferred, more granular approach.
- **Dry-run Mode:** log violations without blocking (test before enforcement).

**What it protects vs what it doesn't:**

| Item | VPC-SC | IAM | Firewall |
|---|---|---|---|
| Rogue service account | ✅ (API-level boundary) | ❌ (IAM allows if role granted) | ❌ (not aware of IAM) |
| Copying data to a project outside the perimeter (e.g., export to an outside bucket) | ✅ (egress denied unless an egress rule allows it) | ❌ (doesn't control where data goes) | ❌ (not aware of API calls) |
| Network-layer attack | ❌ | ❌ | ✅ |
| Overprivileged IAM | ❌ | Audit only | ❌ |

**Supported services:** most Google Cloud APIs, including Cloud Storage, BigQuery, Spanner, Firestore, Pub/Sub, Secret Manager, Cloud KMS, Cloud SQL, Bigtable, Agent Platform (Vertex AI), Compute Engine, GKE, Cloud Run functions, Cloud Composer and many more; see the "Supported products and limitations" page for the current list and any restrictions. VPC-SC protects the **service APIs** (e.g., `compute.googleapis.com`); it does not filter network packets, so you still need firewalls and network isolation (→ Ch03) for VM-to-VM or internet traffic.

### Security Command Center

Unified view of security posture and threats across GCP.

**Tiers:**

- **Standard:** free; basic posture management (a limited set of Security Health Analytics detectors), findings dashboard.
- **Premium:** paid; full Security Health Analytics, Event Threat Detection / Container Threat Detection / VM Threat Detection, compliance reporting, attack path simulation, Sensitive Data Protection discovery integration.
- **Enterprise:** paid; everything in Premium plus **multi-cloud** (AWS, Azure) coverage, case management and SOAR-style playbooks, and Mandiant threat intelligence.

**Components:**

- **Security Health Analytics:** automated posture checks (e.g., "firewall rule too permissive", "unencrypted Cloud Storage bucket").
- **Threat Detection:** identifies active threats (suspicious IAM grants, data exfiltration patterns, intrusions).
- **Findings:** list of issues with severity (CRITICAL, HIGH, MEDIUM, LOW). Each finding links to remediation steps.
- **Gemini in Security:** AI-assisted investigation and summarisation of findings and attack paths; tier availability changes, check current docs. ⚠ verify

### Cloud Armor

**WAF + DDoS protection.** Sits at the Google Cloud edge (before traffic reaches your Load Balancer).

**Features:**

- **Layer 7 WAF rules:** custom rules (CEL expressions on headers, URI path, source IP/region) plus **preconfigured WAF rules** based on OWASP ModSecurity Core Rule Set (SQLi, XSS, etc.).
- **DDoS protection:** always-on mitigation of L3/L4 volumetric attacks for load-balanced traffic is built in (standard protection).
- **Rate limiting:** throttle or rate-based ban per client key (e.g., IP) over a time window.
- **Adaptive Protection:** ML-based detection of L7 DDoS anomalies with suggested rules (availability depends on the Standard vs Enterprise tier). ⚠ verify
- **Edge security policies:** a separate policy type that filters requests at the edge before they hit the CDN cache (for Cloud CDN / Media CDN / Cloud Storage backends), versus backend security policies for the origin backend services.

**When to choose:**

- Website / API exposed to internet; concern about abuse.
- Want DDoS mitigation without third-party CDN.
- Regulatory requirement for WAF logging.

**Exam point:** Cloud Armor policies attach to backend services of Google Cloud load balancers and are enforced at Google's edge, so blocked traffic never reaches your VPC backends. → Ch03.

### Cloud IDS, Cloud NGFW, hierarchical firewall policies

Brief coverage; detailed in → Ch03.

- **Cloud IDS:** managed intrusion detection (Palo Alto Networks threat detection) using mirrored VPC traffic; it detects and alerts, it does not block.
- **Cloud NGFW:** Google Cloud's managed, distributed firewall service (firewall policies with FQDN, geo, threat-intelligence and tag-based matching); the Enterprise tier adds intrusion prevention (IPS) and TLS inspection.
- **Hierarchical firewall policies:** org-wide / folder-level firewall rules, inherited by all VPCs (vs per-VPC rules).

### Confidential Computing

Encrypts data **in use** (while it's being processed by CPU). Hardware-based TEE (Trusted Execution Environment).

**Types:**

- **Confidential VMs:** Compute Engine VMs on AMD SEV / SEV-SNP or Intel TDX (and confidential GPUs on supported accelerators). Memory is encrypted in hardware; SEV-SNP/TDX add integrity protection and attestation.
- **Confidential GKE nodes:** Kubernetes nodes with Confidential Computing enabled. Workloads in those nodes have data in use encrypted.
- **Confidential Space:** managed environment for running sensitive workloads with attestation and isolated compute.

**When to choose:**

- Processing highly sensitive data (trade secrets, genomics, financial modeling).
- Need to prove to third parties that data was never visible to GCP staff or hypervisor.

**Limitations:** only specific machine series, CPU platforms and regions support it; some features (e.g., live migration) may be limited by machine type.

### Shielded VMs

**Hardware-based security** for VMs (not the same as Confidential Computing).

**Features:**

- **Secure Boot:** verify kernel integrity at boot.
- **vTPM:** virtual Trusted Platform Module; stores cryptographic keys.
- **Integrity Monitoring:** detect unauthorized kernel/OS modifications.

**When to choose:** defend against rootkit/bootkit attacks; compliance requirement for OS-level integrity.

### Software supply chain security

#### Artifact Registry

Central repository for container images, language packages (Maven, npm, Python, Go). Scanned for vulnerabilities by **Artifact Analysis** (CVE matching).

#### Artifact Analysis

Continuous scanning of images in Artifact Registry (and on GKE). Reports vulnerabilities with severity.

#### Binary Authorization

Deploy-time policy enforcement: only container images with the required **attestations** (signed by trusted attestors, e.g., after Cloud Build and vulnerability checks) can be deployed to GKE or Cloud Run. Supports dry-run mode and break-glass.

#### Cloud Build provenance and SLSA

- **Provenance:** Cloud Build can generate signed **SLSA build provenance** (attestation of what source, builder and build steps produced the image) for container images pushed to Artifact Registry. An SBOM (Software Bill of Materials) is a separate artifact (generated e.g. via Artifact Analysis / build tooling), not the same thing as provenance.
- **SLSA:** Supply-chain Levels for Software Artifacts, v1.0 Build track L0–L3. Cloud Build provenance meets Build Level 3 (signed provenance, hardened build platform with isolated builds).

**Workflow:** Git commit → Cloud Build → compile → scan artifacts → sign attestation → push to registry → Binary Authorization checks signature before deploy.

#### Assured Open Source Software

Google-vetted open-source packages (analyzed for security, licensing, provenance). Option to use only Assured packages in builds.

#### GKE policy enforcement

Can enforce: container image must come from approved registry; must have recent vulnerability scan; must have valid signature.

---

## Decision guides

### Encryption choice matrix

**"What encryption should I use for data at rest?"**

```
Is compliance/regulatory mandate demanding customer-controlled keys?
  YES → Use CMEK (Cloud KMS)
  NO → Does Google-managed default meet your risk?
    YES → Use Google-managed (default)
    NO → Reason?
      A) Key must be outside Google Cloud entirely → EKM
      B) Client must supply the key per request, Google never persists it → CSEK (Cloud Storage / Compute Engine only)
      C) Need key in GCP but managed by you → CMEK (standard choice)
```

### Secret vs Key vs Config matrix

```
Is it encryption-key material (AES, RSA)?
  YES → Cloud KMS

Is it an application credential (API key, DB password, certificate, OAuth client secret)?
  YES → Secret Manager (versioned, audited, scheduled rotation notifications via Pub/Sub)

Is it a non-sensitive config value?
  YES → Config file / env var (or Secret Manager if you want centralized, audited storage)
```

### "Does service X support CMEK?"

| Service | CMEK? | Notes |
|---|---|---|
| Cloud Storage | ✅ | Per bucket default key (or per object); require via org policy `gcp.restrictNonCmekServices` |
| BigQuery | ✅ | Per dataset or table; project/org default key possible |
| Cloud SQL | ✅ | Per instance |
| Spanner | ✅ | Per database |
| Firestore | ✅ | Database-level |
| Cloud Bigtable | ✅ | Per cluster |
| Dataflow | ✅ | Job-level CMEK for pipeline state and temporary storage; sources/sinks use their own CMEK |
| Dataproc | ✅ | Cluster-level (persistent disks, cluster metadata, job args) |
| Cloud Run | ✅ | Container images/source and env vars via Cloud Run CMEK |
| Agent Platform | ✅ | Training data, models, endpoints and many resources can use CMEK; see SDP below |

### VPC Service Controls decision

**"Should I use VPC-SC?"**

```
Does your data-access requirement include prevention of exfiltration by rogue code / insider?
  YES → VPC-SC (API-level boundary)

Can you afford the operational complexity (perimeters, ingress/egress rules, access levels)?
  YES → Proceed

Are all services you need supported?
  Check the supported-products list → if a needed service is unsupported or has limitations, supplement with network isolation / other controls (→ Ch03)
```

### Sensitive Data Protection (DLP) in pipeline

```
Step 1: Inspect source data for PII (Sensitive Data Protection / Cloud DLP Inspect API)
  ↓
Step 2: Decide fate of PII:
  A) De-identify (mask / redact / hash) → proceed with masked data
  B) Tokenize → maintain referential integrity; use token for joins
  C) Format-preserving encryption → reversible encryption that keeps format (a 16-digit number stays 16 digits)
  D) Bucket / categorize → generalize (e.g., age 23 → "20-30")
  ↓
Step 3: Integrate into Dataflow / BigQuery transform / Cloud Function
Step 4: Re-test with Cloud DLP Risk Analysis (re-identification risk)
```

---

## Well-Architected Framework lens

| Pillar | Application in Data Security |
|---|---|
| **Operational Excellence** | Automate key rotation (Cloud KMS rotation schedules or Autokey). Monitor access to sensitive data (Audit Logs). Incident response playbooks for breaches. |
| **Security** | Encryption at rest (CMEK/EKM where required), in transit (TLS/ALTS), in use (Confidential Computing). Least-privilege IAM + VPC-SC. Secret Manager for credentials. Regular security posture checks (SCC). |
| **Reliability** | Key rotation without downtime (Cloud KMS versioning). Protect against accidental key destruction (destroy-scheduled delay, restrict `cloudkms.admin`). EKM adds a dependency: if the external key manager is unavailable, data is inaccessible. Audit log retention for compliance & recovery. |
| **Cost Optimization** | Google-managed encryption is free and enough when no mandate for key control exists. CMEK adds Cloud KMS key-version and operation charges (HSM/EKM cost more than software keys). Scope SDP scans to high-risk datasets and sample large tables. |
| **Performance Optimization** | Each KMS-backed encrypt/decrypt adds a call (mitigated by envelope encryption and caching in integrated services). EKM adds network round-trips to the external manager. Confidential VMs have some performance overhead depending on workload and machine type; benchmark. |
| **Sustainability** | Right-size scanning and retention (SDP scans, log retention) to avoid needless compute/storage; prefer managed defaults over bespoke client-side crypto pipelines. |

---

## AI/GenAI angle

### Securing AI workloads

**Gemini Enterprise Agent Platform (formerly Vertex AI)** deployments typically need:

1. **CMEK for training data & models:** if training on sensitive data (healthcare, finance), encrypt datasets in BigQuery / Agent Platform Feature Store with CMEK.
2. **VPC-SC around Agent Platform:** isolate Agent Platform APIs in a VPC-SC perimeter; restrict egress (models cannot call unvetted external APIs).
3. **Secret Manager for API keys:** Agent calling external APIs (search, payment gateway) should retrieve keys from Secret Manager, not hardcode.
4. **Sensitive Data Protection on prompts/responses:**
   - Before sending user query to Gemini, inspect for PII (e.g., SSN, email).
   - After Gemini response, re-check for PII (model may have leaked info from training data).
   - De-identify if found.

### Model Armor (Prompt Security)

**Model Armor:** a standalone Google Cloud service (`modelarmor.googleapis.com`, managed via Security Command Center) that screens LLM prompts and responses. It integrates with Agent Platform (Gemini), Gemini Enterprise, GKE, Apigee and other places, and can also be called directly through its API for any model (including third-party or self-hosted).

**Threats it mitigates:**

- **Prompt injection and jailbreak:** user (or injected content) tricks the model into ignoring its instructions or bypassing safety guidelines (e.g., "Ignore previous instructions...", role-play attacks).
- **Harmful content:** responsible-AI filters for hate speech, harassment, sexually explicit and dangerous content in prompts and responses.
- **Sensitive data leakage:** Sensitive Data Protection filter (basic infoTypes, or advanced with your own inspect/de-identify templates) catches PII/secrets in prompts and responses.
- **Malicious URLs:** detects malicious links in prompts and responses.

**Features:**

- **Templates:** you define which filters are enabled and the confidence threshold for each (low/medium/high) per use case; applications call `sanitizeUserPrompt` / `sanitizeModelResponse` with a template, or Agent Platform applies it automatically.
- **Floor settings:** minimum filter requirements set at organization, folder or project level; templates in that scope must be at least as strict.
- **Enforcement mode:** inspect-only (log findings) vs inspect-and-block.
- **Integration with SDP:** the SDP filter inspects prompts/responses for sensitive data.
- **Monitoring:** findings and metrics are visible in Security Command Center / Cloud Logging.

**Exam angle:** look for "prompt injection / jailbreak / harmful content screening for LLM prompts and responses" → Model Armor (templates + floor settings); "PII in prompts" → Sensitive Data Protection (directly or via the Model Armor SDP filter).

### Securing model deployment

- **Private endpoints:** deploy Agent Platform Endpoints in private IP (not internet-facing). Access via Private Service Connect or VPC.
- **CMEK for models/datasets:** training data in BigQuery + Feature Store encrypted with CMEK.
- **Least-privilege agent identity:** service account running agent has minimal IAM (only read specific datasets, call specific APIs).
- **Model provenance:** Model Registry records model versions and lineage (which training job and data produced it); sign and attest artifacts in the build pipeline.

### OWASP LLM Top 10 (2025) applied to GCP

| Risk | Mitigation |
|---|---|
| **Prompt injection** | Model Armor prompt injection / jailbreak filter |
| **Improper output handling** | Model Armor response screening; de-identify with SDP; validate/escape model output before passing it to other systems |
| **Data and model poisoning** | Audit and control training data sources; least-privilege write access to datasets; provenance of models and data |
| **Supply chain** | Artifact Analysis scanning, Binary Authorization (only attested artifacts deployed), SLSA provenance, Assured OSS |
| **Excessive agency** | Least-privilege agent identities / service accounts; IAM conditions; limit the tools the agent may call; human approval for risky actions |
| **Sensitive information disclosure** | SDP on prompts/responses (Model Armor SDP filter); VPC-SC egress control; CMEK; least-privilege data access |
| **Unbounded consumption** (includes model denial of service) | Cloud Armor rate limiting on exposed endpoints; Agent Platform / Gemini API quotas; budget alerts |

---

## Case-study hooks

### Altostrat Media

*Requirement:* "AI systems auditable & explainable" + "inappropriate-content detection."

**Ch11 solution:**
- Use **Model Armor** for content filtering (hateful/violent/explicit template definitions).
- **Sensitive Data Protection** to detect if user queries contain PII before sending to Gemini (preserve user privacy in chat logs).
- **Sensitive Data Protection (Cloud DLP API)** on generated metadata (ensure no PII leaked in extracted tags).
- **CMEK** for media library (Cloud Storage buckets holding sensitive customer data).
- **Audit Logs** (all access to sensitive content logged, auditable for compliance).

### Cymbal Retail

*Requirement:* "Human-in-the-loop review UI" for AI-generated catalog; "data security & compliance."

**Ch11 solution:**
- **Sensitive Data Protection** to de-identify customer product data before AI enrichment (prevent PII in public catalog).
- **Binary Authorization** on the container images deployed for the enrichment/catalog services (only attested builds run); use Artifact Analysis and access controls for supplier-provided image assets.
- **Agent Platform with CMEK** for training data (customer catalog is proprietary).
- **VPC-SC** to restrict where catalog data can flow (e.g., no export to third-party enrichment services without approval).
- **Secret Manager** for supplier API credentials.

### EHR Healthcare

*Requirement:* "HIPAA compliance; ≥ 99.9 % availability."

**Ch11 solution:**
- **CMEK** for all patient data (Cloud SQL, Firestore, Cloud Storage backups). Key can only be managed by authorized staff.
- **VPC-SC** + private endpoints (EHR APIs not exposed to internet; only accessed from authorized networks).
- **Sensitive Data Protection** scanning on all data ingested from insurers (detect PHI patterns; de-identify before analytics).
- **Audit Logs** with retention & Bucket Lock (HIPAA audit trail requirement).
- **Cloud Armor** on any web-facing endpoints (WAF against injection attacks).

### KnightMotives Automotive

*Requirement:* "Security paramount (past breaches); GDPR; autonomous-driving AI."

**Ch11 solution:**
- **CMEK** for vehicle telemetry, driver behavioral data (encrypted at rest; key access logged).
- **Sensitive Data Protection** on driver location / biometric / health data (de-identify before analytics).
- **Confidential Computing** for autonomous-driving training (trade-secret model training happens in TEE; model never visible to GCP staff).
- **VPC-SC** around autonomous-driving infrastructure (isolated perimeter; no cross-perimeter data flow without an explicit ingress/egress rule).
- **Model Armor** for in-vehicle chatbot (prevent prompt injection attacks from untrusted input / OBD-II ports).
- **Assured Open Source Software** requirement in build policy (avoid compromised dependencies in vehicle software).

---

## Exam traps & keyword → answer

- ❌ **Trap:** "I need encryption; use Cloud KMS." → Not always. Is it for transparent data-at-rest encryption? Use CMEK in Cloud KMS. Is it storing API keys? Use **Secret Manager**. Must the client supply the key per request and Google never persist it? **CSEK** (Cloud Storage and Compute Engine only).

- ❌ **Trap:** "VPC-SC is a firewall." → No. VPC-SC is API-level; it does not filter network packets. It blocks Google API calls that cross the perimeter even if the network path is open and IAM allows. Firewall blocks packets; VPC-SC blocks authorized-but-unwanted exfiltration.

- ✅ **Keyword: "regulatory compliance + customer-controlled keys"** → CMEK (Cloud KMS).

- ✅ **Keyword: "prevent exfiltration by insider / rogue service account"** → **VPC-SC** (API-level boundary) + audit logs (who called what).

- ✅ **Keyword: "data cannot leave GCP"** → VPC-SC perimeter (egress denied by default; allow only named exceptions via egress rules) **+** ensure all services in the perimeter.

- ✅ **Keyword: "key must live on-premises"** → **EKM** (External Key Manager).

- ✅ **Keyword: "client supplies the key with each request" (Cloud Storage / Compute Engine)** → **CSEK**.

- ✅ **Keyword: "find PII across all datasets"** → Sensitive Data Protection (Cloud DLP) **discovery**.

- ✅ **Keyword: "preserve data format but encrypt"** → Sensitive Data Protection **format-preserving encryption (FPE)**.

- ✅ **Keyword: "prompt injection prevention"** → **Model Armor** (prompt injection / jailbreak filter in a template).

- ✅ **Keyword: "only signed images can deploy to GKE"** → **Binary Authorization**.

- ✅ **Keyword: "SLSA attestation"** → Cloud Build provenance (signed SLSA provenance attestation).

- ✅ **Keyword: "DDoS protection without third-party CDN"** → **Cloud Armor**.

- ✅ **Keyword: "data in use encrypted"** → **Confidential Computing** (Confidential VMs / GKE nodes).

---

## Quick-reference cheat table

| Encryption / Security Control | Purpose | When to use | Cost / Limit | Exam keyword |
|---|---|---|---|---|
| **Google-managed** | Default encryption at rest | No mandate for customer key control; cost priority | No extra charge | "no compliance mandate" |
| **CMEK (Cloud KMS, software)** | Transparent encryption; customer controls keys | Compliance needs customer-controlled keys; need key rotation/audit | KMS key-version + operation charges (lowest of the KMS levels) | "regulatory compliance", "customer controls" |
| **CMEK (Cloud KMS, HSM)** | Same as above + hardware security module | Requirement for FIPS 140-2 Level 3 hardware-protected keys | Higher than software keys | "HSM required", "FIPS 140-2 Level 3" |
| **EKM** | CMEK keys on customer's HSM (on-prem / 3rd-party) | Key must never be stored in Google; on-prem / partner key manager mandate | Highest (plus partner costs) | "key on-premises", "hold your own key" |
| **CSEK** | Customer supplies key with each API call | Client must supply the key per request (Cloud Storage / Compute Engine only) | No KMS charge | "customer-supplied key" |
| **Secret Manager** | Store, version, rotate credentials (API keys, DB passwords) | API keys, OAuth tokens, DB credentials | Per active secret version + access operations | "API key rotation", "credential management" |
| **Sensitive Data Protection / Cloud DLP (Inspect)** | Detect PII, credit cards, etc. in data | Before processing / before export | Per-call pricing | "find PII", "detect sensitive data" |
| **Sensitive Data Protection / Cloud DLP (De-identify)** | Mask, redact, hash, or FPE sensitive data | Remove PII before analytics / sharing | Per-call pricing | "anonymize", "remove PII", "format-preserving" |
| **VPC Service Controls** | API-level perimeter; block exfiltration even if IAM allows | Prevent rogue code / insider exfiltration | No direct cost; management overhead | "prevent exfiltration", "insider threat" |
| **Cloud Armor** | DDoS + WAF at Google edge | Public APIs/websites behind Google Cloud load balancers; DDoS/WAF concern | Per policy + per rule + per request (see pricing; Enterprise tier is subscription-based) | "DDoS mitigation", "WAF" |
| **Binary Authorization** | Enforce only signed images deployed to GKE / Cloud Run | Supply-chain security; compliance | No direct cost; CI/CD integration | "only signed images", "image verification" |
| **Cloud Build provenance** | Signed SLSA build provenance attestation for each build | SLSA Build L3; audit trail | Included in Cloud Build | "SLSA", "provenance attestation" |
| **Confidential Computing (VMs)** | Memory/vCPU encrypted while running | Process trade secrets, genomics, financial models | Confidential VM pricing premium varies by machine type | "data in use encrypted", "insider threat" |
| **Confidential Computing (GKE nodes)** | Kubernetes workloads in TEE | GKE + sensitive workloads | Node premium (instance type dependent) | "GKE + encryption in use" |
| **Shielded VMs** | Secure Boot, vTPM, Integrity Monitoring | OS-level tampering defense | No cost (feature of VM) | "rootkit detection", "Secure Boot" |
| **Security Command Center (Premium / Enterprise)** | Posture management + threat detection (Enterprise adds multi-cloud, case management, SOAR) | Proactive posture and threat monitoring | Paid tiers; see SCC pricing | "threat detection", "multi-cloud security posture" |

---

## Practice scenarios

### Scenario 1: Healthcare startup, patient data encryption mandate
A healthcare SaaS startup stores patient records in Cloud SQL. New HIPAA audit requires: **(1) encryption at rest with customer-controlled keys; (2) a full audit trail of every key use; (3) quarterly key rotation; (4) 30-day key destruction delay for compliance audit window.**

Which solution best meets all requirements?

A) Enable Cloud SQL default encryption (Google-managed), add org-policy constraint to limit access.
B) Create a Cloud KMS key (software protection level) with automatic rotation policy; create CMEK-enabled Cloud SQL instance pointing to that key; enable Audit Logs for key usage.
C) Export Cloud SQL data monthly, encrypt locally with customer's RSA key, re-import.
D) Use CSEK (Cloud Storage snapshots) with manual key supply per backup.

**Answer: B**

**Why correct:** CMEK (option B) gives customer control of keys (requirement 1). Cloud KMS Cloud Audit Logs (Data Access logs enabled) record key usage (requirement 2). A rotation schedule of 90 days = quarterly (requirement 3). The key's destroy-scheduled duration can be set to 30 days (requirement 4; it is also the default). Note: if the requirement were instead that even Google cannot access key material, you would need Cloud EKM (plus Key Access Justifications), not a software key.

**Why others wrong:** A) Google-managed encryption doesn't give customer key control; violates HIPAA audit requirement. C) Offline encryption loses benefits of managed rotation & audit trail. D) CSEK works only with Cloud Storage and Compute Engine (not Cloud SQL) and has no automatic rotation.

---

### Scenario 2: Altostrat Media + prompt injection
Altostrat's chatbot receives user queries and calls Gemini API (Agent Platform). Marketing team is concerned: *Could a user's prompt trick the model into leaking podcast transcripts from the internal training data?*

Which control addresses this risk?

A) Increase IAM role permissions so chatbot service account has read-only access to all training data.
B) Deploy Model Armor with a template that enables the prompt injection and jailbreak filter (and the Sensitive Data Protection filter on responses), screening user prompts before they reach Gemini and responses before they return.
C) Replace Gemini API with a simpler rule-based chatbot (no ML).
D) Store training transcripts in Confidential GKE nodes so data is encrypted in use.

**Answer: B**

**Why correct:** Model Armor's prompt injection and jailbreak filter detects attempts to override instructions or extract hidden data before the prompt reaches the model (a **proactive control**), and its Sensitive Data Protection filter can catch sensitive content leaving in responses. (Sensitive Data Protection alone finds sensitive data such as PII; it does not detect injection attempts.) Also apply least privilege so the model only has access to data the user may see.

**Why others wrong:** A) IAM controls access; doesn't prevent prompt injection from user input. C) Over-engineered; removes AI benefits; not a realistic constraint. D) Confidential Computing protects training infrastructure; doesn't prevent adversarial prompts during inference.

---

### Scenario 3: Multi-tenant SaaS + data exfiltration risk
A multi-tenant SaaS company runs Agent Platform and BigQuery for customer analytics. They store customer data in BigQuery datasets. Concern: *A disgruntled employee (with valid BigQuery reader role) could exfiltrate all customer datasets to an external analytics vendor via API.*

Which mechanism prevents this?

A) Remove the employee's BigQuery role immediately upon termination.
B) Deploy VPC Service Controls with a perimeter around BigQuery and the project's Cloud Storage; do not add egress rules to outside projects (egress is denied by default); define an access level (IP range allowlist) for authorized engineers.
C) Enable customer-managed encryption (CMEK) for all BigQuery datasets; log all decryption events.
D) Implement column-level security in BigQuery so the employee only sees obfuscated data.

**Answer: B**

**Why correct:** VPC-SC operates at API level. Data cannot cross the perimeter boundary unless an ingress/egress rule allows it, so even with a valid IAM role the employee's export or copy of BigQuery data to a bucket or dataset in a project outside the perimeter is denied at the API boundary. The access level ensures that only requests from allowed networks/devices can call BigQuery from outside the perimeter. This is **defense in depth** beyond IAM.
**Why others wrong:** A) Doesn't prevent currently-employed malicious actor. C) CMEK encrypts data; doesn't prevent authorized API calls (employee has decrypt permission if they have BigQuery reader role). D) Column-level security doesn't prevent a reader role from calling `ExportData` to external service.

---

### Scenario 4: Sensitive Data Protection + Cymbal Retail
Cymbal Retail is enriching product data with AI. A data pipeline ingests supplier data (with occasional customer contact info embedded), generates product descriptions, and publishes to catalog (public website).

Before publication, you must ensure: **no customer PII appears in public catalog.**

Which approach is best?

A) Manually review every enriched product before publishing (human-in-the-loop, no automation).
B) Use Cloud DLP to inspect supplier data at ingestion; de-identify (mask email/phone) before passing to Gemini; re-inspect model output for PII; block if found.
C) Store all supplier data in a VPC-SC perimeter; restrict export to internal systems only.
D) Encrypt supplier data with CMEK; only authorized roles can decrypt.

**Answer: B**

**Why correct:** Cloud DLP Inspect detects PII patterns (email, phone, SSN). De-identify before Gemini ensures model doesn't process unmasked PII. Re-inspect output (models can leak training data). Blocks high-confidence PII findings → no PII in catalog. Automated, scalable, auditable.

**Why others wrong:** A) Human review is slow; doesn't scale to thousands of products daily. C) VPC-SC doesn't prevent PII in data; it controls where data flows (different problem). D) CMEK encrypts data at rest; doesn't inspect or scrub PII during processing.

---

### Scenario 5: EKM for on-premises key requirement
KnightMotives Automotive requires: *encryption keys for autonomous-driving training data must never be stored in Google Cloud; keys live in on-premises HSM only.*

How do you configure CMEK?

A) Use CMEK with Cloud KMS software protection level; import key from on-premises HSM into Cloud KMS.
B) Use CSEK; supply key directly in API calls.
C) Use Cloud EKM (External Key Manager); integrate the on-premises key manager via a private connection (Cloud Interconnect / EKM via VPC); point Cloud SQL / BigQuery CMEK to the external key.
D) Use Confidential Computing; place training data in Confidential Space (isolated from GCP infrastructure).

**Answer: C**

**Why correct:** Cloud EKM keeps the key material in a supported external key manager outside Google Cloud; Google services call it (over the internet or EKM via VPC) for wrap/unwrap operations. Meets the requirement that keys are never stored in Google Cloud. (The on-prem HSM must be exposed through a supported EKM partner/key manager.)

**Why others wrong:** A) Importing into Cloud KMS means keys cross the boundary (violates requirement). B) CSEK only works for Cloud Storage and Compute Engine; it is not supported for BigQuery/Cloud SQL, and the keys are supplied per request. D) Confidential Computing protects data in use; doesn't control where encryption keys are stored.

---

### Scenario 6: Binary Authorization + SLSA supply-chain
Your CI/CD pipeline builds container images in Cloud Build. You want: *only images with a valid build provenance attestation signed by your pipeline's key can be deployed to GKE.*

What's the full solution?

A) Enable Binary Authorization on GKE; configure a policy requiring attestation(s) from your attestor (e.g., build provenance verified from Cloud Build, signed with your KMS key); the pipeline creates the attestation after each successful build.
B) Store all image hashes in Secret Manager; before deployment, query Secret Manager to verify image is approved.
C) Use CMEK to encrypt all images in Artifact Registry; only authorized roles can decrypt.
D) Require all images to be scanned by Artifact Analysis (vulnerability scanning) before deployment; if no critical CVEs, allow deploy.

**Answer: A**

**Why correct:** Binary Authorization + attestation requirement = only attested images deploy. Cloud Build generates signed SLSA provenance (Build L3) that can be used to verify how an image was built; the attestor's key (in Cloud KMS) controls who can authorize deploys. Exam topic: supply-chain security via attestation.

**Why others wrong:** B) Secret Manager stores secrets, not image hashes; doesn't provide cryptographic proof (no attestation). C) CMEK encrypts image storage; doesn't verify provenance or prevent unsigned image deploy. D) Vulnerability scanning is good practice; but doesn't prevent deployment of unsigned images.

---

### Scenario 7: Cloud Armor + DDoS for Altostrat API
Altostrat's public recommendation API gets spikes of suspicious traffic (10x normal) from botnets. They need: **(1) automatic DDoS mitigation at the edge; (2) rate limiting by IP; (3) WAF to block malformed requests.**

What's the best solution?

A) Deploy Compute Engine VMs behind Cloud Load Balancer (HTTP(S)), then configure firewall rules to block IPs with high request rate.
B) Use Cloud Armor with DDoS protection enabled (automatic), add rate-limiting policy (requests per IP), add WAF rule (regex on URI path to block malformed requests); attach to HTTP(S) Load Balancer.
C) Use Cloud VPN to isolate the API (only authorized partners can access); require authentication on every request.
D) Use Confidential Computing for API servers; encrypt all traffic.

**Answer: B**

**Why correct:** Cloud Armor sits at Google's edge (drops large DDoS before reaching your network). Rate limiting per IP (built-in feature) → throttle botnets. WAF rules (regex, headers, URI patterns) → block malformed requests. Attach to HTTP(S) Load Balancer (single step).

**Why others wrong:** A) Firewall + LB-level filtering happens *inside* your VPC; DDoS traffic already entered your network. Cloud Armor blocks at the edge (much earlier). C) VPN doesn't protect public API; requires handshake (not DDoS defense). D) Confidential Computing encrypts workload; doesn't help with DDoS / rate limiting.

---

## Sources

- Google Cloud KMS documentation: https://cloud.google.com/kms/docs
- Sensitive Data Protection (formerly Cloud DLP): https://cloud.google.com/sensitive-data-protection/docs
- VPC Service Controls: https://cloud.google.com/vpc-service-controls/docs
- Security Command Center: https://cloud.google.com/security-command-center/docs
- Cloud Armor: https://cloud.google.com/armor/docs
- Confidential Computing: https://cloud.google.com/compute/confidential-computing
- Binary Authorization: https://cloud.google.com/binary-authorization/docs
- Cloud Build provenance & SLSA: https://cloud.google.com/build/docs/securing-builds/view-build-provenance
- Artifact Registry: https://cloud.google.com/artifact-registry/docs
- Model Armor overview: https://cloud.google.com/security-command-center/docs/model-armor-overview
- GCP exam guide v6.1: https://cloud.google.com/certification/cloud-architect
- Well-Architected Framework: https://cloud.google.com/architecture/framework
- Cloud KMS Autokey: https://cloud.google.com/kms/docs/autokey-overview
- Security Command Center pricing: https://cloud.google.com/security-command-center/pricing
