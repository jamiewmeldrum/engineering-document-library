# Frontend for Backend Engineers — A Primer №70

*Enough of the frontend to design good APIs for it, reason about the boundary, review a pull request without bluffing, and not be mystified by the build step. Not a React tutorial — the **mental model** of how a modern web frontend works and where it meets your service. Practiq has a React frontend, so the examples use it.*

The framing that makes the frontend legible from a backend perspective: **a modern frontend is a distributed system where the client is a program you deploy to someone else's machine, over a network you don't control, running in a runtime you can't inspect.** Every awkward thing about it follows — you can't trust the client (№60), you can't assume the network, state exists in two places and must be reconciled, and the deployment target is thousands of slightly different browsers.

The second idea, which is the actual mechanism: **the frontend is a function from state to UI.** `UI = f(state)`. You don't write instructions to mutate the screen; you describe what the screen should look like for a given state, and the framework computes the changes. That's the same declarative bargain as SQL (№22 §1.1) or Terraform (№55 §2), and it's the single concept that makes React comprehensible.

Contents:

- **Part 1** — how the browser works
- **Part 2** — the three languages
- **Part 3** — rendering strategies
- **Part 4** — the React model
- **Part 5** — state management
- **Part 6** — the build toolchain
- **Part 7** — the API boundary
- **Part 8** — auth across the boundary
- **Part 9** — performance
- **Part 10** — testing and delivery
- **Part 11** — when to use what

## Translation index — backend concept to frontend equivalent

| You know | The frontend equivalent |
|---|---|
| A service returning a response | A component returning markup |
| Dependency injection | Props (passed down) / context (ambient) |
| Immutable value objects | State updates must produce new objects |
| Caching with TTL and invalidation | React Query / SWR data caching |
| Compilation | The bundler (Vite/webpack) |
| A JAR or container image | The bundle — JS/CSS/HTML assets |
| Server-side session | Cookie or token held by the browser |
| N+1 queries | Waterfall requests (each awaiting the last) |
| Connection pooling | HTTP/2 multiplexing over one connection |
| Blue/green deploy | New bundle at a new hashed filename |

---

# Part 1 — How the browser works

## 1.1 From URL to pixels

1. **DNS** resolves the hostname (№51 §3).
2. **TCP + TLS** handshake (№51 §4, §7).
3. **HTTP request** for the document; the server returns HTML.
4. **Parse HTML → DOM** — a tree of nodes representing the document.
5. **Parse CSS → CSSOM**, combine with the DOM into the **render tree**.
6. **Layout** (compute geometry) → **Paint** (fill pixels) → **Composite** (assemble layers, often on the GPU).
7. **JavaScript executes**, potentially mutating the DOM and triggering layout/paint again.

The costs worth knowing: **`<script>` in the head blocks parsing** unless marked `defer` or `async`; **CSS blocks rendering** because the browser won't paint unstyled content; and **layout is expensive**, so a JavaScript loop that reads and writes geometry alternately forces repeated synchronous layout ("layout thrashing").

## 1.2 The single thread

**JavaScript runs on one thread**, shared with layout and paint. Block it and the page freezes — no scrolling, no clicks, nothing. This is the fundamental performance constraint of the frontend, and the reason everything I/O-related is asynchronous.

The **event loop** is the mechanism: a call stack, a task queue, and a microtask queue. Async work (network, timers) is handed to the browser, and its callback is queued to run when the stack is empty. Promises resolve on the **microtask** queue, which drains before the next task — which is why a promise callback runs before a `setTimeout(…, 0)` scheduled earlier.

The nearest backend analogy is **virtual threads and non-blocking I/O** (№10 §12.2): concurrency without parallelism, achieved by never blocking the one thread you have. Genuinely CPU-heavy work goes to a **Web Worker** (a real separate thread with no DOM access).

## 1.3 Storage in the browser

| Mechanism | Size | Sent to server | Use for |
|---|---|---|---|
| **Cookies** | ~4 KB | **automatically, every request** | session identity (№51 §9) |
| **localStorage** | ~5–10 MB | no | preferences, non-sensitive cache |
| **sessionStorage** | ~5–10 MB | no | per-tab state |
| **IndexedDB** | large | no | offline data, big caches |

`localStorage` is **readable by any JavaScript on the page**, which is why storing an auth token there is an XSS liability (§8.2, №60 §6.3) — a cookie with `HttpOnly` isn't.

---

# Part 2 — The three languages

**HTML** — structure and semantics. Semantic elements (`<nav>`, `<main>`, `<button>`) matter for accessibility and SEO; a `<div>` with a click handler is not a button and won't work with a keyboard or screen reader.

**CSS** — presentation. Worth knowing conceptually: the **box model** (content, padding, border, margin), **flexbox** (one-dimensional layout) and **grid** (two-dimensional), **specificity** (which rule wins — the source of most "why isn't my style applying"), the **cascade**, **media queries** for responsive design, and **custom properties** (CSS variables). Modern projects often use a utility framework (Tailwind) or CSS modules to avoid global-namespace collisions.

**JavaScript / TypeScript** — behaviour. Modern JS (ES2015+) has classes, modules, arrow functions, destructuring, spread, `async/await`, and optional chaining. **TypeScript** adds static types and is the default for anything non-trivial — as a Java developer you'll find it familiar and immediately valuable, and it's the single biggest quality lever available in a frontend codebase.

The JavaScript quirks worth being warned about: `==` does type coercion (**always use `===`**), `this` binding is context-dependent, `null` and `undefined` are distinct, and floating-point arithmetic has the usual traps — plus everything is single-threaded (§1.2).

---

# Part 3 — Rendering strategies

**Where the HTML is produced** is the defining architectural decision:

| Strategy | HTML built | Pros | Cons |
|---|---|---|---|
| **SSR** | on the server, per request | fast first paint, SEO, works without JS | server load, server needed per request |
| **CSR / SPA** | in the browser, by JS | rich interactivity, cheap hosting, one API | slow first paint, JS-dependent, weaker SEO |
| **SSG** | at build time | fastest, cheapest (just files on a CDN) | only for content that isn't per-user |
| **ISR / hybrid** | mixed per route | best of both | more complexity |
| **Islands / RSC** | mostly server, JS only where needed | small bundles | newer, more concepts |

The industry has swung from server-rendered pages, to full SPAs, and back toward server-rendering with selective interactivity — the recognition being that shipping a megabyte of JavaScript to render a mostly-static page was a poor trade.

**For Practiq:** an SPA served as static files from S3/CloudFront talking to your API is a perfectly reasonable choice — cheap, simple, and it keeps a clean API boundary. SEO matters little for an authenticated learning tool; if public question pages ever needed to rank, that's when SSR becomes worth the complexity.

---

# Part 4 — The React model

## 4.1 Components

The UI is a tree of **components** — functions that take inputs and return a description of markup:

```jsx
function QuestionCard({ question, onApprove }) {
  return (
    <div className="card">
      <h3>{question.stem}</h3>
      <span>Difficulty: {question.difficulty}</span>
      <button onClick={() => onApprove(question.id)}>Approve</button>
    </div>
  );
}
```

**JSX** is syntax sugar compiled to function calls — it looks like HTML in JavaScript but it's JavaScript. Composition (§ №41 §1.1) is the organising principle here exactly as it is in backend design: small components combined into larger ones.

## 4.2 Props and state

**Props** flow *down* from parent to child and are read-only from the child's perspective — effectively constructor injection (№14 §2.3). **State** is a component's own data, and changing it triggers a re-render:

```jsx
function QuestionList() {
  const [questions, setQuestions] = useState([]);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    fetch('/api/v1/questions?conceptId=42')
      .then(r => r.json())
      .then(data => { setQuestions(data); setLoading(false); });
  }, []);                                    // [] = run once on mount

  if (loading) return <Spinner />;
  return <>{questions.map(q => <QuestionCard key={q.id} question={q} />)}</>;
}
```

**`UI = f(state)`** in practice: you never write "update that heading's text." You set state; React re-runs the function and works out the minimal DOM changes.

## 4.3 The virtual DOM and reconciliation

Touching the real DOM is slow. React renders to a lightweight in-memory tree, **diffs** it against the previous one, and applies only the differences. The `key` prop matters here — it tells React which list items correspond across renders, and getting it wrong (using array indices for a reorderable list) causes state to attach to the wrong element. Newer frameworks (Svelte, Solid) achieve the same end by compiling precise updates instead, avoiding the diff entirely.

## 4.4 Hooks

Functions that let a component use React features:

| Hook | For |
|---|---|
| `useState` | local state |
| `useEffect` | side effects — data fetching, subscriptions, cleanup |
| `useContext` | consume ambient values without prop-drilling |
| `useMemo` / `useCallback` | memoise expensive values / stable function identities |
| `useRef` | a mutable box that doesn't trigger re-renders; DOM access |
| `useReducer` | complex state transitions (Redux-shaped, locally) |

The rules: **call hooks unconditionally at the top level** (order identifies them across renders), and only from components or other hooks.

**`useEffect` is where most bugs live.** The dependency array controls when it re-runs: `[]` once on mount, `[id]` when `id` changes, omitted entirely means *every render* — which, if the effect sets state, is an infinite loop. Effects should also return a cleanup function to unsubscribe or abort in-flight requests.

## 4.5 Immutability

State updates must produce **new objects**, because React detects change by reference comparison:

```jsx
setQuestions([...questions, newQuestion]);          // new array — re-renders
questions.push(newQuestion); setQuestions(questions); // same reference — no re-render
```

Exactly the immutability discipline from №10 §11.3 and №41 §1.2, enforced by the framework.

---

# Part 5 — State management

## 5.1 The categories

The distinction that resolves most state-management confusion: **server state and client state are different problems.**

- **Server state** — data owned by your backend, fetched over the network. It's a *cache*, and it needs the things caches need: loading and error states, staleness, refetching, invalidation, deduplication (№31 §9).
- **Client state** — UI state that belongs to the browser: which tab is open, form input, a modal's visibility.

Treating server data as if it were local state is the classic error — you end up hand-writing cache invalidation badly.

## 5.2 The tools

**Local state** (`useState`) for anything one component owns. **Lift state up** to the nearest common parent when siblings must share. **Context** for genuinely ambient values (theme, current user) — but note it re-renders all consumers, so it's not a general state store. **Server-state libraries** (TanStack Query, SWR) for API data: caching, background refetch, deduplication, retries, optimistic updates — this is the right default for anything from your API. **Global client-state stores** (Zustand, Redux) only when genuinely global, complex client state exists.

> **The tell — state:** most applications need **far less** global state than they think. Server data belongs in a query library; the rest is usually local. Reach for a global store when you've felt the pain, not preemptively (№40 §6, YAGNI).

---

# Part 6 — The build toolchain

## 6.1 Why there's a build at all

Browsers don't run JSX or TypeScript, and shipping hundreds of separate module files would be slow. The build **transpiles** (TS/JSX → JS), **bundles** (many modules into few files), **tree-shakes** (drops unused code), **minifies**, **hashes filenames** for cache-busting, and processes CSS and assets.

It's the frontend's compiler, and the output — a `dist/` folder of static files — is its artifact, equivalent to your JAR (№56 §6).

## 6.2 The pieces

**Vite** is the current standard bundler/dev server (fast, native ES modules in dev); **webpack** is the older incumbent; **esbuild**/**SWC** are the fast Go/Rust transpilers underneath many toolchains. **npm/pnpm/yarn** manage dependencies, with a **lockfile** pinning exact versions (commit it — same reasoning as №55 §3.1). **ESLint** and **Prettier** handle linting and formatting. **Meta-frameworks** (Next.js, Remix) bundle routing, SSR and build configuration into an opinionated whole.

`node_modules` being enormous is normal and universal; it's build-time only and never ships.

## 6.3 What ships

Hashed static assets (`app.a1b2c3.js`), served from a CDN with long cache lifetimes — the hash changes when the content does, so cache invalidation is automatic (№31 §9.2, №51 §10.3). **Bundle size is a real budget**: every kilobyte is parse and execution time on a mid-range phone. Code-splitting by route means users download only what they need.

---

# Part 7 — The API boundary

The part that most directly concerns you, because **you design it**.

## 7.1 What frontends actually want

**Shapes that match screens, not tables.** A question list screen wants a question with its concept name inlined — not a question plus a separate call per concept. Under-fetching forces waterfalls; over-fetching wastes bandwidth. Projections (№20 §3.8) are the right tool: a `QuestionSummary` record shaped for the list view.

**Predictable, consistent structure** — the same envelope, the same error shape, the same date format (ISO-8601, UTC) everywhere. Inconsistency costs the frontend a special case per endpoint.

**Cursor pagination** rather than offset (stable under insertion, and faster at depth — №20 §3.8), with the cursor opaque so the client doesn't build assumptions on it.

**Useful errors** — a stable machine-readable code, a human-readable message, and per-field validation details:

```json
{ "code": "VALIDATION_FAILED", "message": "Question could not be saved",
  "errors": [{ "field": "difficulty", "message": "must be between 1 and 5" }] }
```

The frontend needs to highlight the right field; a 400 with a bare string forces string-matching.

**Correct status codes** (№51 §5.2) — clients branch on them. 401 means "re-authenticate", 403 means "don't retry", 409 means "conflict, reload" (your `@Version` collision, №21 §2.7), 422 means "fix the input", 429 means "back off".

## 7.2 The waterfall problem

```
GET /questions          → 200ms
  then for each: GET /concepts/{id}   → 20 × 150ms sequentially
```

That's an N+1 at the network layer — the same pathology as №20 §2.8, but each round trip costs 100ms rather than 1ms, so it's far more damaging. **Fix it on the server** by returning what the screen needs, or provide a batch endpoint. This is exactly why "design the API for the caller's use case, not your data model" (№41 §4.4) matters.

## 7.3 REST, GraphQL, or generated clients

**REST** is the sensible default: simple, cacheable, universally understood. **GraphQL** lets clients request precisely the shape they need — genuinely solves over- and under-fetching for complex, varied UIs, at the cost of server complexity, caching difficulty and query-cost control. **Generating a typed client from OpenAPI** is the highest-value-per-effort improvement available at this boundary: the frontend gets compile-time type safety against your actual contract, and a breaking change fails the build rather than production.

---

# Part 8 — Auth across the boundary

## 8.1 The flow

For a browser SPA the recommended shape is **OIDC Authorisation Code with PKCE** (№60 §5.2), with the resulting session held in a cookie set by your backend.

## 8.2 Where to keep the token — the decision that matters

| Storage | XSS-safe | CSRF-safe | Verdict |
|---|---|---|---|
| **`HttpOnly` cookie** | **yes** — JS can't read it | needs `SameSite` / CSRF token | **preferred** |
| `localStorage` | **no** — any script can read it | yes (not sent automatically) | common, weaker |
| In-memory only | yes | yes | safest, lost on refresh |

The frequent argument is "cookies are vulnerable to CSRF, so use `localStorage`" — but **`SameSite=Lax` largely solves CSRF, while nothing solves XSS-reading-`localStorage`**. Prefer an `HttpOnly; Secure; SameSite=Lax` cookie (№51 §9.2, №60 §3.3).

## 8.3 The rest of the boundary

**CORS** applies the moment the frontend and API are different origins — and it's fixed by *your server's headers*, not by the frontend (№51 §9.5). Serving both from one origin via CloudFront path routing avoids it entirely, which is worth doing if you can.

**Never trust the client** (№60 §1.1). Hiding an admin button is UX; the authorisation check must be server-side on every request. Client-side validation is for fast feedback; the server validates again, always.

---

# Part 9 — Performance

## 9.1 Core Web Vitals

Google's user-centred metrics, and the ones people actually optimise against:

- **LCP** (Largest Contentful Paint) — when the main content appears. Target under 2.5s.
- **INP** (Interaction to Next Paint) — responsiveness to input. Target under 200ms.
- **CLS** (Cumulative Layout Shift) — visual stability. Target under 0.1.

## 9.2 What actually helps

**Ship less JavaScript** — the dominant factor. Code-split by route, lazy-load below-the-fold components, audit dependencies (a date library can be larger than your application code).

**Serve from a CDN** with long cache lifetimes on hashed assets (§6.3).

**Optimise images** — usually the largest bytes on a page. Modern formats, correct sizing, lazy loading, and explicit dimensions to prevent layout shift.

**Fix waterfalls** (§7.2) — often the biggest single win, and it's a backend fix.

**Backend latency is frontend performance.** A p99 of 800ms on your API is 800ms the user waits regardless of how good the frontend is. The frontend and backend share one perceived-performance budget (№57 §3.2).

---

# Part 10 — Testing and delivery

## 10.1 Testing

The same pyramid, different tools (№44 §2): **unit** (Vitest/Jest for pure logic), **component** (React Testing Library — render a component, interact as a user would, assert on what's visible), **integration** (with the API mocked at the network layer via MSW), and **E2E** (Playwright/Cypress driving a real browser — few, critical journeys only).

The guiding principle from React Testing Library is worth borrowing generally: **test what the user experiences, not implementation details.** Query by visible text and accessible role rather than CSS classes — which is the same "behaviour not implementation" rule as №44 §3.2, and it has the pleasant side effect of nudging you toward accessible markup.

## 10.2 Delivery

The pipeline mirrors the backend (№56): install → lint → typecheck → test → build → deploy static assets to S3/CloudFront → invalidate the CDN. Because the output is static files, deployment is a sync and a cache invalidation, and **rollback is redeploying the previous build** — cheap and fast.

Two frontend-specific concerns: **environment configuration must be baked at build time** (the browser has no server-side env), so either build per environment or fetch configuration at runtime from an endpoint; and **users may be running an old bundle** for as long as their tab is open, so the API must tolerate slightly stale clients — the same backward-compatibility discipline as rolling deploys (№56 §10).

## 10.3 Accessibility, briefly

Not optional, and mostly cheap if you start early: semantic HTML, keyboard operability for everything, visible focus states, `alt` text, sufficient contrast, labelled form fields, and ARIA only where semantics genuinely fall short. Automated checks (axe, Lighthouse) catch a useful fraction; keyboard-only navigation catches most of the rest.

---

# Part 11 — When to use what

**A. SPA, SSR, or static?** Tell → SPA: an authenticated application behind a login (Practiq). Tell → SSR: SEO or fast first paint on content pages matters. Tell → SSG: content that isn't per-user. Default: **SPA on a CDN for an app; add SSR only when SEO or first paint demands it.**

**B. REST or GraphQL?** Tell → REST: straightforward resources, caching matters, small team. Tell → GraphQL: many clients with divergent data needs, deep nesting, over-fetching is a measured problem. Default: **REST, with well-shaped projections per screen.**

**C. Where does state live?** Tell → server-state library: it came from your API. Tell → local `useState`: one component owns it. Tell → context: genuinely ambient. Tell → global store: complex client state shared widely. Default: **server library for API data, local for the rest.**

**D. Token in a cookie or localStorage?** Tell → cookie (`HttpOnly; Secure; SameSite=Lax`): almost always. Tell → memory: highest security, accepting re-auth on refresh. Default: **cookie** (§8.2).

**E. TypeScript or JavaScript?** Tell → TypeScript: anything beyond a trivial script. Default: **TypeScript** — the strongest single quality lever, and immediately familiar coming from Java.

**F. Component library or bespoke CSS?** Tell → library (MUI, shadcn/ui): you want speed and consistency and aren't a designer. Tell → bespoke: distinctive design is a product requirement. Default (a backend engineer's portfolio project): **a component library** — it'll look better than hand-rolled CSS and take a fraction of the time.

**G. Fix it in the frontend or the API?** Tell → API: waterfalls, wrong data shapes, missing fields, chatty endpoints. Tell → frontend: rendering, interaction, presentation. Default: **if the frontend is making many calls to assemble one screen, that's an API design problem** (§7.2).

---

# How to expand this

- *Related:* №51 Parts 5, 9 (HTTP, cookies, CORS, the browser security model), №60 §6 (XSS and output encoding), №43 (designing the system this is a client of), №41 §4.4 (API design for the caller), №56 (the delivery pipeline).
- *Candidates for deeper treatment:* **React in depth** — rendering behaviour, memoisation, Server Components; **the Practiq API contract** designed screen-by-screen with generated TypeScript types; **web performance** as a discipline with real measurement; **accessibility** properly.

*Concepts are stable; the frontend ecosystem moves faster than any other area in this library. Framework specifics — React APIs, the current best bundler, meta-framework capabilities — change on a scale of months, so verify tooling choices against current sources rather than trusting this document's specifics.*
