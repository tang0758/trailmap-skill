# Trailmap

> A stateful decision-path map for human-AI problem solving.

[中文](README.zh-CN.md) · [Detailed usage](docs/USAGE.md)

Trailmap is an Agent Skill for Codex, Claude Code, and other skill-capable coding agents. It keeps alternative ideas, explored paths, and deferred work visible while you and an AI agent solve a problem over multiple turns.

## The problem

A debugging or design conversation rarely follows one straight line:

- the AI suggests causes A and B
- you investigate A and discover A1 and A2
- B remains plausible but disappears from working memory
- after several experiments, nobody remembers what was tried or why it stopped

Chat history contains the words, but it does not provide an explicit execution map. Trailmap records the decision structure separately so that the current path, pending alternatives, and closed conclusions remain traceable.

## A 30-second example

Start with two possible causes:

```text
$trailmap Login failures may come from token refresh or network retries. Start with token refresh.
```

Trailmap proposes a concise confirmation draft:

```text
A  Token refresh failure   [active]
B  Network retry failure  [pending]
```

While investigating A, remember another possibility without switching away:

```text
$trailmap pending The token cache may be stale; keep the current path active.
```

Record what happened, then return to B with an isolated decision context:

```text
$trailmap update A
$trailmap resume B clean
```

Nothing is written until you explicitly confirm the proposed change.

## Core capabilities

- **Keep alternatives visible.** Record A, B, and C before following one of them.
- **Add ideas without losing focus.** Attach a sibling `pending` path while the current `active` path remains unchanged.
- **Build a tree of decisions.** Create child paths when the current direction splits again.
- **Explore alternatives in parallel.** Start subagent exploration for pending paths while the main active path remains unchanged.
- **Isolate risky subagent work.** Use `--worktree` to run a subagent path in a separate Git worktree.
- **Record outcomes.** Save summaries, conclusions, status, and code-change reminders for each path.
- **Resume deliberately.** Choose `clean` context to limit cross-path influence or `informed` context to include conclusions from related paths.
- **See the whole trail.** List the workspace or export the active topic as Mermaid `graph LR` or a text tree.

## How the model works

Trailmap organizes work as a tree of paths under a topic:

```text
Topic: Login failure investigation

A  Token refresh                 [paused]
├─ A1  Refresh race condition    [closed: discarded]
└─ A2  Expired refresh token     [active]
B  Network retry                 [pending]
```

Every topic has at most one active path. A path has one of four statuses:

```text
active   currently being explored
pending  recorded but not started
paused   explored and temporarily set aside
closed   finished or no longer being pursued
```

A closed path also has a closure classification:

```text
done       completed, confirmed, or resolved
blocked    cannot continue for now
discarded  disproved, unhelpful, or intentionally abandoned
```

`done`, `blocked`, and `discarded` are `closed_as` values, not additional path statuses.

## Install

Current stable release: `v0.1.0`.

### Codex

Ask Codex:

```text
Install the trailmap skill from https://github.com/tang0758/trailmap-skill/tree/main/trailmap
```

Or use the Codex skill installer:

```powershell
python "$env:USERPROFILE\.codex\skills\.system\skill-installer\scripts\install-skill-from-github.py" --repo tang0758/trailmap-skill --path trailmap
```

Invoke it with `$trailmap`.

To install a fixed release manually for Codex:

```powershell
New-Item -ItemType Directory -Force "$env:USERPROFILE\.codex\skills" | Out-Null
git clone --branch v0.1.0 --depth 1 https://github.com/tang0758/trailmap-skill.git "$env:TEMP\trailmap-skill"
Copy-Item -Recurse -Force "$env:TEMP\trailmap-skill\trailmap" "$env:USERPROFILE\.codex\skills\trailmap"
```

### Claude Code

Install as a personal skill:

```bash
mkdir -p ~/.claude/skills
git clone https://github.com/tang0758/trailmap-skill.git /tmp/trailmap-skill
cp -R /tmp/trailmap-skill/trailmap ~/.claude/skills/trailmap
```

For a fixed release, add `--branch v0.1.0 --depth 1` to the `git clone` command.

For a project-only installation, copy the same `trailmap/` directory to `.claude/skills/trailmap`. Invoke it with `/trailmap`.

Restart the agent session if the skill is not discovered immediately.

## Essential commands

Claude Code uses `/trailmap`; Codex uses `$trailmap`. The examples below use Codex syntax:

```text
$trailmap                              Create a topic, root paths, or child paths
$trailmap pending <idea>               Add a sibling pending path; keep active unchanged
$trailmap list                         List topics and paths across the workspace
$trailmap show [key]                   Show the active topic or one path
$trailmap update <key>                 Record progress and outcome
$trailmap subagent <key>               Start subagent exploration for an existing path
$trailmap subagent <key> --worktree    Start subagent exploration in an isolated worktree
$trailmap subagent <key> --worktree --base <ref>
$trailmap ... --subagent B --allow-shared-code
$trailmap resume <key> clean|informed  Switch the active work path
$trailmap close <key> done|blocked|discarded
$trailmap rename <topic title>
$trailmap map [text]                   Output Mermaid or a text tree
```

See the [detailed usage guide](docs/USAGE.md) for child paths, cross-topic resume, confirmation drafts, and complete command behavior.

## Safety boundaries

Trailmap stores workspace-local records under:

```text
.trailmap/marks/
```

Write operations first show a concise draft and require explicit confirmation. Trailmap records code-change information as reminders, but it does not automatically:

- stash or revert files
- commit changes
- switch Git branches
- isolate working-tree changes between paths

`resume clean` controls conversational context only. Existing workspace code can still affect the resumed path.

Subagent exploration may run in the same workspace as the main active path. Trailmap warns about shared-code risk, but it does not isolate files unless worktree mode is explicitly confirmed.

Worktree mode creates a local branch and worktree after confirmation. Trailmap records the path and branch but does not merge, commit, clean up, or apply worktree changes automatically. Retained worktree changes stay in that worktree until you inspect or integrate them yourself.

## Current limitations

- Trailmap records decision points and path-level updates, not every chat message.
- It models a tree through parent keys, not a general graph.
- It exports Mermaid or text but does not write directly to Notion.
- It reports Git-related risk. Except for confirmed worktree creation, it does not manage Git integration.

## Documentation

- [Detailed usage](docs/USAGE.md)
- [中文产品介绍](README.zh-CN.md)
- [Skill instructions](trailmap/SKILL.md)
