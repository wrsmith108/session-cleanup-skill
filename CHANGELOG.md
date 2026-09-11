# Changelog

All notable changes to the Session Cleanup skill are documented here.

Format follows [Keep a Changelog](https://keepachangelog.com/en/1.0.0/).
Versions follow [Semantic Versioning](https://semver.org/).

---

## [1.7.1] — 2026-08-28

### Fixed
- **Batched remote branch deletion can fail atomically**: Phase 3b's `git push origin --delete` guidance now warns that a batch mixing already-gone remote refs (deleted by delete-branch-on-merge) with real ones can be rejected as a whole, with no refs actually deleted — even though the per-ref error output only names the already-gone refs, making the batch look partially successful. Found via a real cleanup run: a 19-branch batch where 16 refs no longer existed remotely printed 16 "remote ref does not exist" errors and exited 1; the other 3 refs, never named in any error, were also not deleted. Guidance now says to re-run `git ls-remote --heads origin` after any non-zero exit and retry with exactly the refs still actually present.

## [1.7.0] — 2026-08-25

### Added
- **Phase 4c no longer trusts a project's orphan-prune script alone**: it now always runs a manual cross-check too (`docker volume ls`'s `com.docker.compose.project` labels against `git worktree list`, plus a dangling-volume/live-container confirmation before calling anything orphaned). Also extends 4c's scope to two categories the script-only approach never covered: build-cache pruning (`docker builder prune`, low-risk, doesn't touch any current image) and superseded same-repository image tags (moderate-risk, ask first). Explicitly excludes volumes/images belonging to unrelated Docker Compose projects sharing the same daemon. Found via a real cleanup run: a project's own `prune-orphaned-docker-volumes.sh --dry-run --include-unlabeled` reported zero candidates, but the manual cross-check found ~2.7GB across 6 fully orphaned volume groups the script's ownership predicate didn't recognize (a newer per-package `<slug>_<package>-node-modules` naming shape postdated the script's pattern matching). Total reclaim across all three categories that run: ~37GB (orphaned volumes + build cache + superseded images).

## [1.6.0] — 2026-07-25

### Added
- **Container-inventory review**: project-provided report-only inventories now distinguish live-old/excessive advisory findings from orphaned cleanup candidates. The workflow recommends confirming resource pressure with `docker stats`; neither age nor count is deletion authority, and stopping live containers always requires consent.

## [1.5.0] — 2026-07-23

### Added
- **Phase 4c — orphaned Docker resource sweep**: the routine per-worktree-removal path (Phase 4b's project script, when it also tears down Docker resources) only fires when a worktree is removed *through* that script. Worktrees that disappear another way — crash, manual `rm -rf`, a bare `git worktree remove` — never trigger it, so orphaned per-worktree volumes/images silently accumulate with nothing to catch them. Phase 4c looks for a project-specific orphan-prune script (if one exists), runs it in dry-run/report-only mode, cross-checks reported project slugs against Phase 4a's `git worktree list` output as a second safety check, and presents the count and reclaimable space to the user before deleting anything — same Guided Decision pattern as the rest of the skill. Silently skipped if the project has no such script.

---

## [1.4.1] — 2026-07-22

### Fixed
- **Phase 5 Reset Guard checked working-tree cleanliness only, not which branch was checked out**: a clean-but-non-main checkout (e.g. a docs/feature branch with an open PR) would have its own tip discarded by `git reset --hard origin/main`. Guard now also confirms `git branch --show-current` is `main`, checking out `main` first if not, before any reset. Found via a real near-miss where the main checkout was on a branch with 5 unmerged commits backing an open PR when Phase 5 was about to run.

---

## [1.4.0] — 2026-07-06

### Fixed
- **Phase 2 Category-A gating no longer relies on two-dot diff file count for branches with a merged PR**: that diff measures drift against *today's* main, not unreleased content, and misclassifies old-but-merged branches in high-merge-velocity repos. `gh pr` MERGED state is now authoritative for Category A; the two-dot diff is only used as a gate for no-PR/open-PR branches.

---

## [1.3.0] — 2026-04-23

### Added
- **Phase 1b now scans `MEMORY.md` for `TTL: YYYY-MM-DD` markers** and surfaces entries whose date has passed for user-approved pruning, pairing with memory-governance-style class-4 (project state with TTL) classification.

---

## [1.2.1] — 2026-03-28

### Fixed
- **Remote branch deletion now batched** (SMI-3710): Phase 3b previously deleted remote branches one at a time in a loop, triggering a full pre-push hook run per branch (~3 min). All remote deletes are now issued in a single `git push origin --delete b1 b2 ... bN` invocation, running the hook once and eliminating mid-loop partial-deletion state on hook failures.

---

## [1.2.0] — 2026-03-01

### Fixed
- **Double-run bug** (#1): Skill would re-invoke the cleanup agent unconditionally even after cleanup had already completed in the same session, requiring the user to manually interrupt the Task tool call.

### Added
- **Pre-Dispatch Check** section in `SKILL.md`: before spawning the agent, the coordinator now runs `git status --porcelain` and `git branch --list | grep -v "^\* main$"`. If both return empty (clean working tree + only `main` branch), cleanup has almost certainly already run — Claude asks the user to confirm before proceeding rather than launching unconditionally.

---

## [1.1.0] — 2026-02-22

Initial published release.

### Added
- **Five-phase cleanup workflow**: Doc Review → Branch Audit → Branch Cleanup → Worktree Cleanup → Final State
- **Phase 1 — Doc Review**: CLAUDE.md staleness scan, MEMORY.md update prompts, dirty submodule detection and commit
- **Phase 2 — Branch Audit**: `gh pr list` status check for every non-main branch; two-dot diff (`git diff main <branch>`) as the content gate to correctly distinguish squash-merge artifacts from genuine unreleased content
- **Phase 3 — Branch Cleanup**: cherry-pick consent flow for docs-only branches, push prompt for code/mixed branches, per-branch remote delete loop before local delete
- **Phase 4 — Worktree Cleanup**: project-script detection (`scripts/remove-worktree.sh`), Docker warning when no script found, `git worktree prune` after removal
- **Phase 5 — Final State**: uncommitted-work guard before `git reset --hard origin/main`, structured summary report with acted/clean/skipped state per phase
- **Safety gates throughout**: explicit user approval required before Phase 3 deletions; cherry-pick conflict triggers `--abort` + branch preservation; `--no-verify` requires documented user consent; graceful abort on "stop"/"cancel"
- **`gh` CLI pre-flight check**: halts with recovery options if not installed or authenticated
- README published to `wrsmith108/session-cleanup-skill`
