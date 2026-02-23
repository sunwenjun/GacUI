# 2.1 Guidelines：工程规则与动作约束（翻译总结）

## 涉及文档

- `.github/Guidelines/Building.md`
- `.github/Guidelines/Running-UnitTest.md`
- `.github/Guidelines/SourceFileManagement.md`

对应中文翻译文件：

- `.github/Guidelines/Building.zh-CN.md`
- `.github/Guidelines/Running-UnitTest.zh-CN.md`
- `.github/Guidelines/SourceFileManagement.zh-CN.md`

## 这三份文档主要讲了什么

### 1) Building.md：构建动作边界

核心是“只能走统一脚本，不要绕过流程”。

- 构建唯一入口：`copilotBuild.ps1`
- 构建前先停止调试器：`copilotDebug_Stop.ps1`
- 结果判定唯一依据：`Build.log` 的最终状态
- 通过 `Build.log.unfinished` 判断“是否仍在构建中”
- 不要手工删日志，避免破坏排障链路

这份文档实际上在强调：**构建过程必须可重复、可追踪、可审计**。

### 2) Running-UnitTest.md：测试动作边界

核心是“测试执行也要走统一入口，并明确筛选机制和结果判读方式”。

- 测试唯一入口：`copilotExecute.ps1`
- 测试前先停止调试器
- 测试文件筛选由 `*.vcxproj.user` 的 `LocalDebuggerCommandArguments` 控制
- 只有在“预期文件被跳过”时才允许改过滤器
- 不要删除 `*.vcxproj.user`，不要清空过滤器
- 结果判定唯一依据：`Execute.log` 最后汇总（`Passed test files/cases`）

这份文档的重点是：**控制测试范围 + 保证结果可信 + 降低无效全量执行成本**。

### 3) SourceFileManagement.md：文件增删改动作边界

核心是“源文件变更不只是改物理文件，还要维护工程元数据”。

- 解决方案 (`*.sln/*.slnx`) 与项目 (`*.vcxproj/*.vcxitems`) 是组织主干
- `*.filters` 决定 IDE 中虚拟目录呈现
- `*.vcxproj.user` 是本地运行参数容器（通常不进 git）
- 新增文件要同时维护项目文件与 filters 映射
- 重命名/删除文件要同步更新所有相关工程文件

这份文档的本质是：**保证 IDE 视图、编译输入和运行配置三者一致**。

## 可以学到哪些实用技巧

### 技巧 1：把“脚本入口”当作工程协议，不走捷径

- 统一入口（构建/测试）能减少“我这边能跑、你那边不行”的环境偏差。
- 故障排查时先看统一日志，比看零散终端输出更可靠。

### 技巧 2：用 unfinished 临时文件判断流程状态

- `Build.log.unfinished` / `Execute.log.unfinished` 是非常实用的“进度锁”。
- 自动化脚本里可以先检查该文件是否存在，再决定是否读取正式日志。

### 技巧 3：测试过滤器要“最小必要变更”

- 只在必要时调整 `*.vcxproj.user` 过滤器。
- 保留与当前任务强相关但未直接修改的测试文件，有助于防止回归。

### 技巧 4：源文件管理要遵循“双同步”

- 同步物理文件与工程元数据（`*.vcxproj/*.vcxitems/*.filters`）。
- 文件重命名后，优先核对 Include 路径与 Filter 路径是否都已更新。

### 技巧 5：把日志“最终摘要模式”做成验收标准

- 构建：`Build succeeded + 0 Warning(s) + 0 Error(s)`
- 测试：`Passed test files: X/X + Passed test cases: Y/Y`
- 这类固定模式适合脚本自动校验与 CI 质量门禁。

## 一句话总览

这三份 Guidelines 合在一起，定义了一个完整的工程动作闭环：
**如何构建、如何测试、如何维护工程文件结构**，并且都强调“统一入口、统一日志、可追踪变更”。
