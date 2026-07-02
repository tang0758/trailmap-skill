# Trailmap Lite 回归测试契约

本文档是 Trailmap Lite 的 RED/GREEN 场景契约。它先固定旧 Skill 的已知失败，再定义后续改写必须满足的行为。Trailmap Lite 只记录、切换和展示路径；除非用户显式调用 `$trailmap` 或 `/trailmap`，否则不得参与对话。

本文示例使用 Codex 的 `$trailmap` 语法。Claude Code 将前缀替换为 `/trailmap`，其余行为相同。

## RED 基线

以下结果来自改写前的 `trailmap/SKILL.md`。重跑时应保留完整提示词，并判断回复是否越过“只记录”边界。

### RED-001 pending 后继续排障

提示词：

```text
We are urgently debugging a production login failure on path A. Record a pending sibling path B: check network retry. Keep A active. We have only 10 minutes, so make the record useful and then help me continue.
```

观察到的回复：

```text
Path B recorded as pending: check network retry. Path A remains active. For A, send the failing login request's status code, response body, and relevant server log or stack trace.
```

失败：虽然记录了 B 并保留 A，但随后索要请求和日志，继续参与问题排查。

### RED-002 resume 隐式重开并加入执行

提示词：

```text
Path A token refresh investigation is currently active but stalled. Resume closed path B because new evidence suggests cache ordering; switch to B and continue working immediately. The incident is ongoing and the user does not want extra questions.
```

观察到的回复：

```text
Switching from stalled Path A to Path B. I'll resume the cache-ordering investigation immediately and leave Path A pending.
```

失败：

- 原 active 路径 A 被改为 `pending`，而不是 `paused`。
- 已关闭的 B 被隐式重开，没有要求显式 `--reopen`。
- Agent 宣布立即继续调查，加入了路径执行。

### RED-003 update 缺少 Topic 与持久化契约

提示词：

```text
Update path A: token expiry has been ruled out. We changed auth.ts while testing. Record this quickly; the deadline is in five minutes.
```

观察到的回复：

```text
Path A update noted: token expiry ruled out; auth.ts changed during testing.
```

失败：回复没有 `Topic: <id> | <title>` 标记，也没有可验证的 Topic 持久化行为。

### RED 静态证据

运行：

```powershell
rg -n "agent_run|worktree|codechange|clean|informed|active_topic_id|\.trailmap/marks" trailmap/SKILL.md
```

#### 实际结果

- 命令退出码为 `0`，共输出 `99` 个匹配行。
- 代表性证据包括：第 8、15 行的 `.trailmap/marks` 旧存储，第 24、454 行的 `active_topic_id` 全局选择状态，以及第 122、152、186 行的 `agent_run`、worktree 和 `codechange` 执行跟踪。
- 第 3、125、138 行还命中 `clean`/`informed` 上下文模式；这些结果足以证明旧 Skill 保留了 Lite 必须移除的能力。

RED 期望：命令找到匹配项。任一匹配都证明旧 Skill 仍包含 Lite 不接受的执行编排、上下文模式、代码改动跟踪或全局选择状态。

## GREEN 通用验收规则

### 调用边界

- 只有显式 `$trailmap ...` 或 `/trailmap ...` 调用才触发 Trailmap。
- 普通对话即使提到“路径”“备选方向”“继续 A”或与下列基线提示完全相同，也不得自动读写 Trailmap、输出 Topic 标记或提醒用户使用 Trailmap。
- Trailmap 回复不得排障、实现、检查代码、检查 Git、索要日志、建议下一步工作或宣布将继续执行某条路径。
- Trailmap 不启动 subagent，不创建 worktree，不修改业务文件，也不运行路径中描述的任务。

### Topic 选择与存储

- 每个已选 Topic 的回复第一行始终是 `Topic: <id> | <title>`。
- 同一聊天恢复后，从聊天中最新一条合法 Topic 标记恢复选择；不依赖进程内记忆。
- 新聊天没有 Topic 标记时，用户使用 `use <topic-id>` 选择已有 Topic。
- 若 workspace 只有一个 Topic，可自动选择；有多个 Topic 且无标记时，只列出 id/title 并要求 `use <topic-id>`；没有 Topic 时，只要求 `new <topic-title>`。
- 每个 Topic 独立存放在 `.trailmap/topics/<topic-id>.json`。不同 Topic 必须写入不同文件。
- 不创建 `index.json`，不存储 `active_topic_id`，也不通过其他全局文件记录当前选择。
- 读写某个 Topic 不得改变其他 Topic 文件。

### 路径与写入

- 一个 Topic 最多一条 `active` 路径；Topic 的 `active` 字段必须与该路径 key 一致，无 active 路径时为 `null`。
- 路径 key 在 Topic 内唯一且不可变。状态只允许 `active`、`pending`、`paused`、`closed`。
- `pending` 标题逐字保存用户提供给 `<title>` 的文本，不改写、不翻译、不补充推断。
- `update` 只记录用户提供的 note；不得从代码、Git、对话上下文或路径标题生成额外结论。
- 显式且无歧义的命令立即写入。只有 AI 代拟 update 内容、输入有歧义或输入无效时才要求确认。
- 每次写入前重读目标 Topic；若读取后发现并发变化，拒绝覆盖并要求用户重试。
- 不提供删除命令。`close` 只改变状态并保留路径及历史；任何命令都不得删除 Topic、路径或 update。

### 输出约束

- 输出使用完成当前命令所需的最少信息，不复述整份 Topic JSON，不解释内部实现。
- 成功的 `pending`、`update`、`resume`、`close`、`rename` 通常只输出 Topic 标记、单行结果和必要警告。
- 不附带调试建议、泛化的“下一步”、使用教程或确认问题；只有歧义、无效输入、并发冲突和 AI 代拟 update 例外。
- `list`、`show` 和 `map` 可以按数据量展开，但不得混入执行建议。

## GREEN 核心基线

### GREEN-001 pending 只记录

前置状态：Topic `login-failure | Production login failure` 中 A 为 `active`。

调用：

```text
$trailmap pending check network retry --key B
```

期望：

- B.title 精确等于 `check network retry`，B.status 为 `pending`。
- B 与 A 为 sibling，二者 `parent` 相同。
- A 仍为 `active`，Topic.active 仍为 `A`。
- 回复第一行为 `Topic: login-failure | Production login failure`。
- 回复不请求状态码、响应体、日志或 stack trace，不提供任何排障建议。

### GREEN-002 resume 正常切换

前置状态：A 为 `active`，B 为 `pending` 或 `paused`。

调用：

```text
$trailmap resume B
```

期望：A 变为 `paused`，B 变为 `active`，Topic.active 变为 `B`。除非调用包含 `--note`，不得为 A 或 B 自动生成总结。回复不宣布继续执行 B。

### GREEN-003 resume closed 只给显式命令

前置状态：A 为 `active`，B 为 `closed`，B.closed_as 为 `discarded`。

调用：

```text
$trailmap resume B
```

期望：不写文件，A 与 B 状态不变。回复在 Topic 标记后只说明 B 已关闭，并打印：

```text
resume B --reopen --note "<reason>"
```

不得推断 reason、隐式重开 B、把 A 改为 pending，或加入 B 的执行。

### GREEN-004 update 只记录 supplied note

前置状态：A 存在。

调用：

```text
$trailmap update A token expiry has been ruled out. We changed auth.ts while testing.
```

期望：

- A.updates 新增一条 note，内容精确等于用户提供的整段文本。
- 不检查 `auth.ts`，不运行 Git，不生成 `codechange`、文件列表、结论或额外摘要。
- 路径状态不变。
- 回复第一行为 `Topic: <id> | <title>`，并简短确认记录成功。

### GREEN-005 未调用时完全不参与

将 RED-001、RED-002、RED-003 的提示词作为普通对话发送，不带 `$trailmap` 或 `/trailmap`。

期望：Trailmap 不读取或写入任何 Topic 文件，不输出 Topic 标记，不把普通请求改写为 Trailmap 命令，也不注入路径状态说明。

## GREEN 命令变化场景

### TC-001 `new` 创建独立 Topic

调用：

```text
$trailmap new Production login failure --id login-failure
```

期望：创建 `.trailmap/topics/login-failure.json`，id/title 分别为 `login-failure` 和 `Production login failure`；回复以对应 Topic 标记开头。不创建 `index.json`。省略 `--id` 时生成稳定、可用且不覆盖现有文件的 id；id 冲突时拒绝覆盖。

### TC-002 `use` 只选择 Topic

前置状态：`login-failure.json` 和 `billing-timeout.json` 均存在。

调用：

```text
$trailmap use billing-timeout
```

期望：回复以 `Topic: billing-timeout | <title>` 开头，不修改两个 Topic 文件，不写全局选择状态。后续调用依据聊天中的最新标记继续使用 `billing-timeout`。

### TC-003 恢复聊天与新聊天

场景 A：恢复的聊天中先后出现 login-failure 和 billing-timeout 标记。期望选择最后出现的 billing-timeout。

场景 B：新聊天没有标记且存在多个 Topic。期望只列出可选 Topic，并要求显式调用 `$trailmap use <topic-id>`，不猜测最近 Topic。

场景 C：新聊天没有标记且仅有一个 Topic。期望可自动选择该 Topic，并在回复首行打印标记。

场景 D：新聊天没有标记且 workspace 中没有 Topic。期望只要求显式调用 `$trailmap new <topic-title>`，不创建空 Topic，不输出伪造的 Topic 标记。

### TC-004 `pending` 默认创建 sibling

前置状态：A1 为 active 且 `parent = "A"`。

调用：

```text
$trailmap pending check clock skew --key A2 --note low confidence
```

期望：A2.parent 为 `A`，A2.status 为 `pending`，title 精确等于 `check clock skew`，note 精确等于 `low confidence`；A1 和 Topic.active 不变。

### TC-005 `pending --child` 与 `--parent`

前置状态：当前 active 路径为 A；另有路径 B 可供第二次调用引用。

调用一：

```text
$trailmap pending inspect cache ordering --key A1 --child
```

期望：新路径 A1.parent 等于当前 active 路径 key。

调用二：

```text
$trailmap pending compare edge retries --key B1 --parent B
```

期望：B 必须存在且未造成 key 冲突；B1.parent 为 `B`。`--child` 与 `--parent` 同时出现时输入无效，不写入。

### TC-006 无参数与 `list`

调用：

```text
$trailmap
$trailmap list
```

期望：两者等价，只列当前 Topic 的路径，显示 key、逐字标题和状态；closed 路径显示 `done`、`blocked` 或 `discarded` 分类。不修改文件，不给执行建议。

### TC-007 `list all`

调用：

```text
$trailmap list all
```

期望：按 Topic 分组展示所有 Topic 的 id/title 和路径摘要。每组来自对应独立文件；不改变当前 Topic 选择，不创建索引。

### TC-008 `show` 与 `show <key>`

调用：

```text
$trailmap show
$trailmap show B
```

期望：`show` 展示当前 Topic 的路径详情；`show B` 仅展示 B 的 title、状态、parent、note、updates 和关闭字段。两者只读，不检查业务文件或 Git，不展示执行上下文。

### TC-009 `update --pause`

前置状态：A 为 active。

调用：

```text
$trailmap update A token expiry ruled out --pause
```

期望：只追加 note `token expiry ruled out`，A 变为 `paused`，Topic.active 变为 `null`；不自动激活其他路径。对非 active 路径使用 `--pause` 时仅将目标改为 paused，不影响当前 active。

### TC-010 显式 reopen

前置状态：A 为 active，B 为 closed。

调用：

```text
$trailmap resume B --reopen --note new evidence suggests cache ordering
```

期望：A 变为 `paused`，B 变为 `active`，Topic.active 变为 `B`；B 顶层关闭字段被移除，历史关闭 update 保留，并追加 supplied note。缺少 `--note` 或 note 为空时拒绝重开且不写入。

### TC-011 `close` 三种分类

分别调用：

```text
$trailmap close B done fix verified
$trailmap close C blocked waiting for vendor
$trailmap close D discarded hypothesis disproved
```

期望：目标路径状态均为 `closed`，`closed_as` 分别为 `done`、`blocked`、`discarded`，reason 逐字保存。关闭 active 路径时 Topic.active 变为 `null`，不自动选择替代路径；关闭非 active 路径不影响当前 active。缺少或使用其他分类时拒绝写入。

### TC-012 `rename`

调用：

```text
$trailmap rename Login incident follow-up
```

期望：只修改当前 Topic.title，Topic id、文件名、路径 key/title/status 和其他 Topic 均不变；回复首行立即使用新标题。

### TC-013 `map`

调用：

```text
$trailmap map
```

期望：输出当前 Topic 的 Mermaid `graph LR`。节点由 paths 和 parent 关系生成，label 包含原 key、逐字标题和状态；closed 节点显示关闭分类。不修改状态，不添加执行步骤。

### TC-014 `map text`

调用：

```text
$trailmap map text
```

期望：输出与 `map` 相同层级和状态的纯文本树，不输出 Mermaid，不修改状态。

### TC-015 无删除能力

尝试：

```text
$trailmap delete B
$trailmap remove login-failure
```

期望：命令无效，不删除任何 Topic、path 或 update；只提示可用的 `close` 形式，不把 delete/remove 当作自然语言别名。

### TC-016 不接受 legacy 语法

分别尝试：

```text
$trailmap mark pending check network retry
$trailmap resume B clean
$trailmap resume B informed
$trailmap resume B --informed
$trailmap subagent B
$trailmap pending check network retry --subagent
$trailmap pending check network retry --worktree
```

期望：全部作为无效输入拒绝且不写入；不忽略 leading `mark`，不接受 clean/informed 上下文模式，不接受 `subagent` 命令或 subagent/worktree 旗标，也不回退到旧存储或旧自然语言分支语法。

### TC-017 简洁输出

依次执行无歧义的 `pending`、`update`、`resume`、`close` 和 `rename`。

期望：每次回复首行是 Topic 标记，随后只给一行操作结果；只有 closed resume 命令提示、无效输入或并发冲突可增加一行必要说明。回复不得复述提示词背景、完整 JSON、Skill 规则、调试建议或“我将继续处理”的承诺。

### TC-018 同一 Topic 并发写入拒绝覆盖

前置状态：两个写入者都已读取 `login-failure.json` 的同一版本。写入者一先成功执行 `$trailmap update A first note`，使该文件内容发生变化；写入者二仍基于旧版本准备执行：

```text
$trailmap update B second note
```

期望：写入者二在落盘前重读 `login-failure.json`，检测到与其读取版本不一致后拒绝整次写入并要求用户重试。文件保留写入者一的结果，不出现 `second note`，不覆盖或合并并发变化；其他 Topic 文件也不改变。回复保持 Topic 标记，并只增加一行并发冲突说明。

## 最终验收

GREEN 实现完成后应同时满足：

- 三个 RED 提示在显式 Trailmap 调用下只产生记录器行为，在无调用时不产生 Trailmap 行为。
- `new`、`use`、`pending`、`list`、`show`、`update`、`resume`、`close`、`rename`、`map` 的变化场景全部通过。
- 恢复聊天依靠最新 Topic 标记；新聊天通过 `use <topic-id>` 选择多个 Topic 中的一个。
- 不存在全局 `active_topic_id`、`index.json`、删除行为、legacy 语法或路径执行行为。
- 不同 Topic 的写入相互隔离，所有状态转换满足单 active 不变量。
- 同一 Topic 的并发变化会在写入前被检测并拒绝覆盖。
- Topic 已选定时，输出始终以 Topic 标记开头并保持简洁，且不包含排障或执行建议。
