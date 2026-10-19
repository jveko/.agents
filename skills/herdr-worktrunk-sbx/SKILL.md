---
name: herdr-worktrunk-sbx
description: Use with herdr-worktrunk when the repo root has .config/sbx.toml - lanes live only in Docker sandboxes (worktrunk-sbx's sbx-lane) with no local worktree - for creating a sandbox lane, spawning and supervising it, landing it (rebase + check in the sandbox), resolving a sandbox lane conflict or failed check, and removing it.
---

# Herdr + Worktrunk, sandbox lanes

## Overview

**Extends `herdr-worktrunk`; everything there applies unless this skill overrides it.** A repo whose root has `.config/sbx.toml` runs its lanes in Docker sandboxes through worktrunk-sbx: a lane is a **branch ref locally and a checkout only in its sandbox**. No local worktree is ever created — not by you, not by `wt`. `sbx-lane` owns the lane lifecycle (create, land, remove) the way `wt` does in vanilla.

`sbx-lane` is the compiled binary installed at `~/.local/bin/sbx-lane`; install it once from `~/workspace/projects/worktrunk-sbx` with `bun install --frozen-lockfile` then `bun run install-bin`, `~/.local/bin` on `PATH` (the steps: worktrunk-sbx's `docs/SETUP.md`, §2). Run it from the repo root. Usage and failure modes: worktrunk-sbx's `docs/OPERATIONS.md`.

**REQUIRED:** `herdr-worktrunk` (supervision loop, answering, recovery table, brief pattern) · `herdr` (env gate, ids-from-JSON).

## Step overrides

| Vanilla step | Sandbox lane |
|---|---|
| 0. Gate | unchanged, plus: `.config/sbx.toml` exists at the repo root (else use vanilla) |
| 1. Create + warm | `sbx-lane create <branch>` — branch ref + sandbox + clone + boot + herdr server + ssh alias + a herdr workspace `<branch>` nested under the repo's (a `shell` tab into the sandbox), in the FOREGROUND (returns when ready, ~1 min). **Never** `wt switch --create` for a lane. Re-running it repairs a half-provisioned lane |
| 2. Nested workspace | none — `create` opened it. `spawn` adds one lane tab per agent: the agent's live state, also in the agent panel beside local agents; Enter in the tab opens the agent itself, `ctrl+b q` comes back. Tabs never wake a parked sandbox |
| 3+4. Spawn + brief | `sbx-lane spawn <branch> <lane> --brief /tmp/<brief>.txt [--kind omp]` — uploads the brief to the sandbox's `/tmp`, creates workspace + pane, starts the agent, sends the pointer, confirms pickup by polling; a typed-but-unsubmitted pointer gets **Enter alone** |
| 5. Supervise | `sbx-lane watch` (background; `--list` snapshot) replaces `scripts/watch-lanes` — this repo's local agents AND every sandbox of the repo, never another project's; rows `id pane status branch`. It remembers each fire by lane and turn: re-arm with no flags — a handled lane stays quiet, a lane you prompt again fires when it finishes that turn or asks something new. Read: `sbx-lane herdr <branch> -- agent read <id> --source recent-unwrapped`. Next instruction to an idle/done lane: `sbx-lane prompt <branch> <lane> "<text>"` (or `--brief <file>`) — it returns once herdr sees the new turn, so re-arm `watch` only after it; never type a follow-up with `pane send-text`. Answer a question (`blocked`): `sbx-lane herdr <branch> -- pane send-text <pane> "<answer>"` + `… pane send-keys <pane> enter`; verify `blocked → working` |
| 6. Land | `sbx-lane land <branch>` — only when the lane is `idle`/`done` (it rebases the lane's working tree). Never `wt merge` |
| Teardown | `sbx-lane remove <branch>` — refused while anything is unlanded; deletes sandbox, alias, herdr workspace, anchor worktree, records AND the local branch. No `herdr workspace close`, no `wt remove`. `sbx-lane land <branch> --remove` lands and removes in one step |

`sbx-lane list` shows every recorded lane of the repo with its sandbox state (`running` bills, `stopped` is parked), age, when its TTL park fires, and whether the local branch is on the target (`sandbox has more` when the last fetch holds commits the target lacks) — check it after a batch so nothing is left running; `sbx-lane gc` (then `gc --apply`) removes landed idle lanes, forgets sandboxes that are gone and finds sandboxes no lane records. `watch`, `list` and lane tabs never wake a parked sandbox; without credentials `watch` skips every sandbox (one stderr line per branch) and `list` refuses.

A lane you aren't watching keeps billing until its TTL: `sbx-lane park <branch>` stops it now (returns at once; the next work command wakes it — one that finds the sandbox still stopping waits the stop out, up to 5 min by default (`WT_SBX_STOP_WAIT_MS`), then starts it; at that limit it exits 75: re-run it). One herdr session per project: from a plain terminal in the checkout, `sbx-lane session` opens the project's own session, so another project's lanes never show in it.

Lane tabs closed by hand, or bare shells after a herdr restart without the worktrunk-sbx herdr plugin: `sbx-lane mirror` (from a herdr pane) reopens every running lane's workspace and tabs and restarts bare lane tabs. Parked lanes are only named; any work command (`spawn`, `herdr`, `fetch`, `review`) wakes one.

Review before landing: `sbx-lane review <branch>` fast-forwards the local lane branch and prints `git log -p <target>..<branch>` in one step (`--stat` for a summary, `--oneline` for commits only) — no checkout needed.

Reading results in a script: every command has `--help`; `list`, `status`, `doctor` and `gc` take `--json` (one document on stdout), `watch --json` streams one JSON line per event. Exit codes: 0 ok · 1 failed · 2 usage · 3 precondition (credentials, `.config/sbx.toml` or a lane record missing — run `sbx-lane doctor`) · 4 refused (read the refusal: it needs a flag, a clean sandbox or a landed lane) · 75 transient (SDK deadline, lock busy, a still-stopping sandbox — re-run) · 130 interrupted. The `re-run:` and `next:` lines stay on stdout.

## Landing

`land` = in the sandbox: refuse loose work → rebase the lane onto the **local** target (sent as a bundle, unpushed commits included) → run `.config/sbx.toml`'s `check` → here: fast-forward the lane branch and the target. No squash, no push, nothing built locally.

| Refusal | Do |
|---|---|
| `the sandbox has work outside the lane's commits` | prompt the lane to commit (or clean up) its work; land again |
| `conflicts with <target>` | the rebase was aborted, nothing moved. Prompt the lane: "run `git rebase refs/sbx-land/<target>`, resolve the conflicts preserving both intents, `GIT_EDITOR=true git rebase --continue`, rerun your gates, report" → land again |
| `check failed` | the lane stays rebased; prompt it with the failing output to fix + commit; land again |
| `<target> … can't fast-forward` | the target's own worktree has changes in the way, or the target moved — sort the target out, land again |

Land lanes **one at a time**: each land rebases onto a target that already holds the previous lands, and `check` runs on that combined tree — with a full-gate `check` (e.g. `hk check --all`) the vanilla "full gate on final main after the batch" is covered by the last land's check.

## Removal

`sbx-lane remove <branch>` after the land. A refusal lists exactly what would be lost (lane commits not on the target, loose work in the sandbox); land it or resolve it. `--force` discards what was listed — that is the user's decision, never yours.

## Common mistakes

| Mistake | Reality | Fix |
|---|---|---|
| `wt switch --create` / `herdr worktree open` for a sandbox lane | creates the local worktree the repo opted out of | `sbx-lane create` |
| `wt merge` / `wt remove` on a lane | no worktree to run in; bypasses the sandbox guard | `sbx-lane land` / `sbx-lane remove` |
| Landing a `working` lane | the rebase rewrites files under the agent | wait for `idle`/`done` |
| Re-sending the brief text when the lane didn't start | it was typed, just not submitted | `spawn` already sends Enter alone; if still idle, read the pane first |
| Waiting for `create` in the background | the lane isn't spawnable until it returns | run it in the foreground |
| Spawning after `create` failed on `sandbox-boot` | the agent's config never arrived — an omp lane opens with "No models available" / "No model selected" | report the failed steps it listed; re-run `create` (re-boots) once the cause is fixed, then spawn |
| Debugging networking when a lane says "No models available" | it's the boot (config pull), not the network | re-run `create`, then restart the lane's agent: exit it in its pane, `sbx-lane herdr <branch> -- agent start <lane> --kind omp --pane <pane>`, re-send the pointer |
| Changing a ticket's state from memory (Done, In progress) | IDs from a shortlist get mixed up — a rejected ticket gets closed | re-read the ticket by its key right before the change, confirm its title matches the lane, then change it; re-read after to verify |
| Leaving landed lanes around | a running sandbox bills until parked (TTL) or removed | `remove` after landing; `sbx-lane list` after a batch |

## Brief adjustments

Use `brief-skeleton.md` from `herdr-worktrunk` with these slots changed: the lane's cwd is the sandbox repo (`~/workspace` there), not a local worktree path; the brief file stays local (`/tmp/<lane>.txt`) — `spawn` uploads it; tell the lane its gates run in the sandbox (toolchain installed by the repo's `mise.toml` / image) and that it must **commit and never push** — landing is yours (`sbx-lane land`, not `wt merge`). Its report file is in the sandbox's `/tmp`: read it with `ssh -F ~/.config/worktrunk-sbx/ssh_config "$(git config --get worktrunk.state.<branch>.vars.sbx-ssh-host)" cat /tmp/<lane>-report.md` — read-only, and like any ssh it wakes a parked sandbox.
