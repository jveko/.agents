---
name: herdr-worktrunk
description: Use when orchestrating git-worktree coding lanes in Herdr: opening a nested worktree workspace, spawning an omp/codex/claude lane, delivering a mission brief, answering a blocked lane (question or approval dialog), landing with wt merge, resolving a lane rebase conflict, tearing down lane workspaces - also when herdr agent start fails with invalid_agent_argument or agent_pane_not_found.
---

# Herdr + Worktrunk

## Overview

**Core principle:** Worktrunk (`wt`) is the **single worktree engine** — create, warm, merge, remove. Herdr owns **panes, nested workspaces, and agent lifecycle** — never a worktree.

Never create worktrees *through* Herdr: its built-in worktree action stays deliberately unbound (the worktrunk plugin owns its keys), and that path skips `wt` post-start warm hooks, yielding cold trees that panic in `build.rs`.

**REQUIRED:** `herdr` skill (env gate, ids-from-JSON discipline) · `worktrunk` skill (hooks, approvals, `-C` rules).

## When to Use

You are the **supervisor** over many lane agents: each lane is a **lane orchestrator** running its own brief in a `wt` worktree (structure: brief-skeleton.md in this directory), and you create and warm the trees, open nested Herdr workspaces, spawn lanes, deliver briefs, answer blocked lanes, land merged branches, resolve lane rebase conflicts, and tear everything down.

**Not for:** creating worktrees through Herdr (that is `wt`-only — the Herdr path yields cold trees); headless one-shot runs (`-p/--print` has no lifecycle to supervise); solo work that needs no lane.

## Quick reference

| Step | Command |
|---|---|
| 0. Gate | `test "${HERDR_ENV:-}" = 1` before ANY herdr control call |
| 1. Create + warm | `wt switch --create <branch> --no-cd` — post-start hooks (copy-ignored, npm ci, artifact symlinks, dist stub) run in background; `--no-cd` keeps your pane's cwd put |
| 2. Nested workspace | `herdr worktree open --workspace "$HERDR_WORKSPACE_ID" --path <wt-path> --label <branch> --no-focus` → JSON with real ids (`wX:p1`) |
| 3. Spawn lane (bare) | `herdr agent start <name> --kind <omp\|claude\|codex> --pane wX:p1` — **no args after `--`, ever** |
| 4. Deliver brief | Write the brief to a file first (structure: **brief-skeleton.md** in this directory); then `herdr agent prompt <name> "FIRST read /tmp/<brief>.txt in full - it is your mission brief. Then execute it."` |
| 5. Supervise | Background `bash scripts/watch-lanes`; it exits printing lanes that are blocked/idle/done → `herdr agent read <id>` → answer via `herdr pane send-text <pane_id> "<answer>"` + `herdr pane send-keys <pane_id> enter` (NOT `agent prompt` — see below) → verify `agent get` flipped → re-arm |
| 6. Land | Supervisor only: `wt -C <wt-path> merge --no-squash --no-remove` — merges the CURRENT branch into TARGET (defaults to main; no branch selector). Both flags are load-bearing, see Landing below. Lanes NEVER merge or push |

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

## Landing, conflicts, teardown

**Land:** `wt -C <lane-path> merge --no-squash --no-remove` — pipeline = commit → (squash skipped) → **rebase onto target** → hooks → fast-forward target → cleanup (suppressed by `--no-remove`). Disjoint lanes rebase clean; the branch's own pre-merge gates already ran per-lane.

**Conflicts:** the rebase stops OPEN in the lane worktree. Resolve with the harness conflict device: `read` the conflicted file (registers `conflict://N`) → `write { path: "conflict://N", content: "@theirs"|"@ours"|<composed union> }` — pick a side for competing edits, compose the union when both intents apply (e.g. two lanes appended different clauses to one paragraph) → `git add <file>` → `GIT_EDITOR=true git rebase --continue`. A commit made obsolete by an already-merged lane → `git rebase --skip`.

**After the batch:** lanes' individual gates do NOT cover the combined tree. Run the repo's full gate on final main (e.g. `just build-ebpf` → `just ci` → `just test`). Expect two recurring misses: a lane that never ran the formatter ships fmt violations (run the repo's fmt + commit), and a stale primary `node_modules` fails the web check (`tsr: command not found` → `npm --prefix web ci`).

**Teardown:** `herdr workspace close <id>` per lane workspace (NEVER `--group` — it cascades to the primary) kills the lane processes/panes; then `wt remove <branch>` per worktree — the integrated-ancestor check auto-allows fully-merged branches (no `-D`), removal runs in the BACKGROUND (use `--foreground` to await), and `wt list` can race it: verify disk with `ls .worktrees/`. `--reap` kills detached processes holding the tree (TTY/interactive processes are spared); `-f` dirty, `-D` unmerged.

## Brief file pattern

Briefs are self-contained (scope, file:line evidence, pipeline) and stored OUTSIDE the worktree (e.g. `/tmp/<name>.txt`) so they never dirty `git status`. State whether repo MCP config is present in the worktree, and make the brief authoritative over any ticket text it summarizes. **For the structure itself, start from brief-skeleton.md in this directory** — role line, ordered scope queue with file:line evidence and deferred-items note, hard rules, questions-upward contract, a SKILLS line naming which skill each phase must load, the six-phase pipeline (brainstorm → parallel research → design+plan → cold-review gate → subagent implementation → final report), and env notes. The research phase deploys BOTH agent kinds: read-only **scouts** on the codebase AND a **librarian** on the real world (upstream docs, kernel/UAPI headers, crate and protocol sources — source-verified answers with citations), so external-contract claims are checked against reality before the design relies on them.

## Supervision loop

```bash
bash scripts/watch-lanes        # background job: exits printing the lanes that need you
bash scripts/watch-lanes --list # snapshot: every lane + status, no wait
```

The watcher polls every `WATCH_INTERVAL_S` (default 20s) across ALL workspaces and exits on the first lane that is `blocked` (approval/question dialog), `idle`, or `done` (`idle`/`done` both mean ready for input); `working` and `unknown` never fire — `unknown` covers detached/sleeping agents. Output is one tab-separated line per triggering lane: `id`, `pane_id`, `status` — `id` is the agent name when the lane has one, otherwise the pane id; either works as the target of any `herdr agent` command. No lane ids go in: the script discovers every lane in the session itself.

Run it as a backgrounded job; on wake: `herdr agent read <printed id>` the question, answer it as in step 5 (SELECT dialog → `pane send-keys <pane_id> enter` alone on the highlighted option; free-text → `pane send-text` + `enter`; `agent prompt` only works on a lane in a normal idle/done turn), then **verify `agent get` flipped `blocked → working`** before re-arming (restart the watcher). A still-blocked lane means the answer did not land — re-read, diagnose the dialog type, never re-send blindly. `blocked` means the lane raised a question or approval dialog — that is the supervision channel.
