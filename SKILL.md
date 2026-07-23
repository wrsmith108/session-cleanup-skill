---
name: "Session Cleanup"
description: "End-of-session housekeeping for git repositories. Use when the user asks to 'end of session', 'clean up session', 'session housekeeping', 'clean up branches', 'clean up worktrees', or 'clean up remote branches'."
version: "1.5.0"
---

# Session Cleanup

Structured end-of-session housekeeping: doc review, branch audit, worktree removal, and remote sync.

## Behavioral Classification

**Type**: Guided Decision

**Directive**: ASK, THEN EXECUTE

Audits each category and presents findings before taking any destructive action. Deletions always follow an explicit audit phase.

## Execution Context Requirements

This skill spawns a general-purpose subagent that performs git operations and file edits.

**Foreground execution required**: Yes

**Required tools**: Read, Write, Edit, Bash, Grep, Glob

**Fallback**: If tools are denied, the subagent returns a checklist of recommended actions for the coordinator to apply manually.

## Usage

Invoke at the end of a working session:

```
/session-cleanup
```

Or trigger via phrases: "end of session", "clean up branches", "clean up worktrees", "session cleanup"

**Out of scope**: PR creation/merge (`/ship`), production deploys, Linear issue updates (`/linear`), resolving merge conflicts.

## Five Phases

1. **Doc Review** — CLAUDE.md staleness, MEMORY.md updates, MEMORY.md TTL pruning (class-4 entries past their expiry), submodule dirty state
2. **Branch Audit** — PR status check, unpushed commit detection, squash-merge artifact filtering
3. **Branch Cleanup** — Push unreleased content, cherry-pick docs to main, delete merged branches
4. **Worktree Cleanup** — Remove stale worktrees via project script or `git worktree remove`; audit for orphaned Docker volumes/images left by worktrees that disappeared outside the normal removal path (crash, manual `rm -rf`, bare `git worktree remove`), via a project-specific orphan-prune script if one exists
5. **Backlog Sweep** — Query Linear for issues in Backlog state with no activity in 30+ days. For each, check if the work was completed by a related commit (search git log for issue ID or title keywords). Present findings for user to mark Done or re-prioritize. Monthly cadence; also consider a scheduled remote trigger (`/schedule`) to enforce.
6. **Final State** — Sync main, confirm clean working tree, report summary

## Pre-Dispatch Check (REQUIRED — run before spawning agent)

Before launching the cleanup agent, run these two commands via Bash:

```bash
git status --porcelain
git branch --list | grep -v "^\* main$"
```

**If both return empty** (working tree clean AND only `main` branch exists locally), cleanup has almost certainly already run this session. Do NOT spawn the agent. Instead, tell the user:

> "The repo looks clean already — only `main`, nothing uncommitted. Cleanup appears to have already run this session. Run it again anyway?"

Only proceed to Dispatch if the user confirms, or if either check returns output (there is real work to do).

## Dispatch

```
Use the Task tool with subagent_type="general-purpose" and pass the full contents
of ~/.claude/skills/session-cleanup/agent-prompt.md as the prompt.
```

---

## Changelog

### v1.5.0 (2026-07-23)
- **Added**: Phase 4c — orphaned Docker resource sweep. The routine per-worktree-removal path (Phase 4b's project script, when it also tears down Docker resources) only fires when a worktree is removed *through* that script. Worktrees that disappear another way — crash, manual `rm -rf`, a bare `git worktree remove` — never trigger it, so orphaned per-worktree volumes/images silently accumulate with nothing to catch them. 4c looks for a project-specific orphan-prune script (if one exists), runs it in dry-run/report-only mode, cross-checks reported project slugs against Phase 4a's `git worktree list` output as a second safety check, and presents the count + reclaimable space to the user before deleting anything — same Guided Decision pattern as the rest of the skill. Silently skipped if the project has no such script. Found via a real backlog: 300 orphaned volumes/images (~49GB) had accumulated on one machine, none of it caught by the routine per-removal path because they predated it or were removed out-of-band.

### v1.4.1 (2026-07-22)
- **Fixed**: Phase 5 Reset Guard checked working-tree cleanliness only, not which branch was checked out — a clean-but-non-main checkout (e.g. a docs/feature branch with an open PR) would have its own tip discarded by `git reset --hard origin/main`. Guard now also confirms `git branch --show-current` is `main`, checking out `main` first if not, before any reset. Found via a real near-miss: the main checkout was on a branch with 5 unmerged commits backing an open PR when Phase 5 was about to run.

### v1.4.0 (2026-07-06)
- **Fixed**: Phase 2 Category-A gating no longer relies on two-dot diff file count for branches with a merged PR — that diff measures drift against *today's* main, not unreleased content, and misclassifies old-but-merged branches in high-merge-velocity repos. `gh pr` MERGED state is now authoritative for Category A; the two-dot diff is only used as a gate for no-PR/open-PR branches. Found and fixed after `chore/smi-5359-quarantine-deliverables` (1 real unmerged commit) showed ~400 files in a raw two-dot diff during a cleanup pass. (SMI-5554 session)

### v1.3.0 (2026-04-23)
- **Added**: Phase 1b now scans `MEMORY.md` for `TTL: YYYY-MM-DD` markers and surfaces entries whose date has passed for user-approved pruning. Pairs with the `memory-governance` skill's class-4 (project state with TTL) classification. (SMI-4432 Pass 4)

### v1.2.1 (2026-03-28)
- **Fixed**: Phase 3b remote branch deletion now batches all deletes into a single `git push origin --delete b1 b2 ... bN` invocation instead of a per-branch loop. Eliminates O(N) pre-push hook runs and avoids partial-deletion state from mid-loop hook failures. (SMI-3710)

### v1.2.0
- Initial structured release with five-phase workflow and pre-dispatch idempotency check
