# Trailmap Concise Confirmation Design

## Goal

Make Trailmap write-operation confirmation drafts concise for humans while preserving the complete structured data written to disk.

## Presentation Model

Every write operation still builds a complete internal draft before confirmation. The default user-facing view shows only information needed to approve or correct the operation:

- short identifiers and titles
- goals, hypotheses, summaries, or conclusions relevant to the decision
- final status or status transition
- a short explanation of any default choice
- one explicit confirmation question at the end

Hide internal or repetitive fields by default:

- `source`
- `created_from`
- `parent`
- timestamps
- repeated status-change lists
- the complete persistence structure

Hidden fields remain part of the internal draft and are written normally after confirmation. Show the complete draft when the user explicitly asks to expand details.

## Command Views

### No Subcommand

Show:

- topic `id` and `title`
- each new path's `key`, `status`, `title`, `goal`, and `hypothesis`
- a short explanation of the default active-path selection

Do not show `source`, `created_from`, `parent`, or a separate status-change section unless needed to resolve ambiguity.

### `pending`

Show the new path's `key`, `title`, `goal`, and `hypothesis`, plus one sentence confirming that the current active path remains unchanged.

### `update`

Show `key`, `summary`, `conclusion`, and `status_after`. Show changed files and the code-change summary only when code changed.

### `resume`

Show the current path's leave summary, the target path, the selected `clean` or `informed` mode, and the resulting status transition. Preserve code-change warnings when workspace state may contaminate a clean context.

### `close`

Show `key`, `closed_as`, `closed_reason`, and final summary. Show code-change details only when relevant.

### `rename`

Show the old topic title and the proposed new title.

## Conditional Expansion

Automatically show additional fields only when:

- a path-key conflict or parent relationship is ambiguous
- a closed path would be reopened
- workspace code changes may affect a `clean` resume
- the user asks to see the complete draft

## Confirmation Safety

All write operations must end with an explicit question such as:

```text
Confirm writing the above changes?
```

No state is written until the user explicitly confirms. Read-only operations remain unchanged.

## Documentation

Update both README files to state that Trailmap presents concise confirmation drafts while retaining complete internal records. Do not duplicate the full per-command presentation rules in the README files.

## Validation

Forward-test these scenarios with fresh agents:

1. Create root paths for learning Linux drivers and learning AI. The draft contains only topic identity, concise path details, the default active choice, and the confirmation question.
2. Add a pending sibling without repeating the current path's full context.
3. Resume with `clean` while code changes exist and retain the contamination warning.
4. Verify every write operation ends with an explicit confirmation question.
5. Ask to expand details and verify the complete internal draft can be shown.

Also run static checks that the data model, storage paths, command semantics, confirmation requirement, and Git-safety rules remain unchanged.

## Non-Goals

This change does not alter the stored schema, path statuses, command syntax, active-path behavior, resume semantics, storage location, or Git behavior.
