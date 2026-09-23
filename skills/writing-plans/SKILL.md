---
name: writing-plans
description: Use when you have a spec or requirements for a multi-step task, before touching code
---

# Writing Plans

## Overview

Write comprehensive implementation plans assuming the engineer has zero context for our codebase and questionable taste. Document everything they need to know: which files to touch for each task, code, testing, docs they might need to check, and how to verify the work. Give them the whole plan as bite-sized tasks. DRY. YAGNI. TDD.

Assume they are a skilled developer, but know almost nothing about our toolset or problem domain. Assume they don't know good test design very well.

**Announce at start:** "I'm using the writing-plans skill to create the implementation plan."

**Save plans to:** `docs/plans/YYYY-MM-DD-<feature-name>.md`

## Workflow Order (CRITICAL)

**You MUST follow this exact sequence:**

1. **Research** - Gather codebase context, read existing patterns
2. **Write the plan** - Draft the full implementation plan, self-review it, and save it to `docs/plans/`
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

If the spec covers multiple independent subsystems, it should have been broken into sub-project specs during brainstorming. If it wasn't, suggest breaking this into separate plans - one per subsystem. Each plan should produce working, testable software on its own.

## File Structure

Before defining tasks, map out which files will be created or modified and what each one is responsible for. This is where decomposition decisions get locked in.

- Design units with clear boundaries and well-defined interfaces. Each file should have one clear responsibility.
- Prefer smaller, focused files over large ones that do too much. Plans are easier to execute when each file can be held in context.
- Files that change together should live together. Split by responsibility, not by technical layer.
- In existing codebases, follow established patterns. If the codebase uses large files, don't unilaterally restructure - but if a file you're modifying has grown unwieldy, including a split in the plan is reasonable.

This structure informs the task decomposition. Each task should produce self-contained changes that make sense independently.

## Bite-Sized Task Granularity

**Each step is one action (2-5 minutes):**
- "Write the failing test" - step
- "Run it to make sure it fails" - step
- "Implement the minimal code to make the test pass" - step
- "Run the tests and make sure they pass" - step

## Plan Document Header

**Every plan MUST start with this header:**

```markdown
# [Feature Name] Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use subagent-driven-development (recommended) with dispatching-parallel-agents for independent tasks to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** [One sentence describing what this builds]

**Architecture:** [2-3 sentences about approach]

**Tech Stack:** [Key technologies/libraries]

---
```

## Task Structure

````markdown
### Task N: [Component Name]

**Files:**
- Create: `exact/path/to/file.py`
- Modify: `exact/path/to/existing.py:123-145`
- Test: `tests/exact/path/to/test.py`

- [ ] **Step 1: Write the failing test**

```python
def test_specific_behavior():
    result = function(input)
    assert result == expected
```

- [ ] **Step 2: Run test to verify it fails**

Run: `pytest tests/path/test.py::test_name -v`
Expected: FAIL with "function not defined"

- [ ] **Step 3: Write minimal implementation**

```python
def function(input):
    return expected
```

- [ ] **Step 4: Run test to verify it passes**

Run: `pytest tests/path/test.py::test_name -v`
Expected: PASS
````

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

## No Placeholders

Every step must contain the actual content an engineer needs. These are **plan failures** - never write them:
- "TBD", "TODO", "implement later", "fill in details"
- "Add appropriate error handling" / "add validation" / "handle edge cases"
- "Write tests for the above" without actual test code
- "Similar to Task N" - repeat the code because the engineer may be reading tasks out of order
- Steps that describe what to do without showing how - code blocks required for code steps
- References to types, functions, or methods not defined in any task

## Self-Review

After writing the complete plan, look at the spec with fresh eyes and check the plan against it. This is a checklist you run yourself - not a subagent dispatch.

- [ ] **Spec coverage:** Skim each section/requirement in the spec. Can you point to a task that implements it? List any gaps.

- [ ] **Placeholder scan:** Search your plan for red flags - any of the patterns from the "No Placeholders" section above. Fix them.

- [ ] **Type consistency:** Do the types, method signatures, and property names you used in later tasks match what you defined in earlier tasks? A function called `clearLayers()` in Task 3 but `clearFullLayers()` in Task 7 is a bug.

If you find issues, fix them inline. If you find a spec requirement with no task, add the task before Oracle review.

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
Plan complete and saved to `docs/plans/<filename>.md`. Two execution options:

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
