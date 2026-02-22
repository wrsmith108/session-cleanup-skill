# Session Cleanup Agent

You are executing a structured end-of-session housekeeping workflow for a git repository. Work through all five phases in order.

---

## Safety Rules (Read First)

1. **Never delete** a branch without first checking `gh pr list --state all` for its PR status
2. **Confirm before Phase 3** — always show the user the categorized branch list and get explicit approval before deleting anything
3. **Cherry-pick requires consent** — show the user the file list and ask before cherry-picking any commit to main
4. **Never force-push main** — cherry-pick individual commits instead
5. **Conflict recovery** — if `git cherry-pick --no-commit` exits non-zero, immediately run `git cherry-pick --abort`, report the conflict, and do not delete the source branch
6. **Reset guard** — before `git reset --hard origin/main`, confirm working tree is clean; abort if not
7. **`--no-verify` requires user consent** — never bypass hooks automatically; show the hook error, explain why, and ask the user to confirm before using `--no-verify`
8. **Two-dot diff is the gate** — `git diff main <branch>` (two-dot, content) is what determines if a branch has unreleased content; the three-dot log is only for finding commit hashes to cherry-pick
9. **Abort gracefully** — if the user says "stop" or "cancel" at any point, report what was completed, list what was skipped, and exit

---

## Pre-Flight Checks

Before starting any phase, verify the environment:

```bash
# 1. Confirm we are in a git repository
git rev-parse --is-inside-work-tree || { echo "Not a git repo. Aborting."; exit 1; }

# 2. Check for a remote
git remote -v
```

If no remote is configured: skip Phases 2, 3, and the remote-sync in Phase 5. Note this in the summary.

```bash
# 3. Check gh CLI is installed and authenticated (required for Phase 2)
gh auth status
```

If `gh` is not installed or not authenticated: halt and tell the user. Offer two options:
- A: Authenticate with `gh auth login` and then re-run
- B: Skip PR status checks and proceed in manual mode (user must confirm each deletion manually)

```bash
# 4. Confirm working tree is clean before starting
git status --porcelain
```

If output is non-empty: report the uncommitted changes and ask the user if they want to commit, stash, or abort before cleanup begins.

---

## Phase 1: Doc Review

### 1a. CLAUDE.md
Scan for stale entries: deploy commands referencing deleted branches, monitoring jobs that no longer exist, or workflow names that have changed. Report findings; do not auto-edit unless clearly wrong.

### 1b. MEMORY.md
Check `~/.claude/projects/<project>/memory/MEMORY.md` if it exists. Identify anything from this session worth persisting. Add entries following the existing format — semantic groupings, not chronological logs.

### 1c. Submodule dirty state

```bash
git submodule status
```

If any submodule shows as `-dirty`:

```bash
# Check if the submodule has a remote configured
git -C <submodule-path> remote -v
```

If no remote: commit locally and note "Submodule committed locally — no remote configured, push skipped."

If remote exists:
```bash
# Step 1: Commit inside the submodule
git -C <submodule-path> status --short
git -C <submodule-path> add -A
git -C <submodule-path> commit -m "docs: [describe what changed]"
git -C <submodule-path> push

# Step 2: Update pointer in main repo
git add <submodule-path>
git commit -m "chore: update <submodule-name> submodule pointer"
```

If Phase 1c produced a new commit on main, push it before Phase 2.

**Phase 1 summary line** (always emit, even if nothing was done):
- `[Phase 1 — Docs] All clean, nothing to do.` OR describe what changed.

---

## Phase 2: Branch Audit

### 2a. List all local branches

```bash
git branch -vv
```

For each non-main branch, note tracking status and last commit message.

### 2b. Check PR status for each branch

```bash
gh pr list --state all --json number,title,state,headRefName,mergedAt \
  --jq '.[] | select(.headRefName == "<branch>") | [.number, .state, (.mergedAt // "open")] | @tsv'
```

### 2c. Detect real content differences (two-dot diff)

For every branch with ahead commits, run both checks:

```bash
# Three-dot: find commit hashes not yet in main (for cherry-pick targeting only)
git log main..<branch> --oneline

# Two-dot: check ACTUAL content difference today (the gating check)
git diff main <branch> --name-only
```

- If two-dot diff shows **0 files**: squash-merge artifact — content is identical to main, safe to delete.
- If two-dot diff shows **files**: genuine unreleased content — categorize as B.

### 2d. Categorize each branch

| Category | Criteria |
|----------|----------|
| **A: Safe to delete** | PR merged AND two-dot diff shows 0 content difference |
| **B: Needs attention** | Has genuine content differences from main |
| **C: In progress** | No merged PR |

### 2e. For Category B — classify content type

```bash
git diff main <branch> --name-only
```

Classify as:
- **Docs-only**: only `.md`, `.txt`, `.claude/`, `docs/` files changed
- **Code**: any `.ts`, `.js`, `.py`, source files changed
- **Mixed**: both docs and code

### 2f. Confirmation gate (REQUIRED before Phase 3)

Present the full categorized list to the user and ask:

> "Here is what I found:
>
> **Category A — Safe to delete (merged, no unreleased content)**:
> - `branch-a` (PR #N merged)
> - `branch-b` (PR #N merged)
>
> **Category B — Has unreleased content**:
> - `branch-c` (docs-only: 2 files)
> - `branch-d` (mixed: code + docs)
>
> **Category C — In progress (keeping)**:
> - `branch-e`
>
> Proceed with cleanup?"

Use `AskUserQuestion` to get approval before continuing. If the user declines, report the audit findings and exit.

---

## Phase 3: Branch Cleanup

Only execute after explicit user approval from Phase 2f.

### 3a. Handle Category B branches first

For each branch with genuine content differences:

**Docs-only changes**: Show the user the file list and the relevant commit message, then ask:

> "Cherry-pick these doc changes to main? [yes / no / show diff]"

If yes:
```bash
# Find the specific commit(s) with the doc changes
git log main..<branch> --oneline

# Cherry-pick without committing first
git cherry-pick <commit-hash> --no-commit
```

**If cherry-pick fails** (exit code non-zero):
```bash
git cherry-pick --abort
```
Report the conflict to the user. Do NOT delete this branch. Ask the user to resolve manually.

If cherry-pick succeeds:
```bash
# Verify only expected files staged
git diff --cached --name-only

# Commit
git commit -m "<original message>"
```

**Code changes or mixed changes**: Push the branch and ask the user if they want to open a PR. Do not cherry-pick partial content from mixed branches.

### 3b. Delete Category A branches

Delete remote before local (so local delete serves as confirmation):

```bash
# Remote first — one at a time to handle partial failures cleanly
for branch in <branch1> <branch2> ...; do
  git push origin --delete "$branch" && echo "Remote deleted: $branch" || echo "Remote delete FAILED: $branch"
done

# Then local
git branch -D <branch1> <branch2> ...
```

If `git push origin --delete` fails for a branch (e.g. pre-push hook), stop and report the error to the user. Ask before using `--no-verify`. Always state the specific reason when bypassing hooks.

### 3c. Stale remote-only branches

```bash
git branch -r | grep -v "origin/HEAD\|origin/main"
```

Cross-check against `gh pr list --state merged` and delete confirmed-merged remote-only branches (same per-branch loop as 3b).

**Phase 3 summary line** (always emit): List deleted branches or "All clean, nothing to do."

---

## Phase 4: Worktree Cleanup

### 4a. List worktrees

```bash
git worktree list
```

Identify all non-main worktrees.

### 4b. Remove stale worktrees

Check for a project-specific worktree removal script (common names: `remove-worktree.sh`, `worktree-remove.sh`):

```bash
ls ./scripts/remove-worktree.sh 2>/dev/null || ls ./scripts/worktree-remove.sh 2>/dev/null && echo "script exists"
```

If a project script exists (preferred — often handles Docker containers, networks, and other cleanup the project needs):
```bash
# Run with whatever flags your project's script accepts
./scripts/remove-worktree.sh <worktree-path>
```

If no script: warn the user before proceeding:
> "No removal script found. Docker containers for this worktree will NOT be stopped automatically. Please run `docker compose --profile dev down` in `<worktree-path>` before I remove the worktree, or I can remove it and you handle Docker manually."

Then:
```bash
git worktree remove <worktree-path> --force
git worktree prune
```

Do not remove worktrees that show recent file modification times or have active processes.

**Phase 4 summary line** (always emit): List removed worktrees or "All clean, nothing to do."

---

## Phase 5: Final State

### 5a. Sync local main

First, confirm working tree is clean:

```bash
git status --porcelain
```

If output is non-empty: stop. Do not reset. Report what is uncommitted and ask the user how to proceed (commit, stash, or skip sync).

If clean:
```bash
git fetch origin --prune
git reset --hard origin/main
```

### 5b. Confirm clean state

```bash
git branch          # Should show only: * main
git worktree list   # Should show only main repo path
git status          # Should show: nothing to commit, working tree clean
```

### 5c. Report summary

Each phase must appear with one of three states: **acted**, **clean** (checked, nothing to do), or **skipped** (with reason).

```
## Session Cleanup Complete

**Phase 1 — Docs**
- CLAUDE.md: [checked — no stale entries / N issues found]
- MEMORY.md: [N entries added / checked — no changes needed]
- Submodules: [checked — all clean / committed N files to <name> / skipped — no remote]

**Phase 2–3 — Branches**
- Deleted (merged): <branch-a>, <branch-b>  [or: checked — no merged branches]
- Cherry-picked to main: [file] from <branch-c>  [or: skipped — user declined]
- Kept (in progress): <branch-d>  [or: none]
- Remote-only cleanup: [N deleted / checked — all clean]

**Phase 4 — Worktrees**
- Removed: <path-1>  [or: checked — no stale worktrees]
- Docker warning issued: [yes/no]

**Phase 5 — Sync**
- main: synced to origin/main @ <short-hash>  [or: skipped — uncommitted work present]
- Working tree: clean  [or: dirty — describe]
```

---

## Out of Scope

- Creating or merging PRs (use `/ship`)
- Deploying to production (use deploy commands in CLAUDE.md)
- Updating Linear issue status (use `/linear`)
- Resolving merge conflicts on active branches
