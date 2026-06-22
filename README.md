# Trailmap Skill

Trailmap is an Agent Skill for tracking decision branches during an AI coding conversation. It is designed to work with skill-capable coding agents such as Codex and Claude Code.

It helps mark branching points, remember skipped paths, add pending paths without switching away from current work, resume a path with clean or informed context, record path progress, and export a Mermaid or text map.

## Install for Codex

In Codex, ask:

```text
Install the trailmap skill from https://github.com/tang0758/trailmap-skill/tree/main/trailmap
```

Or use the skill installer helper:

```powershell
python "$env:USERPROFILE\.codex\skills\.system\skill-installer\scripts\install-skill-from-github.py" --repo tang0758/trailmap-skill --path trailmap
```

Restart Codex after installing so the skill is discovered.

## Install for Claude Code

Personal skill, available across projects:

```bash
mkdir -p ~/.claude/skills
git clone https://github.com/tang0758/trailmap-skill.git /tmp/trailmap-skill
cp -R /tmp/trailmap-skill/trailmap ~/.claude/skills/trailmap
```

Project skill, available only in the current repository:

```bash
mkdir -p .claude/skills
git clone https://github.com/tang0758/trailmap-skill.git /tmp/trailmap-skill
cp -R /tmp/trailmap-skill/trailmap .claude/skills/trailmap
```

In Claude Code, invoke it with:

```text
/trailmap
```

Claude Code watches skill directories for changes, but restart the session if the top-level skills directory did not exist when the session started.

## Usage

Invoke the skill explicitly in Codex:

```text
$trailmap
```

Invoke the skill explicitly in Claude Code:

```text
/trailmap
```

Claude Code commands:

```text
/trailmap
/trailmap pending <idea>
/trailmap list
/trailmap show [key]
/trailmap update <key>
/trailmap resume <key> clean|informed
/trailmap resume <topic_id> <key> clean|informed
/trailmap close <key> done|blocked|discarded
/trailmap rename <topic title>
/trailmap map [text]
```

Codex commands:

```text
$trailmap
$trailmap pending <idea>
$trailmap list
$trailmap show [key]
$trailmap update <key>
$trailmap resume <key> clean|informed
$trailmap resume <topic_id> <key> clean|informed
$trailmap close <key> done|blocked|discarded
$trailmap rename <topic title>
$trailmap map [text]
```

## Confirmation drafts

Write operations show a concise confirmation draft with only decision-relevant fields. Trailmap still keeps the complete structured draft internally and writes it only after explicit confirmation. Ask to expand details when you need to inspect internal fields.

Model basics:

- A topic stores a flat `paths` list.
- Each path has a unique `key`, optional `parent`, `status`, `created_from`, `goal`, `hypothesis`, and `updates`.
- Path statuses are `active`, `pending`, `paused`, and `closed`.
- Closed paths use `closed_as`: `done`, `blocked`, or `discarded`.
- The base Trailmap command creates child paths under the current active path.
- The `pending` subcommand adds a sibling pending path without changing the active path.

Use the `pending` subcommand when you want to remember another possible path beside the current active path without switching away from the current work.

Trailmap stores workspace-local branch records under:

```text
.trailmap/marks/
```

If an older workspace already has `.codex/marks/`, Trailmap can continue using it for backward compatibility.

It does not automatically stash, revert, commit, or change git state.
