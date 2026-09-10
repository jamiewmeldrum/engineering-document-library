# Observability & Production Operations — A Primer №57

*How you understand a system you can't attach a debugger to. The three pillars, what to measure, how to alert without drowning, and what to do when it breaks at 3am. The other half of №44 — that document is correctness before deployment; this is correctness after it.*

The distinction that motivates the whole field: **monitoring tells you whether the things you predicted are happening; observability tells you what's happening when you didn't predict it.** A dashboard of CPU and memory is monitoring — you decided in advance those mattered. Observability is the property of a system that lets you ask questions you didn't anticipate: "why are requests from this one customer slow, only on Tuesdays, only for questions tagged to concept 42?" You can't dashboard that in advance; you need enough high-cardinality detail recorded to interrogate it after the fact.

The second idea, which is the practical core: **you cannot debug what you cannot see, and production is a black box by default.** Every technique here is about making the box transparent — and the effort has to go in *before* the incident, because during one you can only use what you already instrumented.

Contents:

- **Part 1** — monitoring vs observability
- **Part 2** — logs
- **Part 3** — metrics
- **Part 4** — traces
- **Part 5** — OpenTelemetry and the tooling landscape
- **Part 6** — what to measure
- **Part 7** — SLIs, SLOs and error budgets
- **Part 8** — alerting
- **Part 9** — dashboards
- **Part 10** — incident response
- **Part 11** — when to use what

## Question index

| The question | The pillar | §|
|---|---|---|
| Is the system healthy right now? | metrics + dashboards | §3, §9 |
| What exactly happened to this request? | logs (with a trace ID) | §2 |
| Where did the 2 seconds go? | traces | §4 |
| Is this worse than yesterday? | metrics | §3 |
| Which users are affected? | logs/traces with high-cardinality attributes | §1.2 |
| Did the deploy cause it? | metrics annotated with deploys | §9.2 |
| Are we meeting our promises? | SLIs/SLOs | §7 |
| Should someone be woken up? | alerting philosophy | §8 |
| Why did it break, and how do we prevent it? | post-mortem | §10.4 |

---

# Part 1 — Monitoring vs observability

## 1.1 The difference

**Monitoring** — collecting predefined signals and checking them against thresholds. Answers **known questions**: is CPU high, is the disk full, is the error rate above 1%.

**Observability** — a property of the system: can you determine its internal state from its external outputs? Answers **unknown questions**, which are the ones incidents are actually made of.

The practical distinction is **cardinality**. Monitoring aggregates: "p99 latency is 800ms." Observability preserves detail: "p99 latency is 800ms, and it's entirely requests from tenant X hitting the concept-filter endpoint with more than 50 results." The second lets you find the cause; the first only tells you something is wrong.

## 1.2 Cardinality — the concept that separates the two

**Cardinality** is the number of distinct values a field can take. `environment` is low cardinality (three values); `user_id` is high (millions).

**High-cardinality data is where the answers live** — the cause is nearly always specific to some user, some endpoint, some version, some region. But high cardinality is exactly what metrics systems handle badly: a metric tagged with `user_id` creates a separate time series per user, and the cost explodes.

Hence the division of labour that structures everything below: **metrics for aggregate health (low cardinality, cheap, always on); logs and traces for detail (high cardinality, expensive, sampled).**

---

# Part 2 — Logs

## 2.1 Structured logging

The single highest-value change most codebases can make. Not this:

```
2026-07-23 14:32:01 ERROR Failed to approve question 4821 for user jamie
```

but this:

```json
{"timestamp":"2026-07-23T14:32:01Z","level":"ERROR","message":"question approval failed",
 "questionId":4821,"userId":"jamie","status":"PENDING","traceId":"a1b2c3d4",
 "service":"practiq-api","version":"1.4.2","error":"OptimisticLockException"}
```

The difference is queryability. The first can only be grepped with a regex someone invents under pressure. The second supports `level:ERROR AND questionId:4821`, aggregation by error type, and correlation by `traceId`. **Machines read logs far more than humans do** — write for the machine.

In Java: SLF4J with a JSON encoder (Logback/Log4j2), and use parameterised logging so the structure survives:

```java
log.error("question approval failed", kv("questionId", id), kv("status", status), e);
```

## 2.2 Correlation IDs

**The one thing to implement first.** Generate an ID at the edge, attach it to every log line, and propagate it through every call:

```java
// a filter at the entry point
String traceId = request.getHeader("X-Trace-Id") != null
        ? request.getHeader("X-Trace-Id") : UUID.randomUUID().toString();
MDC.put("traceId", traceId);      // now every log line in this thread carries it
try { chain.doFilter(...); } finally { MDC.clear(); }
```

Without it, reconstructing one request's journey through a system means guessing from timestamps. With it, one query returns the whole story. (MDC is thread-local, so be careful across async boundaries and virtual threads — the context must be propagated explicitly.)

## 2.3 Levels, and what to log

| Level | For | Example |
|---|---|---|
| **ERROR** | something failed and needs attention | unhandled exception, failed write |
| **WARN** | recovered, but notable | retry succeeded, fallback used, deprecation |
| **INFO** | significant business events | question approved, user registered |
| **DEBUG** | detail for diagnosis | off in production, on when needed |
| **TRACE** | very fine detail | rarely enabled |

**Log the decision, not the traversal.** "Approved question 4821 (2 reviews, threshold 2)" is worth logging; "entering method approve()" is not. Include enough context to act — ids, states, the values that drove the decision.

**Never log secrets, passwords, tokens, full card numbers or unnecessary PII.** Logs get shipped to third parties, retained for months, and read by many people. This is a real and common breach vector.

**Log exceptions with the stack trace**, and log them **once** — catching, logging and rethrowing at every layer produces the same trace five times and makes the log useless (№40 §7.2).

## 2.4 Aggregation

Per-instance log files are useless when instances are ephemeral (a Fargate task that died took its logs with it). Ship logs to a central store — CloudWatch Logs, ELK/OpenSearch, Loki, Datadog — where they're searchable across instances and retained past the container's life. Set **retention** deliberately: logs are cheap to write and expensive to store forever.

---

# Part 3 — Metrics

## 3.1 What they are

Numeric measurements over time, cheap to store and aggregate. Where a log is an event, a metric is a **number with dimensions**, sampled continuously.

| Type | Is | Example |
|---|---|---|
| **Counter** | monotonically increasing | requests_total, errors_total |
| **Gauge** | a value that goes up and down | active_connections, queue_depth, heap_used |
| **Histogram** | distribution of values into buckets | request_duration — **enables percentiles** |
| **Summary** | client-computed quantiles | similar, less aggregatable |

## 3.2 Percentiles, and why averages lie

**Never alert on averages.** If 99 requests take 50ms and one takes 10 seconds, the average is 150ms — which looks fine and describes nobody's experience. Percentiles describe reality:

- **p50 (median)** — the typical experience.
- **p95 / p99** — the tail, where the pain is.
- **p99.9** — the worst, and often where systemic problems announce themselves first.

At scale, the tail matters more than it seems: if a page makes 20 backend calls, a p99 of 2 seconds means roughly a fifth of page loads contain at least one 2-second call.

**Histograms are what let you compute percentiles** after the fact and aggregate them across instances — which is why you record durations as histograms rather than gauges.

## 3.3 Labels and cardinality, again

```
http_requests_total{method="GET", endpoint="/questions", status="200", version="1.4.2"}
```

Labels make metrics sliceable — but **each unique label combination is a separate time series**, so cost is multiplicative. Adding `user_id` or a raw URL with IDs in it (`/questions/4821`) can generate millions of series and take down your metrics backend. Normalise paths to route templates (`/questions/{id}`), and keep high-cardinality dimensions in logs and traces where they belong (§1.2).

---

# Part 4 — Traces

## 4.1 The model

A **trace** follows one request through the whole system. It's a tree of **spans**, each representing one operation with a start time, duration, and attributes.

```
Trace: a1b2c3d4                                         total 847ms
├─ GET /api/v1/questions                    [847ms]
│  ├─ auth.validate                          [12ms]
│  ├─ QuestionService.list                  [810ms]
│  │  ├─ SELECT question WHERE ...           [45ms]
│  │  ├─ SELECT concept WHERE id=?            [8ms]   ← 
│  │  ├─ SELECT concept WHERE id=?            [7ms]   ← N+1
│  │  ├─ SELECT concept WHERE id=?            [9ms]   ←  detected
│  │  └─ ... ×50
│  └─ serialise                               [25ms]
```

**That picture is why tracing exists.** The N+1 problem (№20 §2.8) is invisible in metrics (the endpoint is "just slow") and painful to spot in logs; in a trace it's obvious at a glance. Tracing answers **"where did the time go?"** in a way nothing else does.

## 4.2 Context propagation

A trace works because the trace ID and parent span ID travel with the request — as HTTP headers (the W3C `traceparent` standard), message attributes on a queue, or thread-local context in-process. **Every hop must propagate it**, which is why instrumentation libraries hook the HTTP client, the database driver and the message producer.

If a service drops the context, the trace breaks in two and you lose the connection at exactly the point you needed it.

## 4.3 Sampling

Tracing every request at scale is expensive. **Head-based sampling** decides at the start (keep 1%) — cheap and simple, but it discards most errors, which are exactly what you want. **Tail-based sampling** decides after the trace completes (keep everything slow or failing, plus a baseline sample of successes) — much more useful, more infrastructure.

Practical default: sample low for successful fast requests, **keep 100% of errors and slow requests**.

---

# Part 5 — OpenTelemetry and the landscape

**OpenTelemetry (OTel)** is the vendor-neutral standard — one set of APIs, SDKs and a collector for traces, metrics and logs, exporting to whatever backend you choose. Its value is that instrumentation is no longer vendor lock-in: instrument once, switch from CloudWatch to Datadog to Grafana without touching application code. It has become the default answer, and the **AWS Distro for OpenTelemetry** is the supported path into AWS's tooling.

Auto-instrumentation is the fastest start: a Java agent that instruments HTTP servers, clients, JDBC and common libraries with no code changes, giving you traces and basic metrics immediately.

| Concern | Options |
|---|---|
| Metrics | Prometheus, CloudWatch, Datadog |
| Logs | CloudWatch Logs, OpenSearch/ELK, Loki, Datadog |
| Traces | Jaeger, Tempo, AWS X-Ray, Datadog |
| Dashboards | Grafana, CloudWatch Dashboards |
| All-in-one | Datadog, New Relic, Honeycomb, Grafana Cloud |

For Practiq on AWS: **CloudWatch plus X-Ray is the low-effort path** (native, no infrastructure to run); instrumenting with **OTel** keeps the door open to moving later.

---

# Part 6 — What to measure

Three standard frameworks, each for a different subject:

**The Four Golden Signals** (Google SRE) — for any user-facing service:

| Signal | Is | Watch |
|---|---|---|
| **Latency** | how long requests take | p50/p95/p99, **and separate successes from failures** |
| **Traffic** | how much demand | requests/sec |
| **Errors** | how many fail | rate, and by type |
| **Saturation** | how full the system is | CPU, memory, queue depth, **connection pool utilisation** |

**RED** — for request-driven services: **R**ate, **E**rrors, **D**uration. Essentially the golden signals minus saturation, and the right default for an API.

**USE** — for resources: **U**tilisation, **S**aturation, **E**rrors. The right lens for a database, a disk, a thread pool.

**Practiq-specific things worth instrumenting**: HTTP rate/errors/duration per route; **HikariCP pool utilisation and wait time** (pool exhaustion is a classic outage cause, №20 §1.6); query duration; JVM heap, GC pause time and count (№10 §13); extraction queue depth and job duration; and business metrics — questions approved per day, extraction success rate — which are often the first signal that something subtle is wrong.

**Instrument the boundaries first**: inbound requests, outbound calls, database, queue. That covers most of what you need for a fraction of the effort of instrumenting everything.

---

# Part 7 — SLIs, SLOs and error budgets

## 7.1 The definitions

- **SLI** (Indicator) — a *measurement* of service quality. "Proportion of requests served under 300ms."
- **SLO** (Objective) — your *target* for that indicator. "99.5% of requests under 300ms over 30 days."
- **SLA** (Agreement) — a *contract* with consequences, usually financial. You rarely have one; the SLO is the operationally useful thing.

Set SLOs on what **users experience** — availability and latency of the endpoints they use — not on CPU. Nobody cares about your CPU.

## 7.2 Error budgets — the genuinely useful idea

If your SLO is 99.5%, then **0.5% of requests are allowed to fail** — that's your **error budget**. Over 30 days, 99.5% availability permits roughly 3.6 hours of downtime.

Why this reframing matters:

- It makes reliability **quantitative rather than moral**. "Is this reliable enough?" becomes "have we spent our budget?"
- It **prices risk**. Budget remaining → ship faster, take the risky refactor. Budget exhausted → freeze features, spend the time on stability.
- It kills the unwinnable argument between "move fast" and "don't break things" by making the trade-off explicit and measurable.

**100% is the wrong target.** Chasing it costs exponentially more for diminishing returns, and it means you can never deploy. Choose a number that reflects what users actually need.

---

# Part 8 — Alerting

## 8.1 The philosophy

**Alert on symptoms, not causes.** High CPU is not a problem if users are fine; a 5% error rate is a problem regardless of what the CPU is doing. Cause-based alerts fire constantly for conditions that don't matter and miss the failures you didn't anticipate.

**Every page must be actionable and urgent.** If the response is "yes, we know" or "I'll look tomorrow", it should not be a page. Alerts that don't require immediate human action belong on a dashboard or a ticket.

**Alert fatigue is the real failure mode.** A team that receives forty alerts a night stops reading them, and the one that mattered is lost in the noise. **Fewer, better alerts beat comprehensive alerting** — every noisy alert should be tuned or deleted, not tolerated.

## 8.2 What to alert on

Good pages: error rate above the SLO burn threshold; latency p99 breaching the SLO; the service being down (synthetic check); a queue growing unboundedly; **error-budget burn rate** (you'll exhaust the month's budget in six hours); certificate expiry (with days of warning, №51 §8.3); disk approaching full.

Not pages: a single failed request; CPU at 80%; an individual instance restarting when the fleet is healthy; anything self-healing.

**Multi-window burn-rate alerting** is the modern refinement: alert when the budget is burning fast over a short window *and* confirmed over a longer one, which catches genuine problems quickly while suppressing brief blips.

## 8.3 Runbooks

Every alert should link to a **runbook**: what this means, how to confirm it, what to check, how to mitigate, when to escalate. Written when you *build* the alert, while you understand it — not at 3am by someone who has never seen it. This is one of those small disciplines with disproportionate payoff.

---

# Part 9 — Dashboards

## 9.1 Design

Build for a **specific question and audience**. A dashboard trying to show everything shows nothing. Useful shapes:

- **Service overview** — the golden signals for one service, top to bottom, so health is readable in five seconds.
- **Incident dashboard** — what you open during an outage: errors, latency, saturation, recent deploys, dependency health.
- **Business dashboard** — questions approved, extraction throughput, active users.

Practical rules: most important at the top-left; consistent time ranges across panels; annotate **deployments** on the timeline (§9.2); show SLO attainment against target; and prefer a few clear panels to a wall of sparklines.

## 9.2 Deployment annotations

Marking deploys on your metric graphs is the cheapest high-value feature in any dashboard. "What changed?" is the first question in most incidents (№52 §11), and a vertical line at the moment of the deploy answers it instantly.

---

# Part 10 — Incident response

## 10.1 The order of operations

**Mitigate first, diagnose second.** The instinct to find the root cause while users are affected is the wrong one. Restore service — roll back, scale up, fail over, disable the feature flag — *then* investigate calmly with the pressure off (№56 §12.F).

## 10.2 The flow

**Detect** (alert or report) → **Triage** (how bad, who's affected, declare severity) → **Communicate** (a status update; someone owns comms so responders can work) → **Mitigate** (restore service) → **Diagnose** (with the fire out) → **Fix** → **Learn** (§10.4).

Roles matter once more than one person is involved: an **incident commander** coordinates and decides (and does *not* debug), a **communications lead** handles updates, and **responders** investigate. Without this, five people debug the same thing and nobody talks to stakeholders.

## 10.3 The first five minutes

1. **What changed?** Deploys, config, feature flags, infrastructure. Most incidents are recent changes.
2. **How bad?** Error rate, affected users, which endpoints.
3. **Can I mitigate now?** Roll back, flag off, scale, fail over.
4. **Who needs to know?**

Then the technical sweep: dashboards → recent deploys → error logs (filtered by trace ID once you have an example) → traces for the slow path → the layered checks in №52 §11 and №51 §11.

## 10.4 Blameless post-mortems

Written for every significant incident. The premise, which is not a platitude but an engineering position: **people act reasonably given the information and systems they have. If someone could take the system down with one command, the system permitted it — that's the finding.** Blame produces hiding, and hiding produces repeat incidents.

Contents: a timeline, user impact, what happened technically, contributing factors (plural — single root causes are usually a simplification), what went well, and **action items with owners and dates**. The output must be *changes to the system* — a guard rail, a test, an alert, a runbook — not "be more careful."

## 10.5 Beyond incidents

**On-call** should be sustainable: a rota with enough people, compensated, with a handover, and a rule that a night of pages generates work to stop it recurring. **Chaos engineering** — deliberately injecting failure to verify resilience — is worthwhile once the basics are solid. **Game days** rehearse incident response before you need it.

---

# Part 11 — When to use what

**A. Log, metric or trace?** Tell → metric: a number you'll aggregate and alert on. Tell → log: a discrete event with detail you'll query. Tell → trace: understanding latency across components. Default: **metrics for health, logs for detail, traces for where the time went.**

**B. What level to log?** Tell → ERROR: needs human attention. Tell → WARN: recovered but notable. Tell → INFO: a significant business event. Tell → DEBUG: off in production. Default: **INFO for decisions, ERROR for failures, and never log the traversal.**

**C. Alert or dashboard?** Tell → alert: requires immediate human action. Tell → dashboard: useful to look at, not urgent. Default: **if you wouldn't want to be woken for it, it's not a page.**

**D. Sample or keep everything?** Tell → keep all: errors, slow requests, low-volume services. Tell → sample: high-volume successful requests. Default: **100% of errors, sample the happy path.**

**E. Managed or self-hosted?** Tell → managed (CloudWatch/Datadog): small team, don't want to run Elasticsearch. Tell → self-hosted (Prometheus/Grafana/Loki): cost at scale, or specific needs. Default (Practiq): **CloudWatch + X-Ray, instrumented via OTel to keep options open.**

**F. Roll back or investigate?** Tell → roll back: users are affected and a known-good version exists. Tell → investigate: the impact is contained, or rollback is unsafe (a migration). Default: **mitigate first, always.**

**G. What SLO?** Tell → higher (99.9%+): revenue-critical, user-facing. Tell → lower (99%): internal tools, batch processing. Default: **pick a number users would actually notice, never 100%.**

---

# How to expand this

- *Related:* №31 §11 (observability in distributed systems), №44 §9 (debugging — this is its production counterpart), №52 §11 (the Linux troubleshooting sweep), №51 §11 (the network ladder), №56 (deployments as the thing you're watching), №54 §10 / №91 §9 (the AWS services).
- *Candidates for deeper treatment:* **instrumenting Practiq end to end** with OpenTelemetry — traces through the API into Postgres, custom metrics, the dashboards; **SLO design workshop** — choosing indicators and targets for a real service; **an incident-response playbook** with severity definitions and runbook templates; **log-cost management** at scale (sampling, tiering, retention).

*Stable practice, written from knowledge — the pillars, golden signals, SLO/error-budget model and incident practice don't drift. Tooling and vendor capabilities change; OpenTelemetry in particular is still evolving its logs and metrics specifications, so check current status before committing.*
