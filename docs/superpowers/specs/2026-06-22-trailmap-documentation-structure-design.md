# Trailmap Documentation Structure Design

## Goal

Split Trailmap documentation into concise product introductions and complete usage guides in both English and Chinese. Product pages should explain why Trailmap matters and help a new user start quickly; usage guides should remain the authoritative human-facing reference for commands, state, storage, and safety behavior.

## File Structure

```text
README.md              English product introduction
README.zh-CN.md        Chinese product introduction
docs/USAGE.md          English detailed usage guide
docs/USAGE.zh-CN.md    Chinese detailed usage guide
```

The two product introductions share the same information architecture. The two usage guides also share the same information architecture and behavioral claims.

## Product Introduction

Each product README uses a product-oriented but technically credible structure:

1. product name and one-sentence positioning
2. the collaboration problem Trailmap solves
3. a 30-second workflow example
4. core capabilities
5. a compact state and tree-model overview
6. installation for Codex and Claude Code
7. the minimum useful command set
8. links to the detailed usage guides
9. data and Git safety boundaries
10. current limitations

The product README does not include the full JSON data model or exhaustive command explanations. It distinguishes path status from closure classification:

```text
path status: active / pending / paused / closed
closed_as:   done / blocked / discarded
```

## Detailed Usage Guides

Each usage guide covers:

1. Topic, Path, Parent, Active Path, and Created From
2. Codex and Claude Code invocation syntax
3. every supported subcommand with real examples
4. root paths, child paths, and sibling pending paths
5. the differences between `list`, `show`, and `map`
6. progress and code-change recording with `update`
7. context boundaries for `resume clean|informed`
8. closure behavior and closure classifications
9. concise confirmation drafts and explicit confirmation before persistence
10. the data model and `.trailmap/marks/` storage layout
11. Git safety boundaries
12. a recommended workflow, an end-to-end example, and current limitations

The current Chinese usage material is migrated and corrected rather than copied mechanically. The English guide receives equivalent coverage.

## Navigation

- `README.md` links to `README.zh-CN.md` and `docs/USAGE.md`.
- `README.zh-CN.md` links to `README.md` and `docs/USAGE.zh-CN.md`.
- Each usage guide links back to its corresponding product README and to the other language.
- Relative links must work from GitHub repository pages.

## Source Of Truth

`trailmap/SKILL.md` is the behavioral source of truth. Documentation must use only the supported invocation forms:

```text
/trailmap <subcommand>
$trailmap <subcommand>
```

The rewrite must not introduce unsupported commands or behavior such as `add`, `set`, `resume next`, automatic path switching, or automatic Git operations.

## Scope

This change restructures and rewrites documentation only. It does not modify the Skill behavior, command syntax, data model, storage format, or Git behavior.

## Validation

Verify that:

- all four files exist and cross-language links resolve
- product READMEs remain concise and point to their usage guides
- usage guides cover every command in `trailmap/SKILL.md`
- status and `closed_as` are described separately
- `.trailmap/marks/`, explicit confirmation, and Git safety rules remain accurate
- unsupported command examples are absent
- English and Chinese documents make equivalent behavioral claims
- Markdown formatting and internal relative links are valid
