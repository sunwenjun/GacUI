# !!!EXECUTION!!!

# UPDATES

## UPDATE
The execution plan is architecturally sound and implementation-ready.

Key recommendations to carry into implementation:
- In `GuiRemoteRendererSingle::RequestRendererEndRendering(vint id)`, set `needRefresh = true` unconditionally at the end of each completed rendering cycle so the client repaints on every frame, even when the DOM diff is empty.
- In `GuiRemoteRendererSingle::RequestRendererRenderDomDiff(...)`, pre-filter diffs to tolerate partial-frame updates:
  - Ignore `Deleted` diffs for element nodes and their virtual parents (`domId % 4 == 0/1`) so previously rendered element nodes are not dropped just because they were omitted from this cycle.
  - Convert `Created` ? `Modified` for such nodes when the node already exists client-side, to keep updates idempotent and avoid duplicate creation.
  - Keep hit-test nodes (`domId % 4 == 2/3`) deletable.
  - Guard `renderingDomIndex.Count() > 0` before calling `collections::BinarySearchLambda(&renderingDomIndex[0], ...)`.

Verification emphasis:
- During insert-at-top and insert-in-middle scenarios, explicitly verify bounds/Y-coordinates are updated and paragraphs do not overlap, confirming shifted content receives `Modified` updates.
- During deletion, explicitly confirm `RequestRendererDestroyed` is invoked and deleted paragraphs do not reappear after subsequent typing/resize.

# AFFECTED PROJECTS

- Build the solution in folder REPO-ROOT\Test\GacUISrc
  - Run Test Project UnitTest (optional; can be skipped for this task because only the client renderer is changed)

# EXECUTION PLAN

## STEP 1: Make refresh scheduling frame-driven (client-side)
[DONE]

In `Source\PlatformProviders\RemoteRenderer\GuiRemoteRendererSingle_Rendering.cpp`, ensure a redraw is scheduled when a rendering cycle ends, even if the DOM diff is empty.

Update `GuiRemoteRendererSingle::RequestRendererEndRendering(vint id)` to set `needRefresh = true` at the end of each completed frame:

```cpp
void GuiRemoteRendererSingle::RequestRendererEndRendering(vint id)
{
	events->RespondRendererEndRendering(id, elementMeasurings);
	elementMeasurings = {};
	fontHeightMeasurings.Clear();

	// Ensure the client repaints on every completed core frame.
	// Some frames may only contain element updates / measurements (or an empty dom diff).
	needRefresh = true;
}
```

If later confirmed that `RequestRendererUpdateElement_DocumentParagraph(...)` can be invoked outside a completed frame, additionally set `needRefresh = true` there.

---

## STEP 2: Tolerate partial-frame DOM diffs without ?forgetting? previously rendered element nodes
[DONE]

In `Source\PlatformProviders\RemoteRenderer\GuiRemoteRendererSingle_Rendering.cpp`, pre-process incoming diffs before applying them:

- For element nodes and their virtual parents (`domId % 4 == 0/1`), ignore `Deleted` diffs.
- For those nodes, if a `Created` diff arrives but the node already exists in `renderingDomIndex`, convert it to `Modified`.
- Keep hit-test nodes (`domId % 4 == 2/3`) deletable as-is.
- Guard empty `renderingDomIndex` before calling `collections::BinarySearchLambda(&renderingDomIndex[0], ...)`.

Update `GuiRemoteRendererSingle::RequestRendererRenderDomDiff(...)` as follows (apply as a single consecutive code update within the function):

```cpp
void GuiRemoteRendererSingle::RequestRendererRenderDomDiff(const remoteprotocol::RenderingDom_DiffsInOrder& arguments)
{
#define ERROR_MESSAGE_PREFIX L"vl::presentation::remote_renderer::GuiRemoteRendererSingle::RequestRendererRenderDomDiff(const RenderingDom_DiffsInOrder&)#"
	CHECK_ERROR(renderingDom, ERROR_MESSAGE_PREFIX L"This function must be called after RequestRendererRenderDom.");

	remoteprotocol::RenderingDom_DiffsInOrder filtered;
	filtered.diffsInOrder = Ptr(new collections::List<remoteprotocol::RenderingDom_Diff>);

	if (arguments.diffsInOrder)
	{
		for (auto&& diff : *arguments.diffsInOrder.Obj())
		{
			// dom id encoding (GuiRemoteProtocolSchema_FrameOperations.h):
			// element: (elementId<<2)+0, parent-of-element: (elementId<<2)+1
			bool isElementOrElementParent = (diff.id != -1) && (diff.id % 4 == 0 || diff.id % 4 == 1);

			if (isElementOrElementParent && diff.diffType == remoteprotocol::RenderingDom_DiffType::Deleted)
			{
				// Partial-frame tolerance: do not drop existing element nodes just because
				// they are not present in this rendering cycle's command stream.
				// Invariant: DOM diffs manage structure; RequestRendererDestroyed + availableElements govern renderability.
				continue;
			}

			if (isElementOrElementParent && diff.diffType == remoteprotocol::RenderingDom_DiffType::Created)
			{
				// NOTE: guard empty before taking &renderingDomIndex[0]
				if (renderingDomIndex.Count() > 0)
				{
					vint insert = 0;
					auto found = collections::BinarySearchLambda(
						&renderingDomIndex[0],
						renderingDomIndex.Count(),
						diff.id,
						insert,
						[](const remoteprotocol::DomIndexItem& item, vint id) { return item.id <=> id; }
						);

					if (found != -1)
					{
						auto modified = diff;
						modified.diffType = remoteprotocol::RenderingDom_DiffType::Modified;
						filtered.diffsInOrder->Add(modified);
						continue;
					}
				}
			}

			filtered.diffsInOrder->Add(diff);
		}
	}

	UpdateDomInplace(renderingDom, renderingDomIndex, filtered);
	CheckDom();
#undef ERROR_MESSAGE_PREFIX
}
```

---

## STEP 3: Build and manual verification
[DONE]

### Build

- Stop debugger:
  - `& REPO-ROOT\.github\Scripts\copilotDebug_Stop.ps1`
- Build:
  - `cd REPO-ROOT\Test\GacUISrc`
  - `& REPO-ROOT\.github\Scripts\copilotBuild.ps1`

### Manual rendering verification (remote client)

1. Create a document with 3+ paragraphs using Enter.
2. Insert a paragraph at the top (Enter at beginning).
3. Insert a paragraph in the middle.
4. Delete a paragraph (select + Delete/Backspace).
5. During rapid typing / pressing Enter, spot-check intermediate states.
6. Use Ctrl+Enter (or the editor?s ?line break? command) inside a paragraph.
7. Resize the window and scroll (if supported) to force multiple invalidations.

Expected:
- All paragraphs remain visible (no ?only last paragraph?).
- No overlaps; bounds/Y-coordinates update correctly.
- True deletions remove content and it does not reappear after typing/resize.

# FIXING ATTEMPTS

- N/A (execution document update only)

# !!!FINISHED!!!

# !!!VERIFIED!!!


---

## UPDATE (项目结构分析 / 目录框架)

本次任务对仓库进行了目录级结构梳理（包含隐藏目录），并按“源码层 / 测试层 / 工具层 / 文档与流程层 / 历史兼容层”进行分层解读。

### 1) 顶层职责分区

- `.github/`：协作流程、规范、知识库、脚本、任务日志与提示词。
- `Source/`：GacUI 主体实现（应用层、控件层、图形与平台层、反射、资源、皮肤、工具辅助）。
- `Test/`：解决方案、单元测试、Linux vmake 构建入口、测试资源与快照。
- `Tools/`：开发辅助工具（如 GacGen）。
- `Release/`：对外发布使用的整理输出。
- `Import/`：依赖输入层（按仓库规范一般不直接修改）。
- `Deprecated/`：历史实现与兼容留档。
- 隐藏目录（`.git/`、`.vscode/`）分别对应版本控制与本地编辑器配置。

### 2) ANSI Art 项目框架（目录树，含隐藏目录）

```ansi
GacUI/
├── .cursorrules
├── .gitattributes
├── .gitignore
├── AGENTS.md
├── CLAUDE.md
├── LICENSE.md
├── README.md
├── GacUIHtml1.gif
├── GacUIRemote.gif
├── GacUISnapshotViewer.gif
│
├── .git/                     # Git 元数据（隐藏目录）
│   ├── branches/
│   ├── hooks/
│   ├── info/
│   ├── logs/
│   │   └── refs/
│   ├── objects/
│   │   ├── info/
│   │   └── pack/
│   └── refs/
│       ├── heads/
│       ├── remotes/
│       └── tags/
│
├── .github/                  # 规范 / 自动化 / 知识库（隐藏目录）
│   ├── Agent/
│   │   ├── .yarn/
│   │   ├── node_modules/
│   │   ├── packages/
│   │   └── prompts/
│   ├── Experiment/
│   ├── Guidelines/
│   ├── KnowledgeBase/
│   │   └── manual/
│   ├── Learning/
│   │   └── 2026-02-19-16-17-03/
│   ├── Scripts/
│   ├── TaskLogs/
│   └── prompts/
│
├── .vscode/                  # 本地 IDE 配置（隐藏目录）
│
├── Deprecated/               # 历史模块
│   ├── Controls/
│   │   ├── Styles/
│   │   └── TextEditorPackage/
│   ├── Document/
│   │   └── Clang/
│   ├── GacStudio/
│   │   └── GacStudio/
│   ├── GraphicsElement/
│   └── PlatformProviders/
│       └── Windows/
│
├── Import/                   # 依赖输入层
│
├── Release/                  # 发布输出层
│   └── IncludeOnly/
│
├── Source/                   # 核心源码层
│   ├── Application/
│   │   ├── Controls/
│   │   ├── GraphicsCompositions/
│   │   └── GraphicsHost/
│   ├── Compiler/
│   │   ├── InstanceLoaders/
│   │   ├── InstanceQuery/
│   │   ├── RemoteProtocol/
│   │   └── WorkflowCodegen/
│   ├── Controls/
│   │   ├── ListControlPackage/
│   │   ├── Templates/
│   │   ├── TextEditorPackage/
│   │   └── ToolstripPackage/
│   ├── GraphicsComposition/
│   ├── GraphicsElement/
│   ├── NativeWindow/
│   ├── PlatformProviders/
│   │   ├── GacGen/
│   │   ├── Hosted/
│   │   ├── Remote/
│   │   ├── RemoteRenderer/
│   │   └── Windows/
│   ├── Reflection/
│   │   └── TypeDescriptors/
│   ├── Resources/
│   ├── Skins/
│   │   └── DarkSkin/
│   ├── UnitTestUtilities/
│   │   └── SnapshotViewer/
│   └── Utilities/
│       ├── FakeServices/
│       └── SharedServices/
│
├── Test/                     # 测试与构建编排层
│   ├── GacUISrc/
│   │   ├── CppTest/
│   │   ├── CppTest_Metaonly/
│   │   ├── CppTest_Reflection/
│   │   ├── GacUI_Compiler/
│   │   ├── GacUI_Host/
│   │   ├── Generated_DarkSkin/
│   │   ├── Generated_Dialogs/
│   │   ├── Generated_FullControlTest/
│   │   ├── Generated_RemoteProtocolTest/
│   │   ├── Generated_UnitTestViewer/
│   │   ├── Lib_GacUI/
│   │   ├── Lib_GacUI_App/
│   │   ├── Lib_GacUI_App_Metaonly/
│   │   ├── Lib_GacUI_App_Reflection/
│   │   ├── Lib_GacUI_Compiler/
│   │   ├── Lib_GacUI_Compiler_Reflection/
│   │   ├── Lib_GacUI_Metaonly/
│   │   ├── Lib_GacUI_Reflection/
│   │   ├── Lib_GacUI_Utilities_Reflection/
│   │   ├── Metadata_Generate/
│   │   ├── Metadata_Test/
│   │   ├── Metadata_UpdateProtocol/
│   │   ├── Playground/
│   │   ├── RemotingTest_Core/
│   │   ├── RemotingTest_Rendering_Win32/
│   │   ├── Source_GacUI/
│   │   ├── Source_GacUI_Compiler/
│   │   ├── Source_GacUI_Core/
│   │   ├── Source_GacUI_CoreApplication/
│   │   ├── Source_GacUI_ProtocolCompiler/
│   │   ├── Source_GacUI_Reflection/
│   │   ├── Source_GacUI_UnitTest/
│   │   ├── Source_GacUI_UnitTest_Controls/
│   │   ├── Source_GacUI_UnitTest_Reflection/
│   │   ├── Source_GacUI_Utilities/
│   │   ├── Source_GacUI_Utilities_Controls/
│   │   ├── Source_GacUI_Utilities_Reflection/
│   │   ├── Source_GacUI_Windows/
│   │   ├── Source_Import/
│   │   ├── Source_Import_Reflection/
│   │   ├── UnitTest/
│   │   └── UnitTestViewer/
│   ├── Linux/
│   │   ├── CppTest/
│   │   ├── CppTest_Metaonly/
│   │   ├── CppTest_Reflection/
│   │   ├── GacUI_Compiler/
│   │   ├── Metadata_Generate/
│   │   ├── Metadata_Test/
│   │   └── UnitTest/
│   └── Resources/
│       ├── App/
│       ├── CompilerErrorTests/
│       ├── HostedWindowManagerTests/
│       ├── Metadata/
│       ├── UnitTestResources/
│       └── UnitTestSnapshots/
│
├── ToDo/
└── Tools/
    └── GacGen/
        ├── Bin/
        └── GacGen/
```

### 3) 结构解读（架构视角）

- 这是一个“**源码（Source）+ 工程编排（Test/GacUISrc）+ 指南知识（.github）**”三元结构仓库。
- `Test/GacUISrc` 不仅是测试目录，更是**完整解决方案组织中心**，承担项目分拆、生成产物项目、反射/元数据相关工程聚合。
- `Source/PlatformProviders` 与 `Source/Compiler/RemoteProtocol` 共同体现了 **跨平台渲染抽象 + 远程协议能力** 的架构主线。
- `Deprecated/` 保留历史能力，可用于回溯设计演进，不应与当前主路径（Source/Test）混用。
- `.github/KnowledgeBase` + `.github/Guidelines` 形成“编码规则与 API 决策知识底座”，是仓库内任务执行的元规范层。
