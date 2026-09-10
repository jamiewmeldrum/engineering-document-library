# The Tool Cards №90

*A catalogue of the tools a backend engineer meets, one card each, grouped by the **problem they solve**. The organising question is not "what is this?" but **"when would I reach for this instead of the obvious alternative?"** — because in almost every category there's a default answer and a set of reasons to deviate.*

**Card format:** *What it is · Reach for it when · Not when · AWS equivalent.* Each category opens with the problem it addresses and closes with a default. ~110 tools.

**Two caveats, stated up front.** First, **the default is usually right** — most categories have a boring answer (Postgres, Redis, Docker, GitHub Actions, Terraform) that serves the overwhelming majority of systems, and the interesting entries below exist for specific pressures you should be able to name before adopting them. Second, **this is the fastest-drifting document in the library**: tools rise, stall and get acquired. Mechanisms and problem categories are stable; specific currency should be checked before you commit.

Contents: **1** Relational databases · **2** NoSQL & specialised stores · **3** Caching · **4** Messaging & streaming · **5** Search · **6** Containers & orchestration · **7** IaC & configuration · **8** CI/CD · **9** Web servers, proxies & gateways · **10** Observability · **11** Profiling & load testing · **12** Build tools & package management · **13** Version control & collaboration · **14** Secrets, identity & security · **15** Data & analytics · **16** Java & JVM ecosystem · **17** Local development

---

# 1 · Relational databases

*The problem: durable, consistent, queryable structured data with transactions and constraints. Start here for essentially everything (№20 §7).*

**PostgreSQL** — open-source object-relational DBMS; the modern default. **Reach for it when:** you need a database. Rich types (JSONB, arrays, ranges), extensions (PostGIS, pgvector), strong SQL compliance, permissive licence. **Not when:** you need a specific commercial feature or an embedded engine. **AWS:** RDS for PostgreSQL, Aurora PostgreSQL.

**MySQL / MariaDB** — the other dominant open-source RDBMS. **Reach for it when:** existing expertise, or an ecosystem built around it (WordPress). InnoDB stores rows physically in PK order — the clustered-index model Postgres doesn't have (№20 §4.4). **Not when:** you want the richest SQL and type system. **AWS:** RDS, Aurora MySQL.

**SQLite** — embedded, serverless, a single file, in-process. **Reach for it when:** local/on-device storage, CLI tools, tests, or genuinely low-concurrency single-writer workloads. **Not when:** multiple concurrent writers or a network service. **AWS:** n/a (it's in-process).

**SQL Server** — Microsoft's enterprise RDBMS, T-SQL. **Reach for it when:** a .NET/Windows estate or existing licences. **Not when:** cost-sensitive greenfield. **AWS:** RDS for SQL Server.

**Oracle Database** — the enterprise incumbent; extremely capable, extremely expensive. **Reach for it when:** you've inherited it. **Not when:** you have a choice. **AWS:** RDS for Oracle.

**CockroachDB / YugabyteDB** — distributed SQL, horizontally scalable, **Postgres-wire-compatible**. **Reach for it when:** you genuinely need multi-region writes or scale beyond one node while keeping SQL and transactions. **Not when:** one Postgres would do — which is nearly always. **AWS:** Aurora (limited overlap), Aurora DSQL.

**H2 / HSQLDB** — embedded Java databases. **Reach for it when:** a fast throwaway test database and you accept dialect drift. **Not when:** you're testing SQL behaviour — use Testcontainers with the real engine (№44 §7.3).

> **Default: PostgreSQL.** Deviate only for an inherited estate, an embedded requirement, or measured scale beyond a single node.

---

# 2 · NoSQL & specialised stores

*The problem: data shapes that relational tables handle awkwardly, or scale characteristics a single node can't reach. Usually a **complement** to Postgres, not a replacement (№31).*

**DynamoDB** — managed key-value/document store; single-digit-ms at any scale. **Reach for it when:** known key-based access patterns, massive or spiky scale, serverless architectures. **Not when:** you need ad-hoc queries or joins (№91 §3). **AWS:** native.

**MongoDB** — document database, flexible schema. **Reach for it when:** genuinely heterogeneous or deeply nested documents, rapid schema evolution. **Not when:** Postgres JSONB would do — which it often would. **AWS:** DocumentDB (compatible), or Atlas on AWS.

**Cassandra / ScyllaDB** — wide-column, extreme write throughput, tunable consistency, no single point of failure. **Reach for it when:** enormous write volumes, time-series or event data, multi-datacentre. **Not when:** below "enormous" — the modelling burden is real. **AWS:** Keyspaces.

**Neo4j** — graph database (Cypher). **Reach for it when:** deep variable-length traversals are the *primary* workload — fraud rings, recommendations. **Not when:** your data is merely graph-*shaped*; junction tables handle that fine (№20 §4.1). **AWS:** Neptune.

**InfluxDB / TimescaleDB / Prometheus** — time-series. **Reach for it when:** high-volume timestamped metrics with time-window queries. TimescaleDB is a Postgres extension, so it's the lowest-friction option. **AWS:** Timestream.

**pgvector / Pinecone / Weaviate / Qdrant** — vector similarity search over embeddings. **Reach for it when:** semantic search, RAG, recommendations. **Start with pgvector** — it's a Postgres extension, so no new datastore. **Not when:** keyword search suffices. **AWS:** OpenSearch vector engine, Aurora/RDS pgvector, Bedrock Knowledge Bases.

**etcd / ZooKeeper / Consul** — distributed coordination and configuration; consensus-backed (№31 §6). **Reach for it when:** leader election, service discovery, distributed locks. **Not when:** you can avoid distributed coordination entirely — usually you can. **AWS:** partly Cloud Map, partly managed service internals.

> **Default: Postgres, plus one specialised store only when you can name what Postgres does badly.**

---

# 3 · Caching

*The problem: repeated expensive reads. The highest-leverage performance work in most systems (№31 §9).*

**Redis** — in-memory data structure store: strings, hashes, sorted sets, streams, pub/sub, Lua, optional persistence. **Reach for it when:** caching, sessions, rate limiting, leaderboards, simple queues, distributed locks. The default. **Not when:** your working set exceeds memory economics. **AWS:** ElastiCache for Redis / Valkey; **MemoryDB** for durable Redis-as-a-database.

**Valkey** — the Linux Foundation fork of Redis created after Redis Ltd's 2024 licence change; drop-in compatible. **Reach for it when:** you want open-source governance. (Redis has since relicensed again — check current terms if licensing matters to you.) **AWS:** ElastiCache for Valkey.

**Memcached** — pure multi-threaded cache, no persistence or replication. **Reach for it when:** simple large-scale key-value caching and nothing more. **Not when:** you'd benefit from Redis's data structures — which is most of the time. **AWS:** ElastiCache for Memcached.

**Caffeine** — in-process JVM cache, excellent eviction. **Reach for it when:** per-instance caching of small hot data. **Not when:** the cache must be shared or consistent across instances (№31 §5.1). **AWS:** n/a.

**Hibernate second-level cache** — ORM-level entity caching (№21 §7). **Reach for it when:** read-mostly reference data with a clear eviction story. **Not when:** write-heavy — bulk updates bypass it and leave it stale.

**Varnish / CDN caching** — HTTP-layer caching in front of the application. **Reach for it when:** cacheable responses and you want to never reach the app. **AWS:** CloudFront.

> **Default: CDN at the edge, Redis for shared application caching, in-process only for immutable reference data.**

---

# 4 · Messaging & streaming

*The problem: decoupling producers from consumers, absorbing bursts, and doing slow work off the request path (№31 §7).*

**Amazon SQS** — managed queue; standard (at-least-once, high throughput) or FIFO (ordered, deduplicated). **Reach for it when:** distributing work to competing consumers on AWS. Nearly zero operational cost. **Not when:** you need replay or multiple independent consumers. **AWS:** native.

**RabbitMQ** — mature broker with rich routing (exchanges, bindings), AMQP. **Reach for it when:** complex routing topologies, or protocol requirements (AMQP/MQTT/STOMP). **Not when:** SQS would do and you're on AWS. **AWS:** Amazon MQ.

**Apache Kafka** — distributed append-only log; ordered per partition, replayable, multiple consumer groups. **Reach for it when:** event streaming, several independent consumers of the same stream, replay, event sourcing, high throughput. **Not when:** you need a simple work queue — the operational weight isn't justified. **AWS:** MSK, or Kinesis Data Streams as the native analogue.

**Kinesis Data Streams** — AWS-native sharded streaming. **Reach for it when:** Kafka semantics without running Kafka. **Not when:** you need Kafka's ecosystem. **AWS:** native.

**Amazon SNS** — pub/sub fan-out to queues, functions, HTTP, email, SMS. **Reach for it when:** one event, many independent consumers (SNS → several SQS queues is the canonical pattern). **AWS:** native.

**Amazon EventBridge** — event bus with content-based routing, schema registry, SaaS sources, scheduling, archive and replay. **Reach for it when:** routing logic, AWS-service events, or cron at scale. **Not when:** simple high-throughput fan-out — SNS is cheaper. **AWS:** native.

**Apache Pulsar** — streaming with multi-tenancy and tiered storage. **Reach for it when:** Kafka's limitations bite in specific ways. **Not when:** Kafka works. **AWS:** n/a managed.

**NATS** — very lightweight, very fast messaging. **Reach for it when:** low-latency internal service messaging, edge/IoT. **Not when:** you need durable replay (JetStream adds it).

**Debezium** — change data capture from database logs into Kafka. **Reach for it when:** you need to stream database changes without dual-writes — the alternative to the outbox pattern (№31 §8.2). **AWS:** DMS with CDC.

> **Default (AWS): SQS for work queues, SNS/EventBridge for fan-out, Kinesis or MSK only when you need a replayable log.**

---

# 5 · Search

*The problem: text search and analytics beyond what `LIKE` and B-trees can do (№22 §9.3).*

**Elasticsearch / OpenSearch** — distributed inverted-index search and analytics. **Reach for it when:** full-text relevance ranking, faceting, log analytics at scale. **Not when:** Postgres full-text search suffices — it usually does below a few million documents. OpenSearch is the AWS-backed fork after Elastic's 2021 licence change (Elastic has since returned to an OSI licence). **AWS:** OpenSearch Service.

**Postgres full-text search** — `tsvector`, GIN indexes, ranking, built in. **Reach for it when:** you already have Postgres and need decent search. **Not when:** you need advanced relevance tuning or huge scale. **AWS:** included in RDS.

**Meilisearch / Typesense** — lightweight, fast, typo-tolerant search engines. **Reach for it when:** instant-search UX without Elasticsearch's operational weight. **AWS:** self-hosted.

**Apache Solr** — the older Lucene-based engine. **Reach for it when:** an existing deployment. **Not when:** greenfield.

> **Default: Postgres FTS until it demonstrably doesn't scale, then OpenSearch.**

---

# 6 · Containers & orchestration

*The problem: packaging an application with its environment, then running many of them reliably (№50).*

**Docker** — the container engine and image format everyone knows. **Reach for it when:** local development, building images, CI. **AWS:** ECR for registry; images run anywhere.

**Podman** — daemonless, rootless container engine, Docker-CLI-compatible. **Reach for it when:** security-conscious environments, rootless requirements, or avoiding Docker Desktop licensing. **Not when:** you depend on the Docker socket.

**containerd / CRI-O** — the low-level runtimes underneath. **Reach for it when:** you're operating Kubernetes and want to understand or configure the runtime. Not day-to-day tools.

**BuildKit / buildx** — Docker's modern builder: parallelism, cache mounts, build secrets, multi-arch. Now the default. **Reach for it when:** always — you already are.

**Kaniko / Buildah** — build images without a Docker daemon. **Reach for it when:** building inside a Kubernetes cluster or a locked-down CI environment.

**Jib** — builds optimised OCI images straight from Gradle/Maven, **no Dockerfile, no daemon**, with excellent layer caching. **Reach for it when:** a JVM service and you'd rather not maintain a Dockerfile — a genuine option for `practiq-api`. **Not when:** you need fine control or aren't on the JVM.

**Kubernetes** — the container orchestrator: scheduling, self-healing, service discovery, scaling, declarative desired state. **Reach for it when:** many services, multiple teams, multi-cloud, or you need its ecosystem. **Not when:** a handful of services — the complexity is a real and permanent tax. **AWS:** EKS.

**Amazon ECS** — AWS-native orchestrator, far simpler than Kubernetes, free (pay only for compute). **Reach for it when:** you're on AWS and don't specifically need Kubernetes. **AWS:** native, with **Fargate** removing instance management entirely.

**Docker Compose** — multi-container local development from one YAML file. **Reach for it when:** running your app plus Postgres plus dependencies locally. **Not when:** production orchestration.

**Helm** — templated, versioned Kubernetes manifests. **Reach for it when:** you're on Kubernetes and deploying anything non-trivial.

**Nomad** — simpler orchestrator, schedules containers and non-container workloads. **Reach for it when:** Kubernetes is overkill but you need scheduling.

> **Default: Docker to build, ECS/Fargate to run on AWS. Kubernetes when you can name why.**

---

# 7 · IaC & configuration management

*The problem: infrastructure that's reproducible, reviewable and recoverable (№55).*

**Terraform** — declarative HCL, multi-cloud, vast provider ecosystem, the industry standard. BSL-licensed since 2023; HashiCorp acquired by IBM in 2025. **Reach for it when:** provisioning cloud infrastructure. **Not when:** configuring software *inside* machines.

**OpenTofu** — the Linux Foundation fork of Terraform, MPL-licensed, largely compatible; has added state encryption. **Reach for it when:** you want open governance or permissive licensing.

**AWS CloudFormation** — AWS-native declarative IaC; AWS manages the state. **Reach for it when:** AWS-only and you want native drift detection and rollback. **Not when:** multi-cloud, or you dislike the verbosity.

**AWS CDK / Pulumi** — infrastructure in TypeScript/Python/Java/Go. **Reach for it when:** you want loops, types, abstractions and unit tests around infrastructure. **Not when:** you want the constraint of a declarative DSL (real code generates a great deal of infrastructure very easily).

**Ansible** — agentless configuration management over SSH. **Reach for it when:** configuring existing machines, orchestrating operational procedures. **Not when:** provisioning cloud resources — that's Terraform's job. **AWS:** Systems Manager overlaps.

**Packer** — builds machine images (AMIs) from a definition. **Reach for it when:** you need golden VM images. **Not when:** you're containerised — the image *is* the artifact.

**Crossplane** — manage cloud resources through Kubernetes CRDs. **Reach for it when:** already deeply Kubernetes-native.

> **Default: Terraform/OpenTofu for provisioning; containers instead of configuration management.**

---

# 8 · CI/CD

*The problem: turning a commit into a verified, deployed artifact automatically (№56).*

**GitHub Actions** — CI/CD integrated with GitHub; huge marketplace, OIDC to AWS, generous free tier. **Reach for it when:** your code is on GitHub. The default for most projects, including Practiq. **AWS:** CodePipeline/CodeBuild as the native alternative.

**GitLab CI** — deeply integrated with GitLab, strong for self-hosted. **Reach for it when:** you're on GitLab.

**Jenkins** — the veteran; infinitely extensible, self-hosted, plugin-heavy. **Reach for it when:** complex bespoke pipelines or an existing installation. **Not when:** greenfield — the maintenance burden is real.

**CircleCI / Buildkite / Drone** — hosted or hybrid CI. **Reach for it when:** you want specific performance characteristics or self-hosted runners with a hosted control plane.

**AWS CodePipeline / CodeBuild / CodeDeploy** — AWS-native CI/CD. **Reach for it when:** you want everything inside AWS with IAM-native permissions. **Not when:** GitHub Actions is simpler and you're already there.

**ArgoCD / Flux** — GitOps for Kubernetes: the cluster continuously reconciles to match git. **Reach for it when:** Kubernetes and you want declarative, auditable deployments.

**Spinnaker** — sophisticated multi-cloud deployment with advanced canary analysis. **Reach for it when:** large-scale deployment orchestration. **Not when:** anything smaller.

> **Default: GitHub Actions, OIDC into AWS, deploying to ECS.**

---

# 9 · Web servers, proxies & gateways

*The problem: terminating connections, routing, and everything that happens before your application (№51 §10).*

**nginx** — web server and reverse proxy; fast, ubiquitous. **Reach for it when:** serving static files, reverse proxying, TLS termination, load balancing. **AWS:** ALB covers most of the proxying.

**Envoy** — modern L7 proxy; dynamic configuration, observability, the data plane for most service meshes. **Reach for it when:** a service mesh, or advanced traffic management. **AWS:** App Mesh (deprecating — check current guidance), or ALB for simpler cases.

**HAProxy** — high-performance TCP/HTTP load balancer. **Reach for it when:** demanding L4/L7 load balancing on your own infrastructure. **AWS:** NLB/ALB.

**Traefik / Caddy** — proxies with automatic service discovery and automatic HTTPS (Let's Encrypt). **Reach for it when:** local/small deployments where automatic TLS is a delight.

**Apache HTTP Server** — the veteran web server. **Reach for it when:** legacy or `.htaccess` requirements.

**Kong / Tyk** — full API gateways: auth, rate limiting, transformation, plugins. **Reach for it when:** many APIs and consumers needing centralised policy. **AWS:** API Gateway.

**Istio / Linkerd** — service meshes: mTLS, traffic policy, observability between services. **Reach for it when:** many services on Kubernetes and cross-cutting network policy is a real problem. **Not when:** fewer than perhaps a dozen services — the complexity is substantial.

> **Default (AWS): ALB terminates TLS and routes; CloudFront at the edge; no mesh until it hurts.**

---

# 10 · Observability

*The problem: understanding a running system you can't debug directly (№57).*

**OpenTelemetry** — the vendor-neutral standard for traces, metrics and logs; instrument once, export anywhere. **Reach for it when:** always — it's the correct instrumentation layer regardless of backend. **AWS:** ADOT (AWS Distro for OpenTelemetry).

**Prometheus** — pull-based metrics with a dimensional data model and PromQL. **Reach for it when:** self-hosted metrics, Kubernetes (it's the ecosystem default). **Not when:** you'd rather not operate it. **AWS:** Amazon Managed Service for Prometheus.

**Grafana** — dashboards and alerting over almost any data source. **Reach for it when:** visualisation. The de facto standard. **AWS:** Amazon Managed Grafana.

**Amazon CloudWatch** — AWS-native metrics, logs, alarms, dashboards. **Reach for it when:** you're on AWS and want zero infrastructure. **Not when:** you need PromQL or sophisticated querying.

**Grafana Loki** — log aggregation indexed by labels rather than full text; cheap. **Reach for it when:** you want Grafana-native logs without Elasticsearch costs.

**ELK / OpenSearch stack** — Elasticsearch + Logstash + Kibana for log search and analytics. **Reach for it when:** heavy log search and analysis. **Not when:** you don't want to run it. **AWS:** OpenSearch Service.

**Jaeger / Tempo / Zipkin** — distributed tracing backends. **Reach for it when:** you need traces and aren't using a commercial platform. **AWS:** X-Ray.

**Datadog / New Relic / Honeycomb / Dynatrace** — commercial all-in-one observability. **Reach for it when:** you want everything integrated and will pay for it; Honeycomb in particular is built for high-cardinality exploratory analysis (№57 §1.2). **Not when:** cost matters more than convenience — bills scale alarmingly with data volume.

**Sentry** — error tracking and aggregation with rich context. **Reach for it when:** you want exceptions grouped, deduplicated and attributed. Excellent value.

**PagerDuty / Opsgenie** — on-call scheduling and alert routing. **Reach for it when:** you have an on-call rota.

> **Default (Practiq): CloudWatch + X-Ray, instrumented via OpenTelemetry to keep options open.**

---

# 11 · Profiling & load testing

*The problem: finding out why it's slow, and whether it survives load (№13 §9).*

**JDK Flight Recorder + Mission Control** — built-in JVM profiling, ~1% overhead, production-safe. **Reach for it when:** any JVM performance question. First reach.

**async-profiler** — low-overhead sampling profiler without safepoint bias; flame graphs. **Reach for it when:** CPU or allocation hotspots. **Not when:** JFR already answered it.

**Eclipse MAT** — heap dump analysis; dominator tree, path to GC roots. **Reach for it when:** hunting a memory leak. The definitive tool.

**JMH** — the JVM microbenchmark harness; handles warmup, forking, dead-code elimination. **Reach for it when:** comparing two implementations of hot code. **Not when:** you actually need a load test.

**VisualVM / JProfiler / YourKit** — GUI profilers. **Reach for it when:** interactive local investigation; the commercial ones are excellent and paid.

**k6** — load testing with JavaScript scenarios; developer-friendly, scriptable, CI-friendly. **Reach for it when:** load testing an API. Good default.

**Gatling** — Scala/Java load testing with strong reporting. **Reach for it when:** JVM-shop load testing.

**Apache JMeter** — the veteran; GUI-driven, very capable, dated.

**Locust** — Python-based load testing. **Reach for it when:** your team writes Python.

**perf / strace / bpftrace** — Linux-level profiling and tracing. **Reach for it when:** the problem is below the JVM (№52).

> **Default: JFR first, flame graph next, MAT for leaks, k6 for load.**

---

# 12 · Build tools & package management

*The problem: compiling, dependency resolution, and reproducible artifacts.*

**Gradle** — JVM build tool; Groovy or **Kotlin DSL**, incremental builds, build cache, flexible. **Reach for it when:** JVM projects wanting speed and flexibility (Practiq's choice). **Not when:** you'd rather have Maven's rigidity and convention.

**Maven** — declarative XML, convention over configuration, enormous ecosystem. **Reach for it when:** you value predictability and standard structure over flexibility.

**npm / pnpm / yarn** — JavaScript package managers. **pnpm** is notably faster and more disk-efficient. **Reach for it when:** any JS project; commit the lockfile.

**Vite** — frontend build tool and dev server; fast, ES-module-native. **Reach for it when:** modern frontend builds. The current default (№70 §6.2).

**webpack** — the older, highly configurable bundler. **Reach for it when:** an existing project or unusual bundling needs.

**Bazel** — hermetic, reproducible, polyglot builds at scale. **Reach for it when:** a large monorepo with genuine build-time pain. **Not when:** anything smaller — the setup cost is very high.

**Nexus / Artifactory / CodeArtifact** — private artifact repositories. **Reach for it when:** internal libraries, or proxying public registries for reliability and audit. **AWS:** CodeArtifact.

> **Default: Gradle (Kotlin DSL) for JVM, pnpm + Vite for frontend.**

---

# 13 · Version control & collaboration

*The problem: history, review and coordination (№53).*

**Git** — distributed version control. Universal. **AWS:** CodeCommit (closed to new customers — use GitHub).

**GitHub** — hosting plus PRs, Actions, Issues, Packages, security scanning. **Reach for it when:** almost always; the ecosystem gravity is enormous.

**GitLab** — hosting with integrated CI, registry and DevOps features; strong self-hosted story. **Reach for it when:** you want one integrated platform or self-hosting.

**Bitbucket** — Atlassian's offering. **Reach for it when:** you're in a Jira/Confluence estate.

**Gitea / Forgejo** — lightweight self-hosted git hosting. **Reach for it when:** you want your own server without GitLab's weight.

**pre-commit / Husky / lint-staged** — git hook management. **Reach for it when:** you want formatting and linting enforced before commit. Keep hooks fast.

> **Default: GitHub.**

---

# 14 · Secrets, identity & security

*The problem: credentials, authentication, and keeping vulnerabilities out (№60).*

**AWS Secrets Manager** — secrets with automatic rotation and native RDS integration. **Reach for it when:** you need rotation. **Not when:** simple config — Parameter Store is free.

**AWS SSM Parameter Store** — hierarchical config and secrets; free standard tier, KMS encryption. **Reach for it when:** most configuration and secret storage on AWS.

**HashiCorp Vault** — secrets management, dynamic short-lived credentials, encryption as a service, PKI. **Reach for it when:** multi-cloud, dynamic database credentials, or advanced requirements. **Not when:** AWS-native suffices.

**AWS KMS** — managed encryption keys; envelope encryption (№91 §7). **Reach for it when:** any encryption on AWS.

**Keycloak** — open-source identity provider; OIDC/SAML, self-hosted. **Reach for it when:** you want full control of identity. **AWS:** Cognito.

**Auth0 / Okta** — managed identity platforms. **Reach for it when:** you want auth solved with minimal effort and will pay. **AWS:** Cognito.

**Amazon Cognito** — AWS-native user pools and identity pools. **Reach for it when:** AWS-native auth without another vendor.

**Dependabot / Renovate** — automated dependency updates; Renovate is more configurable. **Reach for it when:** always — this is your primary supply-chain defence (№60 §8).

**Snyk / Trivy / Grype** — vulnerability scanning for dependencies, images and IaC. Trivy is excellent, free and fast. **Reach for it when:** CI security scanning. **AWS:** ECR scanning, Inspector.

**tfsec / Checkov** — IaC security scanning. **Reach for it when:** you write Terraform (№55 §8).

**SonarQube** — static analysis for quality and security. **Reach for it when:** you want tracked code-quality gates. **AWS:** CodeGuru overlaps.

**gitleaks / TruffleHog** — secret scanning in repositories and history. **Reach for it when:** always — as a hook and in CI.

**OWASP ZAP / Burp Suite** — web application security testing. **Reach for it when:** you want to probe your own application; Burp is the professional standard.

> **Default: Parameter Store for config, Secrets Manager for rotation, Cognito for identity, Trivy + Dependabot + gitleaks in CI.**

---

# 15 · Data & analytics

*The problem: moving, transforming and analysing data at volume (№91 §10).*

**Apache Spark** — distributed data processing. **Reach for it when:** large-scale ETL or ML pipelines. **Not when:** SQL on your database would do. **AWS:** EMR, Glue.

**Apache Airflow** — workflow orchestration as Python DAGs. **Reach for it when:** complex scheduled data pipelines with dependencies. **AWS:** MWAA, or Step Functions for lighter needs.

**dbt** — SQL-based transformation with testing, documentation and lineage. **Reach for it when:** you're transforming data inside a warehouse. Genuinely excellent.

**Apache Flink** — stateful stream processing. **Reach for it when:** real-time computation over streams. **AWS:** Managed Service for Apache Flink.

**Snowflake / BigQuery / Redshift** — cloud data warehouses. **Reach for it when:** analytical queries over large historical data separate from your OLTP database. **AWS:** Redshift.

**DuckDB** — in-process analytical (OLAP) database; astonishingly fast on local files. **Reach for it when:** analysing Parquet/CSV locally, or embedded analytics. A genuine delight.

**Apache Iceberg / Delta Lake** — open table formats over object storage; ACID, time travel, schema evolution. **Reach for it when:** building a data lakehouse. **AWS:** supported by Athena, Glue, EMR.

**Athena** — serverless SQL over S3. **Reach for it when:** ad-hoc queries on data already in S3. Partition and use Parquet or the cost surprises you.

**Metabase / Superset / QuickSight** — BI and dashboards. **Reach for it when:** non-engineers need to explore data. Metabase is the easiest to start with.

> **Default: keep analytics in Postgres until it hurts; then Athena over S3 before a warehouse.**

---

# 16 · Java & JVM ecosystem

*The libraries you'll actually import (№10, №14).*

**Micronaut** — compile-time DI, fast startup, low memory, native-image-friendly. **Reach for it when:** microservices, serverless, or you value compile-time safety (Practiq's choice). **Not when:** you need Spring's ecosystem breadth.

**Spring Boot** — the dominant JVM framework; unmatched ecosystem and hiring pool. **Reach for it when:** you want the largest ecosystem and the most answers online. **Not when:** startup time and memory dominate.

**Quarkus** — Kubernetes-native Java, compile-time optimised, excellent native-image support. **Reach for it when:** Red Hat ecosystem or container-first Java.

**Hibernate** — the JPA implementation (№21). **Reach for it when:** ORM. **Not when:** you want SQL control — see jOOQ.

**jOOQ** — type-safe SQL DSL generated from your schema. **Reach for it when:** SQL is your model and you want the compiler to check it. **Not when:** you want an object graph and change tracking.

**MyBatis** — SQL mapped explicitly to objects. **Reach for it when:** DBAs own the SQL.

**Flyway / Liquibase** — database migrations. Flyway is simpler (versioned SQL); Liquibase more abstracted and database-agnostic. **Reach for it when:** any schema evolution. Non-negotiable.

**Jackson** — JSON serialisation. The default. **Reach for it when:** JSON. Avoid enabling default polymorphic typing (№60 §10.5).

**JUnit 5 / AssertJ / Mockito / Testcontainers** — the testing stack (№44). AssertJ for fluent assertions, Testcontainers for real dependencies in tests. **Reach for it when:** always.

**Lombok** — annotation-driven boilerplate reduction. **Reach for it when:** you want fewer getters. **Not when:** records cover it (№10 §2.2) — prefer the language feature.

**MapStruct** — compile-time bean mapping. **Reach for it when:** mapping entities to DTOs at scale, and you want it generated rather than reflective.

**Resilience4j** — circuit breakers, retries, rate limiters, bulkheads (№31 §10). **Reach for it when:** calling anything over a network.

**SLF4J + Logback / Log4j2** — logging facade and implementations. **Reach for it when:** always; add a JSON encoder for structured logs (№57 §2.1).

**Guava / Apache Commons** — utility libraries. **Reach for it when:** they solve something the JDK doesn't — which is less than it used to be.

**GraalVM** — native image compilation and a polyglot runtime. **Reach for it when:** startup time and memory dominate (№13 §12.E).

> **Default (Practiq): Micronaut + Hibernate + Flyway + Jackson + JUnit5/AssertJ/Mockito/Testcontainers + Resilience4j.**

---

# 17 · Local development

*The problem: a fast, faithful, low-friction loop on your own machine.*

**Docker Desktop / OrbStack / Colima / Podman Desktop / Rancher Desktop** — local container runtimes. **OrbStack** is notably fast on macOS; **Colima** is a lightweight CLI option; Docker Desktop requires a paid licence for larger companies. **Reach for it when:** you need containers locally.

**Testcontainers** — real dependencies (Postgres, Kafka, LocalStack) started from your test code. **Reach for it when:** integration tests. The single biggest improvement to test fidelity available (№44 §7.3).

**LocalStack** — AWS services emulated locally. **Reach for it when:** developing against S3/SQS/DynamoDB without an AWS account or costs. **Not when:** you need exact parity — emulation drifts.

**direnv / dotenv** — per-directory environment configuration. **Reach for it when:** juggling project-specific env vars.

**jq / yq** — command-line JSON and YAML processors. **Reach for it when:** any shell pipeline touching structured data (№52 §10.3). Essential.

**httpie / curl / Bruno / Postman** — HTTP clients. **curl** for scripting and CI, **httpie** for human-friendly ad-hoc requests, **Bruno** as a git-friendly Postman alternative.

**tmux** — persistent terminal sessions. **Reach for it when:** long-running work over SSH (№52 §3.5).

**ripgrep (rg) / fd / bat / fzf** — modern replacements for grep, find, cat and interactive filtering. **Reach for it when:** you spend time in a terminal; the speed difference is not subtle.

**IntelliJ IDEA** — the JVM IDE. **Reach for it when:** Java. The refactoring and debugging tooling is a genuine productivity multiplier (№40 §9.2).

**VS Code** — lightweight, extensible, excellent for frontend and polyglot work.

> **Default: OrbStack or Docker Desktop, Testcontainers for tests, IntelliJ for Java, jq and ripgrep always.**

---

# How to use this catalogue

- **Read the category, not the tool.** The category tells you what problem exists; the default tells you what to use; the cards tell you when to deviate. Adopting a tool without being able to state the pressure that justifies it is how systems accumulate complexity nobody can remove.
- **Related library docs:** №54 and №91 (AWS specifically), №31 (the distributed-systems concepts under §2–4), №50 (§6), №55 (§7), №56 (§8), №57 (§10), №60 (§14), №20/№22 (§1), №13 (§11), №10/№14 (§16).
- **Candidates for expansion:** a **decision-tree version** ("I need to store data → is it relational? → …"); **cost comparisons** for the AWS equivalents; **a Practiq-specific shortlist** of what to adopt next and in what order.

*The fastest-drifting document here. Categories and problem framings are stable; specific tools change status — licences shift (Redis, Elastic, Terraform all changed in recent years), tools get acquired or abandoned, AWS renames services. Verify anything you're about to adopt.*
