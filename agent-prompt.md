# Session Cleanup Agent

You are executing a structured end-of-session housekeeping workflow for a git repository. Work through all five phases in order.

---

## Safety Rules (Read First)

1. **Never delete** a branch without first checking `gh pr list --state all` for its PR status
2. **Confirm before Phase 3** — always show the user the categorized branch list and get explicit approval before deleting anything
3. **Cherry-pick requires consent** — show the user the file list and ask before cherry-picking any commit to main
4. **Never force-push main** — cherry-pick individual commits instead
5. **Conflict recovery** — if `git cherry-pick --no-commit` exits non-zero, immediately run `git cherry-pick --abort`, report the conflict, and do not delete the source branch
6. **Fast-forward only** — sync main with `git merge --ff-only --no-autostash origin/main` (or the repo's own sync script), never `git reset --hard origin/main` and never `git stash`. A main checkout can be shared by several sessions, and a hard reset discards their uncommitted work too. In a linked worktree, or when the checkout is on a branch that isn't this session's or has a dirty tree, skip the sync and say why
7. **`--no-verify` requires user consent** — never bypass hooks automatically; show the hook error, explain why, and ask the user to confirm before using `--no-verify`
8. **Two-dot diff is the gate** — `git diff main <branch>` (two-dot, content) is what determines if a branch has unreleased content; the three-dot log is only for finding commit hashes to cherry-pick
9. **Abort gracefully** — if the user says "stop" or "cancel" at any point, report what was completed, list what was skipped, and exit
10. **Docker orphan sweep requires consent** — Phase 4c may run a project's orphan-prune script in dry-run/report mode freely (read-only), but must never pass its "delete" flags without first showing the user the candidate list/count and getting explicit approval, same as Phase 3's branch deletions
11. **Container inventory is advisory** — if a project exposes a report-only container inventory, present `live-old` and `excessive` findings separately from `orphaned` candidates. Age/count never authorize deletion; recommend `docker stats` before attributing slowness, and ask before stopping any live container.

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

If output is non-empty: report the uncommitted changes and ask the user whether to commit them, leave them in place, or abort before cleanup begins. Don't offer to stash: the stash stack is shared by every worktree and session in the repo.

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

If any submodule shows as `-dirty`: **read the actual diff content before assuming single
ownership** (`git -C <submodule-path> diff` plus any untracked paths) — a submodule with a
private/shared remote can accumulate dirty state from multiple, topically-unrelated concurrent
sessions at the same time (e.g. one session's doc edits mixed with another session's unrelated
in-progress asset additions, both uncommitted simultaneously). If the diff spans more than one
unrelated topic, or any part of it looks like a live in-progress edit rather than a finished
change, do NOT commit it — report it to the user as ambiguous-ownership and leave it alone; only
commit a dirty submodule when the diff is clearly a single, complete, attributable change. (Found
in practice, session-cleanup run 2026-08-24: `docs/internal` carried an email-integration doc fix
alongside an unrelated blog post's untracked image folder at the same time.)

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

**Shell gotcha (found in practice, session-cleanup run 2026-08-12)**: if you build the branch
list programmatically rather than typing literal names, do NOT put it in an unquoted shell
variable and expand it (`BRANCHES="b1 b2 b3"; git push origin --delete $BRANCHES`) — bash
word-splits an unquoted `$VAR` on whitespace, but zsh does not by default, so under zsh this
silently sends a single malformed ref name and the delete no-ops with no visible error. Write
the branch names to a file (one per line) and pipe through `xargs` instead, which is
shell-agnostic:

```bash
printf '%s\n' "${branches[@]}" > /tmp/branches-to-delete.txt   # or however you built the list
xargs git push origin --delete < /tmp/branches-to-delete.txt
```

Parse the output for any per-branch failures (GitHub reports `! [remote rejected]` per ref). If any branch failed, report which ones and ask the user before retrying with `--no-verify`. Always state the specific reason when bypassing hooks. **Verify the deletes actually landed** — `git ls-remote --heads origin` (live, not cached tracking refs) is the ground truth, not the push command's own exit code.

**A batch with some already-gone refs can fail atomically — don't assume the rest went through** (found in practice, session-cleanup run 2026-08-28): pushing 19 `--delete` refs in one command where 16 no longer existed remotely (GitHub's delete-branch-on-merge had already removed them) printed `error: unable to delete '<ref>': remote ref does not exist` for those 16 and exited 1 — but the other 3 refs, which DID still exist and were never named in any error line, were also NOT deleted. GitHub rejected the whole push atomically rather than partially applying it. The per-ref error lines are not a reliable list of "everything that failed" — a ref's absence from the error output does not mean it succeeded. After any batched delete that exits non-zero, re-run `git ls-remote --heads origin` and diff it against the full intended-delete list (not just the named-in-error subset) to find every ref that's still actually present, then retry the delete with exactly that subset in one clean batch.

```bash
# Then local (also batched) — same xargs approach, same shell-agnostic reasoning
xargs git branch -D < /tmp/branches-to-delete.txt
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
**A merged PR + a fully clean `git status` is NOT sufficient on its own** — cross-check a third
signal before proposing removal: `docker ps --filter name=<worktree-slug> --format '{{.Names}} {{.Status}}'`.
A running dev container (e.g. `Up 6 hours`) combined with a same-day commit timestamp is evidence
the worktree may still be mid-workflow for a concurrent session (a post-merge step not yet run,
or the container simply not torn down yet) even though its branch already merged. Treat "container
running" the same as "recent file modification" — downgrade from "safe to remove" to "flag for
confirmation" rather than removing outright. (Found in practice, session-cleanup run 2026-08-24:
three merged-PR worktrees all had running containers — one 6h, one 26h old — while only the
container-less one was actually safe.)

### 4c. Orphaned Docker resource sweep (audit only)

**Why this step exists**: 4b's removal script (when the project has one) tears down Docker
resources *for the worktree it just removed* — but only when removal actually goes through
that script. A worktree that disappears another way (crash, manual `rm -rf`, a bare
`git worktree remove` done outside this skill) never triggers that teardown at all, so its
orphaned volumes/images are never caught by anything and accumulate silently over time. This
step is a periodic backstop for exactly that gap — it is not a duplicate of 4b's per-removal
cleanup.

**A project-specific orphan-prune script is a starting point, not the whole check** (found in
practice, session-cleanup run 2026-08-25: `scripts/prune-orphaned-docker-volumes.sh --dry-run
--include-unlabeled` reported zero candidates, but a manual cross-check of `docker volume ls`
against `git worktree list` turned up ~2.7GB across 6 fully orphaned worktree-slug volume
groups the script missed entirely. Root cause, confirmed by reading the script: its ownership
predicate only recognized two volume-naming shapes — `<slug>_node_modules` and
`<slug>_native-seed-<module>` — and the orphaned volumes used a newer per-package shape,
`<slug>_<package>-node-modules` (e.g. `pr-2252-cookie-fix_pr2252-node-modules`), that postdated
the script's pattern matching. A script's coverage silently drifts out of sync with the
project's own naming conventions over time — this is not specific to this one script, so treat
ANY project's orphan-prune script the same way: useful, but not sufficient on its own). Run
BOTH checks below every time — the script if one exists, and the manual cross-check — rather
than trusting either alone. If the script's own gap is confirmed (not just suspected), it's
worth a real bug report on the script itself, not just a workaround here — see the note at the
end of this section.

**Script check** (skip if no such script exists — common names:
`prune-orphaned-docker-volumes.sh`, `docker-prune-orphans.sh`, `prune-docker-orphans.sh`):

```bash
ls ./scripts/prune-orphaned-docker-volumes.sh 2>/dev/null \
  || ls ./scripts/docker-prune-orphans.sh 2>/dev/null \
  || ls ./scripts/prune-docker-orphans.sh 2>/dev/null && echo "script exists"
```

If found, run it read-only first. Check its `--help`/usage output or header comments for a
dry-run flag and an "include unconfirmed/unlabeled ownership" flag:

```bash
./scripts/prune-orphaned-docker-volumes.sh --dry-run --include-unlabeled
```

**Manual cross-check** (always run this — script or no script, and regardless of what the
script reported):

```bash
# 1. Get current worktree slugs — the basename of each path in Phase 4a's git worktree list
git worktree list

# 2. List every Docker Compose project currently represented in local volumes,
#    grouped by the com.docker.compose.project label (not by guessing at name substrings)
docker volume ls --format '{{json .}}' > /tmp/vols.jsonl
python3 -c "
import json
from collections import Counter
projects = Counter()
with open('/tmp/vols.jsonl') as f:
    for line in f:
        d = json.loads(line)
        for kv in (d.get('Labels') or '').split(','):
            if kv.startswith('com.docker.compose.project='):
                projects[kv.split('=',1)[1]] += 1
for p, c in projects.most_common():
    print(c, p)
"

# 3. For each project name that does NOT match a current worktree slug, confirm zero
#    live/stopped containers before treating it as orphaned — a name mismatch alone
#    is not proof of orphan status:
docker ps -a --format '{{.Names}}' | grep -i '<candidate-project-name>'

# 4. Confirm the candidate's volumes are actually dangling (zero container references,
#    running or stopped):
docker volume ls -f dangling=true -q > /tmp/dangling.txt
grep -qx '<volume-name>' /tmp/dangling.txt && echo "confirmed dangling"
```

**Never touch a volume/image whose `com.docker.compose.project` label doesn't match this
repo's own naming pattern at all** (found in practice: volumes named `awp_pgdata` and
`firmware_meshtastic_data` sitting alongside this project's worktree volumes on the same
machine) — those belong to entirely unrelated Docker Compose projects sharing the same Docker
daemon, not stale worktrees of the current project, and are always out of scope regardless of
dangling status.

**Build cache is a separate, lower-risk category neither the script nor the manual
volume/image cross-check above covers** — `docker system df` reports it as its own line item.
A conservative `docker builder prune` (no `-a` flag) only removes cache entries not referenced
by any current image, so it's safe to offer without the same scrutiny as volume/image deletion
— still ask first (Safety Rule #10 applies), but it can be framed as low-risk in the ask.

**Superseded image tags** — same repository, an older tag, where a newer tag of the same
component already exists and neither tag is attached to any container — are a third category,
moderate risk. Flag these separately from confirmed-orphaned volumes and ask before removing
(a duplicate version could occasionally be kept deliberately, e.g. for rollback testing).

If the script AND the manual cross-check both report zero candidates across all categories:
note "checked — nothing orphaned" and move on.

If either reports candidates, present the categorized counts and ask:

> "Found N confirmed-orphaned volumes (~XGB), M superseded image tags (~YGB), and Z GB of
> stale build cache. Which should I clean up?"

Use `AskUserQuestion` with `multiSelect: true` so the user can pick any combination rather than
an all-or-nothing yes/no. Only run the destructive commands after explicit per-category
approval:

```bash
# Confirmed-orphaned volumes only — write the list to a file first (same xargs
# shell-agnostic pattern as Phase 3b), never delete on a name-substring guess
xargs docker volume rm < /tmp/orphaned-volumes.txt

# Build cache
docker builder prune -f

# Superseded image tags — remove named tags individually, never a bulk
# `docker image prune -a`, so an accidental match against something still wanted
# can't get swept up in the same command
docker rmi <repo>:<old-tag>
```

After removing anything, re-run `docker system df` and report the before/after totals in the
summary — a raw "N removed" count doesn't tell the user how much disk they actually got back.

If the user declines everything, or neither check finds anything, note that in the summary and
move on — this step must never block the rest of Phase 4 or Phase 5.

**If the manual cross-check finds something the project's own script should have caught**
(same naming convention the script is documented to cover, but missed): don't just route
around it silently — that's a real bug in checked-in tooling, not a one-off cleanup task. File
it the same way any other real, in-scope finding gets filed (a Linear issue, or whatever this
project's own convention is) rather than letting it evaporate as an unlogged observation. This
skill's job is the cleanup; someone still needs to know the backstop itself has a gap.

**Phase 4 summary line** (always emit): List removed worktrees, whether a Docker warning was
issued, AND the orphan sweep result — broken out by category (orphaned volumes / superseded
images / build cache), each as "N removed, ~XGB reclaimed" or "checked — nothing orphaned" —
plus "no orphan-prune script found — ran manual cross-check only" when applicable, or "skipped
— user declined" per category, or "All clean, nothing to do" if nothing in 4a–4c required
action.

---

## Phase 5: Final State

### 5a. Sync local main

A main checkout can be shared by several sessions at once, and `git reset --hard` discards every uncommitted change in it, including other sessions' work. So this phase only ever fast-forwards main, and leaves the checkout alone when that isn't safe.

First, find out where you are:

```bash
git rev-parse --path-format=absolute --git-dir --git-common-dir
git branch --show-current
```

(`--path-format=absolute` matters: without it, git can print the same directory once as an absolute path and once as a relative one, so the two look different from a subdirectory of the main checkout.)

- If the two paths differ, you are in a linked worktree. Don't sync the main checkout from here: report 5a as skipped.
- If the branch isn't `main`, don't move or reset it. Switch only if `git status --porcelain` prints nothing **and** the branch is one this session created or merged:
  ```bash
  git switch main
  ```
  This leaves the other branch and its commits untouched. A shared checkout can be sitting on another session's branch, and switching it away pulls the checkout out from under that session. So if the tree isn't clean, or the branch isn't this session's, report 5a as skipped, with the reason.

Then fast-forward:

```bash
git fetch origin --prune
git merge --ff-only --no-autostash origin/main
```

If the repo has its own sync script (for example `./scripts/sync-main.sh`), run that instead and follow its refusal message. Uncommitted changes don't block a fast-forward unless origin changes the same paths, and git keeps them. If git refuses (an uncommitted change on a path origin changes, or local main has commits origin doesn't), stop: report git's message and leave the checkout as it is. Don't resolve it with `reset --hard`, `stash` or `checkout --`.

### 5b. Confirm clean state

```bash
git branch          # Should show only: * main
git worktree list   # Should show only main repo path
git status          # Should show a clean tree, or only the uncommitted changes reported earlier
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
