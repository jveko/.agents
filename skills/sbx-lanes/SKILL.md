---
name: sbx-lanes
description: Use when the repo root has .config/sbx.toml - its coding lanes live only in Docker sandboxes driven by the sbx-lane CLI, with no local worktree - for creating a sandbox lane, spawning and briefing a lane agent, supervising lanes in Herdr, answering a blocked lane, landing it (rebase + check in the sandbox), resolving a lane conflict or failed check, parking, and removing it.
---

# Sandbox lanes (`sbx-lane`)

## Overview

A repo whose root has `.config/sbx.toml` runs its lanes in Docker sandboxes: a lane is a **branch ref locally and a checkout only in its sandbox**. No local worktree is ever created — not by you, not by `wt`, not by herdr. `sbx-lane` owns the whole lane lifecycle; Herdr shows the lanes and carries your answers to them.

You are the **supervisor**: each lane runs one agent on its own brief in its sandbox; you create lanes, spawn and brief agents, watch them, answer them, review, land and remove.

`sbx-lane` is the compiled binary at `~/.local/bin/sbx-lane`. Install it once from `~/workspace/projects/worktrunk-sbx`: `mise install` (the Bun its `mise.toml` pins), `bun install --frozen-lockfile`, `bun run install-bin`, with `~/.local/bin` on `PATH` (worktrunk-sbx's `docs/SETUP.md`, §2). Run it from the repo root. Usage and every failure mode: worktrunk-sbx's `docs/OPERATIONS.md`.

**REQUIRED:** the `herdr` skill (env gate, ids taken verbatim from JSON). The lane's state lives in one record per lane under `.git/sbx-lane/state/`; never edit it by hand, and never delete `.git/sbx-lane` — `remove` cannot find a sandbox no record names.

**Not for:** a repo without `.config/sbx.toml` — its lanes are local worktrees (`herdr-worktrunk`).

## Lifecycle

| Step | Command |
|---|---|
| 0. Gate | `test "${HERDR_ENV:-}" = 1` before any herdr control call, and `.config/sbx.toml` at the repo root. Opting a repo in is the user's call: `sbx-lane init` drafts the file (never overwrites one). `sbx-lane doctor` triages credentials, the gateway, herdr, the installed binary and the lane records |
| 0b. Name | branch `<type>/<topic>` (fix, feat, refactor, docs, test, chore; 2–4 kebab words naming the change); lane (agent) name = `<topic>`, `[a-z][a-z0-9_-]{0,31}`, unique. Never a ticket key, wave or lane number. Branch names use only `[A-Za-z0-9._/-]` (anything else is refused) |
| 1. Create | `sbx-lane create <branch>` in the FOREGROUND (~1 min): branch ref + sandbox + clone + boot + herdr server + a herdr workspace `<branch>` nested under the repo's, with a `shell` tab into the sandbox. Re-running it repairs a half-provisioned lane |
| 2. Spawn + brief | write the brief to a local file (structure: `brief-skeleton.md` in this directory), then `sbx-lane spawn <branch> <lane> --brief /tmp/<lane>.txt [--kind omp]` — uploads it to the sandbox's `/tmp` as `/tmp/<lane>+<brief>`, starts the agent, sends the pointer and confirms pickup; a typed-but-unsubmitted pointer gets Enter alone. Spawn adds a lane tab per agent: Enter in it opens the agent (`--takeover`), `ctrl+b q` comes back; tabs never wake a parked sandbox |
| 3. Supervise | background `sbx-lane watch` (see Supervision loop) |
| 4. Review | `sbx-lane review <branch>` fast-forwards the local lane branch and prints `git log -p <target>..<branch>` (`--stat`, `--oneline`) — no checkout needed |
| 5. Land | `sbx-lane land <branch>` — only when the lane is `idle`/`done` (it rebases the lane's working tree). One land at a time |
| 6. Remove | `sbx-lane remove <branch>` after the land, or `sbx-lane land <branch> --remove` in one step — deletes sandbox, herdr workspace, anchor worktree, the lane record AND the local branch |
| Between turns | `sbx-lane park <branch>` stops a lane you aren't watching now (returns at once); the next work command wakes it |

Never `wt switch`, `wt merge`, `wt remove`, `herdr worktree open/create` or `herdr workspace close` for a sandbox lane: there is no local worktree, and those skip the sandbox's guards.

## Delegation

`--kind omp` is the house default; use `claude`/`codex` when a task needs them (an unverified kind warns before anything is started). One lane per independent workstream — no two lanes may own the same files. The brief is the contract and the lane's report carries its state: you never work a lane's queue inline.

## Supervision loop

```bash
sbx-lane watch          # background job: exits printing the lanes that need you
sbx-lane watch --list   # snapshot: every lane + status, no wait
sbx-lane watch --json   # one JSON line per event
```

`watch` covers this repo's local agents AND every sandbox of the repo, never another project's; rows are `id pane status branch`. It fires on `blocked` (a question or approval dialog), `idle` and `done` (both: ready for input); `working` and `unknown` never fire. It remembers each fire by lane and turn: re-arm it with no flags — a handled lane stays quiet, a lane you prompt again fires when it finishes that turn or asks something new. `watch` has no `--except` (that is the vanilla `scripts/watch-lanes` flag; an unknown flag is exit 2). Without credentials it skips every sandbox, one stderr line per branch.

On each wake: read the lane — `sbx-lane herdr <branch> -- agent read <id> --source recent-unwrapped` — then answer by what it is showing:

| The lane is | Answer with |
|---|---|
| `idle`/`done`, needs its next instruction | `sbx-lane prompt <branch> <lane> "<text>"` (or `--brief <file>`) — it returns once herdr sees the new turn, so re-arm `watch` only after it. Never type a follow-up into the pane |
| `blocked` on a SELECT or permission dialog | `sbx-lane herdr <branch> -- pane send-keys <pane> enter` **alone** — Enter submits the highlighted option; typed text pollutes or vanishes. Approval-type dialogs are the user's decision: surface them, never auto-answer |
| `blocked` on a free-text question | `sbx-lane herdr <branch> -- pane send-text <pane> "<answer>"` then `… pane send-keys <pane> enter` (`prompt` refuses a blocked lane and says so) |

Then **verify `blocked → working`** (`sbx-lane herdr <branch> -- agent get <id>`) before re-arming. Still blocked means the answer did not land — re-read, diagnose the dialog type, never re-send blindly.

## Accounting

`sbx-lane list` shows every lane of the repo with its sandbox state (`running` bills, `stopped` is parked), age, when its TTL park fires, and whether the local branch is on the target (`sandbox has more` when the last fetch holds commits the target lacks) — check it after a batch so nothing is left running. `sbx-lane status` is the same for every local repo from one account sweep. `sbx-lane gc` (then `gc --apply`) is the batch-end sweep: it closes lanes whose sandbox answers 404, reports landed, parked, ahead and other machines' lanes without touching them, lists sandboxes no lane records, and sweeps litter — it never deletes a sandbox or a branch, and never wakes one.

These wake a parked lane: `create` (re-provisioning), `spawn`, `prompt`, `herdr`, `fetch`, `review`, `land`, `measure`, `remove`, and `shell`. A work command that finds the sandbox still stopping waits the stop out, up to 5 min (`SBX_LANE_STOP_WAIT_MS`), then exits 75 — re-run it. `mirror`, `watch`, `list`, `status`, `gc` and lane tabs never wake one.

One herdr session per project: from a plain terminal in the checkout, `sbx-lane session` opens the project's own session, so another project's lanes never show in it. Lane tabs closed by hand, or bare shells after a herdr restart without the bundled herdr plugin: `sbx-lane mirror` (from a herdr pane) reopens every running lane's workspace and tabs; parked lanes are only named.

Scripts: every command has `--help`; `list`, `status`, `doctor` and `gc` take `--json` (one document on stdout). Exit codes: 0 ok · 1 failed · 2 usage · 3 precondition — credentials/gateway → `sbx-lane doctor`; no `.config/sbx.toml` → `sbx-lane init`; no lane record → `sbx-lane create` · 4 refused (read it: it needs a flag, a clean sandbox or a landed lane) · 75 transient (SDK deadline, lock busy, a still-stopping sandbox — re-run) · 130 interrupted. The `re-run:` and `next:` lines stay on stdout.

## Landing

`land` = in the sandbox: refuse loose work → rebase the lane onto the **local** target (sent as a bundle, unpushed commits included) → run `.config/sbx.toml`'s `check` → here: fast-forward the lane branch and the target. No squash, no push, nothing built locally.

| Refusal | Do |
|---|---|
| `the sandbox has work outside the lane's commits` | prompt the lane to commit (or clean up) its work; land again |
| `conflicts with <target>` | the rebase was aborted, nothing moved. Prompt the lane with the exact `git rebase <ref>` command the refusal printed (the ref carries a hash suffix — never type it from memory): "run `<that command>`, resolve the conflicts preserving both intents, `GIT_EDITOR=true git rebase --continue`, rerun your gates, report" → land again |
| `check failed` | the lane stays rebased; prompt it with the failing output to fix + commit; land again |
| `check timed out` | the check outran 30 min (`SBX_LANE_CHECK_TIMEOUT_MS` raises it); the lane stays rebased and the remote check keeps running — wait for it (or raise the budget) before landing again |
| `the sandbox has a rebase in progress` | nothing was touched; the message names the commit it replays onto and whether it is this lane's own — finish (`git rebase --continue`) or discard (`git rebase --abort`) in the sandbox, then land again |
| `the rebase timed out` | the sandbox rebase outran its budget (`SBX_LANE_REBASE_TIMEOUT_MS`, 300000 ms default); inspect the sandbox, then land again |
| `<b> is working` / `<b> is blocked` | wait for `idle`/`done`, then land |
| `… moved during the land` / `gained a commit during the check` / `… during the land` | nothing landed — land again |
| `local <b> has commits the lane lacks` | work exists only on the local branch — bring it into the lane or drop it deliberately (the user's call), then land |
| `the sandbox has no <b> branch` | the sandbox repo lost the lane branch — re-run `sbx-lane create <b>` to re-seed it |
| `<target> … can't fast-forward` | the target's own worktree has changes in the way, or the target moved — sort the target out, land again |

Land lanes **one at a time**: each land rebases onto a target that already holds the previous lands, and `check` runs on that combined tree — with a full-gate `check` (e.g. `hk check --all`) the last land's check covers the batch.

## Removal

`sbx-lane remove <branch>` after the land. A refusal lists exactly what would be lost (lane commits not on the target, loose work in the sandbox); land it or resolve it. It also refuses while the lane agent is `working`/`blocked` — wait for `idle`/`done`. `--force` discards what was listed — that is the user's decision, never yours. A removal run outside herdr keeps the lane's herdr workspace and anchor worktree; `sbx-lane gc --apply` from a herdr pane finishes them.

## Common mistakes

| Mistake | Reality | Fix |
|---|---|---|
| `wt switch --create` / `herdr worktree open` for a sandbox lane | creates the local worktree the repo opted out of | `sbx-lane create` |
| `wt merge` / `wt remove` on a lane | no worktree to run in; bypasses the sandbox guard | `sbx-lane land` / `sbx-lane remove` |
| Landing a `working` lane | the rebase rewrites files under the agent | wait for `idle`/`done` |
| Typing a follow-up into the pane | text typed into an idle input box races herdr's state, and the watcher fires on the turn the lane already finished | `sbx-lane prompt` |
| Re-sending the brief text when the lane didn't start | it was typed, just not submitted | `spawn` already sends Enter alone; if still idle, read the pane first |
| Waiting for `create` in the background | the lane isn't spawnable until it returns | run it in the foreground |
| Spawning after `create` failed on `sandbox-boot` | the agent's config never arrived — an omp lane opens with "No models available" / "No model selected" | report the failed steps it listed; re-run `create` (re-boots) once the cause is fixed, then spawn. A re-run of `sbx-lane spawn <branch> <lane> --brief <file>` adopts the running agent and re-delivers the pointer (an agent already working gets no second pointer) — never type the pointer by hand |
| Debugging networking when a lane says "No models available" | it's the boot (config pull), not the network | re-run `create`, then restart the lane's agent: exit it in its pane, `sbx-lane herdr <branch> -- agent start <lane> --kind omp --pane <pane>`, then re-run `sbx-lane spawn <branch> <lane> --brief <file>` |
| Leaving lanes running between turns or after a batch | a running sandbox bills until its TTL parks it | `sbx-lane park` between turns; `remove` after landing; `sbx-lane list` after a batch |
| Changing a ticket's state from memory | IDs from a shortlist get mixed up — a rejected ticket gets closed | re-read the ticket by its key right before the change, confirm its title matches the lane, then change it; re-read after |

## Briefs

Start from `brief-skeleton.md` in this directory. The lane's cwd is the sandbox repo (`~/workspace` there); the brief file stays local (`/tmp/<lane>.txt`) and `spawn` uploads it. Tell the lane its gates run in the sandbox (toolchain from the repo's `mise.toml` / image) and that it must **commit and never push** — landing is yours.

Its report file is in the sandbox's `/tmp` as `/tmp/<lane>-report.md`. Read the agent's own output with `sbx-lane herdr <branch> -- agent read <id> --source recent-unwrapped`; for the report file, run `sbx-lane shell <branch>` (it wakes a parked lane) and `cat /tmp/<lane>-report.md` in it.

Scope each brief item to the ticket's cited evidence and stated fix, never "and any sibling …"; anything wider, or removing public API, is the user's call before spawning. For a behaviour change give the evidence and the gap, not the fix — the lane's own design step picks it.

## Tickets (optional)

Lanes need no tracker. When the work comes from tickets, load the tracker's skill too — **`kaneo`** for Kaneo — and it owns the ticket. For a lane, **started** is the spawn and **finished** is the land: give the ticket skill the lane name, branch, landed commits, check result and report summary at those moments. Problems a lane notices outside its scope go in its report, never in its commits; collect them for the user.

## Backlog waves

For a long run over a ticket backlog: waves run in parallel, one lane slot per wave; inside a slot tickets run one at a time, one ticket per lane (create → spawn → supervise → review → land → ticket → remove → next). Keep the run's ledger in `$(git rev-parse --git-common-dir)/lane-waves.md` (never committed; resume from it). Verify the chosen waves' tickets with read-only subagents first, show the plan once as a question and WAIT for the go, then run without asking except for a real question. Across slots, lands go one at a time. Finish with a table per slot, the not-planned list, the found-during-run list, and `sbx-lane list` showing nothing left running.
