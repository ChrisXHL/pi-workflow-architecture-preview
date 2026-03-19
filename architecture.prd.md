# pi workflow 插件架构说明

## 一、产品定位

这是一个运行在 pi 内部的 **文件驱动型 workflow runtime**。

它的目标不是只把一串步骤画出来，而是把“长流程任务”稳定拆成：
- 可定义的 phase
- 可落盘的状态
- 可隔离执行的单节点 worker
- 可在 TUI 中持续观察和人工干预的运行面板

主要角色：
- **操作员 / 用户**：用 `/wf-*` 命令推进流程、审批节点、重试失败步骤
- **workflow-engine runtime**：负责编排、状态更新、worker 启动、UI 刷新
- **worker skill**：在独立 pi 会话里执行单节点任务

核心价值：
1. 把 workflow 的“真相”放进 `workflow/` 目录，而不是藏在会话记忆里。
2. 把每个节点变成可复现的独立执行单元，而不是在一条长对话里连续漂移。
3. 在自动执行中保留人工 gate，例如 `review -> waiting_approval -> /wf-approve`。

为什么采用这套架构：
- **可恢复**：会话关闭后，仍能从文件恢复状态。
- **可审计**：每个节点都有独立 `input.json / prompt.md / output.json / output.md`。
- **可编排**：`spec.json` 决定 phase、skill、输入输出和 prompt 注入规则。

---

## 二、整体概览

这个插件可以分成四层：

1. **定义层**：`workflow/spec.json` + `workflow/input.json`
2. **运行时状态层**：`state.json` + `jobs/*/job.json` + `vars.json`
3. **执行层**：`prepareJobExecutionFiles` + 隔离 `pi` worker + 输出回收
4. **展示控制层**：status、widget、`/wf-board`、`/wf-*` 命令

### 总体主路径

```text
用户输入 /wf-* 命令
  ↓
workflow-engine 读取 spec + input + state + jobs + vars
  ↓
选中当前节点并渲染 prompt
  ↓
校验 required_refs / strict_refs
  ↓
启动隔离 pi worker 执行该节点
  ↓
worker 写回 output.json / output.md / 主产物
  ↓
engine 包装为 $job-* / $node*，更新 state/jobs/vars
  ↓
UI 展示最新状态，并决定下一步动作
```

### 当前实现的 sample pipeline

默认内置的是一个文章型串行样例：

| 阶段 | 目标 | 做什么 | 不做什么 |
|------|------|--------|---------|
| init | 先产出流程计划 | 调用 `workflow-orchestrator` 生成 plan artifact | 不做完整内容写作 |
| research | 基于计划做调研 | 调用 `article-worker` 生成 research artifact | 不做终稿 |
| draft | 生成初稿 | 读取 research，产出 draft | 不做人审 gate |
| review | 人工前置审阅 | 生成 review artifact，并进入 `waiting_approval` | 不直接自动完成终稿 |
| final | 终稿整合 | 综合 draft + review 产出 final | 不再回溯修改前序定义 |

### 系统边界

它当前更像 **串行 phase runner**，而不是真正的 DAG 调度器。

现阶段明确支持：
- 顺序 phase 执行
- 引用注入
- 单节点隔离 worker
- 审批门
- 失败重试 / 重启
- 文件驱动状态恢复

现阶段不重点支持：
- 并行调度
- 拓扑排序
- 多 worker 资源队列
- 动态分支决策图

---

## 三、V1 — 总体架构视图

### 1. 目标
把 workflow 的定义、执行、状态、产物和 UI 统一进一个可恢复 runtime。

### 2. 路径总览

```text
命令层
  ↓
workflow-engine
  ↓
读取 workflow/ 文件
  ↓
准备节点执行文件
  ↓
启动隔离 worker
  ↓
回收输出并更新状态
  ↓
刷新 status / widget / board
```

### 3. 路径详细展开

1. **命令入口**：用户通过 `/wf-run`、`/wf-next`、`/wf-approve`、`/wf-retry` 等命令进入 runtime。
2. **定义读取**：engine 从 `spec.json` 读取 phase 定义，从 `input.json` 读取工作流输入。
3. **状态读取**：engine 从 `state.json` 与 `jobs/*/job.json` 还原当前运行位置和节点状态。
4. **变量注入**：engine 从 `vars.json` 读取 `$input`、`$agent_ctx`、`$job-*`、`$node*`。
5. **节点准备**：调用 `prepareJobExecutionFiles` 渲染 prompt，生成节点运行文件。
6. **隔离执行**：engine 以 `pi --no-session --no-extensions --no-skills --skill ...` 的方式启动独立 worker。
7. **结果回收**：读取 `output.json`、`output.md` 和主 artifact。
8. **状态回写**：更新 job 状态、全局 state、变量池和 UI 展示。

### 4. 模块定义

**① wf-* 命令层** — 用户与 runtime 的统一交互面。
负责把“开始、继续、审批、重试、重启、聚焦”翻译成 engine 的动作。

**为什么必须单独拆出来：** 因为命令层是人机交互协议，而不是执行逻辑本身。把命令语义和运行时分开，才能同时支持命令行和 board overlay。

**② workflow-engine runtime** — 真正的编排中心。
负责初始化 scaffold、选择当前节点、准备执行文件、回收 worker 输出、重算状态并刷新 UI。

**③ workflow/ 文件层** — 唯一事实来源。
其中：
- `spec.json` 管定义
- `input.json` 管本次输入
- `state.json` 管全局运行态
- `jobs/*/job.json` 管单节点状态
- `vars.json` 管变量池
- `artifacts/*.md` 管阶段产物

**④ 隔离 worker** — 短生命周期执行单元。
每次只执行一个节点，不继承当前长对话的隐式上下文。

**⑤ UI 展示层** — 稳定状态栏 + widget + 实验性 board。
负责把文件状态投影给用户，并提供少量控制动作。

### 5. 模块清单
- 命令层
- workflow-engine runtime
- workflow/ 文件层
- 隔离 worker
- UI 展示层

### 6. 本阶段不包含
- 多 worker 并发调度
- DAG 图级依赖求解
- 分布式任务队列

---

## 四、V2 — 单节点执行路径

### 1. 目标
保证任意一个 workflow 节点都能在 **可验证上下文** 下被安全执行。

### 2. 路径总览

```text
/wf-next
  ↓
选择当前节点
  ↓
标记 running
  ↓
渲染 prompt
  ↓
校验 required_refs
  ↓
启动隔离 worker
  ↓
读取 output.json
  ↓
更新 vars / state / jobs
```

### 3. 路径详细展开

1. **选择节点**：基于 `state.selected_job` 找当前节点；若节点已 success，则跳转到 `firstPendingJob()`。
2. **预校验**：若节点处于 `waiting_approval` 或 `failed`，阻止继续推进，要求 `/wf-approve` 或 `/wf-retry`。
3. **切换 running**：先把 job 状态切到 `running`，保证 UI 立即同步。
4. **渲染 prompt**：按 phase 的 `prompt_template` 把 `{{$input}}`、`{{$nodeN.raw}}` 等变量注入。
5. **校验引用**：如果 `required_refs` 缺失且 `strict_refs=true`，直接 fail fast。
6. **生成节点运行文件**：写入 `jobs/<job-id>/input.json` 和 `prompt.md`。
7. **隔离执行**：起独立 pi worker 运行 skill。
8. **读取输出**：要求 worker 至少写出 `output.json`；否则节点视为失败。
9. **包装变量**：把主产物读出来，经 `wrapOutput()` 写回 `$job-xxx` 和 `$nodeN`。
10. **审批门**：若 worker 返回 `waiting_approval`，流程停住等待人工批准。
11. **重算状态**：根据所有 job 状态更新 `completed / running / failed / pending_approval`。
12. **返回下一步动作**：向用户提示 `/wf-next` 或 `/wf-approve`。

### 4. 关键设计理由

**为什么要 fail fast，而不是即使缺引用也让 worker 试试？**
因为下游 prompt 依赖的是明确的上游结果。少了 `$node3.raw` 仍然运行，得到的大多不是可审计的“容错”，而是难定位的脏输出。

**为什么 worker 必须写 `output.json`？**
因为 runtime 不能只凭自然语言 stdout 猜测节点状态。结构化 JSON 是 runtime 与 worker 之间最小、稳定的协议层。

**为什么 review 要走 `waiting_approval`？**
因为 workflow 面向长流程生产任务，必须给人留一个确定的审阅插点，而不是把所有节点都默认为全自动成功。

### 5. 本阶段不包含
- 自动分流到多个并行节点
- 对 output.json 的复杂 schema 校验
- 多轮人工对话式审批

---

## 五、V3 — 文件与状态模型

### 1. 目标
让 runtime 可恢复、可审计、可被其他技能或自动化系统继续消费。

### 2. 路径总览

```text
spec / input / state / jobs / vars
  ↓
workflow-engine 读写这些文件
  ↓
worker 写回 runtime files + artifacts
  ↓
UI 只展示这些文件所表达的状态
```

### 3. 关键文件定义

**① `workflow/spec.json`** — 流程蓝图。
定义 `workflow_type`、`goal`、`deliverable`、`phases`、`approval_policy`。

**② `workflow/input.json`** — 本次运行输入。
通常包括 `user_input`、`agent_ctx` 或其他业务字段。

**③ `workflow/state.json`** — 全局状态机。
保存：
- `status`
- `current_phase`
- `selected_job`
- `pending_approval`
- `completed_jobs`
- `failed_jobs`

**④ `workflow/jobs/*/job.json`** — 单节点状态文件。
保存：
- `job_id`
- `phase`
- `skill`
- `status`
- `summary`
- `retries`
- `requires_approval`
- `last_error`

**⑤ `workflow/vars.json`** — 变量池。
负责承接：
- `$input`
- `$agent_ctx`
- `$job-001`
- `$node1`
- 其他上游节点包装结果

**⑥ `workflow/artifacts/*.md`** — 阶段产物。
是下游节点真正消费的业务内容层。

**⑦ `workflow/jobs/<job-id>/` 运行文件** — 节点执行证据。
包括：
- `input.json`
- `prompt.md`
- `output.json`
- `output.md`

### 4. 为什么 UI 不是事实来源

status、widget 和 board 虽然能展示状态，但它们只是：
- 读 `state.json`
- 读 `job.json`
- 读 artifact preview
- 把这些信息投影给操作员

因此 UI 可以重绘、关闭、重新打开，但 workflow 不会丢。

### 5. 本阶段不包含
- 用数据库替代 workflow/ 文件层
- 把 UI 变成主控制事实来源
- 引入外部消息总线

---

## 六、长期壁垒

1. **文件协议稳定化**：一旦 `spec / state / jobs / vars / artifacts` 协议稳定，更多 skill 和工具可以围绕它生长。
2. **隔离 worker 模式**：把单节点执行从长上下文里抽离，会比单一会话式 agent 更可复现、更适合工程化。
3. **人工 gate 能力**：把自动化和人工审批结合起来，适合高价值长流程任务。
4. **runtime 可视化**：同一套文件状态既能驱动命令，也能驱动 TUI board，这为后续更强的观测面板打下基础。

---

## 七、要避免的坑

- **问题：** 把它误当成真正的 DAG 调度器。  
  **正确做法：** 当前先把它理解为“文件驱动的串行 phase runner + 审批门”。

- **问题：** 认为 UI 持有工作流状态。  
  **正确做法：** UI 只是投影，真实状态永远在 `workflow/` 文件中。

- **问题：** 让 worker 直接依赖当前聊天上下文。  
  **正确做法：** 保持 `--no-session` 的隔离执行假设，只信 workflow 文件和 skill。

- **问题：** 缺引用时仍继续执行。  
  **正确做法：** 使用 `required_refs + strict_refs` 在运行前阻断错误上下文。

- **问题：** 下游节点直接耦合上游文件路径细节。  
  **正确做法：** 统一通过 `vars.json` 注入 `$job-* / $node*`。

---

## 附录 A：架构原则

1. 文件优先于内存。
2. 单节点优先于整条长上下文链。
3. 状态可审计优先于“看起来能跑”。
4. 审批插点优先于盲目全自动。

## 附录 B：核心数据对象

### WorkflowState
- workflow_id
- workflow_type
- status
- current_phase
- selected_job
- pending_approval
- completed_jobs
- running_jobs
- failed_jobs
- artifacts
- updated_at

### WorkflowJob
- job_id
- order
- phase
- skill
- status
- summary
- input_artifacts
- output_artifacts
- retries
- requires_approval
- last_error
- updated_at

## 附录 C：全部模块速查表

| 模块 | 作用 | 关键输入 | 关键输出 |
|------|------|----------|----------|
| wf-* 命令层 | 人机入口 | 用户命令 | engine action |
| workflow-engine runtime | 编排核心 | spec/input/state/jobs/vars | worker launch + state update |
| prepareJobExecutionFiles | 节点准备 | prompt_template + vars | input.json + prompt.md |
| 隔离 worker | 单节点执行 | prompt.md + skill | output.json + artifact |
| vars.json | 提示词注入总线 | node outputs | $job-* / $node* |
| UI 层 | 状态展示与控制 | state/jobs/artifacts | operator feedback |
