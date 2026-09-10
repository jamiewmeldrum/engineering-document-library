# AWS Service Reference №91

*A card per service at even depth, written to build understanding rather than to pass an exam. Where №54 teaches the **judgment** (which service, given these constraints) and weights toward exam frequency, this covers the estate evenly and asks a different question of each service: **what is it, how does it actually work, and what's the model underneath?** Companion to №54 — that doc's decision tables tell you what to pick; this one tells you what you're picking.*

**Card format:** *What it is · How it works · Key concepts · Choose it over · Gotchas.* Roughly 60 services, grouped by category. Read a category at a time rather than front-to-back.

**Caveat on specifics:** quotas, limits and pricing change constantly. Mechanisms and models — how a NAT gateway routes, how DynamoDB partitions, how KMS envelope encryption works — are stable and are what this document optimises for. Treat any number here as "the right order of magnitude, verify before you depend on it."

Contents: **1** Compute · **2** Storage · **3** Databases · **4** Networking · **5** Content delivery & edge · **6** Integration & messaging · **7** Identity & security · **8** Developer & deployment · **9** Observability & management · **10** Analytics & data

---

# 1 · Compute

## EC2 (Elastic Compute Cloud)
**What** Virtual machines. The foundational service everything else was built to avoid needing.
**How** A hypervisor (AWS's Nitro system) partitions physical hosts. Nitro offloads networking, storage and security to dedicated hardware cards, so guests get near-bare-metal performance and AWS can offer bare-metal instances on the same platform. You pick an **AMI** (a disk image), an **instance type** (the hardware shape), and it boots in a subnet with an **ENI** (virtual network card).
**Key concepts** Instance families (T burstable, M general, C compute, R memory, I storage, P/G accelerated) with generation and size (`m7g.xlarge` — 7th gen, Graviton, extra-large). **Graviton** = AWS's ARM chips, meaningfully cheaper per unit performance if your workload compiles for ARM. **User data** runs at first boot for bootstrapping. **Instance metadata** at `169.254.169.254` exposes identity and credentials — IMDSv2 requires a session token, which defeats the SSRF attacks that plagued v1. **Placement groups** control physical proximity: cluster (one rack, low latency), spread (separate hardware), partition (rack groups).
**Choose it over** Lambda/Fargate when you need OS control, long-running processes, specific licensing, GPU access, or predictable heavy load where reserved pricing beats per-request.
**Gotchas** T-family instances earn and spend **CPU credits** — exhaust them and you're throttled to a baseline fraction of a core, which produces baffling "it was fast yesterday" behaviour. Stopping an instance releases its public IP unless you attached an Elastic IP.

## Lambda
**What** Run a function without managing servers; AWS provisions, scales and bills per invocation.
**How** Your code is packaged (zip or container image) and, on invocation, AWS creates an **execution environment** — a lightweight micro-VM (Firecracker) with your runtime and code. **Cold start** is that creation plus your initialisation code; subsequent invocations reuse a warm environment. Concurrency scales by creating more environments, one per simultaneous request. Environments are frozen between invocations, which is why background threads and unflushed state don't survive.
**Key concepts** Handler signature per runtime. Code *outside* the handler runs once per environment — put SDK clients, connection pools and config parsing there. **Layers** share dependencies. **Versions** are immutable; **aliases** point at versions and support weighted shifting. **Reserved concurrency** caps and guarantees; **provisioned concurrency** pre-warms. Three invocation models: synchronous (caller waits), asynchronous (AWS queues internally, retries twice, then DLQ), and event-source-mapping (Lambda polls SQS/Kinesis/DynamoDB Streams and invokes in batches).
**Choose it over** EC2/Fargate for event-driven, spiky, or short work where paying for idle is waste.
**Gotchas** 15-minute ceiling. Traditional database connection pools are an anti-pattern — thousands of concurrent environments each opening connections will exhaust Postgres (**RDS Proxy** exists for exactly this). VPC-attached Lambdas historically had painful cold starts; much improved, but VPC attachment still means you need NAT or endpoints for outbound AWS calls.

## ECS (Elastic Container Service)
**What** AWS's own container orchestrator.
**How** You register a **task definition** (a JSON spec: image, CPU/memory, ports, environment, IAM roles, logging — conceptually a pod spec). A **task** is a running instantiation. A **service** maintains N tasks, replaces unhealthy ones, and registers them with a load balancer. The **scheduler** places tasks onto capacity — either EC2 instances running the ECS agent, or Fargate.
**Key concepts** Two IAM roles that people constantly conflate: the **task execution role** lets the ECS agent pull the image from ECR and write logs; the **task role** is what your application code uses to call AWS. **awsvpc** network mode gives each task its own ENI and security group — clean, but ENIs per instance are limited. Service discovery via Cloud Map; auto-scaling on CPU/memory/ALB request count.
**Choose it over** EKS when you don't need Kubernetes specifically — ECS is dramatically simpler and free (you pay only for compute).
**Gotchas** Task definitions are versioned and immutable; "updating" creates a new revision, and forgetting to point the service at it is a common confusion.

## Fargate
**What** Serverless compute *for containers* — a capacity provider for ECS and EKS, not a separate orchestrator.
**How** You stop managing EC2 instances entirely; AWS runs your task on its own managed infrastructure, billing per vCPU-second and GB-second of the resources your task declares.
**Key concepts** You pick CPU/memory from a fixed matrix of valid combinations. Each task gets its own kernel-level isolation. No SSH, no daemonsets, no host access. Fargate Spot offers big discounts with interruption.
**Choose it over** EC2 launch type when operational overhead matters more than per-unit cost — no patching, no cluster capacity planning, no bin-packing.
**Gotchas** More expensive per unit of compute than well-utilised EC2, so at large steady scale EC2 can win. No GPU support. No privileged containers.

## EKS (Elastic Kubernetes Service)
**What** Managed Kubernetes control plane.
**How** AWS runs the control plane (API server, etcd) across AZs; you supply worker nodes (managed node groups, self-managed, or Fargate). Standard Kubernetes from there.
**Key concepts** **IRSA** (IAM Roles for Service Accounts) maps a Kubernetes service account to an IAM role via OIDC — the right way to give pods AWS permissions. AWS VPC CNI gives pods real VPC IP addresses (which means pod density is bounded by ENI limits). Add-ons for CoreDNS, kube-proxy, EBS CSI driver.
**Choose it over** ECS when you need the Kubernetes ecosystem, multi-cloud portability, or your team already knows k8s.
**Gotchas** There's an hourly charge for the control plane per cluster. Kubernetes' operational complexity is real and doesn't disappear because the control plane is managed.

## Elastic Beanstalk
**What** A PaaS layer that provisions and manages conventional AWS resources for you.
**How** You upload code; Beanstalk creates an **environment** (EC2 instances, ASG, ELB, security groups, optionally RDS) from a **platform** (Java/Node/Python/Docker...) and handles deployment, health monitoring and capacity. The resources are ordinary AWS resources in your account — you can inspect and even modify them.
**Key concepts** Environment types (web server vs worker, the latter pulling from SQS). Deployment policies trading downtime against cost: all-at-once, rolling, rolling-with-additional-batch, immutable, blue/green. `.ebextensions` config files for customisation. Saved configurations for reproducibility.
**Choose it over** raw EC2 when you want a conventional app deployed without designing the infrastructure; over ECS/Fargate when you don't want to containerise.
**Gotchas** Coupling an RDS instance *into* the environment ties its lifecycle to the environment — terminate the environment and the database goes. Create databases separately.

## AWS Batch
**What** Managed batch job scheduling.
**How** You define **job definitions** (container + resources), submit **jobs** to **queues**, and Batch provisions **compute environments** (EC2 or Fargate, on-demand or Spot) to run them, scaling to zero when idle.
**Key concepts** Job dependencies and array jobs for parallel workloads; queue priorities; automatic instance-type selection to fit the job mix.
**Choose it over** rolling your own ASG for scientific computing, rendering, ETL, or any queue-of-work-items shape — especially with Spot.

## Lightsail / App Runner / Outposts
**Lightsail** — bundled VPS at a flat monthly price (instance + storage + transfer). Simplicity over flexibility; good for small sites and prototypes, and it can peer into a VPC. **App Runner** — point it at a container image or source repo and get an autoscaling HTTPS service with no infrastructure at all; the shortest path from code to running service. **Outposts** — physical AWS racks installed in your datacentre, running the same APIs, for latency or data-residency requirements.

---

# 2 · Storage

## S3 (Simple Storage Service)
**What** Object storage: a flat namespace of key→object mappings, effectively unlimited.
**How** Not a filesystem — there are no real directories, just keys that often contain slashes and a console that renders them as folders. Objects are replicated across multiple devices and AZs within a region, giving eleven nines of durability. Requests are HTTP; the "path" is the key. Since December 2020 all operations are **strongly read-after-write consistent** — an object is immediately readable after a successful PUT, including overwrites.
**Key concepts** **Versioning** keeps every revision (deletes insert a marker rather than removing data). **Lifecycle rules** transition objects between storage classes and expire them by age. **Storage classes** trade retrieval latency and cost: Standard → Intelligent-Tiering (auto-moves based on observed access) → Standard-IA / One Zone-IA → Glacier Instant / Flexible / Deep Archive. **Multipart upload** splits large objects into parallel parts (required above 5 GB, sensible above ~100 MB). **Presigned URLs** grant time-limited access to a specific object without credentials, signed by your key. **Event notifications** fire to Lambda/SQS/SNS/EventBridge on object creation or deletion. **S3 Select** runs SQL against a single object so you transfer only matching rows. **Replication** (cross- or same-region) copies objects asynchronously; requires versioning.
**Choose it over** EBS/EFS whenever the access pattern is whole-object read/write over HTTP rather than random block or POSIX file access.
**Gotchas** Bucket names are globally unique across all AWS accounts. Prefix-based request-rate scaling means poorly distributed key prefixes can throttle (5,500 GET/s per prefix, so spread hot keys). Incomplete multipart uploads accumulate and bill silently — add a lifecycle rule. Public access is blocked by default at four levels; deliberately unblocking all of them is how buckets leak.

## EBS (Elastic Block Store)
**What** Network-attached block devices for EC2 — virtual disks.
**How** A volume lives in **one AZ** and attaches to an instance over the network (Nitro makes this fast enough to feel local). The block device is raw; you format and mount it. **Snapshots** are incremental point-in-time copies stored in S3; only changed blocks are stored, but each snapshot is independently restorable.
**Key concepts** Volume types: **gp3** (general-purpose SSD; 3,000 IOPS and 125 MB/s baseline, both independently provisionable — the sane default), **gp2** (older, performance scales with size), **io1/io2 Block Express** (provisioned IOPS for demanding databases; io2 offers higher durability), **st1** (throughput-optimised HDD for large sequential reads), **sc1** (cold HDD). Volumes can be resized and type-changed live. Encryption is transparent, KMS-backed, and covers data at rest, in transit to the instance, and in snapshots.
**Choose it over** instance store when data must survive a stop; over EFS when only one instance needs the data and you want block-level performance.
**Gotchas** AZ-bound — you cannot attach a volume across AZs; you snapshot and restore. Detaching without unmounting risks filesystem corruption. Snapshot restores are lazily loaded, so first-touch reads are slow until blocks are hydrated.

## EFS / FSx / Instance Store
**EFS** — managed NFS. Multiple instances (and Lambda, and containers) mount the same filesystem concurrently, across AZs; capacity grows and shrinks automatically. Performance modes and throughput modes (bursting vs provisioned vs elastic) matter under load. Lifecycle management moves cold files to Infrequent Access. Choose it for shared state across a fleet — uploads, shared config, home directories. Slower than EBS for random I/O.
**FSx** — a family of managed third-party filesystems: **FSx for Windows File Server** (SMB, AD-integrated), **FSx for Lustre** (HPC, links to S3 as a backing store), **FSx for NetApp ONTAP** and **OpenZFS**. Choose when you need those specific protocols or features.
**Instance Store** — physically-attached NVMe on the host. Highest possible IOPS, zero durability: data is lost on stop, terminate or host failure. Correct for caches, scratch space, and replicated distributed stores that handle their own durability.

## Storage Gateway / DataSync / Snow / Backup
**Storage Gateway** — a virtual appliance on-premises presenting local NFS/SMB/iSCSI/VTL interfaces backed by S3/Glacier. Three modes: File (files as S3 objects), Volume (block, cached or stored), Tape (a virtual tape library replacing physical tapes). For hybrid environments that can't be rewritten.
**DataSync** — an agent-based bulk transfer service moving data between on-prem storage and AWS (or between AWS services) with validation, scheduling and throttling. Prefer it over hand-rolled scripts for one-off or recurring migrations.
**Snowball / Snowcone / Snowmobile** — physical devices shipped to you for data transfer where the network would take too long. The arithmetic that justifies them: 100 TB over a 1 Gbps link is roughly 10 days at full saturation.
**AWS Backup** — a central plane for backup policies across EBS, RDS, DynamoDB, EFS, FSx and more: schedules, retention, vaults, cross-region and cross-account copies, and compliance reporting.

---

# 3 · Databases

## RDS
**What** Managed relational databases: Postgres, MySQL, MariaDB, Oracle, SQL Server.
**How** AWS provisions an EC2 instance and EBS storage you don't see, installs and patches the engine, takes backups, and manages failover. You get an endpoint and admin-level (not superuser) database access; no OS access.
**Key concepts** **Multi-AZ** maintains a synchronous standby in another AZ and fails over automatically by repointing DNS — this is availability, not scale; the standby serves no traffic. **Read replicas** are asynchronous copies that *do* serve reads, can be cross-region, and are promoted manually. **Automated backups** enable point-in-time recovery within a retention window (up to 35 days) by combining daily snapshots with transaction logs. **Parameter groups** tune engine settings; **option groups** add engine features. **RDS Proxy** pools and multiplexes connections — the answer to Lambda exhausting your connection limit.
**Choose it over** self-managing on EC2 unless you need OS access, an unsupported engine, or extreme tuning.
**Gotchas** Encryption at rest must be enabled at creation — retrofitting means snapshot, copy-with-encryption, restore. Major version upgrades cause downtime and can't be undone. The maintenance window will apply patches; know when it is.

## Aurora
**What** AWS's reimplementation of MySQL and Postgres with a cloud-native storage layer.
**How** This is the interesting bit. Aurora decouples compute from storage: the database instances write **redo log records** (not pages) to a distributed storage fleet that replicates **six copies across three AZs** and handles replication, repair and backup itself. Quorum writes (4 of 6) and reads (3 of 6) tolerate losing an entire AZ plus one more node. Storage auto-grows in 10 GB increments to 128 TB. Because all instances share the same storage, replicas lag by milliseconds and adding one doesn't copy data.
**Key concepts** Up to 15 replicas, any of which can be promoted in ~30 seconds. **Aurora Serverless v2** scales capacity in fine-grained increments in-place, suiting spiky or intermittent load. **Global Database** replicates to other regions with sub-second lag for DR and local reads. **Backtrack** (MySQL) rewinds the cluster in place without a restore. Cluster endpoint (writer) vs reader endpoint (load-balanced across replicas) — using the wrong one is a common bug.
**Choose it over** RDS for higher throughput, faster failover, and cheaper read scaling; over DynamoDB when you need SQL, joins and transactions.
**Gotchas** More expensive at small scale. I/O charges can surprise on write-heavy workloads (Aurora I/O-Optimized pricing exists to address this).

## DynamoDB
**What** A managed key-value and document database with predictable single-digit-millisecond latency at effectively any scale.
**How** Data is spread across **partitions** by a hash of the **partition key**; each partition lives on SSD and is replicated across three AZs. Throughput and storage are distributed evenly across partitions, which is why key design dominates performance: all traffic to one key means all traffic to one partition. Adaptive capacity absorbs modest imbalance, but a genuinely hot key is a design error.
**Key concepts** Primary key is partition key alone, or partition + **sort key** (enabling range queries within a partition — the basis of single-table design). **GSI** (any attributes as key, own throughput, eventually consistent) vs **LSI** (same partition key, alternative sort key, created only at table creation, shares throughput, supports strong consistency). **Capacity**: on-demand (per-request billing, instant scaling) vs provisioned (RCU/WCU with auto-scaling, cheaper for steady load). **Streams** emit an ordered change log per partition key for 24 hours, typically consumed by Lambda. **TTL** deletes expired items in the background at no cost. **Transactions** give ACID across up to 100 items. **Conditional writes** implement optimistic concurrency. **DAX** is a DynamoDB-aware write-through cache giving microsecond reads.
**Choose it over** RDS when the access patterns are known and key-based, scale is large or spiky, and you don't need ad-hoc queries or joins.
**Gotchas** Query needs the partition key; anything else is a **Scan**, which reads the whole table. Item size caps at 400 KB. You must model for your access patterns *up front* — retrofitting a new query shape often means a new index or a migration. Eventually consistent reads by default.

## ElastiCache / MemoryDB
**ElastiCache** — managed Redis or Memcached. **Redis** offers persistence, replication with automatic failover, cluster mode (sharding), pub/sub, sorted sets, Lua scripting, and transactions; **Memcached** is a simpler multi-threaded pure cache with no persistence or replication. Use for cache-aside caching, session stores, rate limiters, leaderboards and queues. Gotcha: cache invalidation is yours to design (№31 §9), and a cold cache after failover can stampede the database.
**MemoryDB for Redis** — Redis as a *durable primary database*, with a multi-AZ transaction log, rather than a cache. Choose when you want Redis semantics without a separate source of truth.

## Redshift / Athena / OpenSearch
**Redshift** — a petabyte-scale columnar data warehouse. Columnar storage plus compression means analytical scans read only the columns needed; work is distributed across nodes by **distribution key**, and **sort keys** enable zone-map skipping. **Redshift Spectrum** queries S3 directly without loading. **Serverless** removes cluster management. Choose for complex analytical SQL over large historical data — not for OLTP.
**Athena** — serverless SQL directly over S3 using Presto/Trino, with schemas from the Glue Data Catalog. You pay per terabyte scanned, so **partitioning and columnar formats (Parquet) cut cost by orders of magnitude**. Choose for ad-hoc querying of logs and data lakes with zero infrastructure.
**OpenSearch Service** — managed OpenSearch/Elasticsearch: an inverted index for full-text search, plus log analytics and dashboards. Choose when `LIKE` queries or DynamoDB can't express your search.

## The specialised stores
**Neptune** — graph database supporting Gremlin, openCypher and SPARQL; for traversal-heavy relationship queries (fraud rings, recommendations, knowledge graphs). **DocumentDB** — MongoDB-API-compatible document store on an Aurora-like storage layer. **Timestream** — purpose-built time-series with automatic tiering from memory to magnetic and time-series functions built in. **QLDB** — an immutable, cryptographically verifiable ledger with a full change history. **Keyspaces** — serverless Cassandra-compatible wide-column. **DMS** — Database Migration Service, replicating between engines with minimal downtime; pairs with **SCT** (Schema Conversion Tool) for heterogeneous migrations.

---

# 4 · Networking

## VPC
**What** A logically isolated virtual network in a region, defined by a CIDR block.
**How** Software-defined networking implemented at the hypervisor/Nitro level: packets are encapsulated and routed according to your route tables, with security groups enforced at the ENI. There's no physical topology to reason about — "subnets" are routing and AZ constructs, not broadcast domains (there is no broadcast or multicast).
**Key concepts** A **subnet** is a CIDR slice bound to one AZ; it's *public* if its route table sends `0.0.0.0/0` to an **internet gateway**. AWS reserves **five IPs per subnet** (network address, VPC router, DNS, reserved, broadcast). **Route tables** direct traffic by destination prefix, longest match wins. **ENIs** are the virtual NICs that carry security groups and IPs. **DHCP option sets** control DNS. **Flow logs** record accepted/rejected traffic metadata to CloudWatch or S3.
**Gotchas** CIDR blocks can be extended but never shrunk; overlapping CIDRs make future peering impossible, so plan address space deliberately across environments and accounts.

## Security groups vs NACLs
**Security group** — stateful, instance-level, allow-rules-only. Return traffic for an allowed connection is automatically permitted. Rules can reference other security groups, which is how you express "the database accepts 5432 only from the application tier" without hardcoding IPs. All rules are evaluated as a union.
**NACL** — stateless, subnet-level, supports explicit **deny**, evaluated in rule-number order with first match winning. Because it's stateless you must allow return traffic explicitly (typically ephemeral ports 1024–65535). Use for coarse subnet-wide blocks — blocking a malicious IP range — not day-to-day access control.

## Internet gateway, NAT, endpoints
**Internet gateway** — a horizontally-scaled, highly-available VPC attachment that performs 1:1 NAT between private and public IPs. Free.
**NAT gateway** — managed outbound-only NAT so private subnets can reach the internet without being reachable from it. AZ-scoped, so one per AZ for high availability. Billed hourly *and* per GB processed, which makes it a common source of unexpected cost.
**VPC endpoints** — private connectivity to AWS services without traversing the internet. **Gateway endpoints** (S3 and DynamoDB only) are free and work via route-table entries. **Interface endpoints / PrivateLink** put an ENI in your subnet with a private DNS name, billed hourly plus per GB, and also expose *third-party or your own* services privately across accounts.

## Peering, Transit Gateway, hybrid
**VPC peering** — a direct one-to-one connection between two VPCs, any account or region. Non-transitive (A–B and B–C does not give A–C) and requires non-overlapping CIDRs. Fine for a handful of VPCs; unmanageable at scale.
**Transit Gateway** — a regional hub that connects VPCs, VPNs and Direct Connect in a hub-and-spoke topology with route tables for segmentation. The scalable answer past a few VPCs, and supports transitive routing.
**Site-to-Site VPN** — IPsec tunnels over the public internet to your on-prem router. Quick to set up, encrypted, but subject to internet variability.
**Direct Connect** — a dedicated physical circuit into an AWS location. Consistent latency and bandwidth, lower data-transfer rates, but lead times measured in weeks; typically backed up by a VPN.

## Elastic Load Balancing
**ALB** — layer 7. Understands HTTP, so it can route on path, host, header, query string and method; supports WebSockets, HTTP/2, redirects, fixed responses, authentication via Cognito/OIDC, and targets that are instances, IPs, Lambda functions or containers. Health checks are HTTP-based.
**NLB** — layer 4. Forwards TCP/UDP/TLS at very high throughput with ultra-low latency, preserves source IP, and provides a **static IP per AZ** (or Elastic IP), which matters for allow-listing.
**GWLB** — layer 3, for inserting fleets of virtual security appliances transparently into the traffic path using GENEVE encapsulation.
Common concepts: target groups, health checks, cross-zone load balancing, deregistration delay (connection draining), TLS termination with ACM certificates, sticky sessions via cookies.

## Route 53
**What** DNS with health checking and traffic management, plus domain registration.
**How** Authoritative DNS served from a global anycast network. Health checks probe endpoints from multiple locations and can be composed (calculated health checks) or driven by CloudWatch alarms.
**Key concepts** Routing policies: simple, weighted (traffic splitting), latency-based, failover (active/passive), geolocation, geoproximity (with bias), multivalue answer. **Alias records** are an AWS extension pointing at AWS resources — free to query, and legal at the zone apex where CNAME is not. **Private hosted zones** serve names inside a VPC. **Resolver endpoints** enable hybrid DNS resolution to and from on-premises.

---

# 5 · Content delivery & edge

## CloudFront
**What** A CDN with hundreds of edge locations and regional caches.
**How** A request hits the nearest edge; on a cache miss it goes to a regional edge cache and then the **origin**, and the response is cached according to headers and your **cache policy**. Cache keys are configurable (which headers, cookies and query strings matter), and getting them wrong either fragments the cache or serves the wrong content.
**Key concepts** **OAC** (Origin Access Control, replacing OAI) locks an S3 origin so it's only reachable through CloudFront. **Signed URLs** (one file) and **signed cookies** (many files) gate private content. **Behaviours** map path patterns to different origins and policies. **Invalidations** purge cached objects — versioned filenames are cheaper and better. **Lambda@Edge** and **CloudFront Functions** run code at the edge (the latter is lighter, faster and cheaper for header manipulation and redirects). Origins can be S3, ALB, API Gateway or any HTTP server.
**Gotchas** Caching is only as good as your cache-control headers; forwarding all cookies or query strings effectively disables caching.

## Global Accelerator
**What** Static anycast IPs that route traffic into the AWS backbone from the nearest edge.
**How** Unlike CloudFront it doesn't cache — it improves the *network path*, entering AWS's private backbone early rather than traversing the public internet. Supports TCP and UDP, does health-checked failover between regional endpoints in seconds, and gives you two fixed IPs.
**Choose it over** CloudFront for non-cacheable, non-HTTP, or latency-sensitive traffic (gaming, VoIP, IoT), or when you need static IPs for allow-listing.

## API Gateway
**What** A managed front door for APIs — routing, auth, throttling, transformation.
**How** Requests hit an endpoint (edge-optimised, regional or private), pass through authorisation, request validation and optional transformation, then integrate with a backend: Lambda, HTTP, or an AWS service directly.
**Key concepts** **REST APIs** support request/response mapping templates (VTL), API keys with usage plans, caching, WAF integration and canary deployments. **HTTP APIs** are cheaper and lower-latency with a reduced feature set — the better default for straightforward Lambda or HTTP proxying. **WebSocket APIs** manage persistent connections with `$connect`/`$disconnect`/route keys. **Stages** are deployment environments with stage variables. **Authorisers**: IAM, Cognito user pools, or Lambda (token or request-based) with result caching. **Lambda proxy integration** passes the raw request through and expects a specific response shape.
**Gotchas** Default 29-second integration timeout. VTL mapping templates are powerful and miserable to debug — prefer proxy integration and handle it in code.

## AppSync
**What** Managed GraphQL (and now Events). Resolvers map GraphQL fields to DynamoDB, Lambda, RDS, OpenSearch or HTTP sources, with built-in subscriptions over WebSockets for real-time updates, offline sync via Amplify, and fine-grained auth per field. Choose it when clients need to shape their own queries or you want real-time subscriptions without building the plumbing.

---

# 6 · Integration & messaging

## SQS
**What** A managed message queue that decouples producers from consumers.
**How** Messages are stored redundantly across AZs. Consumers **poll**; a received message becomes invisible for the **visibility timeout** while being processed, and must be explicitly **deleted** — if the consumer dies, the message reappears. That's the at-least-once model, and why consumers must be idempotent (№31 §8).
**Key concepts** **Standard** queues offer near-unlimited throughput, best-effort ordering, at-least-once delivery. **FIFO** queues guarantee ordering within a **message group** and deduplicate within a 5-minute window. **Long polling** (up to 20s) eliminates empty responses and cost. **DLQ** captures messages that exceed `maxReceiveCount` — essential for poison-message handling. Max message size 256 KB (larger payloads go in S3 with a pointer). Retention 4 days by default, up to 14. **Delay queues** and per-message timers postpone delivery.
**Gotchas** Visibility timeout shorter than processing time causes duplicate processing — a top source of real bugs. FIFO throughput is bounded, and ordering is per message group, not per queue.

## SNS
**What** Pub/sub notification service: publish once, deliver to many subscribers.
**How** A **topic** fans out to subscriptions — SQS queues, Lambda functions, HTTP endpoints, email, SMS, mobile push. Delivery is push-based with retry policies per protocol; failures can go to a DLQ.
**Key concepts** The **fan-out pattern** (SNS → several SQS queues) gives each consumer its own durable, independently-paced copy — the canonical decoupling design. **Message filtering** lets subscribers receive only matching messages based on attributes. **FIFO topics** pair with FIFO queues for ordered fan-out.

## EventBridge
**What** A serverless event bus with routing rules — the evolution of CloudWatch Events.
**How** Events (JSON with a defined envelope) arrive on a bus from AWS services, your applications, or SaaS partners. **Rules** match event patterns and route to targets, optionally transforming the payload.
**Key concepts** The **default bus** receives AWS service events (state changes across the estate); **custom buses** carry your domain events; **partner buses** receive SaaS events. **Schema registry** discovers event structures and generates code bindings. **Scheduler** runs cron and rate expressions at scale. **Archive and replay** re-delivers historical events — invaluable for recovering from a consumer bug. **Pipes** connect a source to a target with optional filtering and enrichment.
**Choose it over** SNS when you need content-based routing, AWS-service event sources, schemas or replay; SNS remains simpler and cheaper for raw high-throughput fan-out.

## Kinesis family
**Data Streams** — an ordered, replayable log sharded by partition key. Each shard supports 1 MB/s in and 2 MB/s out (or enhanced fan-out for 2 MB/s per consumer). Records persist 24 hours by default, up to 365 days, and multiple independent consumers each track their own position. This is the AWS analogue of Kafka; choose it when several consumers need the same stream, order matters per key, or replay is required.
**Data Firehose** — a fully managed delivery pipeline that buffers and writes to S3, Redshift, OpenSearch or third parties, with optional Lambda transformation and format conversion to Parquet. No shards to manage; near-real-time rather than real-time.
**Managed Service for Apache Flink** — stateful stream processing with SQL or Flink applications.
**Video Streams** — ingestion and storage for media and ML pipelines.
**MSK** — fully managed Apache Kafka when you want Kafka itself, its ecosystem, or portability.

## Step Functions
**What** Serverless orchestration expressed as a state machine.
**How** You define states in Amazon States Language (JSON/YAML): Task, Choice, Parallel, Map, Wait, Pass, Succeed, Fail. The service tracks execution state durably, handles retries with backoff and catch blocks per state, and visualises every execution.
**Key concepts** **Standard** workflows run up to a year with exactly-once execution and full history — the right choice for business processes and sagas (№31 §8.2). **Express** workflows run up to five minutes at very high volume with at-least-once semantics and CloudWatch-based observability. Direct SDK integrations call 200+ AWS services without Lambda glue. **Callback patterns** (`waitForTaskToken`) pause until an external system responds — how you model human approval steps.
**Choose it over** chaining Lambdas manually: retries, error handling, state and visibility come free, and the workflow becomes inspectable rather than implicit.

## Amazon MQ
Managed ActiveMQ or RabbitMQ. Choose only when you need standard protocols (JMS, AMQP, MQTT, STOMP) — usually because you're migrating an existing application. For new work, SQS/SNS/EventBridge are cheaper and scale further.

---

# 7 · Identity & security

## IAM
**What** The authentication and authorisation layer for every AWS API call.
**How** Every request is signed (SigV4) and evaluated against all applicable policies: identity-based, resource-based, permissions boundaries, SCPs and session policies. The evaluation is deny-by-default; an explicit `Deny` anywhere is final; otherwise an explicit `Allow` grants.
**Key concepts** **Users** hold long-lived credentials (avoid for workloads); **groups** collect users; **roles** are assumable identities with temporary credentials and are the correct mechanism for services, cross-account access and federation. **Policies** are JSON with Effect/Action/Resource/Condition; conditions are where fine-grained control lives (`aws:SourceIp`, `aws:PrincipalOrgID`, `s3:prefix`, tag matching for ABAC). **Instance profiles** attach roles to EC2. **Access Analyzer** identifies resources shared outside your trust boundary and can generate least-privilege policies from CloudTrail history.
**Gotchas** `iam:PassRole` is the quiet privilege-escalation vector — granting it broadly lets a user hand a powerful role to a service they control. Policy evaluation with boundaries and SCPs is an intersection, so permissions can be silently capped somewhere you're not looking.

## STS / Organizations / Identity Center
**STS** issues temporary credentials: `AssumeRole` (cross-account and service roles), `AssumeRoleWithWebIdentity` (OIDC, used by Cognito and EKS IRSA), `AssumeRoleWithSAML` (enterprise federation), plus session tagging.
**Organizations** groups accounts into an OU hierarchy with consolidated billing and **Service Control Policies** — guardrails that cap what member accounts may do, never grant. Account-per-environment is the strongest isolation boundary AWS offers.
**IAM Identity Center** provides workforce single sign-on across accounts with permission sets, integrating with an external IdP. The modern replacement for per-account IAM users.

## KMS / CloudHSM
**KMS** — managed keys with FIPS-validated backing. The mechanism worth understanding is **envelope encryption**: KMS never encrypts your bulk data. Instead `GenerateDataKey` returns a plaintext data key *and* an encrypted copy; you encrypt data locally with the plaintext key, discard it, and store the encrypted key alongside the ciphertext. To decrypt, you ask KMS to decrypt the data key. This keeps the master key inside KMS while allowing unlimited data throughput. **Key policies** (resource-based) are the primary access control, combined with IAM. **Encryption context** provides authenticated additional data and appears in CloudTrail. Customer-managed keys support automatic annual rotation; AWS-managed keys are free but less controllable. Multi-region keys replicate for cross-region decryption.
**CloudHSM** — single-tenant hardware security modules where you alone hold the keys. For regulatory requirements KMS can't satisfy, or when you need the HSM's own APIs (PKCS#11).

## Secrets Manager / Parameter Store
**Secrets Manager** stores secrets encrypted with KMS and — the differentiator — **rotates them automatically** via a Lambda function, with native integration for RDS, Redshift and DocumentDB. Supports cross-account and cross-region replication. Priced per secret per month.
**SSM Parameter Store** stores configuration and secrets in a hierarchy, with `String`, `StringList` and KMS-encrypted `SecureString` types. The standard tier is free; no built-in rotation. Versioning and change notification via EventBridge. For most configuration, Parameter Store is the right default; reach for Secrets Manager when rotation or its integrations earn the cost.

## Cognito
**User Pools** — a managed user directory handling sign-up, sign-in, MFA, password policies, hosted UI, and federation with social and SAML/OIDC providers. On success it issues **JWTs** (ID, access, refresh tokens) that your API validates — API Gateway and ALB can do this natively.
**Identity Pools (Federated Identities)** — exchange an identity (from a user pool, a social provider, or SAML) for **temporary AWS credentials** via STS, so a client can call S3 or DynamoDB directly with fine-grained, per-user permissions.
The distinction is authentication (user pools) versus AWS authorisation (identity pools); many applications use only the former.

## The protective services
**WAF** — layer-7 rules on CloudFront, ALB, API Gateway and AppSync: managed rule groups for OWASP-style threats, rate-based rules, IP sets, geo-matching and custom logic on any request component.
**Shield** — Standard is free and always on for network/transport DDoS; Advanced adds application-layer protection, a 24/7 response team, and cost protection against scaling during an attack.
**GuardDuty** — continuous threat detection analysing CloudTrail, VPC Flow Logs, DNS logs, EKS audit logs and S3 data events with ML and threat intelligence. No agents; findings feed Security Hub or EventBridge.
**Inspector** — automated vulnerability scanning of EC2, ECR images and Lambda functions against CVE databases, with risk-scored findings.
**Macie** — ML-driven discovery and classification of sensitive data (PII, credentials) in S3, with alerting on exposure.
**Security Hub** — aggregates findings across all of the above plus third parties, and scores against standards (CIS, AWS Foundational Security Best Practices).
**Detective** — builds a graph from your logs to investigate a finding's blast radius after the fact.
**ACM** — free public TLS certificates with automatic renewal for AWS-integrated services (ALB, CloudFront, API Gateway); private CA available for internal PKI. Note certificates for CloudFront must live in `us-east-1`.

---

# 8 · Developer & deployment

## CloudFormation
**What** Declarative infrastructure as code, native to AWS.
**How** You submit a template describing desired resources; CloudFormation computes and executes a change plan, tracking every resource in a **stack** with dependency ordering, rollback on failure, and drift detection.
**Key concepts** Template anatomy: Parameters, Mappings, Conditions, Resources, Outputs, Transform. **Intrinsic functions** — `!Ref`, `!GetAtt`, `!Sub`, `!Join`, `!ImportValue`. **Change sets** preview modifications before applying. **Nested stacks** and cross-stack exports compose larger systems. **StackSets** deploy across accounts and regions. **DeletionPolicy** and `UpdateReplacePolicy` protect stateful resources from accidental destruction. **Custom resources** call a Lambda for anything CloudFormation doesn't natively support.
**Gotchas** Some property changes force **replacement** rather than update — silently recreating a database. Read the change set. Stuck `UPDATE_ROLLBACK_FAILED` states are a known pain.

## SAM / CDK / Amplify
**SAM** — a CloudFormation transform with concise serverless resource types (`AWS::Serverless::Function`, `::Api`, `::SimpleTable`) plus a CLI for local invocation and testing (`sam local invoke`, `sam local start-api`) and guided deployment.
**CDK** — write infrastructure in TypeScript, Python, Java, Go or C#, using **constructs** that compose into higher-level abstractions and synthesise to CloudFormation. You get loops, conditionals, types and unit tests; you also get the ability to generate a great deal of infrastructure from very little code, which cuts both ways.
**Amplify** — a full-stack toolchain for web and mobile: hosting with CI/CD, plus generated backends (auth via Cognito, data via AppSync/DynamoDB, storage via S3) and client libraries.

## The Code* suite
**CodeCommit** — managed Git hosting with IAM-based access.
**CodeBuild** — fully managed build service; a `buildspec.yml` defines install/pre_build/build/post_build phases, artifacts and caching. Runs in a container image you choose; scales per build with no build servers.
**CodeDeploy** — deploys to EC2/on-prem, Lambda and ECS. An `appspec.yml` defines files and lifecycle hooks (`BeforeInstall`, `AfterInstall`, `ApplicationStart`, `ValidateService`). Supports in-place and blue/green, with canary and linear traffic-shifting configurations and automatic rollback on alarm.
**CodePipeline** — orchestrates source, build, test, approval and deploy stages with parallel actions and manual approvals.
**CodeArtifact** — managed artifact repository (npm, Maven, PyPI, NuGet) with upstream proxying to public registries.
**CodeGuru** — automated code review (Reviewer) and production profiling to find performance and cost hotspots (Profiler).
**ECR** — private container registry with image scanning, lifecycle policies, immutable tags, cross-region replication and IAM-based auth (№50).

---

# 9 · Observability & management

## CloudWatch
**What** The metrics, logs, alarms and dashboards service.
**How** Metrics are time-series identified by namespace, name and **dimensions**, retained with decreasing granularity over 15 months. AWS services publish automatically; you publish your own with `PutMetricData` (or embedded metric format from logs). Logs arrive as events in streams within groups.
**Key concepts** **Alarms** evaluate a metric against a threshold over periods and act via SNS, Auto Scaling or EC2 actions; **composite alarms** combine them to cut noise. **Logs Insights** provides a query language over log groups. **Metric filters** turn log patterns into metrics. **Contributor Insights** finds top-N contributors. The **CloudWatch agent** is required for memory, disk and process metrics from EC2 (the hypervisor cannot see inside the guest). **Synthetics canaries** script user journeys; **RUM** captures real user telemetry; **Evidently** does feature flags and experiments.

## X-Ray
**What** Distributed tracing across services.
**How** Instrumented services emit **segments** (and nested **subsegments**) tagged with a trace ID propagated via the `X-Amzn-Trace-Id` header. X-Ray assembles them into a trace and aggregates traces into a **service map** showing latency and error rates per edge. **Sampling rules** control cost and volume.
**Key concepts** **Annotations** are indexed and filterable; **metadata** is stored but not searchable — use annotations for anything you'll query on. Native integration with Lambda, API Gateway, ECS, Beanstalk and the SDKs. The AWS Distro for OpenTelemetry is the vendor-neutral path (№31 §11).

## CloudTrail / Config / Systems Manager
**CloudTrail** — records every API call: who, what, when, from where. **Management events** (control plane) are on by default with 90 days of history; **data events** (S3 object-level, Lambda invocations) are high-volume and opt-in. Trails deliver to S3 for long-term retention, optionally with log-file integrity validation. This is your forensic record — "who deleted the bucket."
**Config** — records resource configuration over time and evaluates **rules** for compliance, with remediation actions and conformance packs. Answers "what did this look like last Tuesday, and was it ever non-compliant?"
**Systems Manager** — a large toolbox worth knowing: **Session Manager** (browser/CLI shell to instances with no SSH, no bastion, no open ports — audited via CloudTrail), **Patch Manager**, **Run Command**, **State Manager**, **Automation** runbooks, **Inventory**, and **Parameter Store**. Session Manager alone justifies learning it.

## Trusted Advisor / Compute Optimizer / Health / Control Tower
**Trusted Advisor** checks across cost, performance, security, fault tolerance and service limits. **Compute Optimizer** uses CloudWatch history and ML to recommend right-sized EC2, EBS, Lambda and ECS-on-Fargate configurations. **Personal Health Dashboard** reports events affecting *your* resources specifically. **Control Tower** sets up a multi-account landing zone with guardrails, an account factory and centralised logging — the opinionated way to start an Organization.

---

# 10 · Analytics & data

**Glue** — serverless ETL plus the **Data Catalog**, a central metastore (Hive-compatible) that Athena, Redshift Spectrum and EMR all read. Crawlers infer schemas from S3; jobs run Spark or Python shell transformations.
**EMR** — managed Hadoop/Spark/Hive/Presto clusters for large-scale processing, with Spot-heavy task nodes for cost and EMR Serverless for no cluster management.
**Lake Formation** — builds on Glue to add fine-grained (table, column, row, cell) permissions across a data lake, centralising access control that would otherwise be scattered across S3 policies.
**QuickSight** — BI dashboards with the SPICE in-memory engine and ML-driven insights.
**Data Exchange / DataZone** — third-party data subscriptions, and data governance/cataloguing across an organisation.

---

# How this pairs with the rest of the library

- **№54** — the exam-oriented companion: domains, decision tables, question patterns, and a study plan for SAA-C03 and DVA-C02. Use this document to *understand* a service; use that one to *choose between* services under a constraint.
- **№51 Networking** — the protocol foundations under VPC, ELB, Route 53, CloudFront and TLS. Read it first if the networking cards feel thin on *why*.
- **№31 Distributed Systems** — the concepts under SQS, SNS, Kinesis, Step Functions, caching and multi-AZ design.
- **№50 Docker** — the container model under ECR, ECS, Fargate and EKS.
- **№20 Data-Access** — the relational foundations under RDS and Aurora.
- **№43 System Design** — assembling these services into architectures.

*Mechanisms and models here are stable; quotas, pricing and feature availability change continuously. AWS documentation is the authority for anything you'd build on.*
