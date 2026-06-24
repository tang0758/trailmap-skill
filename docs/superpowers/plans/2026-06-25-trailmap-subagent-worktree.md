# Trailmap Subagent Worktree Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Document Trailmap v0.3.0 worktree mode for subagent path exploration.

**Architecture:** Extend the existing documentation-only Trailmap skill contract with an explicit `--worktree` execution mode, optional `--base <ref>`, and `agent_run.worktree` metadata. Preserve v0.2.0 shared-workspace behavior as the default, keep one main active path per topic, and update bilingual docs with worktree creation, confirmation, failure, result, read-view, and resume rules.

**Tech Stack:** Markdown, Agent Skill instructions, PowerShell static checks, Git

---

## File Map

- Modify `trailmap/SKILL.md`: behavioral source of truth for worktree flags, data model, validation, confirmation drafts, read views, and resume warnings.
- Modify `docs/USAGE.md`: English detailed usage for worktree mode.
- Modify `docs/USAGE.zh-CN.md`: Chinese detailed usage for worktree mode.
- Modify `README.md`: concise English product capability, command summary, and safety note.
- Modify `README.zh-CN.md`: concise Chinese product capability, command summary, and safety note.
- Verify `docs/superpowers/specs/2026-06-25-trailmap-subagent-worktree-design.md`: design source for v0.3.0.

### Task 1: Add Worktree Execution Model To Skill

**Files:**
- Modify: `trailmap/SKILL.md`
- Verify: `docs/superpowers/specs/2026-06-25-trailmap-subagent-worktree-design.md`

- [ ] **Step 1: Run the failing model contract**

```powershell
$skill = Get-Content -Raw trailmap/SKILL.md
foreach ($required in @(
  'agent_run.worktree',
  'creating',
  'ready',
  'retained',
  'removed',
  'base_dirty',
  'diff_summary'
)) {
  if (-not $skill.Contains($required)) { throw "Missing worktree model marker: $required" }
}
```

Expected: fail before implementation because worktree metadata is not documented.

- [ ] **Step 2: Add worktree fields to `agent_run`**

In `trailmap/SKILL.md`, extend the existing `agent_run` section with:

```markdown
Worktree execution fields:

```json
{
  "worktree": {
    "enabled": true,
    "status": "ready",
    "path": ".worktrees/trailmap/login-timeout/B",
    "branch": "trailmap/login-timeout/B",
    "base_ref": "HEAD",
    "base_sha": "abc123",
    "base_dirty": true,
    "changed_files": [],
    "diff_summary": ""
  }
}
```

Rules:

- `agent_run.worktree` is present only for `--worktree` runs.
- `agent_run.worktree.status` may be `creating`, `ready`, `failed`, `retained`, or `removed`.
- `base_ref` defaults to `HEAD`; `--base <ref>` overrides it.
- `base_sha` stores the resolved base commit.
- `base_dirty` records whether the main workspace had uncommitted changes at worktree creation time.
- `changed_files` and `diff_summary` summarize worktree changes relative to `base_sha`.
- Do not store worktree metadata in `codechange`; copy only path-level summaries into update `codechange`.
- Do not start a new subagent run for a path while `agent_run.status` is `running` or `reported`.
```

- [ ] **Step 3: Re-run the model contract**

```powershell
$skill = Get-Content -Raw trailmap/SKILL.md
foreach ($required in @(
  'agent_run.worktree',
  'creating',
  'ready',
  'failed',
  'retained',
  'removed',
  'base_ref',
  'base_sha',
  'base_dirty',
  'changed_files',
  'diff_summary',
  'while `agent_run.status` is `running` or `reported`'
)) {
  if (-not $skill.Contains($required)) { throw "Missing worktree model content: $required" }
}
git diff --check -- trailmap/SKILL.md
```

Expected: exit `0`.

- [ ] **Step 4: Commit the model update**

```powershell
git add trailmap/SKILL.md
git commit -m "Add Trailmap subagent worktree model"
```

### Task 2: Define Worktree Flags And Validation

**Files:**
- Modify: `trailmap/SKILL.md`

- [ ] **Step 1: Run the failing flag contract**

```powershell
$skill = Get-Content -Raw trailmap/SKILL.md
foreach ($required in @(
  '--worktree',
  '--base <ref>',
  'mutually exclusive',
  '.worktrees/trailmap/<topic-id>/<path-key>'
)) {
  if (-not $skill.Contains($required)) { throw "Missing worktree flag marker: $required" }
}
```

Expected: fail until worktree flags are documented.

- [ ] **Step 2: Extend subagent flag parsing**

In `trailmap/SKILL.md`, update the subagent flags section with:

```markdown
- `--worktree` requests isolated Git worktree execution for selected subagent paths.
- `--base <ref>` sets the Git ref used to create the worktree; default is current `HEAD`.
- `--worktree` and `--allow-shared-code` are mutually exclusive.
- `--worktree` may be combined with `--informed`.
- Do not support `--no-worktree`; shared workspace is already the default.
```

- [ ] **Step 3: Add worktree validation rules**

In the `subagent <key>` section and new-path subagent behavior, add:

```markdown
For `--worktree`:

- Resolve `--base <ref>` or current `HEAD` before creating the worktree.
- If the base ref does not resolve, write `agent_run.status: "blocked"` and `agent_run.worktree.status: "failed"` after confirmation; do not fall back to `HEAD`.
- Reject commands that combine `--worktree` and `--allow-shared-code`.
- Create one worktree and one branch per selected path.
- Do not copy uncommitted main-workspace changes into the worktree.
- Record `base_dirty: true` when the main workspace has uncommitted changes.
```

- [ ] **Step 4: Add naming rules**

Add:

```markdown
Default worktree path:

```text
.worktrees/trailmap/<topic-id>/<path-key>/
```

Default branch:

```text
trailmap/<topic-id>/<path-key>
```

If the directory exists and is non-empty, or the branch already exists, append the same unique suffix to both. Do not reuse non-empty directories or existing branches.
```

- [ ] **Step 5: Re-run the flag contract**

```powershell
$skill = Get-Content -Raw trailmap/SKILL.md
foreach ($required in @(
  '--worktree',
  '--base <ref>',
  'mutually exclusive',
  'Do not support `--no-worktree`',
  '.worktrees/trailmap/<topic-id>/<path-key>',
  'trailmap/<topic-id>/<path-key>',
  'Do not reuse non-empty directories or existing branches',
  'do not fall back to `HEAD`'
)) {
  if (-not $skill.Contains($required)) { throw "Missing worktree flag content: $required" }
}
git diff --check -- trailmap/SKILL.md
```

Expected: exit `0`.

- [ ] **Step 6: Commit the flag behavior**

```powershell
git add trailmap/SKILL.md
git commit -m "Define Trailmap worktree flags"
```

### Task 3: Define Worktree Confirmation And Git Safety

**Files:**
- Modify: `trailmap/SKILL.md`

- [ ] **Step 1: Run the failing safety contract**

```powershell
$skill = Get-Content -Raw trailmap/SKILL.md
foreach ($required in @(
  '.gitignore',
  '.worktrees/',
  'will not be copied',
  'will not merge',
  'will not remove worktrees automatically'
)) {
  if (-not $skill.Contains($required)) { throw "Missing worktree safety marker: $required" }
}
```

Expected: fail until confirmation and safety rules are documented.

- [ ] **Step 2: Add worktree confirmation draft requirements**

Add a section near confirmation rules:

```markdown
### Worktree Confirmation Drafts

Before creating a worktree, show:

```text
path
context mode
base ref and base sha
branch to create
worktree path to create
whether the main workspace has uncommitted changes
whether .gitignore will be updated
that uncommitted changes will not be copied
that Trailmap will not merge, commit, stash, revert, apply, or copy code automatically
```

Do not create the branch or worktree before explicit confirmation.
```

- [ ] **Step 3: Add `.gitignore` safety rules**

Add:

```markdown
`.worktrees/` must be ignored before creating project-local worktrees. If `.worktrees/` is not ignored, include this exact change in the confirmation draft:

```text
+ .worktrees/
```

After confirmation, Trailmap may create `.gitignore` if missing or append only `.worktrees/` if present. Do not reorder or reformat `.gitignore`.
```

- [ ] **Step 4: Add no automatic integration or cleanup rules**

Add:

```markdown
Trailmap does not automatically merge, cherry-pick, apply patches, copy files, commit, stash, revert, or switch the main workspace branch.

When a worktree subagent reports, keep the worktree by default. Set `agent_run.worktree.status` to `retained`. The first worktree version does not implement a cleanup command and does not remove worktrees automatically.
```

- [ ] **Step 5: Re-run the safety contract**

```powershell
$skill = Get-Content -Raw trailmap/SKILL.md
foreach ($required in @(
  'Worktree Confirmation Drafts',
  '.gitignore',
  '.worktrees/',
  'uncommitted changes will not be copied',
  'does not automatically merge',
  'does not remove worktrees automatically',
  'Set `agent_run.worktree.status` to `retained`'
)) {
  if (-not $skill.Contains($required)) { throw "Missing worktree safety content: $required" }
}
git diff --check -- trailmap/SKILL.md
```

Expected: exit `0`.

- [ ] **Step 6: Commit safety rules**

```powershell
git add trailmap/SKILL.md
git commit -m "Define Trailmap worktree safety rules"
```

### Task 4: Define Worktree Results, Read Views, And Resume Warnings

**Files:**
- Modify: `trailmap/SKILL.md`

- [ ] **Step 1: Run the failing result contract**

```powershell
$skill = Get-Content -Raw trailmap/SKILL.md
foreach ($required in @(
  'worktree.path',
  'worktree.branch',
  'worktree.changed_files',
  'Worktree diff unavailable',
  'not merged into the current workspace'
)) {
  if (-not $skill.Contains($required)) { throw "Missing worktree result marker: $required" }
}
```

Expected: fail until result and resume behavior are documented.

- [ ] **Step 2: Extend subagent report schema**

In `Subagent Reports`, add worktree-mode report fields:

```markdown
For worktree runs, the report must also include:

```text
worktree.path
worktree.branch
worktree.base_ref
worktree.base_sha
worktree.changed_files
worktree.diff_summary
```
```

- [ ] **Step 3: Add worktree codechange rules**

Add:

```markdown
For worktree runs, derive `codechange.files` from the worktree diff against `agent_run.worktree.base_sha`, not from the main workspace.

If the worktree diff cannot be read, use:

```json
{
  "changed": true,
  "files": [],
  "summary": "Worktree diff unavailable; inspect worktree manually."
}
```

Store structured worktree details in `agent_run.worktree`; copy only the path-level summary into update `codechange`.
```

- [ ] **Step 4: Extend read views**

Update list/map/show rules:

```markdown
`list` and `map` show compact worktree state only, for example `[pending, subagent running, worktree]` or `pending / subagent running / worktree`.

`show <key>` expands worktree path, branch, base ref, base sha, base dirty, changed files, and diff summary.
```

- [ ] **Step 5: Extend resume warnings**

Add:

```markdown
If a resumed path has `agent_run.worktree.status: "retained"` with changed files or diff summary, warn that those code changes are not merged into the current workspace and `resume clean` will not apply them.

Clean resume may include the target path's own worktree artifact summary, but not unconfirmed full handoff reasoning.
```

- [ ] **Step 6: Re-run the result contract**

```powershell
$skill = Get-Content -Raw trailmap/SKILL.md
foreach ($required in @(
  'worktree.path',
  'worktree.branch',
  'worktree.changed_files',
  'Worktree diff unavailable; inspect worktree manually.',
  '[pending, subagent running, worktree]',
  'pending / subagent running / worktree',
  'not merged into the current workspace',
  'will not apply them'
)) {
  if (-not $skill.Contains($required)) { throw "Missing worktree result content: $required" }
}
git diff --check -- trailmap/SKILL.md
```

Expected: exit `0`.

- [ ] **Step 7: Commit result behavior**

```powershell
git add trailmap/SKILL.md
git commit -m "Define Trailmap worktree result handling"
```

### Task 5: Update English Documentation

**Files:**
- Modify: `README.md`
- Modify: `docs/USAGE.md`

- [ ] **Step 1: Run the failing English worktree docs contract**

```powershell
$readme = Get-Content -Raw README.md
$usage = Get-Content -Raw docs/USAGE.md
$combined = $readme + "`n" + $usage
foreach ($required in @(
  '--worktree',
  '--base <ref>',
  '.worktrees/',
  'worktree branch',
  'retained worktree'
)) {
  if (-not $combined.Contains($required)) { throw "Missing English worktree docs marker: $required" }
}
```

Expected: fail until English docs cover worktree mode.

- [ ] **Step 2: Update `README.md`**

Add a concise capability or safety note:

```markdown
- **Isolate risky subagent work.** Use `--worktree` to run a subagent path in a separate Git worktree.
```

Add command examples:

```text
$trailmap subagent <key> --worktree
$trailmap subagent <key> --worktree --base <ref>
```

Add safety text:

```markdown
Worktree mode creates a local branch and worktree after confirmation. Trailmap records the path and branch but does not merge, commit, clean up, or apply worktree changes automatically.
```

- [ ] **Step 3: Update `docs/USAGE.md`**

Add detailed sections near subagent usage:

```markdown
### Run a subagent in a worktree
### Worktree confirmation and safety
### Worktree results
### Resume paths with retained worktree changes
```

Include examples:

```text
$trailmap subagent B --worktree
$trailmap subagent B --worktree --informed
$trailmap subagent B --worktree --base origin/main
$trailmap pending Network retry may be the cause --subagent --worktree
$trailmap Login failure may be token, network, or cache; start token --subagent B,C --worktree
```

Document:

- worktree mode is opt-in
- `--worktree` and `--allow-shared-code` are mutually exclusive
- `.worktrees/` must be ignored
- Trailmap may append `.worktrees/` to `.gitignore` after confirmation
- base defaults to `HEAD`
- no fallback when `--base <ref>` cannot resolve
- uncommitted main-workspace changes are not copied
- no automatic merge, commit, cleanup, patch apply, or file copying
- `list/map` show compact worktree state
- `show <key>` shows path, branch, base, changed files, and diff summary
- `resume clean` warns about retained unmerged worktree changes

- [ ] **Step 4: Re-run English docs contract**

```powershell
$readme = Get-Content -Raw README.md
$usage = Get-Content -Raw docs/USAGE.md
$combined = $readme + "`n" + $usage
foreach ($required in @(
  'Isolate risky subagent work',
  '$trailmap subagent <key> --worktree',
  '--base <ref>',
  '--worktree` and `--allow-shared-code` are mutually exclusive',
  '.worktrees/',
  '.gitignore',
  'base defaults to `HEAD`',
  'uncommitted changes are not copied',
  'retained worktree',
  'does not merge, commit, clean up, or apply worktree changes automatically'
)) {
  if (-not $combined.Contains($required)) { throw "Missing English worktree docs content: $required" }
}
git diff --check -- README.md docs/USAGE.md
```

Expected: exit `0`.

- [ ] **Step 5: Commit English docs**

```powershell
git add README.md docs/USAGE.md
git commit -m "Document Trailmap worktree usage in English"
```

### Task 6: Update Chinese Documentation

**Files:**
- Modify: `README.zh-CN.md`
- Modify: `docs/USAGE.zh-CN.md`

- [ ] **Step 1: Run the failing Chinese worktree docs contract**

```powershell
$readme = Get-Content -Raw README.zh-CN.md
$usage = Get-Content -Raw docs/USAGE.zh-CN.md
$combined = $readme + "`n" + $usage
foreach ($required in @(
  '--worktree',
  '--base <ref>',
  '.worktrees/',
  'worktree branch',
  'retained worktree'
)) {
  if (-not $combined.Contains($required)) { throw "Missing Chinese worktree docs marker: $required" }
}
```

Expected: fail until Chinese docs cover worktree mode.

- [ ] **Step 2: Update `README.zh-CN.md`**

Add a concise capability or safety note:

```markdown
- **隔离高风险 subagent 改动。** 使用 `--worktree` 让 subagent path 在独立 Git worktree 中运行。
```

Add command examples:

```text
$trailmap subagent <key> --worktree
$trailmap subagent <key> --worktree --base <ref>
```

Add safety text:

```markdown
worktree 模式会在确认后创建本地 branch 和 worktree。Trailmap 只记录 path 和 branch，不会自动 merge、commit、清理或应用 worktree 改动。
```

- [ ] **Step 3: Update `docs/USAGE.zh-CN.md`**

Mirror the English usage behavior with natural Chinese. Add sections:

```markdown
### 在 worktree 中运行 subagent
### worktree 确认草案与安全边界
### worktree 结果
### resume 带 retained worktree 改动的路径
```

Include the same command examples and document:

- worktree 是显式 opt-in
- `--worktree` 和 `--allow-shared-code` 互斥
- `.worktrees/` 必须被忽略
- 确认后 Trailmap 可以向 `.gitignore` 追加 `.worktrees/`
- base 默认是 `HEAD`
- `--base <ref>` 不存在时不 fallback
- 主工作区未提交改动不会复制到 worktree
- 不自动 merge、commit、清理、apply patch 或复制文件
- `list/map` 只显示紧凑 worktree 状态
- `show <key>` 展示 path、branch、base、changed files 和 diff summary
- `resume clean` 会提示 retained worktree 改动尚未合并到当前工作区

- [ ] **Step 4: Re-run Chinese docs contract**

```powershell
$readme = Get-Content -Raw README.zh-CN.md
$usage = Get-Content -Raw docs/USAGE.zh-CN.md
$combined = $readme + "`n" + $usage
foreach ($required in @(
  '隔离高风险 subagent 改动',
  '$trailmap subagent <key> --worktree',
  '--base <ref>',
  '--worktree` 和 `--allow-shared-code` 互斥',
  '.worktrees/',
  '.gitignore',
  'base 默认是 `HEAD`',
  '未提交改动不会复制',
  'retained worktree',
  '不会自动 merge、commit、清理或应用 worktree 改动'
)) {
  if (-not $combined.Contains($required)) { throw "Missing Chinese worktree docs content: $required" }
}
git diff --check -- README.zh-CN.md docs/USAGE.zh-CN.md
```

Expected: exit `0`.

- [ ] **Step 5: Commit Chinese docs**

```powershell
git add README.zh-CN.md docs/USAGE.zh-CN.md
git commit -m "Document Trailmap worktree usage in Chinese"
```

### Task 7: Validate Complete Worktree Contract

**Files:**
- Verify: `trailmap/SKILL.md`
- Verify: `README.md`
- Verify: `README.zh-CN.md`
- Verify: `docs/USAGE.md`
- Verify: `docs/USAGE.zh-CN.md`

- [ ] **Step 1: Verify command and model coverage**

```powershell
$files = @('trailmap/SKILL.md', 'docs/USAGE.md', 'docs/USAGE.zh-CN.md')
$full = ($files | ForEach-Object { Get-Content -Raw $_ }) -join "`n"
foreach ($required in @(
  '--worktree',
  '--base <ref>',
  'agent_run.worktree',
  'base_dirty',
  'changed_files',
  'diff_summary',
  'retained',
  '.worktrees/',
  '.gitignore'
)) {
  if (-not $full.Contains($required)) { throw "Missing final worktree contract: $required" }
}
```

Expected: exit `0`.

- [ ] **Step 2: Verify safety boundaries**

```powershell
$files = @('trailmap/SKILL.md', 'README.md', 'README.zh-CN.md', 'docs/USAGE.md', 'docs/USAGE.zh-CN.md')
$full = ($files | ForEach-Object { Get-Content -Raw $_ }) -join "`n"
foreach ($required in @(
  'mutually exclusive',
  'does not automatically merge',
  'does not remove worktrees automatically',
  'uncommitted changes will not be copied',
  'do not fall back to `HEAD`'
)) {
  if (-not $full.Contains($required)) { throw "Missing worktree safety rule: $required" }
}
foreach ($forbidden in @(
  'worktree mode is default',
  'automatically merge worktree',
  'automatically remove worktree',
  'silently fallback to HEAD',
  'automatically copy uncommitted changes into the worktree'
)) {
  if ($full.Contains($forbidden)) { throw "Forbidden worktree behavior documented: $forbidden" }
}
```

Expected: exit `0`. If a forbidden phrase appears inside an explicit negative sentence, rewrite the sentence to avoid ambiguous grep failures.

- [ ] **Step 3: Verify bilingual consistency markers**

```powershell
$en = Get-Content -Raw docs/USAGE.md
$zh = Get-Content -Raw docs/USAGE.zh-CN.md
foreach ($marker in @('--worktree', '--base <ref>', '.worktrees/', '.gitignore', 'agent_run.worktree', 'retained')) {
  if (-not $en.Contains($marker)) { throw "English usage missing marker: $marker" }
  if (-not $zh.Contains($marker)) { throw "Chinese usage missing marker: $marker" }
}
```

Expected: exit `0`.

- [ ] **Step 4: Verify formatting and repository status**

```powershell
git diff --check
git status --short --branch
```

Expected: no whitespace errors. `git status` should show a clean tree after commits, with local branch ahead of `origin/main` until pushed.

- [ ] **Step 5: Final review checklist**

Manually inspect:

```powershell
git show --stat --oneline HEAD~6..HEAD
rg -n 'worktree|agent_run.worktree|--base|base_dirty|retained|.gitignore|.worktrees' trailmap/SKILL.md README.md README.zh-CN.md docs/USAGE.md docs/USAGE.zh-CN.md
```

Confirm:

- `--worktree` is opt-in.
- `--worktree` and `--allow-shared-code` are mutually exclusive.
- `--worktree` still requires confirmation before creating local Git side effects.
- `.gitignore` mutation is limited to adding `.worktrees/` after confirmation.
- `--base <ref>` failure blocks the run and does not silently change base.
- Worktree results are retained and never merged automatically.
- `resume clean` warns when retained worktree changes are not in the current workspace.

## Completion Criteria

- Trailmap documents v0.3.0 worktree execution mode for subagent paths.
- Existing shared-workspace subagent behavior remains the default.
- Worktree metadata, safety rules, and failure behavior are encoded in `trailmap/SKILL.md`.
- Bilingual README and usage docs describe equivalent behavior.
- Static checks pass and each task is committed separately.
