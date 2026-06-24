# Trailmap

> 面向人机协同解决问题的有状态决策路径图。

[English](README.md) · [详细使用说明](docs/USAGE.zh-CN.md)

Trailmap 是面向 Codex、Claude Code 等 AI coding agent 的 Agent Skill。它把对话中出现的备选方案、已经探索的路径和暂时搁置的方向组织成一张可持续更新的路线图。

## Trailmap 解决什么问题

排查问题或做技术决策时，很少只有一条直线：

- AI 同时提出 A、B 两种可能性
- 你沿着 A 深入，又发现 A1 和 A2
- B 仍然可能成立，却逐渐从当前上下文中消失
- 做过几轮实验后，人和 AI 都说不清哪些方向试过、为什么停下

聊天记录保留了文字，却没有明确表达当前工作主线和分支状态。Trailmap 将决策结构单独记录下来，让正在探索的路径、待回溯方案和已经关闭的结论始终可见。

## 30 秒理解 Trailmap

先记录两个可能原因：

```text
$trailmap 登录失败可能来自 token 刷新或网络重试，先检查 token 刷新。
```

Trailmap 会先生成简洁确认草案：

```text
A  Token 刷新失败  [active]
B  网络重试失败    [pending]
```

探索 A 时临时想到另一个可能性，可以先记在旁边，不切换当前路径：

```text
$trailmap pending 可能是 token 缓存没有更新，先记录，保持当前路径。
```

记录 A 的结果，再以相对独立的上下文回到 B：

```text
$trailmap update A
$trailmap resume B clean
```

任何状态写入前，Trailmap 都会展示草案并等待明确确认。

## 核心能力

- **不丢备选方案。** 在选择主线前记录 A、B、C，后续随时查看。
- **不中断当前工作。** 在 active 路径旁边原地新增 sibling pending 路径，当前 active 完全不变。
- **形成决策树。** 当前方向再次分叉时，在它下面创建 child paths。
- **记录路径结果。** 保存摘要、结论、状态和代码改动提醒。
- **有边界地回溯。** 用 `clean` 限制其他路径影响，或用 `informed` 带入相关路径结论。
- **看清探索全貌。** 查看跨主题列表，或把当前主题输出为 Mermaid `graph LR` 和文本树。

## Trailmap 如何组织路径

Trailmap 在主题下面用树组织路径：

```text
主题：登录失败排查

A  Token 刷新                  [paused]
├─ A1  刷新竞态                [closed: discarded]
└─ A2  Refresh Token 已过期    [active]
B  网络重试                    [pending]
```

每个主题最多只有一条 active path。路径状态只有四种：

```text
active   当前正在探索
pending  已记录，尚未开始
paused   探索过，暂时放下
closed   已结束或不再继续
```

closed 路径还会记录关闭分类：

```text
done       已完成、已证实或已解决
blocked    当前条件下无法继续
discarded  已排除、无效或决定放弃
```

`done`、`blocked`、`discarded` 是 `closed_as` 的值，不是额外的路径状态。

## 安装

### Codex

直接告诉 Codex：

```text
Install the trailmap skill from https://github.com/tang0758/trailmap-skill/tree/main/trailmap
```

也可以使用 Codex Skill 安装脚本：

```powershell
python "$env:USERPROFILE\.codex\skills\.system\skill-installer\scripts\install-skill-from-github.py" --repo tang0758/trailmap-skill --path trailmap
```

安装后通过 `$trailmap` 调用。

### Claude Code

安装为个人 Skill：

```bash
mkdir -p ~/.claude/skills
git clone https://github.com/tang0758/trailmap-skill.git /tmp/trailmap-skill
cp -R /tmp/trailmap-skill/trailmap ~/.claude/skills/trailmap
```

如果只希望在当前项目使用，将仓库中的 `trailmap/` 目录复制到 `.claude/skills/trailmap`。安装后通过 `/trailmap` 调用。

如果没有立即识别 Skill，请重启对应 agent 会话。

## 最小命令集

Claude Code 使用 `/trailmap`，Codex 使用 `$trailmap`。以下示例采用 Codex 语法：

```text
$trailmap                              创建主题、根路径或 child paths
$trailmap pending <idea>               新增 sibling pending，保持 active 不变
$trailmap list                         查看 workspace 中所有主题和路径
$trailmap show [key]                   查看活跃主题或某条路径
$trailmap update <key>                 记录路径进展和结果
$trailmap resume <key> clean|informed  切换当前工作路径
$trailmap close <key> done|blocked|discarded
$trailmap rename <topic title>
$trailmap map [text]                   输出 Mermaid 或文本树
```

child path、跨主题回溯、确认草案和完整命令行为请查看[详细使用说明](docs/USAGE.zh-CN.md)。

## 安全边界

Trailmap 在 workspace 中保存记录：

```text
.trailmap/marks/
```

写操作会先展示简洁草案，并且只在明确确认后落盘。Trailmap 只记录代码改动提醒，不会自动：

- stash 或 revert 文件
- commit 代码
- 切换 Git 分支
- 隔离不同路径对应的工作区改动

`resume clean` 只控制对话上下文。工作区里已经存在的代码仍然可能影响回溯后的路径。

## 当前限制

- Trailmap 记录决策点和路径级进展，不保存每一轮聊天。
- 它通过 parent key 表达树，不处理通用图关系。
- 它可以输出 Mermaid 或文本，但不会直接写入 Notion。
- 它会提示 Git 风险，但不管理 Git 状态。

## 文档

- [详细使用说明](docs/USAGE.zh-CN.md)
- [English product overview](README.md)
- [Skill 指令](trailmap/SKILL.md)
