# GacUI `.github/` 协作系统深度解析（含代码级实现、完整案例与时序推演）

## 1. 为什么 `.github/` 是本仓库的“控制平面”

在这个仓库里，`.github/` 不是单纯的 CI 配置目录，而是一个完整的“开发流程操作系统”——它把 **规范、知识、任务文档、自动化脚本、智能 Agent 门户** 串成闭环。

> 可以把它理解成：
>
> - `Source/` 是业务代码层（被修改对象）
> - `Test/` 是工程编排与验证层
> - `.github/` 是流程控制层（告诉你怎么改、怎么验、怎么记录、怎么回滚）

---

## 2. `.github/` 结构分层与职责映射

## 2.1 Guidelines：工程规则与动作约束

- 代表文件：
  - `.github/Guidelines/Building.md`
  - `.github/Guidelines/Running-UnitTest.md`
  - `.github/Guidelines/SourceFileManagement.md`
- 作用：定义“可执行动作边界”，例如构建、运行、调试、文件增删改应遵循的规则。

## 2.2 KnowledgeBase：技术决策知识底座

- 入口：`.github/KnowledgeBase/Index.md`
- 作用：在“改代码前”提供 API 选择和设计解释，防止拍脑袋式改动。
- 特点：不仅有项目级指导，还有细分专题（类型系统、反射、正则、时间、集合等）。

## 2.3 prompts：任务协议层（Prompt Protocol）

- 代表文件：
  - `.github/prompts/code.prompt.md`
  - `.github/prompts/4-execution.prompt.md`
  - `.github/prompts/5-verifying.prompt.md`
- 作用：把“自然语言请求”映射为标准化执行流程（实现 -> 构建 -> 测试 -> 回归复查）。

## 2.4 TaskLogs：任务状态机持久化

- 代表文件：
  - `.github/TaskLogs/Copilot_Task.md`
  - `.github/TaskLogs/Copilot_Planning.md`
  - `.github/TaskLogs/Copilot_Execution.md`
- 作用：将上下文从“临时对话”落地为“可审计文档”。
- 典型行为：`copilotPrepare.ps1` 会初始化或备份这些文件，保证每次任务有可追溯的起点。

## 2.5 Scripts：可执行自动化入口

- 代表文件：
  - `.github/Scripts/copilotPrepare.ps1`（准备/备份任务文档）
  - `.github/Scripts/copilotBuild.ps1`（统一构建入口）
  - `.github/Scripts/copilotExecute.ps1`（统一执行入口）
  - `.github/Scripts/copilotShared.ps1`（公共方法：定位 solution、可执行文件、调试参数）
- 作用：把“规范动作”固化成脚本，减少人工命令偏差。

## 2.6 Agent：智能门户与会话编排

- 入口：`.github/bot.ps1`
- 核心代码：`.github/Agent/packages/CopilotPortal/src/*.ts`
- 作用：启动本地 Portal 服务，管理 Copilot 会话、任务、作业、实时回调与状态追踪。

---

## 3. 逻辑是如何实现的（结合代码）

## 3.1 启动层：`bot.ps1` 触发 Agent 门户

`bot.ps1` 做了三件事：

1. 进入 `.github/Agent`
2. `yarn install` 安装依赖
3. `yarn compile` 编译 TS
4. 启动 `yarn portal --port 9999`

这说明 Agent 子系统是“独立可运行服务”，而不是零散脚本集合。

## 3.2 服务层：`index.ts` 做 API 路由分发

`.github/Agent/packages/CopilotPortal/src/index.ts` 是 HTTP 入口：

- 负责读取 `--port`、`--test`
- 自动向上查找 `.git` 以定位 repoRoot
- 提供 `/api/*` 路由分发：
  - `api/token`
  - `api/copilot/session/*`
  - `api/copilot/task/*`
  - `api/copilot/job/*`
- 静态页面托管 `assets/`

本质上它是“控制平面网关”。

## 3.3 实时层：`sharedApi.ts` 的 LiveEntity + Token 模型

这是本系统最关键的设计之一：

- `createLiveEntityState()` 创建带响应缓冲区的实体状态
- `waitForLiveResponse()` 支持 token 粒度的长轮询
- `pushLiveResponse()` 负责投递并触发等待中的请求返回
- `closeLiveEntity()` / `shutdownLiveEntity()` 负责关闭与清理

为什么这么设计：

- 多个前端消费者可并行读取同一会话历史
- 通过 `token.position` 维护各自读指针
- 避免 WebSocket 复杂度，使用 HTTP 长轮询实现“准实时”体验

## 3.4 会话层：`copilotApi.ts` 负责会话生命周期

`copilotApi.ts` 维护 `sessions: Map<string, SessionState>`，核心行为：

- `helperSessionStart()`：创建会话，绑定回调，将每段输出推入 live 缓冲
- `apiCopilotSessionQuery()`：触发 `session.sendRequest()`，结果通过 live API异步读取
- `apiCopilotSessionLive()`：通过 token 拉取增量回调
- `apiCopilotSessionStop()`：标记关闭并触发 live 结束

这形成了“请求与响应分离”的结构：

- Query：只负责投递
- Live：负责消费流

## 3.5 任务层：`taskApi.ts` 实现可恢复任务执行

`taskApi.ts` 在“会话之上”实现任务语义：

- `CopilotTaskImpl` 维护任务状态、运行时变量、活动会话
- `sendMonitoredPromptTask()` 对崩溃场景进行重试（MAX_CRASH_RETRIES）
- `TaskStoppedError` 确保 stop 后快速终止后续请求
- `monitorSessionTools()` 从 tool 事件中提取结构化结果（例如布尔判断、文档摘要）

这是从“聊天接口”到“可编排任务执行引擎”的关键跃迁。

## 3.6 脚本层：构建与执行统一入口

- `copilotBuild.ps1`
  - 通过 `GetSolutionDir` 自动定位 `.sln`
  - 读取 `VLPP_VSDEVCMD_PATH`
  - 统一调用 msbuild 并输出日志到 `Build.log`
- `copilotExecute.ps1`
  - 自动找最近构建产物
  - 自动读取 `.vcxproj.user` 调试参数
  - 输出到 `Execute.log`

这使“构建/运行”不依赖个人本地习惯，而是标准化流水线动作。

---

## 4. 一个完整案例：从“需求输入”到“可追踪输出”

下面给出一个可复现实战案例：

场景：开发者要执行一个任务（例如代码分析任务），并在 Portal 中实时看结果。

### 步骤 A：准备上下文

1. 执行准备脚本（逻辑上）：`copilotPrepare.ps1`
2. 任务文档初始化到 `TaskLogs/`：Task / Planning / Execution
3. Agent 根据 prompt 规则选择执行模式（code / execute / verify 等）

### 步骤 B：启动门户服务

1. `bot.ps1` 启动 portal
2. 浏览器访问 `index.html`
3. 前端先调用 `api/config` 获得 repoRoot

### 步骤 C：开启会话并发请求

1. 调用 `api/token` 获取 live token
2. 调用 `api/copilot/session/start/{model}` 创建 session
3. 调用 `api/copilot/session/{id}/query` 发送用户请求

### 步骤 D：增量消费结果

1. 前端循环调用 `api/copilot/session/{id}/live/{token}`
2. 服务端从 LiveEntity 读取该 token 未读响应
3. 回传 callback 列表（onReasoning / onMessage / onToolExecution ...）

### 步骤 E：任务编排与失败恢复（可选）

1. 调用 `api/copilot/task/start/...` 启动任务
2. `taskApi.ts` 中若 session 崩溃 -> 自动替换会话并重试
3. 通过 task live API 持续看到 `[SESSION STARTED]`、`[SESSION CRASHED]`、`[TASK SUCCEEDED]`

### 步骤 F：关闭与清理

1. 调用 `api/copilot/session/{id}/stop` 或 `api/stop`
2. `closeLiveEntity` / `shutdownServer` 触发资源释放
3. 客户端继续 drain 直到收到 closed/notfound

---

## 5. 推理式逐步演示（为什么这个架构成立）

## 推理 1：为什么不用单次请求直接返回完整答案？

因为 Agent 输出包含多阶段事件（reasoning/message/tool/agent turn），天然是流式事件，不是单一字符串。`LiveEntity` 允许有序缓存并增量消费。

## 推理 2：为什么要 token + position，而不是全局游标？

因为多个消费者（页面、调试器、任务追踪）读取速度不同。全局游标会导致慢消费者丢消息。token 的独立 position 可解耦读取进度。

## 推理 3：为什么 query 与 live 分离？

`query` 只负责触发，`live` 负责拉流；分离后：

- 请求入口更轻
- 超时与重试语义集中在 live 层
- 前端逻辑更可控（轮询间隔、重连策略）

## 推理 4：为什么任务层要自己管理重试？

会话层只保证“会话能交互”，任务层才知道“业务是否成功”。因此 crash retry 放在 `taskApi.ts` 才能结合 criteria/tool 反馈做决策。

## 推理 5：为什么 `.github/Scripts` 还要保留独立 PowerShell？

Portal 解决“智能任务协同”，而脚本解决“工程动作标准化”。两者职责不同：

- 脚本保证 build/run/debug 一致性
- Portal 保证 AI 交互与任务状态可视化

两层叠加形成完整开发闭环。

---

## 6. ANSI Art 时序图（端到端流程）

```ansi
+----------------+        +----------------------+        +-------------------------+
| Developer/User |        | Copilot Portal(Server)|       | Copilot SDK / Session  |
+--------+-------+        +-----------+----------+        +------------+------------+
         |                            |                                |
         | 1) start bot.ps1           |                                |
         |--------------------------->| yarn install/compile/portal    |
         |                            |                                |
         | 2) GET /api/token          |                                |
         |--------------------------->| create UUID token              |
         |<---------------------------| {token}                        |
         |                            |                                |
         | 3) POST /api/copilot/session/start/{model}                 |
         |--------------------------->| helperSessionStart()           |
         |                            |------------------------------->| startSession()
         |                            |<-------------------------------| session created
         |<---------------------------| {sessionId}                    |
         |                            |                                |
         | 4) POST /api/copilot/session/{id}/query                    |
         |--------------------------->| session.sendRequest()          |
         |                            |------------------------------->| stream callbacks
         |                            |<-------------------------------| onReasoning/onMessage...
         |                            | pushLiveResponse(entity)       |
         |                            |                                |
         | 5) GET /api/copilot/session/{id}/live/{token} (loop)       |
         |--------------------------->| waitForLiveResponse()          |
         |<---------------------------| {responses:[...]}              |
         |                            |                                |
         | 6) POST /api/copilot/task/start/... (optional)             |
         |--------------------------->| CopilotTaskImpl.execute()      |
         |                            | crash? replace session + retry |
         |<---------------------------| task live events               |
         |                            |                                |
         | 7) POST /api/stop          |                                |
         |--------------------------->| shutdownLiveEntity + stopClient|
         |<---------------------------| {}                             |
         |                            |                                |
+--------+-------+        +-----------+----------+        +------------+------------+
|   TaskLogs/    |        |   Scripts(.ps1)      |        |  Build/Test Artifacts   |
+----------------+        +----------------------+        +-------------------------+
    ^  planning/execution persisted   ^ build/execute unified entry
    +-------------------------------------------------------------------+
```

---

## 7. 结论

1. `.github/` 在本仓库中扮演“流程控制中枢”，并非普通元数据目录。
2. 其核心实现是“文档协议（prompts+TaskLogs）+ 自动化脚本（Scripts）+ 智能服务（Agent Portal）”三位一体。
3. 代码级关键在 `sharedApi.ts` 的 token-live 设计、`copilotApi.ts` 的生命周期管理、`taskApi.ts` 的崩溃恢复机制。
4. 这种设计能在复杂任务里同时满足：
   - 实时可观测
   - 多消费者并行读取
   - 失败可恢复
   - 任务全过程可审计

