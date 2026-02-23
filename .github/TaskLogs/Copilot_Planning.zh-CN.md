# !!!规划文档（中文翻译）!!!

# 更新

## 更新说明

执行计划总体可行且最小化，但需要补充强制约束与安全保护：

- 在 `GuiRemoteRendererSingle::RequestRendererEndRendering(...)` 设置 `needRefresh = true`，确保每个已完成帧都会重绘，即使该帧 DOM diff 为空或仅有测量/元素更新。
- 若实际流程中 `RequestRendererUpdateElement_DocumentParagraph(...)` 可能脱离 DOM diff 独立触发，则也必须设置 `needRefresh = true`；若不加需明确解释安全性。
- 在 `RequestRendererRenderDomDiff(...)` 的 diff 过滤中：
  - 忽略元素节点及其虚拟父节点（`domId % 4 == 0/1`）的 `Deleted`，以容忍局部帧流。
  - 若该类节点收到 `Created` 但客户端已存在，转换为 `Modified`。
  - 关键：调用 `collections::BinarySearchLambda(&renderingDomIndex[0], ...)` 前必须检查 `renderingDomIndex` 非空。
- 命中测试节点（`domId % 4 == 2/3`）保持可删除，避免命中树陈旧和无限增长。
- 在过滤代码注释中明确生命周期不变量：
  - DOM diff 管结构；
  - `RequestRendererDestroyed` 与 `Render()` 中 `availableElements` 判定决定元素是否真正可渲染，防止“僵尸渲染”。
- 验证场景除多段插入/换行/resize 外，必须加“删除段落”与“快速输入中间态”检查。

# 影响项目

- 构建目录：`REPO-ROOT/Test/GacUISrc`
- 单元测试：可选（本任务仅改客户端渲染器）

# 执行计划

## STEP 1：将刷新调度改为“按帧驱动”

### 变更

在 `Source\PlatformProviders\RemoteRenderer\GuiRemoteRendererSingle_Rendering.cpp` 中，确保渲染帧结束时总会请求重绘，即使 DOM diff 为空。

核心修改：

- 在 `RequestRendererEndRendering(vint id)` 末尾设置 `needRefresh = true`。

可选补充（仅在确认存在帧外段落更新时启用）：

- 在 `RequestRendererUpdateElement_DocumentParagraph(...)` 也设置 `needRefresh = true`。

### 原因

当前客户端主要依赖 `CheckDom()` 或 `Paint()` 驱动刷新；当某些帧只有元素更新而无 DOM diff 时，可能出现漏刷，导致内容陈旧或看似“被清空”。

---

## STEP 2：容忍局部帧 DOM diff，避免“遗忘”历史段落节点

### 观察

- 核心端 DOM diff 由当帧命令流构建。
- 客户端会以最新 `renderingDom` 进行整窗重绘。
- 若某帧为裁剪后的局部更新，DOM 可能未包含历史段落。
- `DiffDom(...)` 可能发出 `Deleted`，客户端直接应用后会把旧段落从本地 DOM 删掉，导致后续不再渲染。

### 变更

在 `RequestRendererRenderDomDiff(...)` 应用前做预处理：

1. 对 `domId % 4 == 0/1` 节点的 `Deleted`：忽略。
2. 对同类节点的 `Created`：若本地已存在，改为 `Modified`。
3. `domId % 4 == 2/3` 节点保持删除逻辑不变。
4. 所有二分查询前加空数组保护。

### 原因

- 能保证局部帧不会误删历史段落。
- 删除仍由 `RequestRendererDestroyed` + `availableElements` 兜底，不会留下可见“僵尸元素”。

---

## STEP 3：构建与手工验证

### 构建

- 停止调试器（如在 Windows 环境脚本流中）：`copilotDebug_Stop.ps1`
- 在 `Test/GacUISrc` 执行构建脚本：`copilotBuild.ps1`

### 手工验证要点

1. 使用 Enter 创建 3+ 段。
2. 在顶部插入新段。
3. 在中间插入新段。
4. 删除一个段落（选中 + Delete/Backspace）。
5. 快速输入 / 连续回车时观察中间状态。
6. 段内使用 Ctrl+Enter。
7. 通过窗口 resize/scroll 触发多次失效重绘。

### 期望

- 所有段落持续可见，不会只剩最后一段。
- 不重叠，Y 坐标随插入/删除正确更新。
- 真删除后不会在后续输入或 resize 后“复活”。

# !!!结束（中文翻译版）!!!
