# Git — A Primer №53

*The tool you use more than any other and the one most people run on memorised incantations. This teaches **the model first** — because once you understand what Git actually stores, the commands stop being spells and become obvious consequences. Then the workflows, the undo operations (the genuinely valuable part), and how to investigate history.*

The single insight that makes Git click: **a commit is a complete snapshot of your project, and a branch is nothing but a movable label pointing at one commit.** Not a diff, not a folder, not a copy of the code — a label. Merging, rebasing, resetting, cherry-picking and every "how do I undo this" question become mechanical once you hold that. Most Git confusion is people reasoning about branches as if they were containers of commits, when they're pointers into a graph.

The second, and the one that removes fear: **almost nothing in Git is truly lost.** Committed work is recoverable for weeks even after you "delete" it, because `reflog` records every position `HEAD` has occupied. Knowing that changes how boldly you use the tool.

Contents:

- **Part 1** — the model: objects, commits, branches, HEAD
- **Part 2** — the three trees and the basic cycle
- **Part 3** — branching and merging
- **Part 4** — rebase, and merge vs rebase
- **Part 5** — working with remotes
- **Part 6** — undoing things
- **Part 7** — conflicts
- **Part 8** — investigating history
- **Part 9** — the useful rest
- **Part 10** — branching strategies and good practice
- **Part 11** — when to use what

## Panic index — "I've done something wrong"

| Situation | Command | §|
|---|---|---|
| Changed a file, want it back | `git restore <file>` | §6.1 |
| Staged something by mistake | `git restore --staged <file>` | §6.1 |
| Bad commit message (not pushed) | `git commit --amend` | §6.2 |
| Forgot a file in the last commit | `git add <file> && git commit --amend --no-edit` | §6.2 |
| Undo last commit, keep the changes | `git reset --soft HEAD~1` | §6.3 |
| Undo last commit, discard the changes | `git reset --hard HEAD~1` | §6.3 |
| Undo a commit that's already pushed | `git revert <sha>` | §6.4 |
| Committed to the wrong branch | `git reset --hard HEAD~1` then cherry-pick | §6.5 |
| Need to switch branches mid-work | `git stash` | §9.1 |
| Deleted a branch with work on it | `git reflog` then `git branch <name> <sha>` | §6.6 |
| Rebase has gone wrong | `git rebase --abort` | §4.2 |
| Pulled and got a mess | `git reset --hard origin/<branch>` (discards local!) | §6.3 |
| Which commit broke this? | `git bisect` | §8.3 |
| Who wrote this line and why? | `git blame` then `git show` | §8.2 |

---

# Part 1 — The model

## 1.1 What Git actually stores

Git is a **content-addressable object store**. Four object types, each identified by the SHA-1/SHA-256 hash of its content:

| Object | Holds |
|---|---|
| **blob** | file contents (no name — just bytes) |
| **tree** | a directory listing: names → blobs and other trees |
| **commit** | a pointer to one tree, plus parent commit(s), author, date, message |
| **tag** | a named pointer to a commit, with a message |

So a **commit is a snapshot**, not a diff. It points to a tree that represents your entire project at that moment. Git *shows* you diffs by comparing two snapshots, but it stores the states. (Identical files across commits are stored once, because content-addressing deduplicates — that's why full snapshots aren't wasteful.)

## 1.2 Commits form a DAG

Each commit records its **parent**, so commits form a directed acyclic graph pointing backwards through history. A merge commit simply has **two** parents.

```
A ── B ── C ── D          ← main
           \
            E ── F        ← feature   (E's parent is C)
```

That graph *is* your repository's history. Everything else is navigation and manipulation of it.

## 1.3 Branches are labels

**A branch is a file containing a commit hash.** Literally — look in `.git/refs/heads/`. It's a movable pointer, which is why creating one is instant and free regardless of repository size.

When you commit, the branch label moves forward to the new commit. That's the entire mechanism.

**HEAD** is a pointer to *what you currently have checked out* — normally the name of a branch (`ref: refs/heads/main`). "Detached HEAD" simply means HEAD points directly at a commit rather than at a branch, so new commits have no label following them — which is why they're easy to lose (recoverable via `reflog`, §6.6).

This model explains things that otherwise seem arbitrary:

- Deleting a branch deletes a *label*, not commits. The commits persist until garbage collection, unreferenced.
- "Fast-forward" merge = just sliding a label forward, because there's nothing to combine.
- Rebase "moves" commits by **creating new ones** with different parents — the originals still exist until GC, which is why rebasing rewrites hashes.

> **The tell — the model:** when confused, draw the graph. Where are the commits, and where do the labels point? Almost every Git question resolves to "which commit should this label point at, and what should the graph look like?"

---

# Part 2 — The three trees and the basic cycle

## 2.1 The three areas

Git's distinguishing feature versus other VCSs is the **staging area** (index) — an intermediate space where you compose your next commit.

```
Working directory  →  Staging area (index)  →  Repository (HEAD)
   your files            git add                  git commit
```

- **Working directory** — the files you're editing right now.
- **Staging area** — what will go into the next commit. You choose.
- **Repository** — committed history.

The staging area exists so you can commit *part* of your work. If you fixed a bug and also reformatted three files, you can stage and commit them separately — which produces a history where each commit is one logical change (§10.3).

## 2.2 The everyday cycle

```bash
git status                     # what's changed, staged, untracked — run this constantly
git diff                       # working dir vs staged
git diff --staged              # staged vs last commit (what you're about to commit)
git add <file>                 # stage a file
git add -p                     # stage HUNKS interactively — the underused power tool
git commit -m "message"        # commit staged changes
git log --oneline --graph      # see the graph
```

`git add -p` deserves emphasis: it walks you through each change hunk-by-hunk and asks whether to stage it. It lets you split a messy working directory into clean, logical commits, and it forces you to *re-read your own diff* before committing — which catches a surprising number of stray debug statements.

---

# Part 3 — Branching and merging

## 3.1 Branching

```bash
git branch feature/questions-filter          # create (label at current commit)
git switch feature/questions-filter          # move HEAD to it
git switch -c feature/questions-filter       # create and switch (modern)
git checkout -b feature/questions-filter     # same, older syntax
git branch -d old-branch                     # delete (safe: refuses if unmerged)
git branch -D old-branch                     # force delete
```

`switch` and `restore` are the modern split of the overloaded `checkout` (which did both branch-switching and file-restoring, a known source of confusion). Prefer them.

## 3.2 Merging

```bash
git switch main
git merge feature/questions-filter
```

Two possible outcomes:

**Fast-forward** — if `main` hasn't moved since the branch was created, there's nothing to combine; Git just slides the label forward.

```
Before:  A ── B ── C(main) ── D ── E(feature)
After:   A ── B ── C ── D ── E(main, feature)
```

**Three-way merge** — both branches have new commits, so Git finds the common ancestor and creates a **merge commit with two parents**:

```
A ── B ── C ── F ──── M(main)
           \         /
            D ── E ─╯   (feature)
```

Useful flags: `--no-ff` forces a merge commit even when a fast-forward is possible (preserving the fact that a branch existed); `--squash` combines all the branch's changes into a single un-committed change set for you to commit as one.

---

# Part 4 — Rebase, and merge vs rebase

## 4.1 What rebase does

Rebase **replays your commits onto a new base**, creating new commits with new hashes and new parents:

```
Before:  A ── B ── C ── F        (main)
              \
               D ── E            (feature)

After:   A ── B ── C ── F        (main)
                         \
                          D' ── E'   (feature — new commits, same changes)
```

```bash
git switch feature
git rebase main            # replay feature's commits on top of main
```

The result is a **linear history** with no merge commit. Note D' and E' are *new commits*: same content, different parents, different hashes.

## 4.2 Interactive rebase — cleaning up before you share

The most valuable everyday use:

```bash
git rebase -i HEAD~4        # edit the last 4 commits
```

An editor opens listing the commits, and you choose an action per line:

| Action | Does |
|---|---|
| `pick` | keep as is |
| `reword` | change the message |
| `edit` | pause to amend the content |
| `squash` | merge into the previous commit, combining messages |
| `fixup` | merge into the previous, **discarding** this message |
| `drop` | delete the commit |

This is how you turn "wip", "fix", "fix again", "actually fix" into one clean commit before opening a PR. Reordering lines reorders the commits.

If it goes wrong at any point: **`git rebase --abort`** returns you exactly to where you started. `--continue` proceeds after resolving a conflict.

## 4.3 The golden rule

**Never rebase commits that others may have based work on** — anything already pushed to a shared branch. Rebasing rewrites hashes, so collaborators' history no longer matches yours, and reconciling that is genuinely painful.

The safe formulation: **rebase your own unpushed work freely; merge anything shared.** If you must force-push a rebased branch (normal for your own PR branch), use `--force-with-lease` rather than `--force` — it refuses if someone else has pushed in the meantime, which turns a potential data-loss event into an error message.

## 4.4 Choosing

| | **Merge** | **Rebase** |
|---|---|---|
| History | true, with all branching visible | linear, tidy |
| Hashes | preserved | rewritten |
| Safe on shared branches | **yes** | **no** |
| Conflicts | resolved once, in the merge | possibly once **per commit** replayed |
| Traceability | shows how work actually happened | shows how you wish it had |

> **The tell — merge or rebase:** **rebase to clean up your own branch before sharing; merge to integrate shared branches.** A common, sane team convention: rebase your feature branch onto main to stay current, squash-or-tidy your commits, then merge the PR. Whatever you choose, the team should choose the same thing — mixed conventions produce confusing history.

---

# Part 5 — Working with remotes

## 5.1 The model

A **remote** is a named reference to another copy of the repository (`origin` by convention). Crucially, Git keeps **remote-tracking branches** — `origin/main` is your local record of where `main` was on the remote *when you last fetched*. It is not live.

```bash
git fetch origin             # update remote-tracking branches. Changes nothing local.
git pull                     # fetch + merge (or rebase) into your branch
git push origin main         # send your commits
```

**`fetch` is always safe** — it only updates your knowledge of the remote. `pull` is `fetch` plus an integration step that modifies your working branch, which is why surprises happen. A useful habit: `git fetch` then `git log HEAD..origin/main` to see what's coming before integrating.

## 5.2 Pull strategies

```bash
git pull --rebase            # replay your local commits on top of the fetched ones
git config --global pull.rebase true     # make it the default
```

`pull --rebase` avoids the clutter of "Merge branch 'main' of origin..." commits that appear when two people work on the same branch. Widely recommended as a default for the shared-branch case.

## 5.3 Pushing

```bash
git push -u origin feature/x         # first push; -u sets upstream tracking
git push                             # thereafter
git push --force-with-lease          # after rebasing YOUR branch — safer than --force
```

---

# Part 6 — Undoing things

The genuinely valuable section. The key question is always: **what state do you want, and has it been shared?**

## 6.1 Discard uncommitted work

```bash
git restore <file>                  # discard working-directory changes to a file
git restore --staged <file>         # unstage, keeping the changes
git restore .                       # discard ALL working changes (unrecoverable — no commit exists)
git clean -fd                       # delete untracked files and directories (also unrecoverable)
```

These are the genuinely dangerous ones, precisely because uncommitted work has never been recorded. **Commit early — even a rough WIP commit — and everything afterwards is recoverable.**

## 6.2 Amend the last commit

```bash
git commit --amend                       # change the message and/or add staged changes
git commit --amend --no-edit             # add staged changes, keep the message
```

Amend **replaces** the last commit with a new one (new hash), so only do it on unpushed commits (§4.3).

## 6.3 Reset — move the branch label

`reset` moves the current branch pointer to a different commit. The mode determines what happens to your files:

| Mode | Branch label | Staging area | Working directory |
|---|---|---|---|
| `--soft` | moves | **unchanged** | **unchanged** |
| `--mixed` *(default)* | moves | reset | **unchanged** |
| `--hard` | moves | reset | **reset — changes destroyed** |

```bash
git reset --soft HEAD~1     # undo the commit, keep everything staged (recommit differently)
git reset HEAD~1            # undo the commit, keep changes unstaged
git reset --hard HEAD~1     # undo the commit and throw the changes away
```

`--soft` is the one to reach for when you committed too early or want to restructure. `--hard` is the only destructive one — and even then the *commit* is recoverable via reflog (§6.6); only uncommitted changes are truly gone.

## 6.4 Revert — undo publicly

```bash
git revert <sha>            # create a NEW commit that undoes <sha>
```

`revert` doesn't rewrite history; it adds a commit whose changes are the inverse. **This is the correct way to undo anything already pushed** — everyone's history stays valid, and the record shows that a change was made and then undone, which is honest and traceable.

**reset vs revert:** reset rewrites (private history), revert appends (public history).

## 6.5 Committed to the wrong branch

```bash
git log --oneline -1                 # note the sha
git reset --hard HEAD~1              # remove it from the wrong branch
git switch correct-branch
git cherry-pick <sha>                # apply it here
```

## 6.6 Reflog — the safety net

```bash
git reflog                           # every position HEAD has been in, with shas
git branch recovered <sha>           # resurrect work at that point
git reset --hard <sha>               # or go straight back
```

**`reflog` is the reason you can be bold with Git.** It records every `HEAD` movement — commits, checkouts, resets, rebases — for around 90 days by default. A "lost" commit after a bad `reset --hard` or a botched rebase is almost always sitting right there. Learn this command before you need it.

---

# Part 7 — Conflicts

## 7.1 What they are

A conflict occurs when two branches change **the same region of the same file** and Git can't determine which to keep. It marks the file and stops:

```
<<<<<<< HEAD
    return questionRepository.findAll(spec);
=======
    return questionRepository.findAll(spec, pageable);
>>>>>>> feature/pagination
```

Above the `=======` is **your current branch** (HEAD); below is the **incoming** change.

## 7.2 Resolving

1. `git status` lists the conflicted files.
2. Edit each: choose one side, the other, or write a combination. **Delete all the marker lines.**
3. `git add <file>` to mark it resolved.
4. `git commit` (merge) or `git rebase --continue` (rebase).

Escape hatches: `git merge --abort` / `git rebase --abort` return you to the pre-attempt state.

Useful: `git checkout --ours <file>` / `--theirs <file>` take one side wholesale — but note **"ours" and "theirs" invert during a rebase**, because rebase replays *your* commits onto *their* base, making the upstream branch "ours." That inversion catches everyone at least once.

## 7.3 Reducing them

Small, frequent merges from the shared branch (a long-lived branch diverges further every day); short-lived branches; agreed formatting so reformatting doesn't create phantom conflicts; and communication when two people are in the same file. **`git rerere`** (reuse recorded resolution) remembers how you resolved a conflict and reapplies it automatically — genuinely useful during a long rebase where the same conflict recurs.

---

# Part 8 — Investigating history

This is where Git earns its keep beyond version storage.

## 8.1 Log

```bash
git log --oneline --graph --all           # the shape of the repository
git log -p <file>                         # every change to a file, with diffs
git log -S "findApprovedByConcept"        # commits that ADDED or REMOVED this string  ← powerful
git log --since="2 weeks ago" --author=jamie
git log main..feature                     # commits on feature that aren't on main
git show <sha>                            # one commit in full
```

**`git log -S`** ("pickaxe") is the underused one: it finds the commit where a particular piece of code appeared or vanished, which is usually exactly what you want when tracking down where behaviour came from.

## 8.2 Blame

```bash
git blame <file>                # who last changed each line, and in which commit
git blame -L 40,60 <file>       # just those lines
```

Then `git show <sha>` on the interesting commit to read the message and the surrounding change. **Use it to find context, not culprits** — the value is the commit message explaining *why*, which is precisely why writing good messages matters (§10.3).

## 8.3 Bisect — the debugging power tool

Binary search through history to find the commit that introduced a bug. Logarithmic: 1,000 commits takes about 10 tests.

```bash
git bisect start
git bisect bad                    # current commit is broken
git bisect good v1.2.0            # this old tag was fine
# Git checks out a midpoint; you test and tell it:
git bisect good                   # ...or: git bisect bad
# repeat until Git names the first bad commit
git bisect reset                  # return to where you were
```

And it can be automated entirely if you have a test that detects the bug:

```bash
git bisect run ./gradlew test --tests QuestionServiceTest
```

Git then finds the culprit with no interaction. **This is the single highest-leverage Git command for debugging** (№44 §9.1) — it converts "when did this break?" from archaeology into a mechanical process.

---

# Part 9 — The useful rest

## 9.1 Stash

```bash
git stash                        # shelve working changes, clean the working directory
git stash -u                     # include untracked files
git stash pop                    # reapply and remove from the stash
git stash list                   # what's shelved
```

For "I need to switch branches right now but I'm mid-change." Don't let stashes accumulate — they're easy to forget and carry no message by default (`git stash push -m "message"` helps).

## 9.2 Cherry-pick

```bash
git cherry-pick <sha>            # apply one commit's changes here
```

For hotfixes that need to go to both a release branch and main, or rescuing one commit from an abandoned branch. Used habitually it produces duplicated commits and confusing history.

## 9.3 Tags

```bash
git tag -a v1.0.0 -m "First release"
git push origin v1.0.0
```

Annotated tags (`-a`) are proper objects with a message and author — use them for releases. Lightweight tags are just labels.

## 9.4 Worktrees

```bash
git worktree add ../practiq-hotfix main
```

Check out a second branch into a *separate directory* sharing the same repository. Better than stashing when you need to work on two branches simultaneously — no context switching, both builds intact.

## 9.5 Hooks

Scripts in `.git/hooks` that run at lifecycle points: `pre-commit` (lint/format), `commit-msg` (enforce message format), `pre-push` (run tests). Local by default and not versioned, so teams use tools like Husky or pre-commit to share them. Keep them fast — a slow pre-commit hook gets bypassed with `--no-verify`, which defeats the purpose.

## 9.6 .gitignore and large files

Ignore build output, IDE files, secrets, dependencies (`build/`, `.idea/`, `.env`, `node_modules/`). Note ignoring only affects **untracked** files — something already committed must be removed with `git rm --cached`.

**A committed secret is compromised**, even if you delete it in the next commit — it lives in history forever, and history is usually already pushed. Rotate the credential; don't just remove it. (`git filter-repo` or BFG can scrub history, but rewriting shared history is disruptive and the secret should be assumed leaked regardless.)

---

# Part 10 — Branching strategies and good practice

## 10.1 The strategies

| Strategy | Shape | Suits |
|---|---|---|
| **Trunk-based** | everyone commits to `main`; very short-lived branches; feature flags for incomplete work | CI/CD, frequent deploys, high test confidence |
| **GitHub Flow** | `main` is always deployable; branch → PR → review → merge → deploy | most teams, most of the time |
| **Git Flow** | `main`, `develop`, `feature/*`, `release/*`, `hotfix/*` | versioned releases, scheduled shipping |
| **Fork and PR** | contributors fork, PR upstream | open source |

**GitHub Flow is the sensible default** for most product work, and trunk-based is where high-performing continuous-deployment teams end up. Git Flow was designed for a versioned-release world and is heavier than most web products need — it's frequently adopted out of habit rather than fit.

For Practiq as a solo project: `main` plus short-lived feature branches, merged via PR (even self-reviewed — a PR gives you a diff to read and a place to write context, both of which catch mistakes).

## 10.2 Keep branches short-lived

The single most effective practice. A branch alive for a day merges cleanly; one alive for three weeks diverges, conflicts, and becomes a merge you dread. If a feature is genuinely large, break it into deliverable pieces or hide it behind a **feature flag** and merge the incomplete work safely.

## 10.3 Good commits

**One logical change per commit.** Not one file, not one day's work — one change, so it can be reviewed, reverted or cherry-picked independently. `git add -p` (§2.2) is how you achieve this from a messy working directory.

**Good messages** follow the conventional shape:

```
Short summary in imperative mood, ~50 chars

Why this change was needed, and what approach was taken. The diff
already shows WHAT changed — the message should explain WHY, and
note anything non-obvious about the approach or alternatives rejected.

Refs: PRACTIQ-142
```

Imperative mood ("Add pagination", not "Added" or "Adds") because it completes the sentence "this commit will…". A blank line after the summary is required for tooling to distinguish subject from body.

The reason to bother: **`git blame` plus a good message is documentation that can't drift** (№40 §4.2). When someone finds a strange line in eighteen months, the commit message is their only source of the reasoning.

Many teams use **Conventional Commits** (`feat:`, `fix:`, `refactor:`, `docs:`, `chore:`) which enables automated changelogs and semantic versioning — worth adopting if you want that automation.

## 10.4 Good pull requests

Small and single-purpose (large PRs get rubber-stamped, not reviewed — №40 §10). Explain context in the description: what, why, how to test. **Separate refactoring commits from behaviour commits** so the reviewer can read them independently. Self-review the diff before requesting review — you'll catch the debug statement.

---

# Part 11 — When to use what

**A. Merge or rebase?** Tell → rebase: your own unpushed branch, tidying before sharing. Tell → merge: integrating shared branches, or anything already pushed. Default: **rebase private work, merge public work.**

**B. Reset or revert?** Tell → reset: the commits are local and unpushed. Tell → revert: they're pushed and shared. Default: **revert anything public — never rewrite shared history.**

**C. Which reset mode?** Tell → `--soft`: keep everything, recommit differently. Tell → `--mixed`: keep the changes, re-stage selectively. Tell → `--hard`: genuinely discard. Default: **`--soft` unless you mean to destroy work.**

**D. Stash or WIP commit?** Tell → stash: a brief interruption, minutes. Tell → WIP commit on a branch: anything longer, or anything you'd hate to lose. Default: **commit — it's recoverable via reflog; stashes are easy to forget.**

**E. Squash or keep the commits?** Tell → squash: the intermediate commits are noise ("wip", "fix typo"). Tell → keep: each commit is a meaningful, independently-revertable step. Default: **tidy with interactive rebase so the surviving commits each mean something.**

**F. Branch or feature flag?** Tell → branch: the work lands within a few days. Tell → flag: it's larger, or you want to merge incomplete work safely. Default: **short branches; flags for anything spanning weeks.**

**G. Force-push?** Tell → yes: your own PR branch after a rebase, using `--force-with-lease`. Tell → never: a shared branch like `main`. Default: **`--force-with-lease`, never plain `--force`.**

---

# How to expand this

- *Related:* №44 §9 (bisect as a debugging method); №40 §10 and §9.1 (good PRs, separating refactoring from behaviour commits); №56 CI/CD (planned — branches and PRs as pipeline triggers).
- *Candidates for deeper treatment:* **Git internals properly** (the object database, packfiles, refs, how a merge algorithm actually works); **recovering from disasters** (a fuller playbook — corrupted repos, botched filter-repo, lost stashes); **monorepo workflows** (sparse checkout, submodules vs subtrees, and why submodules disappoint).

*Stable tooling, written from knowledge — the object model and command semantics don't drift. `switch`/`restore` are the modern replacements for the overloaded `checkout`; both remain available.*
