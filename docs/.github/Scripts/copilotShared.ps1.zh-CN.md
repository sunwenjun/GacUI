# `.github/Scripts/copilotShared.ps1` 中文翻译

## 脚本用途
为构建/执行/调试脚本提供公共方法：
- 查找 `cdb.exe`
- 自动定位 solution 目录
- 在标准输出目录中选择最新可执行文件
- 从 `.vcxproj.user` 提取调试参数

## 公共函数翻译

### 1) `GetCDBPath`
- 从环境变量 `CDBPATH` 读取 `cdb.exe` 路径。
- 若未设置：抛异常并提示示例路径和 WDK 安装说明。
- 返回 `CDBPATH`。

### 2) `GetSolutionDir`
- 从当前目录开始，逐级向父目录搜索 `*.sln`。
- 找到后打印“Found solution folder”。
- 若一直未找到则抛异常。
- 返回 solution 所在目录。

### 3) `GetLatestModifiedExecutable($solutionFolder, $executableName)`
- 预定义 4 个候选路径：
  - `Debug|Win32`
  - `Release|Win32`
  - `Debug|x64`
  - `Release|x64`
- 收集实际存在的 `.exe` 及其修改时间。
- 若都不存在则抛异常。
- 按修改时间降序排序并返回最新文件对象（含路径、配置、时间）。

### 4) `GetDebugArgs($solutionFolder, $latestFile, $executable)`
- 目标文件：`<solution>/<project>/<project>.vcxproj.user`
- 若存在：
  - 读取 XML
  - 按 `$latestFile.Configuration` 匹配 `PropertyGroup/@Condition`
  - 若存在 `LocalDebuggerCommandArguments` 则返回该参数
- 若解析失败或未找到：打印警告/提示并返回空字符串。
