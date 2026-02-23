# 执行（Execution）

- 查看 `REPO-ROOT/.github/copilot-instructions.md` 中的 `Accessing Task Documents` 与 `Accessing Script Files`，了解其中提到的 `*.md` 和 `*.ps1` 文件上下文。
- 所有 `*.md` 与 `*.ps1` 文件都应当已存在；除非明确要求，否则不要创建新文件。
  - `Copilot_Execution.md` 文件应当已经存在。
  - 如果找不到这个文件，说明你看的目录不对。
- 按照 `REPO-ROOT/.github/copilot-instructions.md` 中 `Leveraging the Knowledge Base` 的要求，在 `REPO-ROOT/.github/KnowledgeBase/Index.md` 查找本项目所需知识与文档。

## 目标与约束（Goal and Constraints）

- 你将依据 `Copilot_Execution.md` 对源码应用改动。

## Copilot_Execution.md 结构

- `# !!!EXECUTION!!!`：该文件总是以这个标题开始。
- `# UPDATES`：
  - `## UPDATE`：可能出现多次，每个小节都是我提供的更新描述原文。
- `# AFFECTED PROJECTS`
- `# EXECUTION PLAN`
- `# FIXING ATTEMPTS`

## 步骤 1：识别问题（Identify the Problem）

- 执行文档位于 `Copilot_Execution.md`。
- 在“最新聊天消息”里查找 `# Update`。
  - 忽略聊天历史里的同名标题。

### 执行计划（仅当最新消息中没有标题时）

如果“最新聊天消息”中包含任何标题，则忽略本节。
这表示我正在发起一个全新的请求。

- 将 `Copilot_Execution.md` 中的所有代码变更应用到源码。
  - 注意缩进与换行，应与目标文件既有风格一致。
- 每完成 `Copilot_Execution.md` 中一个步骤，就在该步骤标题后追加 `[DONE]` 标记，以便中断后可恢复进度。

### 同步更新源码与文档（仅当最新消息出现 `# Update` 时）

如果“最新聊天消息”里没有 `# Update`，则忽略本节。
这表示我会提出对源码的额外更新。

- 将最新聊天消息里 `# Update` 下的问题描述，原样复制到 `Copilot_Execution.md` 的 `# UPDATES` 中，新增一个 `## UPDATE` 小节。
- 按该更新修改源码。
- 同步更新文档，使之与源码保持一致。

## 步骤 2：确保代码可编译，但不要运行单元测试

- 在 `REPO-ROOT/.github/copilot-instructions.md` 的 `External Tools Environment and Context` 中查看如何访问构建脚本。
  - 严格遵循该说明，因为这个仓库不使用常规工具链。
- 每一次“构建-修复”流程都必须在子代理中执行。
  - 一次“构建-修复”包含一次 `Build Unit Test` 与 `Fix Compile Errors`。
  - 主代理应为每次尝试调用不同的子代理。
  - 主代理不得自己执行构建并读取构建结果。

### 在子代理中执行以下指令（`Build Unit Test` 与 `Fix Compile Errors`）

#### Build Unit Test

- 查看 `Copilot_Execution.md` 的 `# AFFECTED PROJECTS`，确定需要构建哪些解决方案。
- 检查是否存在警告或错误。
  - 如何检查编译结果请参考 `External Tools Environment and Context`。

#### Fix Compile Errors

- 若存在编译错误，必须全部修复：
  - 若存在编译警告，只修复由你本次改动引入的警告，不要修复其他警告。
  - 若存在编译错误，需谨慎判断问题在被调用方还是调用方；修改前先参考相似代码。
  - 每一次源码修复尝试都要记录：
    - 原始改动为什么没有生效。
    - 你接下来要做什么。
    - 你为何认为这能解决构建失败。
    - 将以上内容写入 `Copilot_Execution.md` 的 `# FIXING ATTEMPTS`，使用 `## Fixing attempt No.<attempt_number>` 小节。
- 修复完成后，退出当前子代理，并让主代理回到 `步骤 2：确保代码可编译，但不要运行单元测试`。
- 当代码可编译后：
  - **不要运行任何测试**，测试将在后续任务中完成。

## 步骤 3：验证编码风格（Verify Coding Style）

- `Copilot_Execution.md` 中的代码改动可能存在缩进和风格问题。
  - 逐条检查并确保：
    - 缩进正确，且与周边代码一致。
    - 代码风格（尤其换行）遵循周边文件惯例。
- 确保任何空行都不包含空格或制表符。
