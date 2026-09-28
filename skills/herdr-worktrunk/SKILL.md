---
name: herdr-worktrunk
description: Use when spawning or supervising coding-agent lanes that live in git worktrees - creating Worktrunk (wt) worktrees from Herdr, opening nested worktree workspaces, starting omp/codex/claude agents into worktree panes, delivering a long mission brief to a pane, writing a lane brief or mission prompt skeleton, answering a lane that went blocked, landing merged lane branches (wt merge removes the worktree), resolving a lane rebase conflict, tearing down lane workspaces/worktrees, or when herdr agent start fails with invalid_agent_argument or agent_pane_not_found.
---

# Herdr + Worktrunk

## Overview

Worktrunk (`wt`) is the **single worktree engine** — create, warm, merge, remove. Herdr owns **panes, nested workspaces, and agent lifecycle**. Herdr's built-in worktree action stays deliberately unbound (the worktrunk plugin owns its keys): never create worktrees *through* Herdr, because that path skips `wt` post-start warm hooks and yields cold trees that panic in `build.rs`.

**REQUIRED:** `herdr` skill (env gate, ids-from-JSON discipline) · `worktrunk` skill (hooks, approvals, `-C` rules).

## Quick reference

| Step | Command |
|---|---|
| 0. Gate | `test "${HERDR_ENV:-}" = 1` before ANY herdr control call |
| 1. Create + warm | `wt switch --create <branch> --no-cd` — post-start hooks (copy-ignored, npm ci, artifact symlinks, dist stub) run in background; `--no-cd` keeps your pane's cwd put |
| 2. Nested workspace | `herdr worktree open --workspace "$HERDR_WORKSPACE_ID" --path <wt-path> --label <branch> --no-focus` → JSON with real ids (`wX:p1`) |
| 3. Spawn lane (bare) | `herdr agent start <name> --kind <omp\|claude\|codex> --pane wX:p1` — **no args after `--`, ever** |
| 4. Deliver brief | Write the brief to a file first (structure: **brief-skeleton.md** in this directory); then `herdr agent prompt <name> "FIRST read /tmp/<brief>.txt in full - it is your mission brief. Then execute it."` |
| 5. Supervise | Poll `herdr agent list \| jq` for `agent_status=="blocked"` → `herdr agent read <name>` → answer via `herdr pane send-text <pane_id> "<answer>"` + `herdr pane send-keys <pane_id> enter` (NOT `agent prompt` — see below) → re-arm |
| 6. Land | Orchestrator only: `wt -C <wt-path> merge --no-squash --no-remove` — merges the CURRENT branch into TARGET (defaults to main; no branch selector). Both flags are load-bearing, see Landing below. Lanes NEVER merge or push |

## Common mistakes (observed failures)

| Mistake | Reality | Fix |
|---|---|---|
| Spawn lanes with `omp -p` or `agent start … -- -p "…"` | `-p/--print` = "Non-interactive mode: process prompt and exit" — headless one-shot, no TUI, no lifecycle states, nothing to supervise | bare `agent start`, then deliver work via `agent prompt` |
| Paste a long/multi-line brief into `agent start … --`, e.g. `-p "$(cat brief.txt)"` | double failure: `invalid_agent_argument: agent arguments cannot be encoded safely for the target shell`, AND `-p` makes it headless anyway | argv stays tiny and quote-free; the brief lives in a file, the prompt is a one-line pointer to it |
| Pass `wX` as the pane | `agent_pane_not_found` — workspace id ≠ pane id (`wX:p1`); greedy `${var%%:*}` slicing silently truncates it | take ids verbatim from creation JSON; never re-derive them |
| Answer a blocked question with `agent prompt` (or `agent send-text`) | prompt is REJECTED on a question/approval dialog (`agent_blocked`); `agent send-text` does not exist — text goes through the PANE surface | `agent read` first, then `herdr pane send-text <pane_id> "<answer>"` + `herdr pane send-keys <pane_id> enter`. Approval-type blocks are the user's decision — surface them, never auto-answer |
| Answer a SELECT-type dialog by typing an answer (observed: long `send-text` + `enter` pair silently did nothing, status stayed `blocked`) | ask dialogs come as radio-SELECT lists or free-text; select lists submit the HIGHLIGHTED option with Enter alone — typed text pollutes or vanishes | read the dialog first: SELECT → `pane send-keys <pane_id> enter` only; free-text → `pane send-text` + `enter`. ALWAYS confirm `agent get` flipped `blocked → working` before moving on; still blocked means nothing landed — re-read, never re-send blindly |
| Use `herdr worktree create` to make trees | bypasses `wt` hooks → cold worktree | creation is `wt`-only; `herdr worktree open` on EXISTING trees is fine |
| `wt merge --no-squash` without `--no-remove` | merge pipeline DELETES the worktree and branch after the target moves (worktree removal is the default, like the picker's post-merge cleanup) | always land with `--no-remove` until verification finishes; remove later with `wt remove` |
| Land while the primary checkout is dirty | merge refuses the local-main fast-forward: `Can't push to local main branch: conflicting uncommitted changes … Commit or stash … first` | commit or stash primary dirt FIRST, then merge |
| Approve hook commands yourself | approvals are a user trust decision | user runs `wt config approvals add`; never pass `--yes` for them |
| Rely on the UI-focused pane | focus belongs to the user or another client | explicit `--pane <id>` / `--current`; `--no-focus` for background lanes |

## Landing, conflicts, teardown (observed end-to-end 2026-09-25)

**Land:** `wt -C <lane-path> merge --no-squash --no-remove` — pipeline = commit → (squash skipped) → **rebase onto target** → hooks → fast-forward target → cleanup (suppressed by `--no-remove`). Disjoint lanes rebase clean; the branch's own pre-merge gates already ran per-lane.

**Conflicts:** the rebase stops OPEN in the lane worktree. Resolve with the harness conflict device: `read` the conflicted file (registers `conflict://N`) → `write { path: "conflict://N", content: "@theirs"|"@ours"|<composed union> }` — pick a side for competing edits, compose the union when both intents apply (e.g. two lanes appended different clauses to one paragraph) → `git add <file>` → `GIT_EDITOR=true git rebase --continue`. A commit made obsolete by an already-merged lane → `git rebase --skip`.

**After the batch:** lanes' individual gates do NOT cover the combined tree. Run the repo's full gate on final main (here: `just build-ebpf` → `just ci` → `just test`). Live catches: lanes that never ran fmt shipped violations (fixed via `just fmt` + commit), and a stale primary `web/node_modules` failed `web-check` with `tsr: command not found` → `npm --prefix web ci`.

**Teardown:** `herdr workspace close <id>` per lane workspace (NEVER `--group` — it cascades to the primary) kills the lane processes/panes; then `wt remove <branch>` per worktree — the integrated-ancestor check auto-allows fully-merged branches (no `-D`), removal runs in the BACKGROUND (use `--foreground` to await), and `wt list` can race it: verify disk with `ls .worktrees/`. `--reap` kills detached processes holding the tree (TTY/interactive processes are spared); `-f` dirty, `-D` unmerged.

## Brief file pattern

Briefs are self-contained (scope, file:line evidence, pipeline) and stored OUTSIDE the worktree (e.g. `/tmp/<name>.txt`) so they never dirty `git status`. State whether repo MCP config is present in the worktree, and make the brief authoritative over any ticket text it summarizes. **For the structure itself, start from brief-skeleton.md in this directory** — role line, ordered scope queue with file:line evidence and deferred-items note, hard rules, questions-upward contract, a SKILLS line naming which skill each phase must load, the six-phase pipeline (brainstorm → parallel research → design+plan → cold-review gate → subagent implementation → final report), and env notes. The research phase deploys BOTH agent kinds: read-only **scouts** on the codebase AND a **librarian** on the real world (upstream docs, kernel/UAPI headers, crate and protocol sources — source-verified answers with citations), so external-contract claims are checked against reality before the design relies on them.

## Supervision loop

```bash
while :; do
  blocked=$(herdr agent list | jq -r \
    '.result.agents[] | select((.name // "") != "") | select(.agent_status=="blocked") | .name')
  [ -n "$blocked" ] && { echo "BLOCKED: $blocked"; break; }
  sleep 20
done
```

Run it as a backgrounded job; on wake: `herdr agent read <name>` the question, answer it as in step 5 (SELECT dialog → `pane send-keys <pane_id> enter` alone on the highlighted option; free-text → `pane send-text` + `enter`; `agent prompt` only works on a lane in a normal idle/done turn), then **verify `agent get` flipped `blocked → working`** before re-arming. A still-blocked lane means the answer did not land — re-read, diagnose the dialog type, never re-send blindly. `blocked` means the lane raised a question or approval dialog — that is the supervision channel.
