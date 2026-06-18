# Trailmap 产品使用说明

Trailmap 是一个面向 AI coding agent 的 Agent Skill，用来管理一次排查、设计或决策过程中的“分叉路径”。它适合 Codex、Claude Code 等支持 `SKILL.md` 结构的 agent 使用。

它解决的问题是：当 AI 给出多个方向，或者你自己尝试了多条路径后，不再依赖记忆去回想“还有哪些路没走”“A 路径试过什么”“现在要不要回到 B”。Trailmap 会以 map 的方式记录你尝试过的路径、想要尝试的路径、路径结果等信息。

## 适用场景

- 一个 bug 有多个可能原因，例如 A 是 token 问题，B 是网络重试问题。
- 你先沿着 A 深挖，后来发现跑偏，但不想忘记 B。
- 你正在 A 路径上工作，临时想到另一个可能性，只想先挂成 pending。
- 你手动尝试了 ABC 多种方案，之后需要回看每条路径做过什么。
- 想把探索路径导出成 Mermaid 思维导图或文本树，方便放进 Notion、文档或 issue。

Trailmap 不适合当作完整聊天记录工具。它只记录关键分叉点和路径级进展。

## 核心概念

**主题 Topic**

一个可独立追踪的问题或工作线，例如“登录超时排查”。主题不等于 chat/conversation。一个 chat 可以有多个主题，一个主题也可以跨多个 chat 继续。

**路径 Path**

某个具体探索方向。每条路径只有一个唯一标识 `key`，例如 `A`、`B`、`A1`、`P2`。`key` 既是人看的短标识，也是内部引用标识。

**父路径 Parent**

路径可以挂在另一条路径下面形成树。根路径的 `parent` 是空；子路径的 `parent` 是父路径的 `key`。

**活跃路径 Active Path**

当前正在探索的路径。每个主题最多只有一条 active path。主题用 `active` 保存当前路径的 `key`。

**路径来源 Created From**

每条路径都会记录它创建时的来源上下文，包括为什么创建、当时已确认什么、有哪些假设、有哪些约束。这用于 `resume` 子命令的 `clean` 模式，避免把其他路径的结论污染进来。

## 安装

Codex 中直接说：

```text
Install the trailmap skill from https://github.com/tang0758/trailmap-skill/tree/main/trailmap
```

或使用安装脚本：

```powershell
python "$env:USERPROFILE\.codex\skills\.system\skill-installer\scripts\install-skill-from-github.py" --repo tang0758/trailmap-skill --path trailmap
```

Claude Code 个人 skill：

```bash
mkdir -p ~/.claude/skills
git clone https://github.com/tang0758/trailmap-skill.git /tmp/trailmap-skill
cp -R /tmp/trailmap-skill/trailmap ~/.claude/skills/trailmap
```

Claude Code 项目 skill：

```bash
mkdir -p .claude/skills
git clone https://github.com/tang0758/trailmap-skill.git /tmp/trailmap-skill
cp -R /tmp/trailmap-skill/trailmap .claude/skills/trailmap
```

安装后如未识别，重启对应 agent 会话。

## 启用方式

Codex：

```text
$trailmap
```

Claude Code：

```text
/trailmap
```

之后可以使用自然语言或直接子命令。Claude Code 使用 `/trailmap <subcommand>`，Codex 使用 `$trailmap <subcommand>`。

Claude Code 命令：

```text
/trailmap
/trailmap pending <idea>
/trailmap list
/trailmap show [key]
/trailmap update <key>
/trailmap resume <key> clean|informed
/trailmap close <key> done|blocked|discarded
/trailmap rename <topic title>
/trailmap map [text]
```

Codex 命令：

```text
$trailmap
$trailmap pending <idea>
$trailmap list
$trailmap show [key]
$trailmap update <key>
$trailmap resume <key> clean|informed
$trailmap close <key> done|blocked|discarded
$trailmap rename <topic title>
$trailmap map [text]
```

## 数据模型

Trailmap 使用简化模型：

```json
{
  "id": "login-timeout",
  "title": "登录超时排查",
  "active": "A",
  "source": "2026-06-17 当前会话：排查登录超时",
  "paths": [
    {
      "key": "A",
      "title": "检查 token 过期",
      "status": "active",
      "parent": null,
      "created_from": {
        "reason": "登录超时可能来自 token 或网络；A 是 token 方向。",
        "confirmed": [],
        "assumptions": [],
        "constraints": []
      },
      "goal": "验证是否由 token 过期或刷新失败导致",
      "hypothesis": "401 可能来自 token 刷新失败",
      "updates": []
    }
  ]
}
```

关键点：

- 不再有 `decision node`。
- 不再有 `node_id/path_key` 双标识，只保留 `key`。
- 不保存 `children`，通过 `parent` 反向推导树。
- 不保存 `topic.status`，主题是否 open 由路径状态推导。
- 人类展示时会把 `title/goal/hypothesis` 合并成一段路径说明。

## 路径状态

Trailmap 只有 4 个路径状态：

```text
active   当前正在探索
pending  已记录，尚未开始
paused   探索过，但暂时放下
closed   已关闭
```

关闭路径时再用 `closed_as` 表示关闭分类：

```text
done       已完成/证实/解决
blocked    暂时无法继续
discarded  跑偏、无效或不再考虑
```

如果路径重新打开，顶层的 `closed_as/closed_reason/closed_at` 会移除，关闭历史保留在 `updates` 里。

## 常用命令

### 创建分叉或子路径

```text
$trailmap
```

如果当前没有活跃主题，基础命令会创建新主题和根路径。根路径的 `parent` 是空。

如果当前已经有 active path，基础命令表示“当前路径下面出现了子分叉”：

- 原 active path 保留为一条路径，并变成 `paused`。
- 新建子路径，子路径的 `parent` 是原 active path 的 `key`。
- 选择一条子路径作为新的 `active`。
- 其他子路径是 `pending`。

示例：

```text
$trailmap A 里面现在可能是缓存问题，也可能是刷新逻辑问题，先查刷新逻辑
```

可能得到：

```text
A 检查 token 过期 [paused]
  A1 检查刷新逻辑 [active]
  A2 检查缓存问题 [pending]
```

### 原地新增 pending 路径

```text
$trailmap pending 可能是缓存写入顺序导致，先记下来
```

用于：你正在某条路径上工作，临时想到另一个可能性，只想把它作为待回溯路径记录下来，但不想切换当前工作路径。

Trailmap 会做：

- 新增一条当前 active path 的 sibling path。
- 新路径状态是 `pending`。
- 当前 active path 继续保持 `active`。
- `topic.active` 完全不变。
- 不生成当前路径的离开摘要。

也可以继续在直接命令后使用自然语言：

```text
$trailmap pending 检查网络重试，先不要切走
$trailmap pending 服务端限流，继续当前 A
```

### 查看总览

```text
$trailmap list
```

`list` 子命令是跨主题工作台总览。它显示每个主题下所有状态的路径：

```text
active
pending
paused
closed
```

closed 路径也会显示关闭详情，但只显示 `closed_as + closed_reason`，不展开完整 `updates`。

### 查看详情

```text
$trailmap show
```

显示 active topic 的详情，包括当前 active path、路径说明、最近更新、待回溯路径和 closed 路径。

查看某条路径：

```text
$trailmap show B
```

`show <key>` 子命令只在 active topic 内查找路径。第一版不支持用 topic ID 查询。

### 更新路径进展

```text
$trailmap update A
```

Trailmap 会生成结构化更新草案：

```text
time
summary
conclusion
status_after
closed_as, only when status_after=closed
codechange.changed
codechange.files
codechange.summary
```

确认后，这条更新会追加到路径的 `updates`，并同步更新路径状态。

### 回到另一条路径

```text
$trailmap resume B clean
$trailmap resume B informed
```

`resume` 会完成工作路径切换。切换前，Trailmap 会先总结当前 active path 的离开摘要，默认把当前路径标记为 `paused`，确认后再把目标路径设为 `active`。

跨主题回溯：

```text
$trailmap resume login-timeout B clean
```

这会同时切换活跃主题和活跃路径。

### clean 和 informed 的区别

`clean` 只带：

- 目标路径的 `created_from`
- 目标路径自己的标题、目标、假设
- 目标路径自己的历史更新
- 约束条件
- 代码改动提醒

它不会带入兄弟路径的详细尝试过程，适合避免被 A 路径污染 B 路径判断。

`informed` 会在 `clean` 基础上，额外带入兄弟路径、父路径或已探索路径的摘要结论，并明确标注为“其他路径上下文”。

### 关闭路径

```text
$trailmap close A done
$trailmap close B blocked
$trailmap close C discarded
```

关闭后路径状态是 `closed`，关闭分类写入 `closed_as`。如果关闭的是当前 active path，`topic.active` 会变成空；之后用 `resume` 子命令激活另一条路径。

### 重命名主题

```text
$trailmap rename 登录超时排查
```

第一版只支持重命名当前活跃主题，不重命名 path key 或路径标题。

### 生成路径图

```text
$trailmap map
```

默认输出 Mermaid `mindmap`。它只基于 `paths[].parent` 生成路径树，展示 `key/title/status/closed_as`，不展示完整 updates。

输出纯文本树：

```text
$trailmap map text
```

`list` 和 `map` 子命令的区别：

- `list` 子命令是跨主题工作台，用来看所有主题和路径状态。
- `map` 子命令是当前主题结构图，用来看路径树长什么样。

## 数据保存位置

Trailmap 在当前 workspace 中保存记录：

```text
.trailmap/marks/
  index.json
  <topic-id>.json
```

如果旧项目里已经存在 `.codex/marks/`，Trailmap 可以继续使用旧目录以保持兼容；新项目默认使用 `.trailmap/marks/`。

## 代码改动处理原则

Trailmap 只记录和提醒代码改动，不会自动执行这些操作：

- stash
- revert
- commit
- 切分支
- 回滚文件

如果某条路径记录了代码改动，使用 `resume` 子命令的 `clean` 模式时会提醒你：clean 只隔离对话上下文，不隔离工作区代码。

## 推荐工作流

1. 遇到多个根方向时，用基础 Trailmap 命令创建主题和根路径。
2. 当前路径内部又分出子方向时，用基础 Trailmap 命令创建 child paths。
3. 工作中临时想到旁边的可能性，用 `pending` 子命令记录 sibling pending path。
4. 每完成一次关键尝试，用 `update` 子命令记录阶段结果。
5. 需要回到另一条路径时，用 `resume` 子命令的 `clean` 模式。
6. 如果需要参考其他路径结论，用 `resume` 子命令的 `informed` 模式。
7. 路径结束后，用 `close` 子命令和 `done|blocked|discarded` 分类关闭。
8. 需要展示全局结构时，用 `map` 子命令输出 Mermaid 或文本。

## 示例

创建根分叉：

```text
$trailmap 登录超时可能是 token 过期，也可能是网络重试导致。先走 token，网络重试先记录。
```

原地挂一个 pending：

```text
$trailmap pending 可能是服务端限流导致，先记下来，不切换当前路径
```

更新 A：

```text
$trailmap update A 我检查了 token TTL 和刷新逻辑，没发现明显异常，没有改代码，A 暂停。
```

回到 B：

```text
$trailmap resume B clean
```

生成图：

```text
$trailmap map
```

## 当前限制

- 第一版不直接写入 Notion，只输出 Mermaid 或文本，方便手动复制。
- 第一版不自动管理 git 状态。
- 第一版不记录每轮聊天，只记录分叉点和路径级更新。
- 第一版使用 `paths[].parent` 表达树，不处理复杂图关系。
