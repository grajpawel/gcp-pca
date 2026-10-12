# Chapter 10 — IAM and Resource Hierarchy

## Exam objectives covered

From the official exam guide (S3 Security & Compliance, 3.1 part 1):
- **Resource hierarchy:** Organization, folders, projects, resources; resource inheritance; design patterns for landing zones (by environment, by team, by region); foundation blueprint.
- **IAM fundamentals:** Principals (users, service accounts, groups); roles (basic, predefined, custom); allow policies; additive inheritance; deny policies; IAM Conditions and conditional role bindings; Principal Access Boundary; Privileged Access Manager (just-in-time access).
- **Groups-not-users principle:** Best practice for scalability and auditability.
- **Service accounts:** Types (user-managed, default, Google-managed, agents); attached vs. key-based auth; key management and why to avoid keys; impersonation (roles, token creation); short-lived credentials; best practices; default service account risks.
- **Workload Identity Federation:** For AWS, Azure, OIDC, GitHub, GitLab; GKE Workload Identity Federation specifics.
- **Cloud Identity and federation:** Google Workspace, SSO/SAML federation, Active Directory integration (Google Cloud Directory Sync, Workforce Identity Federation).
- **Organization Policy Service:** Constraints (list/boolean/custom), inheritance, resource location constraint, domain-restricted sharing, disabling service account key creation.
- **Secure remote access:** IAP (TCP forwarding, web apps), OS Login, context-aware access / Chrome Enterprise Premium (BeyondCorp), bastion alternatives.
- **Separation of duties and least privilege:** Concrete role-split examples.
- **Policy Intelligence tools:** Recommender (IAM), Policy Troubleshooter, Policy Analyzer.
- **Auditing IAM changes:** (→ Ch12 for detail).
- **Identity for AI workloads:** Agent identities, service accounts for Agent Platform, least privilege for agents and MCP.

## What the exam really tests

1. **Design choices for identity & access in multi-tenant, regulated environments:** When to use groups vs. custom roles, when to split a service account into two, how to apply org policy without breaking automation.
2. **Hybrid & multi-cloud identity:** Choosing between WIF (AWS/Azure/GitHub) and Cloud Directory Sync (on-prem AD) based on workload location and federation maturity.
3. **Least privilege in practice:** Given a scenario (dev vs. prod, data analyst vs. DevOps lead), split roles and avoid the "Editor" trap.
4. **Service account lifecycle:** When keys are a liability (avoid), when impersonation is better, why default SAs should be restricted, agent identity patterns.
5. **Audit & compliance readiness:** IAM change logging, policy drift detection, audit trails tied to actual resource access.
6. **Landing-zone architecture:** Org structure (folder per environment vs. per team) affects inheritance, policy blast radius, and auditability.
7. **AI workload identity:** Agents and MCP systems need short-lived, least-privilege credentials; never embed API keys.
8. **Organization Policy as a control plane:** It restricts *which resource configurations are allowed* (guardrails for everyone) vs. IAM's *who can do what*; they are complementary.

## Core concepts

### Resource hierarchy and inheritance

GCP's resource hierarchy is four-level: **Organization (root) → Folders → Projects → Resources**. Each level can hold identity policies; policies are additive downward.

| Level | Scope | Typical use | Policy inheritance |
|-------|-------|-------------|-------------------|
| Organization | Entire org | Company-wide governance | ✅ Child folders/projects inherit all Org policies |
| Folder | Logical groups (business unit, environment) | Partition by team, geography, or env | ✅ Child projects & folders below inherit |
| Project | GCP billing unit, resource container | Single microservice, team, or workload | ✅ Resources inherit project IAM |
| Resource | Compute, storage, API, etc. | Fine-grained access | Lowest level (some resources have child resources, e.g. dataset to table, and pass policies down) |

**Additive inheritance rule:** The effective allow policy of a resource is the union of the allow policy on the resource and those inherited from its ancestors. If you have Viewer at Org level, you stay a Viewer on every project below even if no role is granted on the project itself; a role granted higher cannot be removed lower. Allow policies only add. **Deny policies** and Principal Access Boundary policies are the only IAM mechanisms that take access away.

### Landing-zone patterns

Choose based on governance needs and team structure:

1. **By environment** (Dev/Staging/Prod)
   - Folder per environment; separate service accounts per environment; org policy and IAM can be stricter on the prod folder (e.g., tighter constraints, PAM for privileged access).
   - Pros: Clear blast radius, easy cost allocation per env.
   - Cons: Cross-env resource sharing requires Shared VPC or cross-project links.

2. **By team/business unit** (Data, DevOps, Product)
   - Folder per team; each team manages its own projects; shared foundational services (logging, monitoring) in a separate shared-services folder.
   - Pros: Team autonomy, clear ownership.
   - Cons: Harder to enforce org-wide policies; risk of shadow IT.

3. **By region** (US, EU, APAC)
   - Rarely used alone; usually combined with env/team (Folder: Region → Subfolders: Env or Team).
   - Pros: Compliance with data residency (org policy can enforce storage location per region folder).
   - Cons: Complex hierarchy; management overhead.

**Foundation blueprint:** Start with Org → Folder: Shared-Services + Env-Prod + Env-Staging + Env-Dev. Each env folder gets its own monitoring project, networking project, and team projects.

### IAM model

#### Principals

- **User:** Google account or Workspace account; email address; uniquely identified by email.
- **Group:** Collection of users/service accounts; managed in Cloud Identity or Workspace; inherited membership (nested groups supported).
- **Service Account:** Non-interactive identity for apps, VMs, functions; has a unique numeric ID, email (`SERVICE_ACCOUNT@PROJECT_ID.iam.gserviceaccount.com`), optional keys.
- **Workload Identity principal:** External identity (AWS role ARN, Azure managed identity, OIDC subject claim, GitHub repo context); authenticated via federation.

**Best practice:** Always use **Groups, not Users** for IAM roles. Reasons: scalability (adding/removing users updates group membership, not 100 IAM policies); auditability (audit trail shows group changes); least privilege (easier to grant and revoke in bulk).

#### Role types

| Type | Scope | When to use | Examples |
|------|-------|------------|----------|
| **Basic** | Any level (usually granted on a project) | Not formally deprecated, but avoid in production: very broad permissions across all services. Viewer = read-only, Editor = read/write on most services, Owner = Editor + manage IAM and billing setup. | Viewer, Editor, Owner (use predefined or custom roles instead) |
| **Predefined** | Service-specific; inherited | Most common; curated by Google for each service. | `roles/compute.instanceAdmin.v1`, `roles/storage.admin`, `roles/bigquery.dataEditor`, `roles/iam.serviceAccountAdmin` |
| **Custom** | Organization or project (not folder) | Exact least-privilege fit; e.g., "dev can deploy functions but not edit IAM" | `roles/custom.deploymentEngineer` (project-level: `projects/PROJECT_ID/roles/...`; org-level: `organizations/ORG_ID/roles/...`) |

**Custom role creation:** List permissions you need (e.g., `cloudfunctions.functions.create`, `cloudfunctions.functions.update`, `logging.logEntries.create`), bundle into a custom role. Wildcards are not supported in custom roles; list permissions explicitly. Custom roles are not maintained by Google: you must add new permissions yourself.

#### IAM policies and inheritance

An IAM **allow policy** is a binding: `[ Principal, Role, [Resource] ]`. Attach to any level (org, folder, project, resource). Inherited downward; you cannot revoke at a lower level. Example:

```
Organization IAM:
  + Group "data-team@company.com" → roles/bigquery.dataEditor

Project "analytics-dev" (child of Org):
  + data-team@company.com inherits BigQuery Data Editor at project level
  + Cannot revoke from project (policy is inherited)
```

#### Deny policies (new in 2025–2026 exam)

**Deny policies** explicitly forbid principals from using specific permissions, overriding allows. Attached to org, folder, or project; inherited downward (up to 500 deny policies per resource, managed via the IAM v2 API). Permissions are written in FQDN form, e.g. `compute.googleapis.com/instances.delete`. Use cases:
- Prevent deletion of critical resources: deny `compute.googleapis.com/instances.delete` for all principals except an SRE group (`exceptionPrincipals`).
- Guardrail on an organization-wide basis: deny a sensitive permission (e.g. creating IAM roles) for everyone except a small admin group, regardless of what roles they hold.
- Tag-based guardrails: a deny rule can carry a condition, but deny conditions only recognize resource tag functions (e.g. `resource.matchTag('ORG_ID/env', 'prod')`); time-based conditions are only for allow bindings.

**Evaluation order:** IAM conceptually evaluates (1) Principal Access Boundary policies (is the principal eligible to access the resource at all?), then (2) **deny policies**, then (3) allow policies; (in reality all are evaluated together). If any step blocks, access is denied; deny always wins over allow.

#### Conditions and conditional role bindings

Bind a role to a principal **only when** certain conditions are true:

- **Time-based:** `request.time < timestamp("2026-12-31T23:59:59Z")` (also `request.time.getHours("America/New_York")` etc. for business hours)
- **Resource attributes:** resource type, name, service, and tags, e.g. `resource.type == "compute.googleapis.com/Instance"` or `resource.name.startsWith("projects/_/buckets/prod-")`
- **Network / device context:** there is no direct IP attribute; use an Access Context Manager access level (IP ranges, device policy) via `request.matchAccessLevels(...)` (Context-Aware Access / Chrome Enterprise Premium)
- **Principal attributes:** principal type / subject (mainly for PAB bindings and workforce identities)

**Example:** Grant Compute Admin role to DevOps team, but only in the Dev project and only during business hours (9 AM–6 PM, weekdays).

#### Principal Access Boundary (PAB)

Defines the **set of resources a group of principals is eligible to access**, regardless of role grants. A PAB policy is bound to a *principal set* (a workforce pool, workload identity pool, Workspace domain, or the service accounts/agent identities of a project, folder or organization) and lists the **organizations, folders or projects** the principals may access (resource granularity is project/folder/org, not individual buckets or tables). PAB can only block, never grant: allow policies still must grant the role. Example:
- All service accounts in project `tenant-a` are bound to a PAB policy that makes them eligible only for resources in project `tenant-a` and the shared `platform` project.
- Even if someone later grants one of these service accounts a role on project `tenant-b`, access is blocked.
- Principals not subject to any PAB policy remain eligible for everything; PAB policies are additive (union of eligible resources); they fail closed on evaluation errors.

Used for:
- Multi-tenant applications: ensure a tenant's service accounts cannot access another tenant's projects.
- Preventing access to resources outside your organization (phishing / data exfiltration risk).
- Keeping service accounts limited to specific projects. PAB only blocks permissions in its enforcement version (not all permissions are supported).

#### Privileged Access Manager (PAM)

Just-in-time (JIT) temporary elevation of privileges. Admins define **entitlements** (eligible requesters, roles on an org/folder/project scope, maximum grant duration of up to 7 days, optional justification, optional approvers). A requester asks for a grant for a limited time window (e.g., 2 hours); once approved (or immediately, if no approval is required) a temporary IAM role binding is created and removed automatically on expiry. Basic roles (Owner/Editor/Viewer) cannot be used in entitlements. Integrates with Cloud Audit Logs for compliance. Audit trail shows *who* requested, *why*, *when*, and *what they did*.

### Service accounts

Service accounts are non-interactive identities for applications, deployed services, and agents. Key differences from user accounts:

#### Types

| Type | Created by | Managed by | Lifespan | Use case |
|------|------------|-----------|----------|----------|
| **User-managed** | You (gcloud / console) | Your org (can rotate keys, impersonate) | Persistent | Apps deployed on VMs, GKE pods, functions, agents |
| **Default** | Google (auto-created per project) | Project members with IAM | Persistent; should be restricted | Compute Engine, GKE (avoid giving it Editor) |
| **Google-managed (service agents)** | Google (for Google services) | Google | System-managed | Service agents such as `service-PROJECT_NUMBER@gcp-sa-*.iam.gserviceaccount.com`, which let a service act on your resources; do not remove their roles |
| **Agent accounts** | Google (Agent Platform) | Google (system-managed) | Persistent; short-lived token issuance | AI agents, reasoning engines, MCP connections |

#### Authentication methods

| Method | Credentials | Lifespan | Use |
|--------|-------------|----------|-----|
| **Key (avoid)** | JSON private key file on disk | Indefinite (manual rotation) | Legacy; security risk; don't use in prod |
| **Attached service account** | Service account attached to the resource (Compute Engine VM, Cloud Run, Cloud Run functions, GKE via Workload Identity Federation for GKE); credentials come from the metadata server | Short-lived tokens, auto-refreshed | VMs, serverless, GKE (recommended) |
| **Impersonation (token creation)** | User/SA impersonates another SA (requires `roles/iam.serviceAccountTokenCreator` on that SA) | Short-lived (1 h default; up to 12 h only if allowed by org policy) | User acting on behalf of SA; temporary cross-team access |
| **Workload Identity Federation** | External identity (AWS, Azure, OIDC token) exchanged for short-lived GCP credentials | Short-lived (1 h) | Hybrid/multicloud apps, CI/CD pipelines |
| **Application Default Credentials (gcloud auth application-default login)** | User credentials stored locally by gcloud; client libraries obtain short-lived access tokens (ideally combined with impersonation: `--impersonate-service-account`) | Access tokens ~1 h | Dev machines, CLI scripts |

**Best practices:**
- Never store service account keys on disk. Use Workload Identity Federation for apps outside GCP.
- For Google Cloud VMs: attach a user-managed service account at VM creation; use OS Login for human access. (On-prem workloads cannot have an attached SA: use Workload Identity Federation.)
- For dev machines: use `gcloud auth application-default login` (short-lived, refreshed by gcloud).
- For CI/CD (GitHub Actions, GitLab): use Workload Identity Federation with OIDC token exchange.
- For agents (AI, MCP): use short-lived credential issuance; never hardcode API keys.

#### Key management and risks

**Keys are a liability:**
- No automatic rotation; you must rotate manually.
- If leaked (committed to Git, exposed in logs), they grant indefinite access until revoked.
- Audit trail doesn't show *who* used the key, only that *a key* (of that service account) was used.

**Why avoid keys:**
- Modern alternatives (Workload Identity Federation, impersonation, attached SAs) are more secure and auditable.
- Org Policy can disable key creation (`iam.disableServiceAccountKeyCreation`) and key upload (`iam.disableServiceAccountKeyUpload`). Organizations created since 2024 get these (and other "secure by default" constraints, e.g. `iam.automaticIamGrantsForDefaultServiceAccounts` so the default SAs no longer get Editor) enforced automatically; older organizations must enable them. Exceptions can be made with a conditional/tag-scoped or project-level override.

If you must use keys (legacy app, external vendor):
- Store in Secret Manager, not as files.
- Rotate every 90 days.
- Grant keys only to trusted admins; never share.
- Revoke immediately if leaked.

#### Default service account risks

Each GCP project auto-creates a default service account. Problems:
- Often given the Editor role automatically (legacy behavior; disabled by default in newer organizations via `iam.automaticIamGrantsForDefaultServiceAccounts`).
- Shared across unrelated workloads (VMs, Functions, etc.).
- Lack of granularity: audit trail can't distinguish which workload used it.

**Solution:** Create separate user-managed service accounts per workload; grant minimal roles; restrict the default SA to `roles/viewer` or disable it entirely.

### Workload Identity Federation (WIF)

Exchange external identity (AWS, Azure, on-prem Okta, GitHub, GitLab) for short-lived GCP credentials **without storing GCP keys outside Google Cloud**.

#### How it works

1. External system (e.g., GitHub Actions) obtains a credential from its own IdP: an OIDC/SAML token, or for AWS the signed AWS request/role credentials.
2. The credential is sent to Google's Security Token Service (STS).
3. STS validates it against the workload identity pool provider, applies the attribute mapping/condition, and returns a short-lived *federated* access token.
4. Either the federated token is used directly (direct resource access: roles are granted to the `principal://` / `principalSet://` identity) or it is used to impersonate a service account for a service-account access token.
5. Token expires (1 h default); repeat for next request.

#### When to use

- **GitHub Actions CI/CD:** Deploy to GCP without hardcoding service account keys.
- **AWS EC2 app calling GCP APIs:** EC2 assumes AWS role → STS exchanges for GCP token.
- **On-prem Okta IdP:** IdP issues OIDC → exchange for GCP access token.
- **Multi-cloud app:** Unified identity across clouds; same app code in AWS, Azure, GCP.

#### Configuration

1. Create a **Workload Identity Pool** and, inside it, a **Workload Identity Provider** (AWS, OIDC with issuer URL / allowed audience, or SAML) with attribute mappings and an attribute condition (e.g., restrict to one GitHub repo).
2. Grant IAM roles to the external identity: either directly on the target resource to a `principal://...` / `principalSet://...` identity (preferred, no service account needed), or let it impersonate a service account (grant `roles/iam.workloadIdentityUser` on the SA to the principal set).
3. External app: obtain the external credential, call STS with it + the provider resource name, and receive a short-lived GCP token (client libraries do this from a credential configuration file).

#### GKE Workload Identity Federation

Official name: **Workload Identity Federation for GKE** (formerly "Workload Identity"). It is enabled on the cluster (workload pool `PROJECT_ID.svc.id.goog`; always on in Autopilot) and lets Kubernetes service accounts (KSAs) call Google APIs. Two ways: (1) **direct resource access** (recommended): grant IAM roles directly to the KSA principal (`principal://iam.googleapis.com/projects/PROJECT_NUMBER/locations/global/workloadIdentityPools/PROJECT_ID.svc.id.goog/subject/ns/NAMESPACE/sa/KSA_NAME`), no GSA needed; (2) **service account impersonation**: link the KSA to a GSA (`roles/iam.workloadIdentityUser` + annotation), for APIs that do not support direct access. The GKE metadata server handles the token exchange for pods.

External to GKE: use Workload Identity Federation with workload identity pools (GitHub, AWS, Azure).

### Cloud Identity and federation

#### Cloud Identity

Google's identity-as-a-service (IDaaS) platform; it is the identity layer a Google Cloud Organization hangs on. Provides:
- Directory of users and groups.
- The same identity layer as Google Workspace, without Gmail/Docs.
- SSO to/from third-party IdPs via SAML/OIDC (Okta, Microsoft Entra ID (formerly Azure AD), Ping Identity).
- MFA / 2-step verification enforcement.
- Device management (endpoint management).

Editions: **Cloud Identity Free** and **Cloud Identity Premium** (Premium adds advanced device/security management, more SSO and app-management features, and support). Exact feature split and per-user pricing change over time: check the current pricing page. ⚠ verify

#### Google Workspace

Workspace (formerly G Suite) is Cloud Identity + Gmail + Docs/Sheets/Drive/Meet. For organizations heavily invested in Google productivity tools.

#### Workforce Identity Federation (WIF)

Lets your *workforce* (employees, partners, contractors) authenticate to Google Cloud with an external IdP (Entra ID, Okta, AD FS, any OIDC or SAML 2.0 IdP) **without syncing identities into Cloud Identity**. Identities live in a **workforce identity pool** (organization-level); IAM roles are granted directly to `principal://` / `principalSet://` identities of the pool; they get short-lived tokens for the console, gcloud and APIs. Example: partner contractors accessing your GCP project with their own Okta credentials.

**Workforce vs. Workload Identity Federation:**

| Aspect | Workforce | Workload |
|--------|-----------|----------|
| **Principal** | End-user (human) | Non-interactive (app, CI/CD, VM) |
| **IdP** | Corporate SSO, Okta, Azure AD, etc. | OIDC provider, AWS, GitHub, GitLab |
| **Token** | Workforce identity token | OIDC token, federated credential |
| **Use** | Contractors, partners accessing console/APIs | Apps, pipelines, cross-cloud, agents |
| **Binding** | Roles granted directly to workforce pool principals (`principalSet://.../workforcePools/POOL/...`), no service account | Roles granted directly to workload pool principals, or the pool impersonates a service account |

#### Active Directory integration (on-prem)

Two approaches:

1. **Google Cloud Directory Sync (GCDS):**
   - One-way sync: AD/LDAP → Cloud Identity (users, groups); passwords are NOT synced.
   - Runs in your data center on a schedule; reads from AD, writes to Cloud Identity (groups & users).
   - On-prem admins manage AD; AD stays the source of truth.
   - Typically combined with SSO (SAML) to AD FS / Entra ID so users authenticate against the corporate IdP.

2. **Workforce Identity Federation (OIDC or SAML 2.0):**
   - No sync and no Cloud Identity accounts: users authenticate at the external IdP and are mapped into a workforce identity pool.
   - IAM roles are granted to pool principals/groups (mapped from IdP claims).
   - Users sign in to the console via a workforce login URL; access is token-based and short-lived.

**When to use:**
- **GCDS (+ SSO):** you want users and groups to exist as Cloud Identity accounts (needed e.g. for Workspace, or features that require Cloud Identity users).
- **Workforce WIF:** you want to avoid provisioning/syncing accounts, or you have users from several external IdPs; identities stay in the IdP.

### Organization Policy Service

Enforces *what configurations are allowed* (guardrails on resource configuration) across the resource hierarchy. Different from IAM: IAM says *who can do it* (principal-centric); Org Policy restricts *what can be done to resources*, whoever the principal is. It is neither "allow-first" nor "deny-first": constraints are either enforced or not (boolean) or have allowed/denied value lists.

#### Constraint types

| Type | Mechanism | Example |
|------|-----------|---------|
| **List constraint** | Allowed/denied list of values | `gcp.resourceLocations`: allowed = [`in:eu-locations`] (restrict where resources can be created) |
| **Boolean constraint** | Enforced or not enforced | `iam.disableServiceAccountKeyCreation`: enforced (prevent all SA key creation); `compute.skipDefaultNetworkCreation` (no default VPC) |
| **Custom constraint** | CEL condition on a resource's fields for supported resource types, with an action (ALLOW/DENY) on CREATE/UPDATE | `resource.settings.ipConfiguration.ipv4Enabled == false` on `sqladmin.googleapis.com/Instance`; created per organization and named `custom.NAME` |

#### Key constraints for the exam

- **`compute.skipDefaultNetworkCreation`:** Prevent auto-creation of default VPC (security).
- **`iam.disableServiceAccountKeyCreation`:** Prevent SA key creation; force Workload Identity Federation or impersonation.
- **`iam.disableServiceAccountKeyUpload`:** Prevent uploading your own key material for service accounts.
- **`iam.disableServiceAccountCreation`:** Prevent creation of new service accounts.
- **`iam.allowedPolicyMemberDomains`:** Domain-restricted sharing (only identities from listed Cloud Identity/Workspace customer IDs can be granted roles).
- **`gcp.resourceLocations`:** Resource location restriction (list constraint, supports value groups such as `in:eu-locations`).
- **`compute.requireShieldedVm`:** All VMs must use Shielded VM images (UEFI, Secure Boot, vTPM, integrity monitoring).
- **`storage.uniformBucketLevelAccess`:** Force uniform bucket-level access; disables object ACLs.
- **`compute.requireOsLogin`:** Force OS Login on VMs.
- **`sql.restrictPublicIp`:** Prevent public IP on Cloud SQL instances.

#### Inheritance

Policies are inherited down the hierarchy, and by default a policy set on a child **replaces** the parent's for that child (it can override the parent, unless the parent uses list rules with `inheritFromParent: true`, in which case values are merged). Boolean enforcement at the org can therefore be turned off lower unless you restrict who has `roles/orgpolicy.policyAdmin` (and use tags/conditions to scope exceptions).

#### Resource location constraint

Use the list constraint **`gcp.resourceLocations`** (not a `compute.*` constraint) with allowed values such as `in:eu-locations`, `in:europe-west1-locations`, or specific regions `europe-west3`. It applies to supported services (e.g., Cloud Storage buckets, BigQuery datasets, Compute Engine resources) when resources are created and does not move existing resources. Common for GDPR / data residency.

#### Domain-restricted sharing

The **`iam.allowedPolicyMemberDomains`** list constraint restricts which identities (by Cloud Identity / Workspace customer ID) can be granted IAM roles, preventing accidental sharing with external parties (e.g., granting a role to a personal Gmail account). Because `allUsers` / `allAuthenticatedUsers` are outside any domain, it also blocks public (anonymous) IAM bindings. Newer organizations get it enforced by default.

### Secure remote access

#### Identity-Aware Proxy (IAP)

Verifies user identity *before* traffic reaches your application. Acts as a reverse proxy. Use cases:
- **SSH to VMs (via TCP forwarding):** Only authorized users can SSH; audit trail shows who accessed what and when.
- **RDP to Windows VMs (via TCP forwarding):** Same as SSH.
- **Web apps (HTTP/HTTPS):** Apps behind IAP (via an external/internal Application Load Balancer, App Engine, Cloud Run, etc.) receive the user's identity in headers (`X-Goog-Authenticated-User-Email`) and a signed JWT (`X-Goog-IAP-JWT-Assertion`) that the app should verify; access needs the `IAP-secured Web App User` role.
- **TCP forwarding (`gcloud compute ssh --tunnel-through-iap`, `gcloud compute start-iap-tunnel`):** Tunnel arbitrary TCP (SSH, RDP, MySQL) through IAP to VMs without external IPs; access needs `IAP-secured Tunnel User` (`roles/iap.tunnelResourceAccessor`) and a firewall rule allowing the IAP range `35.235.240.0/20`.

IAP uses identity and context (access levels) to authorize each request instead of network location. Best for:
- Internal web apps (not public).
- Admin consoles requiring fine-grained IAM control.
- Hybrid: on-prem user → IAP → VM in GCP.

#### OS Login

Centralized Linux/Windows VM access management with IAM. Instead of SSH keys stored in instance/project metadata, OS Login ties POSIX accounts and SSH public keys to the user's Google identity (and to service accounts); access is granted with IAM roles (`roles/compute.osLogin`, `roles/compute.osAdminLogin` for sudo, `roles/compute.osLoginExternalUser` for users outside the org). Optional 2-step verification (`enable-oslogin-2fa` metadata). Forces individual user accountability.

**Org policy:** `compute.requireOsLogin` (boolean) mandates OS Login for VMs in the scope where it applies. (Enabled by default on new projects in recent organizations ⚠ verify.)

**Pros:**
- Centralized key and access management through IAM; instant revocation.
- Audit: logs show the user identity, not a shared key.
- MFA: can require 2-step verification for SSH.

**Cons:**
- Needs the OS Login agent/guest environment on the VM (supported images) and IAM roles per user; directory accounts must be Google identities (Cloud Identity/Workspace, or workforce identity federation).

#### Context-aware access and Chrome Enterprise Premium (BeyondCorp)

Google's BeyondCorp (zero-trust) architecture: access control based on *context* (device posture, IP/location, identity), not network perimeter. **Chrome Enterprise Premium** (formerly BeyondCorp Enterprise) builds on Access Context Manager **access levels** and provides:
- **Device posture checking:** Is the device encrypted, screen-locked, OS up-to-date, corporate-managed?
- **Context-aware access:** Apply access levels to IAP-protected apps/VMs, Google Cloud APIs and Console (via IAM Conditions or VPC Service Controls), and Workspace apps.
- **Threat and data protection** in Chrome (DLP, malware scanning) as part of the Premium offering.

Integrates with Cloud Identity / Workspace for user + device assessment.

#### Bastion alternatives

| Method | Setup | Audit | Cost |
|--------|-------|-------|------|
| **Bastion VM** | Dedicated Jump host in VPC; users SSH to bastion, then to target | OS Login or SSH keys; manual logging | A VM to run, patch and secure |
| **IAP SSH** | VM needs no external IP; user has IAM role; SSH via gcloud `--tunnel-through-iap` | Built-in Cloud Audit Logs; no bastion | No VM cost; IAP TCP forwarding has no separate charge |
| **OS Login** | Google identity + OS Login on VM (works together with IAP SSH) | Centralized, audit-friendly | No extra charge |
| **Private connectivity + Cloud SQL Auth Proxy** | Cloud SQL without public IP (private IP / Private Service Connect); app uses Cloud SQL Auth Proxy | Cloud SQL audit logs | No bastion needed; lower attack surface |
| **VPN (Cloud VPN / Interconnect)** | Hybrid: on-prem → GCP via VPN; on-prem user uses own auth | Network-level logs; limited user-level detail | Hourly tunnel charge + data transfer (see pricing) |

Recommendation for the exam: **IAP SSH** is the modern standard for GCP-native workloads; **VPN** for hybrid; **OS Login** for centralized user+device control.

### Separation of duties and least privilege

Concrete examples:

**Scenario:** E-commerce platform with DevOps, Data, and Finance teams.

| Team | Allowed | NOT allowed | Roles |
|------|---------|------------|-------|
| **DevOps** | Deploy functions, update VMs, view logs | No IAM changes, no Finance data access | `roles/cloudfunctions.admin`, `roles/compute.admin` (prod folder), `roles/logging.viewer` |
| **Data** | Query BigQuery datasets, read GCS buckets | No VM/function changes, no Finance tables | `roles/bigquery.dataEditor` (data project), `roles/storage.objectViewer` |
| **Finance** | Create reports from Finance BigQuery dataset, no data export | No function/VM access, read-only on Finance data | Predefined `roles/bigquery.dataViewer` on the Finance dataset + `roles/bigquery.jobUser` on the project (or a custom role with the permissions below, granted on the dataset) |

**Custom role example:**
```
Finance Report Reader (custom):
  - bigquery.datasets.get
  - bigquery.datasets.list
  - bigquery.tables.get
  - bigquery.tables.list
  - bigquery.tables.getData
  - (project level, separately) bigquery.jobs.create via roles/bigquery.jobUser, needed to run queries
  - Intentionally omitted: bigquery.tables.export (and write permissions); to block export even if granted elsewhere, add a deny policy on bigquery.googleapis.com/tables.export
```
Note that BigQuery export to an external location can also be limited with VPC Service Controls (→ Ch11).

### Policy Intelligence tools

#### Recommender (IAM Recommender)

Analyzes permission usage (typically the last 90 days) and recommends IAM role changes (replace an over-broad role with a smaller one, remove unused bindings), plus recommendations for unused service accounts/keys. Example: "User X hasn't used the permissions in Compute Admin; recommend replacing it with a smaller role."

**Use:** Quarterly reviews; identify and remove unused roles; support least-privilege audits.

#### Policy Troubleshooter

Explains: "Why can / can't principal P use permission X on resource R?" Returns **GRANTED** or **NOT GRANTED** with the evaluated allow policies, deny policies and Principal Access Boundary policies that explain the result (it does not evaluate Organization Policy constraints).

**Use:** Debugging permission issues after the fact. (Testing a policy change *before* deployment is **Policy Simulator**.)

#### Policy Analyzer

Answers "which principals have which access to which resource" by analyzing allow policies across the hierarchy (e.g., who can access a bucket, what can a user access). Helps visualize blast radius of a policy change.

**Use:** Identify over-broad roles; audit who has access to sensitive resources.

### Identity for AI workloads

**Agent identity:** AI agents (e.g., on Agent Runtime, formerly Agent Engine / Reasoning Engine) run as service accounts or, newer, as dedicated **agent identities** (a principal type that PAB policies can also target). Never hardcode API keys or embed credentials in prompt engineering.

**Service account for Agent Platform:**
- Agent jobs run as a service account (prefer user-managed over the default Compute SA or the service agent).
- Service account needs least-privilege roles (e.g., `roles/aiplatform.user` plus only the data roles it needs, not `roles/editor` or broad `aiplatform.admin`).
- For multi-tenant agents: use PAB to limit data access.
- For production agents: enforce that credentials are short-lived (1 h or less); disable key-based auth.

**MCP (Model Context Protocol) servers:**
- MCP connections (e.g., integration with Claude, internal tools) need service-account-based auth.
- Auth is again short-lived: GCP credentials refreshed on each call.
- Avoid embedding MCP credentials in environment; use Workload Identity or Secret Manager.

## Decision guides

### Should I use IAM role or Organization Policy?

```
Is the goal to enforce a *resource configuration limit* (e.g., "buckets must be in EU")?
  YES → Organization Policy (gcp.resourceLocations, other constraints, or custom constraint)
  NO ↓

Is the goal to grant *permissions* (e.g., "data team can read BigQuery")?
  YES → IAM allow policy (predefined or custom role)
  NO ↓

Is the goal to *deny* a permission (e.g., "no one can delete prod VMs except SREs")?
  YES → IAM Deny policy (or PAB to limit which projects/orgs principals can reach)
  NO ↓

Consult Policy Troubleshooter or Analyzer.
```

### Choosing a remote-access method

```
Is access human (interactive SSH/RDP)?
  YES ↓
    Is the machine in GCP?
      YES → IAP SSH (simplest, audit-friendly)
          OR OS Login (if you need key management + MFA)
      NO → VPN/Interconnect (hybrid) or IAP for on-prem resources (IAP connector)
  NO → Not applicable

Is access application (service-to-service)?
  YES ↓
    Is both app and GCP resource in GCP?
      YES → Attach service account (VMs) or Workload Identity Federation for GKE (pods)
      NO ↓
        Is app in AWS?
          YES → Workload Identity Federation (AWS role → GCP service account)
        Is app on-prem or other cloud?
          YES → Workload Identity Federation (OIDC) or VPN + attached SA
```

### Service account authentication method

```
Is this a VM in Compute Engine?
  YES → Attach service account at VM creation (metadata server)

Is this a pod in GKE?
  YES → Workload Identity Federation for GKE (direct KSA principal grant, or KSA ↔ GSA impersonation)

Is this a Cloud Function or Cloud Run?
  YES → Attach service account to function/service; no keys

Is this a GitHub Actions CI/CD pipeline?
  YES → Workload Identity Federation + OIDC token exchange

Is this a legacy app that cannot be updated?
  YES (last resort) → Service account keys (rotate every 90 days, store in Secret Manager)

Never hardcode credentials in code or environment variables.
```

## Well-Architected Framework lens

| Pillar | Application |
|--------|------------|
| **Operational Excellence** | IAM Recommender (remove unused roles), Policy Analyzer (audit who has access), Cloud Audit Logs (track all changes), PAM (short-lived elevation with justification). |
| **Security** | Least privilege (custom roles, deny policies, PAB), Organization Policy (enforce encryption, resource location, disable dangerous actions), service account best practices (no keys, short-lived creds, WIF for multi-cloud), OS Login + MFA, context-aware access. |
| **Reliability** | Service account identity tied to resource; no credential sprawl. Workload Identity eliminates key rotation risk. Short-lived tokens auto-refresh; expired tokens fail fast (not hidden bugs from revoked keys). |
| **Cost Optimization** | WIF for hybrid/multi-cloud eliminates key storage and rotation effort; IAP TCP forwarding removes the need for bastion VMs. |
| **Performance** | Token acquisition adds small latency on first use, then tokens are cached until near expiry; attached service accounts (metadata server) are the simplest path. IAM is rarely a performance factor; avoid per-request token minting. |
| **Sustainability** | No direct impact; IAP + OS Login eliminate bastion VMs (fewer resources). |

## AI/GenAI angle

1. **Agent identity:** Agents (Agent Runtime, formerly Agent Engine/Reasoning Engine) run as service accounts or agent identities. Ensure least-privilege roles (not Owner/Editor). For multi-tenant agents with one project per tenant, use Principal Access Boundary to keep them within their own project(s) and prevent cross-tenant data leakage.
2. **MCP and third-party integrations:** When agents integrate with external tools (MCP servers, Apigee proxies, internal APIs), use short-lived credentials (1 h or less). Never embed API keys in agent prompts or memory.
3. **Sensitive data protection in agent workloads:** AI agents often process PII, health records, or financial data. Enforce data location constraints (via Organization Policy) and VPC Service Controls (→ Ch11) to keep agent data within compliance boundaries.
4. **Audit and explainability:** Cloud Audit Logs must track agent actions (which agent, which service account, what resources accessed, when). Use IAM Conditions (time-based, or access-level based) where agent access should be limited to business-hours operations.
5. **Agent supply-chain security:** If agents call external APIs or Model Garden models, validate that the external service is approved (supply-chain security, → Ch11). Use Artifact Registry + Binary Authorization to control which model containers agents run.
6. **Workforce vs. Workload Identity Federation for AI teams:** Data scientists may use Workforce Identity Federation (corporate IdP such as Entra ID/AD FS → GCP console access). Agent applications use Workload Identity Federation (external OIDC → service account). Clearly separate.

## Case-study hooks

### Altostrat Media
- **Problem:** "Operational workflows across GCP + on-prem; auditable & explainable AI systems."
- **IAM solution:** Hybrid identity (Google Cloud Directory Sync for on-prem staff in Cloud Identity, Workload Identity Federation for on-prem metadata-extraction jobs). Agents run with least-privilege service accounts; audit logs tie agent actions to specific workflows.
- **Landing zone:** Folder per workload (transcoding, metadata extraction, recommendations); shared folder for compliance/audit logging.

### Cymbal Retail
- **Problem:** "Human-in-the-loop AI; data security for customer data and agent interactions."
- **IAM solution:** Data team uses custom role (can approve/reject in UI, not export data). Agents run as service accounts in their own project, with a PAB policy keeping them eligible for only that project (plus required shared projects) and narrowly scoped roles (access only their own agent state, not customer data directly). Workforce Identity Federation for employee access (e.g., marketing team viewing dashboards via their Okta account).

### EHR Healthcare
- **Problem:** "HIPAA compliance, centralized auditing, legacy insurer integrations on-prem."
- **IAM solution:** Cloud Directory Sync (on-prem AD → Cloud Identity) for healthcare staff. Separate service accounts for cloud apps vs. on-prem legacy interfaces (Workload Identity Federation). PAM for privileged access to PHI databases (require justification, time-limited). Organization Policy enforces encryption (CMEK via `gcp.restrictNonCmekServices`) and resource location (`gcp.resourceLocations`, e.g. US-only if the company chooses so; HIPAA itself does not mandate US-only storage).

### KnightMotives Automotive
- **Problem:** "Data monetisation (EU GDPR); autonomous-driving dev/test; security paramount."
- **IAM solution:** Landing zone split by team (autonomous engineering, supply chain, sales) and environment (dev/test/prod). GDPR compliance via Organization Policy (`gcp.resourceLocations` = EU locations for EU customer data). Workforce Identity Federation for employee access (employees in AD/Entra ID); Workload Identity Federation for CI/CD pipelines (GitHub → GCP). Autonomous-driving service accounts get time-bound access (dev/test hours only) via IAM Conditions on role bindings, or just-in-time access via PAM.

## Exam traps & keyword → answer

1. **"Default service account vs. custom"** → Default is shared, hard to audit. Always create custom service accounts per workload.
2. **"Can I revoke a role at a child level if parent granted it?"** → No, policies are additive. Use **Deny policy** to override; or move resource to a folder without the parent policy.
3. **"Best way to manage 100 users' access?"** → Groups, not users. Add/remove from group; group membership is a single audit trail.
4. **"Should I use keys or Workload Identity?"** → Workload Identity (or attached service account). Keys are for legacy apps only; they don't auto-refresh and audit doesn't show who used them.
5. **"Can I use a service account key for a Cloud Run service?"** → No, attach the service account directly. Cloud Run fetches credentials from metadata server (Workload Identity-like behavior).
6. **"Organization Policy constraint vs. IAM role"** → Constraint restricts *what configurations are allowed* on resources (for everyone). Role is *who can do it* (allow-based). They are complementary; both are needed for least privilege.
7. **"Replace SSH keys in metadata / shared bastion keys"** → OS Login (IAM-managed, per-user, revocable) and IAP TCP forwarding (no external IPs, no bastion).
8. **"Deny policy overrides allow?"** → Yes. If any deny applies, action is blocked, regardless of allows. (Order: PAB eligibility, then deny, then allow.)
9. **"IAP for internal web app vs. public?"** → IAP is for *internal* apps (only authenticated users). Public apps need application-level auth.
10. **"Workload Identity Federation vs. Cloud Directory Sync"** → WIF is for external identities (AWS, GitHub, OIDC). GCDS syncs on-prem AD users to Cloud Identity (for app-level use and console access). Both can be used together.
11. **"Principal Access Boundary (PAB) limiting a service account?"** → Yes, PAB defines which orgs/folders/projects the principals are *eligible* to access (a ceiling); it never grants. Effective access = allow policy grants AND PAB eligibility AND no deny. PAB works at project/folder/org granularity, not bucket or table.
12. **"Custom role with hundreds of permissions — best practice?"** → No, too broad; prefer predefined roles, and custom roles only with the minimal set; test with Policy Troubleshooter/Recommender.

## Quick-reference cheat table

| Concept | When | How | Example |
|---------|------|-----|---------|
| **Basic role** | Avoid in production (very broad) | Viewer, Editor, Owner | Avoid; use predefined or custom |
| **Predefined role** | Most common; standard job function | Browse roles in console or `gcloud roles list` | `roles/compute.instanceAdmin.v1`, `roles/storage.admin` |
| **Custom role** | Exact least-privilege fit needed | List permissions, create role in org/folder, grant to groups | Data team: `bigquery.dataEditor` + `logging.logEntries.create`, no deletion |
| **Deny policy** | Block dangerous action (override allows) | Define denied principals, denied permissions (FQDN form), exceptions, optional tag condition | Deny `compute.googleapis.com/instances.delete` on prod project; except SRE group |
| **IAM Condition** | Time, resource, or context-based access | Add condition when granting role (time-based, resource-based, access level) | `request.time < timestamp(...)` for temporary access |
| **Principal Access Boundary** | Multi-tenant or third-party sandboxing | Create PAB policy listing eligible orgs/folders/projects; bind it to a principal set | Ensure tenant A service accounts can only reach tenant A's project |
| **Service account key** | Last resort (legacy) | Generate, store in Secret Manager, rotate every 90 days | Avoid; use WIF or attached SA instead |
| **Workload Identity Federation for GKE** | Pod in GKE needing Google API credentials | Enable on cluster (workload pool); grant roles directly to the KSA principal, or bind KSA↔GSA (impersonation) | GKE pod's KSA `pod-sa` is granted `roles/bigquery.dataViewer` directly (or impersonates `gcp-sa`) → calls BigQuery |
| **Workload Identity Federation** | App outside GCP (AWS, Azure, GitHub, on-prem OIDC/SAML) | Create workload identity pool + provider (attribute mapping/condition), grant roles to the pool principal (or let it impersonate a SA), exchange external token via STS | GitHub Actions: token → STS → GCP token → deploy to GCP |
| **OS Login** | Centralized SSH access via IAM (+ optional 2SV) for humans | Enable `enable-oslogin=TRUE` metadata or org policy `compute.requireOsLogin`; grant `roles/compute.osLogin`/`osAdminLogin` | SRE logs into prod VM; audit shows SRE's identity, not a shared key |
| **Organization Policy** | Enforce global limit (resource location, disable key creation, etc.) | Define constraint (list/boolean/custom CEL), attach to org/folder/project | `storage.uniformBucketLevelAccess` enforced (no object-level ACLs) |
| **Cloud Directory Sync** | Sync on-prem AD/LDAP users and groups to Cloud Identity | Install GCDS in data center, configure LDAP connection and sync rules | On-prem staff automatically appear in Cloud Identity |
| **Workforce Identity Federation** | Workforce (employees, partners, contractors) accessing GCP with their external IdP, without syncing accounts | Create workforce identity pool (org-level), configure OIDC/SAML provider, grant roles to pool principals | Partner contractor logs in with Okta → accesses GCP project |
| **IAP** | Secure access to internal web app or SSH/RDP/TCP | Enable IAP on resource (web app) or use TCP forwarding; grant `IAP-secured Web App User` or `IAP-secured Tunnel User` | Employee SSH to prod VM via `gcloud compute ssh VM --tunnel-through-iap` |
| **Recommender (IAM)** | Quarterly least-privilege audit | Review Cloud Console Recommendations tab; approve removals | Remove Viewer role from contractor's account after project ended |
| **Policy Troubleshooter** | Debug permission denied error | Input principal, permission, resource; see which allow/deny/PAB policies explain GRANTED / NOT GRANTED | Why can't data team export BigQuery table? (Deny policy blocks it) |
| **Landing zone (by env)** | Clear prod/staging/dev isolation | Folder per env; separate teams, policies, quotas | Org → Folder: Prod (strict constraints) + Staging (moderate) + Dev (permissive) |
| **Landing zone (by team)** | Team autonomy, shared platform services | Folder per team + shared-services folder | Org → Folder: Data (team) + DevOps (team) + Shared (logging, networking) |

## Practice scenarios

**Scenario 1: Service Account Key Leak**
Your Compute Engine VM has a service account key stored in `/var/secrets/key.json`. A developer accidentally commits the key to a public GitHub repository. What should you do immediately?

A) Delete the key file from the VM; problem solved.
B) Disable or delete the key on the service account (Console/gcloud), then audit Cloud Audit Logs for unauthorized use; afterwards move the VM to an attached service account (no key).
C) Rotate the key on the VM.
D) Notify GitHub to remove the file from history.

**Correct:** B. Disable/delete the exposed key first (stop immediate bleeding; deleting the key invalidates it). Then audit logs to determine if it was used maliciously. A is incomplete (key on GitHub still valid). C only creates a new key; unless the old key is deleted it stays valid. D helps but doesn't prevent unauthorized use.

---

**Scenario 2: Conditional Access for Finance Data**
Finance team needs BigQuery read access to a sensitive dataset, but only during business hours (9 AM–5 PM US/Eastern) and from the office network (10.0.0.0/8). How do you configure this with least privilege?

A) Grant the Finance group `roles/bigquery.dataViewer` on the dataset (plus `roles/bigquery.jobUser`) with an IAM Condition: a time condition (`request.time.getHours(...)`/day-of-week) combined with an Access Context Manager access level for the office network (`request.matchAccessLevels`).
B) Grant the role to the Finance group without conditions; use a bastion VM to control network access.
C) Grant Finance group `roles/bigquery.dataEditor`; use Context-Aware Access to enforce MFA.
D) Use Organization Policy to restrict BigQuery access by IP only.

**Correct:** A. IAM Conditions support time-based attributes, and network/device context via access levels (IAM has no raw `origin.ip` attribute). Predefined viewer role provides least privilege (not Editor). B doesn't enforce time; C is too broad (dataEditor allows updates, not just read). D (Org Policy) is not an access-control mechanism for principals by IP.

---

**Scenario 3: Multi-Cloud App Identity**
Your Node.js app runs on AWS EC2 and calls GCP BigQuery. You want to avoid storing a GCP service account key on EC2. What's the best approach?

A) Generate a GCP service account key, store it in AWS Secrets Manager.
B) Use Workload Identity Federation: create a workload identity pool with an AWS provider in GCP, and grant the EC2 instance's AWS IAM role (as a federated principal, or via impersonation of a GCP service account) access to BigQuery.
C) Use VPN to connect EC2 to GCP; attach a GCP service account to a GKE pod and grant the EC2 app access via Shared VPC.
D) Set up Cloud Directory Sync to sync AWS users to Cloud Identity.

**Correct:** B. WIF is designed for cross-cloud scenarios. The workload presents its AWS credentials, which Google STS exchanges for short-lived GCP tokens (the AWS provider type is built in; no OIDC issuer needed). A stores a GCP key outside GCP (liability). C requires GKE (unnecessary for this scenario) and Shared VPC does not grant API identity. D is for user identity, not service-to-service.

---

**Scenario 4: Organization Policy Constraint**
Your org must comply with GDPR (EU data residency). You need to ensure all Cloud Storage buckets are created in EU regions only. Which organization policy constraint should you use?

A) `storage.uniformBucketLevelAccess`
B) `gcp.resourceLocations` list constraint allowing `in:eu-locations`
C) `compute.skipDefaultNetworkCreation`
D) IAM deny policy on project owners.

**Correct:** B. The resource location restriction (`gcp.resourceLocations`) is the standard organization policy for data residency. A (uniform bucket-level access) is unrelated to location. C (default VPC creation) is unrelated. D (deny policy) doesn't enforce location; it only blocks permissions for principals.

---

**Scenario 5: Service Account for Autonomous Agents**
You're deploying an AI agent (using Agent Platform) that processes customer orders and writes results to BigQuery. The agent should not access any other tables, projects, or external APIs. How do you configure its identity?

A) Attach a single service account with `roles/bigquery.editor` on the entire project.
B) Create a dedicated service account for the agent; grant it write access (e.g., `roles/bigquery.dataEditor`, or a custom role with `bigquery.tables.updateData`) only on the target table/dataset, plus `roles/bigquery.jobUser` if it runs jobs. Optionally add a Principal Access Boundary so the agent's project's service accounts can only reach the intended project(s).
C) Create two service accounts: one for orders, one for writes; grant different roles.
D) Use Cloud Identity to create a user for the agent.

**Correct:** B. A dedicated service account with resource-level grants gives least privilege; PAB adds an outer boundary at project/folder/org level (it cannot restrict to a single table). A is too broad (editor on whole project). C is over-engineered for this requirement. D (Cloud Identity) is for human users, not agents.

---

**Scenario 6: Auditing IAM Changes**
Your compliance team must prove that only SREs can approve IAM changes to production projects. Which tools should you use?

A) Organization Policy `iam.disableServiceAccountCreation`.
B) IAM role `roles/iam.roleAdmin`; grant only to SRE group.
C) Cloud Audit Logs (Admin Activity) + access control via custom IAM roles or deny policies; audit trail shows who made changes.
D) Workload Identity Federation to external SRE IdP.

**Correct:** C. Cloud Audit Logs show the audit trail (who, what, when); custom roles + deny policies enforce that only SREs can make changes. B alone doesn't create an audit trail. A prevents new SAs (too restrictive). D is for authentication, not authorization.

---

**Scenario 7: Temporary Access for Contractor**
A contractor needs to debug a GKE cluster in your staging environment for 5 days. You want to grant temporary access, with justification required and automatic expiration. What's the best approach?

A) Add contractor to a "staging-readers" group; grant the group `roles/container.viewer`.
B) Use Privileged Access Manager (PAM): create an entitlement with `roles/container.viewer` (or developer role if debugging needs it), maximum duration of 5 days, justification required (and approval if desired). PAM enforces the time limit and logs the request.
C) Create a service account for the contractor; share the key via email.
D) Grant contractor direct IAM role on the project with an IAM Condition (time-based).

**Correct:** B. PAM is designed for temporary, justified privilege elevation. It auto-expires and has full audit trail. A doesn't provide time-limit or justification. C shares credentials (security risk). D works for expiry but lacks PAM's justification/approval workflow and request audit trail. (PAM entitlements allow a maximum grant duration of 7 days; longer needs are re-requested.)


## Sources

- https://cloud.google.com/iam/docs
- https://cloud.google.com/iam/docs/policy-types
- https://cloud.google.com/iam/docs/service-account-overview
- https://cloud.google.com/iam/docs/workload-identity-federation
- https://cloud.google.com/iam/docs/using-iam-securely
- https://cloud.google.com/resource-manager/docs/cloud-platform-resource-hierarchy
- https://cloud.google.com/iam/docs/deny-overview
- https://cloud.google.com/iam/docs/conditions-overview
- https://cloud.google.com/iam/docs/principal-access-boundary-policies
- https://cloud.google.com/iam/docs/pam-overview
- https://cloud.google.com/docs/authentication/application-default-credentials
- https://cloud.google.com/chrome-enterprise-premium/docs/overview
- https://cloud.google.com/docs/authentication/api-keys
- https://cloud.google.com/architecture/identity/federating-gcp-with-active-directory-synchronizing-user-accounts
- https://cloud.google.com/iam/docs/workforce-identity-federation
- https://cloud.google.com/kubernetes-engine/docs/concepts/workload-identity
- https://cloud.google.com/resource-manager/docs/organization-policy/restricting-service-accounts
- https://cloud.google.com/organization-policy/docs
- https://cloud.google.com/organization-policy/docs/list-constraints
- https://cloud.google.com/organization-policy/docs/custom-constraints
- https://cloud.google.com/iap/docs
- https://cloud.google.com/compute/docs/oslogin
- https://cloud.google.com/policy-intelligence/docs/overview
