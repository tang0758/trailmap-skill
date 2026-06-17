---
name: trailmap
description: Track decision branches and exploration paths during Codex conversations. Use when the user wants to mark a branching point, remember skipped alternatives, add a pending path without switching away from the current path, list pending paths, resume a previous path with clean or informed context, record path progress, close a path, rename the active topic, or generate a Mermaid/text map of explored and pending branches.
---

# Trailmap

Manage decision branches for a workspace. Persist branch state under `.codex/marks/`. Treat records as user-confirmed artifacts: generate drafts first, ask the user to confirm or edit, then write to disk.

## Storage

Use workspace-local storage:

```text
.codex/marks/
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

Use a tree model. Decisions and paths are nodes. Keep internal IDs stable and user-facing path keys short.

Topic:

```json
{
  "id": "login-timeout",
  "title": "登录超时排查",
  "status": "open",
  "source": "2026-06-16 当前 Codex 会话",
  "active_path_id": "path-20260616-001",
  "nodes": []
}
```

Decision node:

```json
{
  "node_id": "decision-20260616-001",
  "type": "decision",
  "title": "登录超时可能来自 token 或网络",
  "parent_id": null,
  "common_context": {
    "confirmed": [],
    "assumptions": [],
    "constraints": []
  },
  "children": ["path-20260616-001", "path-20260616-002"],
  "related_node_ids": []
}
```

Path node:

```json
{
  "node_id": "path-20260616-001",
  "type": "path",
  "path_key": "A",
  "title": "检查 token 过期逻辑",
  "status": "active",
  "goal": "验证登录超时是否由 token 过期或刷新失败导致",
  "hypothesis": "401 可能来自 token 刷新失败",
  "parent_id": "decision-20260616-001",
  "notes": [],
  "related_node_ids": []
}
```

Path update:

```json
{
  "time": "2026-06-16T10:30:00+08:00",
  "summary": "检查了 token TTL 和刷新逻辑。",
  "conclusion": "暂未发现 token 刷新失败证据。",
  "status_after": "paused",
  "codechange": {
    "changed": false,
    "files": [],
    "note": "只读代码，无修改。"
  }
}
```

Valid path statuses:

```text
active
pending
paused
done
blocked
discarded
```

## Commands

Support natural language around these command forms.

### `mark`

Create a decision branch.

If no active topic exists, draft a new topic and root decision. If an active topic exists, add the new decision under the current active path by default. When creating under an active path, draft a leave-summary for the current path, mark it `paused`, and make the chosen new child path `active` after confirmation.

Path key rules:

- Let AI choose short keys by default, usually `A/B/C`.
- Allow explicit keys such as `P1/P2`.
- Check uniqueness within the topic.
- Never overwrite an existing path key.

Before writing, show a confirmation draft containing topic, decision, common context, paths, active path, and any status changes.

### `mark pending <idea>`

Add a new pending path beside the current active path without switching work context.

Use this when the user is working on one path and temporarily thinks of another possibility that should be remembered for later, while the current `active_path_id` must remain unchanged.

Behavior:

- Require an active topic and active path.
- Find the parent decision of the current active path.
- Add one new path node as another child of that same parent decision.
- Set the new path status to `pending`.
- Keep the current active path status as `active`.
- Keep `active_path_id` unchanged.
- Do not draft a leave-summary for the current active path.
- Do not create a new decision node unless the user explicitly asks for a new decision point.

Accept natural language equivalents such as:

```text
mark pending 可能是缓存写入顺序导致，先记下来
mark 先不要切走，旁边加一个 pending：检查网络重试
mark 继续当前 A，但把服务端限流作为待回溯路径记录
```

Before writing, show a confirmation draft containing the current active path, the parent decision, the new pending path key/title/goal/hypothesis, and the unchanged `active_path_id`.

### `mark list`

Show open topics in the workspace and actionable paths only:

```text
active
pending
paused
blocked
```

Hide `done` and `discarded` paths. Include codechange warnings when known.

### `mark show`

Default: show active topic summary, active path details, recent updates, and pending/paused/blocked paths.

Also support:

```text
mark show <path_key>
mark show <topic_id>
```

Resolve `<path_key>` against the active topic first; if no path matches, resolve as topic id.

### `mark update <path_key>`

Generate a structured update draft for the path:

```text
time
summary
conclusion
status_after
codechange.changed
codechange.files
codechange.note
```

Use git/workspace context to infer changed files when possible, but do not require it. After confirmation, append the update to `notes` and set the path status to `status_after`.

### `mark resume <path_key> clean|informed`

Switch active work path. `clean` and `informed` are required unless the user clearly states the mode in natural language.

Before switching:

1. Identify current active path.
2. Draft a leave-summary update for the current path.
3. Default its `status_after` to `paused`.
4. Include codechange summary and warnings.
5. Ask for confirmation.
6. After confirmation, write the leave-summary, set old path status, set target path `active`, and update `active_path_id`.

If target path is `done` or `discarded`, warn that it is closed and ask for explicit reopen confirmation. If confirmed, add a reopen note and set it active.

For cross-topic resume, support:

```text
mark resume <topic_id> <path_key> clean
mark resume <topic_id> <path_key> informed
```

This updates `active_topic_id` and the topic's `active_path_id`.

### Resume Context Modes

`clean` includes only:

- decision common context
- target path title, goal, hypothesis
- target path's own notes
- constraints
- warnings about existing code changes

Do not include detailed context from sibling paths.

`informed` includes everything from `clean`, plus summarized conclusions from sibling or previously explored paths. Clearly label this material as "other-path context" to avoid contaminating the target path.

Never auto-stash, revert, commit, or otherwise change git state. If code changes are recorded, warn and provide a checklist only.

### `mark close <path_key>`

Close a path after confirmation. Require one of:

```text
done
blocked
discarded
```

Draft and confirm:

```text
close status
close reason
final summary
codechange summary if relevant
```

Then update path status and append a closing note.

### `mark rename <title>`

Rename only the active topic. Confirm before writing. Do not rename path keys or path titles in v1.

### `mark map`

Output Mermaid mindmap for the active topic by default.

```text
mark map
```

Use Mermaid `mindmap` only. Include decisions, paths, statuses, and compact update summaries.

```text
mark map text
```

Output a plain text tree only.

If multiple topics exist, default to active topic.

## Confirmation Rule

Any operation that writes or changes state must first show a draft and wait for explicit user confirmation.

Write operations include:

```text
mark
mark pending
mark update
mark resume
mark close
mark rename
```

Read-only operations do not require confirmation:

```text
mark list
mark show
mark map
```

## Non-Goals

Do not automatically roll back code, stash changes, commit changes, or create git branches.

Do not directly push to Notion in v1. Mermaid and text output should be easy to copy into Notion or other documentation tools.

Do not record every chat turn. Record decision points and path-level updates only.
