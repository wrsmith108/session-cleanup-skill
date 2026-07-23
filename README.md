# session-cleanup

A Claude Code skill for structured end-of-session git housekeeping.

## What it does

Runs a five-phase cleanup at the end of a working session:

1. **Doc Review** — Checks CLAUDE.md for stale entries, MEMORY.md for anything worth persisting, and commits dirty submodule state
2. **Branch Audit** — Checks PR status for every non-main branch; uses two-dot diff to distinguish genuine unreleased content from squash-merge artifacts
3. **Branch Cleanup** — Pushes or cherry-picks unreleased content to main, then deletes merged branches locally and remotely
4. **Worktree Cleanup** — Removes stale worktrees via project script or `git worktree remove`; sweeps for orphaned Docker volumes/images left by worktrees removed outside the normal path (crash, manual removal), via a project-specific prune script if one exists
5. **Final State** — Syncs local main to origin, confirms clean working tree, reports summary

## Install

```bash
# Via Skillsmith MCP
# In Claude Code: "install session-cleanup skill"

# Or manually
git clone https://github.com/wrsmith108/session-cleanup-skill ~/.claude/skills/session-cleanup
```

## Usage

```
/session-cleanup
```

Or say: `"end of session"`, `"clean up branches"`, `"clean up worktrees"`, `"session cleanup"`

## Safety

- Never deletes a branch without checking PR status first
- Shows categorized branch list and requires explicit approval before any deletion
- Uses two-dot diff (`git diff main <branch>`) as the content gate — not three-dot — to correctly handle squash-merged branches
- Cherry-pick conflicts trigger abort + report; source branch is preserved
- `git reset --hard origin/main` guarded by uncommitted-work check
- `--no-verify` requires explicit user consent with documented reason
- Docker orphan sweep (Phase 4c) runs read-only by default; deletion requires showing the candidate list and getting explicit approval first
- Graceful abort at any phase if user says "stop"

## Requirements

- `git` — core operations
- `gh` CLI — PR status checks (pre-flight check verifies auth before Phase 2)
- Optional: `./scripts/remove-worktree.sh` — project-specific worktree cleanup with Docker support
- Optional: a project-specific Docker orphan-prune script (e.g. `./scripts/prune-orphaned-docker-volumes.sh`) — Phase 4c sweeps orphaned volumes/images if one exists

## Version

**v1.5.0** — See [CHANGELOG.md](CHANGELOG.md) for full release history. Latest: Phase 4c orphaned Docker resource sweep, catching worktrees removed outside the normal cleanup path (crash, manual removal).

## License

MIT
