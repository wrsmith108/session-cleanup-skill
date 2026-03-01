# Changelog

All notable changes to the Session Cleanup skill are documented here.

Format follows [Keep a Changelog](https://keepachangelog.com/en/1.0.0/).
Versions follow [Semantic Versioning](https://semver.org/).

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
