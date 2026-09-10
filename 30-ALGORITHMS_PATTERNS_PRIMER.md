# Algorithms, Data Structures & Problem-Solving Patterns — A Primer

*The formal-CS training you skipped, done practically, and the single biggest interview lever you don't already have. Where the Collections reference gives you the Java data structures and their costs, this gives you the **reasoning** — how to measure cost, how to map a problem to a structure, and the dozen **patterns** that turn "I have no idea" into "oh, this is a sliding-window problem." Java snippets throughout; the ideas are language-agnostic.*

The uncomfortable truth about coding interviews: they don't test whether you can code (you can) — they test whether you can **recognise which of about a dozen patterns a problem is**, and reason about the cost. Nobody invents quicksort under pressure. What separates a pass from a fail is seeing "find a pair that sums to a target" and instantly thinking *hash map* or *two pointers*, then knowing that takes it from O(n²) to O(n). That recognition is learnable, and it's most of what this document teaches.

The one idea underneath everything: **almost every optimisation is a trade of space for time or a pre-arrangement that unlocks a shortcut.** A hash map spends memory to make lookups instant. Sorting spends O(n log n) up front to make binary search, two-pointers, and greedy possible. Once you see problems through that lens — *"what could I pre-compute or pre-arrange to make the expensive part cheap?"* — the patterns become variations on a theme.

Contents:

- **Part 1** — Big-O: measuring cost (the one piece of theory you must own)
- **Part 2** — data structures as tools: the problem → structure mapping
- **Part 3** — the core algorithms: recognise and reason
- **Part 4** — the problem-solving patterns (the interview core)
- **Part 5** — how to solve a coding problem: the method
- **Part 6** — the recognition guide: problem-smell → pattern
- **Part 7** — what matters for the interview vs the job

## Pattern index — the smell → the pattern

The single most useful table here. When a problem has one of these shapes, reach for the named pattern.

| The problem smells like… | Reach for | §|
|---|---|---|
| "find a pair / does X exist / count occurrences" | **hash map/set** (O(1) lookup) | §4.1 |
| "pair summing to target" *in a sorted array* | **two pointers** | §4.2 |
| "does the linked list have a cycle / find its middle" | **fast & slow pointers** | §4.3 |
| "longest/shortest/max **contiguous** subarray or substring" | **sliding window** | §4.4 |
| "sum/average of a **range** [i, j]", asked repeatedly | **prefix sums** | §4.5 |
| "search in **sorted** data" or "smallest value that satisfies…" | **binary search** (incl. on the answer) | §4.6 |
| "shortest path / levels / reachability" in a graph or grid | **BFS** | §4.7 |
| "explore all paths / connected components / does a path exist" | **DFS** | §4.7 |
| "all combinations / permutations / subsets / arrangements" | **backtracking** | §4.8 |
| "count the ways / min/max cost with overlapping choices" | **dynamic programming** | §4.9 |
| "k largest / k smallest / k most frequent / top-k" | **heap** | §4.10 |
| "merge / overlap / scheduling of ranges" | **intervals** (sort + sweep) | §4.11 |
| "next greater/smaller element" | **monotonic stack** | §4.12 |
| "are these connected / group into clusters" | **union-find** | §4.13 |
| "balanced parens / undo / nesting / evaluate expression" | **stack** | §2 |
| "process in order they arrived" | **queue** | §2 |

If you internalise one thing from this document, make it this table. In an interview, silently matching the problem to a row is 80% of the battle.

---

# Part 1 — Big-O: measuring cost

Big-O is the language for reasoning about whether an algorithm survives scale. It's not optional theory — it's how you defend a design decision and how you pass the "can you make it faster?" follow-up.

## 1.1 What it actually measures

Big-O describes how an algorithm's cost **grows as the input grows**, ignoring constant factors and lower-order terms. It's about *scaling*, not absolute wall-clock speed. `O(n)` doesn't mean "n nanoseconds" — it means "double the input, double the work."

Why we drop constants: at scale, the *shape* of growth dominates everything else. An O(n) algorithm that's "slow per step" beats an O(n²) one that's "fast per step" for large enough n — always, eventually. Big-O captures the shape and discards the noise.

## 1.2 The classes, with intuition and the brutal scaling table

| Notation | Name | The intuition | n=1,000 | n=1,000,000 |
|---|---|---|---|---|
| **O(1)** | constant | doesn't care how big the input is | 1 | 1 |
| **O(log n)** | logarithmic | halves the problem each step | ~10 | ~20 |
| **O(n)** | linear | look at each item once | 1k | 1M |
| **O(n log n)** | linearithmic | look at each item, log n times | ~10k | ~20M |
| **O(n²)** | quadratic | each item against every other item | 1M | **1,000,000,000,000** |
| **O(2ⁿ)** | exponential | try every subset | 10³⁰⁰ | heat death |
| **O(n!)** | factorial | try every arrangement | — | — |

The lesson is that last column. O(n²) at a million items is a **trillion** operations — minutes to hours. O(n log n) is twenty million — milliseconds. **The jump from O(n²) to O(n log n) or O(n) is the difference between "instant" and "never finishes"**, and it's almost always achieved by the same two moves: *hash it* or *sort it*.

Two anchors to memorise: **log n grows agonisingly slowly** (log₂ of a billion is ~30 — binary search a billion items in 30 steps), and **2ⁿ explodes instantly** (2³⁰ is already a billion — anything exponential is dead past ~n=25 unless pruned).

## 1.3 How to calculate it

Four mechanical rules:

- **Count the dominant loop.** One pass over n items → O(n).
- **Nested loops multiply.** A loop over n inside a loop over n → O(n²). A loop over n inside a loop over m → O(n·m).
- **Sequential steps add, then drop the smaller.** O(n) then O(n²) → O(n² + n) → **O(n²)** (the n vanishes; only the biggest term survives).
- **Drop constants.** O(2n) → O(n). O(n/2) → O(n). Three separate passes over n is still O(n), not O(3n).

```java
// O(n) — one pass
for (Question q : questions) process(q);

// O(n²) — the accidental killer: contains() is O(n), inside an O(n) loop
for (Question q : questions)
    if (approved.contains(q)) ...        // approved is a List → O(n) each check

// O(n) — the fix: a HashSet makes contains() O(1)
Set<Question> approvedSet = new HashSet<>(approved);
for (Question q : questions)
    if (approvedSet.contains(q)) ...      // O(1) each → O(n) total
```

That last transformation — spotting `list.contains()` inside a loop and hoisting to a `HashSet` — is *the* most common real-world and interview optimisation. Learn to see it on sight.

## 1.4 Space complexity, amortised, and best/average/worst

- **Space complexity** measures *memory* growth the same way. A hash map that stores every element to gain O(1) lookups costs O(n) *space* — that's the trade. Recursion costs stack space proportional to its depth.
- **Amortised** cost averages over a sequence. `ArrayList.add` is *usually* O(1), but occasionally O(n) when it doubles its capacity and copies — averaged over many adds, it's **amortised O(1)**. Same story for `HashMap` resizing (Collections reference §4.1).
- **Best / average / worst.** Quicksort is O(n log n) average but O(n²) worst case (a pathological pivot). Hash-map lookup is O(1) average but O(n) worst (everything collides). Interviewers usually want **worst case**; say which you mean.

> **The tell — Big-O:** the practical skill is spotting the accidental O(n²) — a linear search nested in a loop — and knowing the fix is almost always "hash it" or "sort it first." And keep perspective: for n ≤ ~100, an O(n²) solution is *fine* and often clearer; complexity only matters at scale. State your complexity, then say whether it matters for the expected input size.

---

# Part 2 — Data structures as tools

You have the Java specifics in the Collections reference (costs, `HashMap` internals, when to use each). This is the CS-level view: **each structure is a bundle of trade-offs, and picking the right one is half of algorithm design.** The mapping from *what you need* to *what you reach for*:

| You need… | Structure | Because | Java |
|---|---|---|---|
| indexed access, iterate in order | **dynamic array** | contiguous memory, O(1) index | `ArrayList` |
| O(1) membership / dedup | **hash set** | key → bucket | `HashSet` |
| O(1) key→value lookup | **hash table** | key → bucket | `HashMap` |
| sorted order + range queries | **balanced BST** | ordered, O(log n) | `TreeMap`/`TreeSet` |
| always grab the min/max next | **heap** | partially ordered, O(log n) | `PriorityQueue` |
| LIFO (undo, nesting, recursion) | **stack** | last-in-first-out | `ArrayDeque` |
| FIFO (process in arrival order) | **queue** | first-in-first-out | `ArrayDeque` |
| model relationships / networks | **graph** | nodes + edges | adjacency list (`Map<N,List<N>>`) |
| prefix/autocomplete search | **trie** | tree keyed by prefix | custom |
| connectivity / grouping | **union-find** | near-O(1) merge & find | custom |

## 2.1 The two power tools

Two of these do the heavy lifting in interview problems, and both are the "trade space/prep for speed" idea:

**The hash table** turns "is X here?" and "what maps to X?" from an O(n) scan into an O(1) lookup, by spending O(n) memory. It is the *first thing to reach for* when a brute force does repeated searching. Half the pattern index above is really "use a hash map."

**Sorting** costs O(n log n) once, and unlocks a suite of O(n) or O(log n) techniques that are impossible on unsorted data: **binary search**, **two pointers**, **greedy** sweeps, and **interval merging**. When a problem feels stuck, "what if it were sorted?" is one of the most productive questions you can ask.

## 2.2 Trees and graphs, briefly

- A **tree** is a graph with no cycles and one path between any two nodes (a hierarchy). **Binary search trees** keep left < node < right, giving O(log n) search *if balanced* — and "if balanced" is the catch, which is why production uses self-balancing variants (red-black), and why `TreeMap` is O(log n) guaranteed.
- A **graph** is nodes + edges, directed or not, weighted or not. It models anything relational: your `Concept ↔ SpecSection ↔ Question` web, a road network, a dependency graph. Represented as an **adjacency list** (`Map<Node, List<Node>>` — good for sparse graphs, the usual) or adjacency matrix (dense). Almost every "graph problem" is solved by BFS, DFS, or a shortest-path algorithm (§3.4).

> **The tell — structures:** the interview move is recognising which structure removes the bottleneck. "Repeated lookups?" → hash. "Need order or ranges?" → tree/sorted. "Min/max repeatedly?" → heap. "Relationships/paths?" → graph. Choosing right often collapses the whole problem.

---

# Part 3 — The core algorithms: recognise and reason

You won't implement these from scratch on the job (the library's are better), but you must *recognise* them and reason about their cost.

## 3.1 Searching

- **Linear search** — check each item, O(n). The default on unsorted data.
- **Binary search** — on **sorted** data, halve the search space each step: O(log n). The mechanic and the bug-prone boundaries:

```java
int binarySearch(int[] a, int target) {
    int lo = 0, hi = a.length - 1;
    while (lo <= hi) {                       // <= not < — the classic off-by-one
        int mid = lo + (hi - lo) / 2;        // not (lo+hi)/2 — avoids integer overflow
        if (a[mid] == target) return mid;
        else if (a[mid] < target) lo = mid + 1;
        else hi = mid - 1;
    }
    return -1;
}
```

Two details interviewers watch for: `lo + (hi - lo) / 2` (not `(lo + hi) / 2`, which can overflow) and the `<=` loop condition. Binary search is more general than "find in a sorted array" — see §4.6.

## 3.2 Sorting

You'll call `Arrays.sort()` / `list.sort()`, never write one — but know: the **comparison-sort lower bound is Ω(n log n)** (you can't beat it comparing elements), Java's object sort (**TimSort**) is **stable** (equal elements keep order — Collections reference §6), and the primitive `Arrays.sort` (dual-pivot quicksort) is *not* stable. Conceptually: **merge sort** (divide in half, sort each, merge — stable, O(n log n) guaranteed, O(n) space) and **quicksort** (partition around a pivot — in-place, O(n log n) average but O(n²) worst). You sort not because you need order for its own sake but because §2.1 — it unlocks the O(n) techniques.

## 3.3 Recursion & divide-and-conquer

A function that calls itself on a smaller input, with a **base case** that stops it. The model: trust that the recursive call solves the smaller problem, and combine. Divide-and-conquer (merge sort, binary search) is recursion that splits the problem into independent halves. Two costs to remember: each call consumes **stack space** (deep recursion → `StackOverflowError`), and naive recursion can *recompute* the same subproblem exponentially — which is exactly what DP fixes (§4.9).

## 3.4 Graph traversal — the algorithms behind half the patterns

- **BFS (breadth-first search)** — explore level by level using a **queue**. Finds the **shortest path in an unweighted graph** (the first time you reach a node is via a shortest path). O(V + E).
- **DFS (depth-first search)** — go as deep as possible before backtracking, using a **stack** (or recursion). Good for "does a path exist," connected components, cycle detection, and topological sort. O(V + E).
- **Dijkstra** — shortest path in a **weighted** graph (non-negative weights), using a priority queue. The weighted upgrade of BFS.
- **Topological sort** — linear ordering of a DAG respecting dependencies ("which order to run these tasks / build these modules"). Kahn's algorithm (BFS on in-degrees) or DFS.

The rule of thumb: **shortest/fewest-steps → BFS; explore-everything/does-a-path-exist → DFS; weighted-shortest → Dijkstra; ordering-with-dependencies → topological sort.**

## 3.5 Dynamic programming — demystified

DP intimidates because it's taught abstractly. Concretely: **DP is brute-force recursion where the subproblems overlap, made fast by remembering answers you've already computed.** Two ingredients: **overlapping subproblems** (the same sub-question comes up repeatedly) and **optimal substructure** (the answer is built from answers to sub-questions). Two styles:

- **Top-down (memoisation):** write the natural recursion, then cache results (`Map`/array). Easy to derive from the brute force.
- **Bottom-up (tabulation):** fill a table from the base cases up. No recursion, no stack risk.

Fibonacci is the "hello world": naive recursion is O(2ⁿ) because it recomputes; memoised, it's O(n) because each value is computed once.

```java
Map<Integer, Long> memo = new HashMap<>();
long fib(int n) {
    if (n < 2) return n;
    return memo.computeIfAbsent(n, k -> fib(k - 1) + fib(k - 2));   // compute once, reuse
}
```

## 3.6 Greedy

Make the locally-optimal choice at each step and hope it yields the global optimum. Fast and simple *when it works* — and the hard part is proving it does. It works for some problems (interval scheduling: always pick the earliest-finishing; making change with canonical coins) and *fails* for others (change with arbitrary coin sets needs DP). The interview trap is applying greedy where it doesn't hold, so be ready to justify *why* the local choice is safe, or reach for DP instead.

> **The tell — core algorithms:** recognise them, don't reinvent them. "Shortest path" isn't a puzzle to solve from scratch — it's "BFS/Dijkstra." "Count the ways with overlapping choices" is "DP." Naming the algorithm is the answer; the code is mechanical once you've named it.

---

# Part 4 — The problem-solving patterns

This is the interview core — the dozen shapes that cover the large majority of coding problems. For each: **the smell** that signals it, **the mechanic**, **the cost**, and a snippet. Match the problem to the pattern (via the index at the top) and you've mostly solved it.

## 4.1 Hashing for lookups — the #1 move

**Smell:** "find / does it contain / count / has a pair / seen before." **Mechanic:** trade O(n) space for O(1) lookups, turning a nested-loop O(n²) into a single-pass O(n). The canonical two-sum:

```java
// find two indices whose values sum to target — O(n), one pass
int[] twoSum(int[] nums, int target) {
    Map<Integer, Integer> seen = new HashMap<>();          // value → index
    for (int i = 0; i < nums.length; i++) {
        int need = target - nums[i];
        if (seen.containsKey(need)) return new int[]{seen.get(need), i};
        seen.put(nums[i], i);
    }
    return new int[]{-1, -1};
}
```

The brute force is two nested loops, O(n²). The hash map remembers what it's seen, so each element checks for its complement in O(1). **When in doubt, this is the first optimisation to try.** (Practiq: "have I already extracted this question?" — a `Set` of seen hashes.)

## 4.2 Two pointers

**Smell:** a **sorted** array, and you're looking for a pair/triplet, or partitioning. **Mechanic:** one pointer at each end, move them inward based on the comparison — O(n) after sorting, O(1) space (beats the hash map's O(n) space when the input is already sorted):

```java
// pair summing to target in a SORTED array
int lo = 0, hi = a.length - 1;
while (lo < hi) {
    int sum = a[lo] + a[hi];
    if (sum == target) return new int[]{lo, hi};
    else if (sum < target) lo++;        // need bigger → move left pointer up
    else hi--;                          // need smaller → move right pointer down
}
```

Also the tool for removing duplicates in-place, reversing, and merging sorted lists.

## 4.3 Fast & slow pointers (Floyd's)

**Smell:** a linked list or a cycle — "does it loop," "find the middle," "find the cycle start." **Mechanic:** two pointers at different speeds (slow +1, fast +2). If there's a cycle, they meet; when fast reaches the end, slow is at the middle. O(n), O(1) space.

```java
boolean hasCycle(Node head) {
    Node slow = head, fast = head;
    while (fast != null && fast.next != null) {
        slow = slow.next;               // +1
        fast = fast.next.next;          // +2
        if (slow == fast) return true;  // they met → cycle
    }
    return false;
}
```

## 4.4 Sliding window

**Smell:** "longest / shortest / max / min **contiguous** subarray or substring satisfying a condition." **Mechanic:** maintain a window [left, right]; expand right, and when the condition breaks, shrink left — each element enters and leaves once, so O(n) instead of the O(n²) of checking every subarray. Fixed-size and variable-size variants:

```java
// max sum of any contiguous subarray of size k — fixed window, O(n)
int windowSum = 0;
for (int i = 0; i < k; i++) windowSum += a[i];
int max = windowSum;
for (int i = k; i < a.length; i++) {
    windowSum += a[i] - a[i - k];       // slide: add the new, drop the old
    max = Math.max(max, windowSum);
}
```

For variable windows (e.g. "longest substring with no repeated character"), expand right adding to a `Set`/`Map`, and shrink left until the constraint holds again.

## 4.5 Prefix sums

**Smell:** repeated "sum/average of range [i, j]" queries, or "subarray summing to k." **Mechanic:** precompute cumulative sums once (O(n)); then any range sum is `prefix[j] - prefix[i]` in O(1). Trades O(n) space for O(1) queries.

```java
int[] prefix = new int[a.length + 1];
for (int i = 0; i < a.length; i++) prefix[i + 1] = prefix[i] + a[i];
// sum of a[i..j] inclusive:
int rangeSum = prefix[j + 1] - prefix[i];
```

## 4.6 Binary search — including on the answer

**Smell (obvious):** "search in sorted data" → §3.1. **Smell (the clever one):** "find the **minimum/maximum value that satisfies** some monotonic condition" — even with no array to search. If you can ask a yes/no question whose answer flips exactly once as the candidate increases ("can we do it in ≤ X time?"), you can **binary search the answer space**, turning an O(n) or worse scan of candidates into O(log range). This is the pattern that separates people who "know binary search" from people who *see* it. Example smells: "minimum capacity to ship in D days," "smallest divisor such that…" — binary search the value, check feasibility in O(n).

## 4.7 BFS / DFS on trees, graphs, and grids

**Smell:** anything with nodes/edges, a tree, or a **grid** (grids are graphs — each cell connects to its neighbours). **BFS** for shortest path / level-order / fewest steps; **DFS** for "does a path exist," connected components, flood fill, and exhaustive exploration.

```java
// BFS — shortest path in an unweighted graph; the queue is the whole trick
Queue<Node> q = new ArrayDeque<>();
Set<Node> seen = new HashSet<>();
q.add(start); seen.add(start);
int steps = 0;
while (!q.isEmpty()) {
    int levelSize = q.size();
    for (int i = 0; i < levelSize; i++) {   // process one level at a time
        Node n = q.poll();
        if (n == goal) return steps;
        for (Node nb : n.neighbours())
            if (seen.add(nb)) q.add(nb);    // add() returns false if already seen
    }
    steps++;
}

// DFS — recursive; explore everything reachable
void dfs(Node n, Set<Node> seen) {
    if (!seen.add(n)) return;               // already visited
    for (Node nb : n.neighbours()) dfs(nb, seen);
}
```

For grids, "neighbours" is the four (or eight) adjacent cells. Flood fill, island counting, and maze solving are all this.

## 4.8 Backtracking

**Smell:** "generate **all** combinations / permutations / subsets," or constraint-satisfaction ("place N queens," "solve the sudoku"). **Mechanic:** build a candidate incrementally; at each step try each option, recurse, then **undo** (backtrack) and try the next. It's DFS over the space of partial solutions, pruning branches that can't work.

```java
// all subsets — the backtracking skeleton
void subsets(int[] nums, int start, List<Integer> current, List<List<Integer>> out) {
    out.add(new ArrayList<>(current));          // record the current partial solution
    for (int i = start; i < nums.length; i++) {
        current.add(nums[i]);                   // choose
        subsets(nums, i + 1, current, out);     // explore
        current.remove(current.size() - 1);     // un-choose (backtrack)
    }
}
```

Backtracking is inherently exponential (there *are* exponentially many subsets/permutations) — the skill is **pruning** dead branches early. If the problem asks for a *count* or an *optimum* rather than *all* solutions, it's usually DP instead.

## 4.9 Dynamic programming

**Smell:** "count the number of ways," "min/max cost/length," "can you reach/make X," *with choices that overlap*. **Mechanic:** §3.5 — define the recurrence (the answer in terms of smaller answers), then memoise. The hard part is *finding the recurrence*; the rest is mechanical. Classic shapes: knapsack (choose items under a constraint), longest common subsequence, coin change, edit distance, grid path counting. In an interview, derive the brute-force recursion first, *then* add memoisation — that's the reliable path, and it shows your working.

## 4.10 Heap / top-K

**Smell:** "k largest / k smallest / k most frequent," or "median of a stream," or "merge k sorted lists." **Mechanic:** a heap gives you O(log n) insert and O(1) peek at the min/max. For **top-k**, keep a heap of size k (a *min*-heap for the k *largest* — the smallest of your k sits at the top, ready to be evicted when something bigger arrives). O(n log k) — far better than sorting the whole thing (O(n log n)) when k ≪ n.

```java
// k largest elements — a size-k MIN-heap
PriorityQueue<Integer> heap = new PriorityQueue<>();   // min-heap by default
for (int x : nums) {
    heap.offer(x);
    if (heap.size() > k) heap.poll();     // evict the smallest → the k largest remain
}
// heap now holds the k largest
```

(Practiq: "the 10 hardest unreviewed questions" is a top-k over difficulty.)

## 4.11 Intervals

**Smell:** ranges — "merge overlapping intervals," "can this person attend all meetings," "minimum rooms needed." **Mechanic:** **sort by start** (occasionally by end), then sweep once, comparing each interval to the previous. Sorting is the unlock (§2.1); the sweep is O(n).

```java
// merge overlapping intervals
intervals.sort(Comparator.comparingInt(iv -> iv[0]));   // by start
List<int[]> merged = new ArrayList<>();
for (int[] iv : intervals) {
    if (merged.isEmpty() || merged.get(merged.size()-1)[1] < iv[0])
        merged.add(iv);                                  // no overlap → new interval
    else
        merged.get(merged.size()-1)[1] = Math.max(merged.get(merged.size()-1)[1], iv[1]); // extend
}
```

## 4.12 Monotonic stack

**Smell:** "next greater / next smaller element," "largest rectangle in a histogram," "daily temperatures." **Mechanic:** a stack kept in increasing or decreasing order; as you scan, pop everything the current element "beats," which resolves those elements' answers in O(1) each — O(n) total instead of O(n²). Niche but unmistakable once you know the smell ("for each element, find the nearest bigger one to its right").

## 4.13 Union-Find (disjoint set)

**Smell:** "are these two connected," "how many groups/clusters," "detect a cycle in an undirected graph," "number of islands as edges arrive." **Mechanic:** each element points to a representative; `find` follows the chain to the root, `union` merges two groups. With **path compression** and **union by rank**, both are near-O(1) (amortised inverse-Ackermann — effectively constant). The go-to for **dynamic connectivity** where BFS/DFS would be re-run repeatedly.

```java
int[] parent;
int find(int x) { return parent[x] == x ? x : (parent[x] = find(parent[x])); }  // path compression
void union(int a, int b) { parent[find(a)] = find(b); }
```

> **The tell — patterns:** the win isn't knowing the code (it's short) — it's **recognising the smell fast**. Drill the *mapping* (the pattern index), not the implementations. In the room, name the pattern out loud ("this looks like a sliding-window problem because we want the longest contiguous run") — it shows the interviewer exactly the recognition they're testing for, even before you write a line.

---

# Part 5 — How to solve a coding problem: the method

Recognition gets you the pattern; a *process* gets you a clean solution and the communication marks. Follow these steps out loud — the thinking is what's being assessed, not just the final code.

1. **Clarify.** Restate the problem. Ask about constraints: input size (tells you the target complexity — n=10⁶ rules out O(n²)), value ranges, duplicates, empty/null inputs, sorted or not. Interviewers *plant* ambiguity to see if you ask.
2. **Work an example by hand.** A concrete small case builds intuition and catches misunderstandings before you code.
3. **State the brute force.** Say the obvious O(n²) (or exponential) solution first — "the naive approach is to check every pair, which is O(n²)." This is not a weakness; it establishes a baseline and buys thinking time. *Never* stay silent hunting for the clever answer.
4. **Optimise — find the pattern.** Now ask the productive questions: *What am I recomputing? What could I pre-sort or pre-hash? Which pattern does this smell like?* Match to the index. State the improved complexity before coding.
5. **Confirm the approach, then code.** Say what you're about to do, then write it cleanly — good names, small steps. Talk while you type.
6. **Test.** Walk your code through the example. Then hit the **edge cases** (below). Finding your own bug is a strong signal; the interviewer finding it for you is not.

**Edge-case checklist** (run this on every problem): empty input, single element, all-duplicates, all-same, already-sorted / reverse-sorted, negatives and zero, integer overflow, the target at the boundaries, and — for graphs/trees — cycles, disconnected parts, and null nodes.

**Complexity as a signal:** the given input size *tells you* the intended complexity. n ≤ 20 → exponential/backtracking is expected. n ≤ 10³ → O(n²) is fine. n ≤ 10⁶ → you need O(n log n) or O(n). n ≤ 10⁹ → O(log n) or O(1), so think binary search or maths. Reading this off the constraints is a pro move.

> **The tell — method:** brute force first, *out loud*, always. Then optimise deliberately by naming what you're recomputing and which pattern removes it. The interview rewards visible, structured reasoning over a silent leap to the perfect answer — and the same discipline (state the baseline, name the trade, justify the improvement) is exactly how you defend a design decision on the job.

---

# Part 6 — The recognition guide

The consolidated decision reference. Two lenses: match the **smell** to the pattern (the top index, repeated here as the core), and read the **target complexity** off the constraints.

**By smell → pattern:** the pattern index at the top of this document *is* the decision guide — keep it to hand. The meta-groupings, if you want fewer things to remember:

- **"Find / exists / count / seen"** → **hash** it (§4.1). The default first optimisation.
- **"Sorted, or would-be-easier-sorted"** → **sort**, then two-pointers / binary-search / greedy / intervals (§4.2, §4.6, §4.11).
- **"Contiguous run"** → **sliding window** (§4.4); **"range queries"** → **prefix sums** (§4.5).
- **"Graph / grid / tree / paths"** → **BFS** (shortest) or **DFS** (explore) (§4.7); connectivity → **union-find** (§4.13).
- **"All arrangements"** → **backtracking** (§4.8); **"count ways / optimum with overlap"** → **DP** (§4.9).
- **"Top-k / min-max repeatedly"** → **heap** (§4.10).

**By complexity target (from the constraints):**

| Input size | Expected complexity | Suggests |
|---|---|---|
| n ≤ ~20 | O(2ⁿ), O(n!) | backtracking, brute force |
| n ≤ ~500 | O(n³) | DP with two-plus dimensions |
| n ≤ ~5,000 | O(n²) | nested loops, simple DP |
| n ≤ ~10⁶ | O(n log n), O(n) | sort, hash, sliding window, single pass |
| n ≤ ~10⁹ | O(log n), O(1) | binary search, maths |

---

# Part 7 — What matters: the interview vs the job

Two honest, different answers, because they pull slightly differently.

**For the interview:** it's pattern *recognition* under time pressure. The dozen patterns in Part 4 cover the large majority of coding-round problems; the skill is matching fast and communicating the reasoning (Part 5). Drill the *mapping* (smell → pattern), not rote implementations — you want to *recognise*, then reconstruct the short code. Do enough problems that the smells become reflexive. And keep the honest perspective: **you are not being tested on inventing quicksort or deriving Dijkstra** — you're being tested on recognising "this is a BFS," structuring an answer, and reasoning about cost.

**For the job:** the daily reality is narrower and you already do most of it. It's (1) **choose the right data structure** so operations are cheap (Collections reference), (2) **don't write accidental O(n²)** — the `contains`-in-a-loop trap (§1.3), and (3) **know when a round trip dominates** — the real bottleneck in a backend service is almost never your in-memory algorithm; it's the database query, the N+1, the network call (Engineer's Map §1.2, data-access primer §2.8). A perfectly O(n) loop wrapped around a query in a loop is still slow. The job-relevant instinct is *"count the slow boundary crossings before optimising the loop."*

The two meet in one habit: **reason about cost explicitly**, whether that's Big-O in an interview or "how many DB round trips does this endpoint make" in a code review. Both are the same muscle.

> **The tell — the whole document:** interviews test whether you can look at a problem and *name the pattern*; the job tests whether you can *pick the right structure and not cross slow boundaries needlessly*. Drill the pattern index for the former; keep the "where's the real bottleneck?" instinct for the latter. Neither requires inventing algorithms — both require recognising them and reasoning about cost.

---

# How to expand this

- *The Java data structures in depth* (costs, `HashMap` internals, sorting, concurrency): Collections reference.
- *Where algorithmic cost meets the real bottleneck* (memory hierarchy, round trips): Engineer's Map §1–2; the N+1 problem specifically is data-access primer §2.8.
- *Candidates for their own deeper treatment, if useful:* **graph algorithms in full** (Dijkstra/A*/topological sort/MST with worked code); **dynamic programming as its own primer** (the recurrence-finding method, the classic problem families, top-down vs bottom-up); a **curated problem set** mapped to these patterns, as a drill companion to your interview dashboard — this last one would slot directly into your existing prep system.

*This is a foundations primer written from stable CS knowledge; none of it drifts. The patterns here are the standard interview canon; the value is in drilling the recognition until the smells are reflexive.*
