# Trailmap Command Interface Design

## Goal

Make Trailmap's explicit invocation concise and unambiguous without renaming the product, skill, folder, or repository. Remove the redundant public command shape such as `/trailmap mark show` while preserving compatibility with existing usage.

## Naming Decision

- Product and display name: `Trailmap`
- Skill frontmatter name: `trailmap`
- Skill folder: `trailmap/`
- Repository: `trailmap-skill`
- Claude Code entry point: `/trailmap`
- Codex entry point: `$trailmap`

The alternatives `mark` and `marktrail` are rejected. `mark` is too generic and likely to collide with unrelated intent. `marktrail` is less natural and less memorable than the established `trailmap` name.

## Canonical Command Interface

Treat command words after the skill invocation as direct subcommands:

```text
<trailmap>
<trailmap> pending <idea>
<trailmap> list
<trailmap> show [key]
<trailmap> update <key>
<trailmap> resume <key> clean|informed
<trailmap> close <key> done|blocked|discarded
<trailmap> rename <title>
<trailmap> map [text]
```

`<trailmap>` means `/trailmap` in Claude Code and `$trailmap` in Codex. Invoking it without a subcommand creates a branch point, preserving the behavior of the former bare `mark` command.

## Compatibility

Keep existing usage functional through one normalization rule: ignore an optional leading `mark` before parsing the subcommand.

This allows legacy forms such as `mark show` and `/trailmap mark show`, but user documentation must not recommend them. Natural-language requests that match Trailmap's purpose remain supported.

Compatibility changes command parsing only. It does not change stored records or command behavior.

## Skill Content

Update `trailmap/SKILL.md` so canonical command headings and syntax use direct subcommands rather than repeating the `mark` prefix. State the optional-leading-`mark` compatibility rule once instead of documenting every legacy variation.

Keep the data model and core write rules in `SKILL.md`. Do not introduce a new `references/` split in this change. Reduce token usage conservatively by removing repeated prefixes and redundant examples while retaining examples that clarify:

- sibling creation with `pending`
- `clean` versus `informed` resume context
- path closure categories

Do not change the topic/path schema, statuses, confirmation requirements, storage paths, or Git safety behavior.

## Documentation And Metadata

Update both README files with one canonical form per platform:

- Claude Code examples use `/trailmap <subcommand>`.
- Codex examples use `$trailmap <subcommand>`.

Do not present compatibility syntax in the README files. Update `trailmap/agents/openai.yaml` so its default prompt uses the canonical direct-subcommand interface and does not teach `mark ...` as a second command layer.

Because the skill name and folder remain unchanged, existing installations and `.trailmap/marks/` data require no migration. After repository changes are verified, synchronize the existing local Codex installation.

## Validation

Verify all of the following:

1. `trailmap/SKILL.md` still declares `name: trailmap`.
2. README examples do not recommend `mark show` or `/trailmap mark show`.
3. Codex and Claude Code examples expose the same subcommand set.
4. `SKILL.md` contains the single optional-leading-`mark` compatibility rule.
5. No-argument invocation still creates a branch point.
6. The data model, state transitions, persistence paths, and confirmation rules remain unchanged.
7. The official skill validator passes. If its environment dependency is unavailable, run equivalent static checks and report that limitation.

## Scope

This change does not rename Trailmap, alter stored data, add commands, remove legacy behavior, publish a new package, or change Notion/Git integration behavior.
