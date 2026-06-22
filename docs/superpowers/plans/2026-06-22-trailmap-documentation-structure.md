# Trailmap Documentation Structure Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Replace the current mixed-purpose READMEs with concise bilingual product introductions and add complete bilingual usage guides that accurately reflect `trailmap/SKILL.md`.

**Architecture:** Keep product positioning in the repository-root READMEs and move exhaustive behavioral documentation into `docs/USAGE.md` and `docs/USAGE.zh-CN.md`. Treat `trailmap/SKILL.md` as the behavioral source of truth, maintain equivalent English and Chinese claims, and validate the result with static content and relative-link contracts.

**Tech Stack:** Markdown, PowerShell static checks, Git

---

## File Map

- Modify `README.md`: concise English product introduction and navigation.
- Modify `README.zh-CN.md`: concise Chinese product introduction and navigation.
- Create `docs/USAGE.md`: complete English usage reference.
- Create `docs/USAGE.zh-CN.md`: complete Chinese usage reference.
- Verify `trailmap/SKILL.md`: source of truth only; do not modify.

### Task 1: Rewrite The English Product README

**Files:**
- Modify: `README.md`
- Verify: `trailmap/SKILL.md`

- [ ] **Step 1: Run the failing product-page contract**

```powershell
$readme = Get-Content -Raw README.md
foreach ($required in @(
  '## The problem',
  '## A 30-second example',
  '## Core capabilities',
  'docs/USAGE.md',
  'README.zh-CN.md'
)) {
  if (-not $readme.Contains($required)) { throw "Missing product section: $required" }
}
```

Expected: fail because the current README is a short mixed usage page.

- [ ] **Step 2: Replace the README with the English product structure**

Use these sections in this order:

```markdown
# Trailmap

> A stateful decision-path map for human-AI problem solving.

[中文](README.zh-CN.md) · [Detailed usage](docs/USAGE.md)

## The problem
## A 30-second example
## Core capabilities
## How the model works
## Install
## Essential commands
## Safety boundaries
## Current limitations
## Documentation
```

The opening explains lost alternatives and context drift. The example starts with paths A and B, explores A, records a pending path, and explicitly resumes B. The model section separates `active/pending/paused/closed` from `done/blocked/discarded`. Installation covers Codex and Claude Code using the repository's `trailmap/` directory. The commands use only `/trailmap` and `$trailmap` forms supported by `trailmap/SKILL.md`.

- [ ] **Step 3: Run the English README contract**

```powershell
$readme = Get-Content -Raw README.md
foreach ($required in @(
  'A stateful decision-path map',
  '## The problem',
  '## A 30-second example',
  '## Core capabilities',
  'active', 'pending', 'paused', 'closed',
  'done', 'blocked', 'discarded',
  '/trailmap', '$trailmap',
  'docs/USAGE.md', 'README.zh-CN.md',
  '.trailmap/marks/'
)) {
  if (-not $readme.Contains($required)) { throw "Missing English product content: $required" }
}
foreach ($unsupported in @('trailmap add', 'trailmap set', 'resume next')) {
  if ($readme.Contains($unsupported)) { throw "Unsupported command: $unsupported" }
}
git diff --check -- README.md
```

Expected: exit `0`.

- [ ] **Step 4: Commit the English product README**

```powershell
git add README.md
git commit -m "Rewrite English Trailmap product README"
```

### Task 2: Rewrite The Chinese Product README

**Files:**
- Modify: `README.zh-CN.md`
- Verify: `README.md`

- [ ] **Step 1: Run the failing Chinese product-page contract**

```powershell
$readme = Get-Content -Raw README.zh-CN.md
foreach ($required in @(
  '## Trailmap 解决什么问题',
  '## 30 秒理解 Trailmap',
  '## 核心能力',
  'docs/USAGE.zh-CN.md',
  'README.md'
)) {
  if (-not $readme.Contains($required)) { throw "Missing Chinese product section: $required" }
}
```

Expected: fail because the current file is a detailed mixed usage document.

- [ ] **Step 2: Replace the README with the Chinese product structure**

Use sections equivalent to the English README:

```markdown
# Trailmap

> 面向人机协同解决问题的有状态决策路径图。

[English](README.md) · [详细使用说明](docs/USAGE.zh-CN.md)

## Trailmap 解决什么问题
## 30 秒理解 Trailmap
## 核心能力
## Trailmap 如何组织路径
## 安装
## 最小命令集
## 安全边界
## 当前限制
## 文档
```

Mirror the English behavioral claims without literal, awkward translation. Keep the opening product-focused and move exhaustive explanations to the usage guide.

- [ ] **Step 3: Run the Chinese README contract**

```powershell
$readme = Get-Content -Raw README.zh-CN.md
foreach ($required in @(
  '有状态决策路径图',
  '## Trailmap 解决什么问题',
  '## 30 秒理解 Trailmap',
  '## 核心能力',
  'active', 'pending', 'paused', 'closed',
  'done', 'blocked', 'discarded',
  '/trailmap', '$trailmap',
  'docs/USAGE.zh-CN.md', 'README.md',
  '.trailmap/marks/'
)) {
  if (-not $readme.Contains($required)) { throw "Missing Chinese product content: $required" }
}
foreach ($unsupported in @('trailmap add', 'trailmap set', 'resume next')) {
  if ($readme.Contains($unsupported)) { throw "Unsupported command: $unsupported" }
}
git diff --check -- README.zh-CN.md
```

Expected: exit `0`.

- [ ] **Step 4: Commit the Chinese product README**

```powershell
git add README.zh-CN.md
git commit -m "Rewrite Chinese Trailmap product README"
```

### Task 3: Create The English Usage Guide

**Files:**
- Create: `docs/USAGE.md`
- Verify: `trailmap/SKILL.md`

- [ ] **Step 1: Run the failing English usage contract**

```powershell
if (-not (Test-Path docs/USAGE.md)) { throw 'English usage guide missing' }
```

Expected: fail because the file does not exist.

- [ ] **Step 2: Write the complete English usage guide**

Use this section structure:

```markdown
# Trailmap Usage Guide

[Product overview](../README.md) · [中文使用说明](USAGE.zh-CN.md)

## Core concepts
## Installation and invocation
## Command reference
### Create root or child paths
### Add a sibling pending path
### List all topics
### Show topic or path details
### Update path progress
### Resume with clean or informed context
### Close a path
### Rename the active topic
### Generate a Mermaid or text map
## Confirmation drafts
## Data model
## Storage
## Code-change and Git safety
## Recommended workflow
## End-to-end example
## Current limitations
```

Cover every command declared in `trailmap/SKILL.md`, including cross-topic resume. Explain that the base command creates children under the current active path, while `pending` creates a sibling and leaves `topic.active` unchanged. Explain closure state, concise confirmation drafts, `.trailmap/marks/`, compatibility with `.codex/marks/`, and that clean context does not isolate workspace code.

- [ ] **Step 3: Run the English usage contract**

```powershell
$usage = Get-Content -Raw docs/USAGE.md
foreach ($required in @(
  '../README.md', 'USAGE.zh-CN.md',
  'Topic', 'Path', 'Parent', 'Active Path', 'Created From',
  'pending <idea>', 'list', 'show [key]', 'update <key>',
  'resume <key> clean|informed',
  'resume <topic_id> <key> clean|informed',
  'close <key> done|blocked|discarded',
  'rename <topic title>', 'map [text]',
  'explicit confirmation', '.trailmap/marks/', '.codex/marks/',
  'stash', 'revert', 'commit'
)) {
  if (-not $usage.Contains($required)) { throw "Missing English usage content: $required" }
}
git diff --check -- docs/USAGE.md
```

Expected: exit `0`.

- [ ] **Step 4: Commit the English usage guide**

```powershell
git add docs/USAGE.md
git commit -m "Add English Trailmap usage guide"
```

### Task 4: Create The Chinese Usage Guide

**Files:**
- Create: `docs/USAGE.zh-CN.md`
- Verify: `docs/USAGE.md`, `trailmap/SKILL.md`

- [ ] **Step 1: Run the failing Chinese usage contract**

```powershell
if (-not (Test-Path docs/USAGE.zh-CN.md)) { throw 'Chinese usage guide missing' }
```

Expected: fail because the file does not exist.

- [ ] **Step 2: Write the complete Chinese usage guide**

Use the same structure and behavioral coverage as `docs/USAGE.md`, with natural Chinese headings and examples. Preserve established terms such as主题、路径、父路径、活跃路径、来源上下文、原地新增 pending 路径、简洁确认草案、安全边界, and推荐工作流. Correct the old statement that all human views merge `title/goal/hypothesis`; command confirmation drafts must follow their command-specific fields.

- [ ] **Step 3: Run the Chinese usage contract**

```powershell
$usage = Get-Content -Raw docs/USAGE.zh-CN.md
foreach ($required in @(
  '../README.zh-CN.md', 'USAGE.md',
  '主题', '路径', '父路径', '活跃路径', '来源上下文',
  'pending <idea>', 'list', 'show [key]', 'update <key>',
  'resume <key> clean|informed',
  'resume <topic_id> <key> clean|informed',
  'close <key> done|blocked|discarded',
  'rename <topic title>', 'map [text]',
  '明确确认', '.trailmap/marks/', '.codex/marks/',
  'stash', 'revert', 'commit'
)) {
  if (-not $usage.Contains($required)) { throw "Missing Chinese usage content: $required" }
}
git diff --check -- docs/USAGE.zh-CN.md
```

Expected: exit `0`.

- [ ] **Step 4: Commit the Chinese usage guide**

```powershell
git add docs/USAGE.zh-CN.md
git commit -m "Add Chinese Trailmap usage guide"
```

### Task 5: Validate Bilingual Consistency And Links

**Files:**
- Verify: `README.md`
- Verify: `README.zh-CN.md`
- Verify: `docs/USAGE.md`
- Verify: `docs/USAGE.zh-CN.md`
- Verify: `trailmap/SKILL.md`

- [ ] **Step 1: Verify all relative links resolve**

```powershell
$links = @(
  @{ From = 'README.md'; To = 'README.zh-CN.md' },
  @{ From = 'README.md'; To = 'docs/USAGE.md' },
  @{ From = 'README.zh-CN.md'; To = 'README.md' },
  @{ From = 'README.zh-CN.md'; To = 'docs/USAGE.zh-CN.md' },
  @{ From = 'docs/USAGE.md'; To = '../README.md' },
  @{ From = 'docs/USAGE.md'; To = 'USAGE.zh-CN.md' },
  @{ From = 'docs/USAGE.zh-CN.md'; To = '../README.zh-CN.md' },
  @{ From = 'docs/USAGE.zh-CN.md'; To = 'USAGE.md' }
)
foreach ($link in $links) {
  $sourceDirectory = Split-Path (Resolve-Path $link.From).Path -Parent
  $target = Join-Path $sourceDirectory $link.To
  if (-not (Test-Path $target)) { throw "Broken link target from $($link.From): $($link.To)" }
  if (-not (Get-Content -Raw $link.From).Contains($link.To)) { throw "Link missing in $($link.From): $($link.To)" }
}
```

Expected: exit `0`.

- [ ] **Step 2: Verify commands and behavior against the Skill**

```powershell
$docs = @('README.md', 'README.zh-CN.md', 'docs/USAGE.md', 'docs/USAGE.zh-CN.md')
$full = ($docs | ForEach-Object { Get-Content -Raw $_ }) -join "`n"
foreach ($unsupported in @('trailmap add', 'trailmap set', 'resume next', 'automatically switches')) {
  if ($full.Contains($unsupported)) { throw "Unsupported behavior documented: $unsupported" }
}
foreach ($usageFile in @('docs/USAGE.md', 'docs/USAGE.zh-CN.md')) {
  $usage = Get-Content -Raw $usageFile
  foreach ($command in @('pending', 'list', 'show', 'update', 'resume', 'close', 'rename', 'map')) {
    if (-not $usage.Contains($command)) { throw "$usageFile misses command: $command" }
  }
}
```

Expected: exit `0`.

- [ ] **Step 3: Run final repository checks**

```powershell
git diff --check
git status --short
rg -n 'active|pending|paused|closed|done|blocked|discarded|\.trailmap/marks/' README.md README.zh-CN.md docs/USAGE.md docs/USAGE.zh-CN.md
```

Expected: no formatting errors, clean worktree after commits, and all behavioral markers present.

- [ ] **Step 4: Request a final documentation review**

Review all four files for product clarity, factual accuracy against `trailmap/SKILL.md`, natural English and Chinese, command completeness, and absence of duplicated exhaustive material in the product READMEs. Fix findings in a dedicated review commit if needed.

## Completion Criteria

- The repository has separate bilingual product introductions and detailed usage guides.
- Product READMEs explain Trailmap quickly without becoming command manuals.
- Usage guides cover every supported command and behavioral boundary.
- English and Chinese claims are equivalent and naturally written.
- All relative links resolve.
- No unsupported commands or automatic switching behavior appear.
- `trailmap/SKILL.md` remains unchanged.
