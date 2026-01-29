---
name: completing-task-with-contract
description: タスク完了前に品質ゲート（Lightweight/Heavyweight）を通過させ、未検証のまま完了させないための手順を定義する。タスクを完了としてマークする直前に使う。
skills: [asking-codex-review]
---

# COMPLETE_TASK_WITH_CONTRACT

## Purpose

To enforce strict quality gates before marking a task as "Complete" in the Agent OS. This prevents premature closure of tasks that haven't met the contract requirements.

---

## When to Use

- When you believe a Task is finished.
- BEFORE running `claude task complete`.

---

## Workflow

### Step 1: Identify Task Type

Determine the nature of the task to select the appropriate "Gate Level".

- **Type A: Documentation / Design / Analysis**
  - Changes: Markdown files only, or no file changes.
  - Gate: **Lightweight Gate**

- **Type B: Code Implementation**
  - Changes: Source code (`.ts`, `.js`, `.py`, `.ps1`), Config files.
  - Gate: **Heavyweight Gate**

---

### Step 2: Run Validation Gate

#### Lightweight Gate (Type A)

```bash
# 1. Validate Handover Contract
npx ts-node scripts/validate-handover.ts .agent/handover/plan.md

# 2. Check for "TODO" or "WIP" markers in changed files
# (Manual check by agent)
```

#### Heavyweight Gate (Type B)

```powershell
# 1. Validate Handover Contract (Must be valid)
npx ts-node scripts/validate-handover.ts .agent/handover/plan.md

# 2. Run Light Hook (Syntax/Type check)
powershell -File .\scripts\hook-light.ps1

# 3. Run Skill Contamination Check
npx ts-node scripts/validate-skill-usage.ts

# 4. Run Heavy Hook (Codex Deep Review)
# CRITICAL: Address ALL issues raised by Codex.
powershell -File .\scripts\hook-heavy.ps1
```

---

### Step 3: Verification Result

#### ❌ Rejection (If ANY check fails)

If any command returns a non-zero exit code or Codex reports issues:

1. **DO NOT** complete the task.
2. Fix the reported issues.
3. Re-run Step 2.

#### ✅ Approval (If ALL checks pass)

1. Update `task.md` (Legacy Tracker)
   - Mark the item as `[x]`.

2. Update Telemetry
   - Log the success event.
   ```powershell
   npx ts-node scripts/telemetry.ts log success "Task <ID> Completed via Contract Gate"
   ```

3. **Complete the Task in OS**
   ```powershell
   claude task complete <TASK_ID>
   ```

---

## Example Usage

**Agent:** "I have finished implementing the user login feature."

**Self-Correction:** "Wait, I must verify against the contract before closing."

**Action:**
```powershell
# It's a code change, so Heavy Gate
npx ts-node scripts/validate-handover.ts .agent/handover/plan.md
powershell -File .\scripts\hook-light.ps1
powershell -File .\scripts\hook-heavy.ps1
```

**Result:**
`hook-heavy.ps1` returns "No issues found".

**Completion:**
```powershell
npx ts-node scripts/telemetry.ts log success "Task T-101 Completed"
claude task complete T-101
```
