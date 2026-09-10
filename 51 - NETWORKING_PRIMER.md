# Networking — A Primer

*A working engineer's mental model for how bytes get from one machine to another, and how they're kept private and authentic on the way. Practiq lens: a React frontend talking to a Micronaut API over HTTPS, an API talking to Postgres over TCP, all of it in containers on AWS. Companion to the Docker primer (§2.2's network namespaces are this document's Part 2 in miniature) and the data-access primer (its "a connection is a SQL session" is this document's Part 4). Browser/cookie landscape verified July 2026.*

The organising idea, and the one that makes the rest derivable: **networking is layers, and each layer only talks to the one directly above and below it.** Your Java code hands a string to a socket; the socket doesn't know about Ethernet, and Ethernet doesn't know about HTTP. Each layer wraps the one above it in its own envelope, ships it, and the receiving side unwraps in reverse. Every concept below — an IP address, a port, a TLS handshake, a cookie — lives at exactly one layer and does exactly one job. When something breaks, the diagnostic question is always **"which layer?"**, and Part 11 is the toolkit for answering it.

Contents:

- **Part 1** — the layered model (OSI, TCP/IP, encapsulation)
- **Part 2** — the network layer: IP, addressing, subnets, NAT, routing
- **Part 3** — DNS
- **Part 4** — the transport layer: TCP, UDP, ports, sockets
- **Part 5** — HTTP (1.1, 2, 3) and the application layer
- **Part 6** — cryptography foundations: symmetric, asymmetric, hashing, signatures
- **Part 7** — TLS/SSL: what the handshake actually does
- **Part 8** — certificates and PKI
- **Part 9** — cookies, sessions, tokens, and browser security
- **Part 10** — the infrastructure layer: proxies, load balancers, CDNs, AWS
- **Part 11** — the troubleshooting toolkit
- **Part 12** — when to use what

## Symptom index — open here when something's wrong

| Symptom | Layer | Go to |
|---|---|---|
| "Unknown host" / name doesn't resolve | DNS | §3 |
| "Connection refused" | transport | §4.4 — nothing is listening on that port |
| "Connection timed out" | network/firewall | §4.4 — packets vanishing; firewall/security group/route |
| "Connection reset" | transport | §4.4 — the other end sent RST |
| Works by IP, fails by hostname | DNS | §3 |
| Works from host, fails from container | container net | Docker primer §2.2, §7.5 — use service names, not localhost |
| `localhost` inside a container reaches nothing | container net | §2.2 — that's the container itself |
| Certificate error in browser | TLS/PKI | §8.4 — expired, wrong host, untrusted chain, incomplete chain |
| `PKIX path building failed` (Java) | PKI | §8.5 — the CA isn't in the JVM truststore |
| "Mixed content" blocked | HTTP/TLS | §5, §9 — HTTP resource on an HTTPS page |
| CORS error in browser console | browser policy | §9.5 — it's the *server's* headers, not a bug in your fetch |
| Cookie set but never sent back | cookies | §9.2 — domain/path/Secure/SameSite mismatch |
| Session lost when load-balanced | infra | §10.2 — no sticky sessions / no shared session store |
| Slow first byte, fast after | TCP/TLS | §4.2, §7.3 — handshake cost; check keep-alive/pooling |
| Postgres "too many connections" | transport/pool | data-access primer §1.6 — each connection is a TCP session |
| Random slow requests in prod only | network | §11 — latency, DNS TTL, or a pool, not your code |

---

# Part 1 — The layered model

## 1.1 OSI vs TCP/IP

Two models. **OSI** is the 7-layer teaching model (and the shared vocabulary — people say "layer 7" and "layer 4" constantly). **TCP/IP** is the 4-layer model that describes what actually runs. Learn OSI for the vocabulary; understand that reality is TCP/IP.

| OSI layer | Name | What it does | You'd recognise | TCP/IP |
|---|---|---|---|---|
| **7** | Application | the protocol your app speaks | HTTP, DNS, SMTP, **JDBC's Postgres protocol** | Application |
| 6 | Presentation | encoding, encryption | TLS *(sort of — see below)*, JSON, gzip | Application |
| 5 | Session | conversations | rarely distinct in practice | Application |
| **4** | Transport | end-to-end delivery, **ports** | **TCP, UDP** | Transport |
| **3** | Network | addressing, routing between networks | **IP**, ICMP, routers | Internet |
| 2 | Data link | frames on the local link | Ethernet, ARP, MAC addresses, switches | Link |
| 1 | Physical | actual signals | copper, fibre, radio | Link |

Two honesty notes. **Layers 5 and 6 are largely fiction** — real stacks fold them into the application. And **TLS doesn't fit cleanly**: it's usually called "layer 6" or "between 4 and 7," which tells you the model is a map, not the territory. What you actually need is layers **3, 4, and 7** — IP, TCP/UDP, and HTTP. Those three carry almost everything.

The numbers that matter in conversation: **"layer 4"** = TCP/port-level (a load balancer that forwards connections without reading them); **"layer 7"** = HTTP-aware (a load balancer that routes on URL path or host header). That distinction reappears in §10.

## 1.2 Encapsulation — the envelope model

Each layer wraps the layer above in its own header. Sending "GET /questions" to your API:

```
[ Ethernet header [ IP header [ TCP header [ TLS record [ HTTP request ] ] ] ] ]
  └ MAC addresses  └ IP addrs  └ ports      └ encrypted   └ your actual data
     (link)          (net)      (transport)   (crypto)      (application)
```

The receiving machine unwraps in reverse: the NIC reads the Ethernet frame, the kernel reads the IP header to confirm it's for this host, the TCP header says "port 8080 → give it to the process listening there," TLS decrypts, and your Micronaut controller finally sees `GET /questions`.

**Why this matters practically:** each header is overhead (~40 bytes for TCP/IP per packet), each layer is a place something can be dropped or misconfigured, and **each layer is a place you can debug**. `ping` tests layer 3. `telnet host 8080` tests layer 4. `curl` tests layer 7. That's the ladder in Part 11.

---

# Part 2 — The network layer: IP

## 2.1 What IP does, and doesn't

**IP (Internet Protocol)** does one job: get a packet from one address to another, across networks. It is:

- **Connectionless** — no handshake, no relationship. Each packet is independent.
- **Best-effort / unreliable** — packets may be dropped, duplicated, delayed, or arrive out of order. IP makes **no promises**.
- **Routed hop-by-hop** — each router reads the destination and forwards it one step closer, with no knowledge of the whole path.

Everything reliable (TCP) is built *on top of* this deliberately unreliable foundation. That's the design: keep the network dumb and fast, put the intelligence at the endpoints.

## 2.2 Addresses

**IPv4** — 32 bits, four octets: `192.168.1.10`. ~4.3 billion addresses, exhausted years ago (hence NAT, §2.4). **IPv6** — 128 bits: `2001:db8::1`. Effectively unlimited; adoption is now substantial but IPv4 still dominates internal networks.

Addresses you must recognise on sight:

| Range | Meaning |
|---|---|
| `127.0.0.1` / `::1` | **loopback** — this machine, never leaves it |
| `10.0.0.0/8` | **private** (16.7M addrs) — the AWS VPC default |
| `172.16.0.0/12` | **private** — Docker's default bridge lives here (`172.17.x`) |
| `192.168.0.0/16` | **private** — home routers |
| `169.254.0.0/16` | link-local — includes AWS's `169.254.169.254` metadata endpoint |
| `0.0.0.0` | "all interfaces" when binding; "any" in a route |

**Private vs public** is the whole security model of a VPC: private ranges are not routable on the internet, so a Postgres in `10.0.2.x` is unreachable from outside by construction, not by permission.

> **The tell — binding:** `0.0.0.0` means "listen on every interface"; `127.0.0.1` means "listen only for connections from this machine." Bind a container's server to `127.0.0.1` and **the host can't reach it even with `-p` published** — the container's loopback is its own. That's why the Docker primer's FastAPI example uses `--host 0.0.0.0`. It's the single most common containerised-app networking bug.

## 2.3 Subnets and CIDR

A **subnet** splits an address into a *network part* and a *host part*. **CIDR notation** — `10.0.1.0/24` — says "the first 24 bits are the network; the remaining 8 identify hosts within it."

| CIDR | Netmask | Usable hosts | Typical use |
|---|---|---|---|
| `/32` | 255.255.255.255 | 1 | a single host (security group rules) |
| `/24` | 255.255.255.0 | 254 | a small subnet — the classic |
| `/16` | 255.255.0.0 | 65,534 | a VPC |
| `/8` | 255.0.0.0 | 16.7M | huge |

The arithmetic: **smaller number = bigger network.** `/24` gives 2⁸ = 256 addresses, minus network and broadcast = 254 usable. (AWS also reserves 3 more per subnet, so a `/24` gives you 251.)

Why you care: **a VPC is CIDR blocks**. `10.0.0.0/16` for the VPC, `10.0.1.0/24` as a public subnet (the load balancer), `10.0.2.0/24` as a private subnet (`practiq-api` on Fargate), `10.0.3.0/24` for RDS. The subnet boundary *is* the security boundary — and it's why "the DB is in a private subnet" is a stronger statement than any firewall rule.

## 2.4 NAT

**Network Address Translation** lets many private addresses share one public one. Your router rewrites the source address of outbound packets to its own public IP, remembers the mapping, and rewrites replies back. This is why IPv4 survived exhaustion, and why:

- Outbound works by default; **inbound doesn't** — there's no mapping until you make one (port forwarding).
- In AWS, a private-subnet task reaching the internet (pulling an image, calling an API) needs a **NAT Gateway**. It's also a line item — NAT Gateway data charges surprise people.
- **Docker does exactly this**: `-p 8080:8080` is a NAT rule (iptables/DNAT) mapping the host's port into the container's network namespace.

## 2.5 Routing, and the ICMP tools

Every host has a **routing table**: "for this destination range, send to this next hop." The **default gateway** is the catch-all. Routers repeat this until the packet arrives or its **TTL** (hop counter) hits zero and it's discarded — which is exactly what `traceroute` exploits, sending packets with TTL=1, 2, 3… and collecting the "time exceeded" replies to map the path.

**ICMP** is IP's diagnostic protocol — `ping` (echo request/reply), "destination unreachable," "time exceeded." Note ICMP is **often blocked** by firewalls and security groups: *"ping doesn't work" does not mean "the host is down."* It's a weak signal, and in AWS a wrong one by default.

---

# Part 3 — DNS

## 3.1 What it does

DNS maps names to addresses. `api.practiq.io` → `52.30.1.4`. It's a distributed, hierarchical, heavily-cached database — and it's the single most common cause of "it works on my machine."

The hierarchy reads **right to left**: `api.practiq.io.` → root (`.`) → TLD (`.io`) → authoritative nameserver for `practiq.io` → the record for `api`.

## 3.2 Resolution

1. Your app asks the OS resolver ("stub resolver").
2. It checks its cache, then `/etc/hosts` — **a hosts entry beats DNS entirely**, which is both a debugging tool and a lurking trap.
3. It asks a **recursive resolver** (your ISP's, or `8.8.8.8`/`1.1.1.1`).
4. The resolver walks root → TLD → authoritative, caching each answer for its **TTL**.
5. The answer comes back and is cached at every level — including inside your JVM.

## 3.3 Record types

| Record | Maps to | Notes |
|---|---|---|
| **A** | an IPv4 address | the basic one |
| **AAAA** | an IPv6 address | |
| **CNAME** | another *name* | can't coexist with other records at the same name; not allowed at the apex |
| **ALIAS/ANAME** | a name, apex-safe | provider-specific (Route 53 "Alias") — how you point `practiq.io` at an ALB |
| **MX** | mail servers | |
| **TXT** | arbitrary text | SPF/DKIM, domain-ownership proofs, **ACME certificate validation** (§8.3) |
| **NS** | the authoritative nameservers | delegation |
| **SRV** | host + port | service discovery |

## 3.4 TTL and caching — where the pain lives

Every record carries a **TTL** in seconds. Change an IP with a 24-hour TTL and some clients keep the old one for a day. Hence the standard migration move: **drop the TTL to 60s a day before the change**, cut over, then raise it again.

**The JVM-specific trap** worth knowing as a Java engineer: the JVM caches DNS lookups itself, and historically cached successful lookups **forever** when a security manager was installed (`networkaddress.cache.ttl`). On modern JDKs the default is a modest positive TTL (typically 30s), but **negative** lookups are cached too (`networkaddress.cache.negative.ttl`). If your app resolves an ALB or RDS endpoint once at startup and holds it, a failover that changes the IP can leave you talking to nothing. Long-lived clients should honour TTLs; connection pools generally re-resolve on new connections.

> **The tell — DNS:** if it works by IP and fails by name, it's DNS. If it worked an hour ago and not now, suspect TTL/caching before suspecting code. In containers, DNS resolves **service names** via Docker's/ECS's embedded resolver — `postgres`, not `localhost` (Docker primer §7.5).

---

# Part 4 — The transport layer: TCP and UDP

IP gets a packet to a *host*. The transport layer gets it to the right *process* on that host, and decides what reliability you get.

## 4.1 Ports and sockets

A **port** is a 16-bit number (1–65535) identifying a process's endpoint on a host. A **socket** is the pair `(IP, port)`; a **connection** is the four-tuple `(source IP, source port, dest IP, dest port)` — which is how a server can hold thousands of simultaneous connections on one port: each has a different client IP/port.

| Port | Service |
|---|---|
| 22 | SSH |
| 53 | DNS |
| 80 | HTTP |
| 443 | HTTPS |
| **5432** | **PostgreSQL** |
| 8080 | HTTP alternate — the Micronaut default |
| 6379 / 27017 | Redis / MongoDB |

Ports below 1024 are **privileged** (root to bind) — which is why containers often run the app on 8080 and let a load balancer own 443.

## 4.2 TCP — reliable, ordered, connection-oriented

TCP builds reliability on top of unreliable IP. It gives you:

- **Connection setup** — the **three-way handshake**:
  ```
  Client ──SYN──────────▶ Server      "let's talk, my sequence starts at X"
  Client ◀──SYN-ACK───── Server      "ok, and mine starts at Y"
  Client ──ACK──────────▶ Server      "agreed"           → connection ESTABLISHED
  ```
  That's **one full round trip before a single byte of your data moves** — the reason connection *reuse* (keep-alive, pooling) matters so much. On a 100ms-RTT link, every new connection costs 100ms before it does anything, and add TLS (§7.3) and it's worse.
- **Ordering** — every byte is numbered; the receiver reassembles in order.
- **Reliability** — unacknowledged data is retransmitted.
- **Flow control** — the receiver advertises a window so a fast sender can't overwhelm it.
- **Congestion control** — the sender backs off when the *network* is saturated (slow start, congestion avoidance). This is why TCP throughput ramps rather than starting flat out.
- **Teardown** — FIN/ACK both ways (or an abrupt **RST**).

**Connection states you'll see in `ss`/`netstat`:** `LISTEN` (waiting), `ESTABLISHED` (open), `TIME_WAIT` (the closer waits ~2×MSL to absorb stray packets — thousands of these is normal on a busy client, not a leak), `CLOSE_WAIT` (**the other end closed and your app hasn't** — piles of these *is* an application bug: unclosed sockets).

## 4.3 UDP — fast, unreliable, connectionless

UDP is a thin wrapper over IP: source port, dest port, length, checksum. No handshake, no ordering, no retransmission, no congestion control. **No round trip before you send.**

That's not a deficiency — it's a choice. If you're streaming live video, a packet from 400ms ago is *worthless*; TCP retransmitting it stalls everything behind it (head-of-line blocking) to deliver something you no longer want. UDP lets you drop it and move on.

| | **TCP** | **UDP** |
|---|---|---|
| Connection | handshake first | none — just send |
| Reliability | guaranteed, retransmits | none |
| Ordering | guaranteed | none |
| Congestion control | yes | no (you build it) |
| Header | 20+ bytes | 8 bytes |
| Latency | RTT before data | none |
| Use for | **HTTP, DB connections, anything where correctness matters** | DNS, video/voice, gaming, metrics, **QUIC/HTTP-3** |

> **The tell — TCP or UDP:** ask *"is a late packet still valuable?"* Yes (a database write, an API response, a file) → TCP. No (live audio, a position update, a metric sample) → UDP. Everything Practiq does is TCP: HTTPS to the API, JDBC to Postgres. The interesting modern exception is **HTTP/3, which runs over UDP** — and then rebuilds reliability in userspace (§5.3), precisely so it can avoid TCP's head-of-line blocking.

## 4.4 Reading the failure modes

This is the highest-value thing in this Part — the errors are diagnostic if you know what they mean:

| Error | Means | Look at |
|---|---|---|
| **Connection refused** | the packet **arrived**, and nothing is listening on that port | wrong port, process down, bound to `127.0.0.1` not `0.0.0.0` |
| **Connection timed out** | packets went into a void — no reply at all | firewall/security group dropping, wrong IP, no route, wrong subnet |
| **Connection reset (RST)** | something actively terminated it | app crashed mid-request, proxy timeout, LB idle timeout |
| **Host unreachable** | routing has no path | route table, gateway |
| DNS/unknown host | never got to layer 4 | §3 |

"Refused" vs "timed out" is the fork worth internalising: **refused = you reached the host, wrong port. Timed out = you didn't reach the host** (or something silently dropped you). Refused is a config problem; timed out is a firewall/routing problem. That single distinction resolves most "can't connect to the database" incidents.

---

# Part 5 — HTTP and the application layer

## 5.1 The model

HTTP is a **request/response, stateless** text protocol. Stateless is the key word: **the server remembers nothing between requests** — every request stands alone. Everything that feels stateful (being logged in, a shopping basket) is re-established per request by something the client sends: a cookie, a token, a header. That's the entire reason Part 9 exists.

```http
GET /api/v1/questions?conceptId=42 HTTP/1.1
Host: api.practiq.io
Accept: application/json
Authorization: Bearer eyJhbGci...
Cookie: session=abc123

HTTP/1.1 200 OK
Content-Type: application/json
Cache-Control: max-age=60
Set-Cookie: session=abc123; HttpOnly; Secure; SameSite=Lax

[{"id":1,"stem":"..."}]
```

## 5.2 Methods and status codes

| Method | Semantics | Safe? | Idempotent? |
|---|---|---|---|
| GET | read | yes | yes |
| POST | create / arbitrary action | no | **no** |
| PUT | replace wholesale | no | **yes** |
| PATCH | partial update | no | not necessarily |
| DELETE | remove | no | **yes** |
| HEAD / OPTIONS | headers only / capabilities (CORS preflight) | yes | yes |

**Idempotent** = doing it twice has the same effect as once. It's not trivia: it decides what a client/proxy may safely **retry**. A dropped PUT can be retried; a dropped POST might double-charge someone. That's why "create" endpoints often take an idempotency key.

| Class | Meaning | The ones to know |
|---|---|---|
| **2xx** | success | 200 OK, 201 Created, 204 No Content |
| **3xx** | redirect | 301 permanent, 302/307 temporary, **304 Not Modified** (caching) |
| **4xx** | **client** error | 400 bad request, **401 unauthenticated**, **403 unauthorised**, 404, 409 conflict (**your `@Version` collision**), 422, 429 rate-limited |
| **5xx** | **server** error | 500, 502 bad gateway, 503 unavailable, 504 gateway timeout |

401 vs 403 is the perennial mix-up: **401 = I don't know who you are** (authenticate); **403 = I know who you are and you can't** (authorise).

## 5.3 The versions

| | **HTTP/1.1** (1997) | **HTTP/2** (2015) | **HTTP/3** (2022) |
|---|---|---|---|
| Transport | TCP | TCP | **UDP (QUIC)** |
| Format | text | binary | binary |
| Concurrency | one request at a time per connection (browsers open ~6) | **multiplexed** streams on one connection | multiplexed, **no TCP HOL blocking** |
| Headers | plain, repeated | HPACK compressed | QPACK compressed |
| Handshake | TCP (+TLS separately) | TCP + TLS | **QUIC merges transport + TLS 1.3 → ~1 RTT (0-RTT on resume)** |

The story in one line: HTTP/1.1's problem was **head-of-line blocking** at the HTTP layer (one slow response blocks the connection). HTTP/2 fixed that with multiplexing — but a lost TCP packet still stalls *every* stream, because TCP insists on in-order delivery. So HTTP/3 abandoned TCP, moved to **QUIC over UDP**, and rebuilt reliability per-stream in userspace where one stream's loss doesn't block the others. Note QUIC has **TLS 1.3 baked in** — HTTP/3 is encrypted by definition, and Java's `HttpClient` gained HTTP/3 support in **Java 26** (Java primer §17).

## 5.4 Caching and conditional requests

```http
Cache-Control: max-age=3600, public      # cache for an hour, any cache may
Cache-Control: no-store                  # never cache (auth responses)
ETag: "a1b2c3"                           # a version tag for this representation
```
Next time, the client sends `If-None-Match: "a1b2c3"`; if unchanged, the server returns **304 Not Modified** with no body. Free bandwidth. `Cache-Control` is the modern header; `Expires`/`Pragma` are legacy.

## 5.5 WebSockets, briefly

HTTP is request/response — the server can't initiate. **WebSockets** upgrade an HTTP connection (`Upgrade: websocket`, 101 Switching Protocols) into a persistent, bidirectional TCP channel over the same port 443. Use when you need **server push** (live collaboration, notifications). If you only need occasional server→client updates, **Server-Sent Events** (SSE — plain HTTP, one-way, auto-reconnecting) is much simpler and often the right answer.

---

# Part 6 — Cryptography foundations

Everything in Parts 7–8 is built from four primitives. Get these and TLS stops being magic.

## 6.1 Symmetric encryption

**One shared key** encrypts and decrypts. Fast — hardware-accelerated, gigabytes per second. **AES** (typically AES-256-GCM) and ChaCha20-Poly1305 are the ones in use.

The problem is right there in the name: **both sides need the same secret key**. How do you give it to someone you've never met, over a network an attacker can read? That question is the *entire reason* asymmetric crypto exists.

## 6.2 Asymmetric (public-key) encryption

**Two mathematically related keys.** What one encrypts, only the other can decrypt.

- **Public key** — give it to everyone. Publish it.
- **Private key** — never leaves your machine. Ever.

Two distinct uses fall out, and confusing them is the classic error:

| Goal | Encrypt with | Decrypt with | Because |
|---|---|---|---|
| **Confidentiality** — only they can read it | **their public** key | their private key | only the holder of the private key can open it |
| **Authenticity** — prove *I* wrote it | **my private** key (a signature) | my public key | only I could have produced something my public key opens |

Algorithms: **RSA** (older, key sizes 2048/4096) and **ECDSA/Ed25519** (elliptic curve — much smaller keys for equivalent strength; a 256-bit EC key ≈ a 3072-bit RSA key, so it's the modern default).

The catch: asymmetric crypto is **slow** — orders of magnitude slower than symmetric. You'd never encrypt a video stream with RSA.

> **The tell — the hybrid insight:** this is *the* idea. **Use slow asymmetric crypto once, to agree on a fast symmetric key; then use symmetric crypto for the actual data.** That single sentence is what TLS does (§7.2). Asymmetric solves key exchange; symmetric does the work.

## 6.3 Hashing

A **one-way** function: arbitrary input → fixed-size digest. Same input always gives the same digest; you can't reverse it; changing one bit changes everything.

- **SHA-256** — the workhorse (integrity, fingerprints, **Docker image digests** — Docker primer §3.1, **Git commit IDs**).
- **MD5, SHA-1** — **broken**. Collisions are practical. Never for security.
- **Passwords need slow, salted hashes** — **bcrypt, scrypt, Argon2**, not SHA-256. The whole point is to be *expensive* so brute force is infeasible; SHA-256 is far too fast, and a salt stops one rainbow table cracking every user at once.

## 6.4 Signatures and MACs

A **digital signature**: hash the message, encrypt the hash with your **private** key. Anyone with your public key can verify it — proving **integrity** (unaltered) and **authenticity** (you wrote it) and giving **non-repudiation** (you can't deny it). This is the mechanism behind certificates (§8) and JWTs (§9.3).

A **MAC/HMAC** does the same job with a *shared symmetric* key — faster, but both sides can produce it, so no non-repudiation.

**Key exchange (Diffie–Hellman)** deserves its own mention: DH lets two parties derive a shared secret over a public channel *without ever transmitting it*. The ephemeral variants (**ECDHE**) generate fresh keys per session, which buys **forward secrecy** — steal the server's private key tomorrow and you *still* can't decrypt traffic you recorded today, because that session's key was thrown away. This is why every modern TLS suite uses ECDHE.

---

# Part 7 — TLS/SSL

## 7.1 What it is and what it gives you

**SSL is the dead ancestor; TLS is the actual protocol.** Everyone still says "SSL" (and "SSL certificate"), but SSL 2.0/3.0 are long broken and disabled. What runs today is **TLS 1.2** (fine, widespread) and **TLS 1.3** (2018 — faster, simpler, the default target). TLS 1.0/1.1 are deprecated and disabled everywhere reputable.

"HTTPS" is just **HTTP inside a TLS tunnel**, on port 443. TLS provides exactly three things:

1. **Confidentiality** — nobody in the middle can read it (symmetric encryption, §6.1).
2. **Integrity** — nobody can modify it undetected (AEAD/MAC).
3. **Authenticity** — you're talking to who you think (**certificates**, §8).

That third one is the one people forget, and it's the important one. Encryption without authentication is worthless: you'd have a beautifully private conversation *with the attacker*.

## 7.2 The handshake (TLS 1.3)

```
Client ──ClientHello──────────────────▶      supported versions/ciphers + a DH key share
       ◀─ServerHello, Certificate,──── Server  its DH share, its CERTIFICATE, a signature
         CertificateVerify, Finished
Client ──Finished────────────────────▶      verified the cert; keys derived
       ◀════ encrypted application data ════▶  symmetric from here on
```

What actually happened, in the terms of Part 6:

1. Both sides contribute **ECDHE** key shares and independently derive the same **shared symmetric key** — never transmitted (§6.4). Fresh per session → **forward secrecy**.
2. The server sends its **certificate** (its public key + identity, signed by a CA) and a **signature** made with its **private key**. The client verifies the signature against the cert's public key — proving the server *holds the private key* for that certificate. **This is the authentication step**, and it's the whole game: anyone can send you a copy of Google's certificate; only Google can sign with Google's private key.
3. The client validates the certificate chain (§8.2).
4. Everything after is **AES-GCM** with the derived key — fast symmetric crypto (§6.1).

That's the hybrid insight from §6.2 made concrete: asymmetric to authenticate and agree a key, symmetric for the data.

## 7.3 Why TLS 1.3 matters

TLS 1.2 needed **2 round trips** to handshake; TLS 1.3 needs **1** — and **0-RTT** on resumption (send data with the first packet). On top of TCP's own handshake RTT (§4.2), a fresh HTTPS connection to a 100ms-away server used to cost ~300ms before a byte of your request moved; TLS 1.3 makes it ~200ms, and connection reuse makes it ~0. TLS 1.3 also **removed the broken options** — no RSA key transport (so forward secrecy is mandatory), no CBC, no RC4, no compression. Fewer choices, fewer footguns.

> **The tell — TLS:** terminate TLS at the **load balancer** (AWS ALB + ACM, free certs, auto-renewal) and let it speak plain HTTP to your container inside the VPC — that's the standard AWS pattern and why `practiq-api` listens on 8080, not 443. Use **mTLS** (§8.6) only when both ends must authenticate — service-to-service in a zero-trust mesh, not for a public API. Never disable certificate verification to "make it work" — you've deleted the only thing protecting you from a man-in-the-middle.

---

# Part 8 — Certificates and PKI

## 8.1 The problem certificates solve

TLS proves the server holds a private key. It does **not**, by itself, prove *whose* key it is. An attacker can generate a perfectly good keypair and hand you the public half — you'd establish flawless encryption with them.

So you need to bind **an identity** ("api.practiq.io") to **a public key**, and have that binding vouched for by someone you already trust. That binding, signed, is a **certificate**. The system of trust around it is **PKI**.

## 8.2 X.509, chains, and the root of trust

A **certificate** (X.509) contains: the **subject** (who — the domain), the **subject's public key**, the **issuer** (which CA vouched), a **validity window**, **SANs** (the domains it's actually valid for — the modern field; CN is legacy), key usage constraints, and the **CA's signature** over all of it.

Trust is a **chain**:

```
Root CA (self-signed; in your OS/browser/JVM trust store)
   └─ signs → Intermediate CA
         └─ signs → api.practiq.io   ← your leaf certificate
```

The client verifies each signature up the chain until it reaches a **root it already trusts**. Roots are the anchor: they ship pre-installed in your OS, browser, and the JVM's `cacerts`. That's the leap of faith at the bottom — you trust ~150 root CAs because Mozilla/Microsoft/Apple/Oracle vetted them. Intermediates exist so the root's private key can stay offline in a vault; if an intermediate is compromised, revoke it without burning the root.

**The classic deployment bug:** you install the leaf but not the intermediate. It works in your browser (which cached the intermediate from another site) and fails in `curl`/Java (which didn't). Always serve the **full chain**.

## 8.3 Getting one

| Type | Proves | Use |
|---|---|---|
| **DV** (domain validated) | you control the domain | **almost everything** — Let's Encrypt |
| OV / EV | + organisation vetted | legacy corporate expectation; browsers no longer show the green bar |
| **Wildcard** (`*.practiq.io`) | any one-level subdomain | many subdomains |
| **Self-signed** | nothing — it vouches for itself | **local dev and internal only** |

**Let's Encrypt** made DV certs free and automatic via the **ACME** protocol: prove control (HTTP-01 — serve a token at a URL; or DNS-01 — publish a TXT record, the only option for wildcards), get a 90-day cert, renew automatically. On AWS, **ACM** does the equivalent for free with auto-renewal, attached to an ALB/CloudFront.

The mechanics of a request: generate a **private key** (never leaves you), create a **CSR** (a request containing your public key + identity, self-signed to prove you hold the private key), send the CSR to the CA, receive the signed certificate back.

```bash
# generate a key and CSR
openssl req -new -newkey rsa:2048 -nodes -keyout practiq.key -out practiq.csr
# a self-signed cert for local dev
openssl req -x509 -newkey rsa:2048 -nodes -keyout dev.key -out dev.crt -days 365
```

**Expiry is an outage waiting to happen** — certs are short-lived by design (90 days is now normal, and the industry is moving shorter). Automate renewal or you *will* be paged. Revocation exists (CRL, OCSP, OCSP stapling) but works poorly in practice, which is precisely *why* validity periods keep shrinking.

## 8.4 Why browsers reject a certificate

Five reasons, and the error usually names one:

1. **Expired** (or not yet valid — check the clock; a wrong system clock breaks TLS).
2. **Hostname mismatch** — the SANs don't cover the name you typed.
3. **Untrusted issuer** — self-signed, or a CA not in the trust store.
4. **Incomplete chain** — the intermediate wasn't served (§8.2).
5. **Revoked**.

## 8.5 The Java angle

Java has its own trust store — this is the bit that catches JVM engineers:

- **Truststore** (`$JAVA_HOME/lib/security/cacerts`) — the CAs *you trust*. Verifying others.
- **Keystore** — *your own* private key + cert. Proving who you are.

`PKIX path building failed: unable to find valid certification path to requested target` means exactly one thing: **the CA isn't in the JVM's truststore.** Usually an internal/corporate CA, or a self-signed dev cert. The fix is to import it — **not** to disable verification:

```bash
keytool -importcert -alias internal-ca -file ca.crt \
        -keystore $JAVA_HOME/lib/security/cacerts -storepass changeit
# or point the JVM at your own truststore
java -Djavax.net.ssl.trustStore=/app/truststore.jks \
     -Djavax.net.ssl.trustStorePassword=... -jar app.jar
# debug a handshake
java -Djavax.net.debug=ssl:handshake -jar app.jar
```

## 8.6 mTLS

Normal TLS: the **server** proves itself, the client is anonymous (and authenticates at layer 7 with a token). **Mutual TLS**: the **client presents a certificate too**, so both ends are cryptographically authenticated at the transport layer. Use for service-to-service in a zero-trust network, or high-security B2B APIs. Don't use it for a public web app — you'd have to provision a cert to every user's browser.

---

# Part 9 — Cookies, sessions, tokens, browser security

HTTP is stateless (§5.1). This Part is everything built to work around that — and the browser security model that guards it.

## 9.1 What a cookie is

A small key-value string the server asks the browser to store and send back on every subsequent matching request.

```http
Set-Cookie: session=abc123; Domain=practiq.io; Path=/; Max-Age=3600;
            Secure; HttpOnly; SameSite=Lax
```
```http
Cookie: session=abc123
```
That's it. The state lives server-side (or in the token); the cookie is just the handle.

## 9.2 The attributes — this table is the security model

| Attribute | Does | Get it wrong and… |
|---|---|---|
| `Domain` | which hosts get it (a domain cookie also goes to subdomains) | too broad → leaked to subdomains you don't control |
| `Path` | which paths get it | it's silently never sent |
| `Expires`/`Max-Age` | persistent lifetime; **omit → session cookie**, dies with the browser | |
| **`Secure`** | **HTTPS only** | sent in cleartext over HTTP |
| **`HttpOnly`** | **invisible to JavaScript** | **XSS can steal your session** |
| **`SameSite`** | whether it rides on cross-site requests | **CSRF** |
| `__Host-` / `__Secure-` prefix | browser-enforced constraints | — |

**`SameSite`** is worth its own line because it's the modern CSRF defence:

- `Strict` — never sent cross-site. Safest; a link from another site to your app arrives logged-out.
- **`Lax`** — sent on top-level GET navigations only. **The browser default** now, and the right choice for a session cookie.
- `None` — always sent, **requires `Secure`**. Needed for genuine third-party contexts.

> **The tell — session cookies:** `HttpOnly; Secure; SameSite=Lax`, plus a narrow `Domain`/`Path`. That combination kills the two biggest cookie attacks (XSS theft, CSRF) with three words.

**The third-party cookie landscape (verified July 2026)** — worth being accurate about, because the received wisdom is out of date: **Safari and Firefox have blocked third-party cookies by default for years** (since ~2019–2020). **Google abandoned its Chrome deprecation plan in July 2024**, and as of 2026 Chrome still allows third-party cookies by default, leaving it to user settings; most of Privacy Sandbox was shut down in October 2025. So "third-party cookies are going away" is now wrong as stated — roughly half the web is already cookieless via Safari/Firefox, while Chrome keeps them. **None of this affects Practiq**: first-party session cookies are unaffected by any of it. It matters for ad tech and cross-site tracking, not for your own app's login.

## 9.3 Sessions vs tokens

| | **Server-side session** | **JWT (stateless token)** |
|---|---|---|
| Server stores | session data (memory/Redis/DB) | **nothing** |
| Client holds | an opaque session ID | a **signed** token containing claims |
| Revoke | delete it — **instant** | **you can't** (until it expires) — needs a denylist |
| Scaling | needs shared store or sticky sessions | scales freely |
| Size | tiny | can get large; sent on every request |
| Best for | web apps, anything needing instant logout | APIs, service-to-service, short-lived access |

A **JWT** is `header.payload.signature`, base64url-encoded, dot-separated. The critical points: it's **signed, not encrypted** — anyone can read the payload (`jwt.io`, or `base64 -d`), so **never put secrets in it**. The signature only proves it wasn't tampered with. And **you cannot revoke it** — a stolen token is valid until expiry, which is why access tokens should be short-lived (minutes) and paired with a revocable refresh token.

> **The tell — sessions or JWTs:** the "JWTs are stateless so they're better" reasoning is mostly cargo cult. **Instant logout matters** for a human-facing app, and stateless means you can't have it. Default: **server-side sessions for a browser app** (a session ID in a `HttpOnly; Secure; SameSite=Lax` cookie); **short-lived JWTs for APIs/service-to-service**. If you use JWTs for a browser app, storing them in `localStorage` re-exposes you to XSS theft — the cookie was protecting you.

## 9.4 Same-Origin Policy

An **origin** is `scheme://host:port` — all three. `https://practiq.io` and `https://api.practiq.io` are **different origins**; so are `http://` and `https://` of the same host, and `:8080` vs `:80`.

The **Same-Origin Policy** is the browser's foundational rule: JavaScript from one origin can't read responses from another. Without it, any site you visit could read your webmail using your logged-in cookies. Note it blocks *reading the response*, not *sending the request* — which is exactly the gap CSRF exploits (the request goes, with cookies; the attacker just can't read the reply), and why `SameSite` exists.

## 9.5 CORS — and the thing everyone gets wrong

**CORS** is how a server *opts in* to relaxing the Same-Origin Policy. The essential insight: **CORS is not a bug in your frontend and not something you fix in JavaScript. It's a header the *server* must send.**

```http
Access-Control-Allow-Origin: https://practiq.io
Access-Control-Allow-Methods: GET, POST, PUT, DELETE
Access-Control-Allow-Headers: Content-Type, Authorization
Access-Control-Allow-Credentials: true          # required to send cookies
Access-Control-Max-Age: 86400                   # cache the preflight
```

For anything beyond a "simple" request, the browser sends a **preflight** `OPTIONS` first and only proceeds if the response permits it. Two traps: `Allow-Origin: *` **cannot** be combined with `Allow-Credentials: true` (you must echo a specific origin); and CORS errors are reported to your JS as an opaque network failure, which is why they're so confusing — the request often *succeeded* server-side, the browser just refused to hand you the response.

This is live for Practiq the moment `practiq-frontend` and `practiq-api` are on different origins.

## 9.6 The attacks, in one place

| Attack | Is | Defence |
|---|---|---|
| **XSS** | attacker's JS runs in your page | escape output, CSP, **`HttpOnly` cookies** |
| **CSRF** | attacker's site makes an authenticated request as you | **`SameSite`**, CSRF tokens, check `Origin` |
| **MITM** | someone reads/alters traffic in transit | **TLS + cert verification**, HSTS |
| **Session fixation** | attacker plants a known session ID | **regenerate the session ID on login** |
| **Clickjacking** | your page iframed invisibly over theirs | `X-Frame-Options`/CSP `frame-ancestors` |

Two headers worth knowing: **HSTS** (`Strict-Transport-Security: max-age=31536000; includeSubDomains`) tells browsers "only ever reach me over HTTPS," closing the initial-HTTP-redirect MITM window. **CSP** (`Content-Security-Policy`) whitelists what may load/execute — the strongest XSS mitigation there is.

---

# Part 10 — The infrastructure layer

## 10.1 Proxies

- **Forward proxy** — sits in front of *clients* (corporate egress, the Docker build proxy). The client knows it's there.
- **Reverse proxy** — sits in front of *servers* (nginx, ALB). The *client* doesn't know; it thinks the proxy is the server. This is the one you deploy: TLS termination, routing, compression, static files, rate limiting.

## 10.2 Load balancers — L4 vs L7

Straight from §1.1's layer vocabulary:

| | **L4 (transport)** | **L7 (application)** |
|---|---|---|
| Sees | IP + port | the full HTTP request |
| Routes on | connection tuple | **path, host header, cookies** |
| Can it terminate TLS? | pass-through (or terminate) | **yes** — the usual |
| AWS | **NLB** | **ALB** |
| Faster? | yes, less work | slightly more overhead |
| Use for | raw TCP, extreme throughput, static IPs | **HTTP APIs — the default** |

Algorithms: round-robin, least-connections, IP-hash (a poor man's stickiness). **Health checks** are the real value — the LB stops sending traffic to a failing instance, which is why your app should expose a health endpoint (Docker primer's `HEALTHCHECK`, Micronaut's `/health`).

**The stateless consequence:** if request 1 hits instance A and request 2 hits instance B, an in-memory session is gone. That's why §9.3's "shared store or sticky sessions" matters, and why stateless services are the default — the LB can then send any request anywhere.

## 10.3 The rest of the furniture

- **CDN** (CloudFront) — caches static assets at edge locations near users. Cuts latency and origin load. `practiq-frontend`'s built assets are the textbook case.
- **API Gateway** — auth, rate limiting, routing at the edge.
- **Firewall / Security Group** — packet filtering. AWS **security groups** are **stateful** (allow outbound → the reply is automatically allowed) and **default-deny inbound**; **NACLs** are stateless and operate at the subnet. Security groups can reference *other security groups* — "allow 5432 from the API's SG" rather than from a CIDR, which is the clean way to say what you mean.
- **VPC** — your private network (§2.3). Public subnet = the ALB; private subnets = Fargate tasks and RDS.

**Practiq's likely production path, layer by layer:**

```
Browser
  │ DNS: practiq.io → Route 53 (§3)
  │ TLS 1.3 handshake, ACM certificate (§7, §8)
  ▼
CloudFront (CDN, static assets)  ─┐
  ▼                               │ same origin? → no CORS (§9.5)
ALB  (L7, TLS terminated here, :443)
  │ plain HTTP :8080 inside the VPC
  ▼
ECS Fargate task — practiq-api container (Docker primer §2.2: its own net namespace)
  │ TCP :5432, private subnet, SG allows only the API's SG
  ▼
RDS Postgres  (data-access primer §1.5: each connection = a session)
```

Every arrow in that diagram is a layer from this document. That's the map.

---

# Part 11 — The troubleshooting toolkit

**Work up the layers.** This ladder resolves most incidents without guessing:

```bash
# L3 — is the host reachable? (ICMP is often blocked — a weak signal)
ping api.practiq.io
traceroute api.practiq.io          # where does the path die?

# DNS — does the name resolve, and to what?
dig api.practiq.io                 # +short for just the answer
dig @8.8.8.8 api.practiq.io        # bypass the local resolver
nslookup api.practiq.io

# L4 — is anything listening on that port?
nc -zv api.practiq.io 443          # netcat — the fastest "is the port open"
telnet db.internal 5432
ss -tlnp                           # what's listening locally (modern netstat)
ss -tan | grep TIME_WAIT | wc -l   # connection-state census

# L7 — does the application respond?
curl -v https://api.practiq.io/health          # -v shows DNS, TCP, TLS, headers
curl -I https://api.practiq.io                 # headers only
curl -w '@curl-format.txt' -o /dev/null -s URL # timing breakdown per phase

# TLS/certs
openssl s_client -connect api.practiq.io:443 -servername api.practiq.io
openssl x509 -in cert.crt -text -noout          # read a certificate
openssl x509 -in cert.crt -noout -dates         # just the validity window
echo | openssl s_client -connect host:443 2>/dev/null | openssl x509 -noout -dates

# packets — when nothing else explains it
sudo tcpdump -i any port 5432 -nn
```

`curl -v` is the highest-value single command: it shows you the DNS resolution, the TCP connect, the TLS handshake and certificate, the request headers, and the response — **four layers in one output**.

> **The tell — diagnosis:** don't guess, **climb**. Resolve (DNS) → reach (ping/route) → connect (port) → speak (curl). The layer where it first fails is the layer to fix. And remember §4.4: **refused = wrong port, timed out = firewall/routing.**

---

# Part 12 — When to use what

**A. TCP or UDP.** Tell → TCP: a late packet is still valuable (APIs, DB, files). Tell → UDP: it isn't (live media, metrics, gaming) — or you're building QUIC. Default: **TCP for everything Practiq does.**

**B. HTTP version.** Tell → HTTP/2: default, free multiplexing (ALB/CloudFront do it for you). Tell → HTTP/3: mobile/lossy networks, latency-critical. Tell → HTTP/1.1: simple internal calls, debugging. Default: **let the LB/CDN negotiate; don't think about it.**

**C. TLS termination.** Tell → at the LB: the standard (ACM certs, free, auto-renewed). Tell → end-to-end: compliance requires encryption inside the VPC too. Tell → mTLS: both ends must authenticate (service mesh, B2B). Default: **terminate at the ALB, plain HTTP inside the VPC.**

**D. Certificate source.** Tell → ACM: you're on AWS behind an ALB/CloudFront — free and auto-renewing. Tell → Let's Encrypt: anywhere else. Tell → self-signed: local dev only. Tell → paid OV/EV: a compliance box needs ticking. Default: **ACM in AWS, Let's Encrypt elsewhere.**

**E. Session or JWT.** Tell → server session: browser app, instant logout matters. Tell → short-lived JWT: API/service-to-service, horizontal scale. Default: **session cookie for the web app, JWT for machine callers.** Don't put a JWT in `localStorage`.

**F. Cookie attributes.** Default, non-negotiable: **`HttpOnly; Secure; SameSite=Lax`**, narrow `Domain`/`Path`. `SameSite=None` only for a genuine third-party context, and then `Secure` is mandatory.

**G. L4 or L7 load balancer.** Tell → ALB (L7): HTTP, path/host routing, TLS termination. Tell → NLB (L4): raw TCP, static IPs, extreme throughput. Default: **ALB.**

**H. Same origin or CORS.** Tell → same origin (serve the frontend and API under one domain, e.g. CloudFront path routing): no CORS at all. Tell → CORS: they're genuinely separate origins. Default: **prefer same-origin if you can — it deletes a whole class of problem.**

---

# How to use and expand this document

- Suggested home: `docs/` in `practiq-infrastructure`, alongside the Docker primer.
- Related: **Docker primer** (§2.2 network namespaces, port publishing, Compose DNS), **data-access primer** (§1.5–1.7: the TCP session to Postgres, pooling), **Modern Java primer** (§15 `HttpClient`, §16 security).
- Good next deep-dives, any of which I can expand: **the Practiq VPC** drawn out concretely (subnets, route tables, security groups, the Terraform); **TLS termination end to end on ALB + ACM** with the actual config; **CORS configured in Micronaut** for the frontend/API split, with the preflight traced; **auth end to end** (session cookie vs OAuth2/OIDC vs JWT, with the flows drawn); **reading a `tcpdump`/Wireshark capture** of a TLS handshake, packet by packet; **HTTP caching strategy** for Practiq (CDN, ETags, what's cacheable given the review workflow).

*Caveat: protocol mechanics here (TCP, IP, TLS, HTTP, PKI) are stable and written from knowledge. The browser cookie landscape in §9.2 was verified July 2026 because it has changed materially and most published advice is stale. If you want a specific AWS behaviour or a current TLS cipher recommendation verified, point me at it.*
