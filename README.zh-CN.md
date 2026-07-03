# Trailmap Lite

> 一个只负责记录路径的 Agent Skill：提醒你有哪些方向，但不接管路径上的实际工作。

[English](README.md) | [详细使用说明](docs/USAGE.zh-CN.md)

## 为什么需要 Trailmap

使用 AI 处理问题时，经常会同时出现多个可能方向，但一次只能深入一条。未选择的方案容易被聊天内容淹没，已经尝试过的路径也可能被遗忘，之后想回到旧方向时还要重新整理上下文。

Trailmap Lite 在工作区内保存一张简单路线图：

```text
A  检查 token 刷新       paused
B  检查网络重试          active
C  检查缓存写入顺序      pending
```

它只记录这张图，不负责排查、实现、读取业务代码、检查 Git、创建 Agent，也不会建议路径应该怎么做。Trailmap 命令结束后，普通 Agent 再继续处理实际问题。

## 核心模型

- **Topic**：一个问题或工作主题下的路径集合。
- **Path**：一个具体方向，使用 `A`、`B`、`A1` 等简短且不可变的 key 标识。
- 一个 Topic 最多只有一条 `active` 路径。
- 其他路径状态为 `pending`、`paused` 或 `closed`。
- closed 路径在人类视图中显示为 `done`、`blocked` 或 `discarded`。

每个 Topic 独立存储：

```text
.trailmap/topics/<topic-id>.json
```

工作区不保存全局活跃 Topic。每次 Trailmap 输出都以可见标记开头：

```text
Topic: login-failure | 登录失败排查
```

恢复原 Chat 后可从该标记恢复 Topic；新 Chat 可用 `use <topic-id>` 选择已有 Topic。

## 典型流程

```text
$trailmap new 登录失败排查 --id login-failure
$trailmap pending 检查 token 刷新
$trailmap pending 检查网络重试
$trailmap resume B
$trailmap update B 重试耗尽无法复现登录失败
$trailmap close B discarded 不是 401 的来源
$trailmap map
```

Codex 使用 `$trailmap`，Claude Code 使用 `/trailmap`，两者子命令完全一致。

## 命令

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

不带子命令调用 Trailmap，等同于 `list`。

## 安装

Lite 版本线位于 `trailmap-lite` 分支。克隆仓库后，只需将 `trailmap/` 目录安装为 Agent Skill。

### Codex

```powershell
git clone --branch trailmap-lite --depth 1 https://github.com/tang0758/trailmap-skill.git "$env:TEMP\trailmap-skill"
New-Item -ItemType Directory -Force "$env:USERPROFILE\.codex\skills" | Out-Null
Copy-Item -Recurse -Force "$env:TEMP\trailmap-skill\trailmap" "$env:USERPROFILE\.codex\skills\trailmap"
```

重启 Codex 后，通过 `$trailmap` 显式调用。

### Claude Code

```bash
git clone --branch trailmap-lite --depth 1 https://github.com/tang0758/trailmap-skill.git /tmp/trailmap-skill
mkdir -p ~/.claude/skills
cp -R /tmp/trailmap-skill/trailmap ~/.claude/skills/trailmap
```

如果只希望当前项目使用，可复制到 `.claude/skills/trailmap`。通过 `/trailmap` 调用。

发布 `lite-v0.1.0` 后，可将 `--branch trailmap-lite` 替换为 `--branch lite-v0.1.0`，固定安装该版本。

## 功能边界

Trailmap Lite 不支持路径执行、后台 Agent、代码隔离工作区、上下文加载模式、自动 Git 检查、旧数据迁移或删除命令。这些职责应由 Trailmap 之外的 Agent 或工具承担。

完整状态转换、Topic 恢复、父子路径、重开、输出约束和并发限制见 [USAGE.zh-CN.md](docs/USAGE.zh-CN.md)。

## License

仓库暂未声明许可证。对外分发修改版本前应先补充许可证。
