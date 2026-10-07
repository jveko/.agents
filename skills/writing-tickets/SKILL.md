---
name: writing-tickets
description: Use when turning gap-analysis findings, review results, or brainstormed feature ideas into tracker tickets (Kaneo) - writing self-contained, lane-sized tickets with verified evidence, scope, acceptance criteria and links, deduplicated against existing tickets, previewed before anything is created.
---

# Writing Tickets

## Overview

**A ticket is a lane's whole scope.** The lane that runs it later (herdr-worktrunk) gets the ticket and nothing else — no session, no report, no memory of the analysis that produced it. Everything a lane needs to stay in scope must be IN the ticket, verified against the code at the time it was written.

Observed failure this prevents: tickets that said "Fix: see report section" and pointed at a `local://` report from an expired session. The lanes couldn't read it, so supervisors guessed the fix — one guess deleted 66 public constants.

## Inputs

- **Findings** (gap analysis, audits, review results): problems with evidence.
- **Ideas** (brainstorming what to add or extend): goals without evidence of a defect. They go straight to tickets — the lane that runs one designs it (FULL track, superpowers brainstorming in the lane).

## Ticket format

Title — match the tracker's existing convention; for this workspace:
- finding: `[SEVERITY] [area] <one-line problem>` — SEVERITY is BLOCKER, HIGH, MEDIUM or LOW
- idea: `[FEATURE] [area] <one-line capability>`

Priority: BLOCKER → urgent, HIGH → high, MEDIUM → medium, LOW → low; for an idea, the user's priority (default medium).

Body:

```markdown
**Kind:** gap | feature · **Area:** <area> · **Track hint:** LIGHT | FULL

**Problem:** <what is wrong, and its consequence>      ← gap
**Goal:** <what a user can do once this exists, and why> ← feature

**Evidence:** (verified at <short commit>)
- `path:line` — `<quoted code or config>` — <what it shows>

**Scope:** <files / functions this changes>
**Out of scope:** <neighbouring things it must not touch, with the ticket that owns them if any>

**Acceptance:**
- <observable check or test that proves it done>

**Fix direction:** <the known fix> | open — the lane designs it

**Related:** duplicates / depends on / blocks / part of <keys>
```

## Size: one FULL-sized change

Prefer tickets that deserve the full superpowers pipeline over many tiny ones — but each still landable in one lane run. A ticket is the right size when:

1. **One goal** — statable in one sentence without "and".
2. **It carries a decision or a behaviour change.** If it doesn't (dead code, a doc fix, a missing test), it is too small on its own: fold it into the FULL ticket for the same area as an explicit Scope item ("while here: remove the dead helper at `signer.rs:752`") instead of filing it separately.
3. **One area**, about 5 code files or fewer.
4. **One lane lands it in one run**, with its tests.
5. **One sitting to review** — about 400 changed lines or fewer, docs excluded.

Too big → parent + children (rule 3 below). A ticket that stays LIGHT is a cluster of pure docs/config/CI changes with nothing else in its area to join.

## Rules

1. **Self-contained.** Never "see report", a `local://` / session / artifact link, or "as discussed". Inline the evidence and the fix direction, or write "open — the lane designs it".
2. **Verify before writing.** Re-read every cited line against the current target branch; quote it; record the commit. A finding the code no longer shows is dropped (report it as already fixed), never filed.
3. **One ticket = one landable change** — one lane can implement and land it with its tests. Larger findings or features become a **parent** ticket (the goal, the shared context, the list of children) plus **children**, each landable; link them with the tracker's subtask relation (parent → child) and `blocks` where order matters.
4. **Scope is explicit both ways.** Name what changes AND what must not. The lane will treat anything outside Scope as a question, so a vague Scope stalls it and a wide one licenses drift.
5. **Track hint** by the herdr-worktrunk rule: LIGHT only when no behaviour changes (docs, comments, config, CI, ignore rules, removing provably unused non-public code); everything else FULL. A feature is always FULL.
6. **Acceptance is observable**: a test that fails before and passes after, a command's output, a behaviour a user sees — not "code is cleaner".
7. **No ticket IDs in the code they describe** — tickets reference code, code never references tickets.

## Process

1. **Collect** the items from the source (a file, findings/ideas earlier in the session, or a wave run's found-during-run list). Number them; keep each item's origin (the ticket that found it) for a `related` link.
2. **Verify and enrich in parallel** (skill: dispatching-parallel-agents) when there are more than a handful: one read-only subagent per area re-reads the cited code, quotes evidence, proposes Scope / Out of scope / Acceptance / Track hint. Ideas get Goal, the code they extend, Scope and Acceptance — no invented design.
3. **Dedupe** against open tickets: search the tracker for each item (key terms, cited files). Exact duplicate → drop, note the existing key. Overlapping → file it with a `related` link and narrow its Scope to what the existing ticket doesn't cover.
4. **Size**: fold small items into the FULL ticket of their area (as Scope items) and split anything not landable in one lane into parent + children — see Size above.
5. **Preview, then WAIT.** Show one table — #, title, priority, kind, track, scope files, relations, and dropped items with the reason (already fixed / duplicate of KEY) — and wait for the user's go. Telling and carrying on is not waiting.
6. **Create** in order: parents first, then children, then relations (`subtask` parent → child, `blocks`, `related`). Status: the project's first column (To Do).
7. **Verify**: re-read every created ticket by key; report the table of created keys and relations.

## Common mistakes

| Mistake | Fix |
|---|---|
| "Fix: see report section" | inline the fix direction, or "open — the lane designs it" |
| Filing from memory of an old analysis | re-read the cited lines now; drop what is already fixed |
| One ticket for a whole feature | parent + landable children |
| A ticket per dead helper or doc typo | fold it into its area's FULL ticket as a Scope item |
| Scope without Out of scope | name the neighbours, so parallel lanes stay apart |
| Creating before the user saw the preview | preview table, then wait |
| Re-filing an existing problem | search first; link or drop |
