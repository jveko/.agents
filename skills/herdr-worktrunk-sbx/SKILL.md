---
name: herdr-worktrunk-sbx
description: Use with herdr-worktrunk when the repo root has .config/sbx.toml - lanes live only in Docker sandboxes (worktrunk-sbx's sbx-lane) with no local worktree - for creating a sandbox lane, spawning and supervising it, landing it (rebase + check in the sandbox), resolving a sandbox lane conflict or failed check, and removing it.
---

# Herdr + Worktrunk, sandbox lanes

## Overview

**Extends `herdr-worktrunk`; everything there applies unless this skill overrides it.** A repo whose root has `.config/sbx.toml` runs its lanes in Docker sandboxes through worktrunk-sbx: a lane is a **branch ref locally and a checkout only in its sandbox**. No local worktree is ever created — not by you, not by `wt`. `sbx-lane` owns the lane lifecycle (create, land, remove) the way `wt` does in vanilla.

`sbx-lane` = `bun ~/workspace/projects/worktrunk-sbx/scripts/sbx-lane.ts`. Run it from the repo root. Usage and failure modes: worktrunk-sbx's `docs/OPERATIONS.md`.

**REQUIRED:** `herdr-worktrunk` (supervision loop, answering, recovery table, brief pattern) · `herdr` (env gate, ids-from-JSON).

## Step overrides

| Vanilla step | Sandbox lane |
|---|---|
| 0. Gate | unchanged, plus: `.config/sbx.toml` exists at the repo root (else use vanilla) |
| 1. Create + warm | `sbx-lane create <branch>` — branch ref + sandbox + clone + boot + herdr server + ssh alias + herdr machine `sbx:<branch>`, in the FOREGROUND (returns when ready, ~1 min). **Never** `wt switch --create` for a lane. Re-running it repairs a half-provisioned lane |
| 2. Nested workspace | none — `spawn` creates the lane's workspace on the sandbox's herdr; it shows under the machine in the sidebar |
| 3+4. Spawn + brief | `sbx-lane spawn <branch> <lane> --brief /tmp/<brief>.txt [--kind omp]` — uploads the brief to the sandbox's `/tmp`, creates workspace + pane, starts the agent, sends the pointer, confirms pickup by polling; a typed-but-unsubmitted pointer gets **Enter alone** |
| 5. Supervise | `sbx-lane watch` (background; `--list` snapshot) replaces `scripts/watch-lanes` — local lanes AND every sandbox of the repo; rows `id pane status branch`. Read/answer: `sbx-lane herdr <branch> -- agent read <id> --source recent-unwrapped`, then `sbx-lane herdr <branch> -- pane send-text <pane> "<answer>"` + `… pane send-keys <pane> enter`; verify `blocked → working` |
| 6. Land | `sbx-lane land <branch>` — only when the lane is `idle`/`done` (it rebases the lane's working tree). Never `wt merge` |
| Teardown | `sbx-lane remove <branch>` — refused while anything is unlanded; deletes sandbox, alias, machine, records AND the local branch. No `herdr workspace close`, no `wt remove` |

Review before landing: `sbx-lane fetch <branch>` fast-forwards the local lane branch, then `git log -p <target>..<branch>` — no checkout needed.

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
| Leaving landed lanes around | a running sandbox bills until parked (TTL) or removed; an open herdr machine keeps it awake | `remove` after landing; `herdr machine disable <id>` for lanes you aren't watching |

## Brief adjustments

Use `brief-skeleton.md` from `herdr-worktrunk` with these slots changed: the lane's cwd is the sandbox repo (`~/workspace` there), not a local worktree path; the brief file stays local (`/tmp/<lane>.txt`) — `spawn` uploads it; tell the lane its gates run in the sandbox (toolchain installed by the repo's `mise.toml` / image) and that it must **commit and never push** — landing is yours.
