# Git & Version Control

> **DevOps – Lecture 4 notes.** Anything marked **➕ Extra** was not covered in class. It's there because it fills a gap or comes up often in labs and interviews.
>
> **Builds on:** [Lecture 1 – Linux](01-Linux-Permissions-and-Commands.md) (`vi`, terminal basics) and [Lecture 3 – Networking](03-Networking.md) (HTTPS vs. SSH, port 22).
>
> **Practice:** [Foundations MCQs](Foundations-MCQs.md).

---

## Table of Contents
1. [The Big Picture: Why Git?](#1-the-big-picture-why-git)
2. [The Three Areas of Git](#2-the-three-areas-of-git)
3. [Starting a Repository: `init` & `clone`](#3-starting-a-repository-init--clone)
4. [`git status`](#4-git-status)
5. [Staging & Committing](#5-staging--committing)
6. [Branching](#6-branching)
7. [Merging & Merge Conflicts](#7-merging--merge-conflicts)
8. [Undoing Changes: `reset` & `restore`](#8-undoing-changes-reset--restore)
9. [Cherry-Picking](#9-cherry-picking)
10. [Rebasing](#10-rebasing)
11. [Remotes: `fetch`, `pull`, `push`](#11-remotes-fetch-pull-push)
12. [Best Practices & `.gitignore`](#12-best-practices--gitignore)
13. [Team Workflow: Feature Branches & Pull Requests](#13-team-workflow-feature-branches--pull-requests)
14. [Quick Revision Cheat Sheet](#14-quick-revision-cheat-sheet)
15. [Likely Exam / Interview Questions](#15-likely-exam--interview-questions)

---

## 1. The Big Picture: Why Git?

**Version control** records every change to your files, so you can see **who changed what, when and why**, and go back to any earlier version.

**Git** is a **distributed** version control system: every developer has the **full history** on their own machine. **GitHub** (or GitLab, Bitbucket) is a website that **hosts** Git repositories online so a team can share them.

> 💡 **Simple words mein:** Git = tumhare laptop pe history rakhne wala tool. GitHub = us history ko online share karne ki jagah.

**Why DevOps cares:** all code, scripts, Dockerfiles and infra config (Terraform, Kubernetes YAML) live in Git. **CI/CD pipelines start when someone pushes to Git.**

---

## 2. The Three Areas of Git

Every change moves through three places:

```
 Working Directory  ──git add──▶  Staging Area  ──git commit──▶  Local Repo (.git)  ──git push──▶  Remote (GitHub)
  (files you edit)                (next commit)                  (saved history)                   (shared copy)
        ◀──────────── git restore ──────┘                                     ◀──── git fetch / pull ───┘
```

| Area | What it holds |
|------|---------------|
| **Working directory** | The files you're editing right now |
| **Staging area** (index) | Changes you've picked for the **next** commit |
| **Local repository** | All commits, stored in the hidden `.git` folder |
| **Remote** | The shared copy on GitHub |

> 💡 **Simple words mein:** staging = cart mein daalna, commit = order place karna, push = parcel bhej dena.

---

## 3. Starting a Repository: `init` & `clone`

### `git init`: start a new repo
```bash
mkdir my-project && cd my-project
git init                  # creates a hidden .git folder
ls -a                     # . .. .git
```
- `.git` holds **all metadata**: commits, branches, config. **Delete it and the history is gone.**
- `git init` only makes a **local** repo. To put it on GitHub, create an empty repo on the website (or ➕ `gh repo create`), then link it:
```bash
git remote add origin git@github.com:<user>/my-project.git
git push -u origin main   # -u remembers origin/main as the default upstream
```

### `git clone`: copy an existing repo
```bash
git clone https://github.com/<user>/<repo>.git    # HTTPS
git clone git@github.com:<user>/<repo>.git        # SSH
```

| | HTTPS | SSH |
|-|-------|-----|
| Setup | Simplest, works anywhere | Needs an SSH key added to GitHub |
| Auth | Username + **personal access token** (not your password) | Key pair, no typing |
| Port | 443 | 22 |

**➕ Setting up SSH** (one time):
```bash
ssh-keygen -t ed25519 -C "you@example.com"   # creates ~/.ssh/id_ed25519(.pub)
cat ~/.ssh/id_ed25519.pub                     # paste into GitHub → Settings → SSH keys
ssh -T git@github.com                         # test: "Hi <user>! You've successfully authenticated"
```

**➕ First-time identity** (shows up on every commit):
```bash
git config --global user.name  "Your Name"
git config --global user.email "you@example.com"
```

---

## 4. `git status`

Shows the state of the **working directory and staging area**. Run it all the time.

```bash
$ git status
On branch main
Changes to be committed:            ← STAGED (green)
        modified:   app.py
Changes not staged for commit:      ← MODIFIED, not staged (red)
        modified:   README.md
Untracked files:                    ← NEW, Git isn't tracking these yet
        notes.txt
```

**➕ Related:**
```bash
git status -s          # short form: M = modified, A = added, ?? = untracked
git log --oneline      # one line per commit
git log --oneline --graph --all   # see branches as a tree
git diff               # unstaged changes      git diff --staged   # staged changes
```

---

## 5. Staging & Committing

### Staging: `git add`
```bash
git add app.py         # stage one file
git add .              # stage everything in the current folder (new + modified + deleted)
```
Moves changes from the **working directory → staging area**.

### Committing: `git commit`
```bash
git commit -m "Add login endpoint"     # save staged changes as a commit
git commit -am "Fix typo in header"    # add + commit in one step
```
> ⚠️ `-a` stages only **already-tracked** files. **New (untracked) files are skipped**, so run `git add` for them first.

Each commit gets a unique **commit ID (SHA hash)**, e.g. `a1b2c3d`. You use it for `reset`, `cherry-pick`, etc.

---

## 6. Branching

A **branch** is a separate line of work. You build a feature on its own branch without breaking `main`.

```bash
git branch                    # list branches (* = current)
git branch feature-login      # create only
git checkout feature-login    # switch
git checkout -b feature-login # create AND switch (what we used in class)
```

**➕ Modern equivalents** (clearer, Git 2.23+):
```bash
git switch feature-login       # switch
git switch -c feature-login    # create + switch
git branch -d feature-login    # delete after merging
```

> 💡 **Simple words mein:** branch ek pointer hai jo ek commit pe point karta hai. Naya commit karo, pointer aage badh jaata hai. Isliye branch banana sasta aur fast hai.

---

## 7. Merging & Merge Conflicts

### Merging
Go to the branch you want to update, then merge the other branch **into** it:
```bash
git checkout main
git merge feature-login
```

| Merge type | When | Result |
|------------|------|--------|
| **Fast-forward** | `main` has no new commits since the branch was made | `main` pointer just moves ahead, no extra commit |
| **3-way merge** | Both branches have new commits | Git creates a **merge commit** with two parents |

### Merge conflicts
A **conflict** happens when **both branches changed the same lines** of the same file. Git can't choose, so it asks you.

```
<<<<<<< HEAD
print("Hello from main")
=======
print("Hello from feature")
>>>>>>> feature-login
```
- Between `<<<<<<<` and `=======` → **your current branch** (HEAD)
- Between `=======` and `>>>>>>>` → **the branch being merged**

**Steps to resolve:**
```bash
git status                   # lists "both modified" files
vi app.py                    # keep what you want, DELETE all <<<<<<< ======= >>>>>>> lines
git add app.py               # mark as resolved
git commit                   # finish the merge
# ➕ changed your mind?  git merge --abort
```

---

## 8. Undoing Changes: `reset` & `restore`

### `git restore`: discard working-directory changes
```bash
git restore app.py             # throw away unstaged edits (back to last commit) ⚠️ lost for good
git restore --staged app.py    # unstage, but KEEP the edits in the file
```

### `git reset`: move the branch back / unstage
```bash
git reset app.py               # unstage a file (same as restore --staged)
git reset --soft  HEAD~1       # undo last commit, keep changes STAGED
git reset --mixed HEAD~1       # undo last commit, keep changes UNSTAGED (default)
git reset --hard  HEAD~1       # undo last commit AND delete the changes ⚠️
```

| Mode | Commit undone? | Staging | Working files |
|------|----------------|---------|---------------|
| `--soft` | ✅ | kept | kept |
| `--mixed` (default) | ✅ | cleared | kept |
| `--hard` | ✅ | cleared | **deleted** |

> `HEAD` = the commit you're on now. `HEAD~1` = one commit before it.
>
> **➕ Rule:** never `reset` commits that are **already pushed** and shared. Use **`git revert <commit_id>`** instead. It adds a new commit that undoes the old one, so nobody's history breaks.

---

## 9. Cherry-Picking

Copy **one specific commit** from another branch onto your current branch, **without merging the whole branch**.

```bash
git log --oneline feature-x    # find the commit ID, e.g. 7f3a9c1
git checkout main
git cherry-pick 7f3a9c1        # applies that commit's changes as a NEW commit on main
```
**Use case:** a bug fix was made on a feature branch, and production (`main`) needs just that fix right now.

If it conflicts: fix the file → `git add` → `git cherry-pick --continue` (or `--abort`).

---

## 10. Rebasing

**Rebase** takes your branch's commits and **replays them on top of** another branch's latest commit. The result is a **straight, linear history**.

```
Before:              A──B──C  main
                         \
                          D──E  feature

git checkout feature
git rebase main

After:               A──B──C  main
                            \
                             D'──E'  feature    (D', E' = same changes, new IDs)
```

| | `merge` | `rebase` |
|-|---------|----------|
| History | Keeps the real branching shape + a merge commit | Linear, no merge commit |
| Commit IDs | Unchanged | **Rewritten** (new IDs) |
| Safe on shared branches? | ✅ Yes | ❌ No |

> 💡 **Simple words mein:** rebase = "maan lo maine apna kaam main ke latest version ke baad hi shuru kiya tha." History seedhi line ban jaati hai.
>
> ⚠️ **Golden rule:** don't rebase commits that others have already pulled. Rebase your **own local** branch only.

If it conflicts: fix → `git add` → `git rebase --continue` (or `--abort`).

---

## 11. Remotes: `fetch`, `pull`, `push`

```bash
git remote -v                # list remotes (origin = where you cloned from)
git fetch origin             # DOWNLOAD new commits, don't touch your files
git pull origin main         # fetch + merge into your current branch
git push origin main         # UPLOAD your commits
```

| | `git fetch` | `git pull` |
|-|-------------|------------|
| Downloads new commits? | ✅ | ✅ |
| Changes your working files? | ❌ | ✅ (merges) |
| Safe to run anytime? | ✅ | Can cause conflicts |

**`pull` = `fetch` + `merge`.** ➕ `git pull --rebase` = `fetch` + `rebase`, for a cleaner history.

> ➕ **"Push rejected (non-fast-forward)"** means the remote has commits you don't have. Run `git pull` first, fix any conflicts, then push again.

---

## 12. Best Practices & `.gitignore`

### Good commit messages
- Short, present tense, says **why/what**: `Fix null check in login handler` ✅ vs. `changes` / `final final 2` ❌
- **One logical change per commit.** Commit small and often.

### `.gitignore`
A file in the repo root that lists what Git should **never track**:
```gitignore
node_modules/
__pycache__/
*.log
.env              # secrets, API keys ⚠️
.DS_Store
dist/
```
- Only works for **untracked** files. If a file is already committed, run `git rm --cached <file>` first.
- ➕ Ready-made templates: [github.com/github/gitignore](https://github.com/github/gitignore).

> ⚠️ **Never commit secrets** (passwords, AWS keys, `.env`). Even if you delete them later, they stay in the history.

---

## 13. Team Workflow: Feature Branches & Pull Requests

```bash
git checkout main && git pull           # 1. start from latest main
git checkout -b feature-signup          # 2. new branch for your work
# ...edit...
git add . && git commit -m "Add signup form"   # 3. commit
git push -u origin feature-signup       # 4. push the branch
# 5. On GitHub: open a Pull Request (PR) → teammates review → CI runs tests
# 6. PR approved → merged into main → delete the branch
```

**Pull Request (PR)** = a request to merge your branch into `main`. It's where **code review** and **automated checks** happen before anything reaches `main`.

> **DevOps tip:** protect `main` on GitHub (Settings → Branches) so nobody can push to it directly. Changes must come through a reviewed PR.

---

## 14. Quick Revision Cheat Sheet

```
AREAS       working dir ─add─▶ staging ─commit─▶ local repo ─push─▶ remote
START       git init  (.git folder)          git clone <https | ssh url>
CHECK       git status   git status -s   git log --oneline --graph --all   git diff [--staged]
SAVE        git add <file> / .              git commit -m "msg"    git commit -am "msg" (tracked only)
BRANCH      git branch    git checkout -b x / git switch -c x    git checkout x / git switch x
MERGE       git checkout main; git merge x   fast-forward vs merge commit
CONFLICT    <<<<<<< HEAD (yours) ======= (theirs) >>>>>>> → edit → git add → git commit
UNDO        git restore f (discard)   git restore --staged f (unstage)
            reset --soft (keep staged) / --mixed (keep unstaged) / --hard (delete) HEAD~1
            pushed already? → git revert <id>
CHERRY      git cherry-pick <commit_id>      one commit, new ID
REBASE      git rebase main  → linear history, rewrites IDs, never on shared branches
REMOTE      fetch = download only    pull = fetch + merge    push = upload
IGNORE      .gitignore → node_modules/ *.log .env    already tracked? git rm --cached f
FLOW        branch → commit → push → PR → review + CI → merge
```

---

## 15. Likely Exam / Interview Questions

1. **What is the difference between Git and GitHub?**
   Git is the version control tool that runs locally. GitHub is a hosting service for Git repos, with PRs and collaboration features.
2. **What does `git init` create?** A hidden `.git` folder holding all history and config.
3. **Explain the working directory, staging area and repository.** See [section 2](#2-the-three-areas-of-git).
4. **HTTPS vs. SSH for cloning?** HTTPS is simpler and uses a token. SSH uses a key pair and needs no typing after setup.
5. **`git commit -am` vs. `git add . && git commit -m`?** `-am` skips untracked (new) files.
6. **When does a merge conflict happen and how do you fix it?** Both branches changed the same lines → edit the markers → `git add` → `git commit`.
7. **Fast-forward vs. 3-way merge?** Fast-forward just moves the pointer. A 3-way merge creates a merge commit.
8. **`git reset --soft` vs. `--mixed` vs. `--hard`?** See the table in [section 8](#8-undoing-changes-reset--restore).
9. **`reset` vs. `revert`?** `reset` rewrites history (local only). `revert` adds an undo commit (safe for pushed work).
10. **`restore` vs. `reset`?** `restore` works on files (discard or unstage). `reset` moves the branch to another commit.
11. **What is cherry-picking and when would you use it?** Applying one commit to another branch, e.g. a hotfix.
12. **Merge vs. rebase?** Merge keeps history as it happened. Rebase makes it linear but rewrites commit IDs.
13. **`git fetch` vs. `git pull`?** Fetch only downloads. Pull downloads and merges.
14. **Why use `.gitignore`?** To keep build output, dependencies, logs and secrets out of the repo.
15. **What is a pull request?** A request to merge a branch, used for code review and CI checks before merging into `main`.
