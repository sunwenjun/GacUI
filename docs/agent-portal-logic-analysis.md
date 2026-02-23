# Agent Portal 逻辑实现全链路分析（结合代码）

本文对 `.github/Agent` 子系统进行“启动层 → 服务层 → 实时层 → 会话层 → 任务层 → 脚本层”的完整拆解，并给出一个可执行的端到端案例。

## 1. 启动层：`bot.ps1` 如何触发 Agent 门户

`REPO-ROOT/.github/bot.ps1` 的职责非常聚焦：

1. 切换目录到 `.github/Agent`
2. `yarn install`
3. `yarn compile`
4. 新开 PowerShell 启动 `yarn portal --port 9999`

对应代码如下：

```powershell
Push-Location $PSScriptRoot\Agent
try {
    yarn install
    yarn compile
    start powershell {yarn portal --port 9999}
}
finally {
    Pop-Location
}
```

这说明 Portal 是“可独立运行的 Node 服务”，不是散落在仓库中的脚本片段。

---

## 2. 服务层：`index.ts` 如何做控制平面网关

文件：`.github/Agent/packages/CopilotPortal/src/index.ts`

### 2.1 入口参数与运行模式

- 默认端口 `8888`
- `--port` 覆盖端口
- `--test` 开启测试模式并调用 `setTestMode(true)`

### 2.2 自动发现 repoRoot

`findRepoRoot(startDir)` 会向父目录逐级查找 `.git`，用于把仓库根路径注入 `api/config` 返回。

### 2.3 API 路由分发

`handleApi` 基于 `apiPath` 做显式路由分发，覆盖：

- `api/token`
- `api/copilot/session/*`
- `api/copilot/task/*`
- `api/copilot/job/*`
- 以及 `api/config`、`api/stop`、`api/copilot/models`

**结构特征**：

- API 路径前缀统一为 `/api/`
- 非 API 请求统一落到 `assets/` 静态资源
- `pathname === "/"` 时回退到 `index.html`

因此，它本质上是一个“Portal 控制平面 + 静态前端托管”的单体网关。

---

## 3. 实时层：`sharedApi.ts` 的 LiveEntity + Token 机制

文件：`.github/Agent/packages/CopilotPortal/src/sharedApi.ts`

这是系统实现“HTTP 长轮询准实时流式体验”的关键。

## 3.1 核心状态结构

- `LiveEntityState`
  - `responses: LiveResponse[]`：全量事件缓冲
  - `tokens: Map<string, TokenState>`：每个消费者一个读指针
  - `closed / countDownBegin / countDownMs`：生命周期控制
- `TokenState`
  - `position`：当前 token 已读到的下标
  - `pendingResolve`：挂起请求回调
  - `httpTimeout / batchTimeout`：长轮询超时与批处理窗口

## 3.2 四个关键动作

1. `createLiveEntityState()`：创建可复用、可清理的流状态
2. `waitForLiveResponse()`：按 token 的 `position` 增量读取；无新数据则挂起等待
3. `pushLiveResponse()`：写入事件，并唤醒有未读数据的 token
4. `closeLiveEntity()/shutdownLiveEntity()`：关闭并清理

## 3.3 设计上的“妙处”

- **多消费者并行**：不同 token 互不干扰（各自 `position`）
- **无 WebSocket 依赖**：纯 HTTP 即可实现“实时感”
- **流量优化**：
  - `onMessage/onReasoning` 会做 delta 合并
  - `onEndMessage/onEndReasoning` 会清理历史 delta，减少冗余
  - 5 秒批窗口让突发 token 输出更平滑

---

## 4. 会话层：`copilotApi.ts` 的生命周期管理

文件：`.github/Agent/packages/CopilotPortal/src/copilotApi.ts`

## 4.1 会话状态模型

通过 `sessions: Map<string, SessionState>` 管理会话：

- `session`: SDK 会话对象
- `entity`: 对应 LiveEntity
- `closed/sessionError`: 生命周期与错误状态

## 4.2 会话创建

`helperSessionStart()`：

- 分配 `session-{n}`
- 创建 `LiveEntityState`
- 绑定一组回调（`onStartMessage/onMessage/onEndMessage/...`）
- 每个回调都 `pushLiveResponse(...)` 到实体缓冲

## 4.3 请求响应分离模型

- `apiCopilotSessionQuery()`
  - 只负责 `sendRequest(body)` 投递
  - 不同步返回模型输出
- `apiCopilotSessionLive()`
  - 通过 `(sessionId, token)` 调用 `waitForLiveResponse`
  - 返回“增量事件批次”

这就是典型的 **Query（投递）/Live（消费）分离**。

## 4.4 停止与关闭

- `apiCopilotSessionStop()` 标记会话关闭并 `closeLiveEntity`
- `shutdownServer()` 统一 `shutdownLiveEntity` + `stopCoplilotClient`

---

## 5. 任务层：`taskApi.ts` 如何把会话升级为“可编排任务引擎”

文件：`.github/Agent/packages/CopilotPortal/src/taskApi.ts`

## 5.1 `CopilotTaskImpl` 的职责

`CopilotTaskImpl` 维护：

- 任务模式（借用会话 / 单模型 / 多模型）
- 活跃会话集合 `activeSessions`
- 运行时变量 `runtimeValues`
- 任务状态 `Executing/Succeeded/Failed`

并通过 `executeBorrowing/executeSingle/executeMultiple` 走不同策略。

## 5.2 可恢复执行：崩溃重试

- 常量 `MAX_CRASH_RETRIES = 5`
- `sendMonitoredPromptTask()`：任务会话崩溃时，关闭旧会话并创建新会话重试
- `sendMonitoredPromptDriving()`：驾驶会话按 `entry.drivingSessionRetries` 的模型预算切换重试
- `TaskStoppedError`：当 stop 发生后，`guardedSendRequest()` 立即短路后续请求

## 5.3 工具监控与结构化状态抽取

`monitorSessionTools()` 监听 `tool.execution_start`：

- 记录工具调用集合 `toolsCalled`
- 从工具参数提取结构化信息写入 `runtimeValues`
  - `job_prepare_document` → `reported-document`
  - `job_boolean_true/false` → 布尔判定及 reason

这使任务引擎可以做“条件判断 + 失败重试 + 再决策”。

## 5.4 Task Live API

- `registerJobTask()` 为任务创建独立 `LiveEntity`
- `apiTaskStart()` 启动任务并把回调（success/fail/decision）推入 live 流
- `apiTaskLive()` 通过 token 增量读取
- `apiTaskStop()` 在可关闭模式下触发 stop + close

---

## 6. 脚本层：构建/执行标准化入口

文件：

- `.github/Scripts/copilotBuild.ps1`
- `.github/Scripts/copilotExecute.ps1`
- `.github/Scripts/copilotShared.ps1`

## 6.1 `copilotBuild.ps1`

- 调用 `GetSolutionDir` 自动寻找 `.sln`
- 读取 `VLPP_VSDEVCMD_PATH`
- 使用 `MSBUILD ...` 构建并写入 `Build.log`

## 6.2 `copilotExecute.ps1`

- `GetLatestModifiedExecutable` 自动选择最近构建产物
- `GetDebugArgs` 自动读取 `.vcxproj.user` 调试参数
- 执行并记录到 `Execute.log`

整体效果是把“开发者本地习惯”收敛为“稳定流水线动作”。

---

## 7. 完整案例：从启动到任务结束（一步一步推演）

下面用“启动一个 session，提交任务，前端增量消费结果”的场景演示。

### Step A：启动 Portal

- 执行 `.github/bot.ps1`
- 完成依赖安装、TS 编译并启动 `portal --port 9999`

### Step B：前端获取 token

- 调用 `GET /api/token`
- 服务返回随机 UUID token

### Step C：创建会话

- 调用 `POST /api/copilot/session/start/{model-id}`（body 可带 workingDirectory）
- 服务器验证模型与路径后，创建 `session-{n}` + `LiveEntity`

### Step D：投递问题

- 调用 `POST /api/copilot/session/{session-id}/query`
- 该请求立即返回 `{}`，真正输出走异步 Live 通道

### Step E：长轮询消费

- 前端循环调用 `GET /api/copilot/session/{session-id}/live/{token}`
- `waitForLiveResponse()` 根据 token.position 返回未读增量
- 若 5 秒内无新消息，返回 `HttpRequestTimeout`（前端继续轮询）

### Step F：启动任务（借用会话）

- 调用 `POST /api/copilot/task/start/{task-name}/session/{session-id}`
- `apiTaskStart()` -> `startTask()` -> `CopilotTaskImpl.execute*()`
- 任务回调（`taskDecision/taskSucceeded/taskFailed`）被写入 Task 的 LiveEntity

### Step G：任务端增量消费

- 调用 `GET /api/copilot/task/{task-id}/live/{token}`
- 持续获得决策日志、成功或失败信号

### Step H：关闭

- 会话 stop 或任务结束时，`closeLiveEntity()` 标记关闭并进入倒计时
- token 全部清理后，onDelete 回调移除 Map 状态

---

## 8. ANSI Art 时序图（Session + Task）

```text
+-------------+         +------------------+         +------------------+         +------------------+
|   Frontend  |         |   index.ts(API)  |         |   copilotApi.ts  |         |   sharedApi.ts   |
+-------------+         +------------------+         +------------------+         +------------------+
       |                          |                             |                             |
       |  GET /api/token          |                             |                             |
       |------------------------->|                             |                             |
       |<-------------------------|  {token}                    |                             |
       |                          |                             |                             |
       |  POST /session/start     |---------------------------->| helperSessionStart()        |
       |------------------------->|                             | create callbacks            |
       |<-------------------------|  {sessionId}                | createLiveEntityState()---->|
       |                          |                             |<----------------------------|
       |                          |                             |                             |
       |  POST /session/{id}/query|---------------------------->| session.sendRequest()       |
       |------------------------->|                             | pushLiveResponse(...)------>|
       |<-------------------------|  {}                         |<----------------------------|
       |                          |                             |                             |
       |  GET /session/{id}/live/{token}                        | waitForLiveResponse()------>|
       |------------------------->|---------------------------->|                             |
       |<-------------------------|<----------------------------|  {responses/error}          |
       |                          |                             |                             |
       |  POST /task/start/...    |---------------------------->| apiTaskStart -> startTask   |
       |------------------------->|                             |                             |
       |<-------------------------|  {taskId}                   |                             |
       |                          |                             |                             |
       |  GET /task/{id}/live/{token}                           | task callback push -------->|
       |------------------------->|---------------------------->|                             |
       |<-------------------------|<----------------------------|  taskDecision/succeeded/... |
       |                          |                             |                             |
       |  POST /session/{id}/stop |---------------------------->| closeLiveEntity()---------->|
       |------------------------->|                             |                             |
       |<-------------------------|  {result:"Closed"}         |                             |
       |                          |                             |                             |
```

---

## 9. 结论

1. **架构分层清晰**：启动、网关、实时流、会话编排、任务编排、脚本执行各司其职。  
2. **关键抽象是 LiveEntity**：它把异步 token 流统一成“HTTP 长轮询 + token 游标”模型。  
3. **任务能力来自会话之上封装**：崩溃重试、工具事件抽取、条件判断让系统从 chat 进化为 workflow engine。  
4. **工程可复现性强**：构建与执行脚本统一入口，降低了“机器差异/习惯差异”影响。
