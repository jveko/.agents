# .agents

Personal AI agent configuration for this machine — the `~/.agents` checkout.
This repo is the **global layer** only; every project keeps its own instructions.

## Layout

| Path | Purpose |
|------|---------|
| `AGENTS.md` | Global, user-scope instructions: identity, decision rules, safety hard rules. Applies to every project on this machine. |
| `skills/` | Agent skills — each folder is one skill with its own `SKILL.md`. |
| `.gitignore` | Keeps machine-local state out of the repo (`.skill-lock.json`). |

## How the layers combine

1. Project-level `AGENTS.md` / `CLAUDE.md` in the repo being worked on —
   closest file wins.
2. This repo's `AGENTS.md` — personal defaults that project files cannot
   express.

This repo never overrides a project's own file; it supplies only the defaults
the code cannot tell you.

## Working with skills

Skills are plain files: edit in place, commit one skill per commit when
practical. Skill-manager state (`~/.agents/.skill-lock.json`) is machine-local
and intentionally gitignored — the skill folders themselves are the source of
truth.

Follows the [AGENTS.md standard](https://github.com/agentsmd/agents.md) and the
`~/.agents/` directory convention (global at `~/.agents/`, workspace at
`./.agents/`).
