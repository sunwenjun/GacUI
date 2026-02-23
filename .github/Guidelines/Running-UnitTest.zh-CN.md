# 运行单元测试项目

- 只能运行 `copilotExecute.ps1` 来执行单元测试项目。
- 不要自行调用可执行文件或其他脚本。

## 执行 copilotExecute.ps1

`PROJECT-NAME` 表示项目名称。

测试前，确保调试器已经停止。
如果有任何报错信息，表示调试器并未处于运行状态，这是正常且可接受的。

```
& REPO-ROOT\.github\Scripts\copilotDebug_Stop.ps1
```

然后在 `SOLUTION-ROOT\PROJECT-NAME\PROJECT-NAME.vcxproj` 中运行测试用例：

```
cd SOLUTION-ROOT
& REPO-ROOT\.github\Scripts\copilotExecute.ps1 -Executable PROJECT-NAME
```

## 确保已选中预期测试文件

测试用例分布在多个测试文件中。
在 `PROJECT-NAME\PROJECT-NAME.vcxproj.user` 中存在一个过滤器；当该过滤器生效时，你会在 `Execute.log` 里看到被过滤的测试文件被标记为 `[SKIPPED]`。
过滤器定义在这个 XPath：`/Project/PropertyGroup@Condition="'$(Configuration)|$(Platform)'=='Debug|x64'"/LocalDebuggerCommandArguments`。
只有当该文件存在，且元素中包含一个或多个 `/F:FILE-NAME.cpp`（列出要执行的测试文件）时，过滤器才会生效；未列出的文件会被跳过。
如果该元素存在但没有 `/F:FILE-NAME.cpp`，则会执行所有测试文件，不会跳过任何文件。

**重要**：

只有在你想执行的测试文件被跳过时，才可以编辑 `PROJECT-NAME\PROJECT-NAME.vcxproj.user` 以启用你的过滤器。
- 这通常发生在以下情况：
  - 新增了测试文件。
  - 重命名了测试文件。

你可以清理过滤器中与当前任务无关的文件（例如不存在的文件或完全无关的文件）。
如果某个测试文件当前任务没有直接修改，但测试主题与当前任务紧密相关，建议保留在列表中。

不要删除这个 `*.vcxproj.user` 文件。
不要自行清空过滤器（即删除所有 `/FILE-NAME.cpp`）。我之所以设置过滤器，是因为全量运行既慢又不必要。
忽略 `*.vcxproj.user` 中的 `LocalDebuggerCommandArgumentsHistory`。

## 正确读取测试结果的方法

- 唯一可信来源是单元测试进程的原始输出。
- 必须等待脚本执行结束后再读取日志文件。
  - 不需要读取脚本运行过程中的终端输出。
  - 测试耗时较长，不要着急。
  - 脚本结束后，结果会保存到 `REPO-ROOT/.github/Scripts/Execute.log`。
  - 测试期间会生成临时文件 `Execute.log.unfinished`。测试结束后该文件会自动删除。若你看到此文件，说明测试尚未完成。
- 当所有测试用例通过时，`Execute.log` 最后几行应符合如下模式；否则通常表示程序在最后显示的测试用例处崩溃：
  - "Passed test files: X/X"
  - "Passed test cases: Y/Y"
- 除非有明确要求，不要自行删除日志文件。
