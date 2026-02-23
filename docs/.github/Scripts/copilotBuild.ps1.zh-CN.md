# `.github/Scripts/copilotBuild.ps1` 中文翻译

## 脚本用途
统一构建入口：
- 清理旧构建日志
- 自动定位 solution
- 通过 `VsDevCmd.bat` 初始化 VS 构建环境
- 调用 `MSBUILD` 编译并把输出同时写到控制台和日志

## 主要逻辑（逐段翻译）
1. **加载共享脚本**：引入 `copilotShared.ps1` 的公共函数。
2. **日志清理**：删除 `Build.log` 与 `Build.log.unfinished`。
3. **定位 solution**：
   - 用 `GetSolutionDir` 向上查找最近含 `*.sln` 的目录
   - 选取该目录下第一个 `.sln` 文件
4. **环境变量校验**：
   - 从 `VLPP_VSDEVCMD_PATH` 读取 `VsDevCmd.bat` 路径
   - 若缺失，抛出带示例路径的错误
5. **构建参数拼装**：
   - `Configuration = Debug`
   - `Platform = x64`
   - 使用 `/m:8` 并发构建
   - 追加 `$rebuildControl`（由外部上下文注入）
6. **执行与日志记录**：
   - 通过 `cmd /c` 运行“先调用 VsDevCmd，再执行 MSBUILD”命令链
   - 标准输出/错误经 `Tee-Object` 同时写到控制台与 `Build.log.unfinished`
   - 完成后将 unfinished 重命名为正式 `Build.log`
7. **退出码透传**：`exit $LASTEXITCODE`
