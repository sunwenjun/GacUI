# 构建解决方案

- 只能运行 `copilotBuild.ps1` 来构建解决方案。
- 不要自行使用 msbuild。
- 该脚本会构建解决方案中的所有项目。

## 执行 copilotBuild.ps1

构建前，确保调试器已经停止。
如果有任何报错信息，表示调试器并未处于运行状态，这是正常且可接受的。

```
& REPO-ROOT\.github\Scripts\copilotDebug_Stop.ps1
```

然后运行以下脚本构建解决方案：

```
cd SOLUTION-ROOT
& REPO-ROOT\.github\Scripts\copilotBuild.ps1
```

## 正确读取编译结果的方法

- 唯一可信来源是编译器原始输出。
- 必须等待脚本执行结束后再读取日志文件。
  - 不需要读取脚本运行过程中的终端输出。
  - 构建耗时较长，不要着急。
  - 脚本结束后，结果会保存到 `REPO-ROOT/.github/Scripts/Build.log`。
  - 构建期间会生成临时文件 `Build.log.unfinished`。构建结束后该文件会自动删除。若你看到此文件，说明构建尚未完成。
- 构建成功时，`Build.log` 最后几行会按如下模式显示警告与错误数量：
  - "Build succeeded."
  - "0 Warning(s)"
  - "0 Error(s)"
- 除非有明确要求，不要自行删除日志文件。
