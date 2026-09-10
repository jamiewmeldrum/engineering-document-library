# Application Security — A Primer №60

*The security an application developer owns. Where №51 (Parts 6–9) covers the wire — crypto primitives, TLS, certificates, cookies and the browser model — this covers **the code**: the vulnerability classes, authentication and authorisation, secrets, dependencies, and the habits that close the doors attackers actually use.*

The framing that makes security tractable rather than paralysing: **you are not defending against a genius adversary inventing novel attacks. You are defending against automated scanners and opportunists exploiting well-known, well-documented mistakes.** The overwhelming majority of real breaches are an unparameterised query, an unpatched dependency, a leaked credential, or a missing authorisation check. Get the baseline right and you've eliminated the attacks that actually happen.

The second idea, and the one that changes how you write code: **every input is hostile until proven otherwise.** Not just form fields — headers, cookies, URL parameters, file uploads, API responses from services you trust, message payloads, filenames, database contents that someone else wrote. The mental habit is to ask, at every boundary, *"what if this is not what I expect?"*

Contents:

- **Part 1** — the mindset and the principles
- **Part 2** — injection
- **Part 3** — authentication
- **Part 4** — authorisation
- **Part 5** — OAuth2, OIDC and JWTs
- **Part 6** — input validation and output encoding
- **Part 7** — secrets
- **Part 8** — dependencies and the supply chain
- **Part 9** — data protection
- **Part 10** — the rest of the OWASP Top 10
- **Part 11** — threat modelling and secure development
- **Part 12** — when to use what

## Vulnerability index

| The mistake | The class | Fix | §|
|---|---|---|---|
| String-concatenated SQL | SQL injection | parameterised queries | §2.1 |
| User input in a shell command | command injection | avoid shells; use APIs | §2.3 |
| Unescaped user content in HTML | XSS | contextual output encoding, CSP | §6.3 |
| Anyone can fetch `/orders/123` | broken access control (IDOR) | **check ownership per object** | §4.2 |
| Passwords hashed with SHA-256 | weak credential storage | bcrypt/argon2 | §3.2 |
| Session ID unchanged after login | session fixation | regenerate on privilege change | §3.4 |
| JWT accepted without verifying signature | broken auth | verify signature *and* claims | §5.4 |
| Secret in the repo or Dockerfile | secrets exposure | secret manager, rotate | §7 |
| Dependency with a known CVE | vulnerable components | scan and patch | §8 |
| Stack trace returned to the user | information disclosure | generic errors, log detail | §10.2 |
| File upload written to a user-supplied path | path traversal | normalise and validate | §6.4 |
| No rate limit on login | credential stuffing | throttle, lock out, MFA | §3.5 |

---

# Part 1 — The mindset and the principles

## 1.1 The principles

**Never trust input.** Validate at every boundary, on the server. Client-side validation is a UX feature, not a control — anyone can bypass it with `curl`.

**Least privilege.** Every component gets the minimum access it needs, and no more. The database user your application connects as shouldn't be able to `DROP TABLE`. The IAM role for a service that reads one S3 prefix shouldn't have `s3:*`. The container shouldn't run as root (№50 §7.6). This is the principle that limits *blast radius* when something else fails — and something else will fail.

**Defence in depth.** Assume any single control will be bypassed. A WAF *and* parameterised queries *and* a restricted database user *and* encrypted columns. No single failure should be catastrophic.

**Fail secure.** When something goes wrong — an exception in the auth check, a timeout on a permission lookup — the result must be **deny**. Code that grants access on error is a backdoor.

**Secure by default.** The safe option should require no configuration; the dangerous one should require deliberate effort. If a developer must remember to add a check, the check will eventually be forgotten.

**Don't roll your own crypto or auth.** Use vetted libraries. The failure modes are subtle, invisible in testing, and catastrophic (№51 §6).

## 1.2 The economics

Security is a **trade-off against convenience, latency and development time**, and the right amount is proportional to what you're protecting. Practiq's exam questions are not credit cards, and treating them like state secrets would waste effort you should spend elsewhere. But the **baseline** — parameterised queries, patched dependencies, no secrets in git, real authorisation checks, TLS everywhere — costs almost nothing and closes almost everything. That baseline is non-negotiable regardless of what you're building.

---

# Part 2 — Injection

The oldest and still one of the most damaging classes: **untrusted data is interpreted as code or commands.**

## 2.1 SQL injection

```java
// CATASTROPHIC — the input becomes part of the query
String sql = "SELECT * FROM question WHERE concept_id = " + conceptId;
// input: 1 OR 1=1                    → returns everything
// input: 1; DROP TABLE question --   → exactly what it looks like
```

The fix is **parameterised queries**, and the reason they work is structural, not filtering: the SQL is compiled with placeholders *first*, then values are bound as data. A value can never become syntax, no matter what it contains.

```java
// Safe — JDBC (№20 §1.3)
ps = conn.prepareStatement("SELECT * FROM question WHERE concept_id = ?");
ps.setLong(1, conceptId);

// Safe — JPQL with named parameters
@Query("SELECT q FROM Question q WHERE q.conceptId = :cid")

// Safe — Criteria / specifications: values are always bound (№20 §3.5)
cb.equal(root.get(Question_.conceptId), conceptId)

// STILL VULNERABLE — native query with concatenation
@Query(value = "SELECT * FROM question WHERE status = '" + status + "'", nativeQuery = true)
```

Using an ORM does **not** automatically protect you — it protects you because it parameterises. Concatenate into a native query and you're back where you started.

**What can't be parameterised:** table names, column names, `ORDER BY` fields, and `ASC`/`DESC`. If those must be dynamic, **validate against an allow-list**, never pass the raw value through.

```java
private static final Set<String> SORTABLE = Set.of("id", "difficulty", "created_at");
if (!SORTABLE.contains(sortField)) throw new IllegalArgumentException("invalid sort field");
```

## 2.2 The other injections

Same root cause, different interpreter: **command injection** (user input in a shell), **LDAP injection**, **XPath**, **NoSQL injection** (a query object built from untrusted JSON), **log injection** (newlines forged into log entries to fake records), **header injection** (CRLF into HTTP headers), and **template injection** (user input evaluated by a template engine — often full remote code execution).

## 2.3 Command injection specifically

```java
// DANGEROUS — a shell interprets the string
Runtime.getRuntime().exec("pdftotext " + userFilename);
// filename: "x.pdf; rm -rf /"

// Safer — no shell, arguments passed as a list
new ProcessBuilder("pdftotext", userFilename).start();
```

`ProcessBuilder` with separate arguments avoids shell interpretation entirely — the same structural fix as parameterised SQL. Better still, **use a library rather than shelling out**. Relevant to Practiq's extraction pipeline: any external tool invoked with a user-influenced filename needs this care, plus validation that the filename is what you expect.

---

# Part 3 — Authentication

**Authentication = who are you. Authorisation = what may you do.** Confusing them is the source of a great deal of broken security (and of 401-vs-403 confusion, №51 §5.2).

## 3.1 The mechanisms

| Mechanism | Suits |
|---|---|
| **Username + password + session cookie** | classic browser applications |
| **OAuth2 / OIDC** | delegated identity, SSO, third-party login |
| **API keys** | service-to-service, simple, low-security |
| **mTLS** | service-to-service, high assurance (№51 §8.6) |
| **MFA/TOTP/WebAuthn** | a second factor — the single biggest win against credential theft |

## 3.2 Passwords, if you must store them

**Never store passwords, and never hash them with a fast hash.** SHA-256 is designed to be fast, which is exactly wrong — a GPU computes billions per second. Use a **slow, salted, memory-hard** algorithm designed for the job:

**Argon2id** (current recommendation), **bcrypt** (widely available, still fine), **scrypt**, or **PBKDF2** (when a standard mandates it). Each generates a per-password **salt** automatically — so two users with the same password get different hashes, defeating rainbow tables — and has a tunable **work factor** you raise as hardware improves.

```java
// bcrypt — the salt and cost are embedded in the output string
String hash = BCrypt.hashpw(plaintext, BCrypt.gensalt(12));
boolean ok  = BCrypt.checkpw(candidate, hash);   // constant-time comparison
```

Other rules: enforce **length over complexity** (a long passphrase beats `P@ssw0rd!`), **check against known-breached password lists**, never impose a maximum length below ~64, never truncate, and never log or email a password.

**Best of all: don't handle passwords.** Delegating to an identity provider (Cognito, Auth0, an OIDC provider) removes an entire category of risk you'd otherwise own.

## 3.3 Sessions

For a browser application, a server-side session with an opaque ID in a cookie remains the safest default (№51 §9.3). The cookie must be:

```
Set-Cookie: session=<random>; HttpOnly; Secure; SameSite=Lax; Path=/
```

`HttpOnly` (JavaScript can't read it, so XSS can't steal it), `Secure` (HTTPS only), `SameSite=Lax` (CSRF defence). The session ID must be **cryptographically random** (`SecureRandom`, not `Random` — №51 §6.3) and long enough to be unguessable.

## 3.4 Session lifecycle mistakes

**Session fixation** — an attacker plants a known session ID (via a link or a subdomain), the victim logs in, and the attacker's session is now authenticated. **Fix: regenerate the session ID on login and on any privilege change.** This is a one-line fix that's frequently missed.

Also: implement **absolute and idle timeouts**; **invalidate server-side on logout** (clearing the cookie alone leaves a valid session); and invalidate all sessions on password change.

## 3.5 Protecting the login endpoint

**Rate limit it.** Credential stuffing — replaying username/password pairs from other breaches — is automated and constant. Throttle by IP and by account, apply progressive delays, and consider CAPTCHA after repeated failures.

**Return identical responses** for "no such user" and "wrong password", and take the same time for both — otherwise you've built a user-enumeration oracle. The same applies to password reset and registration.

**Offer MFA.** It defeats credential stuffing almost entirely, which is why it's the highest-value authentication feature you can add.

---

# Part 4 — Authorisation

Consistently the **most common serious vulnerability** in real applications, because it can't be solved by a library — it requires a decision at every access.

## 4.1 The models

**RBAC** (role-based) — permissions attach to roles, roles to users. Simple, coarse. **ABAC** (attribute-based) — decisions from attributes (owner, department, time, resource state). Flexible, complex. **Ownership-based** — the common case: you may act on the things you own.

## 4.2 Broken access control — the big one

```java
// BROKEN — authenticated, but no ownership check
@Get("/api/v1/attempts/{id}")
public Attempt get(Long id) {
    return attemptRepository.findById(id).orElseThrow();
}
```

The user is logged in, so authentication passed — but nothing verifies that *this* attempt belongs to *this* user. Change the ID in the URL and you read someone else's data. That's **IDOR** (insecure direct object reference), and it is everywhere.

```java
// FIXED — the query itself enforces ownership
@Get("/api/v1/attempts/{id}")
public Attempt get(Long id, Principal principal) {
    return attemptRepository.findByIdAndUserId(id, principal.getName())
            .orElseThrow(() -> new NotFoundException(id));
}
```

Two details in the fix worth copying: **push the check into the query** rather than fetching then comparing (harder to forget, and one round trip), and **return 404 rather than 403** for resources the user shouldn't know exist — a 403 confirms the object exists, which leaks information.

## 4.3 The rules

**Check on every request, server-side.** Hiding a button is not access control. **Deny by default** — the absence of a rule means no. **Check at the resource, not the route** — route-level rules miss per-object ownership. **Never trust a client-supplied identity** — take the user from the authenticated session or token, never from a request parameter (`?userId=` is an invitation). And guard **function-level access** too: an admin-only endpoint must check the role, not merely be absent from the UI.

The structural defence for Practiq: your public endpoint's signature hard-codes `APPROVED` with no status parameter, so requesting unapproved content is **unexpressible** rather than merely forbidden. That's the "secure by default" principle (§1.1) implemented in a type signature — the strongest form, because it can't be forgotten.

---

# Part 5 — OAuth2, OIDC and JWTs

## 5.1 What each is

- **OAuth2** — a framework for **delegated authorisation**: letting an application act on a resource owner's behalf without their password. It is *not* an authentication protocol.
- **OIDC** — a thin identity layer **on top of OAuth2** that adds authentication, standard claims, and the **ID token**. This is what you want for "log in with X."
- **JWT** — a *token format* (used by both, and independently). Not a protocol.

The common error is using raw OAuth2 access tokens as proof of identity — they authorise access to a resource, they don't authenticate a user. OIDC exists precisely to fix that.

## 5.2 The flow that matters

**Authorisation Code flow with PKCE** is the current recommendation for essentially every client type — web apps, SPAs and mobile:

```
1. App redirects the user to the identity provider (with a PKCE code_challenge)
2. User authenticates there (the app never sees the password)
3. IdP redirects back with a short-lived authorisation CODE
4. App exchanges code + code_verifier for tokens (back-channel, server-to-server)
5. App receives: ID token (who), access token (what), refresh token (renewal)
```

**PKCE** binds the code to the client that requested it, so an intercepted code is useless — it's why the older Implicit flow (tokens in the URL fragment) is deprecated. Also deprecated: the Resource Owner Password Credentials flow, which hands your password to the application and defeats the point.

## 5.3 The tokens

| Token | Purpose | Lifetime |
|---|---|---|
| **ID token** (JWT) | proves *who the user is* — for your app to consume | short |
| **Access token** | authorises API calls — for the *API* to consume | minutes |
| **Refresh token** | obtains new access tokens | long, **revocable** |

The pattern that follows: **short-lived access tokens plus a revocable refresh token.** Because a JWT can't be revoked (§5.5), you keep its life short and hold the revocation power in the refresh token, which the server can invalidate.

## 5.4 Validating a JWT properly

A JWT is `header.payload.signature`, base64url-encoded. **It is signed, not encrypted** — anyone can read the payload. Validation must check *all* of:

- **The signature**, using the issuer's public key (fetched from its JWKS endpoint and cached).
- **`alg`** — against your expected algorithm. Never trust the token's own header. The classic attacks are `alg: none` (accepting an unsigned token) and algorithm confusion (an RSA public key used as an HMAC secret). A good library with a pinned expected algorithm prevents both.
- **`exp`** (not expired), **`iss`** (the issuer you expect), **`aud`** (intended for you), and `nbf`.

Then, and only then, trust the claims. Use a maintained library — this is not a place to write your own parser.

## 5.5 The JWT trade-off

**You cannot revoke a JWT.** A stateless token is valid until it expires, so a stolen one works until then, and "log out everywhere" doesn't. Mitigations are all partial: short expiry, a denylist of revoked token IDs (which reintroduces the state you were avoiding), or refresh-token rotation.

**Never put secrets in a JWT** (it's readable), and don't let it grow — it's sent on every request.

> **The tell — sessions or tokens:** **server-side sessions for browser applications** (instant logout matters, and an `HttpOnly` cookie protects against XSS theft); **short-lived JWTs for APIs and service-to-service**. Storing a JWT in `localStorage` for a web app trades away the XSS protection the cookie gave you — a common and rarely-justified choice.

---

# Part 6 — Input validation and output encoding

## 6.1 Validate on input, encode on output

Two distinct defences, both needed. **Validation** rejects data that isn't what you expect, at the boundary. **Encoding** makes data safe for the context it's about to enter. Validation alone can't prevent XSS (a legitimate name may contain `<`); encoding alone doesn't stop bad data entering your system.

## 6.2 Validation

**Allow-list, don't deny-list.** Specify what's permitted, not what's forbidden — you will never enumerate all the bad inputs, and attackers only need the one you missed.

Validate **type, length, range, format**, and business rules. Do it **server-side**, at the boundary, and let the type system carry the guarantee afterwards — a validated value object (№41 §8.3) means invalid state can't exist deeper in the system.

```java
public record Difficulty(int value) {
    public Difficulty {
        if (value < 1 || value > 5) throw new IllegalArgumentException("difficulty 1-5");
    }
}
```

Jakarta Bean Validation (`@NotNull`, `@Size`, `@Pattern`, `@Email`) handles the mechanical cases declaratively; note Practiq's decision to validate at the **service boundary** rather than the controller, which means the guarantee holds regardless of how the service is called.

## 6.3 Output encoding and XSS

**XSS is untrusted data interpreted as markup or script by a browser.** The defence is **contextual encoding** — the correct escaping depends on where the data lands:

| Context | Danger | Encode as |
|---|---|---|
| HTML body | `<script>` | HTML entities |
| HTML attribute | breaking out of quotes | attribute encoding |
| JavaScript | injecting code | **avoid entirely**; JSON-encode |
| URL parameter | traversal, redirect | URL encoding |
| CSS | expression injection | CSS encoding |

Modern frameworks help enormously — **React escapes interpolated values by default**, which is why the dangerous escape hatch is named `dangerouslySetInnerHTML`. If you need to accept rich text, **sanitise with a maintained allow-list library** (DOMPurify, OWASP Java HTML Sanitizer); never write your own tag filter.

**CSP** (Content-Security-Policy) is the strongest defensive layer: it tells the browser which sources may execute, so injected inline script is refused even if your escaping fails (№51 §9.6).

## 6.4 File uploads and path traversal

Untrusted filenames are a direct route to reading or writing arbitrary files:

```java
// DANGEROUS — "../../etc/passwd" escapes the directory
Path target = Path.of("/uploads", userFilename);

// Safer — generate your own name, and verify containment
Path base   = Path.of("/uploads").toAbsolutePath().normalize();
Path target = base.resolve(UUID.randomUUID() + extension).normalize();
if (!target.startsWith(base)) throw new SecurityException("path traversal");
```

Also: validate the **content type by inspecting the file**, not by trusting the header or extension; cap the size; store uploads outside the web root (ideally in S3, not the filesystem); never execute or include an uploaded file; and scan for malware if the file is shared with others. Relevant to Practiq's PDF ingestion — a malicious PDF is a real attack vector against parsing libraries, which is a good reason for extraction to run in an isolated container (№50).

---

# Part 7 — Secrets

**A secret in source control is compromised**, permanently, even if the next commit removes it — history is public within your organisation and usually pushed (№53 §9.6). The response is to **rotate**, not merely delete.

The rules: never in code, never in a Docker image layer (`ARG`/`ENV` persist in history — №50 §7.6), never in logs, never in error messages, never in a URL (they land in access logs and browser history — №51 §9).

Where they belong: **AWS Secrets Manager** (rotation, integrations) or **SSM Parameter Store** (free, simple), injected at runtime as environment variables or fetched by the application (№54 §11). CI authenticates via **OIDC** so it holds no long-lived credentials at all (№56 §5.3).

Practical hygiene: **secret scanning** in CI and as a pre-commit hook (gitleaks, GitHub secret scanning) to catch them before they land; **rotate regularly** and immediately on any suspicion; and give each environment and service **its own** credentials, so a dev leak doesn't touch production.

---

# Part 8 — Dependencies and the supply chain

**Your dependencies are your largest attack surface.** A typical Java service has hundreds of transitive dependencies, and you've read the code of approximately none of them. Log4Shell was a logging library; the incident that takes you down will be something equally unremarkable.

The practices, all cheap:

- **Scan continuously** — Dependabot, Snyk, OWASP Dependency-Check, Trivy for images. Wire it into CI (№56 §4.4) so a new CVE opens a PR rather than waiting to be noticed.
- **Patch promptly.** The window between disclosure and mass exploitation is now days.
- **Pin versions** and commit lockfiles, so builds are reproducible and a compromised upstream release doesn't silently enter.
- **Minimise** — every dependency is attack surface plus maintenance. A left-pad-sized library isn't worth a supply-chain risk.
- **Prefer maintained, popular libraries** — an abandoned package will not get a security fix.
- **Beware typosquatting** — malicious packages named a character away from a popular one.
- **Generate an SBOM** so you can answer "are we affected?" in minutes rather than days when the next Log4Shell lands. That question is the one that matters during an incident.

---

# Part 9 — Data protection

**Encryption in transit** — TLS everywhere, including internal traffic where practical; HSTS to prevent downgrade (№51 §7, §9.6).

**Encryption at rest** — enable it; on AWS it's a checkbox backed by KMS (№91 §7). Understand it protects against stolen disks and snapshots, not against an attacker with valid application credentials.

**Minimise what you hold.** The safest data is data you never collected. Shorter retention, fewer fields, aggressive deletion of what you don't need. This reduces both breach impact and compliance burden.

**Classify.** Know what's PII, what's sensitive, what's public — you can't protect uniformly what you haven't distinguished.

**GDPR/UK GDPR realities** for anything with users: a lawful basis for processing, data-subject rights (access, deletion, portability), breach notification within 72 hours, privacy by design, and care with international transfers. Practiq holds relatively little — anonymous attempts and reviewer accounts — which is a good reason to keep it that way and to have the anonymous-attempt cleanup job actually delete.

**Mask in logs and non-production.** Production data in a development database is a breach waiting to happen; anonymise or synthesise it.

---

# Part 10 — The rest of the Top 10

Covering the OWASP categories not already treated:

## 10.1 Security misconfiguration

Default credentials left in place; unnecessary services and ports exposed; debug endpoints reachable in production; directory listing enabled; overly permissive CORS (`Access-Control-Allow-Origin: *` with credentials — №51 §9.5); missing security headers; verbose errors (§10.2); cloud storage left public (the single most common cloud breach). **Harden deliberately and verify — don't assume defaults are safe.**

## 10.2 Information disclosure

A stack trace returned to a user tells an attacker your framework, versions, file paths and sometimes SQL. **Return a generic message and a correlation ID; log the detail** (№57 §2.2). Also remove version-revealing headers (`Server`, `X-Powered-By`), and don't leak existence through differing responses or timing (§3.5, §4.2).

## 10.3 Cryptographic failures

Using MD5 or SHA-1; homemade encryption; ECB mode; hardcoded or reused keys; missing TLS on internal hops; `Random` where `SecureRandom` is required. **Use a vetted library with modern defaults** (AES-GCM, TLS 1.3) and rotate keys (№51 §6).

## 10.4 SSRF

The application is tricked into making a request to an attacker-chosen URL — used to reach internal services or the **cloud metadata endpoint** (`169.254.169.254`) to steal instance credentials (№54 §4.1 — this is what IMDSv2 hardens against). Defences: allow-list destinations, block private and link-local IP ranges, resolve then validate the IP (not just the hostname), and disable redirect following.

## 10.5 Insecure deserialization

Java's native serialization can **execute code during deserialization** — the root of a large family of RCE exploits. **Never deserialize untrusted data with `ObjectInputStream`.** Use JSON with explicit types; if you must, apply an `ObjectInputFilter` allow-list. Note Jackson has had its own polymorphic-typing vulnerabilities — avoid enabling default typing.

## 10.6 Logging and monitoring failures

Not a vulnerability in itself, but it's why breaches go undetected for months. Log authentication events, authorisation failures, and administrative actions; alert on anomalies; and make sure logs are **tamper-resistant** and retained long enough to investigate (№57).

---

# Part 11 — Threat modelling and secure development

## 11.1 Threat modelling, lightly

Four questions, answerable in an hour for a feature:

1. **What are we building?** A diagram with trust boundaries.
2. **What can go wrong?** **STRIDE** as a prompt: **S**poofing, **T**ampering, **R**epudiation, **I**nformation disclosure, **D**enial of service, **E**levation of privilege.
3. **What are we doing about it?** Controls per threat.
4. **Did we do a good job?** Review.

Applied to Practiq's review workflow: could someone spoof a reviewer (authentication)? Tamper with a question after approval (audit trail, optimistic locking)? Deny having approved something (logging)? Read unapproved content (the structural `APPROVED` constraint, §4.3)? Elevate from reviewer to admin (authorisation checks)?

## 11.2 Security in the lifecycle

**Design** — threat model; choose secure defaults. **Code** — the practices above; security-aware review (does this endpoint check ownership? is this query parameterised? where does this input come from?). **Build** — dependency scanning, SAST, secret scanning, image scanning (№56 §4.4). **Deploy** — least-privilege IAM, network isolation, secrets from a manager. **Run** — patching, monitoring, alerting on auth anomalies, incident response (№57 §10).

**Shift left** is the summary: a vulnerability caught in code review costs minutes; the same one found in production costs an incident, a disclosure, and possibly a regulator.

---

# Part 12 — When to use what

**A. Sessions or JWTs?** Tell → session: browser app, instant logout matters. Tell → JWT: API or service-to-service, horizontal scale. Default: **session cookie (`HttpOnly; Secure; SameSite=Lax`) for the web app; short-lived JWTs for machine callers.**

**B. Build auth or delegate?** Tell → delegate (Cognito/Auth0/OIDC): almost always — you avoid owning password storage, reset flows, MFA and breach risk. Tell → build: unusual requirements and the expertise to match. Default: **delegate.**

**C. Validate where?** Tell → boundary: the moment untrusted data enters (controller/service edge). Tell → types: encode the guarantee in value objects so it holds thereafter. Default: **both — validate at the edge, then make invalid states unrepresentable.**

**D. Allow-list or deny-list?** Tell → allow-list: always, where the valid set is enumerable. Tell → deny-list: only where allow-listing is genuinely impossible. Default: **allow-list.**

**E. 403 or 404?** Tell → 404: the user shouldn't know the resource exists. Tell → 403: existence isn't sensitive and a clear error helps. Default: **404 for other people's objects.**

**F. Secrets store?** Tell → Secrets Manager: needs rotation or cross-account sharing. Tell → Parameter Store: simple config and secrets, free. Default: **Parameter Store unless rotation earns the cost** (№54 §11).

**G. How much security for this system?** Tell → more: money, PII, health, credentials, anything regulated. Tell → baseline: internal tools, low-sensitivity data. Default: **the non-negotiable baseline always — parameterised queries, patched dependencies, no secrets in git, real authorisation checks, TLS, least privilege — then scale up with the data's sensitivity.**

---

# How to expand this

- *Related:* №51 Parts 6–9 (crypto, TLS, certificates, cookies, CORS, the browser attack model), №54 §11 / №91 §7 (the AWS security services), №56 §4.4 (security in the pipeline), №50 §7.6 (container hardening), №20 §3 (parameterised queries in context), №57 (detecting attacks).
- *Candidates for deeper treatment:* **auth end to end for Practiq** — OIDC flow, session handling, per-object authorisation, drawn out; **a threat model for Practiq** worked properly with STRIDE; **secure code review checklist** for Java/Micronaut; **supply-chain security** (SBOMs, signed images, provenance, SLSA).

*Stable material, written from knowledge — the vulnerability classes and defensive principles don't drift. Specific recommendations do: password-hashing parameters, TLS cipher preferences and the current OWASP Top 10 ordering are all worth checking against current guidance before you rely on them.*
