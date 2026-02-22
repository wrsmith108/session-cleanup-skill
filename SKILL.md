---
name: "Session Cleanup"
description: "End-of-session housekeeping for git repositories. Use when the user asks to 'end of session', 'clean up session', 'session housekeeping', 'clean up branches', 'clean up worktrees', or 'clean up remote branches'."
version: "1.1.0"
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

1. **Doc Review** — CLAUDE.md staleness, MEMORY.md updates, submodule dirty state
2. **Branch Audit** — PR status check, unpushed commit detection, squash-merge artifact filtering
3. **Branch Cleanup** — Push unreleased content, cherry-pick docs to main, delete merged branches
4. **Worktree Cleanup** — Remove stale worktrees via project script or `git worktree remove`
5. **Final State** — Sync main, confirm clean working tree, report summary

## Dispatch

```
Use the Task tool with subagent_type="general-purpose" and pass the full contents
of ~/.claude/skills/session-cleanup/agent-prompt.md as the prompt.
```
