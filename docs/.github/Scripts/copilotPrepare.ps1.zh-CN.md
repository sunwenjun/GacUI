# `.github/Scripts/copilotPrepare.ps1` 中文翻译

## 脚本用途
准备 Copilot 工作区任务文档；支持：
- 备份已有任务文档到 `Learning/<timestamp>/`
- 仅查询最早一次学习快照目录
- 重建/重置任务日志模板文件

## 参数
- `-Backup`：仅执行备份并删除部分任务文档，不重建模板。
- `-Earliest`：输出 `Learning` 下按目录名排序后的最早目录路径。

> 约束：`-Backup` 与 `-Earliest` 最多只能启用一个。

## 主要逻辑（逐段翻译）
1. **参数互斥检查**：如果 `-Backup` 和 `-Earliest` 同时为真，抛异常。
2. **`-Earliest` 模式**：
   - 定位 `..\Learning`
   - 取目录名排序后的第一个目录
   - 若不存在目录则报错
   - 输出目录完整路径并退出
3. **默认任务文档模板定义**：
   - 覆盖写入：`Copilot_Planning.md` / `Copilot_Execution.md` / `Copilot_Task.md`
   - 仅首次创建：`Copilot_Scrum.md` / `Copilot_KB.md`
4. **备份清单构建**：
   - 将可覆盖的任务文档加入待备份列表（如果文件存在）
   - 额外检查 `Copilot_Execution_Finding.md` 是否存在并加入备份
5. **执行备份**：
   - 创建 `Learning/yyyy-MM-dd-HH-mm-ss/`
   - 把待备份文件复制进去
6. **清理临时/诊断文档**：
   - 删除 `TaskLogs/Copilot_Execution_Finding.md`（若存在）
7. **按模式处理任务文档**：
   - `-Backup`：删除 `filesToOverride` 中已有文件
   - 非 `-Backup`：
     - 覆盖写入 `filesToOverride`
     - 仅在不存在时创建 `filesToCreate`
8. **结束输出**：打印“Copilot preparation completed.”
