# Changelog

## lite-v0.1.0 - 2026-07-03

First release of the recording-only Trailmap Lite line.

### Added

- Explicit `$trailmap` and `/trailmap` invocation with implicit Codex activation disabled.
- Independent Topic files under `.trailmap/topics/` and visible Chat-to-Topic markers.
- Minimal Topic, Path, and Update records.
- `new`, `use`, `pending`, `list`, `show`, `update`, `resume`, `close`, `rename`, and `map` commands.
- Mermaid `graph LR` and plain-text tree output.
- Bilingual product and usage documentation.
- RED/GREEN regression scenarios focused on recording-only behavior.

### Removed

- Subagent execution and worktree orchestration.
- Clean/informed context loading and generated resume context.
- Automatic Git inspection and codechange records.
- Legacy command parsing, old storage compatibility, and migration behavior.
- Confirmation prompts for explicit and unambiguous write commands.

### Compatibility

Trailmap Lite intentionally starts a separate data model and release line. It does not read or migrate records created by the execution-oriented 0.x line.
