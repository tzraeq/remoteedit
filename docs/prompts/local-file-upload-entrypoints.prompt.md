# 本地文件上传入口增强：VS Code 文件树拖放与右键上传

## 业务背景与目标

用户在左侧 VS Code 本地文件树选择文件，右侧 Remote Edit Webview 已打开远程连接和目录，希望通过以下两个入口直接上传：

1. 将本地文件从 VS Code 文件树拖入 Remote Edit Webview 的文件列表。
2. 在本地文件上右键，显示 `Upload to Remote Edit`，上传到 Webview 当前连接的当前目录。

用户已确认把这两个需求写入 prompt。本轮只整理方案，不实现、不生成 checklist。编辑器 root 标签设置属于独立需求，保留在 `docs/prompts/remote-editor-connection-label.prompt.md`，其可行性阻塞不属于本文两个上传入口的前置条件。

成功结果：两个入口都能使用现有上传队列和冲突处理，不再要求用户重新通过 Upload 文件选择框选择同一批文件；上传目标与操作当时 Webview 中的连接和目录一致。

## 已核实的现状与可行性

- Webview 已有拖放上传：入口只识别浏览器 `Files` 类型，从 `DataTransfer.items/files` 取得文件对象；已有按本地路径上传和按内容分块暂存两条链路，没有解析 VS Code 内部资源 URI。[REF-001]
- VS Code 1.90.0 本地文件树拖动会提供 `CodeFiles` 本地路径数组；文件资源还会出现在 `ResourceURLs`、`application/vnd.code.uri-list` 等数据中。标准 `text/uri-list` 在该版本只写入首个文件 URI。[REF-002]
- `CodeFiles` 包含选中的本地文件和目录；`ResourceURLs` 以及此文件树场景的内部 URI 列表会过滤目录。因此，支持文件夹拖入时不能仅适配 URI 列表。[REF-002]
- 扩展已有按本地 URI 收集文件、递归展开目录、加入上传队列、处理冲突和显示进度的能力。[REF-003]
- 扩展端已经保存活动连接和各连接当前路径，也管理 Webview 的创建、关闭。当前路径在成功列目录后更新；但另有连接初始化和导航前设置路径的调用，所以不能把任意回退路径直接当成用户已经看到的有效上传目录。[REF-004]
- 原生 Explorer 菜单可通过 `explorer/context` 注册；项目已有 `setContext` 控制其他菜单状态的用法。Webview 提供 `visible` 和视图状态事件，不需要依赖焦点或屏幕几何位置。[REF-005]

**可行性判断：源码和公开 API 层面，两项均可实现。** 入口数据、连接/目录状态和上传执行链路可以贯通，不需要实验性 API。内部拖放 MIME 属于版本相关格式，应有明确适配及回退；实际跨 Webview 的拖放和菜单生命周期尚未做端到端验证，需要在实现验收中覆盖，不能写成已实测通过。

## 范围与用户行为

来源是当前扩展宿主可访问的本地文件系统资源：`file:` URI 或 VS Code 文件树 `CodeFiles` 提供的本地路径。默认覆盖单个文件、多选文件和文件夹，目录递归沿用已有上传能力。

目标是 Remote Edit 的远程目录。两个入口都使用复制/上传语义，完成后本地文件保留。

### VS Code 文件树拖入 Webview

- Webview 当前连接已连接，文件列表处于可以接受上传的状态时，识别 VS Code 内部文件资源拖放。
- 拖到远程目录行，上传到该目录；拖到文件行或文件列表空白区域，上传到当前目录，沿用现有目标解析。目录悬停展开行为也沿用现状。[REF-006]
- 在落下时固定目标连接和目录，然后解析来源并入队；后续切换连接、进入其他目录不改变这批任务的目标。
- 本地资源只表示上传来源，不能被识别为 Remote Edit 的远程条目移动。
- 从系统文件管理器拖入的现有 `Files` 路径仍可使用；内部资源适配与已有文件对象、内容分块处理衔接，同一拖放不能重复入队。

### 本地文件右键上传

在 VS Code Explorer 的文件/目录右键菜单中新增 `Upload to Remote Edit`。显示条件为：

- 右键资源属于本地 `file:` 资源。
- Remote Edit Webview 实际存在且可见。
- Webview 活动连接有效且已连接，存在与当前展示状态对应、可用于上传的远程目录。

Webview 的可见状态与是否获得焦点分开判断：用户在左侧右键时，右侧 Webview 可以没有焦点。这里的“右侧”是使用场景，不要求 Webview 固定在某一列。[REF-005]

点击菜单后：

1. 取得右键文件和 VS Code 提供的多选资源；右键资源在多选集合中时上传整组选中项，否则使用右键资源。
2. 重新检查有效上传目标，并立即取得 `{ connectionId, targetDirectory }` 快照。菜单打开后连接已断开、Webview 已关闭或目标不可用时，反馈明确原因，不把文件上传到另一个连接或默认 `/`。
3. 将选中的本地资源直接送入共用上传入口，不再弹出文件选择框；已有同名冲突处理继续生效。
4. 队列和传输记录使用已固定的目标；用户随后改变当前连接或目录不会改写任务目的地。

Webview 隐藏、关闭或活动连接断开时，菜单不显示。目录正在切换且目标未确认时暂不可用；成功后恢复。目录加载失败时，按仍然展示的已确认目录判断，不把失败的新路径当成有效目标。

## 方案与关键契约

### 内部拖放数据适配

拟议优先顺序如下；类型识别使用大小写兼容的匹配，payload 按各自格式解析：

| 优先级 | 数据类型 | 格式与用途 |
| --- | --- | --- |
| 1 | `CodeFiles` | JSON 本地路径数组；对本地文件树最完整，覆盖文件和目录 |
| 2 | `application/vnd.code.uri-list` | 完整 URI 列表；覆盖此类型承载的全部文件 |
| 3 | `ResourceURLs` | JSON URI 字符串数组 |
| 4 | `text/uri-list` | 标准 URI 列表回退；不把该类型视为 VS Code 多选来源的完整列表 |

从可用且有效的最完整数据源取得一次资源集合，再做去重；不能把同一拖放的四种表示当成四份上传，也不能优先读取标准单项 URI 后漏掉其他文件。

dragenter/dragover 阶段按类型识别并允许合适的落点；drop 阶段才读取完整 payload。沿用现有拖放反馈，处理 `preventDefault`，使落点执行上传而非触发默认打开资源的行为。

Webview 将资源描述交给扩展端，URI 转本地路径统一在扩展端通过 `vscode.Uri.parse(...).fsPath` 完成；`CodeFiles` 则作为本地路径解析。名称包含中文、空格、`#`、`%` 时不手工截断或二次解码。

只把受支持的本地文件系统来源交给本地上传收集器；非 `file:` URI、畸形资源和已不存在的来源应反馈原因，不能直接把任意 URI 的 path 当成真实本地路径。URI 来源上传的是磁盘内容，沿用现有 Upload 行为，不自动保存编辑器里的未保存修改。

### 共用上传入口

拟提取或补充一个接收“已选本地资源 + 已确定远程目标”的扩展端上传入口，两个新入口共同调用，复用现有上传任务模型与执行链路。具体内部函数命名不属于业务契约。

关键输入为：

```text
localUris: 本地 file URI 集合（去重后）
connectionId: 操作触发时固定的连接 ID
targetDirectory: 操作触发时固定的远程目标目录
source: 沿用现有传输来源字段的合法值
```

资源相同只上传一次；同时选择目录及其后代时，由目录递归覆盖后代，避免重复提交同一来源。不同本地目录下同名文件的远程目标冲突，继续使用已有冲突策略。

优先复用 `collectUploadTransferItems` → `collectUploadPath` 的 URI/目录递归路径。现有 `collectDroppedUploadTransferItems` 中，`kind: directory` 表示已展开清单中的目录节点，会直接加入目录项；不能只给它一个顶层 directory 节点就假定它会上传全部子文件。[REF-003]

传输开始和执行时保持现有连接检查、取消、冲突与失败反馈。队列执行闭包必须使用任务自己的连接和目录，不能执行时重新读取 Webview 的当前目标。

### 菜单与 Webview 状态

拟新增命令 `remoteedit.uploadToCurrentWebviewDirectory`，固定标题 `Upload to Remote Edit`，注册在 `explorer/context`。

拟新增布尔 context key `remoteedit.canUploadToWebview`，菜单条件为：

```text
resourceScheme == file && remoteedit.canUploadToWebview
```

由 `RemoteEditPanel` 提供扩展内部可读取的目标快照/可用状态，汇总实际 panel 可见性、当前活动连接及已确认目录。现有静态实例可能仅用于后台传输，关闭 Webview 后连接也可能仍保留，不能只凭实例或连接存在就显示菜单。[REF-004]

在 Webview 创建/关闭/可见性变化、切换连接、连接断开、目录导航状态变化时更新 context key。命令执行时再次核对同一状态，context key 仅用于菜单展示，不承担上传目标的数据来源。

菜单标题保持固定，目标信息沿用传输队列中的连接和目的目录显示。无需为此新增持久化连接字段或设置。

## 验收标准

以下是拟议功能验收条件，尚未实施：

1. 将 VS Code 本地文件树中的单个文件拖入 Webview 文件列表，文件加入上传队列，目标为当前连接/目录；从两个来源入口进入的任务均不再弹出文件选择框。
2. 一次拖入多个文件，每个文件只入队一次，不因标准 `text/uri-list` 只含首项而漏传，也不因多种 MIME 同时存在而重复上传。
3. 拖入包含文件和子目录的文件夹，完整递归上传；同时选中目录及其后代时不重复上传。覆盖中文、空格及 URI 保留字符的本地路径。
4. 拖到远程目录行使用该目录；拖到文件行或文件列表空白使用当前目录；已有系统文件管理器拖入和远程条目移动行为正常。
5. 右侧 Webview 可见且具有有效上传目标时，本地文件/目录右键出现菜单；左侧获得焦点不影响显示。Webview 隐藏、关闭、无有效连接或没有已确认目标时菜单不显示。
6. 多选本地文件后右键上传使用该多选集合；右键未被选中的文件时使用该文件。文件夹通过同一递归收集器处理。
7. 点击菜单或落下文件后切换 Webview 连接/目录，已入队任务仍上传到原快照目标；菜单打开后目标失效时，执行命令给出明确反馈，不回退到其他连接或默认根目录。
8. 目录加载中及失败后，菜单不把尚未成功展示的新路径当作上传目标；原有效目录仍在展示时按实际状态恢复可用性。
9. 上传遇到同名项、取消或来源读取失败时，沿用已有队列/冲突/反馈规则；本地原文件保留。URI 来源文件有未保存编辑内容时，上传磁盘版本且不触发自动保存。
10. 验证内部 MIME 解析、URI 转换、来源去重、目标快照与菜单生命周期；在支持版本的真实 VS Code 中验证 Explorer → Webview 原生拖放，不以合成 DataTransfer 解析测试代替真实跨视图验收。

## 默认决策与执行假设

- 两个入口默认都覆盖文件、多选与目录。依据：现有 Upload 已支持这些来源，用户目标是直接使用文件树选择。[REF-003]
- 默认按磁盘内容上传，保留现有符号链接跳过规则。依据：已有上传通过本地文件系统读取，目录递归显式跳过符号链接。[REF-003]
- 右键菜单以 Webview 可见为条件；已有后台连接不足以构成目标。依据：用户明确要求右侧 Webview 当前打开的连接和目录，以及 panel 与后台实例分离的现状。[REF-004]
- 共用目标在触发操作时固定。依据：现有队列已通过闭包持有连接、目标目录和本地资源，避免异步期间改写目的地。[REF-003]
- MIME 采用完整本地路径数组优先、完整 URI 列表其次、标准 URI 回退。依据：VS Code 1.90.0 对目录及标准 URI 列表的实际生成规则。[REF-002]
- 真实拖放和 context key 刷新需在执行时复核，属于“不阻塞，仅执行时复核”。如果目标编辑器版本实际未向 Webview 暴露任一受支持数据格式，应带该证据回到 prepare 调整适配，不能默认为单文件成功即完成多选验收。

## 需用户确认的问题

无阻塞性业务问题。用户已确认两个入口都写入 prompt；上述具体命名、数据优先级与状态规则为基于现有代码形成的默认方案，供 review。

## 参考资源

本地路径相对 `E:/workspace/vscode/remoteedit`；行号核实于 2026-09-19。外部源码固定为 VS Code 1.90.0，其他版本通过实际验收复核。

- REF-001
  - 来源：`src/panel/webview/scripts/DragDropUpload.ts`；`src/panel/handlers/fileActionMessageHandlers.ts`。
  - 用途：定位现有 Webview 拖放入口、数据处理及消息路由。
  - 参考范围：DragDropUpload 5–16 行（Files 与本地路径识别）、168–189 行（收集文件对象）、294–404 行（拖放事件及上传消息）；fileActionMessageHandlers 39–55 行（上传消息分发）。
  - 稳定锚点：`isLocalFileDrag`、`collectDroppedFiles`、`handleDragDropUploadDrop`、`RequestDroppedUploadEntries` 分支。
  - 原文事实：识别 `Files`，遍历文件对象和目录 entry，发送本地路径或内容分块上传消息；未解析内部 URI 列表。
  - 推导判断：需要补充内部资源适配，已有系统拖放路径可保留。
  - 适用范围与限制：Webview 拖放上传，不能据此宣称已支持 VS Code 内部资源拖放。
  - 核实状态：已核实。

- REF-002
  - 来源：https://github.com/microsoft/vscode/blob/1.90.0/src/vs/workbench/contrib/files/browser/views/explorerViewer.ts ；https://github.com/microsoft/vscode/blob/1.90.0/src/vs/workbench/browser/dnd.ts ；https://github.com/microsoft/vscode/blob/1.90.0/src/vs/base/browser/dnd.ts ；https://github.com/microsoft/vscode/blob/1.90.0/src/vs/platform/dnd/browser/dnd.ts 。
  - 用途：确认文件树内部拖放的数据格式及多选、目录差异。
  - 参考范围：explorerViewer 1182–1195 行；workbench/browser/dnd 222–246 行、255–276 行、331–358 行；base/browser/dnd 59–81 行；platform/dnd/browser/dnd 37–39 行。
  - 稳定锚点：`FileDragAndDrop.onDragStart`、`fillEditorsDragData`、`DataTransfers.RESOURCES`、`DataTransfers.INTERNAL_URI_LIST`、`CodeDataTransfers.FILES`。
  - 原文事实：本地文件树写入 `CodeFiles` 路径数组，未过滤目录；ResourceURLs 和拖动编辑器 URI 数据过滤文件树目录；内部 URI 列表含全部文件，标准 URI 列表仅首项；对应类型分别为 `CodeFiles`、`ResourceURLs`、`application/vnd.code.uri-list`。
  - 推导判断：目录应优先使用完整本地路径数组，多文件不能只读取标准 URI 类型。
  - 适用范围与限制：VS Code 1.90.0 数据生成证据；实际进入扩展 Webview 的效果仍需端到端验收。
  - 核实状态：已核实。

- REF-003
  - 来源：`src/panel/RemoteEditPanel.ts`。
  - 用途：确认共用上传队列、本地 URI 递归收集和目标固定契约。
  - 参考范围：2830–2886 行（选择框/拖放入队）；3062–3074 行（连接核对）；3076–3133 行（收集与冲突处理）；3426–3553 行（URI 收集、已展开清单处理、目录递归与符号链接规则）。
  - 稳定锚点：`requestUploadEntries`、`requestDroppedUploadEntries`、`requireTransferConnectionId`、`runUploadTransfer`、`collectUploadTransferItems`、`collectDroppedUploadTransferItems`、`collectUploadPath`。
  - 原文事实：上传任务持有 connectionId、targetDirectory、selectedUris；URI 收集使用 fsPath，目录递归走 collectUploadPath；已展开目录节点直接入清单；符号链接跳过。
  - 推导判断：新增入口可共用现有传输执行，目录不能误接成只创建空目录的清单节点。
  - 适用范围与限制：本地扩展宿主可访问的文件系统来源，不代表任意 URI 都可用 fsPath 读取。
  - 核实状态：已核实。

- REF-004
  - 来源：`src/panel/RemoteEditPanel.ts`；`src/panel/PanelState.ts`。
  - 用途：确认 Webview 生命周期、活动连接及有效目录来源。
  - 参考范围：RemoteEditPanel 64–72 行（panel/state）、432–451 行（headless 实例）、604–623 行（附加/关闭 panel）、1597–1605 行（导航前设置路径）、1790–1833 行（成功目录响应）、5086–5104 行（活动路径与连接检查）；PanelState 23–49 行（连接/路径读取写入）。
  - 稳定锚点：`currentPanel`、`getOrCreateHeadless`、`attachPanel`、`handlePanelDisposed`、`openConnectionPath`、`listDirectoryForConnection`、`getActivePath`、`requireActiveConnectionId`、`RemoteEditPanelState`。
  - 原文事实：panel 可独立关闭，后台实例/连接可保留；活动 ID 和路径由宿主保存；目录成功响应更新路径并发出 DirectoryListed，但 getActivePath 存在根目录回退，且导航请求前也可能预设路径。
  - 推导判断：菜单条件需同时确认真实 panel 可见与目标有效，不能只使用共享连接状态或路径回退值。
  - 适用范围与限制：新增目标状态接口的现状依据，不表示该接口和菜单 context key 已存在。
  - 核实状态：已核实。

- REF-005
  - 来源：`src/sidebar/SidebarController.ts`；`package.json`；https://github.com/microsoft/vscode-docs/blob/main/api/references/contribution-points.md ；本地 `node_modules/@types/vscode/index.d.ts`。
  - 用途：确认原生菜单条件机制和 Webview 可见性 API。
  - 参考范围：SidebarController 424–431 行；package.json 773–830 行；官方文档 `contributes.menus` 下的菜单位置列表中 `explorer/context`（核实时 1033 行）；vscode 类型定义 10118–10133 行。
  - 稳定锚点：`updateRemoteClipboardContext` 中的 `setContext`、`contributes.menus`、`explorer/context`、`WebviewPanel.active`、`WebviewPanel.visible`、`WebviewPanel.onDidChangeViewState`。
  - 原文事实：项目已有自定义 context key 控制菜单的机制；官方提供 Explorer 右键菜单位置；Webview API 分别提供 active、visible 和视图状态变化事件。
  - 推导判断：可使用 visible 与新增 context key 控制菜单，无需以 Webview 是否获得焦点为前提。
  - 适用范围与限制：菜单/API 可用性依据；当前上传菜单尚未注册，运行时状态同步仍需实现和验证。
  - 核实状态：已核实。

- REF-006
  - 来源：`src/panel/webview/scripts/DragDropTargets.ts`；`src/panel/webview/scripts/TransfersStatus.ts`。
  - 用途：沿用 Webview 上传落点与可接受拖放状态。
  - 参考范围：DragDropTargets 10–27 行（接收条件及目标目录）、36–75 行（目录悬停展开）；TransfersStatus 553–555 行（传输动作可用状态）。
  - 稳定锚点：`canAcceptFileListDragTarget`、`isFileListDragTarget`、`getFileListDragTargetDirectory`、`scheduleFileListDragTargetAutoOpen`、`canStartTransferAction`。
  - 原文事实：拖放要求活动连接、传输动作可用及 files 视图；目录行使用该目录，其余落点使用当前路径；悬停可导航进入目录。
  - 推导判断：新内部拖放只扩展来源识别，目标解析继续复用。
  - 适用范围与限制：拖入 Webview 文件列表的现有交互。
  - 核实状态：已核实。

## Review 状态与交接说明

状态：**待 review**。

用户已确认两项功能方向及编写 prompt 的范围。本文已收敛来源数据格式、两个触发入口、菜单显示条件、目标快照和共用上传链路；无阻塞性业务问题，可以作为后续 checklist 的输入，但本轮不自动进入 create/execute。

执行时必须保留真实 VS Code 拖放、多选目录完整性、菜单可见性及目标固定的验收要求。当前“可行”是代码/API 链路判断，没有声称运行验收已完成。本轮只新增本文档，没有修改业务代码、配置或既有 checklist。
