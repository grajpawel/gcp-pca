# Chapter 03 — Networking

## Exam objectives covered

From exam guide v6.1 (sections 1.3 & 2.1):
- **1.3 Network, storage, compute:** integration with on-prem/multicloud; cloud-native networking (VPC, peering, firewalls, load balancers, routing, container networking, Shared VPC, Private Service Connect); compute choices mapping to GKE/Cloud Run/Cloud Run functions.
- **2.1 Network topologies:** hybrid (on-prem) networking; multicloud incl. Google-Cloud-to-Google-Cloud; security protection (IPS, access control, firewalls); VPC design & load balancing.

## What the exam really tests

- **VPC topology decisions under constraints:** When to use Shared VPC vs VPC peering vs Private Service Connect vs PSA? Which firewall layer (rules, hierarchical, network policies)?
- **Load balancer selection (the hardest LB question):** Global vs regional, external vs internal, L4 vs L7, proxy vs passthrough, Application LB vs proxy Network LB vs passthrough Network LB—and why not the others?
- **Hybrid & multicloud connectivity:** Direct VPC egress vs Serverless VPC Access vs Cloud VPN vs Dedicated Interconnect—trade-offs in bandwidth, latency, cost, SLA, setup time.
- **Container & serverless networking:** GKE VPC-native (alias IPs, Services, Ingress, Gateway API), Cloud Run's implicit networking, private endpoints via PSC.
- **Security integration:** Firewall policies (rules vs hierarchical vs network policies), Cloud Armor placement, Cloud IDS/NGFW in the stack, VPC Service Controls (→ Ch11).
- **DNS, NAT, CDN as architectural decisions:** When does Cloud NAT belong? When is Cloud DNS private peering better than VPN? Cloud CDN's interaction with load balancers.
- **Cost & operational trade-offs:** Shared VPC reduces sprawl but adds governance; Interconnect is fast but has setup cost and lead time; Cloud NAT scales but counts egress bytes.

## Core concepts

### VPC & subnets

**VPC (Virtual Private Cloud)** is a logically isolated network. GCP offers two creation modes:
- **Auto mode:** GCP creates a subnet in each region (10.128.0.0/9), sized /20. Fast to start; risk of IP collisions if peering many VPCs.
- **Custom mode:** You create subnets manually (CIDR-agnostic), enabling careful IP planning. Required for fine-grained multi-region or peering-at-scale designs. Exam preference: custom mode for any serious architecture.

**Subnets** are regional (not zonal); VMs in the same subnet can communicate directly. Primary internal IP is assigned from subnet CIDR; additional network interfaces can have secondary ranges (used by GKE alias IPs).

**Routes** determine egress for traffic not matching any subnet. Default route (0.0.0.0/0) goes to default internet gateway. Custom routes can point to:
- Next hop IP (peer VM)
- VPC peering connector
- Hybrid connectivity (VPN tunnel / Interconnect VLAN attachment, usually learned dynamically via Cloud Router BGP)
- Internal passthrough Network LB (e.g., as next hop to a network virtual appliance)

Note: Cloud NAT is **not** a route next hop; it is a software-defined service configured on a Cloud Router that translates source IPs for VMs without external IPs whose traffic leaves via the default internet gateway route.

### Firewall & hierarchical policies

**VPC Firewall Rules** are project-scoped, stateful, applied per network. Match on source/dest CIDR, ports, protocols; can reference service accounts or network tags. **Ingress** and **egress** rules both exist; default-deny ingress, allow-all egress.

**Hierarchical Firewall Policies** (HFP) apply at the folder or organization level, enforced before VPC rules. Each rule has an action of allow, deny or goto_next (delegate to the next level); an allow/deny decision made at a higher level is final and cannot be overridden by lower levels. Evaluation order: organization policy → folder policies → VPC firewall rules / global network firewall policies → regional network firewall policies (implied rules last). Decision: use HFP for org-wide mandatory rules (e.g., block external DB access); VPC rules for fine-grained app-level control.

**Global / regional network firewall policies** (the successor to classic VPC firewall rules, part of **Cloud NGFW**) attach to a VPC network, support IAM-governed **secure tags** (instead of network tags), FQDN / geo / threat-intelligence match objects and address groups, and can be shared across the hierarchy. **Cloud NGFW** tiers: Essentials, Standard, Enterprise (Enterprise adds TLS inspection and IPS / threat prevention that can block, using Palo Alto Networks technology). They are not related to Cloud Armor, which protects load-balancer frontends.

### Shared VPC & VPC peering

**Shared VPC** (Host Project + Service Projects) centralizes network & security. One team owns the host VPC; multiple projects' workloads run on it. Benefits: single audit point, centralized firewall, cross-project service discovery via internal DNS. Downside: requires trust and governance; slower for independent teams. Exam pattern: "multiple teams/projects, centralized network and security control (IAM separation: network admins vs service-project admins), shared RFC 1918 space" → Shared VPC (same organization only).

**VPC Peering** connects two VPCs (same or different projects/orgs) directly (**not transitive**; subnet IP ranges must not overlap). The peering must be created from **both** sides to become active. Useful for: federated multi-team architectures, partner integrations, SaaS/producer-consumer VPCs. Traffic stays on Google's internal network with the same latency/throughput as inside one VPC; peering itself is free, normal network egress pricing applies (e.g., cross-zone / cross-region traffic is charged as if inside one VPC).

### Private Google Access & Private Service Connect

**Private Google Access** (PGA) allows VPC resources without public IPs to reach Google APIs (e.g., Cloud Storage, Pub/Sub) via internal routes. Requires enabling on the subnet (plus DNS and a route to the API VIPs, e.g., default internet gateway route for `private.googleapis.com` 199.36.153.8/30 or `restricted.googleapis.com` 199.36.153.4/30; the restricted VIP is used with VPC Service Controls). Does NOT work for non-Google external APIs.

**Private Service Connect** (PSC) lets consumers reach services through an internal IP address in their own VPC, without peering and with overlapping IP ranges allowed:
- **PSC endpoints for Google APIs** (consumer): a global internal IP in your VPC that targets Google APIs (all-apis or the vpc-sc bundle); reachable from on-prem over VPN/Interconnect too, can use custom DNS names.
- **PSC endpoints / backends for published services** (consumer): a forwarding rule (endpoint) or a PSC NEG behind an internal/external load balancer (backend) connects to a producer's **service attachment** (Google-managed services such as Cloud SQL / AlloyDB / Memorystore, partner SaaS or your own services).
- **Producer side:** publish your service by creating a service attachment (fronted by an internal load balancer) and approving consumer connections.

Exam trap: "Private access to Google APIs from VMs without external IP" → Private Google Access (simplest, free) or a PSC endpoint for Google APIs (when you need a dedicated internal IP, on-prem access via a specific IP, or per-environment control). "Expose my service privately to other VPCs/orgs, overlapping IPs, one-way connectivity" → PSC (service attachment), not peering.

**Private Services Access (PSA):** older mechanism that creates a VPC peering to a Google-managed service-producer VPC using an allocated internal range (Cloud SQL, Memorystore, etc.); because peering is non-transitive, on-prem reachability needs custom route export/advertisement. Many managed services now also offer PSC, which is the preferred option for new designs where supported.

### Direct VPC egress & Serverless VPC Access

**Direct VPC Egress:** Cloud Run services, jobs and Cloud Run functions (2nd gen) place instances' network interfaces directly in your VPC subnet, with no connector VMs. Outbound traffic routes via your VPC, enabling: hybrid connectivity (on-prem access via VPN/Interconnect), static egress IP (via Cloud NAT), and VPC firewall rules (using network tags). Google documents it as lower latency and higher throughput than connectors, with no connector instance cost.

**Serverless VPC Access connector:** managed connector instances (VMs in a /28 subnet) bridging serverless to the VPC. Still supported, but Direct VPC egress is the recommended default; connectors remain relevant for products that do not support Direct VPC egress (e.g., App Engine standard, Cloud Run functions 1st gen) and cost money while running.

Decision: New Cloud Run / Cloud Run functions → Direct VPC Egress. Existing connectors → migrate where possible.

### Cloud NAT & Cloud VPN

**Cloud NAT** (managed, software-defined NAT, configured on a Cloud Router; no proxy VMs): outbound-initiated connections only (return traffic allowed, no unsolicited inbound). Provides egress source-IP translation (auto or reserved static IPs) for resources without external IPs. Billed per gateway/VM-hour plus data processed (plus normal egress). Use for: private GKE nodes/pods needing internet access, VMs/serverless (via Direct VPC egress) needing a fixed egress IP for partner allowlists. It is **not** for reaching on-prem (use VPN/Interconnect) and not needed for Google APIs (use Private Google Access). *Private NAT* is a separate mode for private-to-private translation (overlapping ranges, → NCC section).

**Cloud VPN** (HA VPN recommended; Classic VPN is being phased out): IPsec tunnel(s) over the public internet between on-prem/branch (or another cloud) and GCP. ~3 Gbps per tunnel (combined ingress+egress); latency varies (internet-dependent). **99.99 % SLA** requires the HA VPN topology: two interfaces on the HA VPN gateway, each with tunnels to the peer (peer with redundant devices/interfaces, e.g., four tunnels to two peer gateways, or per Google's documented topologies) and dynamic routing (BGP via Cloud Router). Setup ~hours to days. Use for: hybrid connectivity with moderate bandwidth, branch offices, backup path for Interconnect.

### Cloud DNS, Cloud Router & BGP

**Cloud DNS** manages DNS records for your domains. **Private zones** are VPC-scoped, returning internal IPs to that VPC; can be attached to multiple VPC networks (including across projects). **Forwarding** (outbound) routes queries to on-prem DNS; **inbound server policy** lets on-prem resolve GCP private zones; **DNS peering** lets one VPC use another VPC's DNS view (independent of VPC Network Peering). Common exam pattern: "resolve internal names across many VPCs / hybrid" → private zones attached to the VPCs, DNS peering and forwarding zones, usually managed centrally from a hub or Shared VPC host project.

**Cloud Router** (managed BGP router) runs BGP to advertise your VPC CIDR to on-prem (via VPN/Interconnect) and learn on-prem routes dynamically. Enables failover and multi-path routing. Must be in region with hybrid connectivity.

### Interconnect (Dedicated, Partner, Cross-Cloud)

**Dedicated Interconnect:** Direct physical connection to Google at a colocation facility. Circuits of 10, 100 or 400 Gbps (up to 8 circuits per connection); VLAN attachments from 50 Mbps up to the connection capacity. SLA is **topology-based**: 99.9 % with two connections in two different edge availability domains of one metro; **99.99 %** with four connections (two per metro in two metros, each in different edge availability domains) and Cloud Routers in two regions with global routing. Setup: weeks (cross-connect provisioning). Use for: large enterprises, high-throughput hybrid, predictable low latency.

**Partner Interconnect:** Through a supported service provider (you need no physical presence at a Google colo). VLAN attachment capacities 50 Mbps to 50 Gbps (not all sizes offered by every provider). SLA is also **topology-based**: 99.9 % (two VLAN attachments in two edge availability domains of one metro) or **99.99 %** (four VLAN attachments across two metros). The Google SLA covers only Google-to-provider connectivity. Setup: days–weeks. Use for: no colo presence, or <10 Gbps needs.

**Cross-Cloud Interconnect:** Managed, dedicated Layer-2 connectivity between Google Cloud and another cloud (AWS, Azure, Oracle OCI, Alibaba Cloud) over Google-provisioned links (10 or 100 Gbps). Google handles the physical link on the other cloud's side. Same 99.9 % / 99.99 % topology model as Interconnect. It provides private connectivity, **not** automatic failover between providers; resilience comes from redundant attachments/regions and BGP.

| Aspect | Cloud VPN | Dedicated | Partner | Cross-Cloud |
|--------|-----------|-----------|---------|-------------|
| Bandwidth | ~3 Gbps per tunnel | 10/100/400 Gbps circuits | 50 Mbps–50 Gbps per attachment | 10 or 100 Gbps |
| SLA | 99.99 % (HA VPN topology) | 99.9 % / 99.99 % (topology) | 99.9 % / 99.99 % (topology) | 99.9 % / 99.99 % (topology) |
| Setup time | Hours–days | Weeks | Days–weeks | Weeks |
| Cost | Lowest | Highest | Mid | Premium |
| Latency | Internet-dependent | Low, predictable (distance-dependent) | Low, predictable (provider-dependent) | Low, predictable |
| Encryption | IPsec built in | Not encrypted by default (MACsec or HA VPN over Interconnect optional) | Not encrypted by default | MACsec supported ⚠ verify |
| When NOT to use | Very high throughput | Small branch office | Provider not available at location | Not connecting to another cloud |

### Load Balancer decision tree

Load balancers sit in the network stack; choice hinges on scope, layer, and protocol:

Current official naming: **Application Load Balancers** (L7 HTTP/HTTPS; formerly "HTTP(S) LB"), **Proxy Network Load Balancers** (L4 TCP with optional SSL offload; formerly "TCP Proxy / SSL Proxy"), **Passthrough Network Load Balancers** (L4 TCP/UDP plus ESP/GRE/ICMP; external or internal). Each exists in external/internal and global/cross-region/regional deployment modes (global and classic variants are external; cross-region variants are internal). Google Cloud has **no "Gateway Load Balancer"** (that is an AWS product); to insert appliances use an internal passthrough Network LB as a next hop, or Cloud NGFW / network security integration.

**Global vs Regional:**
- **Global:** global external Application LB and global external proxy Network LB (single anycast IP, cross-region HA; the anycast/global behaviour requires **Premium Network Tier**). Cross-region internal Application / proxy Network LBs give multi-region HA behind internal IPs.
- **Regional:** regional external/internal Application LB, regional proxy Network LB, passthrough Network LB (external/internal) (single region, no global hop). Regional external LBs can run in Standard Tier.

**External vs Internal:**
- **External:** Internet-facing; get public IP.
- **Internal:** VPC-only (incl. clients connected via VPN/Interconnect); get internal IP.

**Layer & Protocol:**
- **L7 (Application):** Application LB handles Host/Path routing, TLS termination, Cloud CDN, Cloud Armor, IAP. Proxy-based (terminates the client connection). Use for: microservices, content-based routing, WAF attachment.
- **L4 proxy:** Proxy Network LB terminates TCP (optional TLS offload) for non-HTTP protocols (e.g., SMTP over TLS, MQTT, custom TCP).
- **L4 passthrough:** Network LB (connection-based) forwards packets unmodified: client source IP preserved, TCP/UDP and other IP protocols, suitable for game servers, VoIP/UDP, high-throughput non-HTTP.

**Decision tree (exam favorite):**
1. **Is the app global (multi-region)? Or cross-region HA required?**
   - Yes → Investigate global external Application LB or global external proxy Network LB (Premium Tier).
   - No → Proceed to regional options.

2. **Does the app need Layer 7 routing (Host, Path, or TLS termination)?**
   - Yes → HTTP(S) LB (external) or Internal HTTP(S) LB.
   - No → Proceed to Layer 4.

3. **Is the app protocol HTTP/HTTPS?**
   - Yes → Use HTTP(S) LB.
   - No (TCP, UDP, non-HTTP) → Proceed to Layer 4.

4. **Is the target internal (VPC only) or external?**
   - Internal → Internal passthrough Network LB or Internal Application LB.
   - External → passthrough Network LB (external), proxy Network LB (if global HA / TLS offload), or Application LB.

5. **Is it non-HTTP, UDP, or does it need to preserve the client source IP?**
   - Yes → passthrough Network LB (regional).
   - No, or standard HTTP → Application LB.

**Load Balancer Comparison Table:**

| Type | Scope | Layer | Protocol | External | Proxy | Multi-region | When to use |
|------|-------|-------|----------|----------|-------|------|------------|
| External Application LB (HTTP(S)) | Global (also regional) | L7 | HTTP/HTTPS | Yes | Yes | Yes (global mode) | Web apps, content routing, WAF, CDN. |
| External proxy Network LB (SSL/TCP Proxy) | Global (also regional) | L4 | TCP (optional TLS offload) | Yes | Yes | Yes (global mode, Premium Tier) | Non-HTTP TCP/TLS (SMTP, MQTT, custom), global HA. |
| External passthrough Network LB | Regional | L4 | TCP/UDP/ESP/GRE/ICMP | Yes | No (pass-through) | No | UDP/VoIP/game servers, preserve client IP, high throughput. |
| Internal passthrough Network LB | Regional | L4 | TCP/UDP/other IP | No (internal) | No (pass-through) | No (single-region) | VPC-only L4 services, next hop for appliances. |
| Internal Application LB | Regional (also cross-region) | L7 | HTTP/HTTPS | No (internal) | Yes | Only cross-region mode | VPC-only, path/host routing, TLS. |
| Internal proxy Network LB | Regional (also cross-region) | L4 | TCP | No (internal) | Yes | Only cross-region mode | Internal TCP services incl. on-prem/multicloud backends. |

### Cloud CDN & Cloud Armor

**Cloud CDN** (Content Delivery Network) caches content at Google's edge locations. Attached to HTTP(S) LB. Benefits: reduced origin load, lower latency for end-users, egress cost savings. Caching headers (Cache-Control / Expires) and CDN cache modes determine TTL. Use for: static assets, images, APIs with cacheable responses.

**Cloud Armor** (Web Application Firewall + L3/L4/L7 DDoS protection) attaches to external Application LBs (and also proxy / passthrough Network LBs via network edge policies). Policies can match on IP, geography, header, query string, and custom rules. Offers preset rules (SQLi, XSS, Log4j). Integrates with SIEM via Cloud Logging. (Deep dive → Ch11.)

### Network Connectivity Center & multicloud

**Network Connectivity Center** (NCC): hub-and-spoke connectivity management. The **hub is a Google Cloud resource**; spokes are **VPC spokes** (native NCC attachment, no VPC peering needed), **hybrid spokes** (Cloud VPN / Interconnect VLAN attachments, enabling site-to-site traffic over Google's backbone), and **router appliance spokes** (third-party SD-WAN/NVA). Provides transitive connectivity between spokes (VPC spokes in mesh or star topology), unlike VPC peering. Alternative to manually peering all VPCs (n(n-1)/2 non-transitive peerings). **VPC spokes still require non-overlapping IP ranges**; for overlapping ranges use **Private NAT** (Cloud NAT private mode) with NCC, or PSC.

Exam pattern: "organization with 10 regional VPCs and on-prem data center; needs all to communicate with minimal config" → Network Connectivity Center.

**Multicloud connectivity:** Google Cloud + AWS/Azure. Options:
- **Cross-Cloud Interconnect** (Google-provisioned dedicated Layer-2 link to AWS/Azure/OCI/Alibaba).
- **HA VPN between clouds** (IPsec + BGP; cheaper, internet-based; NCC can tie them together).
- **Application-level federation:** Message queue (Pub/Sub) or API gateway (Apigee) bridges clouds.

### GKE & container networking

**GKE VPC-native** (recommended): pods and services live in VPC address space. Each pod gets a VPC IP (not cluster-internal overlay). Enables network policies, VPC-level security, and simplified debugging. Uses **alias IPs** (secondary CIDR ranges per subnet).

**Services & Ingress:**
- **ClusterIP Service:** cluster-internal virtual IP, resolved by cluster DNS (kube-dns / Cloud DNS for GKE).
- **LoadBalancer Service:** provisions a passthrough Network LB (external, or internal with an annotation).
- **Ingress:** Layer 7 routing (K8s API object); maps to HTTP(S) LB + Google Cloud.
- **Gateway API** (Kubernetes standard, replacing Ingress): more expressive routing, better multi-cluster support.

**Network Policies:** Kubernetes-native ACLs; restrict pod-to-pod traffic (north-south and east-west). Must be enabled on the cluster (enforced by GKE Dataplane V2 (eBPF/Cilium), where it is built in, or Calico). Exam trap: "Secure pod communications in a multi-tenant cluster" → Network Policies (not VPC Firewall Rules, which work at VM/CIDR level via tags/service accounts and cannot select pods by label; pod-to-pod traffic on the same node never crosses the VPC firewall).

### Network Intelligence Center & Cloud IDS

**Network Intelligence Center:** observability and troubleshooting for GCP networks. Tools: Connectivity Tests (simulate end-to-end reachability/config analysis), Network Topology, Performance Dashboard (latency, packet loss), Firewall Insights. **VPC Flow Logs** (a VPC/subnet feature, sampled 5-tuple flow metadata exported to Cloud Logging) feed analysis. Use for: RCA of connectivity issues, traffic analysis. Flow logs are sampled metadata, not a complete audit trail of who accessed what (that is Cloud Audit Logs → Ch12).

**Cloud IDS** (Intrusion Detection System): Google-managed threat detection (Palo Alto Networks technology) that inspects mirrored traffic (Packet Mirroring) through a zonal IDS endpoint. Integrates with Cloud Logging and Security Command Center. Exam mention → Ch11 (deep dive); it is a detection layer, not blocking. For inline blocking IPS use **Cloud NGFW Enterprise** (and Cloud Armor for LB-edge L7/DDoS blocking).

### Network tiers & IP addressing

**Premium Tier (default):** traffic enters/leaves Google's global backbone at the edge point closest to the user ("cold potato"), giving lower latency and more predictable performance; required for global external LBs with a single anycast IP. Higher egress cost.
**Standard Tier:** traffic travels over the public internet between the user and the Google region ("hot potato"), cheaper egress, regional scope only (regional external LBs and regional external IPs; no global LB / anycast).

**IP planning (de-emphasized on new exam but exam still tests it):** Use RFC 1918 ranges (10.0.0.0/8, 172.16.0.0/12, 192.168.0.0/16). Plan for growth: reserve space for future VPCs, peering, Shared VPC. Avoid overlaps with on-prem and partner networks.

### Hub-and-spoke vs mesh topologies

**Hub-and-Spoke:** central hub (on-prem or primary VPC) with satellite VPCs/branches. Traffic flows hub → spoke or spoke → hub. Simple, centralized security, reduced peering complexity. Downside: hub is a bottleneck.

**Mesh:** every entity can communicate directly with every other (full peering, or via Network Connectivity Center). Higher complexity but lower latency and hub dependency. Common in large enterprises or multicloud (e.g., service mesh like Istio).

## Decision guides

**VPC design:**
- Single project, single region, under 5 services? → Auto mode VPC, simplest.
- Multi-region or plans for peering? → Custom mode VPC, explicit CIDR planning.
- Multiple teams/projects requiring centralized security? → Shared VPC (host project + service projects).
- Independent teams, federation preferred? → Separate VPCs + VPC peering + Network Connectivity Center (optional hub).

**Firewall layers:**
- Org-wide mandatory rules (e.g., block all external DB access)? → Hierarchical Firewall Policies.
- Fine-grained app/service security? → VPC Firewall Rules.
- Network-level policy with tags/FQDN/threat intel, IPS (Cloud NGFW Enterprise) or org-wide sharing? → Network Firewall Policies (Cloud NGFW).

**Private connectivity to Google and managed services:**
- Google APIs from private VMs → Private Google Access (simplest) or PSC endpoint for Google APIs.
- Managed services (Cloud SQL, Memorystore) → Private Services Access (PSA, peering-based) or PSC where supported.
- Your own / partner service across VPCs or orgs → PSC service attachment.
- Internet SaaS APIs (not Google)? → Cloud NAT (outbound, optional static IP). On-prem APIs → VPN/Interconnect.

**Hybrid connectivity:**
- Up to a few Gbps, quick setup, encrypted by default? → HA VPN (99.99 % with HA topology).
- 10 Gbps+, predictable latency, colocation presence? → Dedicated Interconnect (99.99 % needs the four-connection two-metro topology).
- No colo presence or <10 Gbps dedicated-like needs? → Partner Interconnect.
- Connecting to AWS/Azure/OCI/Alibaba privately? → Cross-Cloud Interconnect.

**Load balancer:**
- Website or REST API → external Application LB.
- Mobile game / real-time UDP system (non-HTTP) → passthrough Network LB (regional) or proxy Network LB (global TCP).
- Microservices with path/host routing → Application LB + Ingress/Gateway API.
- In-line IDS/firewall appliance → internal passthrough Network LB as next hop, or Cloud NGFW Enterprise.

**Serverless networking:**
- Cloud Run needing on-prem access? → Direct VPC Egress.
- Cloud Run functions needing VPC? → Direct VPC Egress (2nd gen).
- External APIs only, no on-prem? → No VPC needed; public internet.

**Container networking:**
- GKE running in production → VPC-native, Network Policies, Dataplane V2.
- Pod isolation required → Network Policies (requires VPC-native + CNI).
- External users → LoadBalancer Service or Ingress (HTTP(S) LB).

**Observability:**
- "Which service can't reach which other service?" → VPC Flow Logs + Connectivity Tests.
- Detecting intrusions → Cloud IDS.
- DDoS/WAF → Cloud Armor.

## Well-Architected Framework lens

| Pillar | Networking decisions |
|--------|---------------------|
| Operational Excellence | Network Connectivity Center (reduces manual peering), Cloud DNS (automated FQDN management), VPC Flow Logs (RCA), Connectivity Tests, Infrastructure as Code (Terraform for VPC/firewall config) with drift detection. |
| Security & Privacy | Hierarchical Firewall Policies (mandatory rules), VPC Service Controls (perimeter for API access), Private Service Connect (narrow access), Cloud Armor (DDoS + WAF), Cloud IDS (threat detection), Cloud NGFW (IPS). Use secure tags / service accounts as firewall targets. |
| Reliability | HA for hybrid links: Interconnect 99.99 % topology (4 connections, 2 metros), HA VPN as backup. Cloud NAT is regional and software-defined (size ports/IPs for load). Global LBs with multi-region backends. LB health checks (drop unhealthy backends). |
| Cost Optimization | Premium Tier vs Standard (evaluate performance vs egress cost). Keep traffic in-region/in-zone; Cloud CDN (egress savings for cached content). Shared VPC (central governance, fewer duplicate NAT/VPN/Interconnect). Review unused LBs, IPs and idle VPN tunnels / attachments. |
| Performance Optimization | Premium Tier (backbone routing) for global users. Dedicated/Partner Interconnect for predictable hybrid latency. Cloud CDN for edge caching. Place resources close to users; use regional internal LBs for VPC-local services. |
| Sustainability | Not a major network lever; place workloads in low-carbon regions, cache at the edge (Cloud CDN) and avoid unnecessary cross-region traffic. |

## AI/GenAI angle

- **Hybrid ML pipelines:** Agent Platform (formerly Vertex AI) on-prem data → Cloud. Requires hybrid connectivity (VPN or Interconnect) and Private Service Connect for Agent Platform APIs.
- **LLM inference at edge:** Deploy Gemini or custom LLM models to on-prem GPUs, sync prompts/results via secure hybrid link (VPN + Cloud NAT). Private Service Connect enables secure API access.
- **Federated learning:** VPC peering or Shared VPC for cross-project model training coordination. Network policies ensure pod-to-pod isolation in GKE multi-tenant clusters.
- **GenAI APIs:** Cloud Storage, BigQuery (for RAG), Agent Platform all can be reached privately via Private Google Access or Private Service Connect endpoints (no public internet path).
- **Sensitive data in LLMs:** VPC Service Controls (perimeter) restrict Agent Platform API access to projects/identities inside the perimeter (mitigating data exfiltration). Cloud Armor can protect your own LB-fronted AI/inference endpoints from abuse (rate limiting, WAF).

## Case-study hooks

**Altostrat Media** (podcasts, AI-driven content):
- Legacy on-prem ingestion & archival; GCP for scalable processing & storage. → Hybrid connectivity: Dedicated Interconnect or Partner Interconnect for high-throughput media ingest.
- GKE for transcoding, metadata extraction, recommendation models (Agent Platform). → VPC-native GKE, Network Policies for pod isolation, Private Service Connect to Agent Platform APIs.
- 24/7 chatbot (Gemini). → Cloud Run + Direct VPC Egress for on-prem DB access, PSC endpoint (or Private Google Access) to Google APIs.

**Cymbal Retail** (e-commerce, catalog enrichment):
- Hybrid: on-prem (legacy ETL, SFTP) + cloud (modern microservices, Agent Platform). → Partner Interconnect or Cloud VPN (cost/latency trade-off).
- Shared VPC for central catalog & customer services. → Isolate per-team service projects, centralized firewall policies.
- Cloud Run for image generation, LLM-driven product discovery. → Direct VPC Egress for on-prem inventory sync.

**EHR Healthcare** (multi-region HA, HIPAA):
- ≥99.9 % uptime, customer-facing. → Redundant Global HTTP(S) LB, multi-region backend (GKE or Cloud Run in 3+ regions).
- Hybrid insurer integrations (not moving on-prem). → Dedicated or Partner Interconnect (99.99 % topology) for reliable on-prem link, HA VPN as backup. Cloud NAT for egress to 3rd-party internet insurance APIs.
- HIPAA audit logging. → Cloud Audit Logs (Admin Activity + Data Access) routed by a log sink to a Cloud Storage bucket with a locked retention policy (Bucket Lock); VPC Flow Logs add network-level visibility but are sampled metadata, not the audit trail.
- Private access to sensitive DBs (Cloud SQL). → Private Service Connect or PSA; deny public IP.

**KnightMotives Automotive** (autonomous vehicles, real-time):
- Dealer / plant connectivity (many sites, poor rural connectivity). → HA VPN per site (or SD-WAN via NCC router appliance spokes), Interconnect for plants↔HQ/cloud if volume justifies.
- EU GDPR compliance (data residency). → Subnets/resources in EU regions (e.g., europe-west3 Frankfurt), organization policy constraint on resource locations. Private Service Connect / VPC Service Controls for data APIs.
- Multicloud AI training (on-prem GPU cluster + GCP). → Network Connectivity Center hub with hybrid spokes (VPN/Interconnect to on-prem) and VPC spokes; Cross-Cloud Interconnect if also AWS/Azure.
- Real-time autonomous-driving simulation: low, predictable latency / high throughput. → Dedicated Interconnect for training data sync, Premium Tier for user-facing traffic.

## Exam traps & keyword → answer

1. **"Minimize operational overhead for hybrid connectivity"** → Cloud VPN (managed, no physical provisioning, quick setup). Not Interconnect (needs colo/partner coordination and lead time) unless bandwidth/latency demands it.
2. **"Resolve internal names across many VPCs and on-prem"** → Cloud DNS private zones attached to the VPCs + DNS peering / forwarding zones (central hub project). Not host files or per-VM resolvers. (DNS has no "lowest latency" angle; do not pick on that basis.)
3. **"Pod-to-pod traffic filtering in GKE"** → Network Policies (Kubernetes-native; Dataplane V2). Not VPC Firewall Rules (VM/CIDR level, cannot select pods by label).
4. **"Secure Cloud Run access to on-prem database"** → Direct VPC Egress + VPN/Interconnect route to on-prem. Serverless VPC Access connector also works but is the older option (extra connector VMs/cost, lower throughput).
5. **"Global users, lowest latency, single IP, multi-region failover"** → Global external Application LB (anycast, Premium Tier) with multi-region backends (+ Cloud CDN). Not regional passthrough Network LB (single region). No LB offers "sub-millisecond" latency; latency is bounded by distance.
6. **"Transitive routing: 10 VPCs + on-prem hub"** → Network Connectivity Center (VPC + hybrid spokes). Not VPC peering alone (non-transitive, n(n-1)/2 peerings).
7. **"Sensitive data in Cloud Storage; no public IPs; prevent exfiltration"** → Private Google Access (or PSC endpoint for Google APIs) + VPC Service Controls perimeter + IAM/org policy. Cloud Armor is irrelevant here (it protects LB-fronted web backends, not Cloud Storage API access).
8. **"HIPAA audit trail"** → Cloud Audit Logs (Admin/Data Access) → log sink → Cloud Storage with a locked retention policy (Bucket Lock) or a locked log bucket. VPC Flow Logs are sampled network metadata (useful for network forensics) and are not an access audit trail.
9. **"Application Layer 7 routing (Host header)"** → Application LB (HTTP(S)). Not Network LB (L4, no Host header visibility).
10. **"Firewall rule applies to entire org immediately"** → Hierarchical Firewall Policy. Not VPC Firewall Rules (network/project-scoped).
11. **"Cost-effective hybrid: <100 Mbps, non-critical"** → Cloud VPN. Not Dedicated Interconnect (cost/lead time disproportionate).
12. **"Private connectivity between Google Cloud and AWS/Azure"** → Cross-Cloud Interconnect (dedicated Layer-2, Google-provisioned) or HA VPN for low cost. It is a connectivity product, not an automatic failover between providers; resilience still needs redundant attachments/BGP design.
13. **"Multi-region global anycast IP"** → Global external Application LB or global external proxy Network LB (SSL/TCP Proxy) in Premium Tier (Standard Tier = regional only). Not Internal LBs (regional / cross-region internal IPs).
14. **"Block DDoS / web attacks before they reach backends"** → Cloud Armor policy on the external LB (WAF preset rules, rate limiting, geo, L3/L4 DDoS). Not VPC firewall rules alone (they act on the VM after traffic reaches the VPC).
15. **"Two VPCs/on-prem with overlapping CIDRs need to talk"** → Private NAT (with NCC for spokes/hybrid) or Private Service Connect (producer/consumer, overlap allowed). VPC peering and NCC VPC spokes require non-overlapping ranges.

## Quick-reference cheat table

### VPC Connectivity comparison

| Method | Scope | Bandwidth | Latency | SLA | Setup time | Egress cost | When to use |
|--------|-------|-----------|---------|-----|-----------|-----------|-----------|
| VPC Peering | 2 VPCs (same/diff org), non-transitive, no overlapping ranges | No peering-specific cap (VM/network limits) | Same as intra-VPC | No separate SLA | Minutes | Normal egress rules (peering itself free) | Federation, multi-team. |
| Shared VPC | Host + Service projects (same org) | No cap beyond VM/network limits | Same as intra-VPC | No separate SLA | Hours | Normal egress rules | Centralized network/security control, large org. |
| Private Service Connect | Consumer endpoint ↔ producer service attachment (overlap OK, one-way) | Per-endpoint limits apply | Low | No separate SLA | Minutes | Data processing charge on endpoints | Private access to Google APIs, managed/partner/own services. |
| Private Services Access | VPC ↔ Google-managed service VPC (peering) | Service-dependent | Low | Service-dependent | Minutes | Normal egress rules | Cloud SQL, Memorystore (peering-based; PSC preferred where available). |
| Network Connectivity Center | Hub + VPC/hybrid/router-appliance spokes | Spoke-dependent | Varies | Varies | Hours | Inter-spoke data transfer charges apply | Multi-VPC + on-prem transitive connectivity. |
| Cloud VPN | VPC ↔ On-prem/other cloud (internet, IPsec) | ~3 Gbps per tunnel | Internet-dependent | 99.99 % (HA VPN topology) | Hours–days | Per GB egress | Hybrid, branch office, backup path. |
| Dedicated Interconnect | VPC ↔ On-prem (colo cross-connect) | 10/100/400 Gbps circuits | Low, predictable | 99.9 % / 99.99 % (topology) | Weeks | Lower per-GB egress than internet | Enterprise hybrid, high throughput. |
| Partner Interconnect | VPC ↔ On-prem (via service provider) | 50 Mbps–50 Gbps per attachment | Low, provider-dependent | 99.9 % / 99.99 % (topology) | Days–weeks | Lower per-GB egress than internet | Hybrid without colo presence. |
| Cross-Cloud Interconnect | GCP ↔ AWS/Azure/OCI/Alibaba | 10 or 100 Gbps | Low, predictable | 99.9 % / 99.99 % (topology) | Weeks | Lower per-GB egress than internet ⚠ verify | Multi-cloud private link. |

### Load Balancer comparison (focused)

| Type | Scope | Protocol | Proxy | External | Multi-region | Best for |
|------|-------|----------|-------|----------|--------|---------|
| External Application LB (HTTP(S)) | Global (or regional) | HTTP/HTTPS | Yes | Yes | Yes (global, Premium Tier) | Web, REST API, Host/Path routing, CDN, Cloud Armor. |
| External proxy Network LB (TCP/SSL Proxy) | Global (or regional) | TCP (optional TLS offload) | Yes | Yes | Yes (global, Premium Tier) | Non-HTTP TCP/TLS needing global HA (SMTP, MQTT). |
| External passthrough Network LB | Regional | TCP/UDP/ESP/GRE/ICMP | No | Yes | No | UDP, preserve client IP, high throughput, non-HTTP. |
| Internal Application LB | Regional (or cross-region) | HTTP/HTTPS | Yes | No (VPC) | Cross-region mode | VPC-internal microservices. |
| Internal proxy Network LB | Regional (or cross-region) | TCP | Yes | No (VPC) | Cross-region mode | Internal TCP services, hybrid backends. |
| Internal passthrough Network LB | Regional | TCP/UDP/other IP | No | No (VPC) | No | VPC-internal L4; next hop for appliances. |

### Firewall policy hierarchy

| Layer | Scope | Enforcement | When to use |
|-------|-------|-------------|-----------|
| Hierarchical Firewall Policy | Organization/Folder | Evaluated first; allow / deny / goto_next | Org-wide rules: block external DB, mandatory baseline. |
| Network Firewall Policy (Cloud NGFW) | VPC network (global or regional) | Successor to VPC rules; secure tags, FQDN/geo/threat-intel; NGFW Enterprise adds IPS | Network-level rules with IAM-governed tags; inline threat prevention. |
| VPC Firewall Rules | VPC/Project | Allow/Deny per rule (classic) | App-level: service-account/network-tag-based rules. |
| Network Policies (K8s) | GKE cluster | Pod-level, Kubernetes API | Pod isolation, east-west control. |
| Cloud Armor | HTTP(S) LB | WAF + DDoS | HTTP/HTTPS threats, geo-blocking, rate limiting. |

## Practice scenarios

### Scenario 1: Multi-region e-commerce with 99.99% SLA

**Context:** Cymbal Retail has three regional GKE clusters (us-central1, europe-west1, asia-southeast1) serving the same microservices. Each cluster has a Cloud SQL instance. Users worldwide. CEO: "Lowest latency and ≥99.99 % uptime."

**Question:** How do you route external HTTPS traffic globally with automatic failover?

A) Regional Network LB per region, users manually choose region  
B) Global HTTP(S) LB with multi-region backends, anycast IP  
C) 3 separate Cloud VPNs bridging clusters  
D) Global external proxy Network LB (SSL/TCP Proxy)  

**Answer:** B

**Why:** Global HTTP(S) LB is the standard for multi-region HTTP services. Anycast IP guarantees traffic routes to the nearest healthy backend (HA & latency). B is correct. A (manual failover) doesn't meet 99.99 % SLA. C (VPN) is for hybrid, not multi-region. D (proxy Network LB) is for non-HTTP TCP/TLS; it lacks the HTTP-aware routing, CDN and WAF integration of the Application LB.

---

### Scenario 2: Secure on-prem integration for EHR

**Context:** EHR Healthcare must sync patient data from legacy on-prem insurer system (data center in Iowa) to Cloud SQL in GCP. Volume: 50 GB/day. Latency requirement: <100 ms round-trip. Regulatory: HIPAA audit log every transaction.

**Question:** Which hybrid connectivity minimizes cost while meeting latency SLA?

A) HA VPN (two tunnels or more, BGP via Cloud Router)  
B) Dedicated Interconnect (10 Gbps)  
C) Partner Interconnect via local telecom  
D) Cross-Cloud Interconnect to AWS (for redundancy)  

**Answer:** A

**Why:** 50 GB/day ≈ 5 Mbps sustained (far under the ~3 Gbps per-tunnel limit). VPN over the internet has variable latency but a regional path (same continent) normally fits a 100 ms budget, and IPsec gives encryption in transit for HIPAA. HA VPN has the 99.99 % SLA and the lowest cost and lead time. B (Interconnect) is overkill for 50 GB/day. C (Partner) is costlier. D (Cross-Cloud) is the wrong product (connects to another cloud, not on-prem). A is correct. (If the latency SLA were strict or volume grew to many Gbps, Interconnect would be the answer.)

---

### Scenario 3: Pod isolation in GKE multi-tenant cluster

**Context:** A startup runs GKE cluster with 5 dev teams, each with ~10 pods. Teams must NOT communicate with each other (network isolation). Currently, all pods can reach all pods.

**Question:** What is the most Kubernetes-native way to enforce pod isolation?

A) VPC Firewall Rules blocking inter-pod CIDR ranges  
B) Kubernetes Network Policies with Calico/Dataplane V2  
C) GKE Shared VPC with separate service projects per team  
D) Cloud Armor policies on each pod's ingress service  

**Answer:** B

**Why:** Kubernetes Network Policies are the native K8s API for pod-to-pod ACLs. Requires VPC-native GKE + CNI (Calico or GKE Dataplane V2). B is correct. A (VPC Firewall) can't target individual pods (sees only node IPs). C (Shared VPC) is for project isolation, not pod isolation. D (Cloud Armor) is for HTTP(S) LB, not pod traffic.

---

### Scenario 4: Hybrid transitive routing (on-prem hub + 3 VPCs)

**Context:** KnightMotives has on-prem data center (10.0.0.0/8) and three GCP VPCs (10.100.x.x, 10.101.x.x, 10.102.x.x). VPCs must all reach on-prem AND reach each other transitively. VPC peering is non-transitive, so full-mesh peering plus per-VPC VPN/BGP to on-prem would be unmanageable at scale.

**Question:** Which GCP service best simplifies this topology?

A) VPC Peering (all-to-all)  
B) Network Connectivity Center  
C) Shared VPC  
D) Cloud VPN (all VPCs to on-prem)  

**Answer:** B

**Why:** Network Connectivity Center gives one hub (a Google Cloud resource) with the three VPCs as VPC spokes and the on-prem site attached as a hybrid spoke (VPN/Interconnect with Cloud Router BGP), providing transitive connectivity between spokes. B is correct. A (VPC Peering) is non-transitive, so VPC1↔VPC3 via VPC2 or on-prem does not work. C (Shared VPC) shares one VPC across projects; it does not interconnect separate VPCs. D (a VPN per VPC) gives each VPC a path to on-prem but no VPC-to-VPC transit without hairpinning through on-prem.

---

### Scenario 5: Serverless egress with static IP and on-prem access

**Context:** Altostrat Media runs many Cloud Run services that call external SaaS APIs (Slack, Jira, etc.) and some legacy on-prem systems reachable over an existing HA VPN. Current setup: no VPC, public egress with changing source IPs. The SaaS vendors now require a fixed IP allowlist, and the team wants to avoid paying for connector instances.

**Question:** Best approach?

A) Attach all Cloud Run services to Serverless VPC Access connectors → Cloud NAT  
B) Direct VPC Egress into a VPC subnet + Cloud NAT with reserved static IPs (on-prem reached via the existing VPN routes)  
C) Switch to private APIs via Private Service Connect  
D) Use API Gateway to cache external API responses  

**Answer:** B

**Why:** Direct VPC egress places Cloud Run instances in your subnet (no connector VMs, higher throughput, lower latency than connectors), gives access to VPN/Interconnect routes for on-prem, and Cloud NAT with reserved external IPs provides the stable source IPs vendors can allowlist. A also works functionally but connectors are the older option with per-instance cost. C (PSC) targets Google APIs/published services, not arbitrary internet SaaS. D (caching) does not provide a fixed source IP.

---

### Scenario 6: Securing sensitive data at Global HTTP(S) LB

**Context:** Cymbal Retail's product catalog LB receives 10M requests/day. Attackers are scanning for SQLi, attempting geo-spoofing (claiming to be EU users to bypass price restrictions), and one customer's API key was leaked.

**Question:** Which control should be applied at the LB layer?

A) Cloud Armor security policy (preconfigured SQLi WAF rules, geo-based rules, rate limiting)  
B) VPC Firewall Rules (block source IPs)  
C) Private Service Connect to the backend  
D) Cloud DNS query logging  

**Answer:** A

**Why:** Cloud Armor attaches to the external Application LB and can apply preconfigured SQLi/XSS rules, geo-based allow/deny and rate limiting (e.g., per header/IP). B (VPC firewall) acts on the VM after the LB proxy and cannot inspect HTTP payloads. C (PSC) is a connectivity mechanism, not a filtering control. D (DNS logging) is irrelevant. Note: IP-geolocation rules are not proof of user location (VPNs), and a leaked API key should also be revoked/rotated, not only rate-limited.


## Sources

- Google Cloud VPC documentation: https://cloud.google.com/docs/vpc
- Google Cloud Load Balancing overview: https://cloud.google.com/load-balancing/docs/load-balancing-overview
- Private Service Connect: https://cloud.google.com/vpc/docs/private-service-connect
- Cloud Interconnect pricing & SLAs: https://cloud.google.com/interconnect/docs
- Cloud DNS: https://cloud.google.com/dns/docs
- GKE networking (VPC-native, Network Policies): https://cloud.google.com/kubernetes-engine/docs/concepts/network-overview
- Cloud NAT: https://cloud.google.com/nat/docs
- VPC Flow Logs: https://cloud.google.com/vpc/docs/flow-logs
- Cloud Armor: https://cloud.google.com/armor/docs
- Cloud IDS: https://cloud.google.com/ids/docs
- Network Connectivity Center: https://cloud.google.com/network-connectivity-center/docs
- Shared VPC: https://cloud.google.com/vpc/docs/shared-vpc
- Hierarchical Firewall Policies: https://cloud.google.com/vpc/docs/firewalls#hierarchical_firewall_policies
- Interconnect FAQ (SLA, capacity): https://docs.cloud.google.com/network-connectivity/docs/interconnect/support/faq
- Partner Interconnect overview: https://cloud.google.com/network-connectivity/docs/interconnect/concepts/partner-overview
- Cross-Cloud Interconnect overview: https://cloud.google.com/network-connectivity/docs/interconnect/concepts/cci-overview
- Choose a Network Connectivity product: https://docs.cloud.google.com/network-connectivity/docs/how-to/choose-product
- Choose a load balancer: https://cloud.google.com/load-balancing/docs/choosing-load-balancer
- Private NAT for NCC: https://docs.cloud.google.com/nat/docs/about-private-nat-for-ncc
- Official PCA Exam Guide v6.1: https://cloud.google.com/certification/guides/cloud-architect (section 1.3, 2.1)
- Community notes: gcpstudyhub.com, Reddit r/googlecloud (PCA 2026 renewal threads)

