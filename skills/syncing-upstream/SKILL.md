---
name: syncing-upstream
description: Use when asked to sync, update, or merge the omni-mail repo with its upstream project, pull latest OmniMail changes, or bring a fresh squashed "source repo import" up to date with mibgb65-cloud/OmniMail.
---

# Syncing omni-mail with upstream

## Overview

Repo `~/workspace/projects/omni-mail-jveko`: `origin` = private mirror (jveko/omni-mail), `upstream` = source project (mibgb65-cloud/OmniMail, default branch `main`). Sync = merge `upstream/main` into local `main`, preserve local customizations, verify, push fast-forward. Never rebase published `main`; never push to `upstream`.

## Local customizations that MUST survive every sync

The whole delta vs upstream is exactly 7 files:

- 5 deleted workflows: `.github/workflows/{android-ci,android-release,ci,float-release,release}.yml`
- `package.json` `name: "omni-mail"` (upstream: `omnimail`)
- `wrangler.jsonc` real Cloudflare bindings: `database_id: 2afeaf44-…`, `database_name`/`bucket_name: omni-mail`, `bucket_name: omni-mail-backups`, matching `preview_bucket_name`s

Integrity check after any sync: `git diff upstream/main HEAD --stat` → exactly those 7 files, nothing else.

## Steady-state sync (shared ancestry exists)

1. `git fetch upstream --prune`
2. `git rev-list --left-right --count main...upstream/main` — right count = incoming commits; `0` → already in sync, stop.
3. `git merge upstream/main`
4. Resolve conflicts by **combining intent**, never blanket `--ours`/`--theirs`:
   - `package.json`: local `name` + upstream `version`
   - `wrangler.jsonc`: keep local bindings AND take upstream's new sections
   - workflow files: local repo intentionally has no workflows; keep them deleted unless upstream explicitly reworked one
5. Run the 7-file integrity check above.
6. Verify: `npm ci` if `node_modules` missing, then `npm run check` (exit 0) and `npm test` (all tests pass).
7. `git push origin main` — must be fast-forward; if git asks for force, stop and re-diagnose.

## Fresh squashed import (no shared ancestry)

Symptom: local `main` is a single `source repo import` commit; `git merge upstream/main --allow-unrelated-histories` would conflict on **every** differing file.

1. Find the true base commit: the upstream commit whose diff to local HEAD equals only the 7-file customization delta.
   ```bash
   for c in $(git rev-list vN.N.N ^vN.N.M); do echo "$(git diff --shortstat $c HEAD) $c"; done
   ```
   Pick the smallest diff; confirm `git diff <base> HEAD --stat` = the 7 files only.
2. Give git a real merge base: `git replace --graft <local-root-commit> <base>`
3. `git merge upstream/main`, resolve as in steady-state step 4, commit.
4. Remove scaffolding: `git replace -d <local-root-commit>` (the merge commit keeps real ancestry; future syncs are plain merges).
5. Verify and push as in steady-state steps 6–7.

## Common mistakes

| Mistake | Fix |
|---|---|
| `--ours`/`--theirs` on package.json/wrangler → drops one side | Combine both intents per file |
| `--allow-unrelated-histories` without graft first | Graft onto located base, then merge |
| Rebasing published `main` | Merge only; history is shared with origin |
| Pushing before checks/tests run | `npm run check` + `npm test` first |
| Assuming sync worked without diffing | Always run the 7-file integrity check |