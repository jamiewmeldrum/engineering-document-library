# AWS — A Primer for SAA-C03 & DVA-C02 №54

*Comprehensive study documentation for the **AWS Certified Solutions Architect – Associate (SAA-C03)** and **AWS Certified Developer – Associate (DVA-C02)** exams. Both exam versions verified current, July 2026. Structured as a study reference: services grouped by category, exam-relevant facts in tables, and — most importantly — the **decision frameworks** (Part 13) that the exams actually test. Companion to №31 (Distributed Systems), №43 (System Design), №51 (Networking), №50 (Docker).*

**The single most important thing to understand about these exams:** they are not memory tests of service definitions. Almost every question is a **scenario** ending in "which solution meets these requirements *most cost-effectively* / *with least operational overhead* / *most securely*." Four options are usually all technically capable of working; you're choosing on the qualifier. So the study goal isn't "what is SQS" — it's **"given these constraints, SQS or SNS or EventBridge or Kinesis, and why."** Part 13 is therefore the heart of this document; everything before it is the vocabulary you need to use it.

**Quotas caveat:** the limits quoted here are the well-known, exam-stable ones. AWS adjusts quotas regularly — verify anything you'd stake a design on against current AWS documentation.

Contents:

- **Part 1** — the two exams: format, domains, strategy
- **Part 2** — core concepts: regions, AZs, shared responsibility, Well-Architected
- **Part 3** — IAM and identity
- **Part 4** — compute
- **Part 5** — storage
- **Part 6** — databases
- **Part 7** — networking and content delivery
- **Part 8** — application integration and messaging
- **Part 9** — deployment and developer tools
- **Part 10** — observability
- **Part 11** — security services
- **Part 12** — cost optimisation
- **Part 13** — decision frameworks: which service, when
- **Part 14** — exam technique: question patterns and traps
- **Part 15** — a study plan

---

# Part 1 — The two exams

## 1.1 Format (both exams)

| | SAA-C03 | DVA-C02 |
|---|---|---|
| Questions | 65 (50 scored, 15 unscored) | 65 (50 scored, 15 unscored) |
| Time | 130 minutes | 130 minutes |
| Cost | $150 USD | $150 USD |
| Score | 100–1000, **pass at 720** | 100–1000, **pass at 720** |
| Format | multiple choice / multiple response | multiple choice / multiple response |
| Validity | 3 years | 3 years |

Holding any active AWS certification gets you a **50% discount voucher** for the next one.

## 1.2 Domains

**SAA-C03 — you decide *what to use and how it fits together*:**

| Domain | Weight | Essence |
|---|---|---|
| 1. Design Secure Architectures | 30% | IAM, encryption, network isolation |
| 2. Design Resilient Architectures | 26% | multi-AZ, decoupling, fault tolerance |
| 3. Design High-Performing Architectures | 24% | right service, caching, scaling |
| 4. Design Cost-Optimised Architectures | 20% | pricing models, storage classes, right-sizing |

**DVA-C02 — you *build, deploy and debug* on AWS:**

| Domain | Weight | Essence |
|---|---|---|
| 1. Development with AWS Services | 32% | Lambda, DynamoDB, API Gateway, SDKs |
| 2. Security | 26% | Cognito, KMS, IAM for applications, secrets |
| 3. Deployment | 24% | CI/CD, CloudFormation/SAM, Beanstalk, containers |
| 4. Troubleshooting and Optimisation | 18% | X-Ray, CloudWatch, retries, caching |

## 1.3 How they differ, and the overlap

Roughly **30–40% of the content overlaps** — IAM, VPC basics, S3, DynamoDB, Lambda, CloudWatch, KMS all appear in both. The difference is the *lens*:

- **SAA** asks "which architecture?" — breadth across many services, favouring managed and serverless answers, always weighing cost/resilience/security.
- **DVA** asks "how do I implement it?" — depth in a narrower set (Lambda, DynamoDB, API Gateway, SQS/SNS, Cognito, KMS, CodePipeline, X-Ray, CloudFormation/SAM), including SDK behaviour, error codes, retry logic, and deployment mechanics.

**Recommended order: SAA first, then DVA.** SAA gives the broad foundation that makes everything else easier; DVA then needs only a few weeks of focused work on the developer-specific services. Doing DVA first works if you're deep in application code and want the faster win, but you'll do more groundwork later.

> **The tell — exam strategy:** for **SAA**, when in doubt prefer the answer that is *managed, serverless, multi-AZ, and least-operational-overhead*. For **DVA**, prefer the answer that uses *the SDK's built-in mechanism* (retries, pagination, encryption) rather than hand-rolling it. Those two heuristics alone resolve a surprising share of questions.

---

# Part 2 — Core concepts

## 2.1 The global infrastructure

- **Region** — a geographic area (e.g. `eu-west-2`, London). Independent; data doesn't leave unless you move it. Choose for latency, compliance/data residency, service availability, and price (prices differ per region).
- **Availability Zone (AZ)** — one or more discrete datacentres within a region, isolated for power/cooling/networking but connected by low-latency links. **Regions have at least 3.** Deploying across ≥2 AZs is the fundamental resilience move and the answer to a large fraction of SAA questions.
- **Edge location / Point of Presence** — hundreds worldwide; CloudFront, Route 53, and Global Accelerator use them for low-latency delivery.
- **Local Zones** (compute near a metro), **Wavelength** (5G networks), **Outposts** (AWS hardware in your datacentre) — know they exist and their one-line purpose.

## 2.2 The shared responsibility model

**AWS secures the cloud; you secure what's in it.**

| AWS's responsibility ("of" the cloud) | Your responsibility ("in" the cloud) |
|---|---|
| hardware, datacentres, physical security | your data, and its classification |
| the hypervisor, managed-service internals | IAM users, roles, and policies |
| global network infrastructure | OS patching **on EC2** (not on Lambda/RDS) |
| managed-service patching (RDS engine, Lambda runtime) | security-group and NACL rules |
| | application-level security, encryption choices |

The exam boundary that matters: **the more managed the service, the less is yours.** On EC2 you patch the OS; on RDS you don't; on Lambda you don't even have an OS. Encryption *in transit and at rest* is always available to you, and configuring it is always yours.

## 2.3 The Well-Architected Framework — six pillars

The mental checklist the SAA exam is literally built on:

| Pillar | Asks |
|---|---|
| **Operational Excellence** | can we run and monitor this? IaC, small reversible changes |
| **Security** | least privilege, defence in depth, encryption everywhere, traceability |
| **Reliability** | does it survive failure? multi-AZ, auto-recovery, backups, no SPOF |
| **Performance Efficiency** | right resource for the job, serverless where possible, measure |
| **Cost Optimisation** | right-size, right pricing model, right storage class, kill idle |
| **Sustainability** | minimise resources consumed |

When a question says "most cost-effective" or "most resilient," it's asking you to apply one of these pillars while holding the others constant.

---

# Part 3 — IAM and identity

The security backbone. **Heavily tested in both exams** (SAA domain 1 is 30%; DVA domain 2 is 26%).

## 3.1 The entities

| Entity | Is | Note |
|---|---|---|
| **Root user** | the account owner, unrestricted | enable MFA, then **never use it**; only for a handful of account-level tasks |
| **IAM user** | a person or app with long-lived credentials | avoid for workloads — credentials leak |
| **IAM group** | a collection of users | policies attach here; groups can't nest, and **can't be a principal** |
| **IAM role** | a set of permissions **assumed temporarily** | **the correct answer for services and cross-account** — no stored credentials |
| **Policy** | a JSON document of permissions | attached to users/groups/roles (identity) or resources |

**Roles are the single most exam-relevant idea here.** EC2 needs S3 access → instance profile with a role. Lambda needs DynamoDB → execution role. Account A needs account B → cross-account role. An application on-premises → IAM Roles Anywhere or federation. If an answer option involves **storing access keys** on an instance or in code, it is almost certainly wrong.

## 3.2 Policy structure and evaluation

```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Sid": "AllowReadOnPracticqBucket",
    "Effect": "Allow",
    "Action": ["s3:GetObject", "s3:ListBucket"],
    "Resource": ["arn:aws:s3:::practiq-assets", "arn:aws:s3:::practiq-assets/*"],
    "Condition": {"IpAddress": {"aws:SourceIp": "10.0.0.0/16"}}
  }]
}
```

Note the two ARN forms — the bucket for `ListBucket`, the `/*` for object actions. Getting that wrong is a classic real-world and exam trap.

**Evaluation logic — memorise this order:**

1. **Explicit DENY** anywhere → **denied**. Always wins, no exceptions.
2. Otherwise, an explicit **ALLOW** → allowed.
3. Otherwise → **implicit deny** (default).

Policy types: **identity-based** (on user/group/role), **resource-based** (on the resource — S3 bucket policy, SQS queue policy, KMS key policy; these enable cross-account access without assuming a role), **permissions boundaries** (a ceiling on what an identity *can* be granted), **SCPs** (Organizations-level guardrails — they *limit*, never grant), and **session policies**.

**Effective permissions = the intersection** of identity policy, boundary, and SCP — with any explicit deny overriding everything.

## 3.3 Federation and multi-account

- **IAM Identity Center** (formerly AWS SSO) — the modern way to manage human access across many accounts, integrating with an external IdP.
- **STS** — issues temporary credentials; `AssumeRole` is the workhorse. `AssumeRoleWithWebIdentity` backs Cognito identity pools.
- **AWS Organizations** — consolidated billing, **SCPs** as guardrails, account-per-environment as an isolation boundary (a common SAA best-practice answer).

> **The tell — IAM:** roles over users, temporary credentials over long-lived keys, least privilege, explicit deny wins, and SCPs restrict rather than grant. If a question involves credentials on an instance or in source code, look for the role-based option.

---

# Part 4 — Compute

## 4.1 EC2

**Instance families** (know the letters): **T** burstable (dev, low-traffic), **M** general purpose, **C** compute-optimised (CPU-bound), **R/X** memory-optimised (in-memory DBs, caches), **I/D** storage-optimised (high IOPS), **P/G/Inf** accelerated (GPU/ML).

**Purchasing models — a guaranteed cost question:**

| Model | Discount | Commitment | Use for |
|---|---|---|---|
| **On-Demand** | baseline | none | spiky, unpredictable, short-term |
| **Savings Plans** | up to ~72% | 1 or 3 yr $/hr spend | **flexible across instance family/region/service (incl. Lambda, Fargate)** |
| **Reserved Instances** | up to ~72% | 1 or 3 yr specific instance | steady-state, known workload |
| **Spot** | up to ~90% | none — **2-minute interruption notice** | fault-tolerant, stateless, batch, CI |
| **Dedicated Host** | most expensive | — | licensing tied to physical cores, compliance |

The exam signals: "fault-tolerant batch processing, minimise cost" → **Spot**. "Steady predictable workload for 3 years" → **Reserved/Savings Plan**. "Compliance requires physical isolation / BYOL licensing" → **Dedicated Host**. "Unpredictable spiky" → **On-Demand**.

**Placement groups:** **Cluster** (same rack — lowest latency, highest throughput, but a rack failure takes all), **Spread** (separate hardware, max 7 per AZ — critical individual instances), **Partition** (grouped racks — HDFS/Cassandra style).

**Auto Scaling Groups (ASG):** desired/min/max capacity, scaling policies (**target tracking** — the usual answer, e.g. keep CPU at 50%; step scaling; scheduled), health checks (EC2 or ELB), cooldowns, and lifecycle hooks. ASG + ELB across multiple AZs is the canonical resilient-architecture answer.

**Other:** user data (bootstrap script at first boot), instance metadata at `169.254.169.254` (**IMDSv2** is the secure, session-based version — prefer it), AMIs for golden images.

## 4.2 Lambda — heavily tested, especially DVA

The serverless workhorse. Numbers to know:

| Property | Value |
|---|---|
| Max timeout | **15 minutes** |
| Memory | 128 MB – **10,240 MB** (CPU scales with memory) |
| `/tmp` ephemeral storage | 512 MB – 10 GB |
| Deployment package | 50 MB zipped direct, 250 MB unzipped; **10 GB as a container image** |
| Default concurrency | 1,000 per region (soft limit) |
| Payload | 6 MB synchronous, 256 KB asynchronous |
| Environment variables | 4 KB total |

**Concurrency:** **reserved concurrency** guarantees (and caps) a function's share; **provisioned concurrency** keeps instances warm to eliminate **cold starts**. Cold start = the time to initialise a new execution environment; mitigations are provisioned concurrency, smaller packages, and avoiding heavy init. Anything outside the handler runs once per environment and is reused — put SDK clients and DB connections there.

**Invocation models:** **synchronous** (API Gateway, ALB — caller waits, errors return to caller), **asynchronous** (S3, SNS, EventBridge — Lambda retries **twice**, then sends to a **DLQ**/on-failure destination), and **event source mapping / poll-based** (SQS, Kinesis, DynamoDB Streams — Lambda polls and batches).

**Layers** share code/dependencies across functions. **Aliases and versions** enable weighted traffic shifting for canary deploys. **Lambda@Edge** runs at CloudFront edge locations.

## 4.3 Containers

| Service | Is |
|---|---|
| **ECR** | the private container registry (№50) |
| **ECS** | AWS-native orchestrator; **task definitions** + services |
| **EKS** | managed Kubernetes — choose when you need k8s specifically or portability |
| **Fargate** | **serverless compute for containers** — no EC2 to manage; works with ECS and EKS |
| **App Runner** | simplest: source or image → running service |

**ECS launch types:** **EC2** (you manage the instances — cheaper at scale, more control) vs **Fargate** (no instances — less operational overhead, the usual exam answer when "minimise operational overhead" appears).

ECS **task role** (permissions for your application code) vs **task execution role** (permissions for the ECS agent to pull the image and write logs) — a classic DVA distinction.

## 4.4 Other compute

- **Elastic Beanstalk** — PaaS: you give it code, it provisions EC2/ASG/ELB/etc. You still own the resources. Deployment policies matter for DVA (§9.3).
- **Batch** — managed batch computing, great with Spot.
- **Lightsail** — simple fixed-price VPS; the "simplest possible" answer.

---

# Part 5 — Storage

## 5.1 S3 — the most examined service in AWS

Object storage: buckets (globally unique names), objects (key + value + metadata), **max object size 5 TB**, single-PUT max 5 GB, **multipart upload recommended above ~100 MB** (and required above 5 GB). **11 nines of durability.** **Strong read-after-write consistency** for all operations (since Dec 2020 — old material saying "eventually consistent for overwrites" is outdated).

**Storage classes — a guaranteed cost question:**

| Class | Use for | Retrieval |
|---|---|---|
| **Standard** | frequently accessed | instant |
| **Intelligent-Tiering** | **unknown/changing access patterns** | instant, auto-tiers, small monitoring fee |
| **Standard-IA** | infrequent, needs instant access | instant, retrieval fee |
| **One Zone-IA** | infrequent + **recreatable** (single AZ) | instant, cheaper, less durable |
| **Glacier Instant Retrieval** | archive, needs ms access | instant |
| **Glacier Flexible Retrieval** | archive | minutes–hours |
| **Glacier Deep Archive** | long-term compliance, cheapest | **12+ hours** |

**Lifecycle policies** transition objects between classes and expire them on a schedule — the standard answer to "reduce storage costs over time." Note IA classes have a **30-day minimum**, Glacier Deep Archive 180 days.

**Key features:** **versioning** (protects against overwrite/delete; delete adds a marker), **MFA Delete**, **replication** (CRR cross-region / SRR same-region — requires versioning), **Transfer Acceleration** (upload via edge locations — the answer for slow long-distance uploads), **Presigned URLs** (time-limited access without credentials — very DVA), **Event Notifications** (→ Lambda/SQS/SNS/EventBridge), **S3 Select** (SQL over a single object, retrieve only what you need), **Requester Pays**, **Object Lock** (WORM compliance), **Static website hosting**.

**Encryption** (know the four):

| Type | Key managed by | Note |
|---|---|---|
| **SSE-S3** | AWS (AES-256) | default, simplest |
| **SSE-KMS** | AWS KMS, your CMK | audit trail via CloudTrail, key policies; **KMS request quotas can throttle** |
| **DSSE-KMS** | KMS, double encryption | compliance |
| **SSE-C** | **you** supply the key per request | AWS never stores it; HTTPS required |
| Client-side | you, before upload | AWS never sees plaintext |

**Access control:** bucket policies (resource-based, the usual), IAM policies, ACLs (legacy — disable them), **Block Public Access** (on by default; the answer to "prevent accidental exposure"), and **VPC endpoints** for private access.

## 5.2 EBS, EFS, FSx, Instance Store

| | **EBS** | **EFS** | **FSx** | **Instance Store** |
|---|---|---|---|---|
| Type | block, **one AZ** | **NFS file, multi-AZ** | managed file (Windows/Lustre) | **ephemeral** local disk |
| Attach | one instance (or multi-attach io1/io2) | **many instances at once** | many | one, physically attached |
| Persistence | survives instance stop | persistent | persistent | **lost on stop/terminate** |
| Use for | boot volumes, databases | shared content across instances | Windows SMB, HPC | scratch, cache, temp |

**EBS volume types:** **gp3** (default SSD — baseline 3,000 IOPS/125 MB/s, IOPS and throughput independently configurable, cheaper than gp2), **gp2** (older, IOPS tied to size), **io1/io2** (provisioned IOPS — critical databases; io2 Block Express for the highest), **st1** (throughput HDD — big sequential, logs/data warehouse), **sc1** (cold HDD — infrequent). Snapshots are incremental and stored in S3; you can copy them cross-region for DR.

**Other:** **Storage Gateway** (hybrid on-prem↔AWS: File/Volume/Tape), **AWS Backup** (centralised backup policy), **DataSync** (bulk transfer on-prem→AWS), **Snowball/Snowmobile** (physical data transfer — the answer when "petabytes" and "limited bandwidth" appear together).

---

# Part 6 — Databases

## 6.1 RDS

Managed relational: Postgres, MySQL, MariaDB, Oracle, SQL Server, and **Aurora**. AWS handles patching, backups, and failover; you don't get OS access.

**The distinction the exam loves:**

| | **Multi-AZ** | **Read Replicas** |
|---|---|---|
| Purpose | **high availability / DR** | **scale reads** |
| Replication | **synchronous** | **asynchronous** (lag) |
| Standby serves traffic? | **no** (it's a standby) | **yes** (read-only) |
| Failover | automatic, DNS repoints | manual promotion |
| Cross-region? | no (Multi-AZ is within a region) | **yes** |

"Improve read performance" → read replicas. "Survive an AZ failure / minimise downtime" → Multi-AZ. Both, usually.

**Backups:** automated backups with a retention period (1–35 days) enabling **point-in-time recovery**; manual snapshots persist until deleted. Encryption at rest via KMS must generally be enabled at creation (you can encrypt by restoring a snapshot to a new encrypted instance).

**Aurora** — AWS's cloud-native MySQL/Postgres-compatible engine. Exam facts: **6 copies of data across 3 AZs**, self-healing, storage auto-grows to 128 TB, **up to 15 low-latency read replicas**, failover in ~30s, **Aurora Serverless v2** scales capacity automatically (the answer for unpredictable/intermittent workloads), **Global Database** for cross-region (<1s replication), and **Aurora Replicas can be auto-scaled**.

## 6.2 DynamoDB — critical for DVA, common in SAA

Managed NoSQL key-value/document store. Single-digit-millisecond latency at any scale.

**Data model:** table → items → attributes. **Primary key** is either a **partition key** alone, or a **partition key + sort key** (composite). The partition key determines physical placement — a poor choice creates a **hot partition** (№31 §5.3). **Max item size 400 KB.**

**Indexes — a guaranteed question:**

| | **LSI** (Local Secondary Index) | **GSI** (Global Secondary Index) |
|---|---|---|
| Partition key | **same** as the table | **any attribute** |
| Sort key | different | any |
| When created | **only at table creation** | **any time** |
| Capacity | shares the table's | **its own** |
| Consistency | supports strongly consistent reads | **eventually consistent only** |
| Limit | 10 GB per partition key | — |

**Capacity modes:** **On-demand** (pay per request — unpredictable traffic, no capacity planning) vs **Provisioned** (cheaper for steady traffic; RCU/WCU, with auto-scaling). RCU: 1 strongly-consistent read/s of 4 KB (or 2 eventually-consistent). WCU: 1 write/s of 1 KB. `ProvisionedThroughputExceededException` → throttling → use exponential backoff, better key distribution, or on-demand.

**Features:** **DynamoDB Streams** (change data capture → Lambda triggers), **DAX** (in-memory cache, microsecond reads, DynamoDB-specific), **TTL** (auto-expire items — the cheap way to purge old data), **transactions** (ACID across items), **global tables** (multi-region, multi-active), **PartiQL** (SQL-like queries), and **conditional writes** for optimistic concurrency (the DynamoDB analogue of `@Version`, №21 §2.7).

**Query vs Scan:** **Query** uses the partition key and is efficient; **Scan** reads the whole table and is the wrong answer nearly always. If you see "improve performance of a Scan," the answer is usually a redesigned key or a GSI.

## 6.3 The rest

| Service | Is | Reach for it when |
|---|---|---|
| **ElastiCache** | managed Redis / Memcached | caching, sessions, leaderboards; Redis = persistence, replication, pub/sub; Memcached = simple, multi-threaded |
| **Redshift** | petabyte data warehouse (OLAP, columnar) | analytics/BI over large historical data |
| **Athena** | serverless SQL **directly over S3** | ad-hoc queries on logs/data lake, pay per TB scanned |
| **Neptune** | graph database | relationships, social, fraud |
| **DocumentDB** | MongoDB-compatible | document workloads |
| **Timestream** | time-series | IoT, metrics |
| **QLDB** | immutable ledger | verifiable audit history |
| **Keyspaces** | managed Cassandra | wide-column at scale |
| **DMS** | Database Migration Service | migrating (with **SCT** for engine conversion) |

---

# Part 7 — Networking and content delivery

*Protocol-level foundations — TCP/IP, DNS, TLS, load balancing — are in №51. This is the AWS surface.*

## 7.1 VPC

Your isolated virtual network. The components, and what each is for:

| Component | Purpose |
|---|---|
| **VPC** | the network, defined by a CIDR block (e.g. `10.0.0.0/16`) |
| **Subnet** | a CIDR slice **within one AZ**; public or private |
| **Internet Gateway (IGW)** | allows internet access; a subnet is "public" if its route table points `0.0.0.0/0` at an IGW |
| **NAT Gateway** | lets **private** subnets reach the internet outbound only; managed, AZ-scoped (deploy one per AZ for HA) |
| **Route table** | where traffic goes |
| **Security Group** | **stateful** firewall at the **instance/ENI** level; **allow rules only** |
| **NACL** | **stateless** firewall at the **subnet** level; allow **and deny** rules, evaluated in number order |
| **VPC Endpoint** | private access to AWS services without traversing the internet |
| **VPC Peering** | connects two VPCs; **not transitive**, no overlapping CIDRs |
| **Transit Gateway** | hub-and-spoke connecting many VPCs/on-prem — the scalable answer |
| **Flow Logs** | capture IP traffic metadata for troubleshooting/audit |

**Security Group vs NACL is a near-certain exam question:**

| | Security Group | NACL |
|---|---|---|
| Level | instance/ENI | subnet |
| State | **stateful** (return traffic auto-allowed) | **stateless** (must allow both directions) |
| Rules | allow only | allow **and deny** |
| Evaluation | all rules evaluated | **in rule-number order**, first match wins |
| Default | denies all inbound, allows all outbound | default NACL allows all |

Security groups can **reference other security groups** — "allow 5432 from the app tier's SG" — which is the clean, exam-preferred way to express tiering.

**Endpoints:** **Gateway endpoints** (S3 and DynamoDB only, free, via route table) vs **Interface endpoints / PrivateLink** (an ENI in your subnet, most other services, hourly + data cost). "Access S3 without going over the internet" → **gateway endpoint**.

**Reserved IPs:** AWS reserves **5 addresses per subnet** (network, VPC router, DNS, future, broadcast) — a classic detail question.

**Hybrid connectivity:** **Site-to-Site VPN** (over the internet, encrypted, quick to set up) vs **Direct Connect** (dedicated private line — consistent latency, higher bandwidth, weeks to provision, more expensive). "Consistent low-latency dedicated connection" → Direct Connect; "quick, encrypted, cheap" → VPN. DX with a VPN backup is the resilient answer.

## 7.2 Elastic Load Balancing

| Type | Layer | Use for |
|---|---|---|
| **ALB** | 7 (HTTP/HTTPS) | web apps; **path/host-based routing**, WebSockets, containers, Lambda targets |
| **NLB** | 4 (TCP/UDP/TLS) | extreme performance, **static IP**, non-HTTP protocols |
| **GWLB** | 3 | inserting virtual appliances (firewalls, IDS) |
| CLB | legacy | avoid |

Features: health checks, cross-zone load balancing, sticky sessions (for stateful apps — but prefer stateless, №31 §5.1), TLS termination with **ACM** certificates (free, auto-renewing), and connection draining/deregistration delay.

## 7.3 Route 53

Managed DNS (№51 §3) plus health checks and traffic policy. Routing policies — know each:

| Policy | Routes by |
|---|---|
| **Simple** | one record, no logic |
| **Weighted** | percentage split — **canary/blue-green testing** |
| **Latency** | lowest latency region for the user |
| **Failover** | primary/secondary with health checks — **active-passive DR** |
| **Geolocation** | user's location — compliance, localisation |
| **Geoproximity** | location with a bias shift |
| **Multivalue answer** | several healthy IPs, client-side balancing |

**Alias records** (AWS-specific) point at AWS resources (ALB, CloudFront, S3 site) at no charge and work at the **zone apex**, where CNAME can't. That's a favourite question.

## 7.4 CloudFront and edge

**CloudFront** — the CDN. Caches at edge locations, reducing latency and origin load. Exam points: **origins** (S3, ALB, any HTTP), **OAC/OAI** (restrict S3 so it's *only* reachable via CloudFront), **signed URLs** (single file) vs **signed cookies** (multiple files) for private content, **cache behaviours** and TTLs, **invalidations**, field-level encryption, and integration with **WAF** and **Shield**.

**Global Accelerator** — routes over the AWS backbone using **anycast static IPs**; improves TCP/UDP performance for non-cacheable and non-HTTP traffic. CloudFront caches content; Global Accelerator accelerates connections — that's the distinction they test.

---

# Part 8 — Application integration and messaging

This category is the heart of "decoupled architecture" questions (№31 §7).

## 8.1 SQS

Fully managed message queue — a buffer between producers and consumers.

| | **Standard** | **FIFO** |
|---|---|---|
| Throughput | nearly unlimited | 300 msg/s (3,000 with batching) |
| Ordering | best-effort | **strictly ordered** |
| Delivery | **at-least-once** (duplicates possible) | **exactly-once processing** (dedup within 5 min) |
| Naming | any | must end `.fifo` |

Key parameters: **visibility timeout** (default 30s, max 12h — how long a consumed message is hidden; too short causes duplicate processing), **message retention** (default 4 days, max 14), **max message size 256 KB** (use the Extended Client Library with S3 for larger), **long polling** (`ReceiveMessageWaitTimeSeconds` up to 20s — reduces empty responses and cost; prefer it over short polling), **DLQ** (after `maxReceiveCount` failures — the answer for "handle poison messages"), and **delay queues**.

## 8.2 SNS

Pub/sub: one message → many subscribers (Lambda, SQS, HTTP, email, SMS, mobile push). **Fan-out pattern:** SNS topic → multiple SQS queues, so each consumer gets its own durable copy. That's the canonical decoupling answer when several systems must react to one event. Supports message filtering and FIFO topics.

## 8.3 EventBridge

The event bus — SNS's more capable sibling. **Schema registry**, **content-based routing rules**, **SaaS partner sources**, **scheduled events** (cron — the modern replacement for CloudWatch Events), and **archive/replay**. Choose EventBridge when you need routing logic, many AWS-service event sources, or event schemas; choose SNS for simple high-throughput fan-out.

## 8.4 Kinesis

Real-time streaming (the Kafka analogue, №31 §7.2):

| Service | Is |
|---|---|
| **Data Streams** | shards, ordered, replayable; **retention 24h default, up to 365 days**; 1 MB/s in, 2 MB/s out per shard |
| **Data Firehose** | fully managed delivery to S3/Redshift/OpenSearch; near-real-time, no shard management |
| **Data Analytics** | SQL/Flink over streams |
| **Video Streams** | media ingestion |

**SQS vs Kinesis** is a favourite: SQS = a work queue, message deleted after processing, no replay. Kinesis = a stream, multiple independent consumers, ordered per shard, replayable. "Multiple applications must process the same real-time data" → Kinesis.

## 8.5 Step Functions and API Gateway

**Step Functions** — serverless workflow orchestration as a state machine (sequential, parallel, choice, retry, catch). **Standard** (up to 1 year, exactly-once, auditable) vs **Express** (up to 5 min, high volume, at-least-once). The answer for "coordinate multiple Lambdas with error handling and retries" — and the AWS-native way to implement a saga (№31 §8.2).

**API Gateway** — the managed front door for APIs:

| Type | Note |
|---|---|
| **REST API** | full features: request/response transformation, API keys, usage plans, caching, WAF |
| **HTTP API** | cheaper, faster, fewer features — prefer unless you need REST-only features |
| **WebSocket API** | bidirectional (№51 §5.5) |

Exam points: **stages** and stage variables, **usage plans + API keys** (throttling per client), **caching** (reduce backend calls), **authorisers** (Lambda authoriser, Cognito, IAM), **CORS** (№51 §9.5), **request validation**, and integration types (Lambda proxy vs non-proxy — proxy passes the whole request through, which is the common choice).

**Amazon MQ** — managed ActiveMQ/RabbitMQ; the answer only when migrating an existing app that needs **standard protocols** (AMQP, MQTT, JMS) rather than SQS's API.

---

# Part 9 — Deployment and developer tools

DVA-heavy (domain 3 is 24%), and CloudFormation appears in SAA too.

## 9.1 Infrastructure as Code

- **CloudFormation** — declarative YAML/JSON templates. Concepts: **stacks**, **change sets** (preview before applying — the safe practice), **nested stacks**, **StackSets** (deploy across accounts/regions), **drift detection**, **parameters/mappings/conditions/outputs**, **intrinsic functions** (`!Ref`, `!GetAtt`, `!Sub`, `!Join`), **DeletionPolicy** (`Retain` to keep a database when the stack goes), and rollback on failure.
- **SAM** — a CloudFormation extension for serverless: simpler syntax for Lambda/API Gateway/DynamoDB, plus `sam local` for local testing. Very DVA.
- **CDK** — define infrastructure in a real language (TypeScript/Python/Java), synthesised to CloudFormation.
- **Terraform** — third-party, multi-cloud (№55, planned).

## 9.2 CI/CD

| Service | Does |
|---|---|
| **CodeCommit** | managed Git repos |
| **CodeBuild** | build and test; configured by **`buildspec.yml`** |
| **CodeDeploy** | deploys to EC2/on-prem/Lambda/ECS; configured by **`appspec.yml`** |
| **CodePipeline** | orchestrates the stages |
| **CodeArtifact** | artifact repository |
| **CodeGuru** | automated code review and profiling |

Know which file belongs to which service — `buildspec.yml` → CodeBuild; `appspec.yml` → CodeDeploy. That's a direct DVA question.

## 9.3 Deployment strategies

**Elastic Beanstalk deployment policies** (a favourite DVA table):

| Policy | Downtime | Extra cost | Rollback |
|---|---|---|---|
| **All at once** | **yes** | none | redeploy |
| **Rolling** | no (reduced capacity) | none | manual |
| **Rolling with additional batch** | no (full capacity) | small | manual |
| **Immutable** | no | double temporarily | **easy — terminate new ASG** |
| **Blue/Green** | no | double | **easiest — swap URLs back** |
| **Traffic splitting** | no | some | automatic on failure |

**CodeDeploy** modes: **in-place** (update existing instances) vs **blue/green** (new fleet, then shift). Lambda/ECS deployment configs: `Canary10Percent5Minutes`, `Linear10PercentEvery1Minute`, `AllAtOnce`.

## 9.4 The SDK and developer specifics (DVA)

- **Credentials chain** — the SDK looks in order: environment variables → Java system properties → shared credentials file → container credentials → **instance profile / IAM role**. Roles at the bottom, and the correct production answer.
- **Retries and backoff** — the SDK retries throttling and 5xx errors with **exponential backoff and jitter** automatically (№31 §10.2). Exam answer for `ThrottlingException`: exponential backoff.
- **Pagination** — list operations are paginated; use the paginators or follow the continuation token.
- **Exponential backoff, idempotency tokens, and conditional writes** appear constantly in DVA scenarios.

---

# Part 10 — Observability

| Service | Answers |
|---|---|
| **CloudWatch Metrics** | "how is it behaving?" — namespaces, dimensions, **custom metrics** (`PutMetricData`), high-resolution (1s) |
| **CloudWatch Logs** | "what happened?" — log groups/streams, retention, **Logs Insights** for querying, metric filters → alarms |
| **CloudWatch Alarms** | thresholds → SNS/Auto Scaling actions; states OK/ALARM/INSUFFICIENT_DATA |
| **CloudWatch Agent** | needed for **memory and disk** metrics from EC2 (not available by default — classic question) |
| **EventBridge (Events)** | react to state changes; scheduled jobs |
| **X-Ray** | **distributed tracing** — segments, subsegments, annotations (indexed, filterable) vs metadata (not indexed), the service map. The answer to "find the bottleneck across services" |
| **CloudTrail** | **who did what** — API call audit log; management vs data events; the answer to "who deleted the bucket?" |
| **AWS Config** | **resource configuration compliance** over time; "was this ever non-compliant?" |
| **Trusted Advisor** | recommendations across cost, security, fault tolerance, performance, limits |

The three-way distinction the exam tests: **CloudWatch** = performance/operational telemetry. **CloudTrail** = API audit. **Config** = configuration compliance and history.

---

# Part 11 — Security services

| Service | Does |
|---|---|
| **KMS** | managed encryption keys. **Envelope encryption** (a data key encrypts data; KMS encrypts the data key). Customer-managed keys support **automatic annual rotation** and key policies. Regional; `GenerateDataKey` is the developer-facing call. |
| **CloudHSM** | dedicated hardware module — you control the keys entirely (compliance) |
| **Secrets Manager** | secrets with **automatic rotation** (native for RDS); costs per secret |
| **SSM Parameter Store** | config and secrets; **free standard tier**, SecureString via KMS, no built-in rotation |
| **Cognito** | **User Pools** = user directory/authentication (sign-up, sign-in, JWT tokens, MFA, social/SAML federation). **Identity Pools** = exchange an identity for **temporary AWS credentials**. |
| **WAF** | layer-7 filtering on CloudFront/ALB/API Gateway — SQL injection, XSS, rate rules, geo-blocking |
| **Shield** | DDoS protection: Standard free; Advanced paid with cost protection and a response team |
| **GuardDuty** | intelligent threat detection from logs (ML-based) |
| **Inspector** | automated vulnerability scanning of EC2/ECR/Lambda |
| **Macie** | discovers and classifies sensitive data (PII) in S3 |
| **Security Hub** | aggregates findings across security services |
| **Certificate Manager (ACM)** | free public TLS certs, auto-renewed, for ALB/CloudFront/API Gateway |

**Secrets Manager vs Parameter Store** is a guaranteed question: need **automatic rotation** or cross-account secret sharing → Secrets Manager; need **free, simple config storage** → Parameter Store.

**Cognito User Pool vs Identity Pool** is the other: authenticate users into *your app* → User Pool; give them *AWS credentials* to hit S3/DynamoDB directly → Identity Pool.

---

# Part 12 — Cost optimisation

The SAA's 20% domain, and a lens on every other question.

- **Compute:** right-size; Savings Plans/RIs for steady load; **Spot** for fault-tolerant; auto-scaling to match demand; **serverless** to pay only for use; shut down non-production out of hours.
- **Storage:** lifecycle policies to colder S3 classes; **Intelligent-Tiering** for unknown patterns; delete incomplete multipart uploads and old snapshots; gp3 over gp2 (cheaper and faster).
- **Database:** right-size; Aurora Serverless v2 for intermittent load; read replicas only where needed; reserved instances for steady databases.
- **Network — the sleeper cost:** **data transfer OUT to the internet costs money**; inbound is generally free; **cross-AZ traffic costs**, same-AZ is free; **NAT Gateway** charges per hour *and* per GB (a classic surprise bill); use **VPC endpoints** to keep S3/DynamoDB traffic off NAT and the internet; use **CloudFront** to cut origin egress.
- **Tooling:** **Cost Explorer** (analyse and forecast), **Budgets** (alerts on thresholds), **Cost and Usage Report** (granular), **cost allocation tags**, **Compute Optimizer** (right-sizing recommendations), Organizations for consolidated billing (volume discounts pooled).

> **The tell — cost:** when a question says "most cost-effective," look for: serverless over always-on; the coldest storage class that meets the access requirement; Spot where interruption is tolerable; and anything that removes a NAT Gateway or cross-AZ/internet data transfer. When it says "minimise operational overhead," look for the *managed* option (Fargate over EC2, Aurora Serverless over self-managed, SQS over self-hosted brokers).

---

# Part 13 — Decision frameworks: which service, when

**This is the actual exam skill.** Each row is a decision the exams test repeatedly.

## 13.1 Compute

| Requirement | Choose |
|---|---|
| Full OS control, legacy software, licensing | **EC2** |
| Event-driven, short tasks (<15 min), no servers | **Lambda** |
| Containers, no infrastructure management | **ECS + Fargate** |
| Containers, need Kubernetes / portability | **EKS** |
| Just deploy my app, don't make me architect it | **Elastic Beanstalk** |
| Batch jobs, cost-sensitive, interruption-tolerant | **Batch + Spot** |
| Long-running (>15 min) but serverless-ish | **Fargate**, not Lambda |

## 13.2 Storage

| Requirement | Choose |
|---|---|
| Objects, web-accessible, cheap, durable | **S3** |
| Block storage for one instance / a database | **EBS** |
| Shared file system across many instances (Linux) | **EFS** |
| Shared Windows/SMB file share | **FSx for Windows** |
| Temporary scratch, maximum IOPS, disposable | **Instance Store** |
| Archive, retrieval in hours, cheapest | **Glacier Deep Archive** |
| Unknown/changing access pattern | **S3 Intelligent-Tiering** |
| Petabytes to move, limited bandwidth | **Snowball** |

## 13.3 Databases

| Requirement | Choose |
|---|---|
| Relational, transactions, joins, familiar SQL | **RDS** (or **Aurora** for performance/HA) |
| Key-value, massive scale, single-digit-ms, serverless | **DynamoDB** |
| Sub-millisecond caching in front of a database | **ElastiCache** (or **DAX** for DynamoDB) |
| Analytics/BI over huge historical datasets | **Redshift** |
| Ad-hoc SQL over data already in S3 | **Athena** |
| Relationship traversal (social, fraud, recommendations) | **Neptune** |
| Intermittent/unpredictable relational load | **Aurora Serverless v2** |

## 13.4 Messaging and integration

| Requirement | Choose |
|---|---|
| Decouple producer and consumer; work queue | **SQS** |
| One event → many subscribers (fan-out) | **SNS** (→ SQS per consumer) |
| Event routing with filtering, AWS-service sources, schedules | **EventBridge** |
| Real-time streaming, multiple consumers, replay | **Kinesis Data Streams** |
| Stream straight into S3/Redshift with no management | **Data Firehose** |
| Orchestrate a multi-step workflow with retries | **Step Functions** |
| Strict ordering and no duplicates | **SQS FIFO** |
| Migrating an app that needs AMQP/MQTT/JMS | **Amazon MQ** |

## 13.5 Security and identity

| Requirement | Choose |
|---|---|
| An AWS service needs permissions | **IAM role** (never access keys) |
| App user sign-up/sign-in | **Cognito User Pool** |
| Let app users call AWS services directly | **Cognito Identity Pool** |
| Secret with automatic rotation | **Secrets Manager** |
| Free config/parameter storage | **SSM Parameter Store** |
| Encryption keys with audit trail | **KMS** |
| Block SQL injection / XSS at the edge | **WAF** |
| Detect compromised instances/credentials | **GuardDuty** |
| Find PII sitting in S3 | **Macie** |
| Who made this API call? | **CloudTrail** |

## 13.6 Resilience patterns (SAA's bread and butter)

| Requirement | Answer |
|---|---|
| Survive an AZ failure | deploy across **≥2 AZs**; RDS **Multi-AZ**; ASG spanning AZs |
| Survive a region failure | cross-region replication, Route 53 **failover**, Aurora Global / DynamoDB global tables |
| Handle traffic spikes | **ASG** + **SQS** buffering + caching |
| Avoid a single point of failure | ELB, multiple instances, managed multi-AZ services |
| Decouple failing components | **SQS** between tiers, **DLQ** for poison messages |
| Static content resilience + speed | **S3 + CloudFront** |

---

# Part 14 — Exam technique

## 14.1 Read the qualifier first

Nearly every question ends with a qualifier that *is* the question:

| Qualifier | Optimise for |
|---|---|
| "MOST cost-effective" | cheapest that still meets the stated requirement |
| "LEAST operational overhead" | **most managed / serverless** option |
| "MOST secure" | least privilege, encryption, private networking |
| "HIGHEST availability" | multi-AZ, multi-region, no SPOF |
| "MINIMAL downtime" | blue/green, Multi-AZ failover, immutable deploys |
| "MOST performant" | caching, right instance type, closest edge |
| "REAL-TIME" | Kinesis/streaming, not batch |

Two options are often both *correct*; only one is correct **for the qualifier**.

## 14.2 Common traps

- **Access keys on an instance** — nearly always wrong; use an IAM role.
- **"Scan the DynamoDB table"** — usually wrong; redesign keys or use a GSI/Query.
- **Anything requiring SSH/manual patching** when the question says "reduce operational overhead."
- **Multi-AZ ≠ read scaling** and **read replicas ≠ high availability** — the exam deliberately mixes these.
- **NAT Gateway in one AZ** — a single point of failure; one per AZ for HA.
- **Security groups can't deny** — if a question needs an explicit deny, it's a **NACL**.
- **Overly broad IAM** (`"Action": "*"`) — never the "most secure" answer.
- **Storing session state on the instance** — breaks scaling; use ElastiCache/DynamoDB (№31 §5.1).
- **Lambda for >15 minutes** — impossible; the answer is Fargate/Batch/Step Functions.
- **CloudWatch memory metrics** — require the agent; not there by default.

## 14.3 Technique

Eliminate first — usually two options are obviously wrong on a hard constraint (a service that can't do the thing, a timeout that's impossible), leaving a genuine two-way choice you resolve on the qualifier. Watch for **absolute requirements** ("must be encrypted at rest," "cannot traverse the internet," "must be ordered") — these single-handedly eliminate options. Flag and move on rather than burning time; **130 minutes / 65 questions ≈ 2 minutes each**, and the review pass is where flagged questions get resolved. There's no penalty for guessing, so **never leave a question blank**.

---

# Part 15 — A study plan

**Timeline:** SAA-C03 in ~6–8 weeks of steady study for someone with your background (you already have containers, networking, databases, and distributed systems from №50, №51, №20, №31 — that's a real head start). DVA-C02 in ~3–4 weeks after SAA, given the overlap.

**The approach that works:**

1. **Learn the services** — this document plus a video course (Stephane Maarek and Adrian Cantrill are the two standard recommendations) for the areas you're weakest in.
2. **Get hands-on.** Build something small in the free tier: a VPC with public/private subnets, an ALB in front of two instances, an RDS in the private subnet, a Lambda triggered by S3, a DynamoDB table with a GSI. **The concepts stick far better once you've clicked them together**, and Practiq's own deployment is a legitimate study project — building it *is* revision.
3. **Practice exams are non-negotiable.** They teach the question *style*, which is half the exam. Tutorials Dojo is the most-recommended set. Aim for consistent **80%+** before booking, and — this is the important part — **read the explanation for every question you get wrong *and* every one you guessed right.**
4. **Drill the decision tables** (Part 13). Those *are* the exam.
5. **Book the exam** once you're consistently at 80%. A booked date is the forcing function.

**Where your existing library helps:** №51 covers VPC/DNS/TLS/load-balancing concepts properly, so the AWS networking domain is mostly learning AWS's naming for things you know. №31 covers decoupling, queues, caching, and resilience — the conceptual backbone of the SAA resilience domain. №50 covers containers, so ECS/Fargate is mostly AWS specifics. №20 covers relational databases, so RDS/Aurora is largely feature learning.

> **The tell — passing:** the exams reward *judgment between services*, not recall of service descriptions. If you can look at a scenario and say "this needs decoupling because the consumer is slower than the producer, so SQS; and it needs to survive an AZ failure, so Multi-AZ RDS; and the qualifier says cost, so Spot for the workers" — you'll pass comfortably. Study Part 13 hardest.

---

# How to expand this

- *Related library docs:* №51 (networking foundations), №31 (distributed systems), №50 (containers), №20 (databases), №43 (system design — the same skill in interview form), №55 IaC/Terraform (planned), №57 Observability (planned).
- *Candidates for deeper treatment:* a **VPC deep-dive** with worked subnet/route-table/CIDR design; **DynamoDB data modelling** end to end (single-table design, access patterns, GSI strategy) — the hardest DVA topic; **a Practiq reference architecture** built as an exam-style design with justifications; **service-by-service flashcard sets** for the drilling phase; a **CloudFormation/SAM worked template** for the deployment domain.

*Exam versions (SAA-C03, DVA-C02), format and pricing verified July 2026. Service capabilities and quotas change — AWS's own exam guides and documentation are the final authority, and quotas quoted here should be re-checked before you rely on them in a design.*
