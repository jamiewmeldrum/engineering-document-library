# Docker — A Primer

*A working engineer's mental model for containers, with a Practiq lens. Concepts here are stable; where the current tooling landscape matters, it's aligned to the 2026 state (Docker Engine ~29, BuildKit as the default builder, the containerd/runc stack) and the ecosystem-currency claims in Parts 5–6 are verified against current sources. Practiq touches Docker in four places — **Testcontainers** (real Postgres in integration tests), **GitHub Actions** (CI builds), local dev, and **AWS** deployment (ECR/ECS-Fargate) — and the documented service split (`practiq-api`, `practiq-processor`, `practiq-extractor`, `practiq-frontend`, `practiq-infrastructure`) is really "five things that each become an image." The examples lean on that.*

The aim, as with the data-access primer: stop using Docker by rote. By the end you should be able to point at any part of the system — a layer, a mount, a Dockerfile line, a `docker run` flag — and say **what it is, what the kernel is actually doing, and why you'd choose it over the alternative.**

The one idea that unlocks everything else: **a container is not a small virtual machine. It is one ordinary Linux process that the kernel has been told to lie to** — about what it can see (namespaces), what it can use (cgroups), and what its disk looks like (a stacked filesystem). Hold that, and the rest is detail.

Contents:

- **Part 1** — what Docker is, and the problem it solves (and container vs VM, image vs container)
- **Part 2** — how it works under the hood: namespaces, cgroups, the union filesystem, the runtime stack
- **Part 3** — images, layers, and the container's writable layer (the mechanics behind read-only vs writable)
- **Part 4** — file storage: volumes, bind mounts, tmpfs
- **Part 5** — why we use it, honestly (value and cost)
- **Part 6** — the alternatives, and when each is suitable
- **Part 7** — a practical Dockerfile guide: common use cases and how to build them
- **Part 8** — a worked trace: build, then run
- **Part 9** — *when to use what*: consolidated decision frameworks
- **How to expand this doc**

## Symptom index — open here when something's wrong

| Symptom | Go to |
|---|---|
| Changes/data vanished after container restart or `rm` | 3.3–3.4 (the writable layer is ephemeral) → mounts (4) |
| Image is enormous | 3.6 / 7.3 / 7.6 (multi-stage, base image, `.dockerignore`) |
| Rebuilds are slow; the cache never seems to hit | 7.2 (instruction ordering, `.dockerignore`) |
| "Works on my machine" but not in CI/prod | 1.1 (parity is the point); if arch-related, 7.6 (multi-arch) |
| Container exits immediately after start | 7.1 (ENTRYPOINT/CMD; the main process must run in the foreground) |
| Port works inside but not reachable from the host | 7.1 (`EXPOSE` documents; `-p` actually publishes) |
| "Permission denied" on mounted files / files owned by root | 2.2 (user namespaces) + 7.6 (run as non-root, uid mapping) |
| Two containers clobbering each other's data | 3.4 (separate writable layers) → shared volume (4.2) |
| A secret ended up baked into the image | 7.2 / 7.6 (ARG/ENV land in history; use build secrets) |
| Container eating all host CPU/RAM | 2.3 (cgroup limits: `--memory`, `--cpus`) |
| Alpine image: weird crashes, DNS oddities, native-lib failures | 7.6 (musl vs glibc — be wary of Alpine for the JVM/native code) |
| Testcontainers can't find Docker / fails in CI | 5 (it needs a working Docker daemon on the runner) |
| Can't reach one service from another | 7.5 (Compose networking — use service names, not localhost) |

---

# Part 1 — What Docker is, and the problem it solves

## 1.1 The problem

The disease Docker cures is **environment drift**: the app runs on your laptop and dies in CI, or runs in CI and dies in production, because the machines differ in ways nobody wrote down — a different Java point-release, a missing native library, a different `libc`, an env var set in one place and not another. "Works on my machine" is not a joke; it's a description of un-reproducible environments.

Docker's fix is to **package the application together with everything it needs to run** — the runtime, the libraries, the filesystem, the config — into a single immutable artifact (an *image*) that runs the same way on any host with a container runtime. The unit of deployment stops being "code plus a wiki page of setup steps" and becomes "one artifact." Parity across dev, CI, and prod stops being aspirational and becomes the default. That parity is *exactly* why Practiq runs real Postgres in Testcontainers rather than an in-memory stand-in (the dialect-drift problem from the data-access primer): the test database is the same image as production.

## 1.2 What a container actually is

A container is **an isolated process (or process tree) running on the host, sharing the host's kernel**, but given its own private view of the system. It has its own filesystem, its own network stack, its own process table — but there is no second operating system. The `java` process in your `practiq-api` container is a normal process in the host's process list; the host kernel scheduled it. The isolation is a set of kernel features applied to that process, not a machine boundary. (Part 2 is the whole mechanism.)

## 1.3 Container vs virtual machine

This is the distinction people most often hold wrongly, and it drives most of the trade-offs.

| | **Virtual machine** | **Container** |
|---|---|---|
| What's virtualised | the *hardware* (via a hypervisor) | the *OS view* (via kernel features) |
| Runs | a full guest OS + kernel per VM | a process sharing the host kernel |
| Weight | gigabytes, its own kernel + userland | megabytes, just the app + its deps |
| Start time | tens of seconds (a boot) | milliseconds (a process starts) |
| Density | tens per host | hundreds/thousands per host |
| Isolation | strong (hardware boundary) | weaker (shared kernel; a kernel bug can cross it) |
| Best for | strong isolation, different OS/kernel, legacy | fast, dense, reproducible app deployment |

The headline: a VM boots an operating system; a container starts a process. That's why containers are fast and dense, and also why their isolation is weaker — everything shares one kernel, so a kernel-level escape is a real (if rare) concern in a way it isn't for VMs. (For untrusted code you can get VM-grade isolation back with sandboxed runtimes — 6.)

**One nuance that trips people up:** on macOS and Windows there is *no Linux kernel to share*, so Docker Desktop quietly runs a lightweight Linux VM and your containers run inside *that*. "Containers share the host kernel" is true — but on a Mac the "host" is the hidden VM, not macOS. This is why Docker feels heavier and file I/O across the mount boundary is slower on a Mac than on native Linux.

## 1.4 Image vs container — the noun pair everything hangs on

Two words, and confusing them is the root of most beginner trouble:

- An **image** is an immutable, read-only *template* — a packaged filesystem plus metadata (what to run, which env, which port). It's inert. It's the thing you build, tag, push to a registry, and pull.
- A **container** is a *running (or stopped) instance* of an image — the image brought to life as a process, with a thin writable layer of its own on top.

The exact analogy from the data-access primer works: **image is to container as a class is to an object**, or as an executable file on disk is to a process. One image → many containers, each an independent instance. This pair is also the doorway to Part 3's central question, because the *image* is read-only and the *container* adds something writable.

## 1.5 "Docker" the product vs the stack beneath it

"Docker" is used loosely for several things. Untangled, top to bottom:

- The **Docker CLI** (`docker`) — the command-line client you type into. It just sends API requests to…
- The **Docker daemon** (`dockerd`) — the long-running background service that manages images, containers, networks, volumes. This daemon (and its historically root privileges) is the thing several alternatives exist to avoid (6). Under it sits…
- **containerd** — the core container runtime that handles image pull, storage/snapshots, and the container lifecycle. It's a CNCF project, and it's what Kubernetes uses directly (Docker-as-a-k8s-runtime was removed in 2022). Below that…
- **runc** — the low-level OCI runtime that does the actual kernel work: create the namespaces and cgroups, then `exec` the process. Alternatives like `crun` slot in here.
- **OCI (Open Container Initiative)** — the standards (image spec + runtime spec) that make all of this interoperable, so an image built by Docker runs under Podman, containerd, or CRI-O without change.

So `docker run` is really: CLI → `dockerd` → `containerd` → a per-container shim → `runc` → the kernel. You'll almost never touch the lower layers directly, but knowing they're there explains the alternatives in Part 6 — most of them are "swap out one of these boxes."

---

# Part 2 — How it works under the hood

Here's the payoff of "a container is a process the kernel lies to." There are exactly three lies, and each is a distinct kernel feature.

## 2.1 The three mechanisms

| The lie | Kernel feature | Governs |
|---|---|---|
| "You're the only processes here" | **namespaces** | what the process can **see** |
| "This is all the CPU/RAM there is" | **cgroups** | what the process can **use** |
| "This is your whole disk" | **union filesystem** | what the process's **filesystem** looks like |

Everything Docker does at runtime is orchestrating these three. Learn them and the rest is packaging.

## 2.2 Namespaces — what a process can see

A Linux **namespace** partitions a global kernel resource so that processes inside see their own isolated instance of it. A container is a process placed into a fresh set of namespaces. The main ones:

- **mnt** — its own filesystem mount table. This is how a container has a completely different `/` (its image's filesystem) from the host.
- **pid** — its own process-ID space. The container's main process is **PID 1** inside, even though it's some large PID on the host. The container can't see host processes.
- **net** — its own network stack: interfaces, IP addresses, ports, routing table. Two containers can both "bind port 8080" because each has a private network namespace. Port publishing (`-p 8080:8080`) is the host reaching *into* that namespace.
- **uts** — its own hostname and domain name.
- **ipc** — its own inter-process communication (shared memory, semaphores).
- **user** — its own user/group ID mapping. Container UID 0 (root) can be mapped to an *unprivileged* host UID. This is the basis of **rootless** containers: "root" inside is nobody special outside.
- **cgroup** and **time** — its own view of the cgroup hierarchy and system clocks (newer additions).

> **The tell — namespaces:** when you wonder "how can the container have its own hostname / its own PID 1 / its own port 8080," the answer is always "a namespace." Isolation of *visibility* is namespaces; it is not a security wall so much as a set of blinkers. (True security hardening layers user namespaces, seccomp, and capabilities on top.)

## 2.3 cgroups — what a process can use

Namespaces limit what a process *sees*; **control groups (cgroups)** limit what it can *consume*. A cgroup caps and meters resources for a process group: CPU (shares and hard quotas), memory (a hard limit, past which the kernel's OOM killer terminates the process), process count (`pids`), and block-I/O bandwidth. Modern distros use **cgroups v2** (a single unified hierarchy).

This is what `docker run --memory=512m --cpus=1.5` actually sets. It matters in production and in CI: an unbounded container can starve its neighbours or the host. And the memory limit interacts with the JVM specifically — a Java process must be told to respect the container's memory limit (modern JVMs read cgroup limits automatically, but heap sizing against the *container* limit, not the *host's* RAM, is a real Practiq concern for `practiq-api`).

> **The tell — cgroups:** "the container ate all the host's RAM/CPU" or "my container got OOM-killed at 512 MB" is always a cgroup story. Limits are how you make one host safely run many containers.

## 2.4 The union filesystem — what its disk looks like

The third mechanism is where Part 3's question lives, so this is the setup. A container's filesystem is not one disk; it's a **stack of layers** presented as a single unified view by a *union (overlay) filesystem* — on Linux, **OverlayFS**. OverlayFS composes directories into one:

- **lowerdir** — one or more **read-only** layers, stacked. These are the image's layers.
- **upperdir** — a single **writable** layer on top. This is the container's own.
- **merged** — the unified view the container actually sees, as if it were one filesystem.
- **workdir** — internal scratch space OverlayFS needs to make changes atomic.

Reads search top-down and take the first match. Writes go to the upperdir — and here's the crucial rule: modifying a file that lives in a read-only lower layer triggers **copy-up** — the whole file is copied into the upperdir first, then modified there. The lower copy is untouched; the upper copy shadows it. That is **copy-on-write (CoW)**. Deleting a lower-layer file doesn't remove it (it's read-only) — instead a **whiteout** marker is written to the upperdir that hides it from the merged view. Every change a running container makes to its filesystem lands in that one upperdir, and nowhere else. Hold that thought.

## 2.5 The runtime stack in motion

Putting 2.2–2.4 together, `docker run practiq-api` roughly does: the CLI asks `dockerd`; `containerd` ensures the image's layers are present and prepares a snapshot (the read-only lowerdirs plus a fresh writable upperdir); a per-container shim is started; `runc` creates the namespaces and cgroups, `chroot`/pivots into the merged filesystem as `/`, and `exec`s your entrypoint process as PID 1 inside. From then on it's just a Linux process running under those three constraints, and the shim keeps it supervised independently of the daemon. (In recent Docker, the *containerd image store* is the default backing for this — worth knowing if you read about it, but it doesn't change the model.)

---

# Part 3 — Images, layers, and the container's writable layer

This is the heart of the document, and the section built to make the read-only-vs-writable distinction click. Read 2.4 first; this is that mechanism applied.

## 3.1 What an image actually is

An image is not a single blob. It is **an ordered stack of read-only layers, plus a config**:

- Each **layer** is a filesystem *diff* — the set of files added, changed, or removed relative to the layer below — stored as a compressed archive and named by the **digest** (a `sha256` hash) of its contents. Content-addressing means a layer's identity *is* its content.
- The **config** is a JSON document of metadata: the default command/entrypoint, env vars, working dir, exposed ports, the ordered list of layer digests, and a build history.
- A **manifest** ties the config and layers together by digest. A **tag** (`practiq-api:latest`) is just a human-friendly, *mutable* pointer at an *immutable* digest — which is why pinning by digest is more reproducible than pinning by tag.

## 3.2 Layers as diffs, shared and cached

Every image-building instruction that changes the filesystem produces a new layer stacked on the previous ones (7.1). Because layers are content-addressed, two properties fall out that matter enormously:

- **Sharing/dedup.** If ten images on a host share the same Debian base layer, that layer is stored **once**. Pulling an image downloads only the layers you don't already have. This is why images feel huge but pulls are often fast, and why a shared base across `practiq-api`, `practiq-extractor`, etc. saves real space.
- **Build cache.** During a build, an unchanged instruction with unchanged inputs reuses its existing layer instead of re-running (7.2). This is the entire reason build ordering matters.

## 3.3 The container's writable layer

Now `docker run`. On top of the image's read-only layers (the lowerdirs), Docker adds **one thin writable layer** for the container (the upperdir from 2.4). Everything the running container writes — new files, modified files (via copy-up), deletions (via whiteouts) — goes **there and only there**. The image's layers are never touched; they can't be, they're read-only and shared.

Three consequences follow directly from "it's a per-container upperdir":

- **It's private to that container.** Run the same image three times and you get three separate writable layers. Container A's writes are invisible to containers B and C, even though all three share the identical read-only base. (This is why "two containers clobbering each other's data" is a *shared-volume* problem, not a layer problem — 4.)
- **It's ephemeral.** `docker rm` deletes the writable layer. Everything written there is gone. The image is unchanged and can spawn a fresh container with a fresh, empty writable layer. Data you care about must not live here (4).
- **Writes are relatively expensive.** Because modifying a large file from a lower layer copies the whole file up first (CoW), write-heavy workloads against the container filesystem are slower than writing to a mount. Another reason data belongs on a volume.

## 3.4 Read-only layers vs the writable layer — the mechanism laid bare

You asked for the difference between an image layer and a running container's writable layer, and I've deliberately not handed you a one-liner — because the *mechanism* is the understanding, and once you hold the mechanism the distinction is obvious rather than memorised. So, side by side, the properties that fall out of OverlayFS:

| Property | Image layer(s) | Container's writable layer |
|---|---|---|
| OverlayFS role | **lowerdir** | **upperdir** |
| Mutability | **read-only** | **read-write** |
| Who has it | **shared** across all containers of that image (and across images) | **one per container**, private |
| Lifetime | lives in the image; persists, versioned, pushable | **created at `run`, destroyed at `rm`** — ephemeral |
| How it's made | at **build** time, by Dockerfile instructions | at **run** time, automatically |
| What lands in it | the packaged app, deps, filesystem | everything the process writes while running |
| Content-addressed? | yes (digest = identity, dedup, cache) | no — it's mutable scratch |
| Deletions of lower files | n/a (it *is* the lower file) | represented as **whiteout** markers that hide, not remove, the lower file |

If you want the sentence to *check yourself* against after you've read 2.4 and 3.3 — cover it and try to reconstruct it first — it's this: *an image layer is a read-only, content-addressed, shared filesystem diff baked at build time; the writable layer is the single, private, ephemeral copy-on-write layer added on top when the image runs, where every change goes and which dies with the container.* Everything in that sentence is something you can now derive from the overlay mechanism rather than take on faith.

## 3.5 Copy-on-write cost, in practice

CoW has a runtime cost worth internalising: the *first* write to a large file that lives in a lower layer pays to copy the entire file up. So a container that appends to a big log file, or a database writing its data files, into the *container filesystem* is doing the slow thing and filling an ephemeral layer. Databases and anything write-heavy or persistent → a mount (4), not the writable layer. This is also why "store your data in the container" is an anti-pattern with two separate failure modes: it's slow *and* it's lost on `rm`.

## 3.6 Image size and layer optimisation

Because layers stack and each instruction adds one, image bloat is a layering problem with a few standard fixes (expanded with examples in 7.3/7.6): use a **minimal base** (slim/distroless over full OS); **order instructions** so expensive, rarely-changing layers sit low and cache well; **combine** related `RUN` steps so you don't leave intermediate cruft in separate layers; use **multi-stage builds** so the heavy build toolchain never reaches the final image; and add a **`.dockerignore`** so junk (`.git`, `node_modules`, build outputs) never enters the build context in the first place. A note that surprises people: deleting a file in a *later* layer doesn't shrink the image, because the file still exists in the earlier layer underneath the whiteout — you have to avoid adding it, not delete it after.

---

# Part 4 — File storage: volumes, bind mounts, tmpfs

## 4.1 Why you need mounts at all

Part 3 established the writable layer's three problems: it's **ephemeral** (gone on `rm`), **private** (unshareable between containers), and **slow** for heavy writes (CoW). Persistence, sharing, and performance all require stepping *outside* the layer stack, and that's what mounts do — they attach storage from outside the union filesystem at a path inside the container. Three kinds:

## 4.2 Volumes — Docker-managed persistent storage

A **volume** is storage Docker manages, living outside the container's layers (on Linux, under `/var/lib/docker/volumes/`). It survives `docker rm`, can be shared between containers, backed up, and moved. It's the right default for **data you want to keep**: a database's data directory, uploaded files. Named volumes (`-v pgdata:/var/lib/postgresql/data`) are portable and referenced by name; anonymous volumes get a random id and are easy to lose track of.

## 4.3 Bind mounts — a host path into the container

A **bind mount** maps a specific host directory straight into the container (`-v /home/jamie/practiq-api/src:/app/src`). The container sees the live host files; edits on either side are immediately visible. This is the **local-development** tool: bind-mount your source so a change on the host triggers a reload inside the container without rebuilding the image. The trade-off is coupling to the host's exact filesystem layout (and, on macOS, the slower cross-VM I/O from 1.3), so it's a dev convenience, not a production pattern.

## 4.4 tmpfs — in-memory scratch

A **tmpfs mount** lives in the host's RAM, never on disk, and vanishes when the container stops. Use it for sensitive scratch data (a secret you don't want written to disk) or fast ephemeral temp space.

## 4.5 Which one — and where Practiq lands

| | **Volume** | **Bind mount** | **tmpfs** |
|---|---|---|---|
| Managed by | Docker | you (host path) | kernel (RAM) |
| Survives `rm`? | yes | yes (it's on the host) | no |
| Shareable between containers | yes | yes | no |
| Best for | persistent app data | dev source, live reload | secrets, fast scratch |
| Couples you to host layout | no | yes | no |

> **The tell — storage:** keep it? **volume**. Editing it live from the host during dev? **bind mount**. Must never hit disk? **tmpfs**. If none apply, the writable layer is fine — it's scratch that's meant to die with the container. In Practiq terms: **Testcontainers** Postgres needs *no* mount (it's meant to be created and destroyed per test run — the ephemeral writable layer is exactly right); a **local dev** Postgres wants a **named volume** so your data survives `docker compose down`; and hot-reloading `practiq-api` or `practiq-frontend` in dev wants a **bind mount** of the source.

---

# Part 5 — Why we use it, honestly

The value, stated plainly and then costed:

- **Parity / reproducibility** — the same image in dev, CI, and prod; the end of environment drift (1.1). For Practiq this is *why Testcontainers works*: the test Postgres is the production Postgres, so a query that passes tests behaves the same live — the dialect-drift fix from the data-access primer, made concrete.
- **Dependency isolation** — each service carries its own runtime and libraries; `practiq-api` (Java 21) and `practiq-extractor` (Python + pdfplumber) don't have to agree on anything on the host.
- **Immutable, versioned artifacts** — you build once, tag by digest, and deploy that exact artifact; rollback is "run the previous digest."
- **Speed and density** — process-fast startup and hundreds-per-host density (1.3), which is what makes CI ephemeral environments and autoscaling cheap.
- **Portability** — the same image runs on your laptop, a GitHub Actions runner, and AWS Fargate. Build in CI, push to **ECR**, run on **ECS** — no per-environment reinstall.
- **An ecosystem and an on-ramp** — registries (Docker Hub, ECR), Compose for local multi-service wiring, and a straight path to orchestration (k8s/ECS) when you outgrow a single host. Practiq's five-service split is, operationally, five images this ecosystem already knows how to build, store, and run.

And the honest costs, because "intentional" cuts both ways: **added complexity** (another layer of tooling, networking, and mental model); **stateful data is genuinely awkward** (everything about Parts 3–4 is the tax you pay for making persistence a deliberate act); a **larger security surface** (the daemon, image provenance, shared-kernel isolation); **macOS/Windows overhead** (the hidden VM, slow bind-mount I/O — 1.3); and a **real learning curve** whose whole point this document is to shorten. Docker earns its place for a multi-service, reproducibility-sensitive app like Practiq — but it is a tool with costs, not a free good.

---

# Part 6 — The alternatives, and when each is suitable

"Alternative to Docker" splits into several different questions. The landscape below is current (2026); the theme is that **OCI standards make most of these interchangeable** — images built one way run another.

## 6.1 Other engines / runtimes (run containers a different way)

- **Podman** — the main rival. **Daemonless** (each command forks the runtime directly, no `dockerd`) and **rootless by default** (containers run as your user via user namespaces), which is a real security win and sidesteps Docker Desktop's licensing. CLI is ~95% Docker-compatible (`alias docker=podman` gets you far); it defaults to the lighter `crun` runtime. *Suitable when:* security-conscious CI, Linux servers, systemd-managed workloads, or avoiding Desktop licensing. Rougher edges around anything that assumes the Docker socket, and Compose parity.
- **containerd + nerdctl** — the runtime *underneath* Docker, usable directly; `nerdctl` is a Docker-compatible CLI for it. *Suitable when:* you want the production runtime (k8s uses containerd) locally, minus Docker's extras.
- **CRI-O** — a minimal Kubernetes-only runtime (the OpenShift default). *Suitable when:* pure k8s, nothing else.
- **LXC/LXD** — *system* containers (a full init + userland, more VM-like) rather than single-app containers. *Suitable when:* you want a persistent "lightweight machine," not an ephemeral app.
- **gVisor / Kata Containers** — stronger isolation: gVisor interposes a user-space kernel; Kata runs each container in a tiny VM. *Suitable when:* running untrusted code and shared-kernel isolation isn't enough.

## 6.2 Desktop tools (the local GUI/VM around the engine)

Docker Desktop is one option; **Podman Desktop**, **Rancher Desktop**, **OrbStack** (fast, macOS-only), **Colima** (CLI, Mac/Linux), and **Finch** (AWS-backed) are alternatives — several driven by Docker Desktop's paid licensing for larger companies (250+ employees). *Suitable when:* you want a free/faster local setup; for a solo project like yours, Desktop's free tier or OrbStack/Colima are all fine.

## 6.3 Build tools (make images a different way)

- **BuildKit / buildx** — Docker's modern builder, now the default: parallel stages, better caching, build secrets, multi-arch. You already use it whenever you `docker build` (7.2).
- **Buildah** — builds images (often from Dockerfiles) with no daemon; pairs with Podman.
- **Kaniko** — builds images **inside a container/cluster with no Docker daemon**, ideal for building *in* Kubernetes CI where running a daemon is awkward.
- **Jib** — **Java-specific**, and directly relevant to you: it builds an optimised OCI image straight from **Gradle/Maven** with **no Dockerfile and no Docker daemon**, splitting your app into dependency/resource/class layers so only changed classes re-layer (excellent caching). *Suitable when:* a JVM service and you'd rather not maintain a Dockerfile — a genuine option for `practiq-api`. Trade-off: less control and Java-only. Worth a spike to compare against your multi-stage Dockerfile (7.4a).
- **Cloud Native Buildpacks** (Paketo) — build an image from source with *no* Dockerfile at all, by auto-detecting the language. *Suitable when:* you want convention over configuration across many services.

## 6.4 Not-containers-at-all

- **VMs** — when you need strong isolation or a different kernel (1.3).
- **Serverless** (AWS Lambda) — no container to manage at all; the platform runs your function. *Suitable when:* event-driven, bursty, stateless work. (Practiq's anonymous-attempt cleanup is a plausible Lambda candidate later.)
- **PaaS** (Render, Fly.io, Heroku) — you push code, they build and run it (often *using* containers under the hood). *Suitable when:* you want to skip the ops.
- **Nix** — reproducible *environments* without containers, via exact dependency pinning. *Suitable when:* the goal is reproducibility, not isolation/deployment.

## 6.5 Orchestration (where you go *next*, not an alternative)

Docker runs containers on *one* host. Running many across many hosts — scheduling, restart, scaling, service discovery — is **orchestration**: **Kubernetes**, AWS **ECS/Fargate**, Docker **Swarm**, **Nomad**. Given your AWS/Terraform trajectory, **ECS/Fargate** is the most likely Practiq target — you build images in GitHub Actions, push to ECR, and Fargate runs them without you managing servers. Not something you need now; the thing your images are *ready for* when you do.

---

# Part 7 — A practical Dockerfile guide

## 7.1 Anatomy of a Dockerfile

A Dockerfile is a recipe; each instruction that changes the filesystem adds a **layer** (3.2). The core instructions:

- **`FROM image:tag`** — the base layers to build on. `FROM scratch` is the empty base (for static binaries).
- **`WORKDIR /app`** — sets (and creates) the working directory for what follows.
- **`COPY src dest`** — copies from the build context into the image. Prefer it. **`ADD`** also fetches URLs and auto-extracts tarballs — use only when you need those, otherwise `COPY`.
- **`RUN cmd`** — executes a command at **build** time, in a new layer (e.g. installing packages, compiling). *Shell form* (`RUN apt-get update`) runs via `/bin/sh -c`; *exec form* (`RUN ["executable","arg"]`) doesn't invoke a shell.
- **`ENV KEY=val`** — environment variables baked into the image and present at runtime.
- **`ARG KEY`** — a **build-time** variable (passed with `--build-arg`); *not* present at runtime unless copied into an `ENV`. Do **not** pass secrets this way — they linger in build history (7.6).
- **`EXPOSE 8080`** — *documents* a port. It does **not** publish it; `docker run -p 8080:8080` does the actual host→container mapping. (Confusing these is the "port works inside but not from the host" symptom.)
- **`USER appuser`** — the user for subsequent instructions and at runtime. Running as **non-root** is a baseline security practice (7.6).
- **`ENTRYPOINT` vs `CMD`** — the pair people muddle. **`ENTRYPOINT`** is the executable that always runs; **`CMD`** provides default arguments (or, with no ENTRYPOINT, the default command). At `docker run img extra-args`, the args *replace* `CMD` and *append to* `ENTRYPOINT`. Use *exec form* (JSON array) so signals (SIGTERM on stop) reach your process. Typical: `ENTRYPOINT ["java","-jar","app.jar"]`.
- **`HEALTHCHECK`** — a command Docker runs to mark the container healthy/unhealthy (an endpoint that ECS/Compose can act on).
- **`LABEL`** — metadata (source repo, version).

> **The tell — ENTRYPOINT vs CMD:** if the container is one fixed program, put it in `ENTRYPOINT` and its default flags in `CMD`. If you want `docker run img somecommand` to be able to run *anything*, use `CMD` alone. "Container exits immediately" almost always means the main process isn't running in the foreground — a background/daemonised process lets PID 1 exit and the container stops with it.

## 7.2 Build cache and instruction order

Each instruction is a cache entry keyed on the instruction *plus its inputs* (for `COPY`, the file contents). On a build, Docker reuses cached layers until the first change; that change **invalidates every layer after it**. The rule that follows: **order from least- to most-frequently-changing.** Copy dependency manifests and install dependencies *before* copying source, so the expensive dependency layer stays cached across the source edits you make all day:

```dockerfile
COPY build.gradle settings.gradle ./     # changes rarely
RUN gradle dependencies                    # expensive — cached across source edits
COPY src ./src                             # changes constantly
RUN gradle build                           # re-runs, but deps are already cached
```

Get this backwards (copy `src` first) and every one-character source change re-downloads every dependency. BuildKit adds **cache mounts** (`RUN --mount=type=cache,target=/root/.gradle ...`) that persist a package-manager cache across builds *without* baking it into the image, and **build secrets** (`--mount=type=secret`) that are available during a `RUN` but never land in a layer. A `.dockerignore` keeps the context small so unrelated file changes don't bust the cache.

## 7.3 Multi-stage builds

The single most important image-size technique for compiled languages. Use a fat build image to compile, then copy *only the artifact* into a slim runtime image; the toolchain never ships:

```dockerfile
# ---- build stage: has the full JDK + Gradle ----
FROM gradle:8.7-jdk21 AS build
WORKDIR /app
COPY build.gradle settings.gradle ./
RUN gradle dependencies --no-daemon
COPY src ./src
RUN gradle build --no-daemon

# ---- runtime stage: just a JRE + the jar ----
FROM eclipse-temurin:21-jre
WORKDIR /app
COPY --from=build /app/build/libs/practiq-api-*.jar app.jar
EXPOSE 8080
USER 1000
ENTRYPOINT ["java","-jar","app.jar"]
```

The final image contains a JRE and one jar — not Gradle, not the JDK, not your source. Smaller to push/pull, faster to start, far less attack surface.

## 7.4 A gallery of common use cases

**(a) Java / Micronaut backend — `practiq-api`.** The multi-stage build above. Two upgrades worth knowing: Micronaut supports a *layered* jar so dependencies and application classes become separate image layers (better caching — app-only changes don't re-layer dependencies); and **Jib** (6.3) can produce this image from Gradle with no Dockerfile at all. Start with the Dockerfile above; spike Jib to compare.

**(b) Python service — `practiq-extractor` (FastAPI + pdfplumber).**

```dockerfile
FROM python:3.12-slim
WORKDIR /app
COPY requirements.txt .
RUN --mount=type=cache,target=/root/.cache/pip \
    pip install --no-cache-dir -r requirements.txt
COPY . .
RUN useradd -m appuser
USER appuser
EXPOSE 8000
CMD ["uvicorn","main:app","--host","0.0.0.0","--port","8000"]
```

Note the same ordering discipline (requirements before source), a cache mount for pip, `slim` not Alpine (7.6), and a non-root user. `--host 0.0.0.0` is essential — binding to `localhost` inside the container means the host can't reach it.

**(c) React frontend — `practiq-frontend` (build, then serve static).** Multi-stage: build with Node, serve the static output with a tiny nginx.

```dockerfile
FROM node:20-alpine AS build
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

FROM nginx:alpine
COPY --from=build /app/dist /usr/share/nginx/html
EXPOSE 80
```

The final image is nginx plus static files — no Node, no `node_modules`. (Alpine is fine here; the JVM caveat in 7.6 doesn't apply to nginx/static.)

**(d) Local-dev Postgres.** You rarely write this — you use the official image via Compose (7.5) with a named volume for persistence and an init script mounted at `/docker-entrypoint-initdb.d/`.

**(e) Dev container with hot reload.** Same as (b)/(a) but you `-v ./src:/app/src` (bind mount, 4.3) and run the dev server in reload mode, so host edits reload the app with no rebuild.

## 7.5 Docker Compose — wiring services for local dev

Compose describes a multi-service local stack in one file. A cut-down Practiq dev stack — API, Postgres, extractor:

```yaml
services:
  postgres:
    image: postgres:16
    environment:
      POSTGRES_DB: practiq
      POSTGRES_PASSWORD: dev
    volumes:
      - pgdata:/var/lib/postgresql/data      # named volume → survives `down`
    healthcheck:
      test: ["CMD-SHELL","pg_isready -U postgres"]
      interval: 5s

  api:
    build: ./practiq-api
    depends_on:
      postgres:
        condition: service_healthy
    environment:
      DATASOURCE_URL: jdbc:postgresql://postgres:5432/practiq   # service name, not localhost
    ports:
      - "8080:8080"

  extractor:
    build: ./practiq-extractor
    ports:
      - "8000:8000"

volumes:
  pgdata:
```

The one thing that catches everyone: services reach each other by **service name** (`postgres`, not `localhost`) — Compose gives them a shared network where names resolve to containers. `localhost` inside the `api` container is the `api` container itself, not Postgres.

## 7.6 Best practices, consolidated

- **Small base image.** *full* (e.g. `eclipse-temurin:21-jdk`) → most tools, biggest; *slim* (e.g. `python:3.12-slim`) → good default; *Alpine* → tiny **but** uses musl libc, which can subtly break glibc-expecting binaries — **be cautious with Alpine for the JVM and native code** (use `slim`/Temurin instead); *distroless* → smallest attack surface (no shell/package manager), but harder to debug. For `practiq-api`, a Temurin JRE or distroless-java runtime; Alpine is fine for nginx/static (7.4c).
- **Run as non-root** (`USER`) — limits blast radius if the container is compromised.
- **Pin versions** (and ideally digests) for reproducibility — `latest` drifts.
- **`.dockerignore`** — exclude `.git`, `node_modules`, build outputs, secrets; smaller context, faster builds, no accidental cache-busting.
- **One concern per container** — the API, the DB, the extractor are separate images, not one image running several processes.
- **Order layers** least- to most-volatile (7.2); **multi-stage** to keep toolchains out of the runtime (7.3).
- **`HEALTHCHECK`** so the orchestrator knows when the container is actually ready.
- **Never bake secrets** into `ENV`/`ARG`/layers — use build secrets (7.2) at build time and injected env/secret managers at run time.

---

# Part 8 — A worked trace: build, then run

Tying Parts 2–3 together in one narrative, using `practiq-api` and the multi-stage Dockerfile from 7.3.

**Build (`docker build -t practiq-api .`).** BuildKit parses the Dockerfile into a graph and executes it. The build stage pulls `gradle:8.7-jdk21` (its layers, cached if already present), then for each filesystem-changing instruction produces a **read-only layer**: the `COPY` of the build files, the `gradle dependencies` layer (this is the one you protected by ordering — 7.2), the `COPY src`, the `gradle build`. Then the runtime stage starts fresh from `eclipse-temurin:21-jre` and `COPY --from=build` pulls *just the jar* across — the JDK/Gradle layers are discarded, never reaching the final image. The result is an **image**: an ordered stack of read-only layers (3.1) plus a config recording `ENTRYPOINT ["java","-jar","app.jar"]`, `EXPOSE 8080`, `USER 1000`. Nothing is running; the image is inert (1.4). Identical layers are stored once and shared (3.2).

**Run (`docker run -p 8080:8080 practiq-api`).** `containerd` takes the image's read-only layers as **lowerdirs** and adds a fresh, empty **writable upperdir** for this container (2.4, 3.3). `runc` creates the namespaces (its own PID space → the JVM is PID 1 inside; its own net namespace → it binds 8080 privately, and `-p` bridges the host's 8080 into it — 2.2) and applies any cgroup limits (2.3). It pivots into the merged filesystem as `/` and `exec`s `java -jar app.jar` as the entrypoint. The JVM now runs as an ordinary host process under those three constraints. Anything it writes — logs, temp files — lands in **its** writable layer and nowhere else (3.3). Run it a second time and you get a second container with a second, independent writable layer over the same shared read-only base (3.4).

**Stop and remove (`docker stop` / `docker rm`).** Stop sends SIGTERM to PID 1 (which the JVM receives *because* you used exec-form ENTRYPOINT — 7.1); the process exits, the container stops. `rm` deletes the **writable layer** — every runtime write is gone (3.3). The image is untouched and ready to spawn a clean container again. Which is the whole reason persistent data belongs on a **volume** (4), outside this cycle.

That single trace is the document in miniature: build makes read-only layers; run adds one writable layer plus namespaces and cgroups; remove throws the writable layer away. If you can narrate those three steps, you understand Docker's core.

---

# Part 9 — When to use what: consolidated decision frameworks

Each axis is a decision you can now make deliberately. Format: *options → the tell → default.*

**A. Container vs VM vs serverless.** Tell → container: reproducible app deployment, density, speed (the Practiq default). Tell → VM: strong isolation or a different kernel needed. Tell → serverless: event-driven, bursty, stateless. Default: **container for the services; consider Lambda for isolated event jobs later.**

**B. Base image.** Tell → slim: sensible default. Tell → distroless: minimal attack surface in prod, you can debug elsewhere. Tell → full: you need build tools in the running image (rare). Tell → *avoid* Alpine for JVM/native. Default: **slim/Temurin (JVM) or distroless (prod); Alpine only for static/nginx.**

**C. Storage: volume vs bind vs tmpfs vs writable layer.** Tell → volume: keep it. Tell → bind: edit live from host in dev. Tell → tmpfs: must never hit disk. Tell → writable layer: scratch meant to die with the container. Default: **volume for data, bind for dev source, writable layer for the ephemeral rest.**

**D. Multi-stage build?** Tell → yes: any compiled/bundled app (Java, Node build, Go). Tell → no: an interpreted app with no build step and a slim base already. Default: **multi-stage for `practiq-api` and `practiq-frontend`; single-stage slim for `practiq-extractor`.**

**E. ENTRYPOINT vs CMD.** Tell → ENTRYPOINT (+CMD for default flags): the container is one fixed program. Tell → CMD alone: you want `docker run img <anything>` to work. Default: **ENTRYPOINT exec-form for services.**

**F. How to build the image.** Tell → Dockerfile: full control, any language, the portable default. Tell → Jib: a JVM service and you'd rather not maintain a Dockerfile (spike it for `practiq-api`). Tell → Buildpacks: convention-over-config across many services. Default: **Dockerfile; evaluate Jib for the Java service.**

**G. Docker vs an alternative engine.** Tell → Docker: solo dev, best ecosystem, free tier fine. Tell → Podman: rootless/daemonless security, licensing-sensitive orgs, Linux servers. Tell → containerd/CRI-O: you're in Kubernetes. Default (for you): **Docker locally; the images are OCI-standard so you're not locked in.**

**H. Where the app runs at scale.** Tell → single host + Compose: dev and tiny prod. Tell → ECS/Fargate: AWS, no cluster to manage (your likely path). Tell → Kubernetes: multi-cloud/complex orchestration needs. Default: **Compose now; ECS/Fargate when you deploy.**

---

# How to use and expand this document

- Suggested home: `docs/` in `practiq-infrastructure` (or alongside `PRACTIQ_MASTER.md`). It's Practiq-flavoured so it doubles as "why the containers are shaped like this" onboarding.
- Good next deep-dives, any of which I can expand into a standalone worked section: **container networking** end to end (bridge/host/none, port publishing, the Compose network, DNS by service name); **multi-architecture images** (building linux/amd64 on Apple Silicon so it runs on your amd64 servers — the classic "built on my Mac, won't run in prod" incident); **a Jib vs Dockerfile spike** for `practiq-api` with real image-size and build-time numbers; **the full GitHub Actions → ECR → ECS/Fargate pipeline** with the actual workflow YAML; **image security** (scanning, provenance, rootless, distroless, non-root) as a standalone checklist; **Docker for Testcontainers** — how it wires up, why CI runners need a daemon, and the reuse/ryuk mechanics.
- Ask for any of the above and I'll write it against your actual services.
