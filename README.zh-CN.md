# Trailmap 产品使用说明

Trailmap 是一个 Codex Skill，用来管理一次排查、设计或决策过程中的“分叉路径”。它解决的问题是：当 AI 给出多个方向，或者你自己尝试了多条路径后，不再依赖记忆去回想“还有哪些路没走”“A 路径试过什么”“现在要不要回到 B”。

## 适用场景

Trailmap 适合这些情况：

- 一个 bug 有多个可能原因，例如 A 是 token 问题，B 是网络重试问题。
- 你先沿着 A 深挖，后来发现跑偏，但不想忘记 B。
- 你手动尝试了 ABC 多种方案，之后需要回看每条路径做过什么。
- 做技术决策时，先选择 A，同时记录 B，后续可以回溯。
- 想把探索路径导出成 Mermaid 思维导图或文本树，方便放进 Notion、文档或 issue。

Trailmap 不适合当作完整聊天记录工具。它只记录关键分叉点和路径级进展。

## 核心概念

**主题 Topic**

一个可独立追踪的问题或工作线，例如“登录超时排查”。主题不等于 chat/conversation。一个 chat 可以有多个主题，一个主题也可以跨多个 chat 继续。

**分叉点 Decision**

问题出现多条可能路径的地方。例如：

```text
登录超时可能来自 token 过期，也可能来自网络重试。
```

**路径 Path**

某个具体探索方向。路径有用户可见的短标识，例如 `A`、`B`、`P1`、`P2`，用于跳转和回溯。

**活跃主题 Active Topic**

当前默认操作的主题。`mark update A`、`mark map` 等命令默认作用于活跃主题。

**活跃路径 Active Path**

当前正在探索的路径。使用 `mark resume` 回到另一条路径时，Trailmap 会切换活跃路径。

## 安装

在 Codex 中直接说：

```text
Install the trailmap skill from https://github.com/tang0758/trailmap-skill/tree/main/trailmap
```

也可以使用安装脚本：

```powershell
python "$env:USERPROFILE\.codex\skills\.system\skill-installer\scripts\install-skill-from-github.py" --repo tang0758/trailmap-skill --path trailmap
```

安装后重启 Codex，让 `$trailmap` 被识别。

## 启用方式

显式调用：

```text
$trailmap
```

之后可以使用自然语言或 `mark` 命令。Trailmap 的命令词是 `mark`，不是 `branch`。

## 常用命令

### 创建分叉点

```text
mark
```

或自然语言：

```text
mark 现在有 A/B 两条路，先走 A，B 先记录下来
```

如果当前没有活跃主题，Trailmap 会创建新主题和根分叉点。如果已有活跃主题，新分叉点默认挂在当前活跃路径下面。

创建前，AI 会先生成草案，包括：

- 主题名
- 分叉点
- 已确认上下文
- 仍是假设的内容
- 约束条件
- 候选路径
- 哪条路径先设为 active

只有你确认后，Trailmap 才会写入记录。

### 查看待处理路径

```text
mark list
```

显示当前 workspace 下所有 open 主题，以及 `active`、`pending`、`paused`、`blocked` 路径。`done` 和 `discarded` 默认隐藏。

适合回答：

```text
我现在还有哪些路径没走完？
哪些路径暂停了？
哪条路径有代码改动风险？
```

### 查看详情

```text
mark show
```

默认显示活跃主题摘要、当前活跃路径详情、最近更新和待回溯路径。

查看某条路径：

```text
mark show B
```

查看某个主题：

```text
mark show login-timeout
```

### 更新路径进展

```text
mark update A
```

Trailmap 会生成一条结构化更新草案：

```text
time
summary
conclusion
status_after
codechange.changed
codechange.files
codechange.note
```

其中 `codechange` 用来记录这条路径是否改过代码，以及 AI 总结的改动内容。确认后，这条更新会追加到路径记录里，并同步更新路径状态。

### 原地新增 pending 路径

```text
mark pending 可能是缓存写入顺序导致，先记下来
```

这个命令用于：你正在某条路径上工作，临时想到另一个可能性，只想把它作为待回溯路径记录下来，但不想切换当前工作路径。

Trailmap 会做：

- 在当前 active path 的同一父分叉点下新增一条 sibling path。
- 新路径状态设为 `pending`。
- 当前 active path 保持 `active`。
- `active_path_id` 完全不变。
- 不生成当前路径的离开摘要。
- 不创建新的分叉点，除非你明确要求。

也可以用自然语言：

```text
mark 先不要切走，旁边加一个 pending：检查网络重试
mark 继续当前 A，但把服务端限流作为待回溯路径记录
```

写入前，AI 会先展示草案，包括当前 active path、父分叉点、新 pending 路径的 key/title/goal/hypothesis，以及确认 `active_path_id` 不变。

### 回到另一条路径

```text
mark resume B clean
```

或：

```text
mark resume B informed
```

`resume` 会完成工作路径切换。切换前，Trailmap 会先总结当前 active path 的离开摘要，默认把当前路径标记为 `paused`，确认后再把目标路径设为 `active`。

跨主题回溯：

```text
mark resume login-timeout B clean
```

这会同时切换活跃主题和活跃路径。

### clean 和 informed 的区别

`clean` 是干净回溯模式，只带：

- 分叉点前共同上下文
- 目标路径自己的标题、目标、假设
- 目标路径自己的历史记录
- 约束条件
- 代码改动提醒

它不会带入兄弟路径的详细尝试过程，适合避免被 A 路径污染 B 路径判断。

`informed` 会在 `clean` 基础上，额外带入其他路径的摘要结论，并明确标注为“其他路径上下文”。适合你希望利用 A 的结论减少 B 的重复试错时使用。

### 关闭路径

```text
mark close A
```

关闭时需要选择或确认状态：

```text
done
blocked
discarded
```

Trailmap 会生成关闭摘要，确认后写入。

### 重命名主题

```text
mark rename 登录超时排查
```

第一版只支持重命名当前活跃主题，不重命名 path key 或路径标题。

### 生成思维导图

```text
mark map
```

默认输出 Mermaid `mindmap`，适合复制到支持 Mermaid 的文档或 Notion 场景。

输出纯文本树：

```text
mark map text
```

`mark list` 和 `mark map` 的区别：

- `mark list` 是行动视图，用来看“还有哪些没处理”。
- `mark map` 是结构视图，用来看“整个探索树长什么样”。

## 路径状态

Trailmap 使用这些状态：

```text
active      当前正在探索
pending     已记录，尚未开始
paused      探索过，但暂停
done        已完成或证实
blocked     暂时无法继续
discarded   跑偏、无效或不再考虑
```

`mark update` 的 `status_after` 会在确认落盘后同步更新路径状态。

## 数据保存位置

Trailmap 在当前 workspace 中保存记录：

```text
.codex/marks/
  index.json
  <topic-id>.json
```

`index.json` 记录当前活跃主题。每个 `<topic-id>.json` 记录一个主题的分叉树。

## 代码改动处理原则

Trailmap 只记录和提醒代码改动，不会自动执行这些操作：

- stash
- revert
- commit
- 切分支
- 回滚文件

如果某条路径记录了代码改动，`mark resume ... clean` 时会提醒你：clean 只隔离对话上下文，不隔离工作区代码。你需要自行决定是否保留、stash、commit 或 revert。

## 推荐工作流

1. 遇到多个方向时，用 `mark` 创建分叉点。
2. 先选一条路径作为 active，其他路径保持 pending。
3. 工作中临时想到另一个可能性，用 `mark pending <idea>` 原地记录，不切换 active path。
4. 每完成一次关键尝试，用 `mark update <path_key>` 记录阶段结果。
5. 需要回到另一条路径时，用 `mark resume <path_key> clean`。
6. 如果需要参考其他路径结论，用 `mark resume <path_key> informed`。
7. 路径结束后，用 `mark close <path_key>` 标记为 done、blocked 或 discarded。
8. 需要展示全局结构时，用 `mark map` 或 `mark map text`。

## 示例

创建分叉：

```text
$trailmap
mark 登录超时可能是 token 过期，也可能是网络重试导致。先走 token，网络重试先记录。
```

更新 A：

```text
mark update A 我检查了 token TTL 和刷新逻辑，没发现明显异常，没有改代码，A 暂停。
```

回到 B：

```text
mark resume B clean
```

生成图：

```text
mark map
```

## 当前限制

- 第一版不直接写入 Notion，只输出 Mermaid 或文本，方便手动复制。
- 第一版不自动管理 git 状态。
- 第一版不记录每轮聊天，只记录分叉点和路径级更新。
- 第一版使用树模型，不处理复杂图关系；预留 `related_node_ids` 以后扩展。
