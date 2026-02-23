# !!!执行文档（中文翻译）!!!

# 更新

## 结论

执行按计划完成，核心修复已落地：

1. 帧结束统一触发刷新（`needRefresh = true`）。
2. 对局部帧 DOM diff 做过滤容错，避免历史段落误删。

---

## STEP 1：按帧触发刷新

状态：`[DONE]`

在 `Source\PlatformProviders\RemoteRenderer\GuiRemoteRendererSingle_Rendering.cpp` 的
`GuiRemoteRendererSingle::RequestRendererEndRendering(vint id)` 末尾设置 `needRefresh = true`。

作用：即使该帧 DOM diff 为空，或只有测量/元素更新，也会触发重绘。

补充说明：若未来确认 `RequestRendererUpdateElement_DocumentParagraph(...)` 存在帧外调用，再在该路径追加 `needRefresh = true`。

---

## STEP 2：局部帧 DOM diff 容错

状态：`[DONE]`

在 `GuiRemoteRendererSingle::RequestRendererRenderDomDiff(...)` 中，先过滤再应用：

- 对元素节点及其虚拟父节点（`domId % 4 == 0/1`）的 `Deleted`：忽略。
- 对同类节点，若收到 `Created` 且本地已存在：改写为 `Modified`。
- 命中测试节点（`domId % 4 == 2/3`）保留可删除。
- `renderingDomIndex` 为空时避免对 `&renderingDomIndex[0]` 取址。

设计不变量说明：

- DOM diff 负责结构同步；
- 元素是否可绘制由 `RequestRendererDestroyed` 与 `availableElements` 判定；
- 因此不会因为“忽略某些删除 diff”而产生可见僵尸段落。

---

## STEP 3：构建与人工验证

状态：`[DONE]`

### 构建流程（文档中给出的标准步骤）

- 停止调试器：`copilotDebug_Stop.ps1`
- 构建：进入 `Test/GacUISrc` 后运行 `copilotBuild.ps1`

### 手工验证场景

1. Enter 创建 3+ 段。
2. 顶部插入段落。
3. 中部插入段落。
4. 删除段落（Delete/Backspace）。
5. 快速输入与连续回车期间抽样观察中间态。
6. 段内 Ctrl+Enter。
7. resize/scroll 触发多次重绘。

### 验收结果

- 多段落稳定可见，不再“只显示最后一段”。
- 段落位置更新正常，无重叠。
- 删除有效且不会在后续操作中反复出现。

# 修复尝试

- `N/A`（本执行文档以结果汇总为主）

# !!!已完成!!!

# !!!已验证!!!
