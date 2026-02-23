# `.github/Scripts/copilotExecute.ps1` 中文翻译

## 脚本用途
统一执行入口：
- 自动定位最新产出的目标可执行文件
- 自动读取 `.vcxproj.user` 中调试参数
- 执行程序并保存日志

## 参数
- `-Executable <name>`：必填，**不带 `.exe` 后缀**。

## 主要逻辑（逐段翻译）
1. **参数检查**：如果传入值已包含 `.exe`，直接报错。
2. **加载共享脚本并清理日志**：删除 `Execute.log` 与 `Execute.log.unfinished`。
3. **定位 solution**：通过 `GetSolutionDir` 获取根目录。
4. **选择可执行文件**：
   - 调用 `GetLatestModifiedExecutable(solutionFolder, executableName)`
   - 在常见输出目录（Debug/Release + Win32/x64）中找同名 `.exe`
   - 选择“最后修改时间最新”的那一个
5. **读取调试参数**：
   - 调用 `GetDebugArgs(...)`
   - 从 `<solution>/<project>/<project>.vcxproj.user` 中匹配配置平台条件
   - 读取 `LocalDebuggerCommandArguments`
6. **执行与日志记录**：
   - 拼装命令：`"<exePath>" /C <debugArgs>`
   - 通过 `cmd.exe /S /C` 执行，输出经 `Tee-Object` 写日志
   - 将 unfinished 日志重命名为正式 `Execute.log`
7. **退出码透传**：`exit $LASTEXITCODE`
