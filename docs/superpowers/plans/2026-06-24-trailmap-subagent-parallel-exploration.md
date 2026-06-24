# Trailmap Subagent Parallel Exploration Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add Trailmap behavior rules for subagent-assisted parallel path exploration while preserving one main active path per topic.

**Architecture:** Extend the documentation-only Trailmap skill contract with an `agent_run` execution overlay, a new `subagent <key>` command, and `--subagent`, `--informed`, and `--allow-shared-code` flag behavior for new-path creation. Keep path lifecycle statuses unchanged and update bilingual documentation to explain command usage, risk confirmations, report handling, and read-view display.

**Tech Stack:** Markdown, Agent Skill instructions, PowerShell static checks, Git

---

## File Map

- Modify `trailmap/SKILL.md`: behavioral source of truth for data model, commands, confirmation rules, report schema, and read views.
- Modify `docs/USAGE.md`: detailed English usage guide for subagent parallel exploration.
- Modify `docs/USAGE.zh-CN.md`: detailed Chinese usage guide with equivalent behavior.
- Modify `README.md`: concise English product capability and command summary.
- Modify `README.zh-CN.md`: concise Chinese product capability and command summary.
- Verify `docs/superpowers/specs/2026-06-24-trailmap-subagent-parallel-exploration-design.md`: implementation source requirements.

### Task 1: Add Agent Run Model To Skill

**Files:**
- Modify: `trailmap/SKILL.md`
- Verify: `docs/superpowers/specs/2026-06-24-trailmap-subagent-parallel-exploration-design.md`

- [ ] **Step 1: Run the failing model contract**

```powershell
$skill = Get-Content -Raw trailmap/SKILL.md
foreach ($required in @(
  'agent_run',
  'subagent running',
  'running',
  'reported',
  'completed',
  'failed',
  'blocked',
  'cancelled'
)) {
  if (-not $skill.Contains($required)) { throw "Missing agent_run model marker: $required" }
}
```

Expected: fail before implementation because `agent_run` is not documented.

- [ ] **Step 2: Add `agent_run` to the data model**

In `trailmap/SKILL.md`, after the path status rules, add this section:

```markdown
Agent run fields:

```json
{
  "agent_run": {
    "status": "running",
    "mode": "subagent",
    "context_mode": "clean",
    "run_id": "optional-runtime-id",
    "started_at": "2026-06-24T10:00:00+08:00",
    "risk": "shared_workspace_code"
  }
}
```

Rules:

- `agent_run` is optional and represents execution state, not path lifecycle state.
- `agent_run.status` may be `running`, `reported`, `completed`, `failed`, `blocked`, or `cancelled`.
- `agent_run.mode` is `subagent` in v1.
- `agent_run.context_mode` is `clean` by default and may be `informed`.
- `run_id` is optional; record it only when the runtime provides one.
- Keep only the most recent `agent_run` at path top level; historical results belong in `updates`.
- Do not mark a subagent path as a second `active` path. The topic still has at most one main active path.
```

- [ ] **Step 3: Re-run the model contract**

```powershell
$skill = Get-Content -Raw trailmap/SKILL.md
foreach ($required in @(
  'agent_run',
  'execution state, not path lifecycle state',
  'running',
  'reported',
  'completed',
  'failed',
  'blocked',
  'cancelled',
  'Do not mark a subagent path as a second `active` path'
)) {
  if (-not $skill.Contains($required)) { throw "Missing agent_run model marker: $required" }
}
git diff --check -- trailmap/SKILL.md
```

Expected: exit `0`.

- [ ] **Step 4: Commit the model update**

```powershell
git add trailmap/SKILL.md
git commit -m "Add Trailmap agent run model"
```

### Task 2: Define Subagent Commands And Flags

**Files:**
- Modify: `trailmap/SKILL.md`

- [ ] **Step 1: Run the failing command contract**

```powershell
$skill = Get-Content -Raw trailmap/SKILL.md
foreach ($required in @(
  'subagent <key>',
  '--subagent',
  '--allow-shared-code',
  '--informed'
)) {
  if (-not $skill.Contains($required)) { throw "Missing subagent command marker: $required" }
}
```

Expected: fail before command documentation exists.

- [ ] **Step 2: Extend command parsing rules**

In `trailmap/SKILL.md`, under command parsing, add:

```markdown
Subagent flags:

- `--subagent` requests subagent exploration for newly created non-main-active paths.
- `--subagent <key>` or `--subagent <key1,key2>` selects specific newly created path keys for subagent exploration.
- `--allow-shared-code` means the user accepts the shared workspace code risk. It skips the separate risk prompt, not the Trailmap write-confirmation draft.
- `--informed` uses informed context for subagent exploration. Without it, subagent context is `clean`.
```

- [ ] **Step 3: Add `subagent <key>` command section**

Add a new command section before `resume`:

```markdown
### `subagent <key>`

Start subagent exploration for an existing path in the active topic.

Validation:

- Require an active topic.
- Require the target path to exist in the active topic.
- Reject the current main active path.
- Reject closed paths.
- Reject paths whose `agent_run.status` is already `running`.

Before writing, show a concise startup draft with the path key/title/status, context mode, planned `agent_run.status`, and shared workspace code risk. `--allow-shared-code` skips only the separate risk question; it does not skip the write-confirmation draft.

If the subagent tool is unavailable, do not fail the path operation. Write `agent_run.status: "blocked"` with a concise reason after confirmation.
```

- [ ] **Step 4: Extend no-subcommand and `pending` behavior**

Update the no-subcommand and `pending` sections to include:

```markdown
When new non-main-active paths are created, prompt whether to start subagent exploration for zero, one, or multiple candidate paths unless `--subagent` already selects them.

For `pending <idea> --subagent`, the new sibling path remains `pending` and receives `agent_run.status: "running"` after confirmation.
```

- [ ] **Step 5: Re-run the command contract**

```powershell
$skill = Get-Content -Raw trailmap/SKILL.md
foreach ($required in @(
  'subagent <key>',
  '--subagent <key1,key2>',
  '--allow-shared-code',
  'skips only the separate risk question',
  'pending <idea> --subagent',
  'new non-main-active paths'
)) {
  if (-not $skill.Contains($required)) { throw "Missing subagent command content: $required" }
}
git diff --check -- trailmap/SKILL.md
```

Expected: exit `0`.

- [ ] **Step 6: Commit the command behavior**

```powershell
git add trailmap/SKILL.md
git commit -m "Define Trailmap subagent commands"
```

### Task 3: Define Subagent Context, Report, And Confirmation Flow

**Files:**
- Modify: `trailmap/SKILL.md`

- [ ] **Step 1: Run the failing report contract**

```powershell
$skill = Get-Content -Raw trailmap/SKILL.md
foreach ($required in @(
  'path_key',
  'handoff',
  'shared_workspace_code',
  'reported'
)) {
  if (-not $skill.Contains($required)) { throw "Missing report flow marker: $required" }
}
```

Expected: fail until report flow is documented.

- [ ] **Step 2: Add subagent context modes**

After resume context modes, add:

```markdown
### Subagent Context Modes

Subagent context defaults to `clean`.

`clean` includes the topic title, target path `created_from`, target path description, target path updates, minimal current active path identity, shared workspace code risk, and required report schema. It does not include detailed reasoning from sibling paths.

`--informed` adds summarized sibling, parent, current-active, or previously explored path conclusions. Clearly label this material as "other-path context".
```

- [ ] **Step 3: Add subagent report handling**

Add:

```markdown
### Subagent Reports

Subagents must return:

```text
path_key
summary
conclusion
status_after
closed_as, only when status_after=closed
codechange.changed
codechange.files
codechange.summary
handoff
```

When a subagent returns, set `agent_run.status` to `reported` and show a normal `update <key>` draft. The user must confirm before writing the update, changing the path status, setting closure fields, or changing `agent_run.status` to `completed`.

If the user rejects the update draft, do not write the update, do not change code, and keep `agent_run.status` as `reported`.

Subagents may recommend `status_after: closed` and `closed_as`, but they may not directly close a path.
```

- [ ] **Step 4: Re-run the report contract**

```powershell
$skill = Get-Content -Raw trailmap/SKILL.md
foreach ($required in @(
  'Subagent Context Modes',
  'Subagent Reports',
  'path_key',
  'handoff',
  'agent_run.status` to `reported`',
  'keep `agent_run.status` as `reported`',
  'may not directly close a path'
)) {
  if (-not $skill.Contains($required)) { throw "Missing report flow content: $required" }
}
git diff --check -- trailmap/SKILL.md
```

Expected: exit `0`.

- [ ] **Step 5: Commit the report flow**

```powershell
git add trailmap/SKILL.md
git commit -m "Define Trailmap subagent report flow"
```

### Task 4: Update Read Views And Resume Warnings

**Files:**
- Modify: `trailmap/SKILL.md`

- [ ] **Step 1: Run the failing read-view contract**

```powershell
$skill = Get-Content -Raw trailmap/SKILL.md
foreach ($required in @(
  'subagent running',
  'pending / subagent running',
  'currently being explored by a subagent'
)) {
  if (-not $skill.Contains($required)) { throw "Missing read-view marker: $required" }
}
```

Expected: fail until read views and resume warnings are documented.

- [ ] **Step 2: Extend `list` and `show`**

Update read-view sections:

```markdown
For paths with `agent_run`, append compact execution state such as `[pending, subagent running]` or `[paused, subagent reported]`.

`show <key>` includes the latest `agent_run` fields and the subagent handoff/report summary when available.
```

- [ ] **Step 3: Extend `resume` warning**

In `resume`, add:

```markdown
If the target path has `agent_run.status: "running"`, include a warning that the path is currently being explored by a subagent and resuming it in the main session may duplicate work or mix context. Do not block the resume.
```

- [ ] **Step 4: Extend `map` and `map text`**

In `map`, add:

```markdown
When a path has `agent_run`, append compact execution state to the label, for example `B Network retry pending / subagent running`.
```

- [ ] **Step 5: Re-run the read-view contract**

```powershell
$skill = Get-Content -Raw trailmap/SKILL.md
foreach ($required in @(
  '[pending, subagent running]',
  'subagent reported',
  'currently being explored by a subagent',
  'Do not block the resume',
  'pending / subagent running'
)) {
  if (-not $skill.Contains($required)) { throw "Missing read-view content: $required" }
}
git diff --check -- trailmap/SKILL.md
```

Expected: exit `0`.

- [ ] **Step 6: Commit the read-view behavior**

```powershell
git add trailmap/SKILL.md
git commit -m "Show Trailmap subagent state in read views"
```

### Task 5: Update English Documentation

**Files:**
- Modify: `README.md`
- Modify: `docs/USAGE.md`
- Verify: `trailmap/SKILL.md`

- [ ] **Step 1: Run the failing English docs contract**

```powershell
$readme = Get-Content -Raw README.md
$usage = Get-Content -Raw docs/USAGE.md
foreach ($required in @(
  'subagent',
  '--allow-shared-code',
  '$trailmap subagent <key>',
  'shared workspace'
)) {
  if (-not (($readme + "`n" + $usage).Contains($required))) { throw "Missing English docs marker: $required" }
}
```

Expected: fail before English docs mention subagent exploration.

- [ ] **Step 2: Update `README.md`**

Add one core capability:

```markdown
- **Explore alternatives in parallel.** Start subagent exploration for pending paths while the main active path remains unchanged.
```

Add command summary lines:

```text
$trailmap subagent <key>                Start subagent exploration for an existing path
$trailmap ... --subagent B --allow-shared-code
```

Add safety text:

```markdown
Subagent exploration may run in the same workspace as the main active path. Trailmap warns about shared-code risk, but it does not isolate files or manage Git state.
```

- [ ] **Step 3: Update `docs/USAGE.md`**

Add detailed sections covering:

```markdown
### Start subagent exploration for an existing path
### Start subagent exploration for new paths
### Subagent context modes
### Subagent reports
```

Include examples:

```text
$trailmap subagent B
$trailmap subagent B --informed --allow-shared-code
$trailmap pending Network retry may be the cause --subagent --allow-shared-code
$trailmap Login failure may be token or network; start token --subagent B --allow-shared-code
```

Explain that `--allow-shared-code` skips the separate shared-code risk prompt, not the Trailmap write confirmation.

- [ ] **Step 4: Re-run the English docs contract**

```powershell
$readme = Get-Content -Raw README.md
$usage = Get-Content -Raw docs/USAGE.md
foreach ($required in @(
  'Explore alternatives in parallel',
  '$trailmap subagent <key>',
  '--subagent B',
  '--allow-shared-code',
  '--informed',
  'shared workspace',
  'does not skip the Trailmap write confirmation',
  'Subagent reports'
)) {
  if (-not (($readme + "`n" + $usage).Contains($required))) { throw "Missing English docs content: $required" }
}
git diff --check -- README.md docs/USAGE.md
```

Expected: exit `0`.

- [ ] **Step 5: Commit English docs**

```powershell
git add README.md docs/USAGE.md
git commit -m "Document Trailmap subagent usage in English"
```

### Task 6: Update Chinese Documentation

**Files:**
- Modify: `README.zh-CN.md`
- Modify: `docs/USAGE.zh-CN.md`
- Verify: `docs/USAGE.md`

- [ ] **Step 1: Run the failing Chinese docs contract**

```powershell
$readme = Get-Content -Raw README.zh-CN.md
$usage = Get-Content -Raw docs/USAGE.zh-CN.md
foreach ($required in @(
  'subagent',
  '--allow-shared-code',
  '$trailmap subagent <key>',
  '共享工作区'
)) {
  if (-not (($readme + "`n" + $usage).Contains($required))) { throw "Missing Chinese docs marker: $required" }
}
```

Expected: fail before Chinese docs mention subagent exploration.

- [ ] **Step 2: Update `README.zh-CN.md`**

Add one core capability:

```markdown
- **并行探索备选路径。** 为 pending 路径启动 subagent 探索，同时保持主会话 active path 不变。
```

Add command summary lines:

```text
$trailmap subagent <key>                为已有路径启动 subagent 探索
$trailmap ... --subagent B --allow-shared-code
```

Add safety text:

```markdown
subagent 探索可能和主会话 active path 在同一个 workspace 中运行。Trailmap 会提示共享代码风险，但不会隔离文件或管理 Git 状态。
```

- [ ] **Step 3: Update `docs/USAGE.zh-CN.md`**

Mirror `docs/USAGE.md` behavior with natural Chinese. Include:

```markdown
### 为已有路径启动 subagent 探索
### 为新建路径启动 subagent 探索
### subagent 上下文模式
### subagent 报告
```

Include the same command examples and explain that `--allow-shared-code` 只跳过共享代码风险的二次提示，不跳过 Trailmap 写入确认草案。

- [ ] **Step 4: Re-run the Chinese docs contract**

```powershell
$readme = Get-Content -Raw README.zh-CN.md
$usage = Get-Content -Raw docs/USAGE.zh-CN.md
foreach ($required in @(
  '并行探索备选路径',
  '$trailmap subagent <key>',
  '--subagent B',
  '--allow-shared-code',
  '--informed',
  '共享工作区',
  '不跳过 Trailmap 写入确认',
  'subagent 报告'
)) {
  if (-not (($readme + "`n" + $usage).Contains($required))) { throw "Missing Chinese docs content: $required" }
}
git diff --check -- README.zh-CN.md docs/USAGE.zh-CN.md
```

Expected: exit `0`.

- [ ] **Step 5: Commit Chinese docs**

```powershell
git add README.zh-CN.md docs/USAGE.zh-CN.md
git commit -m "Document Trailmap subagent usage in Chinese"
```

### Task 7: Validate Complete Subagent Contract

**Files:**
- Verify: `trailmap/SKILL.md`
- Verify: `README.md`
- Verify: `README.zh-CN.md`
- Verify: `docs/USAGE.md`
- Verify: `docs/USAGE.zh-CN.md`

- [ ] **Step 1: Verify invariant preservation**

```powershell
$skill = Get-Content -Raw trailmap/SKILL.md
foreach ($required in @(
  'exactly one path must have `status: "active"`',
  'Do not mark a subagent path as a second `active` path',
  'execution state, not path lifecycle state'
)) {
  if (-not $skill.Contains($required)) { throw "Invariant not preserved: $required" }
}
```

Expected: exit `0`.

- [ ] **Step 2: Verify command and flag coverage**

```powershell
$files = @('trailmap/SKILL.md', 'docs/USAGE.md', 'docs/USAGE.zh-CN.md')
$full = ($files | ForEach-Object { Get-Content -Raw $_ }) -join "`n"
foreach ($required in @(
  'subagent <key>',
  '--subagent',
  '--allow-shared-code',
  '--informed',
  'agent_run',
  'reported',
  'blocked',
  'handoff'
)) {
  if (-not $full.Contains($required)) { throw "Missing final subagent contract: $required" }
}
```

Expected: exit `0`.

- [ ] **Step 3: Verify unsupported behavior is absent**

```powershell
$files = @('trailmap/SKILL.md', 'README.md', 'README.zh-CN.md', 'docs/USAGE.md', 'docs/USAGE.zh-CN.md')
$full = ($files | ForEach-Object { Get-Content -Raw $_ }) -join "`n"
foreach ($forbidden in @(
  'allow multiple active paths',
  'create a second active path',
  'automatically create a worktree',
  'automatically stash changes',
  'subagent directly closes a path'
)) {
  if ($full.Contains($forbidden)) { throw "Forbidden behavior documented: $forbidden" }
}
```

Expected: exit `0`. Keep these forbidden phrases scoped to unsupported positive claims so the check does not reject correct safety statements.

- [ ] **Step 4: Verify formatting**

```powershell
git diff --check
git status --short --branch
```

Expected: no whitespace errors. `git status` should show only intentional committed branch divergence or a clean tree after all commits.

- [ ] **Step 5: Final review checklist**

Manually inspect:

```powershell
git show --stat --oneline HEAD~6..HEAD
rg -n 'subagent|agent_run|allow-shared-code|informed|reported|blocked|handoff' trailmap/SKILL.md README.md README.zh-CN.md docs/USAGE.md docs/USAGE.zh-CN.md
```

Confirm:

- `trailmap/SKILL.md` is still the behavioral source of truth.
- English and Chinese docs describe the same commands.
- `--allow-shared-code` is consistently described as risk acceptance, not write-confirmation bypass.
- `agent_run.status: reported` is used for returned but unconfirmed reports.
- Git safety language remains consistent with v0.1.0.

## Completion Criteria

- Trailmap documents subagent parallel exploration without allowing multiple main active paths.
- Existing and newly created non-main-active paths can request subagent exploration.
- Shared workspace risk and write confirmation rules are explicit.
- Subagent context modes, report schema, result confirmation, and read-view display are documented.
- Bilingual README and usage docs are consistent with `trailmap/SKILL.md`.
- Static checks pass and each task is committed separately.
