---
name: trailmap
description: Use when the user explicitly invokes Codex $trailmap or Claude Code /trailmap to remember alternatives, track pending or paused paths, resume a recorded path, or view a decision tree.
---

# Trailmap Lite

Trailmap is a recording-only command skill. It records, selects, pauses or resumes, closes, and displays paths only after an explicit `$trailmap ...` or `/trailmap ...` invocation.

Do not solve the underlying problem, request logs, inspect business code or Git, generate plans or advice, orchestrate agents, monitor work, or proactively remind the user. Never run a recorded path. After the concise command response, stop; the normal agent handles later work outside Trailmap.

## Storage Model

Store each Topic independently at `.trailmap/topics/<topic-id>.json`. Do not create `index.json`, persist a selected Topic globally, migrate older records, or modify another Topic file.

Store only these Topic fields:

```json
{
  "id": "login-failure",
  "title": "Production login failure",
  "active": null,
  "paths": []
}
```

Store every path with `key`, `title`, `status`, `parent`, `note`, and `updates`. Use `null` for an absent parent and `""` for an absent path note. Add `closed_as`, `closed_reason`, and `closed_at` only while the path is closed.

Each update contains `time` and `note`, plus optional `status_after`. Include update `closed_as` only when `status_after` is `closed`. Preserve supplied titles, notes, and reasons exactly. Use an ISO 8601 timestamp for `time` and `closed_at`.

Allowed statuses are `active`, `pending`, `paused`, and `closed`. Display a closed path by its human classification: `done`, `blocked`, or `discarded`.

Invariants:

- A Topic has at most one `active` path.
- `topic.active` is `null` or equals the key of that sole active path.
- Path keys are unique within a Topic and never change.
- Topic IDs and filenames never change.

## Topic Selection

For commands needing a current Topic, use the latest valid `Topic: <id> | <title>` marker in the current chat. A valid marker identifies an existing Topic file. Never infer a Topic from conversational context.

When no marker exists:

- Zero Topic files: ask for `new <topic-title>` and do not fabricate a marker.
- One Topic file: select it.
- Multiple Topic files: show only their IDs and titles and require `use <topic-id>`; do not guess.

After selection, every response begins exactly:

```text
Topic: <id> | <title>
```

`new <title> [--id <id>]` writes immediately. An explicit ID must match `^[a-z0-9]+(?:-[a-z0-9]+)*$`, be unused, and never overwrite a file. Without `--id`, generate a short meaningful ASCII slug from the title, translating or transliterating non-ASCII titles when needed, and enforce the same pattern. If no clear slug can be generated, require an explicit ID. On collision, choose the first available `-2`, `-3`, and so on. Creation never changes any existing file.

`use <id>` verifies and reads that Topic, emits its marker, and writes nothing. Selection lasts only through the marker in chat.

## Command Grammar

Arguments after the explicit invocation must match one command exactly. No arguments means `list`.

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

Reject a `mark` prefix, natural-language branching forms, context-mode arguments, execution commands or flags, delete/remove requests, unknown commands, and unsupported options. For delete/remove, only suggest the applicable `close <key> discarded [reason]` form. Do not reinterpret or migrate invalid input; write nothing.

Explicit, unambiguous write commands write immediately without confirmation. The sole draft exception is `update <key>` with neither a note nor `--pause`, as described below.

## Creating Paths

For `pending`, store `<title>` verbatim and store a path-level note only when `--note` is explicitly supplied. Never rewrite the title or generate a goal, hypothesis, explanation, or leave summary.

Check an explicit `--key` for uniqueness. Otherwise choose the shortest clear unused key, preferring `A`, `B`, `C` for roots and `<parent>1`, `<parent>2` for children. Once assigned, a key is immutable.

Creation rules:

- Empty Topic: the first path is a root, becomes `active`, and sets `topic.active`.
- Nonempty Topic with no active path: create a root `pending` path unless `--parent <key>` names an existing parent.
- Active path and no placement option: create a `pending` sibling with `parent` equal to the active path's parent; leave the active path unchanged.
- `--child`: create a `pending` child of the current active path; reject when there is no active path.
- `--parent <key>`: create a `pending` child of that exact existing path.
- `--child` and `--parent` are mutually exclusive.

## Read Commands

`list` shows every path in the current Topic, compactly grouped by state, with key, verbatim title, and human state. No arguments is identical.

`list all` shows Topic summaries only: ID, title, active key, and path counts. It does not change selection.

`show` displays the current active path's details. If no path is active, show a concise Topic summary. `show <key>` uses an exact key lookup and displays that path's exact title, note, updates, parent, status, and closure fields.

`map` emits a Mermaid `graph LR` rooted at the Topic. Derive edges from `parent`; labels contain the original key, verbatim title, and human state. Derive safe unique Mermaid node IDs from keys by replacing unsupported characters, while preserving original keys in labels. `map text` emits the equivalent plain-text tree. Neither form adds work advice.

## Write Commands

### `update`

`update <key> <note>` appends one update containing the exact supplied note and current time. It does not change status or infer any other field.

With no note and no `--pause`, compress only the current chat into one short note draft. Do not inspect code or Git and do not infer status. Show the draft and ask for explicit Trailmap confirmation; write nothing until that confirmation.

`--pause` is accepted only when the exact target is the current active path. Append the note with `status_after: "paused"` if supplied, set the path to `paused`, and set `topic.active` to `null`. With no note, append no update. Reject `--pause` for pending, paused, closed, or non-current paths without changing anything.

### `resume`

Only pending or paused paths can be resumed normally. Pause the old active path, activate the target, and set `topic.active` to the target key. Generate no summary or update. If `--note` is supplied, append it exactly to the old active path only; reject that option when there is no old active path to receive it.

For a closed target without `--reopen`, write nothing. Display its classification and reason, then this exact guidance with the real key substituted:

```text
resume <key> --reopen --note "<重开原因>"
```

Reopening requires a nonempty `--note`. Pause the old active path, activate the target, and set `topic.active` to the target key. Ensure the prior top-level closure fact remains in an update with `status_after: "closed"` and its `closed_as`; retain an existing matching closing update or add one from the closure fields. Then remove top-level closure fields and append the exact reopen note as a new update with `status_after: "active"`. Do not generate any other context.

### `close`

Require exactly one classification: `done`, `blocked`, or `discarded`. Store the supplied reason verbatim; when omitted, store and display `未填写关闭原因` without inference. Append a closing update with the same timestamp and reason, `status_after: "closed"`, and `closed_as`. Set the path status and top-level closure fields. If it was active, set `topic.active` to `null`; never activate another path.

### `rename`

Change only the current Topic's title to the verbatim supplied title, and use the new title in the response marker. Do not change its ID, filename, path keys, path titles, or any other Topic.

## Write Safety

For every existing-Topic write:

1. Read and parse the complete JSON; immediately hash the complete file bytes with SHA-256 and retain that hash only for this operation.
2. Find paths by exact key, snapshot every path status, and modify only the target plus an explicitly required old active path and Topic fields.
3. Serialize the complete object, parse it again, and verify all invariants and the intended status changes. Reject unexpected sibling or parent changes.
4. Immediately before replacement, reread the complete file bytes and compute SHA-256 again. If the hash differs, reject the entire write and instruct the user to retry from the latest file.
5. Replace only the selected Topic file, then reread and verify the serialized result, closure fields, last closing update, status snapshot, and single-active invariant.

For `new`, recheck that the destination is absent immediately before a no-overwrite creation; reject a collision.

Never store a revision field. SHA-256 reread is best-effort conflict detection, not atomic compare-and-swap or locking. It cannot detect the race where both writers verify before either replacement. Advise users to avoid simultaneous writes to the same Topic and retry after any conflict.

Never modify business code, Git state, or unrelated Topic files.

## Output Discipline

After a Topic is selected, the marker is always first. A successful write normally has exactly one result line after it. A successful `resume` keeps that line compact while also showing the target title, its note, and up to three most recent updates; add no advice.

Errors, closed-target responses, draft confirmation, and conflict responses may add one necessary instruction. Read commands may expand only as needed for their specified data. Never dump JSON, explain implementation in the response, suggest next work, or promise to execute a path.
