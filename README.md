# Trailmap Lite

> A recording-only Agent Skill for remembering alternative paths without letting the skill take over the work.

[中文](README.zh-CN.md) | [Detailed usage](docs/USAGE.md)

## Why Trailmap

AI-assisted work often exposes several plausible directions, but only one can be explored at a time. Alternatives get buried in the conversation, attempted paths are forgotten, and returning to an earlier idea requires reconstructing what happened.

Trailmap Lite keeps a small workspace-local map:

```text
A  Check token refresh       paused
B  Check network retry       active
C  Check cache write order   pending
```

It records the map. It does not investigate, implement, inspect code, run Git commands, create agents, or advise how to solve a path. After a Trailmap command finishes, the normal coding agent continues the actual work.

## Core Model

- A **Topic** groups paths for one problem or work line.
- A **Path** is one direction identified by a short immutable key such as `A`, `B`, or `A1`.
- At most one path in a Topic is `active`.
- Other paths are `pending`, `paused`, or `closed`.
- Closed paths are displayed as `done`, `blocked`, or `discarded`.

Each Topic is stored independently:

```text
.trailmap/topics/<topic-id>.json
```

There is no workspace-global active Topic. Every Trailmap response starts with a visible marker:

```text
Topic: login-failure | Login failure investigation
```

That marker lets a resumed chat recover its Topic. A new chat can select an existing Topic with `use <topic-id>`.

## Typical Flow

```text
$trailmap new Login failure investigation --id login-failure
$trailmap pending Check token refresh
$trailmap pending Check network retry
$trailmap resume B
$trailmap update B Retry exhaustion does not reproduce the failure
$trailmap close B discarded Not the source of the 401 response
$trailmap map
```

Codex uses `$trailmap`; Claude Code uses `/trailmap`. The subcommands are the same.

## Commands

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

Calling Trailmap without a subcommand is equivalent to `list`.

## Install

The Lite release line lives on the `trailmap-lite` branch. Clone the repository and install only the `trailmap/` directory as an Agent Skill.

### Codex

```powershell
git clone --branch trailmap-lite --depth 1 https://github.com/tang0758/trailmap-skill.git "$env:TEMP\trailmap-skill"
New-Item -ItemType Directory -Force "$env:USERPROFILE\.codex\skills" | Out-Null
Copy-Item -Recurse -Force "$env:TEMP\trailmap-skill\trailmap" "$env:USERPROFILE\.codex\skills\trailmap"
```

Restart Codex, then invoke `$trailmap` explicitly.

### Claude Code

```bash
git clone --branch trailmap-lite --depth 1 https://github.com/tang0758/trailmap-skill.git /tmp/trailmap-skill
mkdir -p ~/.claude/skills
cp -R /tmp/trailmap-skill/trailmap ~/.claude/skills/trailmap
```

For project-only installation, copy `trailmap/` to `.claude/skills/trailmap`. Invoke it with `/trailmap`.

After `lite-v0.1.0` is published, replace `--branch trailmap-lite` with `--branch lite-v0.1.0` to pin that release.

## Scope

Trailmap Lite intentionally does not support path execution, background agents, isolated code workspaces, context-loading modes, automatic Git inspection, legacy data migration, or deletion commands. Those concerns belong outside this recording skill.

See [USAGE.md](docs/USAGE.md) for exact state transitions, Topic recovery, tree placement, reopening, output rules, and concurrency limitations.

## License

No license has been declared yet. Add a license before redistributing modified copies.
