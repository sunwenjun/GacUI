# !!!任务文档（中文翻译）!!!

# 问题描述
## 任务 No.2：修复多段落文档渲染（客户端）

修复远程渲染富文本文档时“只能显示最后一段”的问题。该问题主要发生在客户端渲染器 `GuiRemoteRendererSingle`，而不是远程协议核心层。核心协议已有较充分单元测试覆盖，现象更像是客户端在增量更新中丢失/覆盖了之前创建的段落元素。

### 需要完成的工作

- 从编辑器语义出发复现与定位：
  - `Ctrl+Enter` 往往是在同一段落内插入换行（通常仍是同一个 `RendererType::DocumentParagraph` 元素 id）。
  - `Enter` 往往会创建新段落（DOM 中出现多个段落元素 id）。
  - 因此重点排查：元素调度/批处理/重绘与逐段更新逻辑。
- 验证元素 id 正确性与是否冲突：
  - 确认核心层在按 Enter 新建段落时会发出不同 element id。
  - 确认客户端不会在 `availableElements` 中把不同段落写到同一 key。
- 澄清问题属于“组合树生命周期”还是“渲染遍历”：
  - 确认段落包装节点被正确创建并插入 composition / 可视树。
  - 确认渲染遍历会走到所有段落节点，而不是只走最新更新的段落。
- 排查共享全局段落状态：
  - 保证每个段落 element id 对应独立 wrapper 与独立 `IGuiGraphicsParagraph`。
- 校验每段落的渲染目标绑定/注册行为：
  - `IGuiGraphicsParagraph` 与渲染目标绑定，需确保注册/反注册不会误删旧段落。
  - 继续使用 `id == -1` 作为“未注册/不可用”的唯一判据。
- 调查段落更新后是否一定触发重绘：
  - 在 `GuiRemoteRendererSingle_Rendering.cpp` 中，`needRefresh` 主要由 `CheckDom()`（DOM变化）与 `Paint()`（OS重绘）驱动。
  - `RequestRendererUpdateElement_*` 本身默认不会触发刷新，需要确认帧边界时机是否可靠。
- 保证段落更新与帧边界都会触发重绘：
  - 建议在 `RequestRendererEndRendering` 设置 `needRefresh = true`。
  - 必要时在 `RequestRendererUpdateElement_DocumentParagraph` 也设置，覆盖“只有元素更新、无 DOM 结构变化”的场景。
- 参考 mock/spec 实现进行行为对齐：
  - 参考 `Source\UnitTestUtilities\GuiUnitTestProtocol_Rendering_Document.cpp` 的元素生命周期和多段落语义。
- 使用标准构建脚本验证编译；本任务可跳过单元测试：
  - 后续任务文档建议在 `## AFFECTED PROJECTS` 标注：若未改核心协议，可不跑 UT。

### 背景理由

- `Ctrl+Enter` 与 `Enter` 行为差异说明触发条件很可能是“单段 vs 多段”，高概率问题点是：
  - 重绘调度（仅最后一次更新触发刷新），或
  - DOM 遍历/裁剪（节点存在但只渲染了一段）。
- 当前客户端 `needRefresh` 依赖 DOM 变化与 OS Paint；若核心发送的是元素更新而无 DOM diff，客户端可能漏刷。
- 核心协议已有 UT 覆盖，而 `GuiRemoteRendererSingle` 端到端自动化较难，因此应优先强化客户端刷新/组合行为鲁棒性，不改协议语义。

# 更新记录

- 任务推进中围绕两条主线展开：
  1. 以“帧结束”作为稳定刷新时机；
  2. 对“局部帧 DOM diff”做容错，避免把历史段落当成删除。

# 洞察与推理（摘要）

## 当前渲染管线（核心 → 客户端）

- 核心每帧发送：`BeginRendering` → 多个 `RenderElement/Boundary` → `EndRendering`。
- DOM diff 转换器把上述命令转换为：首帧全量 `RenderDom`，后续 `RenderDomDiff`。
- 客户端 `GuiRemoteRendererSingle`：保存 DOM / 应用 diff；由定时器驱动 `StartRendering -> Render(dom) -> StopRendering -> RedrawContent()`。

## 现象含义

- `Ctrl+Enter` 双行可见：说明单段落内换行路径基本正常。
- `Enter` 多段只剩最后一段：说明多段场景下历史段落节点可能被 diff 流误删，或刷新节奏不完整。

## 关键修复方向

- 帧驱动刷新：在 `RequestRendererEndRendering` 统一置 `needRefresh = true`。
- DOM diff 容错：
  - 对 `domId % 4 == 0/1`（元素节点及虚拟父节点）的 `Deleted` 做忽略；
  - 若同类节点收到 `Created` 但本地已存在，改写为 `Modified`；
  - `domId % 4 == 2/3`（命中测试节点）仍允许删除；
  - 对空 `renderingDomIndex` 加保护，避免 `BinarySearchLambda(&renderingDomIndex[0], ...)` 未定义行为。

# 影响项目

- 构建目录：`REPO-ROOT/Test/GacUISrc`
- 单元测试：本任务可选（仅改客户端渲染器时可跳过）。

# 执行计划（摘要）

1. 在 `RequestRendererEndRendering` 增加帧末刷新。
2. 在 `RequestRendererRenderDomDiff` 引入局部帧 diff 过滤策略。
3. 完成构建与手工验证（多段新增、顶部/中部插入、删除、快速输入、窗口 resize/scroll 等）。

# !!!结束（中文翻译版）!!!
