# Trailmap 当前完整 Skill 回归测试用例

本文档基于当前 `trailmap/SKILL.md` 的完整行为定义编写，用于验证 Trailmap 最新 Skill 的端到端行为。它不按历史版本拆分，而是覆盖当前 Skill 的全部公开命令、数据模型、安全边界、subagent、worktree 和 JSON 写入安全规则。

## 测试范围

- workspace 本地存储：`.trailmap/marks/` 与旧 `.codex/marks/` 兼容
- topic/path 扁平数据模型与 parent 引用
- concise confirmation draft 与显式确认规则
- 基础路径命令：无子命令、`pending`、`update`、`resume`、`close`、`rename`
- 只读命令：`list`、`show`、`map`、`map text`
- legacy `mark` 前缀兼容
- subagent shared workspace 执行状态
- subagent report 确认流
- worktree subagent 隔离执行
- retained worktree 与 resume warning
- JSON Write Safety 写入与回归校验
- Non-goals：不自动回滚、提交、合并、复制或推送 Notion

## 测试准备

### 环境

- 安装 Skill：当前仓库 `trailmap/`
- Codex 调用方式：`$trailmap`
- Claude Code 调用方式：`/trailmap`
- 本文示例统一使用 Codex 语法。

### 清理状态

在测试 workspace 中清空 Trailmap 状态：

```powershell
Remove-Item -Recurse -Force .trailmap -ErrorAction SilentlyContinue
Remove-Item -Recurse -Force .codex\marks -ErrorAction SilentlyContinue
Remove-Item -Recurse -Force .worktrees -ErrorAction SilentlyContinue
```

### 通用验收规则

- 所有写操作必须先展示草案，明确确认后才写入状态。
- 写操作包括：无子命令、`pending`、`update`、`subagent`、`resume`、`close`、`rename`。
- 只读操作包括：`list`、`show`、`map`；只读操作不要求确认，不修改状态。
- 每个 topic 最多只有一条主 `active` path。
- `topic.active` 必须为 `null`，或等于唯一 active path 的 key。
- closed path 必须有 `closed_as`、`closed_reason`、`closed_at`。
- 非 closed path 顶层不得保留关闭字段。
- `agent_run` 是执行状态，不替代 path 生命周期状态。
- `agent_run.worktree` 只允许出现在 `--worktree` run 中。
- Trailmap 不自动 merge、cherry-pick、apply patch、copy files、commit、stash、revert 或切换主 workspace branch。

## 数据模型与存储

### TC-001 新 workspace 默认写入 `.trailmap/marks/`

命令：

```text
$trailmap 登录失败可能来自 token 刷新或网络重试，先检查 token 刷新。
```

确认后期望：

- 创建 `.trailmap/marks/index.json`。
- 创建一个 topic JSON。
- `index.active_topic_id` 指向该 topic。
- topic 使用扁平 `paths` 列表。
- 不创建 decision nodes、path node IDs、`children`、`parent_id`、`topic.status`。

### TC-002 旧 `.codex/marks/` 兼容

前置条件：

- 不存在 `.trailmap/marks/`。
- 存在合法 `.codex/marks/`。

命令：

```text
$trailmap list
```

期望：

- 继续读取 `.codex/marks/`。
- 不自动迁移。
- 不创建额外副作用。

### TC-003 topic 不变量

校验：

- `topic.active` 是 `null` 或某个 path key。
- `topic.active != null` 时，恰好一个 path 的 `status = "active"` 且 key 匹配。
- 所有 path 都 closed 时，`topic.active = null`。
- topic open/closed 由 paths 推导，不存 `topic.status`。

### TC-004 path 不变量

校验：

- `key` 是 topic 内唯一 path 标识。
- 根路径 `parent = null`。
- child path 的 `parent` 引用同 topic 内存在的 path key。
- 不支持重命名 path key。
- `title`、`goal`、`hypothesis` 在存储中保持独立。
- 每个 path 保留 `created_from`。

## 确认草案

### TC-010 写操作必须先确认

命令：

```text
$trailmap pending 可能是服务端限流，先记录。
```

操作：

- 在草案出现后取消或拒绝。

期望：

- 不新增 path。
- 不写入 `index.json` 或 topic JSON。
- 草案末尾有一个明确确认问题。

### TC-011 concise confirmation 默认隐藏内部字段

命令：

```text
$trailmap pending 可能是系统时间偏差导致，先记录。
```

期望：

- 默认只展示 key、title、goal/hypothesis、状态和必要说明。
- 默认不展示 `source`、完整 `created_from`、`parent`、timestamp、完整持久化 JSON。
- 内部字段仍会在确认后正常写入。

### TC-012 需要展开详情的场景

场景：

- key 或 parent 关系有歧义。
- resume closed path。
- clean resume 可能被代码改动污染。
- 用户要求查看完整字段。

期望：

- 草案展开相关字段。
- 仍需明确确认后才写入。

## 基础路径命令

### TC-020 创建根路径

命令：

```text
$trailmap 登录失败可能来自 token 刷新或网络重试，先检查 token 刷新。
```

确认后期望：

```text
A  Token 刷新排查  [active]
B  网络重试排查    [pending]
```

校验点：

- A/B 是 root path，`parent = null`。
- A 是唯一 active path。
- B 是 pending。
- 草案展示 topic id/title、新 path key/status/title/goal/hypothesis、默认 active 选择说明。

### TC-021 legacy `mark` 前缀兼容

命令：

```text
$trailmap mark pending 可能是缓存写入顺序导致，先记下来。
```

期望：

- 忽略一个可选 leading `mark`。
- 按 `pending` 行为处理。
- 文档或输出不推荐 legacy 形式。

### TC-022 创建 child paths

前置条件：

- A 是当前 active path。

命令：

```text
$trailmap token 刷新内部可能是刷新竞态，也可能是缓存未更新，先查刷新竞态。
```

确认后期望：

```text
A   Token 刷新排查  [paused]
A1  刷新竞态        [active]
A2  缓存未更新      [pending]
```

校验点：

- A 追加 leave-summary update。
- A 变为 `paused`。
- A1/A2 的 `parent = "A"`。
- `topic.active = "A1"`。

### TC-023 显式 path key 查重

命令：

```text
$trailmap 新增 P1/P2 两个方向，先查 P1。
```

期望：

- 允许显式 key。
- 如果 key 已存在，必须拒绝或要求重新选择。
- 不覆盖已有 path。

### TC-024 `pending` 新增 sibling

命令：

```text
$trailmap pending 可能是系统时间偏差导致，先记下来，不切换。
```

期望：

- 新 path 与当前 active path 拥有相同 `parent`。
- 新 path `status = "pending"`。
- 当前 active path 不变。
- `topic.active` 不变。
- 不为当前 active path 追加 leave-summary。
- 不创建 child paths。

### TC-025 `pending` 自然语言等价形式

命令：

```text
$trailmap pending 检查网络重试，先不要切走。
```

期望：

- 按 `pending <idea>` 处理。
- 草案说明当前 active path 保持不变。

### TC-026 `pending` 缺少 active topic

前置条件：

- 没有 active topic 或没有 active path。

命令：

```text
$trailmap pending 可能是 DNS 问题。
```

期望：

- 拒绝写入或提示需要 active topic/path。
- 不创建无 parent 上下文的 pending sibling。

## 只读视图

### TC-030 `list`

命令：

```text
$trailmap list
```

期望：

- 跨 topic 展示 dashboard。
- 按 active、pending、paused、closed 分组。
- active/pending/paused 只显示 key、title 和必要 codechange warning。
- closed 显示 key、title、`closed_as`、`closed_reason`。
- 不展开完整 updates。
- 不修改状态。

### TC-031 `show`

命令：

```text
$trailmap show
```

期望：

- 展示 active topic、active path、active path 最近 updates。
- 展示 pending/paused paths。
- 展示 closed paths 的关闭原因。
- 不修改状态。

### TC-032 `show <key>`

命令：

```text
$trailmap show B
```

期望：

- 只在 active topic 内查找 B。
- 展示 `created_from`、path 描述、完整 updates、codechange、latest `agent_run`、handoff/report 摘要、closure fields。
- 对 worktree run 展开 worktree path、branch、base ref/sha、base dirty、changed files、diff summary。

### TC-033 不支持 `show <topic_id>`

命令：

```text
$trailmap show login-timeout
```

期望：

- 不把 topic id 当作支持的查看形式。
- 提示 active topic 内没有该 path key，或说明 v1 不支持 `show <topic_id>`。
- 不修改状态。

### TC-034 `map`

命令：

```text
$trailmap map
```

期望：

- 输出 Mermaid `graph LR`。
- 使用 `root` 作为 topic node。
- 每个 path 一个 node。
- 从 `paths[].parent` 推导树。
- key 不适合 Mermaid id 时替换为 `_`，label 保留原 key。
- label 展示 key、title、status；closed 展示 `closed: closed_as`。
- 有 `agent_run` 时追加紧凑执行状态。
- worktree run 追加 `worktree` 紧凑状态。
- 不展示完整 updates。

### TC-035 `map text`

命令：

```text
$trailmap map text
```

期望：

- 输出纯文本树。
- 字段与 `map` 一致。
- 不修改状态。

## update / resume / close / rename

### TC-040 `update <key>`

命令：

```text
$trailmap update A 已检查 token TTL，没有发现刷新失败，暂停这个方向，没有修改代码。
```

确认前期望：

- 草案包含 summary、conclusion、`status_after`、codechange。
- 若 `status_after = closed`，草案同时展示 `closed_as`。

确认后期望：

- A 追加 update。
- A 顶层 status 与 `status_after` 同步。
- 非 closed path 不保留顶层关闭字段。
- 使用 `codechange.changed/files/summary`，不使用 `codechange.note`。

### TC-041 update 自然语言暗示关闭但缺少分类

命令：

```text
$trailmap update A 这个方向确认无效，可以关闭。
```

期望：

- 不直接写入。
- 明确草拟解释，例如“Interpreting this update as closing path A as discarded.”
- 要求确认 closure class。

### TC-042 `resume <key> clean`

命令：

```text
$trailmap resume B clean
```

确认前期望：

- 为当前 active path 草拟 leave-summary update。
- 默认旧 path `status_after = paused`。
- 展示目标 B、模式 `clean`、状态变化和 codechange warning。

确认后期望：

- 旧 active path 变为 paused。
- B 变为 active。
- `topic.active = "B"`。
- clean context 不包含 sibling 详细推理。

### TC-043 `resume <key> informed`

命令：

```text
$trailmap resume A2 informed
```

期望：

- informed 包含 clean 全部内容。
- 额外包含 sibling、parent 或已探索路径摘要结论。
- 额外内容明确标记为 `other-path context`。

### TC-044 resume closed path

命令：

```text
$trailmap resume C clean
```

前置条件：

- C 是 closed path。

期望：

- 警告目标 path 已 closed。
- 明确要求 reopen 确认。
- 确认后 C 变为 active。
- 移除 C 顶层关闭字段。
- 关闭历史保留在 updates。

### TC-045 跨 topic resume

命令：

```text
$trailmap resume <topic_id> A clean
```

期望：

- 更新 `index.active_topic_id`。
- 目标 topic 的 A 变为 active。
- 原 active path 追加 leave-summary 并变为 paused。

### TC-046 resume running subagent path

前置条件：

- B 有 `agent_run.status = "running"`。

命令：

```text
$trailmap resume B clean
```

期望：

- 警告 B 正在被 subagent 探索，主会话 resume 可能重复工作或混合上下文。
- 不阻塞 resume。
- 不要求 `--force`。

### TC-047 resume retained worktree path

前置条件：

- B 有 `agent_run.worktree.status = "retained"`。
- B 的 worktree 有 changed files 或 diff summary。

命令：

```text
$trailmap resume B clean
```

期望：

- 警告 retained worktree 改动未合并到当前 workspace。
- 明确 `resume clean` 不应用这些改动。
- clean context 可包含 B 自己的 worktree artifact summary。
- 不包含未经确认的完整 handoff reasoning。

### TC-048 `close <key> done|blocked|discarded`

命令：

```text
$trailmap close B discarded
```

确认后期望：

- B 追加 closing update，`status_after = "closed"`。
- B 顶层 `status = "closed"`。
- B 顶层写入 `closed_as`、`closed_reason`、`closed_at`。
- 如果 B 是 active，`topic.active = null`。
- 不自动激活下一条 path。

### TC-049 close 缺少分类

命令：

```text
$trailmap close B
```

期望：

- 不写入。
- 要求选择 `done`、`blocked` 或 `discarded`。

### TC-050 `rename <title>`

命令：

```text
$trailmap rename 登录失败排查
```

期望：

- 草案展示旧 topic title 和新 title。
- 确认后只修改 active topic title。
- 不重命名 path key 或 path title。

## subagent shared workspace

### TC-060 为已有 pending path 启动 subagent

命令：

```text
$trailmap subagent B --allow-shared-code
```

确认前期望：

- `subagent` 被视为写操作。
- 草案包含 path key/title/status、context mode、计划 `agent_run.status`。
- `--allow-shared-code` 只跳过单独风险提示，不跳过写入确认。

确认后期望：

- B 生命周期状态仍是 pending。
- B 获得 `agent_run.status = "running"`。
- `agent_run.mode = "subagent"`。
- `agent_run.context_mode = "clean"`。
- `agent_run.risk = "shared_workspace_code"`。
- 当前主 active path 不变。

### TC-061 shared workspace 风险提示

命令：

```text
$trailmap subagent B
```

期望：

- 未使用 `--allow-shared-code` 时提示共享 workspace 代码风险。
- 风险提示说明 subagent 和 active path 可能修改同一批文件。
- 仍显示写入确认草案。

### TC-062 `--informed`

命令：

```text
$trailmap subagent B --informed --allow-shared-code
```

期望：

- `agent_run.context_mode = "informed"`。
- subagent context 包含 sibling、parent、current-active 或已探索路径摘要。
- 额外内容标记为 `other-path context`。

### TC-063 新建 root paths 时 `--subagent B,C`

命令：

```text
$trailmap 登录失败可能来自 token、网络或缓存，先查 token --subagent B,C --allow-shared-code
```

期望：

- A 是唯一 active path。
- B/C 是 pending path。
- B/C 均获得独立 `agent_run.status = "running"`。

### TC-064 child paths 创建时自动询问 subagent 候选

命令：

```text
$trailmap token 刷新可能是刷新竞态或缓存写入顺序，先查刷新竞态。
```

期望：

- 命令创建非主 active path 时，询问是否为候选 path 启动 subagent。
- 用户可选 0 条、1 条或多条。
- 选择的 path 获得 `agent_run.status = "running"`。
- 未选择 path 不包含 `agent_run`。

### TC-065 `pending --subagent`

命令：

```text
$trailmap pending 可能是服务端限流 --subagent --allow-shared-code
```

期望：

- 新 sibling path 仍是 pending。
- 当前 active path 不变。
- 新 path 获得 `agent_run.status = "running"`。

### TC-066 禁止对 active path 启动 subagent

命令：

```text
$trailmap subagent A --allow-shared-code
```

期望：

- 如果 A 是当前主 active path，拒绝启动。
- 不写入 `agent_run`。

### TC-067 禁止对 closed path 启动 subagent

命令：

```text
$trailmap subagent C --allow-shared-code
```

前置条件：

- C 是 closed path。

期望：

- 拒绝启动。
- C 状态与关闭字段不变。

### TC-068 禁止重复 running 或 reported run

命令：

```text
$trailmap subagent B --allow-shared-code
```

前置条件：

- B 的 `agent_run.status` 是 `running` 或 `reported`。

期望：

- 拒绝新 run。
- 如果状态是 `reported`，说明必须先接受、拒绝、完成、取消或解决上一份 report。
- 不覆盖现有 `agent_run`。

### TC-069 subagent 工具不可用

命令：

```text
$trailmap subagent B --allow-shared-code
```

前置条件：

- 当前 runtime 没有可用 subagent 工具。

确认后期望：

- 不让 path 操作整体失败。
- B 获得 `agent_run.status = "blocked"`。
- 写入简短 reason。
- B 生命周期状态保持原值。

## worktree subagent

### TC-080 默认不使用 worktree

命令：

```text
$trailmap subagent B --allow-shared-code
```

期望：

- 不出现 `agent_run.worktree`。
- 不创建 `.worktrees/`。
- 不创建 branch。
- shared workspace 是默认模式。

### TC-081 `--worktree` 启动草案

命令：

```text
$trailmap subagent B --worktree
```

确认前期望：

- 展示 path、context mode、base ref、base sha。
- 展示 branch to create。
- 展示 worktree path to create。
- 展示主 workspace 是否 dirty。
- 展示是否会更新 `.gitignore`。
- 明确未提交改动不会复制。
- 明确 Trailmap 不会 merge、commit、stash、revert、apply 或 copy code。
- 未确认前不创建 branch 或 worktree。

确认后期望：

- B 生命周期状态保持原值。
- `agent_run.status = "running"`。
- `agent_run.worktree.enabled = true`。
- `agent_run.worktree.status = "ready"`。
- 记录 `path`、`branch`、`base_ref`、`base_sha`、`base_dirty`。

### TC-082 `pending --subagent --worktree`

命令：

```text
$trailmap pending 网络重试可能是主因 --subagent --worktree
```

期望：

- 新 sibling path 仍是 pending。
- 当前 active path 不变。
- 草案包含完整 worktree startup draft。
- 确认后新 path 获得 `agent_run.worktree.status = "ready"`。

### TC-083 `--worktree --informed`

命令：

```text
$trailmap subagent B --worktree --informed
```

期望：

- 命令被接受。
- `agent_run.context_mode = "informed"`。
- 仍显示完整 worktree 草案。

### TC-084 `--worktree --base <ref>`

命令：

```text
$trailmap subagent B --worktree --base HEAD
```

期望：

- 解析 `HEAD` 为 `base_sha`。
- `agent_run.worktree.base_ref = "HEAD"`。
- `agent_run.worktree.base_sha` 是实际 commit sha。

### TC-085 base ref 不存在

命令：

```text
$trailmap subagent B --worktree --base not-a-real-ref
```

确认后期望：

- 不 fallback 到 `HEAD`。
- 不启动 subagent。
- `agent_run.status = "blocked"`。
- `agent_run.worktree.status = "failed"`。
- 提示修复后重试 `--worktree` 或显式改用 shared workspace。

### TC-086 `--worktree` 与 `--allow-shared-code` 互斥

命令：

```text
$trailmap subagent B --worktree --allow-shared-code
```

期望：

- 拒绝命令。
- 不创建 worktree。
- 不写入 `agent_run`。

### TC-087 `.worktrees/` 与 `.gitignore`

前置条件：

- `.gitignore` 不存在，或未包含 `.worktrees/`。

命令：

```text
$trailmap subagent B --worktree
```

期望：

- 草案展示将追加 `+ .worktrees/`。
- 确认后才创建或更新 `.gitignore`。
- 只追加 `.worktrees/`。
- 不重排、不重格式化已有 `.gitignore`。

### TC-088 dirty main workspace

前置条件：

- 主 workspace 有未提交改动。

命令：

```text
$trailmap subagent B --worktree
```

期望：

- 草案显示 dirty。
- `agent_run.worktree.base_dirty = true`。
- 不复制未提交改动到 worktree。
- subagent clean context 包含 dirty 且未复制警告。

### TC-089 worktree path/branch 冲突

前置条件：

- 默认 worktree 目录非空，或默认 branch 已存在。

命令：

```text
$trailmap subagent B --worktree
```

期望：

- 不复用非空目录。
- 不复用已有 branch。
- 为 path 和 branch 追加相同唯一后缀。

### TC-090 多 path worktree 部分失败

命令：

```text
$trailmap 登录失败可能来自 token、网络或缓存，先查 token --subagent B,C --worktree
```

前置条件：

- 测试环境模拟 C worktree 创建失败。

期望：

- B 成功时：`agent_run.status = "running"`，`worktree.status = "ready"`。
- C 失败时：`agent_run.status = "blocked"`，`worktree.status = "failed"`。
- C 不启动 subagent。
- 成功 path 不因单个失败回滚。

### TC-091 worktree 创建失败不降级

命令：

```text
$trailmap subagent B --worktree
```

前置条件：

- 模拟 branch、directory、git worktree 或 subagent startup 失败。

期望：

- 写入 `agent_run.status = "blocked"`。
- 写入 `agent_run.worktree.status = "failed"`。
- 不 fallback 到 shared workspace。
- 不启动 subagent。

### TC-092 不支持 `--no-worktree`

命令：

```text
$trailmap subagent B --no-worktree
```

期望：

- 不把 `--no-worktree` 当成有效控制参数。
- shared workspace 已是默认模式。
- 不写入意外字段。

## subagent report

### TC-100 shared report 转 update 草案

前置条件：

- B 有 `agent_run.status = "running"`。
- subagent 返回报告。

报告格式：

```text
path_key: B
summary: 检查了网络重试路径。
conclusion: 网络重试错误映射可能导致登录失败。
status_after: paused
codechange.changed: false
codechange.files: []
codechange.summary: 只读排查，没有修改代码。
handoff: 建议主会话继续确认重试错误码映射。
```

期望：

- `agent_run.status` 变为或展示为 `reported`。
- 生成普通 `update B` 草案。
- 未确认前不追加 update。
- 未确认前不改为 `completed`。

### TC-101 确认 report update

操作：

- 确认由 report 转换的 `update B` 草案。

期望：

- B 追加 update。
- B 顶层 status 同步为 `status_after`。
- `agent_run.status = "completed"`。
- 若 `status_after != closed`，不写顶层关闭字段。

### TC-102 拒绝 report update

操作：

- 拒绝或取消 report update 草案。

期望：

- 不追加 update。
- path 生命周期状态不变。
- `agent_run.status` 保持 `reported`。
- 不修改代码。

### TC-103 subagent 建议关闭 path

报告：

```text
path_key: B
summary: 已验证网络重试不是主因。
conclusion: 登录失败与网络重试无关。
status_after: closed
closed_as: discarded
codechange.changed: false
codechange.files: []
codechange.summary: 只读排查，没有修改代码。
handoff: 可回到 token 或缓存方向。
```

期望：

- subagent 不能直接关闭 path。
- Trailmap 生成 update 草案并要求确认。
- 确认后才写入 closed 状态和关闭字段。

### TC-104 worktree report 必填字段

前置条件：

- B 是 worktree subagent path。

报告必须额外包含：

```text
worktree.path
worktree.branch
worktree.base_ref
worktree.base_sha
worktree.changed_files
worktree.diff_summary
```

期望：

- 缺少字段时要求补齐或标记报告不完整。
- 完整报告后 `agent_run.worktree.status = "retained"`。
- worktree 不被自动删除。

### TC-105 worktree codechange 来源

前置条件：

- 主 workspace 和 worktree 都有改动。

期望：

- worktree run 的 `codechange.files` 来自 worktree 相对 `agent_run.worktree.base_sha` 的 diff。
- 不从主 workspace 推导。
- 结构化 worktree 细节保存在 `agent_run.worktree`。
- update 的 `codechange` 只保存 path-level 摘要。

### TC-106 worktree diff 不可读

期望：

```json
{
  "changed": true,
  "files": [],
  "summary": "Worktree diff unavailable; inspect worktree manually."
}
```

- 提示用户手动检查 retained worktree。

### TC-107 确认 worktree report update

期望：

- path 追加 update。
- `agent_run.status = "completed"`。
- `agent_run.worktree.status` 保持 `retained`。
- worktree 目录仍存在。
- 不自动 merge、copy、apply、commit 或删除 worktree。

## JSON Write Safety

### TC-120 写入必须结构化定位 path

适用于：

- `update <key>`
- `close <key>`
- subagent report 写入
- `resume`

期望：

- 解析 JSON 为对象。
- 通过精确 `paths[].key` 定位目标 path。
- 写入前快照所有 path status。
- 只修改目标 path 和明确需要的 topic 字段。
- 序列化后校验不变量。

### TC-121 禁止宽泛文本 patch

期望：

- 不用松散文本上下文 patch 重复字段，如 bare `"status"`、bare `"updates"`、bare `"closed_as"`。
- 如必须文本 patch，hunk 中必须包含目标 path 的唯一 `"key"` 和 `"title"`。

### TC-122 写后不变量校验

每次写后校验：

- `topic.active` 为 `null` 或恰好一个 active path 匹配该 key。
- intended path key 具有 intended status。
- 没有 unintended sibling 或 parent path 被改状态。
- closed path 有关闭字段，最后 update 的 `status_after = "closed"`。

### TC-123 status snapshot 比对

适用于：

- `update <key>`
- `close <key>`
- subagent report writes

期望：

- 比较写前/写后 status snapshot。
- 只有 intended path、resume 时 previous active path、明确新建 paths 可以变状态。
- 如有意外 path 状态变化，停止、诊断、修复 JSON、重新验证后再报告成功。

## Non-goals 与安全边界

### TC-130 不自动 Git 操作

期望：

- 除用户明确确认 `--worktree` 草案后创建隔离 branch/worktree 外，不自动创建 Git branch。
- 不自动 rollback、stash、revert、commit。
- 即使 worktree mode，也不自动 merge、cherry-pick、apply patch、copy files、commit、stash、revert 或切换主 workspace branch。

### TC-131 不直接推送 Notion

期望：

- Trailmap 自身不直接写 Notion。
- `map` 和 `map text` 输出适合复制到 Notion 或其他文档工具。

### TC-132 不记录每轮聊天

期望：

- 只记录决策点和 path-level updates。
- 不把每轮对话写入 Trailmap 状态。

## 最终验收清单

- `.trailmap/marks/` 默认存储正常，旧 `.codex/marks/` 兼容正常。
- 数据模型没有 decision nodes、children、topic.status 等禁用字段。
- 所有写操作都有确认草案，且未确认不写入。
- 只读操作不修改状态。
- topic/path 不变量始终成立。
- closed path 与非 closed path 的关闭字段处理正确。
- `pending` 只新增 sibling，不切换 active，不创建 child。
- no subcommand 在已有 active path 时创建 child paths。
- `resume clean` 与 `resume informed` 上下文边界清晰。
- `subagent` 是写操作。
- shared workspace risk 与 `--allow-shared-code` 行为正确。
- running/reported path 不允许启动新 subagent run。
- `agent_run` 与 path 生命周期状态区分清楚。
- `--worktree` 显式 opt-in，shared workspace 是默认模式。
- `--worktree` 与 `--allow-shared-code` 互斥。
- worktree 创建、失败、retained、report 和 resume warning 行为符合定义。
- worktree 元数据只在 `agent_run.worktree` 中存结构化字段。
- report 必须确认后才写入 update 或关闭 path。
- JSON 写入遵守结构化定位、snapshot 和写后不变量校验。
- Non-goals 没有被实现为自动副作用。
