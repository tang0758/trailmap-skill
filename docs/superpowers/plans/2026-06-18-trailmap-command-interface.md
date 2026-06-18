# Trailmap Command Interface Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Replace Trailmap's redundant public `mark <subcommand>` layer with direct `$trailmap <subcommand>` and `/trailmap <subcommand>` invocation while preserving legacy command compatibility.

**Architecture:** Keep the existing skill name, folder, data model, and persistence behavior. Normalize an optional leading `mark` in `SKILL.md`, define all canonical operations as direct subcommands, and update user-facing documentation and OpenAI metadata to teach only the canonical interface.

**Tech Stack:** Agent Skills Markdown, YAML metadata, PowerShell validation, Git

---

## File Map

- Modify `trailmap/SKILL.md`: define canonical direct subcommands, retain one legacy normalization rule, and preserve all behavioral rules.
- Modify `README.md`: document canonical Codex and Claude Code invocation in English.
- Modify `README.zh-CN.md`: document the same canonical interface in Chinese.
- Modify `trailmap/agents/openai.yaml`: make the default prompt demonstrate a direct Trailmap subcommand.
- No schema, migration, runtime script, or test fixture files are added.

### Task 1: Convert The Skill To Direct Subcommands

**Files:**
- Modify: `trailmap/SKILL.md:141-363`

- [ ] **Step 1: Run the pre-change contract check and verify it fails**

Run:

```powershell
$text = Get-Content -Raw trailmap/SKILL.md
if ($text -match '### `mark show`') { throw 'Canonical command headings still contain mark' }
```

Expected: command fails with `Canonical command headings still contain mark`.

- [ ] **Step 2: Add the canonical invocation and compatibility rules**

At the beginning of `## Commands`, define the parser contract once:

```markdown
Treat arguments after the explicit skill invocation as direct subcommands. In Claude Code, use `/trailmap <subcommand>`; in Codex, use `$trailmap <subcommand>`. An invocation without arguments creates a branch.

For backward compatibility, ignore one optional leading `mark` before parsing. Accept legacy and natural-language forms, but never present them as the recommended syntax.
```

Do not add repeated legacy examples.

- [ ] **Step 3: Rename command headings and canonical examples**

Use these headings:

```markdown
### No subcommand
### `pending <idea>`
### `list`
### `show [key]`
### `update <key>`
### `resume <key> clean|informed`
### `close <key>`
### `rename <title>`
### `map [text]`
```

Within the command section, replace canonical command examples as follows:

```text
pending 可能是缓存写入顺序导致，先记下来
show <key>
resume <topic_id> <key> clean
resume <topic_id> <key> informed
map
map text
```

Rewrite prose references such as `Use mark resume` to `Use the resume subcommand`. Keep natural-language examples only where they clarify `pending` behavior.

- [ ] **Step 4: Update the confirmation command lists**

Use direct subcommands in the write/read lists:

```text
Write: no subcommand, pending, update, resume, close, rename
Read-only: list, show, map
```

Preserve the rule that every state-changing operation requires a draft and explicit confirmation.

- [ ] **Step 5: Run focused Skill checks**

Run:

```powershell
$text = Get-Content -Raw trailmap/SKILL.md
$required = @(
  'name: trailmap',
  'optional leading `mark`',
  '### `pending <idea>`',
  '### `list`',
  '### `show [key]`',
  '### `update <key>`',
  '### `resume <key> clean|informed`',
  '### `close <key>`',
  '### `rename <title>`',
  '### `map [text]`'
)
foreach ($item in $required) {
  if (-not $text.Contains($item)) { throw "Missing required Skill text: $item" }
}
if ($text -match '### `mark') { throw 'Legacy mark prefix remains in a canonical heading' }
```

Expected: exit code `0`, with no output.

- [ ] **Step 6: Review behavioral invariants**

Run:

```powershell
rg -n '\.trailmap/marks|\.codex/marks|active|pending|paused|closed|done|blocked|discarded|clean|informed|explicit user confirmation|Never auto-stash' trailmap/SKILL.md
```

Expected: all storage, status, resume-mode, confirmation, and Git-safety concepts remain present.

- [ ] **Step 7: Commit the Skill command change**

```powershell
git add trailmap/SKILL.md
git commit -m "Use direct Trailmap subcommands"
```

### Task 2: Update User Documentation And Metadata

**Files:**
- Modify: `README.md:42-89`
- Modify: `README.zh-CN.md:70-387`
- Modify: `trailmap/agents/openai.yaml:4`

- [ ] **Step 1: Run the pre-change documentation check and verify it fails**

Run:

```powershell
$readmes = (Get-Content -Raw README.md) + (Get-Content -Raw README.zh-CN.md)
if ($readmes -match '(?m)^mark (show|list|pending|update|resume|close|rename|map)') {
  throw 'README still recommends the legacy mark command layer'
}
```

Expected: command fails with `README still recommends the legacy mark command layer`.

- [ ] **Step 2: Replace the English canonical command block**

Document one form per platform. The command summary must contain:

```text
Claude Code:
/trailmap
/trailmap pending <idea>
/trailmap list
/trailmap show [key]
/trailmap update <key>
/trailmap resume <key> clean|informed
/trailmap close <key> done|blocked|discarded
/trailmap rename <topic title>
/trailmap map [text]

Codex:
$trailmap
$trailmap pending <idea>
$trailmap list
$trailmap show [key]
$trailmap update <key>
$trailmap resume <key> clean|informed
$trailmap close <key> done|blocked|discarded
$trailmap rename <topic title>
$trailmap map [text]
```

Update nearby prose to refer to `pending`, `resume`, and other Trailmap subcommands without teaching `mark ...` aliases.

- [ ] **Step 3: Replace all Chinese usage examples with platform-canonical forms**

Use Claude Code forms where the section is explicitly about Claude Code and Codex forms for the main workflow/examples. Representative examples must become:

```text
$trailmap
$trailmap pending 可能是服务端限流导致，先记下来，不切换当前路径
$trailmap update A 我检查了 token TTL 和刷新逻辑，没发现明显异常，没有改代码，A 暂停。
$trailmap resume B clean
$trailmap map
```

In prose, use terms such as `pending 子命令` or canonical inline forms such as `$trailmap resume B clean`; do not recommend standalone `mark ...` commands.

- [ ] **Step 4: Update the OpenAI default prompt**

Set `trailmap/agents/openai.yaml` to:

```yaml
interface:
  display_name: "Trailmap"
  short_description: "Track branch paths and resume context"
  default_prompt: "Use $trailmap pending to record this alternative without changing the active path."
```

- [ ] **Step 5: Validate documentation and metadata consistency**

Run:

```powershell
$files = @('README.md', 'README.zh-CN.md')
foreach ($file in $files) {
  $text = Get-Content -Raw $file
  foreach ($command in @('pending', 'list', 'show', 'update', 'resume', 'close', 'rename', 'map')) {
    if (-not $text.Contains("/trailmap $command")) { throw "$file missing Claude command: $command" }
    if (-not $text.Contains("`$trailmap $command")) { throw "$file missing Codex command: $command" }
  }
  if ($text -match '(?m)^mark(\s|$)') { throw "$file recommends legacy mark syntax" }
  if ($text.Contains('/trailmap mark') -or $text.Contains('$trailmap mark')) {
    throw "$file recommends a redundant mark layer"
  }
}
$yaml = Get-Content -Raw trailmap/agents/openai.yaml
if (-not $yaml.Contains('Use $trailmap pending')) { throw 'OpenAI default prompt is stale' }
```

Expected: exit code `0`, with no output.

- [ ] **Step 6: Commit documentation and metadata**

```powershell
git add README.md README.zh-CN.md trailmap/agents/openai.yaml
git commit -m "Document direct Trailmap invocation"
```

### Task 3: Validate And Synchronize The Installed Skill

**Files:**
- Verify: `trailmap/SKILL.md`
- Verify: `README.md`
- Verify: `README.zh-CN.md`
- Verify: `trailmap/agents/openai.yaml`
- Synchronize outside repository after approval: `C:\Users\Administrator\.codex\skills\trailmap\`

- [ ] **Step 1: Run the official Skill validator**

Run:

```powershell
python "$env:USERPROFILE\.codex\skills\.system\skill-creator\scripts\quick_validate.py" trailmap
```

Expected: validator reports success. If it fails only because `yaml`/PyYAML is unavailable, record that exact limitation and continue with Step 2; do not install dependencies without approval.

- [ ] **Step 2: Run equivalent static validation**

Run:

```powershell
$skill = Get-Content -Raw trailmap/SKILL.md
if ($skill -notmatch '(?s)^---\r?\nname: trailmap\r?\ndescription: .+?\r?\n---') {
  throw 'Required Skill frontmatter fields are missing'
}
if ($skill -match '### `mark') { throw 'Legacy canonical headings remain' }
if (-not $skill.Contains('optional leading `mark`')) { throw 'Compatibility rule is missing' }
```

Expected: exit code `0`, with no output.

- [ ] **Step 3: Inspect the complete repository diff**

Run:

```powershell
git diff HEAD~2 -- trailmap/SKILL.md trailmap/agents/openai.yaml README.md README.zh-CN.md
git status --short
```

Expected: only the four intended implementation files differ across the two implementation commits. Existing unrelated untracked files remain untouched.

- [ ] **Step 4: Synchronize the local Codex installation after approval**

Copy only the verified skill package:

```powershell
Copy-Item -Recurse -Force trailmap\* "$env:USERPROFILE\.codex\skills\trailmap\"
```

This writes outside the workspace and therefore requires explicit approval. Do not delete the destination directory.

- [ ] **Step 5: Verify the installed copy matches the repository**

Run:

```powershell
$repoHash = Get-FileHash trailmap/SKILL.md, trailmap/agents/openai.yaml
$installedHash = Get-FileHash "$env:USERPROFILE\.codex\skills\trailmap\SKILL.md", "$env:USERPROFILE\.codex\skills\trailmap\agents\openai.yaml"
if ($repoHash[0].Hash -ne $installedHash[0].Hash) { throw 'Installed SKILL.md does not match' }
if ($repoHash[1].Hash -ne $installedHash[1].Hash) { throw 'Installed openai.yaml does not match' }
```

Expected: exit code `0`, with no output. A new Codex session may be required for refreshed skill discovery.

## Completion Criteria

- Canonical usage is `/trailmap <subcommand>` in Claude Code and `$trailmap <subcommand>` in Codex.
- No-argument invocation still creates a branch.
- One optional leading `mark` remains accepted for compatibility.
- README files teach only canonical syntax.
- Data model, persistence, confirmation, status, context-resume, and Git-safety behavior are unchanged.
- Repository validation and installed-copy hash checks pass, with any unavailable validator dependency explicitly reported.
