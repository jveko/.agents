# Lane brief skeleton

Write to `/tmp/<lane>.txt` (outside the worktree — never dirty `git status`), deliver with a one-line pointer via `herdr agent prompt`. Angle-bracket slots are yours to fill; keep every claim file:line-grounded.

---

YOU ARE THE LANE ORCHESTRATOR for <lane N (<area>)>, branch <branch>, worktree <absolute path> (your cwd). No human in the loop — this brief is the AUTHORITATIVE scope (<ticket MUT-NN + review-delta comments, inlined / review findings, inlined>). Work the queue IN ORDER; each item is its own independently-green commit series before the next begins.

SCOPE QUEUE (ordered):
<item 1> — <what + why, with path:line evidence for every claim>. Files: <affected paths>.
<item 2> — <…>
DEFERRED (NOT yours — owned by other lanes/waves): <explicit exclusions, each with its owner, so the lane never wanders into a file collision>.

HARD RULES: <repo invariants — e.g. no sync guards across .await; DB-first mutation spine; no second conventions beside existing ones; scoped gates only; ticket IDs in commit subjects are fine but NEVER in code comments; never merge, never push.>

QUESTIONS: never ask a human. Scope is answered above — no re-scope, no expansion. A genuine unforeseen decision → stop that sub-thread, state ONE precise question with your options, continue any independent work; your supervisor watches this pane and will answer.

SKILLS (discover and load these; each phase MUST use its named skill): brainstorming — phase 1, adapted: NO questions to anyone, decisions recorded internally · dispatching-parallel-agents — phase 2's single parallel batch (codebase scouts + real-world librarian) · writing-plans — phase 3 plan structure · subagent-driven-development — phase 5 execution · systematic-debugging — any failure diagnosis during implementation. Repo AGENTS.md files apply on top throughout.

PIPELINE (whole queue, in order):
1. BRAINSTORM (internal; skill: brainstorming but WITHOUT interrogating anyone): 2-3 candidate designs with tradeoffs per item, pick one — decisions and rejected alternatives are recorded later in the design doc.
2. PARALLEL RESEARCH — ONE batch of parallel read-only agents, BOTH kinds:
   - SCOUTS on the codebase: (a) verify every cited line against current source (lines drift), (b) map callers/invariants/existing tests around each target file, (c) find the in-repo pattern each fix must reuse.
   - LIBRARIAN on the real world: (d) verify every external-contract claim the design will rely on against authoritative sources — upstream crate docs/source, kernel/UAPI headers, RFCs, official CLI/tool docs — returning source-verified answers with citations (what the API really returns, which constants/flags actually exist, real error semantics). An asserted-but-wrong external fact poisons the design: the 2026-09-24 review's sk_msg SK_REDIRECT claim was false (enum sk_action has no such member, per /usr/include/linux/bpf.h) and surfaced only when a lane checked reality.
   All research agents skip formatters, linters, and project-wide tests; librarian findings feed the Design-decisions section exactly like scout findings.
3. DESIGN + PLAN, committed on this branch: docs/plans/specs/<YYYY-MM-DD>-<slug>-design.md (problem, chosen design, alternatives, invariants, test strategy, explicit Design-decisions section) and docs/plans/<YYYY-MM-DD>-<slug>.md (skill: writing-plans — small tasks, each independently green, exact files, observable acceptance criteria, scoped verification commands). Commit subjects: imperative, lowercase; ticket ID allowed in the subject, never in code comments.
4. COLD-REVIEW GATE (auto-approved — no human wait): spawn a FRESH, context-free reviewer subagent whose ONLY inputs are the two docs + this brief; instruction: "verify every claim against actual source in this worktree, cite file:line, verdict READY | READY-WITH-FIXES | NOT-READY". READY → proceed; READY-WITH-FIXES → apply every fix, re-review once; NOT-READY → apply fixes, re-review once, and if still NOT-READY STOP and return the verdict as your final report. The verdict IS the approval. For any re-review (round ≥ 2), tell the reviewer which findings prior rounds raised and that they were addressed: its verdict weighs (a) whether those fixes genuinely landed and (b) NEW material defects only — re-litigating an addressed finding never counts against the verdict, nits/style never change it, and NOT-READY requires at least one new material defect. Otherwise rounds converge on fresh nit volume instead of plan quality. A second NOT-READY hands the stop upward to the supervisor, who either amends the gate (apply the findings + one further round) or re-queues the lane fresh — record which in the final report.
5. IMPLEMENT (skill: subagent-driven-development): plan order; Tester subagent writes the failing test first, implementer subagent greens it, you run the scoped gate and commit before the next task. Scoped gates only (e.g. `just check-crate <crate>`, targeted `cargo nextest -E '<filter>'`, `just lint`) — never project-wide cold CI mid-flight. No stubs/TODOs/placeholders; migrate every caller; delete what the change obsoletes.
6. FINAL REPORT: commit list, verbatim gate/test output as evidence, plan-vs-actual deviations with reasons. Do NOT merge, do NOT push — the supervisor lands the branch with `wt merge --no-squash --no-remove`.

ENV NOTES: post-start warm-up (deps install, artifact symlinks, dist stub) may still be finishing — if a build dependency is missing, wait ~30s and retry once before reporting a blocker. Root-requiring steps may be unavailable — attempt once and record the outcome honestly.

---

Usage notes:
- The brief is authoritative over ticket text it summarizes; cross-checking tickets is allowed, conflicting brief wins.
- Include whether repo MCP config is present in the worktree (it may be — do not assume absence).
- Deferred-items line is load-bearing: it is what keeps parallel lanes off shared files.
