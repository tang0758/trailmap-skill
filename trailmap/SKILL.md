---
name: trailmap
description: Track decision branches and exploration paths during AI coding agent conversations. Use when the user wants to mark a branching point, remember skipped alternatives, add a pending path without switching away from the current path, list paths, resume a previous path with clean or informed context, record path progress, close a path, rename the active topic, or generate a Mermaid/text map of explored and pending branches.
---

# Trailmap

Manage decision branches for a workspace. Persist branch state under `.trailmap/marks/` by default. If an existing `.codex/marks/` directory is already present, continue using it for backward compatibility unless the user asks to migrate. Treat records as user-confirmed artifacts: generate drafts first, ask the user to confirm or edit, then write to disk.

## Storage

Use workspace-local storage:

```text
.trailmap/marks/
  index.json
  <topic-id>.json
```

`index.json` stores workspace state:

```json
{
  "active_topic_id": "login-timeout",
  "recent_topic_ids": ["login-timeout"]
}
```

Each topic file stores one independent problem or work line. A topic is not tied one-to-one to a chat; it may span chats, and one chat may contain multiple topics.

## Data Model

Use a flat path list with parent references. Do not create decision nodes, path node IDs, `path_key`, `children`, `parent_id`, `related_node_ids`, or `topic.status`.

Topic:

```json
{
  "id": "login-timeout",
  "title": "登录超时排查",
  "active": "A",
  "source": "2026-06-17 当前会话：排查登录超时",
  "paths": []
}
```

Rules:

- `active` is `null` or the `key` of the current active path.
- If `active` is not `null`, exactly one path must have `status: "active"` and the same `key`.
- If all paths are closed, set `active` to `null`.
- Topic open/closed state is derived from paths: a topic is open when any path is not closed.
- `source` is optional and human-readable; do not depend on platform conversation IDs.

Path:

```json
{
  "key": "A",
  "title": "检查 token 过期",
  "status": "active",
  "parent": null,
  "created_from": {
    "reason": "登录超时可能来自 token 或网络；A 是 token 方向。",
    "confirmed": [
      "登录后约 30 分钟超时",
      "接口返回 401"
    ],
    "assumptions": [
      "可能是 token 刷新失败",
      "也可能是网络重试导致错误映射"
    ],
    "constraints": [
      "不自动回滚代码"
    ]
  },
  "goal": "验证是否由 token 过期或刷新失败导致",
  "hypothesis": "401 可能来自 token 刷新失败",
  "updates": []
}
```

Rules:

- `key` is the only path identifier. It must be unique inside a topic.
- Do not support path key rename in v1.
- `parent` is `null` for root paths, or another path's `key` for child paths.
- Do not store `children`; derive children with `paths where parent == key`.
- Keep `title`, `goal`, and `hypothesis` separate in storage. General human summaries may merge them into a concise path description; confirmation drafts follow the command-specific fields below.
- Keep `created_from` on each path. Paths created from the same branch may copy the same `created_from` with path-specific `reason` text.

Path statuses:

```text
active
pending
paused
closed
```

Closed path fields:

```json
{
  "status": "closed",
  "closed_as": "discarded",
  "closed_reason": "验证后确认不是主因。",
  "closed_at": "2026-06-17T10:30:00+08:00"
}
```

Rules:

- `closed_as` is required only when `status` is `closed`.
- Valid `closed_as` values are `done`, `blocked`, and `discarded`.
- If a path is reopened, remove top-level `closed_as`, `closed_reason`, and `closed_at`; closure history remains in `updates`.

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
- In human views, append compact execution state such as `[pending, subagent running]`.

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

Path update:

```json
{
  "time": "2026-06-17T10:30:00+08:00",
  "summary": "检查了 token TTL 和刷新逻辑。",
  "conclusion": "暂未发现 token 刷新失败证据。",
  "status_after": "paused",
  "codechange": {
    "changed": false,
    "files": [],
    "summary": "只读代码，无修改。"
  }
}
```

Rules:

- `status_after` must be one of `active`, `pending`, `paused`, or `closed`.
- If `status_after` is `closed`, include `closed_as` in the update.
- After confirmation, append the update and sync the path's top-level status.
- Use `codechange.changed/files/summary`; do not use `codechange.note`.

## Confirmation Draft Presentation

Build the complete structured draft internally, but show a concise human confirmation view by default.

Show only identifiers and titles, decision-relevant goals/hypotheses/summaries/conclusions, resulting statuses or transitions, necessary default-choice explanations, relevant warnings, and one explicit confirmation question at the end.

Hide `source`, `created_from`, `parent`, timestamps, repeated status-change lists, and the complete persistence structure unless a key/parent relationship is ambiguous, a closed path is reopened, code changes may contaminate a clean resume, or the user asks to expand details. Hidden fields remain in the internal draft and are written normally after confirmation.

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

`.worktrees/` must be ignored before creating project-local worktrees. If `.worktrees/` is not ignored, include this exact change in the confirmation draft:

```text
+ .worktrees/
```

After confirmation, Trailmap may create `.gitignore` if missing or append only `.worktrees/` if present. Do not reorder or reformat `.gitignore`.

Trailmap does not automatically merge, cherry-pick, apply patches, copy files, commit, stash, revert, or switch the main workspace branch.

When a worktree subagent reports, keep the worktree by default. Set `agent_run.worktree.status` to `retained`. The first worktree version does not implement a cleanup command and does not remove worktrees automatically.

## Commands

Arguments after explicit skill invocation are direct subcommands. Claude Code uses `/trailmap <subcommand>`; Codex uses `$trailmap <subcommand>`. No arguments creates a branch.

For backward compatibility, ignore one optional leading `mark` before parsing. Accept legacy and natural-language forms, but do not recommend them.

Subagent flags:

- `--subagent` requests subagent exploration for newly created non-main-active paths.
- `--subagent <key>` or `--subagent <key1,key2>` selects specific newly created path keys for subagent exploration.
- `--allow-shared-code` means the user accepts the shared workspace code risk. It skips the separate risk prompt, not the Trailmap write-confirmation draft.
- `--informed` uses informed context for subagent exploration. Without it, subagent context is `clean`.
- `--worktree` requests isolated Git worktree execution for selected subagent paths.
- `--base <ref>` sets the Git ref used to create the worktree; default is current `HEAD`.
- `--worktree` and `--allow-shared-code` are mutually exclusive.
- `--worktree` may be combined with `--informed`.
- Do not support `--no-worktree`; shared workspace is already the default.

### No subcommand

Create a branch.

If no active topic exists, draft a new topic and root paths. Root paths have `parent: null`; one path becomes `active`, and the rest become `pending`.

If an active topic and active path exist, create child paths under the current active path:

- Keep the current active path as a path record.
- Draft a leave-summary update for the current active path.
- Set the current active path to `paused` after confirmation.
- Create new child paths with `parent` equal to the previous active path's `key`.
- Set one child path to `active`; set the others to `pending`.
- Set `topic.active` to the new active child path key.

This is for "the current path has split into child directions" scenarios.

Path key rules:

- Let AI choose short keys by default, usually `A/B/C` for roots or `A1/A2` for children.
- Allow explicit keys such as `P1/P2`.
- Check uniqueness within the topic.
- Never overwrite an existing key.

Before writing, show the topic `id` and title; each new path's key, status, title, goal, and hypothesis; any decision-relevant leave summary; and a short explanation of the default active-path selection.

When new non-main-active paths are created, prompt whether to start subagent exploration for zero, one, or multiple candidate paths unless `--subagent` already selects them. Include a shared workspace code risk warning unless `--allow-shared-code` is present. Selected paths keep their normal lifecycle status and receive `agent_run.status: "running"` after confirmation.

### `pending <idea>`

Add a pending sibling path without switching work context.

Use this when the user is working on one path and temporarily thinks of another possibility that should be remembered for later, while the current `topic.active` must remain unchanged.

Behavior:

- Require an active topic and active path.
- Create one new path with `parent` equal to the current active path's `parent`.
- Set the new path status to `pending`.
- Keep the current active path status as `active`.
- Keep `topic.active` unchanged.
- Do not draft a leave-summary for the current active path.
- Do not create child paths; use no subcommand for child branching.

Accept natural language equivalents such as:

```text
pending 可能是缓存写入顺序导致，先记下来
pending 检查网络重试，先不要切走
pending 服务端限流，继续当前 A
```

Before writing, show the new pending path's key, title, goal, and hypothesis, plus one sentence confirming that the current active path remains unchanged.

For `pending <idea> --subagent`, the new sibling path remains `pending` and receives `agent_run.status: "running"` after confirmation. Include a shared workspace code risk warning unless `--allow-shared-code` is present.

### `list`

Show a cross-topic dashboard for the workspace.

For each topic, group paths by:

```text
active
pending
paused
closed
```

For active, pending, and paused paths, show only `key`, `title`, and compact codechange warning when useful.

For closed paths, show `key`, `title`, `closed_as`, and `closed_reason`. Do not expand full `updates` in list output.

For paths with `agent_run`, append compact execution state such as `[pending, subagent running]` or `[paused, subagent reported]`.

### `show [key]`

Show details for the active topic only.

Default:

- topic title and active path
- active path description, merging title/goal/hypothesis for humans
- recent active path updates
- pending and paused paths
- closed paths with closing reason

Also support:

```text
show <key>
```

Show that path in the active topic, including `created_from`, merged path description, complete `updates`, codechange details, latest `agent_run` fields, subagent handoff/report summary when available, and closure fields if closed.

Do not support `show <topic_id>` in v1.

### `update <key>`

Generate a structured update draft for the path:

```text
time
summary
conclusion
status_after
closed_as, only when status_after=closed
codechange.changed
codechange.files
codechange.summary
```

Use git/workspace context to infer changed files when possible, but do not require it. After confirmation, append the update to `updates` and set the path status to `status_after`.

Before writing, show the path key, summary, conclusion, and `status_after`. If `status_after` is `closed`, also show `closed_as`. Show changed files and the code-change summary only when code changed.

If `status_after` is `closed`, set top-level `closed_as`, `closed_reason`, and `closed_at`. If `status_after` is not `closed`, ensure top-level closure fields are absent.

### `subagent <key>`

Start subagent exploration for an existing path in the active topic.

Validation:

- Require an active topic.
- Require the target path to exist in the active topic.
- Reject the current main active path.
- Reject closed paths.
- Reject paths whose `agent_run.status` is already `running`.
- Reject commands that combine `--worktree` and `--allow-shared-code`.

Before writing, show a concise startup draft with the path key/title/status, context mode, planned `agent_run.status`, and shared workspace code risk. `--allow-shared-code` skips only the separate risk question; it does not skip the write-confirmation draft.

If the subagent tool is unavailable, do not fail the path operation. Write `agent_run.status: "blocked"` with a concise reason after confirmation.

For `--worktree`:

- Resolve `--base <ref>` or current `HEAD` before creating the worktree.
- If the base ref does not resolve, write `agent_run.status: "blocked"` and `agent_run.worktree.status: "failed"` after confirmation; do not fall back to `HEAD`.
- Create one worktree and one branch per selected path.
- Do not copy uncommitted main-workspace changes into the worktree.
- Record `base_dirty: true` when the main workspace has uncommitted changes.

Default worktree path:

```text
.worktrees/trailmap/<topic-id>/<path-key>/
```

Default branch:

```text
trailmap/<topic-id>/<path-key>
```

If the directory exists and is non-empty, or the branch already exists, append the same unique suffix to both. Do not reuse non-empty directories or existing branches.

### `resume <key> clean|informed`

Switch active work path inside the active topic. `clean` and `informed` are required unless the user clearly states the mode in natural language.

Before switching:

1. Identify the current active path.
2. Draft a leave-summary update for the current active path.
3. Default its `status_after` to `paused`.
4. Include codechange summary and warnings.
5. Ask for confirmation.
6. After confirmation, write the leave-summary, set old path status, set target path `active`, and set `topic.active` to the target key.

The confirmation view shows the current path's leave summary, the target path, the selected `clean` or `informed` mode, and the resulting status transition. Preserve relevant code-change warnings, especially when workspace changes may contaminate a `clean` resume.

If target path is `closed`, warn that it is closed and ask for explicit reopen confirmation. If confirmed, append a reopen update, set the target path to `active`, and remove top-level closure fields.

If the target path has `agent_run.status: "running"`, include a warning that the path is currently being explored by a subagent and resuming it in the main session may duplicate work or mix context. Do not block the resume.

For cross-topic resume, support:

```text
resume <topic_id> <key> clean
resume <topic_id> <key> informed
```

This updates `index.active_topic_id` and the target topic's `active`.

### Resume Context Modes

`clean` includes only:

- target path `created_from`
- target path description, merging title/goal/hypothesis for humans
- target path's own updates
- constraints from `created_from`
- warnings about recorded or current code changes

Do not include detailed context from sibling paths.

`informed` includes everything from `clean`, plus summarized conclusions from sibling, parent, or previously explored paths. Clearly label this material as "other-path context" to avoid contaminating the target path.

Never auto-stash, revert, commit, or otherwise change git state. If code changes are recorded, warn and provide a checklist only.

### Subagent Context Modes

Subagent context defaults to `clean`.

`clean` includes the topic title, target path `created_from`, target path description, target path updates, minimal current active path identity, shared workspace code risk, and required report schema. It does not include detailed reasoning from sibling paths.

`--informed` adds summarized sibling, parent, current-active, or previously explored path conclusions. Clearly label this material as "other-path context".

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

### `close <key>`

Close a path after confirmation. Require one of:

```text
done
blocked
discarded
```

Draft and confirm:

```text
key
closed_as
closed_reason
final summary
codechange summary if relevant
```

After confirmation:

- Append a closing update with `status_after: "closed"` and `closed_as`.
- Set the path `status` to `closed`.
- Set top-level `closed_as`, `closed_reason`, and `closed_at`.
- If the closed path was the active path, set `topic.active` to `null`.

Use the resume subcommand to activate another path after closing.

### `rename <title>`

Rename only the active topic. Before writing, show the old topic title and proposed new title. Confirm before writing. Do not rename path keys or path titles in v1.

### `map [text]`

Output a Mermaid `graph LR` for the active topic by default. Build the tree from `paths[].parent`.

```text
map
```

Use Mermaid `graph LR` only. Use `root` for the topic node, and one node per path. Derive Mermaid node ids from path keys; if a key contains characters Mermaid cannot use as an id, replace them with `_`. Node labels must show the original `key`, title, and status; for closed paths show `closed: closed_as`. When a path has `agent_run`, append compact execution state to the label, for example `B Network retry pending / subagent running`. Do not show full updates.

```text
map text
```

Output a plain text tree only, using the same fields.

## Confirmation Rule

Any operation that writes or changes state must first show a draft and wait for explicit user confirmation. Every write draft must end with one explicit confirmation question, such as `Confirm writing the above changes?`. Do not write any state before the user explicitly confirms.

Write operations include:

```text
no subcommand
pending
update
resume
close
rename
```

Read-only operations do not require confirmation:

```text
list
show
map
```

## Non-Goals

Do not automatically roll back code, stash changes, commit changes, or create git branches.

Do not directly push to Notion in v1. Mermaid and text output should be easy to copy into Notion or other documentation tools.

Do not record every chat turn. Record decision points and path-level updates only.
