---
name: writing-plans
description: Use when you have a spec or requirements for a multi-step task, before touching code
---

# Writing Plans

## Overview

Write comprehensive implementation plans assuming the engineer has zero context for our codebase and questionable taste. Document everything they need to know: which files to touch for each task, code, testing, docs they might need to check, and how to verify the work. Give them the whole plan as bite-sized tasks. DRY. YAGNI. TDD.

Assume they are a skilled developer, but know almost nothing about our toolset or problem domain. Assume they don't know good test design very well.

**Announce at start:** "I'm using the writing-plans skill to create the implementation plan."

**Context:** If working in an isolated worktree, it should have been created by your worktree workflow at execution time.

**Save plans to:** `docs/superpowers/plans/YYYY-MM-DD-<feature-name>.md`
- (User preferences for plan location override this default)

## Workflow Order (CRITICAL)

**You MUST follow this exact sequence:**

1. **Research** - Gather codebase context, read existing patterns
2. **Write the plan** - Draft the full implementation plan, self-review it, and save it to `docs/superpowers/plans/`
3. **Oracle review** - ONLY AFTER the plan is written, consult the oracle for review
4. **Present findings** - Show oracle feedback to the user
5. **Execution handoff** - Offer execution options

**NEVER consult the oracle before the plan is written.** The oracle reviews a concrete artifact, not abstract ideas. Write first, review second.

## Plan Document Chunking (CRITICAL)

**The plan document itself MUST be written in chunks.** Plans are often 150+ lines. Writing them in a single `create_file` call risks truncation, hallucinated content, or context exhaustion.

**Pattern:**
1. **First chunk:** `create_file` with the header, goal, architecture, tech stack, file structure, and the first 1-2 tasks
2. **Subsequent chunks:** `edit_file` to append remaining tasks, one or two at a time
3. **Final chunk:** `edit_file` to append the self-review and handoff sections

**Rules:**
- **Each chunk:** ~60-100 lines of plan content
- **Append point:** Use end-of-file appending - each `edit_file` targets the last lines of the previous chunk
- **Verify after each chunk:** Read the file to confirm content was written correctly before appending more
- **Never write the entire plan in one shot** if it exceeds ~80 lines

## Scope Check

If the spec covers multiple independent subsystems, it should have been broken into sub-project specs during brainstorming. If it wasn't, suggest breaking this into separate plans — one per subsystem. Each plan should produce working, testable software on its own.

## File Structure

Before defining tasks, map out which files will be created or modified and what each one is responsible for. This is where decomposition decisions get locked in.

- Design units with clear boundaries and well-defined interfaces. Each file should have one clear responsibility.
- You reason best about code you can hold in context at once, and your edits are more reliable when files are focused. Prefer smaller, focused files over large ones that do too much.
- Files that change together should live together. Split by responsibility, not by technical layer.
- In existing codebases, follow established patterns. If the codebase uses large files, don't unilaterally restructure - but if a file you're modifying has grown unwieldy, including a split in the plan is reasonable.

This structure informs the task decomposition. Each task should produce self-contained changes that make sense independently.

## Task Right-Sizing

A task is the smallest unit that carries its own test cycle and is worth a
fresh reviewer's gate. When drawing task boundaries: fold setup,
configuration, scaffolding, and documentation steps into the task whose
deliverable needs them; split only where a reviewer could meaningfully
reject one task while approving its neighbor. Each task ends with an
independently testable deliverable.

## Step Granularity

**Each step is one action with a checkable result:**
- "Write the failing test" - step
- "Run it to make sure it fails" - step
- "Implement the minimal code to make the test pass" - step
- "Run the tests and make sure they pass" - step
- "Commit" - step

## Plan Document Header

**Every plan MUST start with this header:**

```markdown
# [Feature Name] Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use subagent-driven-development (recommended) with dispatching-parallel-agents for independent tasks to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** [One sentence describing what this builds]

**Architecture:** [2-3 sentences about approach]

**Tech Stack:** [Key technologies/libraries]

**Spec:** [path to the spec/design doc this plan implements — the plan
argues from the spec, so the spec travels with it; executors read both]

## Global Constraints

[The spec's project-wide requirements — version floors, dependency limits,
naming and copy rules, platform requirements — one line each, with exact
values copied verbatim from the spec. Every task's requirements implicitly
include this section.]

## Review Focus

[The five input classes or failure modes the spec implies but no task's
tests exercise that are most likely to bite a person using this software
— one line each, naming the input or condition and the behavior a
reasonable person would expect, most likely first. The spec is a vision
document: it says what the software must do, not everything it will
meet, and its silence on an input is not permission for that input to
break the program. Write the list here, once, with the spec in front of
you. Then, for each line, add the test that pins it to the task that
owns the code, in that task's own step style.]

---
```

## Task Structure

````markdown
### Task N: [Component Name]

**Files:**
- Create: `exact/path/to/file.py`
- Modify: `exact/path/to/existing.py:123-145`
- Test: `tests/exact/path/to/test.py`

**Interfaces:**
- Consumes: [what this task uses from earlier tasks — exact signatures]
- Produces: [what later tasks rely on — exact function names, parameter
  and return types. A task's implementer sees only their own task; this
  block is how they learn the names and types neighboring tasks use.]

- [ ] **Step 1: Write the failing test**

```python
def test_specific_behavior():
    result = function(input)
    assert result == expected
```

- [ ] **Step 2: Run test to verify it fails**

Run: `pytest tests/path/test.py::test_name -v`
Expected: FAIL with "function not defined"

- [ ] **Step 3: Implement `function(input: InputType) -> ResultType` in `exact/path/to/file.py`**

One line on the approach when the signature and the test leave a choice
(which library call, which data structure); a code block only for an
algorithm they do not determine.

- [ ] **Step 4: Run test to verify it passes**

Run: `pytest tests/path/test.py::test_name -v`
Expected: PASS
````

## What a Step Contains

A step is done when the implementer can write exactly one reasonable thing
from it. That is the whole requirement: unambiguous, not complete. Each kind
of step carries what makes it unambiguous and nothing more:

- **A test step:** the test's name and its assertions, as code, with the
  spec's exact values in them.
- **A code step:** the exact signature (name, parameters, return type), the
  file it lives in, and the specific values the spec pins. The implementer
  writes the body. A body appears only for an algorithm the signature and
  tests do not determine, or for exact copy the spec fixes.
- **A verification step:** the command to run and the output that means it
  passed.
- **A reference to another task:** that task's Interfaces block says what
  to use; the plan does not repeat that task's code.

A plan is the set of decisions the implementer cannot make alone. A plan
longer than the code it describes has written the code instead. Lines that
decide nothing ("TBD", "handle edge cases", "add appropriate validation",
"write tests for the above", a type or function no task defines) are the
opposite failure, and the self-review catches both.

## Chunked File Writing

**Large files MUST be written in chunks.** Never put 200+ lines of code in a single `create_file` block. Instead, break it into an initial `create_file` followed by `edit_file` steps.

**Why:** Agents have output limits. A single massive code block risks truncation, hallucinated endings, or context exhaustion. Chunking also makes each step reviewable and debuggable.

**Pattern:**

````markdown
- [ ] **Step N: Create file with initial structure**

Create: `src/features/catalog/product-list.tsx`

```tsx
import { useState } from "react"
import type { Product } from "@klakklik/api-contracts/schemas"

interface ProductListProps {
  products: Product[]
}

export function ProductList({ products }: ProductListProps) {
  const [filter, setFilter] = useState("")

  return (
    <div>
      {/* filtering UI - next step */}
      {/* product grid - step after */}
    </div>
  )
}
```

- [ ] **Step N+1: Add filtering UI**

Edit `src/features/catalog/product-list.tsx` - replace the `{/* filtering UI - next step */}` placeholder:

```tsx
<input
  type="text"
  value={filter}
  onChange={(e) => setFilter(e.target.value)}
  placeholder="Search products..."
/>
```

- [ ] **Step N+2: Add product grid**

Edit `src/features/catalog/product-list.tsx` - replace the `{/* product grid - step after */}` placeholder:

```tsx
<ul>
  {products
    .filter((product) => product.name.includes(filter))
    .map((product) => (
      <li key={product.id}>{product.name}</li>
    ))}
</ul>
```
````

**Rules:**
- **First chunk:** `create_file` with imports, types, and skeleton structure (placeholders for sections coming next)
- **Subsequent chunks:** `edit_file` targeting a specific placeholder or insertion point - always show the `old_str` -> `new_str` replacement
- **Chunk size:** ~30-80 lines per step. Enough to be meaningful, small enough to review
- **Placeholders:** Use descriptive comments such as `{/* filtering UI - next step */}` so the edit target is unambiguous
- **Each chunk is a checkbox step** - reviewable and testable where possible

**When to chunk vs. write whole:**
- **< 80 lines:** Write the whole file in one `create_file` step
- **80-150 lines:** Use judgment - chunk if logically separable
- **> 150 lines:** Always chunk

## Self-Review

After writing the complete plan, look at the spec with fresh eyes and check the plan against it. This is a checklist you run yourself — not a subagent dispatch.

- [ ] **1. Spec coverage:** Skim each section/requirement in the spec. Can you point to a task that implements it? List any gaps.

- [ ] **2. Step scan:** Every step must let the implementer write exactly one reasonable thing, and no step may carry more than that: a line that decides nothing is a gap, a function body the signature and tests already determine is a transcript. Fix both.

- [ ] **3. Type consistency:** Do the types, method signatures, and property names you used in later tasks match what you defined in earlier tasks? A function called `clearLayers()` in Task 3 but `clearFullLayers()` in Task 7 is a bug.

- [ ] **4. Review Focus:** For each input class or failure mode the spec implies, is there a task whose tests exercise it? The five uncovered ones most likely to bite a person go in the Review Focus section, and each line there gets its test added to the owning task. An empty section means you checked and found none, not that you skipped the check.

- [ ] **5. Proportion:** Compare the plan's length to the spec's. A plan several times longer than the spec it implements is a transcript of the program, not a plan. If code blocks are most of the document, replace bodies with signatures, test names and assertions, and check that each step is still unambiguous.

If you find issues, fix them inline. No need to re-review — just fix and move on. If you find a spec requirement with no task, add the task before Oracle review.

## Remember

- Exact file paths always
- Complete code in every step - if a step changes code, show the code
- Exact commands with expected output
- Reference relevant skills by name with explicit requirement markers
- DRY, YAGNI, TDD

## Oracle Review (Mandatory - AFTER Plan Is Written)

**ONLY after the plan is fully written, self-reviewed, and saved, ask the oracle to review it.** Do NOT consult the oracle during planning or before the plan document exists.

Use the oracle tool with:
- **task:** "Review this implementation plan for completeness, correctness, and potential issues"
- **files:** The saved plan file path
- **context:** The original spec/requirements that drove the plan

For a reusable reviewer prompt, see `plan-document-reviewer-prompt.md`.

**Present oracle findings to the user.** Do NOT automatically update the plan - let the user decide which issues to address.

## Execution Handoff

After the Oracle review findings have been presented, offer execution choice:

```markdown
Plan complete and saved to `docs/superpowers/plans/<filename>.md`. Two execution options:

**1. Subagent-Driven (recommended)** - I dispatch a fresh subagent per task, review between tasks, and use parallel execution for independent tasks

**2. Inline Execution** - Execute tasks in this session, one checkpointed batch at a time

Which approach?
```

**If Subagent-Driven chosen:**
- **REQUIRED SUB-SKILL:** Use subagent-driven-development
- **REQUIRED SUB-SKILL:** Use dispatching-parallel-agents to run independent tasks concurrently
- Stay in this session
- Fresh subagent per task plus review between tasks

**If Inline Execution chosen:**
- Stay in this session
- Execute task-by-task with checkpoints for review
- Use dispatching-parallel-agents only when tasks are independent and safe to run concurrently
