# Git

## Why Git? — The Problem It Solves

Every developer on a team is changing the same code at the same time. Without a system to track *who changed what, when, and why* — and to safely combine everyone's work — you get overwritten files, lost history, and no way to undo a bad change. **Git** is a **distributed version control system**: every clone is a full copy of the project *and its entire history*, so you can work offline, branch freely, and never lose the ability to go back.

The word **distributed** matters: there's no single "the" copy. Your laptop has the whole repo; the remote (GitHub) is just the copy everyone agrees to sync through.

### The three areas

Git moves your changes through three places. Understanding these is the whole mental model:

```
Working directory   →   Staging area (index)   →   Local repo   →   Remote repo
   (your edits)          git add                    git commit       git push
```

| Area | What lives here | How code arrives |
|------|-----------------|------------------|
| **Working directory** | The files you're actually editing | You edit them |
| **Staging area** | The changes you've *selected* for the next commit | `git add` |
| **Local repository** | Committed history on your machine | `git commit` |
| **Remote repository** | The shared copy (GitHub) everyone syncs with | `git push` |

The staging area is the piece people miss: it lets you commit *some* of your changes and leave others out — you curate a commit before it's recorded.

## The Everyday Flow

```bash
git clone <url>            # copy a remote repo (+ its history) to your machine
# ... edit files ...
git add file.txt           # stage a change   (working dir → staging)
git add .                  # stage everything
git commit -m "message"    # record it        (staging → local repo)
git push                   # publish           (local repo → remote)
git pull                   # bring others' changes down (remote → working dir)
```

To wire an existing local folder to a remote for the first time you set the **origin** — the default name for "the remote I sync with":

```bash
git init                            # start tracking this folder
git remote add origin <url>         # name the remote 'origin'
git push -u origin main             # push and remember the link
```

`origin` is just a name (a bookmark for the URL); `main` is the branch. `-u` sets the upstream so later you can type plain `git push`/`git pull`.

## Branches

A **branch** is an independent line of work — a movable pointer to a commit. You never develop directly on the production line; you branch off, work in isolation, and merge back when it's ready.

- **`main` / `master`** — the **long-lived** branch that always points at **production**. What's on `main` is what's (or is about to be) live.
- **feature branch** — a **short-lived** branch you cut for one change. You develop and test it across environments, then merge it back and delete it.

```bash
git branch feature-egg-dosa       # create
git checkout feature-egg-dosa     # switch to it
git checkout -b feature-egg-dosa  # create + switch in one step
```

The discipline: *whatever change you want to make, you make it on a feature branch* — never straight on `main`.

## Merge vs Rebase

Both bring one branch's work into another. The difference is **history**.

| | **Merge** | **Rebase** |
|---|-----------|------------|
| Extra commit? | Yes — a **merge commit** with **two parents** | No extra commit |
| History | **Preserved** — you see the branch really happened | **Rewritten** — replays your commits on top, linear history |
| Shape | A visible fork-and-join | A straight line, as if you'd worked after `main` all along |
| Rewrites commits? | No | **Yes** — new commit hashes |

```bash
git merge feature-egg-dosa     # fold the branch in, keep the fork in history
git rebase main                # replay MY commits on top of latest main
```

**The rule that keeps you out of trouble:**

- Use **merge** when a branch is **shared** by multiple people — it's safe because it never rewrites history others already have.
- Use **rebase** only on **private/local** branches — it rewrites commits, and rewriting history someone else has pulled will corrupt their view.

## Git Conflicts

A **conflict** happens when merging and Git finds *different code on the same line* of the same file from two branches — it can't decide which wins, so it stops and asks you.

- It's the **developer's responsibility** — whoever wrote the conflicting code resolves it, because they know the intent. Git won't guess.
- Git marks the clashing region (`<<<<<<<`, `=======`, `>>>>>>>`); you edit the file to the correct result, then `git add` and commit.

**The habit that avoids painful conflicts:** *before* you push or raise a PR, **pull the latest `main` and merge it into your feature branch locally** — resolve any conflicts on your own machine, not in the shared PR. You fix the conflict once, quietly, instead of blocking the team.

```
eggdosa-suresh   ─┐
                  ├─ both edit the same line → conflict → the author resolves
eggdosa-ramesh   ─┘
```

## Branching Strategy — Feature Branching / GitHub Flow

The simplest workable team strategy, and the one taught here, is **Feature Branching** (a.k.a. **GitHub Flow**):

- **`main`** — long-lived, always deployable.
- **feature branch** — short-lived, one per change.

The loop: cut a feature branch → develop → open a **Pull Request** to merge into `main` → resolve conflicts → merge. From `main` you deploy through the environments — **DEV → UAT → PROD** — and once **PROD succeeds you tag** the exact commit with a version:

```bash
git tag v1.4.0
git push origin v1.4.0
```

A tag is a permanent, named bookmark on a commit — the thing you roll back *to* and the version you ship.

**One codebase, many environments.** The *same code* runs in every environment; only the **configuration** changes between them. You don't fork the app per environment — you promote the same artifact and swap config.

Larger teams extend this with more long-lived branches — `develop`, `release` — and more short-lived ones — `feature`, `hotfix` — but the principle is identical: long-lived integration branches, short-lived work branches merged in via PR.

## Undoing Things — Reset, Revert, Squash, Cherry-pick

Git gives you different undo tools depending on **whether the work is still local or already shared**. Getting this split right is the most common interview question.

### git reset — undo *local* commits

`reset` moves the branch pointer back, "un-committing" work. **Only for local branches** — it rewrites history, so never on anything you've pushed and shared. The mode decides where the un-committed code lands:

| Mode | Where your code goes |
|------|----------------------|
| **`--soft`** | Back to the **staging area** (kept, ready to re-commit) |
| **`--mixed`** *(default)* | Back to the **working directory** (kept, unstaged) |
| **`--hard`** | **Deleted** — gone entirely |

```bash
git reset --soft HEAD~1     # undo last commit, keep changes staged
git reset --mixed HEAD~1    # undo last commit, keep changes in working dir
git reset --hard HEAD~1     # undo last commit AND throw the changes away
```

### git squash — combine commits (interactive rebase)

A **feature** is really "the diff between the previous commit and the current one." While developing you make many small commits; before merging you often want them as **one clean commit**. **Squash** (via interactive rebase) collapses several commits into one:

```
1.0.0
1.1.0  →  "Egg Dosa"          # the feature = 1.1.0 - 1.0.0 = the diff of code
```

```bash
git rebase -i d4c10f5        # interactive, from the commit BEFORE the ones you're combining

pick   8bf896  Aloo masala added
squash 538e532 Oil added
squash 3972056 Dosa Batter added
# → save, then write ONE proper message for the combined commit
```

Same rule as rebase: **squash only private branches, never shared ones** — it rewrites history.

### git revert — undo on *shared* branches

When the bad code is **already on a shared/remote branch**, you can't rewrite history (others have it). `revert` is the safe undo: it **creates a new commit that reverses the changes**, leaving the original commit in place.

```bash
git revert <commit>     # new commit that cancels out <commit>
```

| | **reset** | **revert** |
|---|-----------|------------|
| Use on | **Local** branches | **Shared/remote** branches |
| History | **Rewritten** (commit removed) | **Preserved** (nothing rewritten) |
| How it undoes | Moves the pointer back | Adds a **new correcting commit** |

### git cherry-pick — grab one specific commit

If the code you need **already exists** as a commit on another branch, `cherry-pick` copies **just that one commit** onto your current branch — without merging the whole branch.

```
C1
C2   ← the one commit with the code you want

git cherry-pick C2      # C2's changes land as a new commit on your branch
```

### git stash — park work without committing

You're mid-change when an **emergency Production defect** lands and you must switch branches now — but your work isn't ready to commit. **Stash** shelves your uncommitted changes and gives you a clean working directory; you fix the emergency, come back, and restore them:

```bash
git stash              # shelve current changes, clean the working dir
git checkout hotfix    # go fix the emergency
# ... later, back on your branch ...
git stash pop          # bring the shelved changes back into the working dir
```

## Continuous Integration (CI) — Where Git Feeds the Pipeline

Git isn't the end goal — it's the source that automation builds on. **CI** is *continuously integrating* every developer's code into a tested, deployable **artifact**, automatically, through a series of **stages**:

```
clone  →  compile  →  scan  →  unit test cases   →  (artifact)
```

Each stage is a gate: pull the code, build it, scan it for vulnerabilities, run the tests — and only a green run produces an artifact worth deploying.

### Shift-left

**Shift-left** means moving quality checks — **scans and tests — earlier**, *before* deployment, rather than catching problems in production. The two orders contrast the idea:

```
1. Deploy, then scan and test          ← problems found late (expensive)
2. Scan and test, THEN deploy          ← shift-left: fail fast, cheap to fix
```

The artifact is then promoted through the environment chain, the same build carried forward with config swapped at each stop:

```
DEV → SIT → UAT → PERF → PRE-PROD → PROD
```

---

## Quick Reference

| Concept | One-liner |
|---------|-----------|
| **Git** | Distributed version control — every clone holds the full repo **and** its history |
| **Three areas** | Working dir → staging (`add`) → local repo (`commit`) → remote (`push`) |
| **Staging area** | The changes *selected* for the next commit — curate before recording |
| `git clone` / `git pull` | Copy a remote repo down / bring others' changes into your working dir |
| `git add` / `git commit` / `git push` | Stage / record locally / publish to the remote |
| **origin** | The default *name* for the remote you sync with (`git remote add origin <url>`) |
| `-u` / upstream | Links local↔remote branch so plain `git push`/`git pull` work |
| **Branch** | An independent, movable line of work |
| `main` / `master` | **Long-lived** branch that always points at **production** |
| **feature branch** | **Short-lived** branch for one change; merge back and delete |
| **Merge** | Folds a branch in with a **merge commit** (2 parents); **preserves** history |
| **Rebase** | Replays commits for a **linear** history; **rewrites** commits (new hashes) |
| Merge vs rebase rule | **Merge shared branches; rebase private/local branches** |
| **Conflict** | Same line changed on two branches — Git stops; the **author** resolves |
| Avoid conflict pain | **Pull `main` and merge locally** before pushing / raising a PR |
| **Feature Branching / GitHub Flow** | `main` (long-lived) + feature branches → PR → deploy → tag |
| **Tag** (`v1.4.0`) | Permanent named bookmark on a commit — what you ship / roll back to |
| One code, many envs | Same code everywhere; only **configuration** changes per environment |
| **git reset** | Undo **local** commits (rewrites history — never shared) |
| reset `--soft` / `--mixed` / `--hard` | Code → **staging** / **working dir** / **deleted** |
| **git squash** (`rebase -i`) | Combine many commits into one — **private branches only** |
| Feature = diff | A feature is the code diff from the previous commit to the current one |
| **git revert** | Undo on **shared** branches — adds a **new correcting commit**, keeps history |
| reset vs revert | reset = local, rewrites; revert = shared, preserves (new commit) |
| **git cherry-pick** | Copy **one specific commit** onto your branch |
| **git stash** / `stash pop` | Shelve uncommitted work to switch branches / restore it later |
| **CI** | Continuously integrate code into a tested artifact via stages |
| CI stages | `clone → compile → scan → unit test → artifact` |
| **Shift-left** | Move scans/tests **before** deployment — fail fast, cheap to fix |
| Environment chain | `DEV → SIT → UAT → PERF → PRE-PROD → PROD` |

---
