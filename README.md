# Trailmap Skill

Trailmap is a Codex skill for tracking decision branches during a conversation. It helps mark branching points, remember skipped paths, resume a path with clean or informed context, record path progress, and export a Mermaid or text map.

## Install

In Codex, ask:

```text
Install the trailmap skill from https://github.com/tang0758/trailmap-skill/tree/main/trailmap
```

Or use the skill installer helper:

```powershell
python "$env:USERPROFILE\.codex\skills\.system\skill-installer\scripts\install-skill-from-github.py" --repo tang0758/trailmap-skill --path trailmap
```

Restart Codex after installing so the skill is discovered.

## Usage

Invoke the skill explicitly:

```text
$trailmap
```

Core commands:

```text
mark
mark list
mark show
mark pending <idea>
mark update <path_key>
mark resume <path_key> clean
mark resume <path_key> informed
mark close <path_key>
mark rename <topic title>
mark map
mark map text
```

Use `mark pending <idea>` when you want to remember another possible path beside the current active path without switching away from the current work.

Trailmap stores workspace-local branch records under:

```text
.codex/marks/
```

It does not automatically stash, revert, commit, or change git state.
