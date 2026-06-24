# Changelog

## v0.1.0 - 2026-06-24

Initial stable release of Trailmap.

### Added

- Trailmap skill entrypoint for Codex (`$trailmap`) and Claude Code (`/trailmap`).
- Topic and path model for tracking active, pending, paused, and closed exploration paths.
- `pending` command for adding a sibling path without changing the active path.
- `resume` command with `clean` and `informed` context modes.
- `close` command with `done`, `blocked`, and `discarded` closure classifications.
- `map` output as Mermaid `graph LR`, plus plain text tree output with `map text`.
- Bilingual product documentation and detailed usage guides.

### Notes

- Trailmap stores state in workspace-local files under `.trailmap/marks/`.
- Write operations require explicit user confirmation before state is written.
- Trailmap records code-change reminders but does not automatically stash, revert, commit, switch branches, or isolate working-tree files.
