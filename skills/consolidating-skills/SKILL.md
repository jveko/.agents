---
name: consolidating-skills
description: Use when upstream superpowers skills have changed and local ~/.agents/skills needs syncing
---

# Consolidating Skills from Superpowers

## Overview

Your local skills at `~/.agents/skills/` are based on the [superpowers](https://github.com/obra/superpowers) repository. Upstream skills get upgraded over time. This skill provides the process for importing those upgrades while preserving local customizations.

**Core principle:** Preserve local customizations, adopt upstream improvements, never overwrite blindly.

## When to Use

- Upstream superpowers has new commits on `main`
- You want to incorporate skill improvements without losing local changes
- New support files (prompts, scripts, references) were added upstream
- You need to verify which local skills have diverged from upstream

**Directory layout:**
```
~/.agents/skills/                     ← Your active local skills
~/workspace/projects/superpowers/     ← Upstream cloned here
  skills/
    brainstorming/
    consolidating-skills/          ← This skill
    ...
```

## The Process

### 1. Pull Latest Upstream

```bash
cd ~/workspace/projects/superpowers && git pull
```

### 2. Find What Changed

```bash
SKILLS_DIR="$HOME/.agents/skills"
UPSTREAM_DIR="$HOME/workspace/projects/superpowers/skills"

echo "=== SKILL.md files that differ ==="
for skill in brainstorming dispatching-parallel-agents \
             finishing-a-development-branch subagent-driven-development \
             systematic-debugging writing-plans writing-skills; do
  result=$(diff "$SKILLS_DIR/$skill/SKILL.md" "$UPSTREAM_DIR/$skill/SKILL.md" 2>/dev/null)
  if [ $? -ne 0 ]; then echo "--- $skill/SKILL.md differs ---"; echo "$result"; fi
done

echo "=== New files upstream ==="
diff -rq "$UPSTREAM_DIR" "$SKILLS_DIR" | grep "Only in $UPSTREAM_DIR"
```

### 3. Merge SKILL.md Changes

**Surgical merge (recommended):** Read the upstream diff. Apply only the improvements to your local `SKILL.md` by hand. Keep your local customizations:
- `dispatching-parallel-agents` references (upstream uses `superpowers:executing-plans`)
- Oracle review workflow, plan document chunking

**Full replace (only for major rewrites):**
```bash
cp "$UPSTREAM_DIR/writing-plans/SKILL.md" "$SKILLS_DIR/writing-plans/SKILL.md"
# Then re-apply customizations
```

### 4. Copy New Support Files

```bash
cp "$UPSTREAM_DIR/brainstorming/visual-companion.md" "$SKILLS_DIR/brainstorming/"
```

Note: `writing-plans/plan-document-reviewer-prompt.md` was deleted upstream in v6.4.2;
the local copy is kept because the local Oracle Review workflow references it.
Do not treat it as a dead file.

### 5. Verify

```bash
echo "=== Final diff check ==="
for skill in brainstorming dispatching-parallel-agents \
             finishing-a-development-branch subagent-driven-development \
             systematic-debugging writing-plans writing-skills; do
  result=$(diff "$SKILLS_DIR/$skill/SKILL.md" "$UPSTREAM_DIR/$skill/SKILL.md" 2>/dev/null)
  if [ $? -eq 0 ]; then echo "  ✅ $skill - identical to upstream"
  else echo "  ⚠️  $skill - has local customizations (verify intentional)"; fi
done
```

## Overlapping Skills

| Skill | Local Path | Upstream Path |
|---|---|---|
| brainstorming | `~/.agents/skills/brainstorming/` | `skills/brainstorming/` |
| dispatching-parallel-agents | `~/.agents/skills/dispatching-parallel-agents/` | `skills/dispatching-parallel-agents/` |
| finishing-a-development-branch | `~/.agents/skills/finishing-a-development-branch/` | `skills/finishing-a-development-branch/` |
| subagent-driven-development | `~/.agents/skills/subagent-driven-development/` | `skills/subagent-driven-development/` |
| systematic-debugging | `~/.agents/skills/systematic-debugging/` | `skills/systematic-debugging/` |
| writing-plans | `~/.agents/skills/writing-plans/` | `skills/writing-plans/` |
| writing-skills | `~/.agents/skills/writing-skills/` | `skills/writing-skills/` |

## New Skills Only in Upstream

Adopted 2026-09-28 (v6.4.2 pull): `diagnosing-superpowers`, `executing-plans`,
`test-driven-development`, `verification-before-completion`, `requesting-code-review`,
`receiving-code-review`.

Reviewed and declined: `using-git-worktrees`, `using-superpowers` (low value here —
these skills run directly from `~/.agents/skills/`, no bootstrap needed).
Their references in `executing-plans` and `subagent-driven-development` were
neutralized on 2026-09-28 — re-neutralize after overwriting those files.

## Local Customizations to Preserve

- **writing-plans**: `dispatching-parallel-agents` references, Oracle review workflow, plan document chunking, Workflow Order section, `## Remember` bullets, checkbox Self-Review before Oracle review, Inline Execution handoff (no executing-plans), Task Structure ending at Step 4 (no Commit step), local `plan-document-reviewer-prompt.md`
- **subagent-driven-development**: parallel-batch model (max 3, independence revalidation, dual-verdict gate where both verdicts must pass, controller-owned commits) layered on the upstream ledger/workspace rewrite; Integration section uses `dispatching-parallel-agents` not `superpowers:executing-plans`
- **writing-skills**: Frontmatter `name`/`description` fields, Overview line naming `~/.claude/skills` and `~/.agents/skills/`, interpreter-invocation rule for bundled scripts

## Common Mistakes

**Blind overwrite:** Copying upstream SKILL.md over local without re-applying customizations. Always diff first, merge surgically.

**Missing new support files:** Upstream may add prompt files, scripts, or references that a skill's SKILL.md now references. Run the "new files" check every time.

**Skipping verification:** After merging, re-run the diff loop to confirm only intentionally divergent files remain.

## Prior Consolidation

2026-09-28 (v6.4.2 pull, d884ae0 → 8ca22db): overwrote brainstorming,
dispatching-parallel-agents, systematic-debugging, finishing-a-development-branch
(no local divergences worth keeping); full-replaced writing-plans and writing-skills
then re-applied customizations; ported the parallel-batch model onto upstream's
rewritten subagent-driven-development (local spec/quality reviewer prompts superseded
by upstream task-reviewer-prompt.md); adopted 6 upstream-only skills (see above).

Initial sync performed on 2026-05-02. Merged: Model Selection, Implementer Status, Code Organization, escalation guidance, self-review, checkbox syntax, plan reviewer prompt. Preserved: local paths, oracle workflow, chunking, `dispatching-parallel-agents` references. Copied: visual-companion, spec-document-reviewer, plan-document-reviewer, CREATION-LOG, brainstorming scripts.
