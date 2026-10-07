---
name: herdr-worktrunk
description: Use when orchestrating git-worktree coding lanes in Herdr: opening a nested worktree workspace, spawning an omp/codex/claude lane, delivering a mission brief, answering a blocked lane (question or approval dialog), landing with wt merge, resolving a lane rebase conflict, tearing down lane workspaces, running a ticket backlog in waves (subagent triage, then batches of lanes), and when herdr agent start fails with invalid_agent_argument or agent_pane_not_found.
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
| 0b. Project extension | Repo root has `.config/sbx.toml` → lanes live in sandboxes: load the `herdr-worktrunk-sbx` skill — it overrides steps 1, 2, 6 and teardown |
| 1. Create + warm | `wt switch --create <branch> --no-cd` — post-start hooks (copy-ignored, npm ci, artifact symlinks, dist stub) run in background; `--no-cd` keeps your pane's cwd put |
| 2. Nested workspace | `herdr worktree open --cwd <repo-root> --branch <branch> --label <branch> --no-focus` → JSON with real ids (`wX:p1`). Prefer `--cwd`+`--branch`: `--path` requires the canonical path — herdr resolves `/tmp` → `/private/tmp`, anything else fails `worktree_not_found` |
| 3. Spawn lane (bare) | `herdr agent start <name> --kind <omp\|claude\|codex> --pane wX:p1` — **no args after `--`, ever** |
| 4. Deliver brief | Write the brief to a file first (structure: **brief-skeleton.md** in this directory); then `herdr agent prompt <name> "FIRST read /tmp/<brief>.txt in full - it is your mission brief. Then execute it."` |
| 5. Supervise | Background `bash scripts/watch-lanes`; it exits printing named lanes that are blocked/idle/done → `herdr agent read <id>` → answer via `herdr pane send-text <pane_id> "<answer>"` + `herdr pane send-keys <pane_id> enter` (NOT `agent prompt` — see below) → verify `agent get` flipped → re-arm, with `--except <id>` for each lane you are still handling (a done lane you're landing stays done) |
| 6. Land | Supervisor only: `wt -C <wt-path> merge --no-squash --no-remove` — merges the CURRENT branch into TARGET (defaults to main; no branch selector). Both flags are load-bearing, see Landing below. Lanes NEVER merge or push |

## Delegation

Pick the lane kind at spawn: `--kind omp` is the house default; use `claude`/`codex` when a task needs them. Names must match `[a-z][a-z0-9_-]{0,31}` and be unique among live agents. One lane per independent workstream — no two lanes may own the same files (the independence rules of dispatching-parallel-agents apply to spawning too).

The supervisor loop: `herdr agent start <name> --kind <kind> --pane <pane>` returns once the lane is ready → prompt it only while it is `idle` (`agent prompt` rejects a blocked dialog with `agent_blocked`) → deliver the one-line brief pointer → background `bash scripts/watch-lanes` → on each wake: read, answer, verify `blocked → working` → when every lane has reported done, land. You never work a lane's queue inline: the brief is the contract and the lane's ledger carries its state.

## Common mistakes (observed failures)

| Mistake | Reality | Fix |
|---|---|---|
| Spawn lanes with `omp -p` or `agent start … -- -p "…"` | `-p/--print` = "Non-interactive mode: process prompt and exit" — headless one-shot, no TUI, no lifecycle states, nothing to supervise | bare `agent start`, then deliver work via `agent prompt` |
| Paste a long/multi-line brief into `agent start … --`, e.g. `-p "$(cat brief.txt)"` | double failure: `invalid_agent_argument: agent arguments cannot be encoded safely for the target shell`, AND `-p` makes it headless anyway | argv stays tiny and quote-free; the brief lives in a file, the prompt is a one-line pointer to it |
| Widen a ticket in the brief ("and any sibling …"), or invent the fix a ticket defers to a source you can't read ("Fix: see report") | the brief is authoritative, so the lane does exactly what it says — observed: a LOW "unused constant" ticket became 66 deleted public constants | scope = the ticket's cited evidence and stated fix; when the fix lives in an unreadable source, scope to the cited lines and say so in the brief. Anything wider, or any removal of public API, is the user's call — ask before spawning |
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

Briefs are self-contained (scope, file:line evidence, pipeline) and stored OUTSIDE the worktree (e.g. `/tmp/<name>.txt`) so they never dirty `git status`. State whether repo MCP config is present in the worktree, and make the brief authoritative over any ticket text it summarizes. **For the structure itself, start from brief-skeleton.md in this directory** — role line, ordered scope queue with file:line evidence and deferred-items note, hard rules, questions-upward contract, a SKILLS line naming which skill each phase must load, the pipeline, and env notes. Tag each queue item with its track. **FULL** is the superpowers pipeline (parallel research → brainstorming → design+plan docs via writing-plans → cold-review gate → subagent-driven TDD) and the default for every change in behaviour. **LIGHT** (verify the cited lines → change → scoped gates → commit; no brainstorming, no docs, no review round) is only for changes that alter no behaviour: docs, comments, config, CI files, ignore rules, and deleting code proven unused that isn't public API. Changing what code returns or does, or flipping an existing test's expectation, is FULL — observed: a LIGHT "alignment predicate" fix flipped tests with no design review. For a FULL item give the evidence and the gap, never the fix: choosing it is the lane's brainstorming, checked by its cold review. A LIGHT item that turns out to change behaviour is a question to you, not a silent switch. Either way the lane writes its final report to a file and replies with the path: a long report scrolls off the pane before you read it. The research phase deploys BOTH agent kinds: read-only **scouts** on the codebase AND a **librarian** on the real world (upstream docs, kernel/UAPI headers, crate and protocol sources — source-verified answers with citations), so external-contract claims are checked against reality before the design relies on them.

## Supervision loop

**Discover first:** the installed binary is the authority — run `herdr agent` (command group) and a live `herdr agent list | jq` before trusting any status value or flag named below. The trigger set (`blocked`/`idle`/`done`; `working`/`unknown` never fire) was validated against the CLI on 2026-09-29.

```bash
bash scripts/watch-lanes        # background job: exits printing the lanes that need you
bash scripts/watch-lanes --list # snapshot: every lane + status, no wait
bash scripts/watch-lanes --except <id>  # skip a lane you're already landing
```

The watcher polls every `WATCH_INTERVAL_S` (default 20s) across ALL workspaces and exits on the first **named** lane that is `blocked` (approval/question dialog), `idle`, or `done` (`idle`/`done` both mean ready for input); `working` and `unknown` never fire — `unknown` covers detached/sleeping agents. Named-only is the default so other panes in the session never wake the supervisor; pass `--all` to include unnamed panes, and `--list` / `--all --list` for a snapshot with a count. Output is one tab-separated line per triggering lane: `id`, `pane_id`, `status` — `id` is the lane's name, or its pane id for unnamed lanes under `--all`; either form works as the target of any `herdr agent` command. No lane ids go in: the script discovers every lane in the session itself. `--except <id>` (repeatable) skips lanes you are already handling — a lane stays `done` until you tear it down, so without it every re-armed watcher fires at once on the lane you're landing.

Run it as a backgrounded job; on wake: `herdr agent read <printed id>` the question, answer it as in step 5 (SELECT dialog → `pane send-keys <pane_id> enter` alone on the highlighted option; free-text → `pane send-text` + `enter`; `agent prompt` only works on a lane in a normal idle/done turn), then **verify `agent get` flipped `blocked → working`** before re-arming (restart the watcher). A still-blocked lane means the answer did not land — re-read, diagnose the dialog type, never re-send blindly. `blocked` means the lane raised a question or approval dialog — that is the supervision channel.

## Ticket lifecycle

When a lane works a tracker ticket (Kaneo: To Do → In Progress → In Review → Done). Commits carry no ticket IDs, so **the ticket is where its commits are recorded**. Before every state change, re-read the ticket by key and confirm its title matches the lane; re-read after to verify.

| When | Ticket |
|---|---|
| Lane spawned | move to **In Progress**; comment `lane <name>, branch <branch>, track <LIGHT\|FULL>, started` — also the lock that keeps another supervisor off it |
| Lane landed | comment the landed commits (`<sha> <subject>` per line), the check result, and the lane report's summary and deviations; then move to **In Review** |
| Lane parked or dropped | comment why (question pending, superseded, refused) and move back to **To Do** |
| Done | never yours — the user moves it after reviewing and pushing |

Problems a lane notices outside its scope go in its report, never in its commits: collect them for the user (and, in a wave run, in the ledger) to become tickets through the writing-tickets skill, linked `related` to the ticket that found them.

## Backlog waves

For a long run over a ticket backlog instead of one batch: **understand every ticket first, then run them in waves**. The user approves the wave plan once; after that the waves run without asking, stopping only for a real question (a lane's escalation, scope wider than a ticket's evidence, removing public API).

1. **Ledger first.** The run's state lives in `$(git rev-parse --git-common-dir)/lane-waves.md` — per repository, never committed, and it survives restarts and context compaction. If it exists, resume from it instead of triaging again; update it after every state change (planned → running `<lane>`/`<branch>` → landed `<commits>` → in review; or skipped with the reason), and keep a **found during run** list: out-of-scope problems the lanes reported, each with the ticket that found it and its evidence.
2. **Triage wave (skill: dispatching-parallel-agents).** Split the open tickets by area (their tags, or the files they cite) and give each area to ONE read-only subagent. Each verifies every ticket against current target and returns one row per ticket: key · title · severity · state (present / already fixed / duplicate of `<key>` / unclear) · files it touches · track (LIGHT/FULL, by the brief-skeleton rule) · coupled with or depends on `<keys>` · note. Subagents read and report only — they change no file and no ticket.
3. **Plan the waves.** Drop fixed, duplicate and unclear tickets into a "not planned" list with reasons (report them; never close them). An unclear ticket — no verifiable evidence, or a fix that lives in a source nobody can read — is a candidate for a rewrite with the writing-tickets skill, the user's call. Order the rest by severity (BLOCKER → HIGH → MEDIUM → LOW), then dependencies, then any remediation order an epic ticket states. Fill each wave with at most the requested lane count, no file shared between lanes in a wave. **Bundle small tickets**: tickets in one area that share files and are each too small to deserve a design (dead code, doc fixes, a missing test) become ONE FULL lane item — one design and plan covering all of them, each ticket an acceptance item — rather than a pipeline per ticket. At most 3 tickets per lane; the rest go to later waves. Stop at the requested number of waves.
4. **Approval — once.** Show the plan (waves, each lane's tickets, tracks, files, the not-planned list) as a question and WAIT for the go. Telling the user and carrying on is not waiting.
5. **Run each wave** exactly as a batch: briefs, create, spawn, supervise, land one at a time, remove — with every ticket following the Ticket lifecycle above (In Progress at spawn, commits commented and In Review at land). A wave is done when every lane is landed and removed, or parked with a recorded reason.
6. **Between waves:** have one subagent re-verify the next wave's cited lines against the new target (landed waves move lines and can fix or break tickets), adjust the plan in the ledger, then start the wave. No re-approval unless a ticket's scope grew.
7. **Finish:** a final table per wave (ticket, track, landed commits, status), the not-planned list, the found-during-run list (ready for the writing-tickets skill), and the lane list showing nothing left running.
