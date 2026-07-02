# Trailmap Lite Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Replace Trailmap's execution-oriented behavior with a lightweight, workspace-local path recorder that appears only when explicitly invoked.

**Architecture:** Keep `trailmap/SKILL.md` as the portable Agent Skill entrypoint for Codex and Claude Code. Store each visible Topic in an independent `.trailmap/topics/<topic-id>.json` file, keep the selected Topic in the visible chat marker rather than global workspace state, and use a small fixed command set for recording and displaying paths. The repository remains documentation-only; behavior is specified and regression-tested with agent scenarios plus static content checks.

**Tech Stack:** Agent Skills Markdown, JSON records, Mermaid `graph LR`, Git, PowerShell verification commands.

---

### Task 1: Capture the Lite regression contract

**Files:**
- Create: `docs/testing/lite-regression.zh-CN.md`

- [ ] **Step 1: Record the RED baseline scenarios and observed failures**

Create a test document containing the three baseline prompts and these observed failures:

```text
pending baseline: Agent recorded B, then asked for status code, response body, and logs to continue A.
resume baseline: Agent changed old A to pending, reopened closed B implicitly, and announced immediate investigation.
update baseline: Agent acknowledged content but provided no Topic marker or persistence contract.
```

- [ ] **Step 2: Define the GREEN acceptance scenarios**

The document must require:

```text
1. pending stores the user title verbatim, leaves the active path unchanged, and offers no debugging advice.
2. resume changes old active to paused and target to active, but a closed target only prints the explicit --reopen command.
3. update records only the supplied note, does not inspect code or Git, and always prints Topic: <id> | <title>.
4. no invocation means no Trailmap participation.
5. restored chats recover selection from the latest Topic marker; new chats use use <topic-id>.
6. different Topics use different files; there is no global active_topic_id or index.json.
```

- [ ] **Step 3: Verify the contract is RED against the old skill**

Run:

```powershell
rg -n "agent_run|worktree|codechange|clean|informed|active_topic_id|\.trailmap/marks" trailmap/SKILL.md
```

Expected: matches are found, proving the old skill violates the Lite contract.

- [ ] **Step 4: Commit the regression contract**

```powershell
git add docs/testing/lite-regression.zh-CN.md
git commit -m "test: define Trailmap Lite behavior"
```

### Task 2: Rewrite the skill as a recorder

**Files:**
- Modify: `trailmap/SKILL.md`
- Modify: `trailmap/agents/openai.yaml`

- [ ] **Step 1: Replace the frontmatter and core boundary**

Use this trigger-only frontmatter:

```yaml
---
name: trailmap
description: Use when the user wants to remember alternative paths, track pending or paused directions, resume a recorded path, or view a decision tree without asking the agent to execute those paths.
---
```

The opening rule must state that Trailmap only records, switches, closes, and displays paths after explicit `$trailmap` or `/trailmap` invocation. It must forbid problem solving, code/Git inspection, implementation advice, automatic reminders, and execution orchestration.

- [ ] **Step 2: Define the minimal storage model**

Specify `.trailmap/topics/<topic-id>.json` with this shape:

```json
{
  "id": "login-failure",
  "title": "登录失败排查",
  "active": "A",
  "paths": [
    {
      "key": "A",
      "title": "检查 token 刷新",
      "status": "active",
      "parent": null,
      "note": "",
      "updates": []
    }
  ]
}
```

Updates contain `time`, `note`, optional `status_after`, and `closed_as` only when `status_after` is `closed`. Closed paths additionally contain top-level `closed_as`, optional `closed_reason`, and `closed_at`.

- [ ] **Step 3: Define Topic selection without global state**

Every response must start with:

```text
Topic: <id> | <title>
```

The latest marker in the current chat selects the Topic. If absent, auto-select the only Topic, request `use <topic-id>` when several exist, or request `new` when none exist. Do not create `index.json` or `active_topic_id`.

- [ ] **Step 4: Implement the exact command surface**

Document only:

```text
new <topic-title> [--id <id>]
use <topic-id>
pending <title> [--key <key>] [--note <note>] [--child | --parent <key>]
list [all]
show [key]
update <key> [note] [--pause]
resume <key> [--note <note>]
resume <key> --reopen --note <reason>
close <key> <done|blocked|discarded> [reason]
rename <topic-title>
map [text]
```

No arguments are equivalent to `list`. Do not retain `mark`, clean/informed context modes, subagent flags, worktree flags, legacy natural-language branching, or migration behavior.

- [ ] **Step 5: Encode lifecycle and tree invariants**

Require unique immutable keys, one active path at most, `active/pending/paused/closed` storage states, human display of closed paths as `done/blocked/discarded`, sibling-by-default pending paths, explicit child/parent placement, and explicit `--reopen` for closed paths. `resume` must not summarize the old path unless `--note` is supplied.

- [ ] **Step 6: Encode concise write and concurrency behavior**

Explicit commands write immediately. Only AI-generated `update <key>` text, ambiguity, or invalid input requires confirmation. Before each write, reread the target Topic, modify only the exact path key, reject a concurrent change rather than overwrite it, serialize the complete JSON, and verify active/status invariants.

- [ ] **Step 7: Update the Codex agent metadata**

Use:

```yaml
interface:
  display_name: "Trailmap Lite"
  short_description: "Remember paths without executing them"
  default_prompt: "Use $trailmap pending to record this alternative without changing the active path."
```

- [ ] **Step 8: Run static GREEN checks**

Run:

```powershell
rg -n "agent_run|worktree|codechange|clean|informed|active_topic_id|\.trailmap/marks|optional leading mark" trailmap/SKILL.md
rg -n "^###? `(new|use|pending|list|show|update|resume|close|rename|map)" trailmap/SKILL.md
```

Expected: the forbidden-term command exits with no matches; the command scan covers the complete Lite surface.

- [ ] **Step 9: Commit the Lite skill**

```powershell
git add trailmap/SKILL.md trailmap/agents/openai.yaml
git commit -m "feat: simplify Trailmap to path recording"
```

### Task 3: Replace execution-version documentation

**Files:**
- Modify: `README.md`
- Modify: `README.zh-CN.md`
- Modify: `docs/USAGE.md`
- Modify: `docs/USAGE.zh-CN.md`
- Modify: `CHANGELOG.md`
- Delete: legacy files under `docs/superpowers/` except this plan
- Delete: legacy files under `docs/testing/` except `lite-regression.zh-CN.md`

- [ ] **Step 1: Rewrite both product introductions**

Both READMEs must explain the same product boundary, statuses, Topic marker, concise workflow, Codex `$trailmap` and Claude Code `/trailmap` invocation forms, workspace-local storage, and `lite-v0.1.0` release line. They must not describe Trailmap as an executor or context loader.

- [ ] **Step 2: Rewrite both usage guides**

Both guides must document every command from Task 2 with matching examples and cover Topic recovery, `list` versus `list all`, sibling/child creation, pause, explicit reopen, closing classifications, Mermaid/text maps, no automatic analysis, and concurrent same-Topic write rejection.

- [ ] **Step 3: Reset the changelog for the Lite line**

Create a `lite-v0.1.0` entry dated 2026-07-02 that calls out the recording-only scope and the intentional removal of execution features and old data compatibility.

- [ ] **Step 4: Remove execution-only documents**

Delete old subagent/worktree specs, plans, and regression documents. Preserve this Lite plan and the Lite regression contract.

- [ ] **Step 5: Verify bilingual parity and forbidden concepts**

Run:

```powershell
rg -n "subagent|worktree|clean|informed|codechange|active_topic_id|\.trailmap/marks" README.md README.zh-CN.md docs/USAGE.md docs/USAGE.zh-CN.md CHANGELOG.md
rg -n "new|use|pending|list|show|update|resume|close|rename|map" README.md README.zh-CN.md docs/USAGE.md docs/USAGE.zh-CN.md
```

Expected: forbidden execution concepts appear only in the changelog's explicit removal note; each language contains the complete command vocabulary.

- [ ] **Step 6: Commit the Lite documentation**

```powershell
git add README.md README.zh-CN.md docs/USAGE.md docs/USAGE.zh-CN.md CHANGELOG.md docs/superpowers docs/testing
git commit -m "docs: publish Trailmap Lite guidance"
```

### Task 4: Run GREEN agent scenarios and final verification

**Files:**
- Modify only if a GREEN scenario exposes a concrete gap: `trailmap/SKILL.md`, relevant bilingual docs, or `docs/testing/lite-regression.zh-CN.md`

- [ ] **Step 1: Re-run the three RED prompts with the Lite skill supplied**

Expected behavior:

```text
pending: record B only, keep A active, print Topic marker, provide no debugging request.
resume: report that B is closed and print resume B --reopen --note "<reason>" without changing state.
update: record the supplied note only, print Topic marker, and do not inspect Git or create codechange fields.
```

- [ ] **Step 2: Run variation scenarios**

Test restored-chat Topic marker recovery, new-chat `use`, multiple Topics with no marker, `list all`, `pending --child`, `update --pause`, closed-path reopen, and Mermaid `graph LR` output.

- [ ] **Step 3: Close any observed loopholes and re-run affected scenarios**

Only add rules that correspond to an observed failure. Repeat until agents comply without execution advice.

- [ ] **Step 4: Run repository verification**

```powershell
git diff --check
rg -n "<<<<<<<|=======|>>>>>>>" .
rg -n "agent_run|worktree|codechange|clean|informed|active_topic_id|\.trailmap/marks|optional leading mark" trailmap/SKILL.md
git status --short
```

Expected: no whitespace errors, conflict markers, placeholders, or forbidden Lite skill concepts; status contains only intentional changes before the final commit.

- [ ] **Step 5: Request final spec and quality review**

Review the complete diff against this plan, with special attention to accidental execution behavior, mismatched bilingual commands, invalid state transitions, and overlong output requirements.

- [ ] **Step 6: Commit final refinements if needed**

```powershell
git add trailmap README.md README.zh-CN.md docs CHANGELOG.md
git commit -m "test: verify Trailmap Lite behavior"
```
