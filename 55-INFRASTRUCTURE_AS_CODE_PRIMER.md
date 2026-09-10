# Infrastructure as Code — A Primer №55

*Defining your infrastructure in version-controlled code rather than clicking around a console. Terraform/OpenTofu-centred because that's the dominant model and what Practiq uses, but the concepts transfer. Companion to №54 (what the AWS services are) and №56 (how this runs in a pipeline).*

The problem being solved, stated plainly: **infrastructure created by clicking cannot be reviewed, reproduced, or reliably recovered.** Nobody remembers which of forty settings was changed at 11pm eight months ago. Environments drift apart until "works in staging" stops meaning anything. Rebuilding after a disaster becomes archaeology. IaC applies the practices you already take for granted in application code — version control, review, repeatability, testing — to the infrastructure the code runs on.

The single idea that makes Terraform make sense: **you describe the desired end state, and the tool works out how to get there.** You don't write "create a VPC, then a subnet, then a route table." You write "these things exist," and Terraform compares that against reality and computes the difference. Everything distinctive about it — the plan/apply split, the state file, the dependency graph — follows from that declarative model.

Contents:

- **Part 1** — why IaC, and what it replaces
- **Part 2** — declarative vs imperative
- **Part 3** — the Terraform model
- **Part 4** — state: the hard part
- **Part 5** — the language
- **Part 6** — modules
- **Part 7** — environments
- **Part 8** — testing and validation
- **Part 9** — IaC in a pipeline
- **Part 10** — the landscape and the licensing situation
- **Part 11** — practices, gotchas, and when to use what

## Problem index

| Situation | Concept | §|
|---|---|---|
| Someone changed something in the console | drift | §4.4 |
| Two people applied at once and broke state | state locking | §4.3 |
| I need to manage an existing resource | `import` | §4.5 |
| A change would destroy my database | `prevent_destroy`, read the plan | §11.2 |
| Secrets ended up in state | state is sensitive; use a secret manager | §4.6 |
| Repeating the same block for dev/staging/prod | modules + variables | §6, §7 |
| Terraform wants to replace something unexpectedly | forces-replacement attributes | §11.2 |
| I need one resource before another | implicit dependencies / `depends_on` | §3.5 |

---

# Part 1 — Why IaC

## 1.1 What it replaces

**ClickOps** — creating infrastructure through a web console. It works, and it doesn't scale past one environment or one person. The failures are predictable: no record of *what* was configured or *why*; no way to recreate it exactly; environments that diverge silently; no review before a change that takes production down; and total dependence on whoever remembers the setup.

The next stage up — **shell scripts calling the CLI** — is better (it's code) but imperative: it describes *steps*, not *state*, so re-running it is dangerous, partial failures leave you in an unknown position, and it can't tell you what would change before it changes it.

## 1.2 What IaC gives you

- **Reproducibility** — stand up an identical environment from the same code. Staging is genuinely like production because it came from the same definition.
- **Version control** — every change is a reviewable diff with an author, a date, and a message. A bad change is a revert (№53).
- **Documentation that can't drift** — the code *is* the description of what exists.
- **Idempotence** — apply it repeatedly and converge on the same result. This is what makes it safe to run continuously.
- **Disaster recovery** — rebuilding is running the code, not remembering the clicks.
- **Review and automation** — infrastructure changes go through the same PR process as application changes, and can run in CI (§9).

> **The tell — IaC:** if a resource was created by clicking, it doesn't really exist as far as your team's knowledge is concerned. The console is for *looking*; code is for *changing*.

---

# Part 2 — Declarative vs imperative

| | **Imperative** (scripts, CLI, Ansible-ish) | **Declarative** (Terraform, CloudFormation) |
|---|---|---|
| You write | the steps to take | the end state you want |
| Re-running | may duplicate or fail | converges (idempotent) |
| Partial failure | leaves an unknown state | re-apply to reconcile |
| Preview | hard | **built in** (`plan`) |
| Handling drift | invisible | detected on the next plan |

Declarative wins for provisioning because **the tool owns the diff**. You state the destination; it works out whether that means creating, updating, replacing or destroying, and in what order.

The trade-off is real: you give up fine control of *how*, and when the tool's model of a resource doesn't match reality you have less recourse. There's also a genuine split of concerns — **Terraform provisions infrastructure; Ansible configures machines.** Use the right one, or use both (Terraform creates the instance, Ansible or user-data configures it), though in a container world the second half mostly disappears into the image (№50).

---

# Part 3 — The Terraform model

## 3.1 Providers

A **provider** is a plugin that talks to a platform's API — AWS, Google, Azure, Kubernetes, GitHub, Postgres, Datadog. There are thousands. Providers translate Terraform's generic model into API calls.

```hcl
terraform {
  required_version = ">= 1.6"
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"          # pin! ~> 5.0 allows 5.x, not 6.0
    }
  }
}

provider "aws" {
  region = "eu-west-2"
}
```

**Pin provider versions.** An unpinned provider will one day upgrade itself mid-pipeline and produce a plan you didn't expect.

## 3.2 Resources

The things you're managing. Each has a **type** and a **local name**, and together those form its address (`aws_db_instance.practiq`).

```hcl
resource "aws_db_instance" "practiq" {
  identifier             = "practiq-${var.environment}"
  engine                 = "postgres"
  engine_version         = "16.3"
  instance_class         = var.db_instance_class
  allocated_storage      = 20
  db_name                = "practiq"
  username               = "practiq_admin"
  password               = var.db_password        # from a secret store, never literal
  vpc_security_group_ids = [aws_security_group.db.id]
  db_subnet_group_name   = aws_db_subnet_group.private.name
  multi_az               = var.environment == "prod"
  backup_retention_period = 7
  skip_final_snapshot    = var.environment != "prod"

  lifecycle {
    prevent_destroy = true        # refuse to destroy this, ever
  }
}
```

## 3.3 Data sources

Read-only lookups of things Terraform doesn't manage — an existing VPC, the latest AMI, the current account ID:

```hcl
data "aws_ami" "amazon_linux" {
  most_recent = true
  owners      = ["amazon"]
  filter {
    name   = "name"
    values = ["al2023-ami-*-x86_64"]
  }
}
```

Note `most_recent = true` makes your configuration non-deterministic — the AMI can change under you between plans. Pin it if reproducibility matters more than currency.

## 3.4 The workflow

```bash
terraform init          # download providers and modules, configure the backend
terraform fmt           # canonical formatting
terraform validate      # syntax and internal consistency
terraform plan          # ← show me what will change. READ THIS.
terraform apply         # make it so
terraform destroy       # tear it all down
```

**The `plan`/`apply` split is the safety feature**, and the discipline that matters most. `plan` produces a diff: `+` create, `~` update in place, `-/+` **destroy and recreate**, `-` destroy. The `-/+` is the one that ends careers — an innocuous-looking attribute change that forces replacement of a database. Read the plan. Every time.

For pipelines, save the plan and apply exactly that: `terraform plan -out=tfplan` then `terraform apply tfplan`, so what's approved is what runs.

## 3.5 The dependency graph

Terraform builds a DAG from the references between resources and parallelises what it can. Dependencies are usually **implicit** — because `aws_db_instance.practiq` references `aws_security_group.db.id`, Terraform knows the security group must exist first. That's the idiomatic way; use explicit `depends_on` only for dependencies that aren't expressed through a reference (an IAM policy that must exist before a service can assume a role, say).

---

# Part 4 — State: the hard part

Everything difficult about Terraform is state. Understand this and the rest is syntax.

## 4.1 What state is and why it exists

Terraform records what it created in a **state file** (`terraform.tfstate`), mapping your configuration addresses to real-world resource IDs:

```
aws_db_instance.practiq  →  db-ABC123XYZ in eu-west-2
```

Why it's necessary: on the next run Terraform must answer "does this already exist, and does it match?" It could theoretically ask the API to list everything, but it wouldn't know which resources are *its* responsibility, and mapping configuration to reality would be ambiguous. State is the record of ownership and the last-known attribute values, which is also what makes fast plans possible.

## 4.2 Remote state

**Never keep state on a laptop, and never commit it to git.** It's a shared source of truth and it contains secrets (§4.6). Use a remote backend:

```hcl
terraform {
  backend "s3" {
    bucket       = "practiq-terraform-state"
    key          = "prod/terraform.tfstate"
    region       = "eu-west-2"
    encrypt      = true
    use_lockfile = true       # S3-native locking (modern); older setups used a DynamoDB table
  }
}
```

Enable **versioning** on the state bucket — it's your undo for a corrupted state file, and you will eventually want it.

## 4.3 Locking

Two simultaneous applies against one state file will corrupt it. Backends therefore **lock**: one apply at a time, others wait or fail. This is essential the moment more than one human or pipeline can run Terraform. (If a run is killed mid-apply the lock can be left behind; `terraform force-unlock <id>` clears it — after you've confirmed nothing is actually running.)

## 4.4 Drift

**Drift** is reality diverging from state — someone changed a security group in the console, or a service modified something itself. `terraform plan` detects it by refreshing state against the real API, and shows it as a difference. Applying will *revert* the manual change, which is usually what you want and occasionally a nasty surprise.

The cultural fix matters more than the technical one: **once a resource is managed by Terraform, stop changing it in the console.** Where drift is unavoidable (a service that mutates its own tags), `lifecycle { ignore_changes = [tags] }` tells Terraform to leave that attribute alone.

## 4.5 Import, move, and surgery

```bash
terraform import aws_db_instance.practiq db-ABC123    # adopt an existing resource
terraform state list                                   # what's tracked
terraform state show aws_db_instance.practiq
terraform state mv <old-address> <new-address>         # rename without destroy/recreate
terraform state rm <address>                           # stop managing (leaves it alive)
```

**`state mv` is the one that saves you**: renaming a resource in your code makes Terraform see it as "destroy the old, create a new" — `state mv` tells it that it's the same thing under a new name. Modern Terraform also supports declarative `import` and `moved` blocks in configuration, which are safer because they go through the plan.

**Never hand-edit the state file.** If you think you need to, you need `state mv`/`rm`/`import` instead.

## 4.6 State contains secrets

Resource attributes are stored in state in **plaintext** — including database passwords, generated keys and certificates. Consequences: encrypt the state bucket, restrict access to it as tightly as production credentials, and never commit it. Better still, **don't put secrets in Terraform at all**: have it create a Secrets Manager entry with a random value the application reads at runtime, rather than passing a password through the configuration (№54 §11). OpenTofu added client-side state encryption specifically to address this (§10).

---

# Part 5 — The language

## 5.1 Variables, locals, outputs

```hcl
variable "environment" {
  description = "Deployment environment"
  type        = string
  validation {
    condition     = contains(["dev", "staging", "prod"], var.environment)
    error_message = "environment must be dev, staging or prod."
  }
}

variable "db_instance_class" {
  type    = string
  default = "db.t4g.micro"
}

locals {                                   # computed values, DRY within a module
  name_prefix = "practiq-${var.environment}"
  common_tags = {
    Project     = "practiq"
    Environment = var.environment
    ManagedBy   = "terraform"
  }
}

output "db_endpoint" {
  value       = aws_db_instance.practiq.endpoint
  description = "Connection endpoint for the database"
}
```

**Variables** are inputs; **locals** are internal computed values; **outputs** are what a module exposes (and what other configurations can consume). Mark sensitive outputs `sensitive = true` so they're redacted from logs — though note they're still in state.

## 5.2 Expressions, loops, conditionals

```hcl
count = var.environment == "prod" ? 3 : 1               # conditional

resource "aws_subnet" "private" {                        # for_each over a map — preferred
  for_each          = var.private_subnets
  vpc_id            = aws_vpc.main.id
  cidr_block        = each.value.cidr
  availability_zone = each.value.az
  tags              = merge(local.common_tags, { Name = "${local.name_prefix}-${each.key}" })
}
```

**Prefer `for_each` over `count`.** With `count`, resources are tracked by *index* — removing the middle item of a list shifts everything after it, and Terraform destroys and recreates resources that didn't change. `for_each` keys by a stable string, so additions and removals affect only the item concerned. This is one of the most common causes of alarming plans.

Useful functions: `merge`, `lookup`, `try`, `coalesce`, `join`, `split`, `file`, `templatefile`, `jsonencode`, `cidrsubnet` (calculates subnet CIDRs from a VPC block — genuinely handy, №51 §2.3).

---

# Part 6 — Modules

A **module** is a reusable, parameterised group of resources — the function of Terraform.

```
modules/
  network/          main.tf  variables.tf  outputs.tf
  database/
  ecs-service/
environments/
  dev/              main.tf  terraform.tfvars
  prod/
```

```hcl
module "network" {
  source      = "../../modules/network"
  environment = var.environment
  vpc_cidr    = "10.0.0.0/16"
}

module "database" {
  source            = "../../modules/database"
  environment       = var.environment
  subnet_ids        = module.network.private_subnet_ids   # module composition
  instance_class    = var.db_instance_class
}
```

Design guidance: a module should have a **clear single purpose**, a **small interface** (many required variables is a smell, same as a long parameter list — №40 §3.2), and sensible defaults. Version modules from a registry or git tag (`source = "git::...?ref=v1.2.0"`) so a module change doesn't silently alter every consumer.

**Don't over-modularise.** A module wrapping a single resource with no added logic is indirection for nothing (№41 §7). Modules earn their place when they encapsulate a *pattern* — "a service with its load balancer, target group, task definition, security groups and log group."

---

# Part 7 — Environments

Two approaches, and the choice matters:

**Workspaces** — one configuration, multiple named states (`terraform workspace new prod`). Cheap, but everything shares a code path, it's easy to apply to the wrong workspace, and environments that genuinely differ end up riddled with conditionals.

**Separate directories per environment** — each with its own backend key and `.tfvars`, calling shared modules. More files, but **explicit**: prod's configuration is a file you can read, blast radius is contained, and environments can differ deliberately (prod is multi-AZ, dev isn't).

> **The tell — environments:** use **directories per environment with shared modules**. Workspaces suit short-lived, structurally-identical stacks (per-developer sandboxes, ephemeral PR environments); they're a poor fit for dev/staging/prod, where the point is that prod is *different* and *more careful*.

Also separate **state per component**, not one enormous state for everything: network, data, application. Smaller states mean faster plans, smaller blast radius, and independent change cadence — the network changes yearly, the service weekly. Cross-reference with `terraform_remote_state` data sources or by passing values explicitly.

---

# Part 8 — Testing and validation

The ladder, cheapest first:

- **`terraform fmt -check`** and **`terraform validate`** — formatting and syntax. Fast, run on every commit.
- **Linting** — `tflint` catches provider-specific mistakes (invalid instance types, deprecated arguments).
- **Security scanning** — `tfsec`, `checkov`, or `Trivy` flag public S3 buckets, unencrypted volumes, wide-open security groups. **High value for very little effort**; put it in CI.
- **Policy as code** — OPA/Conftest or Sentinel enforce organisational rules ("no resource without a Project tag," "no publicly-readable buckets") as pipeline gates.
- **Plan review** — the human step, and still the most important one.
- **Integration testing** — Terratest (Go) or the native `terraform test` framework actually provision into a sandbox, assert, and destroy. Slow and costs real money; reserve it for shared modules.

---

# Part 9 — IaC in a pipeline

The standard flow, and it maps onto normal code review (№56):

**On pull request:** `fmt -check` → `validate` → `tflint` → `tfsec` → `plan`, with the plan **posted as a PR comment** so the reviewer sees exactly what will change. This is the single highest-value automation in IaC — infrastructure diffs become reviewable artifacts.

**On merge to main:** `apply` the saved plan, gated by a manual approval for production.

Practical requirements: the pipeline authenticates via **OIDC to an IAM role** rather than long-lived access keys (№54 §3.1); state locking prevents concurrent applies (§4.3); and plan output should be checked for destructive changes before approval.

---

# Part 10 — The landscape

## 10.1 The licensing situation (verified July 2026)

Worth knowing precisely, because most older material predates it:

- **August 2023** — HashiCorp relicensed Terraform from the permissive MPL to the **Business Source License (BSL)**: source-available, but restricting use in competing products.
- The community forked the last MPL version as **OpenTofu**, now a **Linux Foundation / CNCF** project with open governance.
- **Early 2025** — **IBM acquired HashiCorp**, so Terraform is now an IBM product.
- By 2026 the two have **meaningfully diverged** — OpenTofu added state encryption and provider-defined functions; Terraform added Stacks and deeper HCP integration — while remaining broadly command- and configuration-compatible.

**For normal internal use, the licence changes nothing.** It matters to vendors building products on Terraform, and to organisations wanting neutral governance. For Practiq, either works and the model above is identical; OpenTofu is a drop-in for most configurations if you prefer the open-governance option.

## 10.2 The alternatives

| Tool | Approach | Choose when |
|---|---|---|
| **Terraform / OpenTofu** | declarative HCL, multi-cloud, huge provider ecosystem | the default for cloud provisioning |
| **CloudFormation** | declarative YAML/JSON, AWS-only, no state file to manage (AWS holds it) | all-in on AWS, want native drift detection and rollback |
| **AWS CDK** | real languages compiled to CloudFormation | developer-heavy teams wanting types, loops, abstractions |
| **Pulumi** | real languages, own engine, multi-cloud | same, but not AWS-locked |
| **Ansible** | imperative-ish, agentless config management | **configuring** machines, not provisioning them |
| **Crossplane** | Kubernetes CRDs manage cloud resources | you're already k8s-native |

The CDK/Pulumi trade is worth naming: a real programming language gives you abstraction and type-checking, and also lets you generate a great deal of infrastructure from a small amount of code — which cuts both ways when the plan comes back with 400 changes.

---

# Part 11 — Practices, gotchas, and decisions

## 11.1 Practices

**Pin everything** — Terraform version, provider versions, module versions. **Remote state, encrypted, versioned, locked.** **Small states** by component. **Tag everything** via a common local, so cost allocation and ownership work (№54 §12). **Never commit** `.tfstate`, `.tfvars` containing secrets, or `.terraform/`. **Read every plan.** **`prevent_destroy`** on databases and anything holding data. **Review as code** — infrastructure PRs get the same scrutiny as application PRs.

## 11.2 The gotchas

**Attributes that force replacement.** Changing certain fields (an RDS `identifier`, an EC2 `availability_zone`) can't be done in place, so Terraform plans a destroy-and-recreate. On a database that's data loss. This is why you read the plan and why `prevent_destroy` exists.

**`count` index shifting** (§5.2) — use `for_each`.

**Secrets in state** (§4.6) — assume state is as sensitive as production credentials.

**Terraform is not a deployment tool.** It provisions infrastructure well; using it to deploy application versions (rebuilding a task definition on every release) couples your release cadence to your infrastructure tooling. Let CI/CD handle deployments (№56); let Terraform own the infrastructure they run on.

**Drift from ClickOps** — the cultural failure that undermines everything (§4.4).

**Provider bugs and lag** — providers occasionally don't cover a new service feature, or model a resource imperfectly. Sometimes the honest answer is a small imperative escape hatch, deliberately isolated.

## 11.3 When to use what

**A. IaC or console?** Tell → IaC: anything that lives beyond an experiment. Tell → console: exploring, reading, one-off investigation. Default: **console to look, code to change.**

**B. Terraform or CloudFormation?** Tell → Terraform/OpenTofu: multi-cloud, better ecosystem, you want the tooling. Tell → CloudFormation: AWS-only shop, want AWS-managed state and native drift/rollback. Default: **Terraform, for the ecosystem and portability of skill.**

**C. Terraform or OpenTofu?** Tell → Terraform: you want HashiCorp/IBM's ecosystem, HCP features, or your organisation already uses it. Tell → OpenTofu: you want open governance, permissive licensing, or state encryption. Default (for Practiq): **either — pick one, note the decision, move on.**

**D. Workspaces or directories?** Tell → workspaces: short-lived, structurally identical stacks. Tell → directories: real environments that differ. Default: **directories per environment, shared modules.**

**E. One state or many?** Tell → one: a small system. Tell → many: separate lifecycles, blast-radius containment, slow plans. Default: **split by component once you have more than a handful of resources.**

**F. Module or inline?** Tell → module: the pattern repeats, or it encapsulates a meaningful unit. Tell → inline: used once, no abstraction earned. Default: **inline first; extract a module on the second use** (№40 §11.B).

**G. Terraform or Ansible?** Tell → Terraform: creating cloud resources. Tell → Ansible: configuring an existing machine's software and files. Default: **Terraform provisions, containers configure** (№50) — Ansible mostly disappears in a container world.

---

# How to expand this

- *Related:* №54 AWS (the services you're provisioning), №91 (how each one actually works), №56 CI/CD (running IaC in a pipeline), №50 Docker (the artifact your infrastructure runs), №42 §8.2 (ADRs — infrastructure decisions deserve them too).
- *Candidates for deeper treatment:* **a complete Practiq Terraform layout** — VPC, subnets, ALB, ECS/Fargate service, RDS, ECR, IAM roles, with modules and environments; **the state-surgery playbook** (import, move, rm, recovering a corrupted state); **testing infrastructure properly** with Terratest and policy-as-code.

*Concepts and workflow are stable. The licensing and fork situation in §10.1 was verified July 2026 because it changed materially and most published material predates it. Provider resource arguments change often — the registry documentation is authoritative.*
