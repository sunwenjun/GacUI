# 任务（Task）

- 查看 `REPO-ROOT/.github/copilot-instructions.md` 中的 `Accessing Task Documents` 与 `Accessing Script Files`，了解其中提到的 `*.md` 和 `*.ps1` 文件上下文。
- 所有 `*.md` 与 `*.ps1` 文件都应当已存在；除非明确要求，否则不要创建新文件。

## 目标与约束（Goal and Constraints）

- 你必须确保源代码可以编译。
- 你必须确保所有测试通过。

## 步骤 1：实现需求（Implement Request）

- 按照聊天消息实现任务。

## 步骤 2：编译（Compile）

- 在 `REPO-ROOT/.github/copilot-instructions.md` 的 `External Tools Environment and Context` 中查看如何访问构建脚本。
  - 严格遵循该说明，因为这个仓库不使用常规工具链。
- 每一次“构建-修复”流程都必须在子代理中执行。
  - 一次“构建-修复”包含一次 `Build Unit Test` 与 `Fix Compile Errors`。
  - 主代理应为每次尝试调用不同的子代理。
  - 主代理不得自己执行构建并读取构建结果。

### 在子代理中执行以下指令（`Build Unit Test` 与 `Fix Compile Errors`）

#### Build Unit Test

- 查看 `REPO-ROOT/.github/Project.md`，确定需要构建哪些解决方案。
- 检查是否存在警告或错误。
  - 如何检查编译结果请参考 `REPO-ROOT/.github/copilot-instructions.md` 的 `External Tools Environment and Context`。

#### Fix Compile Errors

- 若存在编译错误，必须全部修复：
  - 若存在编译警告，只修复由你本次改动引入的警告，不要修复其他警告。
  - 若存在编译错误，需谨慎判断问题在被调用方还是调用方；修改前先参考相似代码。
- 修复完成后，退出当前子代理，并让主代理回到 `步骤 2：编译`。

## 步骤 3：运行单元测试（Run Unit Test）

- 在 `REPO-ROOT/.github/copilot-instructions.md` 的 `External Tools Environment and Context` 中查看如何访问测试与调试脚本。
  - 严格遵循该说明，因为这个仓库不使用常规工具链。
- 每一次“测试-修复”流程都必须在子代理中执行。
  - 一次“测试-修复”包含一次 `Execute Unit Test` 与 `Fix Failed Test Cases`。
  - 主代理应为每次尝试调用不同的子代理。
  - 主代理不得自己执行测试并读取测试结果。

### 在子代理中执行以下指令（`Execute Unit Test`、`Identify the Cause of Failure` 与 `Fix Failed Test Cases`）

#### Execute Unit Test

- 查看 `REPO-ROOT/.github/Project.md`，确定需要执行哪些项目。
- 运行单元测试并确认是否通过。若一切正常，输出中只会看到被执行的测试文件与测试用例。
  - 确保新增测试用例确实被执行。
  - 若测试断言失败，输出会打印 `TEST_ASSERT` 或其他宏的内容。
  - 若测试崩溃，最后一个输出的测试用例通常就是失败点。此时可在代码中加日志辅助定位。
    - 在测试用例中可使用 `TEST_PRINT`。
    - 在业务源码中可使用 `vl::console::Console::WriteLine`。在 `Vlpp` 项目中需要 `#include` `Console.h`；其他项目通常可直接使用 `Console`。
    - 日志不再需要后，应全部删除。

#### Identify the Cause of Failure

- 深入阅读相关源码，提出关于根因的假设。
- 可使用测试中的 `TEST_ASSERT` 或源码中的 `vl::console::Console::WriteLine` 辅助排查。
  - 它们可帮助确认预期代码路径是否被执行。
  - 它们可将变量值转为字符串并打印。
- 若对假设不够确信，可直接调试单元测试以获取更准确线索。
  - 按 `Debugging a Project` 启动调试器并执行 WinDBG 命令。
  - 可设置断点、逐行执行、检查变量。
  - 调试结束后必须停止调试器。
- 若已做多次猜测仍无进展，建议直接进入调试。
  - 断点对于确认执行路径与变量值很有帮助。

#### Fix Failed Test Cases

- 对源码应用修复。
- **不要删除任何测试用例。**
- 修复完成后，退出当前子代理，并让主代理回到 `步骤 2：编译`。
  - `步骤 2：编译` 与 `步骤 3：运行单元测试` 本身没有问题。若没有进展，唯一原因通常是你的改动不正确。

## 步骤 4：再次检查（Check it Again）

- 回到 `步骤 2：编译`，按全部说明再完整执行一轮。
