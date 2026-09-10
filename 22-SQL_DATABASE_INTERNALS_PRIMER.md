# SQL Mastery & Database Internals — A Primer №22

*Two halves that reinforce each other: **SQL fluency beyond what an ORM generates** — joins, window functions, CTEs, set operations — and **what the database is actually doing** underneath: storage, indexes, MVCC, the planner, isolation. Postgres-centred. Where №20 teaches the Java-to-database stack and the ORM, this teaches the database itself.*

The case for learning this properly, given you have an ORM: **an ORM writes your simple queries so you can write the hard ones.** Every non-trivial application eventually needs a report, an analytical query, a bulk operation or a data migration where JPQL either can't express it or expresses it badly — and at that point SQL fluency is the difference between an elegant single query and a slow loop in Java. It's also the layer where performance problems actually live: the query plan, not the Java.

The organising idea for the internals half: **a database is a program that maintains an ordered, durable, concurrently-accessible structure on disk, and almost every behaviour you find surprising is a consequence of that.** MVCC, index types, vacuum, isolation anomalies, why an index sometimes isn't used — all of it follows from the constraints of storing data on a device that's a million times slower than RAM while many clients read and write simultaneously.

Contents:

- **Part 1** — the relational foundation
- **Part 2** — joins properly
- **Part 3** — aggregation and grouping
- **Part 4** — window functions
- **Part 5** — CTEs and recursion
- **Part 6** — subqueries and set operations
- **Part 7** — writing data
- **Part 8** — storage internals
- **Part 9** — indexes internally
- **Part 10** — MVCC and transactions internally
- **Part 11** — the query planner
- **Part 12** — when to use what

## Query index

| I need to… | Use | §|
|---|---|---|
| Combine rows from two tables | `JOIN` | §2 |
| Keep rows with no match | `LEFT JOIN` | §2.2 |
| Count/sum per group | `GROUP BY` + aggregate | §3 |
| Filter *after* aggregating | `HAVING` | §3.2 |
| Rank rows within a group | `ROW_NUMBER() OVER (PARTITION BY …)` | §4.2 |
| Running total / moving average | window function with a frame | §4.4 |
| Compare a row to the previous one | `LAG` / `LEAD` | §4.3 |
| Name a subquery for readability | `WITH` (CTE) | §5.1 |
| Walk a hierarchy | `WITH RECURSIVE` | §5.2 |
| Parents that have/lack children | `EXISTS` / `NOT EXISTS` | §6.1 |
| Insert-or-update | `INSERT … ON CONFLICT` | §7.2 |
| Update from another table | `UPDATE … FROM` | §7.3 |
| Return what was written | `RETURNING` | §7.4 |
| Find out why it's slow | `EXPLAIN (ANALYZE, BUFFERS)` | §11.2 |

---

# Part 1 — The relational foundation

## 1.1 The model

A **relation** (table) is a set of **tuples** (rows) with named, typed **attributes** (columns). SQL is a mostly-declarative language over that model: you state *what* you want, and the planner decides *how* (§11). This is the same declarative bargain as Terraform (№55 §2) — you give up control of the mechanism in exchange for the system optimising it for you.

**Relational algebra** underpins it: selection (`WHERE`), projection (`SELECT` columns), join, union, intersection, difference. Every SQL query reduces to compositions of these, which is why the planner can rewrite your query into an equivalent, faster form.

## 1.2 Logical evaluation order

The order SQL is *written* is not the order it's *evaluated*, and knowing the real order explains several rules that otherwise seem arbitrary:

```
FROM / JOIN  →  WHERE  →  GROUP BY  →  HAVING  →  SELECT  →  DISTINCT  →  ORDER BY  →  LIMIT
```

Consequences: you **cannot use a `SELECT` alias in `WHERE`** (the alias doesn't exist yet) but you **can in `ORDER BY`** (it does by then); `WHERE` filters rows before grouping while `HAVING` filters groups after; and `LIMIT` applies last, so it doesn't reduce the work of the aggregation above it.

## 1.3 NULL — three-valued logic

`NULL` means *unknown*, not zero and not empty string. Comparisons with it yield `NULL`, not true or false:

```sql
NULL = NULL        -- NULL (not true!)
NULL <> 5          -- NULL
x IS NULL          -- the only correct test
COALESCE(x, 0)     -- substitute a default
```

This causes real bugs. `WHERE status <> 'APPROVED'` **excludes rows where status is NULL**, because `NULL <> 'APPROVED'` is unknown, not true. And `NOT IN` with a NULL in the subquery returns **no rows at all** (№20 §5.4) — the single nastiest NULL trap. Aggregates ignore NULLs (`COUNT(col)` skips them; `COUNT(*)` doesn't), and `UNIQUE` constraints permit multiple NULLs, since two unknowns aren't provably equal.

---

# Part 2 — Joins

## 2.1 The types

```sql
-- INNER: only rows matching on both sides
SELECT q.id, c.name
FROM question q
JOIN concept c ON c.id = q.concept_id;

-- LEFT: all rows from the left, NULLs where the right has no match
SELECT c.name, q.id
FROM concept c
LEFT JOIN question q ON q.concept_id = c.id;

-- FULL OUTER: everything from both, NULLs where unmatched
-- CROSS: cartesian product — every combination (rarely intentional)
-- SELF: a table joined to itself, via aliases
```

## 2.2 The LEFT JOIN trap

```sql
-- BROKEN: the WHERE turns this into an inner join
SELECT c.name, COUNT(q.id)
FROM concept c
LEFT JOIN question q ON q.concept_id = c.id
WHERE q.status = 'APPROVED'          -- ← eliminates the NULL rows the LEFT JOIN produced
GROUP BY c.name;

-- CORRECT: the condition belongs in the JOIN
SELECT c.name, COUNT(q.id)
FROM concept c
LEFT JOIN question q ON q.concept_id = c.id AND q.status = 'APPROVED'
GROUP BY c.name;
```

**A condition on the right-hand table in `WHERE` filters out the unmatched rows**, defeating the outer join. Conditions on the outer side go in `ON`; conditions on the preserved side go in `WHERE`. This is one of the most common SQL bugs and it fails silently — the query works, it's just wrong.

## 2.3 Join algorithms

The planner picks among three, and reading a plan means knowing them:

| Algorithm | How | Good when |
|---|---|---|
| **Nested Loop** | for each outer row, look up matches | one side small, indexed inner side |
| **Hash Join** | build a hash of one side, probe with the other | large, unsorted, equality joins |
| **Merge Join** | sort both, walk them together | both already sorted (e.g. by index) |

A **Nested Loop over two large unindexed tables** is the classic disaster — O(n×m). Seeing one in a plan on big inputs usually means a missing index.

## 2.4 Row multiplication

A join to a one-to-many **multiplies parent rows** — one question with three concepts yields three rows. This is the mechanism behind duplicated results, wrong `COUNT`s, and the ORM's `MultipleBagFetchException` (№21 §11.5). If you need existence rather than the child data, use `EXISTS` (§6.1); if you need the parent once with aggregated children, aggregate.

---

# Part 3 — Aggregation

## 3.1 The basics

```sql
SELECT c.name,
       COUNT(*)                    AS total,
       COUNT(*) FILTER (WHERE q.status = 'APPROVED') AS approved,   -- Postgres FILTER
       AVG(q.difficulty)::numeric(3,1) AS avg_difficulty,
       MAX(q.created_at)           AS latest
FROM concept c
JOIN question q ON q.concept_id = c.id
GROUP BY c.id, c.name                    -- group by the PK, select other columns freely
HAVING COUNT(*) > 5
ORDER BY approved DESC;
```

**`FILTER`** is a Postgres feature worth knowing — conditional aggregation without the older `SUM(CASE WHEN … THEN 1 ELSE 0 END)` contortion, and clearer.

Grouping by the primary key lets you select other columns from that table without listing them (functional dependency) — Postgres allows this; some engines don't.

## 3.2 WHERE vs HAVING

`WHERE` filters **rows before grouping**; `HAVING` filters **groups after**. Filter as early as possible: a condition that can go in `WHERE` should, because it reduces the rows the aggregation has to process.

`COUNT(*)` counts rows; `COUNT(col)` counts non-NULL values; `COUNT(DISTINCT col)` counts distinct non-NULLs and is markedly more expensive.

---

# Part 4 — Window functions

The most valuable SQL feature most developers never learn, and the one that eliminates whole categories of application-side loops.

## 4.1 The idea

An aggregate collapses rows into one. A **window function computes across a set of rows while keeping every row**:

```sql
SELECT id, concept_id, difficulty,
       AVG(difficulty) OVER (PARTITION BY concept_id) AS concept_avg,
       difficulty - AVG(difficulty) OVER (PARTITION BY concept_id) AS diff_from_avg
FROM question;
```

Every question row survives, each annotated with its concept's average. Doing that without windows means either a self-join to a grouped subquery or a loop in Java.

`OVER (PARTITION BY … ORDER BY …)` defines the window: `PARTITION BY` splits into groups, `ORDER BY` orders within them.

## 4.2 Ranking

```sql
SELECT id, concept_id, difficulty,
       ROW_NUMBER() OVER (PARTITION BY concept_id ORDER BY difficulty DESC) AS rn,
       RANK()       OVER (PARTITION BY concept_id ORDER BY difficulty DESC) AS rnk,
       DENSE_RANK() OVER (PARTITION BY concept_id ORDER BY difficulty DESC) AS dense
FROM question;
```

`ROW_NUMBER` is always distinct (1,2,3,4); `RANK` gives ties the same value and skips (1,1,3); `DENSE_RANK` ties without skipping (1,1,2).

**The top-N-per-group pattern** — a genuinely common requirement that's awkward without windows:

```sql
SELECT * FROM (
    SELECT q.*, ROW_NUMBER() OVER (PARTITION BY concept_id ORDER BY difficulty DESC) AS rn
    FROM question q WHERE status = 'APPROVED'
) ranked
WHERE rn <= 3;                        -- the 3 hardest approved questions per concept
```

(Postgres also offers `DISTINCT ON` as a shorter route to top-1-per-group.)

## 4.3 LAG and LEAD

Access other rows relative to the current one — for deltas, gaps, and change detection:

```sql
SELECT attempted_at, correct,
       LAG(correct) OVER (PARTITION BY question_id ORDER BY attempted_at) AS previous,
       attempted_at - LAG(attempted_at) OVER (PARTITION BY question_id ORDER BY attempted_at) AS gap
FROM attempt;
```

## 4.4 Frames — running totals and moving averages

`ORDER BY` inside a window implies a default frame of "everything up to the current row", which is what makes running totals work:

```sql
SELECT attempted_at,
       COUNT(*) OVER (ORDER BY attempted_at) AS running_total,
       AVG(score) OVER (ORDER BY attempted_at ROWS BETWEEN 6 PRECEDING AND CURRENT ROW) AS moving_avg_7
FROM attempt;
```

Frame clauses: `ROWS` counts physical rows; `RANGE` uses value ranges (ties included together) — a distinction that matters when the ordering column has duplicates.

---

# Part 5 — CTEs and recursion

## 5.1 Common Table Expressions

`WITH` names a subquery, making complex SQL readable top-to-bottom instead of nested inside-out:

```sql
WITH approved AS (
    SELECT * FROM question WHERE status = 'APPROVED'
),
per_concept AS (
    SELECT concept_id, COUNT(*) AS n, AVG(difficulty) AS avg_diff
    FROM approved GROUP BY concept_id
)
SELECT c.name, p.n, p.avg_diff
FROM per_concept p
JOIN concept c ON c.id = p.concept_id
WHERE p.n >= 10
ORDER BY p.avg_diff DESC;
```

Readability is the main benefit, and it's a large one — a 60-line query built from named steps is comprehensible in a way a triple-nested subquery is not. A note on performance: modern Postgres (12+) **inlines** CTEs into the surrounding query by default, so they no longer act as optimisation fences; add `MATERIALIZED` if you deliberately want the old behaviour (computing once and reusing).

## 5.2 Recursive CTEs

For hierarchies and graphs — a syllabus tree, an org chart, a dependency graph:

```sql
WITH RECURSIVE tree AS (
    SELECT id, parent_id, name, 1 AS depth              -- anchor: the roots
    FROM spec_section WHERE parent_id IS NULL
  UNION ALL
    SELECT s.id, s.parent_id, s.name, t.depth + 1       -- recursive: children of the previous level
    FROM spec_section s
    JOIN tree t ON s.parent_id = t.id
)
SELECT * FROM tree ORDER BY depth, name;
```

The shape is always: an **anchor** query, `UNION ALL`, and a **recursive** query referencing the CTE by name. Guard against cycles with a depth limit or by tracking visited nodes — a cycle in the data produces an infinite loop.

---

# Part 6 — Subqueries and set operations

## 6.1 EXISTS, IN, and the semi-join

```sql
-- concepts that HAVE at least one approved question — a SEMI-JOIN
SELECT * FROM concept c
WHERE EXISTS (SELECT 1 FROM question q
              WHERE q.concept_id = c.id AND q.status = 'APPROVED');

-- concepts with NO approved questions — an ANTI-JOIN
SELECT * FROM concept c
WHERE NOT EXISTS (SELECT 1 FROM question q
                  WHERE q.concept_id = c.id AND q.status = 'APPROVED');
```

`EXISTS` returns each parent **once** regardless of how many children match — unlike a join, which multiplies (§2.4). Use it whenever you need existence and not the child columns.

**Prefer `NOT EXISTS` over `NOT IN`**: if the subquery yields a single NULL, `NOT IN` returns nothing at all, silently (§1.3). This one has cost people real production incidents.

Correlated subqueries (referencing the outer row) are conceptually per-row but are usually rewritten by the planner into a proper semi-/anti-join, so write for clarity and check the plan.

## 6.2 Lateral joins

`LATERAL` lets a subquery reference columns from earlier in the `FROM` — a per-row subquery with the power of a join. The idiomatic top-N-per-group alternative:

```sql
SELECT c.name, q.id, q.difficulty
FROM concept c
CROSS JOIN LATERAL (
    SELECT id, difficulty FROM question
    WHERE concept_id = c.id AND status = 'APPROVED'
    ORDER BY difficulty DESC LIMIT 3
) q;
```

## 6.3 Set operations

`UNION` (combines, **removes duplicates** — a sort or hash, so it costs), `UNION ALL` (combines, keeps duplicates — **cheaper, use it when you know there are none**), `INTERSECT`, `EXCEPT`. Column counts and types must align.

---

# Part 7 — Writing data

## 7.1 The basics

```sql
INSERT INTO question (concept_id, status, difficulty) VALUES (42, 'DRAFT', 3);
INSERT INTO question (concept_id, status) SELECT id, 'DRAFT' FROM concept WHERE …;  -- bulk
UPDATE question SET status = 'APPROVED' WHERE id = 5;
DELETE FROM question WHERE created_at < now() - interval '1 year';
```

## 7.2 Upsert

```sql
INSERT INTO concept (slug, name)
VALUES ('newtons-second-law', 'Newton''s Second Law')
ON CONFLICT (slug) DO UPDATE
    SET name = EXCLUDED.name, updated_at = now();
-- or: ON CONFLICT (slug) DO NOTHING;
```

Atomic insert-or-update, resolved by the database. This is the SQL-level tool for **idempotent writes** (№31 §8.1) — a retry that hits a conflict updates rather than failing, which is precisely what you want from an at-least-once pipeline.

## 7.3 UPDATE … FROM and bulk operations

```sql
UPDATE question q
SET concept_id = m.new_concept_id
FROM concept_migration m
WHERE q.concept_id = m.old_concept_id;
```

A single set-based statement instead of a loop. **This is the key mental shift when leaving the ORM**: SQL operates on *sets*, and a set-based statement that touches a million rows is one round trip and one plan, whereas the equivalent loop in Java is a million round trips (№00 §1.2). Practiq's staging-table CTE seed pipeline is exactly this idea.

## 7.4 RETURNING

```sql
INSERT INTO question (concept_id, status) VALUES (42, 'DRAFT') RETURNING id, created_at;
UPDATE question SET status = 'APPROVED' WHERE id = 5 RETURNING *;
DELETE FROM attempt WHERE attempted_at < now() - interval '90 days' RETURNING id;
```

A Postgres feature that saves a round trip and closes a race — you get the generated ID or the affected rows back from the write itself.

## 7.5 Data-modifying CTEs

```sql
WITH archived AS (
    DELETE FROM attempt WHERE attempted_at < now() - interval '1 year' RETURNING *
)
INSERT INTO attempt_archive SELECT * FROM archived;
```

Move rows between tables atomically in one statement — a genuinely elegant Postgres capability.

---

# Part 8 — Storage internals

## 8.1 Pages and the heap

Postgres stores table data in **8 KB pages** in a **heap** — an unordered collection (№20 §4.4). All I/O happens in whole pages, which is why row size and page density matter: fewer rows per page means more pages read for the same query.

A row's physical address is its **`ctid`** — `(page, offset)`. Every index entry points at a ctid, which is why index maintenance is needed when rows move.

## 8.2 TOAST

Values too large for a page (>~2 KB) are compressed and, if still too big, moved to a separate **TOAST** table, with a pointer left behind. This is transparent, and it's why a table with large `text` columns can look small in `pg_relation_size` — the data is in the TOAST relation. It also means selecting a large column is a second fetch, which is one more argument for projections (№20 §3.8).

## 8.3 The write path and the WAL

The mechanism that delivers durability without an fsync per table write:

1. A change is written to the **WAL** (write-ahead log) — sequential, fast.
2. WAL is **fsynced** at commit. **At this point the transaction is durable.**
3. The data pages themselves are modified in the **shared buffer** cache (memory).
4. Dirty pages are written to disk later, at a **checkpoint**.

Crash recovery replays the WAL from the last checkpoint. The insight worth carrying: **commits are fast because they only require a sequential write to the log**, not scattered random writes to data files. The WAL is also what powers replication (streaming it to standbys), point-in-time recovery, and logical replication/CDC.

## 8.4 MVCC's storage cost, vacuum and bloat

Postgres implements MVCC (§10) by **never updating a row in place**: an `UPDATE` writes a *new row version* and marks the old one dead. Consequences that surprise people:

- **Updates create garbage.** A table updated heavily accumulates dead tuples.
- **`VACUUM`** reclaims dead tuples for reuse; **autovacuum** does it in the background. If it can't keep up, the table **bloats** — physically large with little live data, so scans read more pages and everything slows.
- **`VACUUM FULL`** rewrites the table compactly but takes an **exclusive lock** — not for production tables during business hours.
- **Long-running transactions block vacuum**, because their snapshot may still need those old versions. An idle-in-transaction connection left open for hours is a genuine operational hazard (№20 §1.6).
- **`ANALYZE`** updates the statistics the planner relies on (§11.3).

---

# Part 9 — Indexes internally

## 9.1 B-tree

The default, and the right answer nearly always. A balanced tree of pages: the root and internal pages hold separator keys, leaf pages hold keys plus ctids, and leaves are linked for range scans. Depth is typically 3–4 even for hundreds of millions of rows, so a lookup is a handful of page reads.

It supports `=`, `<`, `>`, `BETWEEN`, `IN`, `IS NULL`, prefix `LIKE 'foo%'`, and — because leaves are in key order — it can satisfy `ORDER BY` without a sort.

**Composite index column order** follows directly from the structure: entries are sorted by the first column, then the second within it, so an index on `(concept_id, status)` serves `concept_id` alone or both together, but **not `status` alone** (the leftmost-prefix rule, №20 §4.3).

## 9.2 The scan types

| Scan | What happens |
|---|---|
| **Seq Scan** | read every page. Correct for small tables or when returning most rows |
| **Index Scan** | walk the index, then fetch each matching row from the heap |
| **Index Only Scan** | answer entirely from the index — **no heap access**, the fast path |
| **Bitmap Heap Scan** | build a bitmap of matching pages, then read them in physical order |

**Bitmap scans** are the middle ground: when a query matches too many rows for random heap fetches to be efficient but too few for a full scan, Postgres collects the ctids, sorts them by page, and reads sequentially. Seeing one in a plan is normal and usually fine.

**Index Only Scans** require the **visibility map** to show the page as all-visible — which is maintained by vacuum. So a heavily-updated table may not get index-only scans until it's vacuumed, which is a genuinely non-obvious performance link.

## 9.3 The other index types

**GIN** — inverted index for composite values: JSONB containment, arrays, full-text search. Large and slower to update, extremely fast to search. **GiST** — extensible, for ranges, geometry and nearest-neighbour. **BRIN** — tiny, stores min/max per block range; brilliant for huge naturally-ordered tables (append-only, time-series) at a fraction of a B-tree's size. **Hash** — equality only; B-tree is nearly always better.

**Partial** (`WHERE status = 'APPROVED'`) and **covering** (`INCLUDE (…)`, enabling index-only scans) are modifiers worth reaching for — both covered with the decision framing in №20 §4.3.

## 9.4 Why an index isn't used

A recurring frustration with a short list of causes: the table is **small** (a scan is genuinely cheaper); the query returns **too many rows** (same); **stale statistics** mislead the planner (§11.3); a **function or cast on the column** (`WHERE lower(email) = …` needs an expression index on `lower(email)`); a **type mismatch** forcing a cast; **leading wildcards** (`LIKE '%foo'` can't use a B-tree — use a trigram index); or the **leftmost-prefix rule** on a composite index.

---

# Part 10 — MVCC and transactions internally

## 10.1 How MVCC works

Every row version carries **`xmin`** (the transaction that created it) and **`xmax`** (the transaction that deleted or superseded it). Each transaction takes a **snapshot** — the set of transactions considered committed at its start — and sees only versions whose `xmin` is visible and whose `xmax` is not.

The consequence, and the reason Postgres is pleasant under mixed load: **readers never block writers, and writers never block readers.** A reader sees an older version rather than waiting for a lock.

The costs are the ones in §8.4: dead versions accumulating, vacuum required, and long transactions holding back cleanup.

## 10.2 Isolation levels, implemented

| Level | Snapshot taken | Prevents |
|---|---|---|
| **Read Committed** *(default)* | **per statement** | dirty reads |
| **Repeatable Read** | **once, at first statement** | + non-repeatable reads, phantoms |
| **Serializable** | as RR + SSI conflict detection | all anomalies |

Read Committed taking a **new snapshot per statement** is why two identical queries in one transaction can return different results — the source of non-repeatable reads. Repeatable Read fixes that by holding one snapshot for the whole transaction.

**Serializable** uses SSI (serializable snapshot isolation) — it doesn't lock; it monitors for dependency patterns that could produce an anomaly and **aborts one transaction** with a serialization failure. So using it obliges you to **implement retry logic**, which is the practical cost.

## 10.3 Locking

**Row locks** are taken by writes and by `SELECT … FOR UPDATE` (pessimistic locking, №20 §2.7). **Table locks** range from `ACCESS SHARE` (a plain read) to `ACCESS EXCLUSIVE` (`VACUUM FULL`, most `ALTER TABLE`, which blocks everything). **Deadlocks** are detected automatically and one transaction is aborted — the same Coffman conditions as in-process deadlock (№12 §2.4), and the same fix: consistent lock ordering.

The migration lesson worth carrying: an `ALTER TABLE` that takes `ACCESS EXCLUSIVE` on a busy table blocks every query for its duration. Postgres has made many operations non-blocking (adding a nullable column with a default is now cheap), but **adding an index must use `CREATE INDEX CONCURRENTLY`** in production, or you lock out writes for the duration. This is why migrations need testing against realistic data volumes (№56 §10).

---

# Part 11 — The query planner

## 11.1 How it decides

The planner is **cost-based**: it enumerates plans (join orders, join algorithms, access methods), estimates each one's cost from **table statistics**, and picks the cheapest. Cost is an abstract number combining estimated page reads and CPU work, weighted by tunable parameters (`random_page_cost` versus `seq_page_cost` — the default assumes spinning disks and is often worth lowering on SSDs).

## 11.2 Reading a plan

```sql
EXPLAIN (ANALYZE, BUFFERS) SELECT …;
```

`EXPLAIN` alone shows the plan and estimates; **`ANALYZE` actually runs it** and shows real times and row counts; **`BUFFERS`** shows cache hits versus disk reads. Read plans **inside-out, bottom-up** — the innermost nodes execute first.

The three things to look at, in order:

1. **Estimated vs actual rows.** A large divergence (`rows=10 … actual rows=45000`) means the planner is working from bad information and every decision above it is suspect. Fix the statistics (§11.3), not the query.
2. **The most expensive node.** Where the time actually goes — often a Seq Scan on a large table, or a Nested Loop with a big inner side.
3. **Buffers.** Heavy `read` versus `hit` means you're going to disk; the data isn't cached and the query is I/O-bound.

Note `ANALYZE` executes the statement — wrap a write in a transaction you roll back.

## 11.3 Statistics

The planner relies on statistics gathered by `ANALYZE`: row counts, most-common values, histograms, distinct counts and correlation. **Stale statistics are the most common cause of a bad plan** — after a bulk load, a large delete, or a schema change, run `ANALYZE` before drawing conclusions. Autovacuum normally handles it; a table that's just been rewritten may not have caught up.

For correlated columns the planner assumes independence and can badly underestimate; **extended statistics** (`CREATE STATISTICS`) teach it about dependencies between columns.

## 11.4 Making a query faster

The ordered playbook: **add the right index** (predicate, join and sort columns, right composite order); **select fewer columns** (enables index-only scans, moves less data); **filter earlier** (`WHERE` before `HAVING`, push conditions down); **rewrite** (`EXISTS` instead of a multiplying join, `UNION ALL` instead of `UNION`); **fix the statistics**; **reduce round trips** (one set-based statement instead of N queries — usually the biggest win in an application, §7.3, №20 §2.8); and only then consider **denormalisation** or **materialised views**, measured.

---

# Part 12 — When to use what

**A. SQL or the ORM?** Tell → ORM: transactional domain operations, CRUD, anything object-shaped. Tell → SQL: reports, analytics, bulk operations, migrations, or anything JPQL expresses badly. Default: **ORM for the domain, SQL for sets** (№20 §7).

**B. Join or EXISTS?** Tell → join: you need columns from the child. Tell → `EXISTS`: you only need to know whether children exist. Default: **`EXISTS` for existence — it doesn't multiply rows.**

**C. Window function or application code?** Tell → window: ranking, running totals, comparing to neighbours, top-N-per-group. Tell → application: the logic is genuinely business logic. Default: **window function — one query beats fetching everything and looping.**

**D. CTE or subquery?** Tell → CTE: readability, reuse within the query, recursion. Tell → subquery: short and inline. Default: **CTE for anything non-trivial; it costs nothing now that they're inlined.**

**E. `UNION` or `UNION ALL`?** Tell → `UNION`: you genuinely need duplicates removed. Tell → `UNION ALL`: you know there are none. Default: **`UNION ALL`** — it skips the dedup sort.

**F. Index or not?** Tell → index: the column appears in `WHERE`, `JOIN` or `ORDER BY` on a table big enough to matter — and **FK columns, which Postgres doesn't index for you**. Tell → don't: small tables, write-heavy columns you never filter on. Default: **index your actual query predicates, and verify with `EXPLAIN` at realistic volume** (№20 §4.5).

**G. Which isolation level?** Tell → Read Committed: almost always. Tell → Repeatable Read: a multi-statement read that must see a consistent snapshot. Tell → Serializable: a genuine invariant across transactions — **and then implement retries**. Default: **Read Committed plus `@Version` optimistic locking** (№20 §1.7).

**H. Fix the query or the schema?** Tell → query: a plan problem, a missing index, a bad rewrite. Tell → schema: the same slow pattern recurs everywhere, or the model fights every query. Default: **query first — schema changes are expensive and rarely the actual problem.**

---

# How to expand this

- *Related:* №20 (the Java stack over this — JDBC, JPA, transactions, and the indexing/EXPLAIN material at a practical level), №21 (the ORM's annotation surface), №31 §4–5 (replication, consistency, sharding), №57 (monitoring query performance in production), №30 §2 (B-trees as a data structure).
- *Candidates for deeper treatment:* **JSONB in Postgres** (operators, GIN indexing, when it beats a table); **full-text search** (`tsvector`, ranking, versus reaching for Elasticsearch); **partitioning** (declarative partitioning, pruning, when it's worth it); **replication and PITR** operationally; **a query-tuning walkthrough** with real plans before and after.

*Postgres-specific but broadly transferable. The SQL standard features are stable; Postgres implementation details (planner behaviour, index capabilities, what's non-blocking in DDL) improve every release — check the docs for the version you run.*
