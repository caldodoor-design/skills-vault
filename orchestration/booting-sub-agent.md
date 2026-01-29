---
name: booting-sub-agent
description: Git Worktreeを使ってSubAgentを物理隔離し、Context Capsuleを渡して並列実行する手順を定義する。タスクの並列化やコンテキスト分離が必要な時に使う。
skills: [creating-context-capsule]
---

# BOOT_SUB_AGENT (v2.0)

## Purpose

This skill defines the workflow for **Booting a SubAgent** with a fresh context. It uses **Git Worktrees** for physical isolation and **Claude Tasks** for state synchronization, enabling safe parallel execution.

---

## When to Use

- When starting a new Task (Phase 4+)
- When parallelizing work across multiple SubAgents
- When context needs to be cleared but work must continue

---

## Workflow

### Step 1: Prerequisite Check

**MANDATORY:** Verify the Context Capsule exists before proceeding.

**Required Files:**

| File | Location | Purpose |
|------|----------|---------|
| `capsule.md` | `.agent/handover/capsule.md` | Session state summary |
| `plan.md` | `.agent/handover/plan.md` | Implementation specification |
| `tests.md` | `.agent/handover/tests.md` | Test cases and acceptance criteria |

**Task Association:**
Identify the Task ID this agent will work on (e.g., `T-101`).

```bash
# Validate plan.md against HANDOVER_CONTRACT
npx ts-node scripts/validate-handover.ts .agent/handover/plan.md
```

---

### Step 2: Isolation Setup (Worktree)

To prevent file conflicts during parallel execution, create a dedicated workspace.

```powershell
# Generate unique Agent ID
$AgentId = "agent-" + (Get-Date -Format "yyyyMMdd-HHmmss")
$WorktreePath = ".agent/worktrees/$AgentId"

# Detect default branch robustly
try {
    # Try to get the upstream HEAD
    $DefaultBranch = (git symbolic-ref refs/remotes/origin/HEAD 2>$null) -replace 'refs/remotes/origin/', ''
} catch {
    $DefaultBranch = $null
}

if (-not $DefaultBranch) {
    # Fallback: check local branches
    if (git show-ref --verify --quiet refs/heads/main) { $DefaultBranch = "main" }
    elseif (git show-ref --verify --quiet refs/heads/master) { $DefaultBranch = "master" }
    else { Write-Error "Could not detect default branch (main/master)."; exit 1 }
}

# Create git worktree
git worktree add -b "subagent/$AgentId" "$WorktreePath" $DefaultBranch

# Copy Capsule to worktree (Capsule is untracked or ignored)
Copy-Item -Recurse .agent/handover/ "$WorktreePath/.agent/"
```

**Why Worktrees?** allows multiple agents to edit files without git index locking issues.

---

### Step 3: Boot Sequence

#### Manual Boot (Standard)

**Instructions for User:**

```
1. Open a NEW Terminal window (Do NOT exit current session if parallel)
2. Navigate to the worktree:
   > cd .agent/worktrees/agent-<timestamp>
   
3. Start Claude with Task Context:
   > $env:CLAUDE_TASK_ID="<TASK_ID>"
   > claude
   
4. Paste the Initial Prompt (see Step 4)
```

**Orchestration Agent Output:**

```
┌─────────────────────────────────────────────────────────────┐
│ 🚀 SubAgent Boot Required (Parallel Mode)                   │
├─────────────────────────────────────────────────────────────┤
│ Task ID: T-xxx                                              │
│ Worktree: .agent/worktrees/agent-1234567890                 │
├─────────────────────────────────────────────────────────────┤
│ ACTION REQUIRED:                                            │
│ 1. Open NEW Terminal                                        │
│ 2. cd .agent/worktrees/agent-1234567890                     │
│ 3. $env:CLAUDE_TASK_ID="T-xxx"                              │
│ 4. claude                                                   │
│ 5. Paste Initial Prompt below                               │
└─────────────────────────────────────────────────────────────┘
```

---

### Step 4: Initial Prompt

**Template:**

```markdown
# SubAgent Initialization (v2.0)

You are a SubAgent working in a dedicated Worktree.
Target Task: **{{TASK_ID}}**

## Skill Mounting

- `thinking/`
- `domain-guard/`
- `domain-work/`
(NO `orchestration/` skills)

## Context Configuration

1. **Root Directory:** You are in a git worktree. Treat this as your root.
2. **Task Context:** You are responsible for Task `{{TASK_ID}}`.
3. **Capsule Location:** `.agent/handover/capsule.md` (Local copy)

## Mission

1. Read `.agent/handover/capsule.md` to understand background.
2. Read `.agent/handover/plan.md` to identify your specific steps.
3. Execute the work for Task `{{TASK_ID}}`.
4. **Important:** When finished, do NOT merge. Report completion.
   - Run `tests.md` validations.
   - Run `scripts/hook-heavy.ps1` for self-check.

## Completion Report

```
## SubAgent Completion Report
### Task {{TASK_ID}}
- Status: [Success/Failure]
- Worktree Branch: subagent/agent-xxxx
- Validation: [Pass/Fail]
```
```

---

### Step 5: Post-Execution Merge (Orchestrator)

When SubAgent reports success:

1. Orchestrator enters the worktree or fetches the branch.
2. Reviews the changes (Codex Review).
3. Merges the branch to `master` (or main dev branch).
4. Deletes the worktree:
   ```powershell
   git worktree remove .agent/worktrees/agent-xxxx
   git branch -D subagent/agent-xxxx
   ```

---
