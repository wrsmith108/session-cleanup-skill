# Session Cleanup Agent

You are executing a structured end-of-session housekeeping workflow for a git repository. Work through all five phases in order.

---

## Safety Rules (Read First)

1. **Never delete** a branch without first checking `gh pr list --state all` for its PR status
2. **Confirm before Phase 3** — always show the user the categorized branch list and get explicit approval before deleting anything
3. **Cherry-pick requires consent** — show the user the file list and ask before cherry-picking any commit to main
4. **Never force-push main** — cherry-pick individual commits instead
5. **Conflict recovery** — if `git cherry-pick --no-commit` exits non-zero, immediately run `git cherry-pick --abort`, report the conflict, and do not delete the source branch
6. **Reset guard** — before `git reset --hard origin/main`, confirm working tree is clean AND current branch is `main`; abort if either check fails (a clean-but-non-main checkout, e.g. a feature/docs branch with an open PR, must never be reset — checkout `main` first, don't reset the branch you're on)
7. **`--no-verify` requires user consent** — never bypass hooks automatically; show the hook error, explain why, and ask the user to confirm before using `--no-verify`
8. **Two-dot diff is the gate** — `git diff main <branch>` (two-dot, content) is what determines if a branch has unreleased content; the three-dot log is only for finding commit hashes to cherry-pick
9. **Abort gracefully** — if the user says "stop" or "cancel" at any point, report what was completed, list what was skipped, and exit
10. **Docker orphan sweep requires consent** — Phase 4c may run a project's orphan-prune script in dry-run/report mode freely (read-only), but must never pass its "delete" flags without first showing the user the candidate list/count and getting explicit approval, same as Phase 3's branch deletions

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

Check `~/.claude/projects/<project>/memory/MEMORY.md` if it exists. Two operations:

**1b-i. Persist new lessons**

Identify anything from this session worth persisting. Invoke the `memory-governance` skill to classify candidate entries before writing — it routes to the correct destination (MEMORY.md / topic file / new skill / decline). Do NOT write directly without classification.

**1b-ii. Prune expired TTL markers (SMI-4432 Pass 4)**

Class-4 memory entries (project state with expiry) carry a `TTL: YYYY-MM-DD` marker. After the date, the entry is noise and should be removed.

```bash
MEMORY_FILE=~/.claude/projects/<project>/memory/MEMORY.md
TODAY=$(date -u +%Y-%m-%d)

# Find all TTL markers and compare to today
grep -nE 'TTL: [0-9]{4}-[0-9]{2}-[0-9]{2}' "$MEMORY_FILE" | while IFS= read -r match; do
  line_no=$(echo "$match" | cut -d: -f1)
  ttl_date=$(echo "$match" | grep -oE 'TTL: [0-9]{4}-[0-9]{2}-[0-9]{2}' | cut -d' ' -f2)
  if [[ "$ttl_date" < "$TODAY" ]]; then
    echo "EXPIRED (TTL $ttl_date, today $TODAY) — line $line_no:"
    sed -n "${line_no}p" "$MEMORY_FILE"
  fi
done
```

If no expired entries: note "checked — no expired TTL entries" in the Phase 1 summary.

If expired entries found: present them to the user with line numbers and ask:

> "These MEMORY.md entries have expired (TTL date is past). Remove them?
>
> - Line N: <line content>
> - Line M: <line content>
>
> [yes / no / review each]"

**Never auto-remove**. Removal requires user confirmation — the session-cleanup skill is guided-decision, not automated. If confirmed, use the `Edit` tool to remove the matched lines. If the user says "review each", iterate one at a time.

**Also detect TTL marker format errors**:

- `TTL: 2026/04/23` (slashes not hyphens) — flag for user to fix
- `TTL 2026-04-23` (missing colon) — flag
- `TTL: 2026-4-23` (not zero-padded) — flag — the regex above won't match

If any format errors are found, surface them so the user can correct the markers (they won't prune reliably until fixed).

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

### 2c. Detect real content differences

**PR-merged status is the authoritative signal, not the two-dot diff — the diff alone is only
reliable for branches with no merged PR.** In a high-merge-velocity repo, a branch cut weeks ago
and merged long since will show a large two-dot diff against *today's* main purely from drift
(hundreds of unrelated commits from other PRs landing in between) — that is not unreleased
content, and gating on diff count alone will misclassify a genuinely-safe-to-delete branch as
Category B. (Found in practice, SMI-5554 cleanup: `chore/smi-5359-quarantine-deliverables` showed
~400 changed files in a raw two-dot diff despite having exactly one unmerged commit.)

```bash
# Authoritative: is there a merged PR for this branch?
gh pr list --state all --json number,title,state,headRefName,mergedAt \
  --jq '.[] | select(.headRefName == "<branch>") | [.number, .state, (.mergedAt // "open")] | @tsv'
```

- **PR state is MERGED**: the branch is safe to delete (Category A) regardless of two-dot diff
  size — do not compute or report the two-dot diff as a gating signal for this branch. If you
  need to double check for a stray uncherry-picked commit, use the three-dot commit log instead
  (see below), never the file-count diff.
- **PR state is OPEN, or no PR exists**: fall back to the two-dot diff as the gate:

```bash
# Three-dot: find commit hashes not yet in main (for cherry-pick targeting only)
git log main..<branch> --oneline

# Two-dot: check ACTUAL content difference today (the gating check — only meaningful here)
git diff main <branch> --name-only
```

  - Two-dot diff shows **0 files**: squash-merge artifact or no real changes — safe to delete.
  - Two-dot diff shows **files**: genuine unreleased content — categorize as B (open PR) or C (no PR).

### 2d. Categorize each branch

| Category | Criteria |
|----------|----------|
| **A: Safe to delete** | PR state is MERGED (two-dot diff not used as a gate for these) |
| **B: Needs attention** | No merged PR, and two-dot diff shows genuine content differences, PR open |
| **C: In progress** | No PR at all, two-dot diff shows genuine content differences |

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
# Remote first — batch all deletes into ONE push to run the pre-push hook once
# (SMI-3710: per-branch loop triggers a full hook run per branch)
git push origin --delete <branch1> <branch2> <branch3> ...
```

Parse the output for any per-branch failures (GitHub reports `! [remote rejected]` per ref). If any branch failed, report which ones and ask the user before retrying with `--no-verify`. Always state the specific reason when bypassing hooks.

```bash
# Then local (also batched)
git branch -D <branch1> <branch2> ...
```

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

### 4c. Orphaned Docker resource sweep (audit only)

**Why this step exists**: 4b's removal script (when the project has one) tears down Docker
resources *for the worktree it just removed* — but only when removal actually goes through
that script. A worktree that disappears another way (crash, manual `rm -rf`, a bare
`git worktree remove` done outside this skill) never triggers that teardown at all, so its
orphaned volumes/images are never caught by anything and accumulate silently over time. This
step is a periodic backstop for exactly that gap — it is not a duplicate of 4b's per-removal
cleanup.

Check for a project-specific Docker orphan-prune script (common names:
`prune-orphaned-docker-volumes.sh`, `docker-prune-orphans.sh`, `prune-docker-orphans.sh`):

```bash
ls ./scripts/prune-orphaned-docker-volumes.sh 2>/dev/null \
  || ls ./scripts/docker-prune-orphans.sh 2>/dev/null \
  || ls ./scripts/prune-docker-orphans.sh 2>/dev/null && echo "script exists"
```

**If no such script exists**: skip this step entirely — not every project has one. No need to
mention it in the summary.

**If a script exists**, run it read-only first. Check its `--help`/usage output or header
comments for a dry-run flag and an "include unconfirmed/unlabeled ownership" flag — most
scripts of this shape separate ownership-confirmed candidates (safe to auto-delete) from
unconfirmed ones (need a human look) behind flags like `--dry-run` / `--include-unlabeled`:

```bash
./scripts/prune-orphaned-docker-volumes.sh --dry-run --include-unlabeled
```

**Cross-check before presenting anything**: extract the project-name slug from each reported
candidate and confirm none of them match the worktree paths listed in Phase 4a's
`git worktree list` output. This is a second, independent check layered on top of whatever
safety predicate the project's own script already implements — not a replacement for it.

If the dry-run reports zero candidates: note "checked — nothing orphaned" and move on.

If it reports candidates, present the counts and ask:

> "Found N orphaned Docker volumes/images that don't match any current worktree (~X reclaimable
> disk space). Prune them?"

If the ownership-confirmed and unconfirmed counts differ meaningfully, offer the narrower scope
too — e.g. "prune only the confirmed-owned ones" vs. "include the unconfirmed-ownership ones as
well" — and let the user pick. Only run the destructive form after explicit approval:

```bash
./scripts/prune-orphaned-docker-volumes.sh --include-unlabeled   # or without the flag, per user's chosen scope
```

If the user declines, or the script doesn't expose these flags, note that in the summary and
move on — this step must never block the rest of Phase 4 or Phase 5.

**Phase 4 summary line** (always emit): List removed worktrees, whether a Docker warning was
issued, AND the orphan sweep result (resources removed + space reclaimed / "checked — nothing
orphaned" / "no orphan-prune script found" / "skipped — user declined"), or "All clean, nothing
to do" if nothing in 4a–4c required action.

---

## Phase 5: Final State

### 5a. Sync local main

First, confirm working tree is clean:

```bash
git status --porcelain
```

If output is non-empty: stop. Do not reset. Report what is uncommitted and ask the user how to proceed (commit, stash, or skip sync).

If clean, next confirm the checkout is actually on `main` — the main working directory can legitimately be sitting on any branch (a docs branch, a feature branch with an open PR, etc.), and `git reset --hard origin/main` moves *whatever branch is currently checked out* to match `origin/main`, discarding that branch's own unmerged commits even though the tree itself is clean:

```bash
git branch --show-current
```

If it prints anything other than `main`: check whether that branch has commits `origin/main` doesn't (`git log origin/main..HEAD --oneline`) and whether it backs an open PR (`gh pr list --head <branch> --state open`). If either is true, do NOT reset it — instead `git checkout main` (this leaves the other branch and its commits untouched), then re-run the clean-tree check above on `main` before proceeding. If the non-main branch has no unique commits and no open PR, it's safe to `git checkout main` the same way before resetting.

Only once the checkout is confirmed to be `main` AND clean:
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
- Orphan Docker sweep: [N resources removed, ~XGB reclaimed / checked — nothing orphaned / no orphan-prune script found / skipped — user declined]

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
