# GacUI 项目 `.github` 目录完整功能分析

> 目标：解释当前仓库 `.github` 目录的功能分层、关键代码实现方式、完整运行案例，以及端到端时序流程（ANSI Art）。

## 1. `.github` 在本项目中的定位

这个项目的 `.github` 并不只是 GitHub 平台配置，而是一个**完整的“Agent 自动化工作平台”**，覆盖：

1. 指令系统（copilot-instructions + prompts）
2. 任务知识库（KnowledgeBase）
3. 构建/执行脚本（Scripts）
4. 工作流实验（Experiment）
5. 任务日志沉淀（TaskLogs / Learning）
6. 一个可运行的 TypeScript Web 服务（Agent/CopilotPortal）

核心入口是 `.github/Agent/packages/CopilotPortal/src/index.ts`，它将「静态页面 + REST API + Copilot Session/Task/Job 执行引擎」绑定在一个 HTTP 服务里。

---

## 2. 目录分层与功能地图

## 2.1 指令与流程编排

- `.github/copilot-instructions.md`
  - 全局执行规范：如何找项目、如何构建、如何跑测试、Linux/Windows 差异等。
- `.github/prompts/*.prompt.md`
  - 按“意图关键词”拆分任务模板（scrum/design/plan/execute/verify/review/code 等）。
- `.github/Project.md`
  - 指定默认工作 solution 与 unit test 项目。

**本质**：这是“任务解释层”。先把用户输入映射到 prompt 模板，再驱动后续执行。

## 2.2 知识与规范

- `.github/KnowledgeBase/Index.md` + 大量 `KB_*.md`
  - API 选型、设计说明、测试框架、编码偏好。

**本质**：这是“决策依据层”。给 Agent 一个稳定的技术知识背景。

## 2.3 脚本自动化层（PowerShell）

- `.github/Scripts/copilotBuild.ps1`：调用 VS 环境 + MSBUILD，并落盘 Build.log。
- `.github/Scripts/copilotExecute.ps1`：定位最新 exe，执行并输出 Execute.log。
- `.github/Scripts/copilotPrepare.ps1`：初始化/备份 TaskLogs。
- `.github/Scripts/copilotPrepareReview.ps1`：Review 文件状态流转。
- `.github/Scripts/copilotShared.ps1`：共享函数（找 sln、找可执行、读取调试参数）。

**本质**：这是“操作执行层”。

## 2.4 Agent Portal（最核心）

- `.github/Agent/packages/CopilotPortal/src/*.ts`
- `.github/Agent/packages/CopilotPortal/assets/*`
- `.github/Agent/packages/CopilotPortal/test/*`

它提供：

- Web 页面：index/jobs/jobTracking/test
- API：session/task/job 生命周期管理
- 运行引擎：串行/并行/循环/分支 WorkTree 执行
- 实时通信：token + long polling 的 live 消息流

**本质**：这是“运行时引擎 + 可视化层”。

---

## 3. 核心代码逻辑如何实现

## 3.1 入口与路由分发（index.ts）

`index.ts` 做了三件事：

1. 解析参数 `--port` 与 `--test`
2. 路由 `GET/POST /api/*` 到各模块 API
3. 提供静态资源页面

关键点：

- 默认启动后若不是 testMode，会自动 `installJobsEntry(entry)` 安装预置 jobsData。
- API 被分成三大域：
  - `copilot/session/*`
  - `copilot/task/*`
  - `copilot/job/*`

这意味着业务层不是“页面按钮直接执行逻辑”，而是通过统一 API 抽象层可复用。

## 3.2 Live 通道与 Token Drain 模型（sharedApi.ts）

`sharedApi.ts` 是该系统最关键的基础设施之一：

- `createLiveEntityState` 创建实体消息缓冲（session/task/job 都用）
- `pushLiveResponse` 追加消息
- `waitForLiveResponse` 按 token + 游标位置增量拉取
- `closeLiveEntity` 进入 closed + 倒计时保留
- `shutdownLiveEntity` 清理超时与挂起请求

设计亮点：

1. **每个 token 独立 position**，实现“同一实体多观察者并行消费”。
2. **batch + timeout**，降低高频 delta 的请求抖动。
3. **closed 后保留窗口**（测试模式 5s、正常 60s），避免客户端断线重连丢尾包。

## 3.3 Session 管理（copilotApi.ts + copilotSession.ts）

`copilotApi.ts`：

- 创建 session map（`session-1`、`session-2`）
- 启停、query、live poll
- 将 SDK 回调统一压入 LiveEntity

`copilotSession.ts`：

- 对 `@github/copilot-sdk` 包装
- 注册自定义 tools：
  - `job_prepare_document`
  - `job_boolean_true`
  - `job_boolean_false`
  - `job_prerequisite_failed`
- 监听 reasoning/message/tool/turn/idle 事件并回调上层

设计亮点：

- 通过 tool 调用结果反向驱动“任务判定（criteria）”与“运行时变量”。
- 对 `glob` 工具禁用（Windows 兼容策略），在 hook 层做前置拦截。

## 3.4 Task 引擎（taskApi.ts）

`startTask` -> `CopilotTaskImpl.execute()` 分三种模式：

1. Borrowing Session：复用外部 session
2. Managed Single Model：单模型托管 session
3. Managed Multiple Models：多模型驱动与重试

能力：

- 会话崩溃重试（`drivingSessionRetries`）
- Criteria 检查：
  - 工具是否调用
  - 条件 prompt 是否返回 boolean true
- failureAction 重试：可注入 additionalPrompt
- stop 机制：`TaskStoppedError` 快速终止

## 3.5 Job 引擎（jobsApi.ts）

`executeWork` 支持 WorkTree 五种节点：

- `Ref`：执行一个 task
- `Seq`：串行
- `Par`：并行
- `Loop`：循环（pre/post condition）
- `Alt`：分支

`startJob` 维护运行状态与取消语义：

- `runningWorkIds`
- `activeTasks`
- `stop()` 会级联停止所有运行中 task

并提供 API：

- 列表：`/copilot/job`
- 启动：`/copilot/job/start/{jobName}`
- 状态：`/copilot/job/{jobId}/status`
- 实时：`/copilot/job/{jobId}/live/{token}`
- 运行中/最近一小时：`/copilot/job/running`

## 3.6 配置数据模型（jobsDef.ts + jobsData.ts）

`jobsDef.ts` 负责：

- Entry/Task/Job/Work 类型系统
- Prompt 变量展开
- workId 自动分配
- entry 合法性校验（模型引用、任务引用、prompt 约束、网格一致性）

`jobsData.ts` 负责：

- 提供实际任务库和作业矩阵
- 定义自动化链路（如 `design-next-automate`）
- 启动时 `validateEntry(entryInput)` 保证配置可执行

---

## 4. 完整案例：`design-next-automate` 如何从输入走到完成

我们选一个“完整且复杂”的链路：`design-next-automate`。

其 work 大致是：

1. design 文档链路（problem-next + review + commit）
2. plan 文档链路（problem + review + commit）
3. summary 文档链路（problem + review + commit）
4. execute-task + git-commit
5. verify-task + git-commit
6. git-push

### Step-by-step 推演

### Step 1：前端发起 Job 启动

客户端调用：

`POST /api/copilot/job/start/design-next-automate`

body 第一行是 workingDirectory，后面是 userInput。

### Step 2：后端创建 JobState

- 分配 `job-<n>`
- 创建 live entity
- 绑定 job callback（jobSucceeded/jobFailed/workStarted/workStopped）

### Step 3：进入 WorkTree 执行

`executeWork` 根据节点类型递归执行：

- Seq 节点：逐个子节点
- Ref 节点：触发 `startTask`

### Step 4：Task 内部发 Prompt 与监控工具

当某个 task 执行时：

1. 展开 prompt 变量
2. `sendRequest` 给 session
3. 从 tool 事件里提取：
   - 文档路径（`job_prepare_document`）
   - 条件真假（`job_boolean_true/false`）
4. criteria 判定是否通过
5. 不通过则按 failureAction 重试

### Step 5：Live API 持续回传

- task 维度：`taskDecision/taskSessionStarted/taskSucceeded...`
- job 维度：`workStarted/workStopped/jobSucceeded...`

UI 通过 token 轮询拉取并渲染流程图状态。

### Step 6：终态收敛

- 成功：jobSucceeded，entity close
- 失败：jobFailed 或 jobCanceled，entity close
- 客户端 drain 到 `JobsClosed` 后结束轮询

---

## 5. 这个系统的工程价值（结论）

1. **结构清晰**：配置（jobsData）与执行引擎（task/job api）分离。
2. **可观测性高**：所有关键事件进入 live channel，天然可做追踪 UI。
3. **鲁棒性好**：session 崩溃重试、多模型 fallback、任务可中断。
4. **可扩展性强**：新增 task/job 主要改配置，少改引擎代码。
5. **风险控制到位**：入口校验 + 配置校验 + 生命周期倒计时清理。

---

## 6. ANSI Art 时序图（端到端）

```text
+------------------+      +-------------------+      +------------------+
| Browser (jobs UI)|      | CopilotPortal API |      | LiveEntity Store |
+------------------+      +-------------------+      +------------------+
         |                          |                           |
         | POST /api/token          |                           |
         |------------------------->| generate UUID             |
         |<-------------------------| {token}                   |
         |                          |                           |
         | POST /api/copilot/job/start/design-next-automate    |
         |------------------------->| create JobState+entity    |
         |                          | startJob()                |
         |                          |----+                      |
         |                          |    | executeWork(Seq)     |
         |                          |    v                      |
         |                          |  startTask(Ref)           |
         |                          |    |                      |
         |                          |    v                      |
         |                          |  startSession(model)      |
         |                          |    |                      |
         |                          |    v                      |
         |                          |  sendRequest(prompt)      |
         |                          |    | tool/reasoning/msg   |
         |                          |    v                      |
         |                          | pushLiveResponse -------->| append responses
         |                          |                           |
         | GET /api/copilot/job/{id}/live/{token}              |
         |------------------------->| waitForLiveResponse       |
         |<-------------------------| {responses:[...]}         |
         | render graph status      |                           |
         |                          |                           |
         | (loop polling...)        |                           |
         |------------------------->| waitForLiveResponse       |
         |<-------------------------| {responses:[...]}         |
         |                          |                           |
         |                          | jobSucceeded/jobFailed    |
         |                          | closeLiveEntity ---------->| closed + countdown
         | GET live drain           |                           |
         |------------------------->|                           |
         |<-------------------------| {error:"JobsClosed"}     |
         | stop polling             |                           |
```

---

## 7. 建议阅读顺序（快速上手）

1. `src/index.ts`（总入口）
2. `src/sharedApi.ts`（live 协议核心）
3. `src/copilotApi.ts` + `src/copilotSession.ts`（session 事件桥接）
4. `src/taskApi.ts`（任务策略与重试）
5. `src/jobsApi.ts`（工作流执行）
6. `src/jobsDef.ts` + `src/jobsData.ts`（配置系统）

