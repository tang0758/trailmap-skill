# Trailmap Concise Confirmation Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Make every Trailmap write-operation confirmation concise for humans while preserving complete internal drafts and persisted records.

**Architecture:** Add one global presentation contract to `SKILL.md`, then keep per-command rules limited to the fields that differ. Update both README files with a short user-facing explanation, and validate behavior through static contracts plus fresh-agent scenarios.

**Tech Stack:** Agent Skills Markdown, PowerShell static checks, Git

---

## File Map

- Modify `trailmap/SKILL.md`: add the global concise-draft contract and reduce command-specific presentation requirements.
- Modify `README.md`: explain concise confirmations without duplicating internal rules.
- Modify `README.zh-CN.md`: provide the same explanation in Chinese.
- No schema, scripts, references, or runtime dependencies are added.

### Task 1: Define Concise Confirmation Drafts

**Files:**
- Modify: `trailmap/SKILL.md:141-365`

- [ ] **Step 1: Run the failing presentation contract**

```powershell
$skill = Get-Content -Raw trailmap/SKILL.md
if (-not $skill.Contains('## Confirmation Draft Presentation')) {
  throw 'Concise confirmation presentation contract is missing'
}
```

Expected: fail with `Concise confirmation presentation contract is missing`.

- [ ] **Step 2: Add the global presentation section**

Insert before `## Commands`:

```markdown
## Confirmation Draft Presentation

Build the complete structured draft internally, but show a concise human confirmation view by default.

Show only identifiers and titles, decision-relevant goals/hypotheses/summaries/conclusions, resulting statuses or transitions, necessary default-choice explanations, relevant warnings, and one explicit confirmation question at the end.

Hide `source`, `created_from`, `parent`, timestamps, repeated status-change lists, and the complete persistence structure unless a key/parent relationship is ambiguous, a closed path is reopened, code changes may contaminate a clean resume, or the user asks to expand details. Hidden fields remain in the internal draft and are written normally after confirmation.
```

- [ ] **Step 3: Replace command-specific verbose draft requirements**

Use these minimum views:

```text
no subcommand: topic id/title; each path key/status/title/goal/hypothesis; default active explanation
pending: new path key/title/goal/hypothesis; active path remains unchanged
update: key/summary/conclusion/status_after; codechange only when changed
resume: leave summary; target; mode; status transition; relevant code warning
close: key/closed_as/closed_reason/final summary; relevant codechange
rename: old title/new title
```

Remove requirements that force `source`, `created_from`, `parent`, or a separate status-change list into the default view. Preserve all write behavior and internal fields.

- [ ] **Step 4: Strengthen the confirmation rule**

Under `## Confirmation Rule`, state that every write draft ends with one explicit question such as:

```text
Confirm writing the above changes?
```

Keep the existing rule that no state is written before explicit confirmation.

- [ ] **Step 5: Run static Skill checks**

```powershell
$skill = Get-Content -Raw trailmap/SKILL.md
foreach ($required in @(
  '## Confirmation Draft Presentation',
  'complete structured draft internally',
  'explicit confirmation question',
  '`source`', '`created_from`', '`parent`',
  'code changes may contaminate a clean resume',
  'user asks to expand details'
)) {
  if (-not $skill.Contains($required)) { throw "Missing: $required" }
}
if (-not $skill.Contains('.trailmap/marks/')) { throw 'Storage rule changed' }
if (-not $skill.Contains('Never auto-stash')) { throw 'Git safety rule changed' }
```

Expected: exit `0` with no errors.

- [ ] **Step 6: Commit the Skill change**

```powershell
git add trailmap/SKILL.md
git commit -m "Simplify Trailmap confirmation drafts"
```

### Task 2: Document Concise Confirmations

**Files:**
- Modify: `README.md`
- Modify: `README.zh-CN.md`

- [ ] **Step 1: Run the failing documentation contract**

```powershell
$english = Get-Content -Raw README.md
$chinese = Get-Content -Raw README.zh-CN.md
if (-not $english.Contains('concise confirmation draft')) { throw 'English explanation missing' }
if (-not $chinese.Contains('简洁确认草案')) { throw 'Chinese explanation missing' }
```

Expected: fail because both explanations are absent.

- [ ] **Step 2: Add a concise English explanation**

Add one short section near the usage/model description:

```markdown
## Confirmation drafts

Write operations show a concise confirmation draft with only decision-relevant fields. Trailmap still keeps the complete structured draft internally and writes it only after explicit confirmation. Ask to expand details when you need to inspect internal fields.
```

- [ ] **Step 3: Add the equivalent Chinese explanation**

```markdown
## 确认草案

写操作默认只展示与决策有关的简洁确认草案。Trailmap 仍在内部保留完整结构，并且只在明确确认后落盘；需要检查内部字段时可以要求展开详情。
```

- [ ] **Step 4: Validate documentation scope**

```powershell
$english = Get-Content -Raw README.md
$chinese = Get-Content -Raw README.zh-CN.md
if (-not $english.Contains('concise confirmation draft')) { throw 'English explanation missing' }
if (-not $chinese.Contains('简洁确认草案')) { throw 'Chinese explanation missing' }
foreach ($text in @($english, $chinese)) {
  if (-not $text.Contains('.trailmap/marks/')) { throw 'Storage documentation changed' }
  if (-not $text.Contains('/trailmap show')) { throw 'Claude command documentation changed' }
  if (-not $text.Contains('$trailmap show')) { throw 'Codex command documentation changed' }
}
```

Expected: exit `0` with no errors.

- [ ] **Step 5: Commit documentation**

```powershell
git add README.md README.zh-CN.md
git commit -m "Document concise Trailmap confirmations"
```

### Task 3: Forward-Test And Deploy

**Files:**
- Verify: `trailmap/SKILL.md`
- Verify: `README.md`
- Verify: `README.zh-CN.md`
- Synchronize after approval: `C:\Users\Administrator\.codex\skills\trailmap\`

- [ ] **Step 1: Run the official Skill validator**

```powershell
python "$env:USERPROFILE\.codex\skills\.system\skill-creator\scripts\quick_validate.py" trailmap
```

Expected: success. If it fails only because PyYAML is unavailable, report the exact dependency limitation and continue with equivalent static checks.

- [ ] **Step 2: Run fresh-agent scenarios**

Use fresh agents that read the updated `trailmap/SKILL.md` without being told the expected output:

1. Create Linux-driver and AI root paths.
2. Add a pending sibling while keeping the active path.
3. Resume clean with recorded code changes.
4. Generate one example for every write command and inspect its final confirmation question.
5. Ask to expand a draft and inspect hidden internal fields.

Expected:

- default drafts omit `source`, `created_from`, `parent`, timestamps, and repeated state lists
- risk warnings remain visible
- every write draft ends with an explicit confirmation question
- expanded drafts can expose complete internal data

- [ ] **Step 3: Run final repository checks**

```powershell
git diff --check
git status --short
rg -n 'Confirmation Draft Presentation|concise confirmation draft|简洁确认草案' trailmap/SKILL.md README.md README.zh-CN.md
```

Expected: clean diff, clean worktree, and all three presentation markers found.

- [ ] **Step 4: Synchronize and verify the local Codex Skill**

Update `SKILL.md` and `agents/openai.yaml` in `C:\Users\Administrator\.codex\skills\trailmap\`, then compare normalized file contents with the repository copies. Do not delete the destination directory.

## Completion Criteria

- Every write operation uses the global concise confirmation view.
- Full structured data remains available internally and persists unchanged after confirmation.
- Risk cases expand automatically and explicit detail requests work.
- Every write operation ends with an explicit confirmation question.
- Both README files explain the behavior without duplicating the full rules.
- Static checks, fresh-agent scenarios, review, and installed-copy comparison pass.
