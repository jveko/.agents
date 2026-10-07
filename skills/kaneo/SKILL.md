---
name: kaneo
description: Use for any Kaneo ticket work - turning gap-analysis findings, review results or brainstormed feature ideas into self-contained, right-sized tickets (verified evidence, scope, acceptance, links, deduplicated, previewed before creation), and working a ticket from start to In Review by any agent (a lane, a solo session, or by hand) - moving it, recording its commits on it, never closing it.
---

# Kaneo

## Overview

Kaneo is the tracker; the board is To Do → In Progress → In Review → Done. This skill covers **writing tickets** and **working them**, whoever does the work — a single agent session, you by hand, or parallel lanes.

**Optional: `herdr-worktrunk`.** To run tickets as parallel lanes (a batch, or a whole backlog in waves), load it alongside this skill: it supervises the lanes, this skill handles their tickets. Without it, a single session works a ticket exactly as described below.

**A ticket is its worker's whole scope.** Whoever works it later gets the ticket and nothing else — no session, no report, no memory of the analysis that produced it. Everything needed to stay in scope must be IN the ticket, verified against the code at the time it was written.

Observed failure this prevents: tickets that said "Fix: see report section" and pointed at a `local://` report from an expired session. The lanes couldn't read it, so supervisors guessed the fix — one guess deleted 66 public constants.

## Inputs

- **Findings** (gap analysis, audits, review results): problems with evidence.
- **Ideas** (brainstorming what to add or extend): goals without evidence of a defect. They go straight to tickets — whoever works one designs it (FULL track: superpowers brainstorming first).

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

**Fix direction:** <the known fix> | open — designed when worked

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

1. **Self-contained.** Never "see report", a `local://` / session / artifact link, or "as discussed". Inline the evidence and the fix direction, or write "open — designed when worked".
2. **Verify before writing.** Re-read every cited line against the current target branch; quote it; record the commit. A finding the code no longer shows is dropped (report it as already fixed), never filed.
3. **One ticket = one landable change** — one lane can implement and land it with its tests. Larger findings or features become a **parent** ticket (the goal, the shared context, the list of children) plus **children**, each landable; link them with the tracker's subtask relation (parent → child) and `blocks` where order matters.
4. **Scope is explicit both ways.** Name what changes AND what must not. Whoever works it treats anything outside Scope as a question, so a vague Scope stalls it and a wide one licenses drift.
5. **Track hint**: LIGHT only when no behaviour changes (docs, comments, config, CI, ignore rules, removing provably unused non-public code); everything else FULL. A feature is always FULL.
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

## Working a ticket

Commits carry no ticket IDs, so **the ticket is where its commits are recorded**. Before every state change, re-read the ticket by key (`get_task_by_ticket_id`) and confirm its title matches the work; re-read after the change to verify.

| When | On the ticket |
|---|---|
| Work starts | move to **In Progress**; comment who works it and where (`lane <name>, branch <branch>, track <LIGHT\|FULL>` for a lane; `session, branch <branch>` otherwise). In Progress is also the lock: never start a ticket someone else has In Progress |
| Work lands on the target | comment the landed commits (`<sha> <subject>` per line), the gate result, and a short summary with any deviation from the ticket; then move to **In Review** |
| Work stops without landing | comment why (question pending, superseded, refused) and move back to **To Do** |
| Done | never the worker's — the user moves it after reviewing and pushing |

Problems noticed outside the ticket's scope are never fixed in passing: note them in the landing comment and hand them to the user as findings for new tickets (Process above), linked `related` to this one.

## Common mistakes

| Mistake | Fix |
|---|---|
| "Fix: see report section" | inline the fix direction, or "open — designed when worked" |
| Filing from memory of an old analysis | re-read the cited lines now; drop what is already fixed |
| One ticket for a whole feature | parent + landable children |
| A ticket per dead helper or doc typo | fold it into its area's FULL ticket as a Scope item |
| Scope without Out of scope | name the neighbours, so parallel lanes stay apart |
| Creating before the user saw the preview | preview table, then wait |
| Re-filing an existing problem | search first; link or drop |
| Changing a state from memory or a shortlist | re-read by key, confirm the title, change, re-read |
| Landing without a comment | the commit list on the ticket is the only link from ticket to code |
| Moving a ticket to Done | Done is the user's, after review and push |
