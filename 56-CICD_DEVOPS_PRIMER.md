# CI/CD & DevOps — A Primer №56

*How code gets from your laptop to production safely and often. The pipeline that turns a commit into a running system, the deployment strategies that make releases boring, and the cultural ideas underneath. GitHub Actions as the concrete example (Practiq's CI), but the concepts transfer.*

The counter-intuitive claim that underpins everything here, and it's empirically supported rather than aspirational: **deploying more often is safer, not riskier.** The instinct is the opposite — releases are scary, so do fewer of them. But a quarterly release bundles three months of changes, so when it breaks you have a thousand suspects, no memory of the reasoning, and a rollback that undoes everything. A release containing one change has one suspect, fresh context, and a trivial rollback. **Smaller batches reduce risk per release *and* total risk.** Everything in this document exists to make small, frequent releases possible.

The second idea, which is what "DevOps" actually meant before it became a job title: **the people who build the software are responsible for running it.** Not a wall to throw artifacts over. That feedback loop — you get paged for your own bugs — is what makes people build observable, resilient, operable systems (№57).

Contents:

- **Part 1** — the problems CI/CD solves
- **Part 2** — continuous integration
- **Part 3** — continuous delivery vs deployment
- **Part 4** — pipeline anatomy
- **Part 5** — GitHub Actions concretely
- **Part 6** — artifacts and registries
- **Part 7** — environments, config and secrets
- **Part 8** — deployment strategies
- **Part 9** — feature flags and decoupling deploy from release
- **Part 10** — databases in a pipeline
- **Part 11** — measuring it: DORA
- **Part 12** — when to use what

## Problem index

| Situation | Concept | §|
|---|---|---|
| "It worked on my machine" | build in CI, containerise | §2.2, №50 |
| Merge conflicts are constant hell | integrate more often, smaller branches | §2.1 |
| Releases are scary events | smaller batches, automated pipeline | §1, §8 |
| A bad deploy took the site down | canary/blue-green + automated rollback | §8 |
| We can't ship because feature X is half-done | feature flags | §9 |
| Secrets are in the repo | secret store + injection at runtime | §7.3 |
| The pipeline is slow so people skip it | caching, parallelism, test tiering | §4.3 |
| A migration broke the running app | expand/contract migrations | §10 |
| Nobody knows if we're improving | DORA metrics | §11 |

---

# Part 1 — The problems being solved

**Integration hell.** Developers work on branches for weeks; merging becomes a multi-day archaeology exercise because everyone changed the same things in different directions. **CI's answer: integrate constantly, in small pieces.**

**Big-bang releases.** Months of accumulated change deployed at once, at night, with a rollback plan nobody has tested. **CD's answer: deploy small changes continuously, so deployment is routine.**

**Manual, unrepeatable processes.** A wiki page of seventeen steps that one person actually knows. **Automation's answer: the pipeline is the process, and it's in version control.**

**Slow feedback.** A bug written on Monday found in staging three weeks later, when the author has forgotten the context entirely. **Fast pipelines' answer: know within minutes.**

---

# Part 2 — Continuous integration

## 2.1 What CI actually means

Not "we have a build server." **CI means every developer integrates their work into the shared main branch frequently — at least daily — and an automated build verifies each integration.**

The frequency is the point. Branches that live for hours diverge trivially; branches that live for weeks diverge structurally. This is why trunk-based development and short-lived branches (№53 §10.2) are the natural partner to CI — long-lived feature branches are, by definition, *deferred* integration.

## 2.2 What runs on every commit

The build must be **reproducible and independent of any developer's machine** — a clean checkout, pinned dependencies, containerised where practical (№50 §5). Then:

- **Compile** — it builds.
- **Unit tests** — fast, and the bulk of the suite (№44 §2).
- **Static analysis and linting** — style, common bugs, complexity.
- **Integration tests** — against real dependencies via Testcontainers (№44 §7.3).
- **Security scanning** — dependency CVEs (§4.4).
- **Package** — build the artifact (§6).

## 2.3 The rules that make it work

**A red build is stopped work.** If main is broken, nobody's work is verifiable. Fixing it takes priority over everything, and if it isn't fixable in minutes, revert the commit that broke it.

**Nobody merges on red.** Enforce it with branch protection rather than convention.

**Fast enough that people wait for it.** Ten minutes is the outer limit for the PR feedback loop; past that, people context-switch and the loop breaks (§4.3).

**Flaky tests are treated as broken** (№44 §3.5) — a suite you don't trust is a suite you don't have.

---

# Part 3 — Continuous delivery vs deployment

The distinction people mix up:

| | **Continuous Delivery** | **Continuous Deployment** |
|---|---|---|
| Every change that passes the pipeline | is **ready** to deploy | **is** deployed |
| Production release | a human clicks a button | automatic |
| Gate | business decision | none |

**Continuous Delivery** is the near-universal goal: the pipeline produces a deployable artifact and deploying is a non-event, but *when* to release remains a choice. **Continuous Deployment** goes further and requires real confidence — comprehensive tests, feature flags, good observability, fast automated rollback.

For Practiq: continuous delivery is the right target. Automated everything up to a manual approval for production.

---

# Part 4 — Pipeline anatomy

## 4.1 The stages

```
Commit → Build → Test → Scan → Package → Deploy(dev) → Test(smoke) → Deploy(prod)
   │        │       │      │       │           │             │             │
 trigger  compile  unit  security image     automatic    verify      approval
                   +int   +deps   → ECR                                gate
```

## 4.2 Fail fast, cheapest first

Order stages by **cost and speed ascending** so failures surface early: lint (seconds) → compile → unit tests → integration tests → security scan → build image → deploy. There's no point spending four minutes building a container image for a commit that fails linting.

## 4.3 Keeping it fast

**Cache dependencies** (Gradle caches, Docker layer caching — №50 §7.2). **Parallelise** independent jobs. **Tier the tests** — unit on every push, integration on PR, slow E2E on merge or nightly (№44 §2). **Only build what changed** in a monorepo. **Right-size runners** — CI time is cheap relative to engineer waiting time.

## 4.4 Security in the pipeline

Cheap, high-value automation: **dependency scanning** (Dependabot, Snyk, OWASP Dependency-Check) for known CVEs — Log4Shell was a logging library; **SAST** for code-level issues; **container image scanning** (Trivy, ECR scan on push); **IaC scanning** (tfsec/checkov, №55 §8); and **secret scanning** so credentials never land in the repo. Each is a few lines of pipeline config and catches a class of problem you'd otherwise find in production.

---

# Part 5 — GitHub Actions concretely

## 5.1 The model

**Workflows** (YAML in `.github/workflows/`) contain **jobs**, which run on **runners** and contain **steps**. Jobs run in parallel by default; `needs:` sequences them.

```yaml
name: CI

on:
  push:
    branches: [main]
  pull_request:

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-java@v4
        with:
          java-version: '21'
          distribution: 'temurin'
          cache: 'gradle'                      # dependency caching

      - name: Build and test
        run: ./gradlew build                   # unit + integration (Testcontainers works on the runner)

      - name: Upload test results
        if: always()                           # even when the build fails — you want the report
        uses: actions/upload-artifact@v4
        with:
          name: test-results
          path: build/reports/tests/
```

## 5.2 A deployment job

```yaml
  deploy:
    needs: build                               # only if build passed
    if: github.ref == 'refs/heads/main'        # only on main
    runs-on: ubuntu-latest
    environment: production                    # GitHub environment → approval gate + secrets
    permissions:
      id-token: write                          # required for OIDC
      contents: read
    steps:
      - uses: actions/checkout@v4

      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::123456789012:role/practiq-deploy
          aws-region: eu-west-2                # ← OIDC: no stored access keys

      - uses: aws-actions/amazon-ecr-login@v2

      - name: Build and push image
        run: |
          docker build -t $ECR_REGISTRY/practiq-api:${{ github.sha }} .
          docker push $ECR_REGISTRY/practiq-api:${{ github.sha }}

      - name: Deploy to ECS
        run: aws ecs update-service --cluster practiq --service api --force-new-deployment
```

## 5.3 The details that matter

**OIDC over stored keys.** `id-token: write` plus `role-to-assume` lets the runner assume an IAM role via short-lived tokens — no long-lived AWS access keys in GitHub secrets (№54 §3.1). This is the single most important security improvement available in a CI pipeline.

**Tag images with the commit SHA**, not `latest`. It makes every deployment traceable to an exact commit and makes rollback a matter of redeploying a known digest (№50 §3.1).

**Least-privilege `permissions:`** on each job — the default token is broader than most jobs need.

**Pin third-party actions** (ideally to a commit SHA, not a moving tag) — an action you don't control runs with access to your secrets.

**Environments** give you approval gates, environment-scoped secrets, and a deployment history.

---

# Part 6 — Artifacts and registries

**Build once, promote the same artifact.** The image tested in dev must be the *identical* image deployed to production — same digest, no rebuild. Rebuilding per environment reintroduces the variance the whole system exists to eliminate.

What differs between environments is **configuration injected at runtime**, never the artifact (§7).

Registries: **ECR** for containers (№54, №91), plus package registries for libraries (Maven Central, npm, CodeArtifact). Practices: immutable tags (a tag can't be repointed), lifecycle policies to expire old images, scan on push, and retain a few known-good versions for rollback.

---

# Part 7 — Environments, config and secrets

## 7.1 The environment ladder

```
dev  →  staging  →  production
```

Each stage increases confidence. **Staging should resemble production** in shape (same infrastructure definitions from the same modules, №55 §7) even if smaller — a staging environment structurally unlike production tests very little.

Promotion means moving *the same artifact* forward, gated by tests and (for production) approval.

## 7.2 Configuration

The **twelve-factor** rule: **strict separation of config from code.** Anything that varies between environments — endpoints, credentials, feature toggles, pool sizes — comes from the environment, not the build.

```
DATASOURCE_URL=jdbc:postgresql://practiq-prod.xxxx.eu-west-2.rds.amazonaws.com:5432/practiq
JAVA_OPTS=-XX:MaxRAMPercentage=75
```

This is what makes "build once, deploy anywhere" possible.

## 7.3 Secrets

Never in the repository, never in an image layer (№50 §7.6), never in logs. The chain: stored in a secret manager (AWS Secrets Manager / SSM Parameter Store, №54 §11) → injected into the container as environment variables or mounted at runtime → the CI system holds only the credential needed to *fetch* them, or better, uses OIDC so it holds nothing.

**A secret committed to git is compromised** even if the next commit removes it (№53 §9.6). Rotate it; don't just delete it.

---

# Part 8 — Deployment strategies

How new code replaces old, and what happens when it's wrong.

| Strategy | Mechanism | Downtime | Cost | Rollback |
|---|---|---|---|---|
| **Recreate** | stop old, start new | **yes** | none | redeploy old |
| **Rolling** | replace instances in batches | no | none | roll back through |
| **Blue/Green** | full parallel environment, switch traffic | no | **2×** temporarily | **instant** — switch back |
| **Canary** | small % of traffic to new, then increase | no | small | shift traffic back |
| **A/B** | route by user attribute | no | small | route back |

**Rolling** is the default for container orchestrators (ECS, Kubernetes): replace tasks a few at a time, with health checks gating each batch. Simple and needs no extra infrastructure, but there's a window where both versions serve traffic — which means **your application must tolerate two versions running simultaneously** (§10).

**Blue/Green** gives the fastest, cleanest rollback: production traffic switches between two complete environments at the load balancer, and reverting is switching back. Costs double resources during the transition.

**Canary** gives the best *risk* profile: 5% of traffic sees the new version while you watch error rates and latency; promote on health, roll back on regression. It's the strategy that catches problems tests don't — real traffic, real data, small blast radius. Needs traffic-shifting infrastructure and good metrics (№57) to be meaningful.

**Automated rollback** is what turns these from ceremony into safety: wire the deployment to a CloudWatch alarm on error rate, so a bad canary reverts itself without a human noticing (№54 §9.3).

---

# Part 9 — Feature flags

The technique that unlocks everything else: **decouple deployment from release.**

Deploy code with the new behaviour switched off; turn it on later, for a subset of users, without deploying anything. That means:

- **Incomplete work can merge to main** — no long-lived branches, so no integration hell (§2.1).
- **Release is a runtime decision**, not a deployment event.
- **Instant "rollback"** — flip the flag, no redeploy.
- **Progressive rollout** — 1% of users, then 10%, then everyone.

```java
if (features.isEnabled("adaptive-question-selection", userId)) {
    return adaptiveSelector.next(userId);
}
return sequentialSelector.next(userId);
```

The cost is real and worth naming: **flags are technical debt with a shelf life.** Every flag doubles a code path, and a codebase with fifty stale flags is untestable. Track them, set expiry, and delete them once the feature is fully rolled out. A flag that has been at 100% for three months should be removed.

---

# Part 10 — Databases in a pipeline

The hardest part of continuous deployment, because you can't roll back data the way you roll back code.

**Migrations run automatically** as part of deployment (Flyway/Liquibase — versioned, ordered, checksummed, №20). But rolling deployments mean **old and new application versions run simultaneously**, so:

**Migrations must be backward compatible.** The schema must work with the *currently running* version and the *incoming* one. Which leads to the **expand/contract** (parallel change) pattern:

1. **Expand** — add the new column, nullable. Old code ignores it; new code can use it.
2. **Migrate** — deploy code that writes both old and new; backfill existing rows.
3. **Contract** — once all instances are on the new version and the data is backfilled, deploy code that only uses the new column, then drop the old one.

Three deployments where a naive rename would be one — but no downtime and no broken window. The naive `ALTER TABLE ... RENAME COLUMN` breaks every running instance of the old version instantly.

Other rules: **never destructive in the same release as the code change** that stops using a column; **test migrations against production-like data volumes** (a migration that takes 40ms on 100 rows may lock a table for minutes on 10 million); and **have a rollback plan** for the data, not just the code.

---

# Part 11 — Measuring it: DORA

Four metrics, from the DORA research, that predict both delivery performance and organisational outcomes:

| Metric | Measures | Elite |
|---|---|---|
| **Deployment frequency** | how often you ship | on demand, multiple per day |
| **Lead time for changes** | commit → running in production | under an hour |
| **Change failure rate** | % of deploys causing degradation | 0–15% |
| **Time to restore service** | how fast you recover | under an hour |

The finding that matters: **speed and stability are not a trade-off.** Teams that deploy frequently *also* have lower failure rates and faster recovery, because small batches, automation and fast feedback improve both. The intuition that "we ship slowly because we're careful" is not supported — slow shipping usually means large batches, which means riskier releases.

The pair worth internalising: **time to restore beats change failure rate.** You cannot prevent all failures; you can make recovery fast and routine. That's why rollback, feature flags and observability matter more than an extra approval gate.

---

# Part 12 — When to use what

**A. Continuous delivery or deployment?** Tell → deployment: strong test coverage, feature flags, good observability, fast rollback. Tell → delivery: anything less, or a business reason to control release timing. Default: **delivery — automate everything, gate production on a human.**

**B. Which deployment strategy?** Tell → rolling: the sensible default for containers. Tell → blue/green: you need instant rollback and can afford double resources. Tell → canary: high-risk changes, and you have the metrics to judge health. Default: **rolling, with canary for risky releases.**

**C. Feature flag or branch?** Tell → flag: work spanning more than a few days, or you want progressive rollout. Tell → branch: small change landing within a day or two. Default: **short branches; flags for anything longer** (№53 §11.F).

**D. Where do tests run?** Tell → every push: unit + fast integration. Tell → PR: full integration. Tell → merge/nightly: slow E2E, load, full security scans. Default: **tier by speed so the PR loop stays under ten minutes.**

**E. Monorepo or many repos?** Tell → mono: shared code, atomic cross-project changes, one pipeline to maintain. Tell → many: independent lifecycles and ownership. Default (Practiq): **separate repos per service is fine given the documented split, with shared modules versioned.**

**F. Rollback or roll forward?** Tell → rollback: the failure is significant and the previous version is known good. Tell → forward: the fix is trivial and verified, or a migration makes rollback unsafe. Default: **roll back first, diagnose after** — restore service, then investigate (№57).

**G. Manual approval gate?** Tell → yes: production, irreversible changes, regulated environments. Tell → no: dev and staging — a gate nobody thinks about is theatre. Default: **automated to staging, approval to production.**

---

# How to expand this

- *Related:* №53 Git (branches and PRs as the pipeline's triggers), №50 Docker (the artifact), №55 IaC (infrastructure through the same pipeline), №57 Observability (how you know a deploy went well), №44 Testing (what the pipeline actually runs), №54 §9 (the AWS deployment services).
- *Candidates for deeper treatment:* **a complete Practiq pipeline** — GitHub Actions → build → Testcontainers → ECR → Fargate, with the full YAML; **zero-downtime database migrations** worked through with real Flyway examples; **progressive delivery** (canary analysis, automated rollback on SLO breach); **supply-chain security** (SBOMs, signed images, provenance).

*Stable practice, written from knowledge — CI/CD principles, deployment strategies, expand/contract and DORA don't drift. GitHub Actions syntax and action versions change; check the current documentation for exact YAML.*
