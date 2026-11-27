# Chapter 05 — Storage and Databases

## Exam objectives covered

- **1.3 Network, storage, compute:** storage types (object, file, DB); data processing solutions; mapping storage to use cases.
- **2.2 Storage systems:** allocation; data processing & compute provisioning; security & access; data transfer & latency; retention & lifecycle; growth planning; protection (backup/recovery).

---

## What the exam really tests

- **Service choice under constraints:** given load patterns (sequential reads, hot keys, global consistency, compliance), pick the right database or storage class—and explain why the others fail.
- **Lifecycle and cost:** Cloud Storage classes (Standard → Nearline → Coldline → Archive), minimum durations, Autoclass, retention policies, and soft-delete interact with billing and disaster recovery planning.
- **High availability & backup strategy:** which services support HA natively (Spanner, AlloyDB Regional, Cloud SQL HA), which require replicas or cross-region setup, and how PITR/backup windows affect RTO/RPO.
- **Data consistency guarantees:** eventual vs strong, global vs regional, and how they affect architecture (e.g., Bigtable row-key design to avoid hotspotting; Spanner for global ACID).
- **Migration source mapping:** MySQL/MariaDB → Cloud SQL or AlloyDB; SQL Server → Cloud SQL or Spanner; MongoDB → Firestore or Cloud SQL (JSON); Redis → Memorystore; on-prem file servers → Filestore.
- **Storage security & compliance:** CMEK (Cloud KMS), uniform bucket-level access, IAM vs object ACLs, Bucket Lock for immutable compliance archives.

---

## Core concepts

### Cloud Storage (object storage)

**Scope & model:**
- Petabyte-scale object store; regional, dual-region (a specific pair of regions on the same continent), or multi-region (a large geographic area such as US, EU or ASIA) placement.
- Strongly consistent (no eventual-consistency complexity).
- Native JSON/gRPC APIs and S3-compatible XML endpoint.
- Per-object versioning, soft delete, and retention policies.

**Storage classes & minimum durations:**

| Class | Min. Duration | Use Case | Cost |
|-------|-------------|----------|------|
| **Standard** | None | Frequent access, hot data (analytics ingest, serving) | Highest storage cost, no retrieval fee |
| **Nearline** | 30 days | Infrequent (backup snapshots, logs archival after 30d) | Lower storage, moderate retrieval fee |
| **Coldline** | 90 days | Rare access (cold archives, compliance archives) | Lower storage, high retrieval fee |
| **Archive** | 365 days | Long-term compliance/legal holds, rarely accessed | Lowest storage, highest retrieval fee (still millisecond access, no restore step) |

**Autoclass:** automatically moves each object to colder classes as it goes unaccessed (and back to Standard when it is read again), based on access patterns; no manual lifecycle rules, and no early-deletion or retrieval fees for the transitions. The terminal class (Nearline by default, Archive optionally) is configurable.

**Lifecycle rules:** transition or delete objects by age, class, or custom metadata tags (e.g., "archive after 90 days").

**Location types:**
- **Region** (single Google region, e.g., us-central1): lowest latency within region, no cross-region replication cost, but single point of failure.
- **Dual-region** (a pair of regions on the same continent, e.g., `nam4` = Iowa + South Carolina): geo-redundant, default replication is asynchronous (most objects replicated within about 1 hour), optional **turbo replication** targets 100 % of objects within 15 minutes; 99.95 % SLA for Standard; higher cost.
- **Multi-region** (a large area such as `US`, `EU`, `ASIA`): geo-redundant across multiple regions in that area, 99.95 % SLA for Standard, higher storage price than regional; best for serving content to users spread across the area.

**Versioning & soft delete:** enable object versioning to keep historical copies; soft delete (on by default) allows recovery of deleted objects within a retention window (default 7 days, configurable 7–90 days, or 0 to disable) before permanent deletion.

**Retention policy & Bucket Lock:** set minimum retention (e.g., 7 years for compliance); Bucket Lock makes retention immutable (no override, critical for healthcare/legal).

**Signed URLs:** time-limited read/write access for unauthenticated users (e.g., browser download links); no IAM required.

**Uniform bucket-level access (UBLA):** disables per-object ACLs; all access via IAM roles. Simplifies compliance (no object-level exceptions).

**CMEK (Customer-Managed Encryption Keys):** integrate with Cloud KMS for regulatory/data-residency requirements (→ Ch11).

**Transfer options:** Storage Transfer Service (scheduled/online bulk copy from AWS S3/Azure/HTTP/on-prem via agents), Transfer Appliance (petabyte-scale physical device), `gcloud storage` (small-scale; replaces gsutil).

---

### Block storage (Persistent Disk, Hyperdisk, Local SSD)

**Persistent Disk (PD):**
- Block storage attached to Compute Engine VMs; survives VM reboot/stop.
- **pd-standard** (HDD): cheap, sequential/throughput-oriented, lowest IOPS per GB.
- **pd-balanced** (SSD): cost-effective, common default choice.
- **pd-ssd** (SSD): higher IOPS per GB than pd-balanced.
- **pd-extreme** (older, high-IOPS tier; Hyperdisk Extreme is the modern replacement).
- Snapshots: point-in-time image; can restore to new disk or VM in another zone/region.
- HA setup: regional persistent disk (synchronously replicated across two zones in a region; you fail over by force-attaching to a VM in the other zone, e.g. via MIG/regional cluster automation).

**Hyperdisk (Google's newer block storage; IOPS, throughput and capacity are provisioned independently):**
- Option for lowest-latency, highest-throughput workloads (databases, SAP, Oracle); available only on newer machine series.
- Types: Hyperdisk Balanced, Hyperdisk Balanced High Availability (replicated across two zones), Hyperdisk Extreme, Hyperdisk Throughput, Hyperdisk ML (read-only multi-attach for model loading); Storage Pools let you share capacity/performance across disks.

**Local SSD:**
- Physically attached NVMe SSD on the host machine; extremely fast (sub-millisecond latency).
- **Ephemeral:** data survives reboot and live migration but is lost when the VM stops/is deleted or the host fails; no snapshots.
- Use: temporary cache, scratch space, very high-frequency I/O (e.g., real-time analytics).

---

### File storage (Filestore, NetApp Volumes)

**Filestore:**
- Managed NFS (NFSv3/NFSv4.1) for shared file access.
- **Basic HDD / Basic SSD** (legacy tiers, kept for backward compatibility): zonal NFSv3, fixed performance, up to 63.9 TiB.
- **Zonal**: single-zone NFS (NFSv3/v4.1, 1-100 TiB), 99.9 % SLA, configurable performance, supports snapshots, backups and replication.
- **Regional** (HA): synchronous replication across zones in a region, 99.99 % SLA, survives a zone outage, production-grade.
- **Enterprise** (older name for the regional HA tier, incl. **Multishares for GKE**): data replicated synchronously across three zones, 99.99 % SLA; Multishares carve many small PVs (1-10 TiB instances) for GKE.
- Use: shared filesystem for GKE, Compute Engine, Data Science workloads (TensorFlow, ML training, batch processing).

**NetApp Volumes (managed NetApp Cloud Volumes):**
- Advanced NFS/CIFS/SMB for customers needing NetApp-specific features (Snapshot, SnapMirror, FlexClone, deduplication).
- Higher cost, but advanced data management.

---

### Cloud SQL (managed relational DB)

**Engine:** MySQL, PostgreSQL, or SQL Server.

**HA setup:**
**Editions:** **Enterprise** (99.95 % SLA, maintenance downtime typically under 60 s, PITR up to 7 days) and **Enterprise Plus** (99.99 % SLA including maintenance, near-zero-downtime maintenance under 1 s, higher-performance machines + data cache, PITR up to 35 days).
**HA setup:**
- **Primary + standby in another zone** (synchronous regional persistent disk replication): automatic failover, typically about a minute; regional HA = 99.95 % (Enterprise) / 99.99 % (Enterprise Plus) SLA.
- Regional placement (standby in a different zone of the same region).

**Read replicas:** asynchronous, cross-region supported; no automatic failover, but can manually promote (also used for cross-region DR).

**PITR (Point-in-Time Recovery):** transaction-log based; restore to any point within the log retention window (1-7 days on Enterprise, up to 35 days on Enterprise Plus).

**Limits & when NOT to use:**
- Single-region primary (cross-region read replicas possible, but not automatically failover).
- Max storage 64 TB per instance (growth planning important).
- Vertical scaling only for writes (one primary); no horizontal write scale-out.
- If needing global consistency or horizontal write scale → Spanner.
- If needing higher performance, HTAP or larger scale on PostgreSQL → AlloyDB for PostgreSQL.

**Backups:** automated daily backups (default keeps 7, configurable up to 365 retained backups); on-demand backups; backups can also be restored to a new instance.

---

### AlloyDB (PostgreSQL-compatible, high-performance managed database)

**What it adds over Cloud SQL (PostgreSQL):**
- **Columnar engine:** analytical queries up to 100× faster than standard PostgreSQL (Google's claim; in-memory column store, keeps row format for OLTP).
- **Disaggregated, auto-scaling storage** and up to 20 read replicas in an **autoscaling read pool**; primary is scaled up/down (not serverless, no scale-to-zero).
- **Higher availability:** 99.99 % SLA including maintenance, automatic failover (typically within about 60 s), cross-region secondary clusters for DR.

**AlloyDB Omni:** downloadable edition for on-premises, other clouds or laptops; same engine, no lock-in.

**When to choose:**
- PostgreSQL workload with analytical + transactional mix (HTAP = Hybrid Transactional & Analytical Processing).
- Need higher performance/availability than Cloud SQL for PostgreSQL without global distribution (single-region primary; cross-region replicas for DR).
- Also supports pgvector / vector search for gen-AI apps.

**vs Cloud SQL:** AlloyDB is newer, faster, usually more expensive and PostgreSQL-only; Cloud SQL is mature, simpler, supports MySQL/SQL Server too and fits traditional OLTP.

---

### Cloud Spanner (globally distributed, strong consistency)

**Unique traits:**
- **Globally consistent ACID transactions** across continents, no race conditions.
- **Regional config:** replicas in three zones of one region, 99.99 % SLA.
- **Dual-region config** (limited countries, e.g. Australia, Germany, India, Japan): two regions in one country, 99.999 % SLA, helps data-residency needs.
- **Multi-region config** (e.g. `nam3`, `eur3`, `nam-eur-asia1`): replicas in several regions plus a witness, 99.999 % SLA; reads can be served locally, writes need cross-region quorum (higher write latency).
- Schema: relational (tables, foreign keys, SQL), but designed for horizontal distribution; also graph/vector/full-text features.
- **PostgreSQL interface:** PostgreSQL-dialect databases for developers who want PostgreSQL syntax/tooling (alternative to GoogleSQL dialect).
- Editions: Standard, Enterprise, Enterprise Plus; capacity from 100 processing units (0.1 node) upward, with managed autoscaler.

**When to choose:**
- Need strong consistency + global distribution (finance, inventory, user identity).
- Willing to accept higher cost for correctness.

**When NOT:**
- Single-region latency-critical (AlloyDB or Cloud SQL cheaper, faster local).
- Eventual consistency acceptable (Firestore, Bigtable).

---

### Bigtable (wide-column NoSQL, high throughput)

**Model:** billions of rows × thousands of columns; sorted by a single row key (the only index), columns grouped into column families.

**Row-key design (critical):**
- **Avoid hotspotting:** don't use sequential IDs (timestamp) as first component (all writes go to one partition); use hash prefix, reverse timestamp, or bucketing.
- Example: `hash(user_id)#timestamp` instead of just `timestamp#user_id`.

**Replication:**
- Single-cluster (one zone): strongly consistent, cheap, 99.9 % SLA, no automatic cross-zone failover.
- Multi-cluster (≥2 clusters in different zones or regions): asynchronous replication (eventually consistent between clusters), automatic failover with multi-cluster routing; SLA 99.99 % (all clusters in one region) or 99.999 % (clusters in 3+ regions). Use single-cluster routing + app profiles if you need read-your-writes.
- Backups (up to 90 days retention) are cluster-level; there is no point-in-time restore.

**When to choose:**
- High-throughput time-series (IoT telemetry, analytics events, stock prices).
- Analytical queries on billions of rows (low-latency scans).
- Not suitable for complex joins or ACID transactions.

---

### Firestore (NoSQL document store)

**Model:** JSON-like documents in collections; real-time updates via listeners; serverless, **strongly consistent** reads and ACID multi-document transactions (regional 99.99 % SLA, multi-region 99.999 % SLA).

**Modes / editions:**
- **Firestore Native mode:** recommended for new mobile/web apps (real-time listeners, offline sync).
- **Datastore mode:** server-side-oriented API for existing Datastore apps (no real-time listeners/mobile SDKs); also strongly consistent on the Firestore backend.
- **Enterprise edition** offers MongoDB compatibility (MongoDB drivers/query API) and is the natural target for MongoDB workloads.

**When to choose:**
- Mobile/web apps (real-time sync, offline support).
- User profiles, chat messages, game state (document-centric).
- Low-latency, read-heavy with modest write scale.

**Limits:**
- Not designed for giant analytical datasets (BigQuery instead).
- Not relational (no joins, limited aggregation); write throughput per document/collection is limited vs Spanner/Bigtable; no global relational SQL.

---

### Memorystore (in-memory cache)

**Engines:** Memorystore for **Valkey** and Memorystore for **Redis Cluster** (sharded, up to 250 nodes, 99.99 % SLA when multi-zone with HA, replicas spread across zones), Memorystore for **Redis** (non-cluster; Basic = no HA, Standard = replica + automatic failover, 99.9 % SLA), Memorystore for **Memcached**. Valkey is the open-source Redis fork and the recommended engine for new workloads ⚠ verify.
- Sub-millisecond latency; session cache, real-time leaderboards, rate limiting.
- Replication is asynchronous, so a failover can lose the last few writes.

**Memcached:**
- Simpler than Redis, pure cache-aside pattern, multi-node (zonal nodes, no replication/persistence), 99.9 % SLA.
- Still offered (not deprecated); choose Redis/Valkey when you need persistence, replication or rich data structures.

**Use:** reduce latency to databases, cache hot data.

---

### BigQuery (data warehouse, also a storage layer)

**As storage:** Petabyte-scale columnar storage optimized for analytics (→ Ch06 for processing/SQL).

**BI Engine:** in-memory accelerator for sub-second BI queries (no moving data to separate cache).

**Partitioning/Clustering:** organize data by date/value to reduce scans (cheaper, faster).

---

## Backup, PITR & disaster recovery strategies

### Per-service backup & recovery windows

**Cloud SQL:**
- Automated daily backups (default: keep 7; retain up to 365 backups) plus on-demand backups.
- PITR: restore to any second within the transaction-log retention window (1–7 days Enterprise, up to 35 days Enterprise Plus); restore always creates a new instance (clone).
- Backup window: configure for off-peak hours.
- Cross-region DR: cross-region read replica (promote manually) or backups copied/restored in another region.
- Use case: daily automated + PITR for user errors, data corruption.

**AlloyDB:**
- Automated and on-demand backups plus continuous backup (WAL-based) for PITR; continuous backup retention is configurable (default 14 days, up to 35 days) ⚠ verify.
- Cross-region DR via secondary clusters (continuous replication, manual promotion/failover).
- Use case: PostgreSQL workloads needing point-in-time recovery.

**Cloud Spanner:**
- Backups: on-demand and scheduled, retained up to 1 year; copyable across regions.
- PITR on all instances (regional and multi-region): version retention period from 1 hour (default) up to 7 days.
- Regional/multi-region replication is synchronous across zones/regions (RPO ≈ 0 inside the configuration); use a multi-region or dual-region config for regional-outage protection.
- Use case: critical financial data, global consistency.

**Bigtable:**
- Backups: managed table backups (retention up to 90 days), copyable across regions/projects; restore to a new table.
- Multi-cluster replication (async): automatic failover with multi-cluster routing if a cluster fails.
- No PITR; use backups plus application-level change capture if finer recovery is needed.
- Use case: time-series data, high-throughput serving.

**Firestore:**
- Scheduled (daily/weekly) backups with retention up to 14 weeks, restored to a new database.
- PITR: optional, retains 7 days of versions (clone/restore from a point in the past); both Native and Datastore mode support backups and PITR.
- Use case: user data, real-time app state.

**Memorystore (Redis/Valkey):**
- Persistence options depend on engine/tier: RDB snapshots (scheduled/on-demand export) and, for Redis Cluster/Valkey, AOF; Valkey and Redis Cluster also support managed backups ⚠ verify.
- HA: automatic failover to replica (async replication, so last few writes may be lost).
- Use case: session cache, can afford data loss; treat as a cache, not a system of record.

**Cloud Storage:**
- Versioning: keep historical object versions; soft delete recovers deleted objects (7-day default retention, configurable 7–90 days).
- No "backup" concept; treat as source of truth.
- Disaster recovery: Dual-region or multi-region placement.
- Use case: immutable archives (Bucket Lock + retention policy).

### HA & failover patterns

| Service | Single Region HA | Multi-Region HA | Failover Time | RPO |
|---------|------------------|-----------------|---------------|-----|
| **Cloud SQL** | HA standby in 2nd zone (sync disk replication) | Cross-region read replica async (manual promote) | Auto, typically ~1 min | ~0 (HA), seconds-minutes (replica lag) |
| **AlloyDB** | Primary + standby in 2nd zone | Secondary cluster in another region (manual promote) | Auto, typically <60 s | ~0 (HA), seconds (cross-region) |
| **Spanner** | Regional: 3 zones, 99.99 % SLA | Dual-/multi-region 99.999 % SLA | Automatic, transparent | ~0 (synchronous replication) |
| **Bigtable** | Single-cluster: no auto failover (99.9 %) | Multi-cluster async, multi-cluster routing | Automatic with multi-cluster routing | Seconds (async lag) |
| **Firestore** | Regional: 3+ zones sync, 99.99 % | Multi-region 99.999 % | Automatic | ~0 (synchronous) |
| **Memorystore** | Redis Basic: no HA; Standard / Cluster / Valkey: replicas + auto failover | n/a (cross-region replication is limited) ⚠ verify | Automatic (seconds) | Seconds (async) |
| **Cloud Storage** | None (single object) | Dual-/multi-region (async; turbo replication option for dual-region) | N/A (served from remaining region) | Near zero for most objects; turbo = 15 min target |

---

## Data growth planning & limits

**Cloud SQL:**
- Max storage 64 TB per instance; approaching → migrate to Spanner/AlloyDB or shard.
- Storage can auto-increase (enable it) but cannot be decreased.
- Monitor growth rate and plan ahead for the 64 TB ceiling.

**AlloyDB:**
- Storage scales automatically (no pre-provisioning); check the current per-cluster storage limit in the docs ⚠ verify.
- Cost scales with stored data + compute.

**Spanner:**
- Petabyte-scale; storage per node is limited (up to 10 TB per node/1000 PU on current configs), so add compute capacity as data grows; managed autoscaler available.

**Bigtable:**
- Designed for billions of rows; no hard limit.
- Cost scales with cluster nodes.

**Firestore:**
- Max document size 1 MiB; keep documents small.
- Avoid hot documents/sequential indexed fields (write-rate limits per document); spread by tenant/time.

**Cloud Storage:**
- Unlimited; Lifecycle + Autoclass manage cost.

**BigQuery:**
- Effectively unlimited; storage is billed (active vs long-term logical/physical) and queries are billed per byte scanned (on-demand) or by slot capacity (editions).

---

## Data security & encryption

**At rest:**
- **Google-managed keys:** default, automatic rotation, zero overhead.
- **CMEK (Cloud KMS):** customer controls rotation & access (compliance requirement).
- Available for the major services: Cloud SQL, Cloud Storage, Persistent Disk/Hyperdisk, BigQuery, Firestore, AlloyDB, Spanner, Bigtable, and Memorystore (Valkey/Redis Cluster) ⚠ verify per service.

**In transit:**
- Encrypted by default between Google services; use TLS for client connections (e.g. require SSL/TLS on Cloud SQL).
- VPC Service Controls (→ Ch11): restrict egress.

**Access control:**
- **Cloud Storage:** UBLA (IAM-only, no ACLs) preferred for compliance.
- **Cloud SQL/Spanner:** IAM roles + database-native auth (passwords, certs).
- **BigQuery:** IAM + column/row-level security (mask PII per role).
- **Bigtable:** IAM roles (project/instance/table level; authorized views for finer scope); no per-row ACLs.
- **Firestore:** IAM + security rules (document-level).

---

## Storage for AI/ML workloads

**Data lakes & preprocessing:**
1. Raw ingest: Cloud Storage (Standard → Nearline → Archive lifecycle).
2. Clean/transform: Dataflow / Dataproc → BigQuery or Memorystore.
3. Feature engineering: BigQuery ML or Feature Store.

**Cloud Storage FUSE mount:**
- Mount GCS bucket as filesystem; ML scripts read `/gcs/bucket/data/*.parquet` directly.
- No local copy needed; good for sequential reads of large training files (small-file random access is slower than local disk).

**Managed Lustre / Parallel file systems:**
- High-throughput, low-latency parallel file system for distributed training and HPC clusters.

**Online feature serving (<5 ms latency):**
- Memorystore (Redis/Valkey): sub-millisecond, in-memory, cached features.
- Bigtable: single-digit ms at scale (also the backing store of Agent Platform Feature Store online serving).
- Cloud SQL / AlloyDB: low-ms, strong consistency.
- Firestore: tens of ms, user-specific features.

**Typical flow:**
1. Batch compute features in BigQuery (hourly/daily).
2. Export to Memorystore (Redis hash).
3. Agent Platform Endpoint custom routine queries Memorystore.
4. Return features + model prediction in <50 ms.

---

## Decision guides

### When to pick each service

**Cloud Storage:**
- Unstructured objects (images, videos, backups, logs, datasets).
- Compliance → Archive class + Bucket Lock + retention policy.

**Persistent Disk / Hyperdisk:**
- VM filesystem, database engine storage.
- Hyperdisk: high-IOPS/high-throughput workloads with independently provisioned performance (newer machine series only).

**Local SSD:**
- Temporary scratch, real-time analytics.
- Tradeoff: speed vs non-persistence.

**Filestore (NFS):**
- Shared filesystem for GKE/Compute Engine (ML, batch).
- NetApp Volumes: advanced data management (Snapshots, SnapMirror).

**Cloud SQL:**
- Traditional OLTP (user accounts, transactions, standard schema).
- Easy migration from MySQL/MariaDB/PostgreSQL/SQL Server.
- HA needed: enable Cloud SQL HA replica.

**AlloyDB:**
- PostgreSQL + heavy analytics (HTAP), higher performance/availability than Cloud SQL.
- Read-heavy workloads (autoscaling read pools).
- Gen-AI apps needing vector search inside PostgreSQL.

**Spanner:**
- Global (or very high-scale) consistency with horizontal write scaling; 99.999 % availability needs.
- Financial transactions, global inventory.
- Premium pricing justified.

**Bigtable:**
- Time-series billions of events, IoT telemetry.
- High-throughput analytics.
- Avoid: complex joins, ACID across rows.

**Firestore:**
- Real-time mobile/web apps, document-centric.
- User profiles, chat, game state.
- Avoid: giant analytical datasets.

**Memorystore (Redis):**
- Cache layer in front of databases.
- Session store, real-time counters/leaderboards.

**BigQuery:**
- Analytics, BI reporting, structured data.

---

## Database & storage decision tree

```
Does data fit in a single region + no ACID needed?
  YES → NoSQL (Firestore / Bigtable)
        Time-series billions of rows? → Bigtable (row-key design critical)
        Document-centric, real-time? → Firestore
  NO (need regional HA or global consistency):
    Need strong consistency globally? → Spanner
    Need OLTP (transactions) + analytics mix? → AlloyDB
    Just OLTP, moderate scale, standard schema? → Cloud SQL
    Extreme throughput (millions/sec)? → Bigtable
    
Fast cache layer? → Memorystore (Redis/Valkey)

Shared filesystem? → Filestore (Regional HA)

Analytics data lake? → Cloud Storage (lifecycle to Archive) + BigQuery

Unstructured files? → Cloud Storage
  Compliance/legal hold? → Archive + Bucket Lock
  Frequently accessed? → Standard
  Infrequent? → Nearline / Coldline
```

---

## Well-Architected Framework lens

| Pillar | Storage & DB Decisions |
|--------|------------------------|
| **Reliability** | HA: Cloud SQL HA, AlloyDB Regional, Spanner multi-region, Firestore strong consistency. PITR (Cloud SQL up to 7 days Enterprise / 35 days Enterprise Plus). Automated backups + retention. Regional Filestore. |
| **Security** | CMEK (Cloud KMS), UBLA (IAM), Bucket Lock (compliance). Network isolation (VPC, PSA). Sensitive Data Protection for PII. |
| **Cost Optimization** | Cloud Storage: lifecycle to cheaper classes. Autoclass removes tuning. Right-size DBs (autoscaling helps). Memorystore only for hot paths. |
| **Performance** | Bigtable row-key design (avoid hotspots). Firestore indexing. Cloud SQL / AlloyDB query optimization. BigQuery clustering. Memorystore <1 ms latency. |
| **Operational Excellence** | Automated backups, snapshots, HA failover. Monitoring: metrics (replication lag, connection count, latency). Logging (queries, errors). Runbooks for failover. |
| **Sustainability** | Compress data (gzip). Archive cold data. BigQuery columnar. Memorystore reduces repeated DB queries. |

---

## AI/GenAI angle

- **Data lakes for AI:** Cloud Storage (Standard + lifecycle to Nearline/Coldline) + BigQuery for centralized training datasets; Dataflow/DataProc to clean/transform.
- **Feature stores:** AlloyDB or Cloud SQL (with Memorystore caching) for real-time lookups in Agent Platform ML inference (low latency, strong consistency).
- **Vector search:** Agent Platform Vector Search queries embeddings fast; backs conversational AI, RAG.
- **FUSE mounts:** Cloud Storage FUSE allows ML training scripts to read GCS objects as filesystem (no local copy).
- **Managed Lustre:** parallel file system for distributed ML training clusters (HPC).
- **BigQuery as feature warehouse:** BigQuery + Feature Store exports features to Memorystore/Firestore for real-time serving.

---

## Case-study hooks

| Case Study | Storage & DB Needs |
|------------|-------------------|
| **Altostrat Media** | On-prem ingestion → Cloud Storage (Nearline/Archive for cold media library). BigQuery for analytics. Memorystore (Redis) for real-time recommendation counters. Firestore for user preferences/sessions. Dataflow to process metadata extraction logs. |
| **Cymbal Retail** | MySQL / SQL Server / MongoDB on-prem → AlloyDB (PostgreSQL, analytics-friendly) or Cloud SQL (PostgreSQL). Redis → Memorystore. Cloud Storage for product images. BigQuery for sales analytics. Firestore for approval queue state (human-in-the-loop). |
| **EHR Healthcare** | MySQL + MS SQL Server on-prem → Cloud SQL (HA, PITR for recovery). Compliance (HIPAA): CMEK, Bucket Lock for backups, UBLA (IAM over object ACLs). Regional Filestore if shared filesystem needed. BigQuery for de-identified research analytics. Read replicas for DR. |
| **KnightMotives Automotive** | Mainframe supply-chain DB → Spanner (global consistency for multi-plant coordination). Vehicle telemetry (very high write rate) → Bigtable (time-series, high throughput). Dealer inventory/CRM → Cloud SQL or AlloyDB. GDPR compliance: CMEK, DLP on PII. Data-monetisation lake: Cloud Storage + BigQuery. |

---

## Exam traps & keyword → answer

1. **"Minimal operational overhead" + "HA"** → Cloud SQL HA (automatic failover), NOT read replicas (manual).
2. **"Global consistency" + "relational transactions" + "horizontal scale"** → Spanner, NOT Cloud SQL/AlloyDB (single-region primary) or Bigtable (no multi-row ACID, no SQL joins).
3. **"Compliance archive" + "immutable"** → Cloud Storage Archive + **Bucket Lock**, NOT just lifecycle rule (can be deleted).
4. **"Lowest storage cost" + "rarely accessed (once a year)"** → Archive class (365-day minimum; highest retrieval fee, so wrong for frequently read data). Unknown/changing access pattern → Autoclass.
5. **"Sub-millisecond latency"** (cache) → Memorystore (Redis), NOT Cloud SQL read replicas (~1–10 ms network).
6. **"Hotspotting"** (all writes to one partition) → Bad Bigtable row-key design; fix the row key (field promotion/salting/hash prefix, reverse timestamp as suffix), NOT just adding nodes.
7. **"HTAP"** (transactions + analytics) → AlloyDB columnar, NOT Cloud SQL (OLTP only).
8. **"Billions of rows" + "high throughput"** → Bigtable, NOT Cloud SQL (scale limit).
9. **"Shared filesystem"** (GKE, Compute Engine) → Filestore NFS, NOT Cloud Storage (no POSIX mount).
10. **"Restore to point in time"** → Cloud SQL PITR (log retention up to 7 days Enterprise / 35 days Enterprise Plus), NOT discrete backups.
11. **"Eventual consistency"** acceptable → Bigtable multi-cluster (async replication); note Firestore, Spanner and Cloud Storage are all strongly consistent.
12. **"Data sovereignty"** (EU GDPR) → Cloud Storage region/dual-region EU (e.g., `europe-west1`), Cloud SQL EU region + CMEK, Org Policy location constraint (→ Ch12).

---

## Quick-reference cheat table

| Service | Model | Scale | Consistency | Latency | HA/Multi-Region | Cost Tier | Best For | Migration Source |
|---------|-------|-------|-------------|---------|-----------------|-----------|----------|------------------|
| **Cloud Storage** | Object | PB+, regional/dual/multi-region | Strong | Tens of ms first byte | Dual-/multi-region (99.95 % Standard SLA) | Low (Standard → Archive) | Backups, logs, data lakes, archives | N/A |
| **Persistent Disk** | Block | TB, zonal or regional | N/A | Low ms | Regional PD (2-zone sync replica) | Low–Medium | VM filesystem, DB storage | N/A |
| **Hyperdisk** | Block | TB, provisioned IOPS/throughput | N/A | Sub-ms to low ms | Balanced HA (2-zone) | Medium–High | Oracle, SAP, high-IOPS DB | N/A |
| **Filestore (Regional/Enterprise)** | NFS file | TiB (up to 100 TiB) | Strong (NFS) | Low ms | Regional (99.99 %); Zonal 99.9 % | Medium–High | Shared FS (GKE, ML, batch) | On-prem NFS |
| **Cloud SQL** | RDBMS (MySQL/PG/MSSQL) | 64 TB per instance | Strong (primary) | Low ms | HA 99.95 % (Ent) / 99.99 % (Ent Plus), read replicas | Medium | Traditional OLTP, easy migration | MySQL, MariaDB, PostgreSQL, SQL Server |
| **AlloyDB** | RDBMS (PostgreSQL) | Auto-scaling storage (check per-cluster limit) | Strong (primary) | Low ms | HA 99.99 %, read pools, cross-region secondary | Medium–High | HTAP (OLTP + analytics), read-heavy | PostgreSQL, Oracle (via DMS) |
| **Spanner** | RDBMS (SQL) | PB, horizontal | Strong, global | Single-digit ms reads; cross-region writes higher | Regional 99.99 %; dual/multi-region 99.999 % | High | Global consistency, finance, inventory | Oracle, SQL Server, custom RDBMS |
| **Bigtable** | Wide-column | Billions rows, PB | Strong single-cluster; eventual across clusters | Single-digit ms | 99.9 % single cluster; 99.99–99.999 % multi-cluster routing | Low–Medium | Time-series, IoT, high-throughput analytics | HBase, Cassandra |
| **Firestore** | Document (NoSQL) | Serverless, GB–TB+ | Strong | Low-tens of ms | Regional 99.99 %; multi-region 99.999 % | Low–Medium | Real-time mobile/web, user profiles | MongoDB (Enterprise edition), Datastore |
| **Memorystore (Valkey/Redis/Memcached)** | In-memory cache | GB–TB (cluster) | Strong per shard (async replicas) | Sub-ms | Cluster/Valkey multi-zone 99.99 %; Redis Standard 99.9 % | Low–Medium | Session cache, rate limiting, counters | Redis/Memcached on-prem |
| **BigQuery** | Columnar data warehouse | Exabyte | Strong (snapshot) | Seconds (interactive) | Multi-region datasets | Slot capacity or per-TiB scanned | Analytics, BI, ML feature engineering | Data warehouses, CSV/Parquet files |

---

## Migration patterns: source → GCP storage

| On-Prem / External | Target Service | Why | Key Considerations |
|-------------------|-----------------|-----|-------------------|
| **Oracle RDBMS** | Cloud SQL / AlloyDB (PostgreSQL) via DMS, Spanner, or Bare Metal Solution (keep Oracle) | Heterogeneous modernisation vs lift-and-shift | Schema/PLSQL conversion; Spanner needs schema refactor for distribution. |
| **MySQL / MariaDB** | Cloud SQL (MySQL) | Native replication, minimal lift (AlloyDB is PostgreSQL only) | Validate row counts/checksums after cutover (e.g. Data Validation Tool). |
| **SQL Server** | Cloud SQL (SQL Server) or Spanner | Cloud SQL for compatible SQL; Spanner for global scale | T-SQL → Standard SQL migration effort. |
| **PostgreSQL** | Cloud SQL (PostgreSQL) or AlloyDB | Minimal schema changes (AlloyDB faster) | Prepared statements, sequences preserved. |
| **MongoDB** | Firestore Enterprise edition (MongoDB-compatible), Cloud SQL (JSON), Bigtable (time-series) | Least rewrite with MongoDB compatibility; relational if schema is stable | Check API/feature compatibility; denormalisation for native Firestore API. |
| **Redis** | Memorystore (Valkey / Redis Cluster / Redis) | Drop-in replacement, same commands | Update connection strings; private connectivity (PSA/PSC, VPN/Interconnect from on-prem). |
| **Apache HBase** | Bigtable | Identical row-key design patterns | Java HBase client → Bigtable client library. |
| **File shares (NFS/SMB)** | Filestore (NFS) or NetApp Volumes (NFS + SMB) | Managed NFS, same mount interface; Filestore has no SMB | Test latency/throughput; SMB needs NetApp Volumes. |
| **Data warehouse (Teradata, Snowflake)** | BigQuery | Columnar, petabyte-scale, cost-per-scan | SQL dialect conversion (minor). Cost model change. |
| **Cold archives (tape, S3 Glacier)** | Cloud Storage Archive | 365-day minimum, Bucket Lock compliance | Archive objects stay online (millisecond access, no restore wait, unlike tape/Glacier); retrieval fees apply. |

---

## Real-world trade-offs

### Consistency vs performance vs cost

**Strong consistency everywhere (Spanner, AlloyDB, Cloud SQL):**
- Pro: no race conditions, simpler app logic.
- Con: higher cost, cross-region latency, smaller scale than eventual-consistent.

**Eventual consistency (Bigtable multi-cluster, async read replicas):**
- Pro: massive scale, low latency, cheap.
- Con: stale reads (old balance, recommendation), complex reconciliation.
- Exam angle: "no stale reads / avoid conflicting writes" → choose a strongly consistent store (Spanner, Cloud SQL/AlloyDB primary, Firestore, Bigtable single-cluster routing).

### Single-region vs multi-region

**Single region (Cloud SQL, Filestore Zonal):**
- Cheaper, lowest latency within region.
- Risk: region-wide outage → data unavailable (SLA 99.95 %, not zero).

**Multi-region (Cloud Storage multi-region, Spanner multi-region, BigQuery):**
- Higher cost; users nearer to a replica get faster reads, but cross-region writes pay quorum latency.
- Use: global apps, disaster recovery, compliance (data residency by region).

**Hybrid approach:**
- Primary single-region (low cost, latency).
- Read replicas in other regions (for DR, reporting).
- Manual promotion of the replica if the primary region fails (RTO depends on runbook automation).

---

## Exam-relevant numbers (verified)

- **Cloud SQL HA failover:** automatic, typically about a minute; SLA 99.95 % (Enterprise) / 99.99 % (Enterprise Plus).
- **Spanner SLA:** regional 99.99 %, dual-/multi-region 99.999 %.
- **Cloud Storage Standard/Nearline/Coldline minimum durations:** 0 / 30 / 90 days.
- **Cloud Storage Archive minimum duration:** 365 days.
- **Cloud Storage soft delete:** on by default, 7 days (configurable 7–90 days).
- **Cloud SQL PITR window:** up to 7 days (Enterprise) / 35 days (Enterprise Plus); max storage 64 TB.
- **Bigtable SLA:** 99.9 % single cluster; 99.99 % multi-cluster routing in one region; 99.999 % across 3+ regions.
- **Firestore SLA:** regional 99.99 %, multi-region 99.999 %; max document 1 MiB; PITR 7 days; backups up to 14 weeks.
- **Filestore SLA:** Zonal 99.9 %, Regional/Enterprise 99.99 %.
- **Memorystore:** Valkey / Redis Cluster multi-zone 99.99 %, Redis Standard and Memcached 99.9 %; latency sub-millisecond in-region.
- **Bigtable throughput:** scales linearly with node count (no fixed per-cluster ceiling quoted here).

---

## Practice scenarios

**Q1: EHR Healthcare—Compliance Archive**  

EHR Healthcare must keep patient records for 7 years (HIPAA) in immutable archive, accessible for audit. Records are rarely accessed. Staff accidentally deleted a bucket policy once. Which approach ensures immutability?
- A) Cloud Storage Archive class + **Bucket Lock** + retention policy 7 years.
- B) Cloud Storage Archive class with lifecycle rule deleting after 7 years.
- C) Persistent Disk snapshots stored in Archive, encrypted with Cloud KMS.
- D) BigQuery table with column-level security and 7-year retention.

**Answer:** A  

**Why:** Bucket Lock makes retention policy immutable; cannot be removed. Archive class + retention policy alone leaves room for admin override. Disk snapshots are not a WORM archive and cannot enforce retention locks. BigQuery is analytics, not a compliance archive.

---

**Q2: Cymbal Retail—Real-Time Recommendations**  

Cymbal Retail wants sub-millisecond latency product-recommendation counters (user viewed product X → increment counter). Cloud SQL read replicas are 10–20 ms away. Which service?
- A) BigQuery with real-time SQL inserts.
- B) **Memorystore (Redis)** with pub/sub for event-driven increments.
- C) Bigtable with row-key `hash(user_id)#timestamp`.
- D) Firestore with Cloud Pub/Sub triggers.

**Answer:** B  

**Why:** Memorystore Redis gives sub-millisecond latency for counter increments (INCR command). Bigtable is single-digit ms (not sub-ms) and overkill for simple counters. Firestore adds network/service round-trip latency. BigQuery is analytics, not a sub-ms serving store.

---

**Q3: KnightMotives—Global Vehicle Inventory**  

KnightMotives plants in Tokyo, Frankfurt, São Paulo must instantly see and update global inventory (paint colors, parts). Write consistency across plants critical. Must support 500+ writes/sec. Which database?
- A) Cloud SQL with read replicas in each region (manual failover).
- B) Bigtable multi-cluster with row-key `region#part_id#timestamp`.
- C) AlloyDB (PostgreSQL-compatible, EU region only).
- D) **Spanner multi-region** (global strong consistency, strong ACID, 99.999 % SLA).

**Answer:** D  

**Why:** Spanner gives global strong consistency with horizontal write scaling + 99.999 % SLA. Cloud SQL replicas are async and read-only (stale reads, single write region). Bigtable multi-cluster replication is eventually consistent (inventory mismatch risk) and has no multi-row ACID. AlloyDB has a single-region primary.

---

**Q4: Altostrat Media—Cold Media Archive**  

Altostrat stores 500 TB of video/audio in Cloud Storage. Newly uploaded objects are accessed heavily by editors, but access patterns afterwards are unpredictable: some old objects are suddenly popular again while most go cold. Cost is critical and the team wants no tuning effort. Which strategy?
- A) Store all in Standard class; use signed URLs to serve.
- B) Manually transition to Coldline after 30 days, Archive after 90 days.
- C) **Autoclass**: objects move to colder classes automatically as they go unaccessed and return to Standard when read again.
- D) Store in Archive class immediately; upgrade to Standard on-demand when accessed.

**Answer:** C  

**Why:** Autoclass handles unpredictable access without manual tuning and without retrieval or early-deletion fees on its transitions. Static age-based lifecycle rules (B) would push objects that become popular again into expensive retrieval charges. Archive-first (D) incurs high retrieval fees and 365-day minimum charges. All-Standard (A) is the most expensive for cold data. (If access patterns were known and predictable, age-based lifecycle rules would also be a valid answer.)

---

**Q5: EHR Healthcare—HA for Patient Records**  

EHR patient database must have 99.95 % SLA and survive one-zone failure with <3 min failover. Currently using Cloud SQL MySQL single-zone. Which HA upgrade?
- A) Add Cloud SQL read replica in another zone (manual promotion).
- B) Migrate to Spanner (overkill, higher cost).
- C) Use Bigtable + JDBC connector (not recommended, different model).
- D) **Cloud SQL HA configuration** (primary + standby in a different zone, automatic failover, typically about a minute).

**Answer:** D  

**Why:** Cloud SQL HA gives automatic failover, 99.95 % SLA (Enterprise edition; Enterprise Plus gives 99.99 %), synchronous replication. Read replicas require manual promotion (misses SLA). Spanner is excessive for single-region OLTP. Bigtable is not relational (EHR schema mismatch).

---

**Q6: Altostrat + Cymbal—ML Feature Serving**  

Both case studies want to serve ML features (user embeddings, product vectors) at single-digit-millisecond latency to Agent Platform Endpoints. Features are computed in BigQuery. Which architecture?
- A) Query BigQuery directly from Agent Platform Endpoint (features in model config).
- B) **Export features to AlloyDB or Cloud SQL + Memorystore cache layer** (low-latency online lookups, sub-ms cache hits).
- C) Leave features as Parquet files in Cloud Storage and read them on each prediction request.
- D) Store features in Cloud Storage Archive and load on demand.

**Answer:** B  

**Why:** Online serving needs an operational, low-latency store (Memorystore cache in front of AlloyDB/Cloud SQL; Bigtable or the managed Feature Store online store would also fit). BigQuery is an analytical warehouse with seconds-level query latency, so it is too slow per request (A). Reading object files per request (C) or Archive objects (D) adds far too much latency and is not designed for key lookups.

---

**Q7: KnightMotives—IoT Telematics at Scale**  

Every vehicle sends GPS, engine and battery telemetry (about 1 million vehicles, each sending one reading per second, so roughly 1M writes/sec sustained). Query: "avg speed last hour by region". 7-year compliance retention. Which database + storage?
- A) **Bigtable for ingestion (multi-cluster HA) + BigQuery for analytics** (row-key = `vehicle_id#reversed_timestamp`, Cloud Storage Archive for cold exports).
- B) Cloud SQL with partitions by region (billions of rows scale limit).
- C) Firestore collections per region (scale, eventual consistency).
- D) AlloyDB columnar engine (PostgreSQL, region-based limits).

**Answer:** A  

**Why:** Bigtable scales linearly to very high write throughput; a row key leading with the high-cardinality vehicle ID spreads load (a key starting with region or a timestamp would hotspot). BigQuery columnar speeds analytics (avg by region). 7-year compliance → Archive storage. Cloud SQL / Firestore / AlloyDB scale limits or consistency issues.

---

**Q8: Design Decision – Filestore Regional for EHR**  

EHR wants to run Kubernetes ML workload on GKE that processes patient records stored in a shared filesystem. Which Filestore tier and why?
- A) Filestore Zonal — cheaper, single zone, acceptable for non-critical jobs.
- B) NetApp Volumes — enterprise SnapMirror replication (overkill unless custom policies needed).
- C) **Filestore Regional (HA)** — synchronous cross-zone replication, 99.99 % SLA, survives a zone outage for business continuity.
- D) Persistent Disk regional snapshot — not a shared filesystem, not suitable for concurrent GKE access.

**Answer:** C  

**Why:** EHR compliance demands HA; Regional Filestore tolerates a zone failure. Zonal (99.9 %) is riskier for this requirement. Standard Persistent Disk is not a shared POSIX filesystem for many pods (ReadWriteOnce). NetApp is premium, not needed without SnapMirror policy.

---

## Sources

- cloud.google.com/docs/storage (Cloud Storage classes, lifecycle, signed URLs, UBLA)
- cloud.google.com/docs/databases (Cloud SQL HA, PITR, read replicas)
- cloud.google.com/docs/bigtable (row-key design, multi-cluster)
- cloud.google.com/docs/firestore (consistency modes, real-time sync)
- cloud.google.com/memorystore and cloud.google.com/memorystore/sla (engines, SLAs)
- cloud.google.com/filestore/docs/service-tiers (Filestore tiers); cloud.google.com/blog/products/containers-kubernetes/filestore-features-for-stateful-workloads-on-gke (SLAs)
- cloud.google.com/sql/mysql (Cloud SQL Enterprise vs Enterprise Plus: SLA, PITR)
- cloud.google.com/firestore/native/docs/disaster-recovery (backups, PITR)
- docs.cloud.google.com/architecture/infra-reliability-guide/building-blocks (Bigtable SLAs)
- cloud.google.com/docs/spanner (global consistency, pricing)
- cloud.google.com/docs/alloydb (columnar engine, autoscaling)
- cloud.google.com/docs/architecture/well-architected-framework (reliability, cost pillars)
- cloud.google.com/docs/filestore (Zonal/Regional tiers, Enterprise tier)
- cloud.google.com/docs/bigquery (as storage, BI Engine)

