---
name: creating-context-capsule
description: コンテキストリセット前にセッション状態（capsule/plan/tests）を保存し、バリデーションを通す手順を定義する。コンテキストが肥大化した時やハンドオフ時に使う。
skills: [booting-sub-agent]
---

# CREATE_CONTEXT_CAPSULE

## Purpose

This skill defines the workflow for generating a **Context Capsule** before clearing context. A Context Capsule preserves the essential session state, implementation plan, and test criteria to enable seamless continuation after a context reset.

---

## When to Use

- Before context becomes too large and needs to be cleared
- When handing off work to another agent or session
- At natural breakpoints in long-running tasks
- When explicitly requested by the user or Antigravity

---

## Workflow

### Step 1: Generate `capsule.md`

Create a high-level summary of the current session state.

**Location:** `.agent/handover/capsule.md`

**Required Sections:**

```markdown
# Session Capsule

## Session ID
[Timestamp or unique identifier]

## Current State
[Brief description of where we are in the task]

## Completed Work
- [List of completed items]

## In Progress
- [List of items currently being worked on]

## Pending
- [List of remaining items]

## Key Decisions Made
- [Important decisions and their rationale]

## Blockers / Open Questions
- [Any unresolved issues]

## Files Modified This Session
- [List of changed files]
```

---

### Step 2: Generate `plan.md`

Create a detailed implementation specification following the **HANDOVER_CONTRACT** structure.

**Location:** `.agent/handover/plan.md`

**Required Structure (HANDOVER_CONTRACT):**

```markdown
# Implementation Plan

## Metadata
- **Task ID:** [Unique identifier]
- **Created:** [Timestamp]
- **Author:** [Agent identifier]
- **Status:** [Draft | Validated | Ready]

## Objective
[Clear, concise statement of what needs to be achieved]

## Context
[Background information necessary to understand the task]

## Technical Specification

### Architecture
[System design and component relationships]

### Implementation Steps
1. [Step with specific file paths and code changes]
2. [Step with specific file paths and code changes]
...

### Dependencies
- [External dependencies]
- [Internal dependencies]

### Constraints
- [Technical constraints]
- [Business constraints]

## Acceptance Criteria
- [ ] [Criterion 1]
- [ ] [Criterion 2]
...

## Risk Assessment
| Risk | Impact | Mitigation |
|------|--------|------------|
| ... | ... | ... |

## Rollback Plan
[Steps to revert changes if needed]
```

---

### Step 3: Generate `tests.md`

Define specific test cases and acceptance criteria.

**Location:** `.agent/handover/tests.md`

**Required Structure:**

```markdown
# Test Cases & Acceptance Criteria

## Unit Tests

### [Test Suite Name]
| Test Case | Input | Expected Output | Priority |
|-----------|-------|-----------------|----------|
| ... | ... | ... | P0/P1/P2 |

## Integration Tests

### [Integration Scenario]
- **Setup:** [Required state]
- **Steps:**
  1. [Action]
  2. [Action]
- **Expected Result:** [Outcome]

## Acceptance Criteria Checklist

### Functional
- [ ] [Criterion with specific, measurable outcome]

### Non-Functional
- [ ] [Performance criterion]
- [ ] [Security criterion]

## Edge Cases
- [ ] [Edge case 1]
- [ ] [Edge case 2]

## Regression Tests
- [ ] [Existing functionality that must not break]
```

---

### Step 4: Validate the Plan

**MANDATORY:** Run the validation script to ensure `plan.md` meets the HANDOVER_CONTRACT.

```bash
npx ts-node scripts/validate-handover.ts .agent/handover/plan.md
```

---

### Step 5: Handle Validation Result

#### If Validation FAILS:

1. **DO NOT proceed** to the completion message
2. Read the validation errors carefully
3. Fix the identified issues in `plan.md`
4. Re-run validation: `npx ts-node scripts/validate-handover.ts .agent/handover/plan.md`
5. Repeat until validation passes

**Example fixes:**
- Missing sections → Add required sections
- Incomplete criteria → Add specific, measurable criteria
- Vague steps → Add file paths and concrete code changes

#### If Validation PASSES:

Proceed to Step 6.

---

### Step 6: Output Completion Message

**Only after validation passes**, output:

```
✅ Context Capsule Ready - Safe to Clear Context

Generated files:
- .agent/handover/capsule.md
- .agent/handover/plan.md
- .agent/handover/tests.md

Validation: PASSED
```

---

## File Structure

After successful execution:

```
.agent/
  handover/
    capsule.md    # Session state summary
    plan.md       # Implementation specification
    tests.md      # Test cases and criteria
```

---

## Validation Loop Flowchart

```
+---------------------------+
|   Generate capsule.md     |
+-------------+-------------+
              |
              v
+---------------------------+
|   Generate plan.md        |
+-------------+-------------+
              |
              v
+---------------------------+
|   Generate tests.md       |
+-------------+-------------+
              |
              v
+---------------------------+
|   Run validate-handover   |
+-------------+-------------+
              |
              v
         +--------+
         | Pass?  |
         +---+----+
        NO   |   YES
    +--------+--------+
    |                 |
    v                 v
+-------+    +------------------------+
| Fix   |    | Output:                |
| plan  |    | "Context Capsule       |
+---+---+    |  Ready - Safe to       |
    |        |  Clear Context"        |
    +------->+------------------------+
   (retry)
```

---

## Important Rules

1. **Never skip validation** - The validation step is mandatory
2. **Never output completion message before validation passes**
3. **Keep capsule.md concise** - It's a summary, not a full log
4. **Keep plan.md detailed** - It must be self-contained for continuation
5. **Keep tests.md specific** - Vague criteria are not acceptable
6. **All three files must be generated** - Partial capsules are invalid

---

## Example Usage

When context is becoming large:

```
Agent: I notice context is getting large. Initiating Context Capsule generation...

[Generates capsule.md]
[Generates plan.md]
[Generates tests.md]
[Runs validation]

Agent: ✅ Context Capsule Ready - Safe to Clear Context

Generated files:
- .agent/handover/capsule.md
- .agent/handover/plan.md
- .agent/handover/tests.md

Validation: PASSED
```
