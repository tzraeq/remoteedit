# 本地文件上传入口增强 - Checklist

## 概述

实现本地 Explorer 拖入 Webview、Explorer 右键 Upload to Remote Edit；两者使用现有上传队列，支持文件、多选、目录递归。反向下载拖放及下载目录选择逻辑保持现状。

- 方案来源：docs/prompts/local-file-upload-entrypoints.prompt.md（两个上传入口与队列契约），仅溯源。
- 编制依据：2026-09-27 用户确认范围并授权“生成清单并全部实现”，指定 dev 分支；已有未提交草稿需审查，不能视作完成。
- 拆分 review：已由本轮“生成清单并全部实现 / 继续”授权覆盖本范围；按共用执行入口、拖放入口、右键入口三个闭环依赖拆分。

## 清单状态说明

- `[ ]` 未开始；`[-]` 进行中；`[x]` 完成；`[?]` 待确认；`[!]` 阻塞。已完成任务保留证据。

## 清单执行规则

1. 连续执行授权覆盖任务 1、2、3，按依赖推进；每项验证后回写。
2. 审查工作树草稿并复用正确实现。不得将未执行的实机验证记为通过。
3. 本轮不包含 commit/push 授权。若技术验证失败，记录具体限制并停止受影响任务。

## 执行前强制上下文

读取以上全局节、全局规范、确认依赖、参考及当前任务完整块。依据当前工作树及声明参考执行，行号漂移时按稳定符号定位；引用不是修改授权。使用 UTF-8 无 BOM。仅修改任务允许文件和本任务完成记录，用户及项目指令仍有效。

## 全局规范清单

- 本地来源为宿主可访问的 file URI / 原生绝对路径；读取磁盘内容，不自动保存编辑器。
- 所有上传进现有队列，复用冲突、取消、错误、进度、符号链接跳过规则。
- 操作时固定 connectionId、targetDirectory，异步收集与排队期间不重新读取当前目标。
- 单次来源去重，父目录递归覆盖已选后代；中文、空格、#、% 正确处理。
- 不实现已放弃的反向拖放下载，不改变下载 picker。

## 全局确认依赖

无未解决确认依赖。真实 VS Code MIME 传递与菜单状态是执行复核，不是业务审批。

## 全局参考

无。各任务仅加载直接需要的参考。

## 任务清单

### 1. 共用本地资源上传队列入口

- [x] **实现与验证** - 共用本地资源上传队列入口
  - **执行前上下文**：全局契约和本任务完整块。
  - **目标文件**：修改 src/panel/RemoteEditPanel.ts 的 URI 上传入队/来源接收；新建或审查 src/panel/LocalUploadSources.ts；修改 PanelMessages.ts、PanelHandlerTypes.ts、handlers/fileActionMessageHandlers.ts 的资源上传消息；新建 src/test/LocalUploadSources.test.ts；修改 package.json 的测试命令与 vscode-uri 开发依赖、package-lock.json；修改 src/test/helpers/ConnectionManagerHarness.ts 的必要 API stub。
  - **范围说明**：直接接收已选本地 URI 和固定远程目标，不弹 picker；旧 picker 入口也复用同一队列函数。验证 URI、绝对路径和去重；非法或失效来源明确反馈。
  - **硬边界**：不能以原生复制或独立传输绕过队列。
  - **执行依据事实**：REF-003：RemoteEditPanel requestUploadEntries/enqueueUriUpload 2890–2931、collectUploadTransferItems 3491–3509、collectUploadPath 3581–3619；原有 URI 收集递归目录并跳过符号链接，队列闭包持有目标。当前草稿已提取 enqueueUriUpload，尚未验收。
  - **规范清单**：使用 vscode.Uri.parse/fsPath、Uri.file 转换，不手工解码。拒绝非 file 来源。父子去重不能误删同前缀兄弟目录。队列任务 run 调用原 runUploadTransfer，保留 source 合法值。
  - **允许的实现判断**：在上述契约内选择私有函数组织和测试 stub；修正现有未验收草稿。
  - **优先级**：未指定，按依赖执行。
  - **前置任务**：无
  - **依赖产物与契约**：无
  - **下游交付**：2、3：提供 parseLocalUploadSources、selectExplorerUploadSources 与 enqueueUriUpload(connectionId,targetDirectory,uris,source)，直接入现有 Upload 队列；以实际代码符号及测试核验。
  - **确认依赖**：无
  - **参考文档**：所列 REF-002 外部源码固定 VS Code 1.90，其余无。
  - **参考代码**：以上 REF 以原 prompt 编号沿用，行号为草稿当前定位；稳定锚点为所列函数名。用途为本任务现状核对；原文事实见执行依据事实；推导为复用已有执行链并补齐入口；适用范围为桌面扩展，真实 UI 效果需实测；核实状态：源码已核实，草稿未验收。
  - **预期产出**：LocalUploadSources 的来源归一化/选择/解析与 RemoteEditPanel.enqueueUriUpload；普通 UT。
  - **完成标准**：npm run compile；node --test out/test/LocalUploadSources.test.js。覆盖 URI 特殊字符、非 file、重复/父子选择、实际临时目录递归、队列目标固定、来源不存在与取消。

  - **完成时间**：2026-09-27 21:46
  - **实际产出与交付结论**：LocalUploadSources 来源解析/去重/选择器及 enqueueUriUpload 已接入 requestUploadResources 与原 picker；任务闭包固定来源和远程目标，目录走原 URI 递归收集器。
  - **验证结果**：npm run compile 通过；LocalUploadSources.test.js 5/5 通过（特殊字符、拒绝非法来源、多选、递归、目标快照、失效连接、缺失文件、取消）。
  - **遗留问题**：无；菜单和前端草稿在任务 2、3 继续验收。

### 2. Explorer 资源拖入 Webview

- [!] **实现与验证** - Explorer 资源拖入 Webview
  - **执行前上下文**：全局契约和本任务完整块。
  - **目标文件**：修改 src/panel/webview/scripts/DragDropUpload.ts 的识别、资源读取和固定目标；必要时修改 DragDropTargets.ts 的本地接收判定；新建 src/test/DragDropUpload.test.ts；package.json 测试命令；可新建 scripts/test-upload-entrypoints.cjs 及 src/test/UploadEntrypoints.integration.ts 作为最小真实 VS Code 验证入口。
  - **范围说明**：大小写兼容识别 CodeFiles、application/vnd.code.uri-list、ResourceURLs、text/uri-list；一次选择最完整有效来源；保留 Files 上传与远程移动。目录行上传该目录，文件行/空白上传当前目录，保留悬停进入。
  - **硬边界**：不能以原生复制或独立传输绕过队列。
  - **执行依据事实**：REF-001：DragDropUpload readDroppedLocalResources/isLocalFileDrag 18–39、collectDroppedFiles（按符号重定位）、handleDragDropUploadDrop 373–443；REF-006：DragDropTargets 10–75。REF-002：VS Code 1.90 explorerViewer.ts 1182–1195、workbench/browser/dnd.ts 222–276、331–358；CodeFiles 含目录、标准 uri-list 仅首项。
  - **规范清单**：dragover 只识别类型，drop 才取数据。远程私有 MIME 不能误入本地上传。原生路径/URI 发送 requestUploadResources；Files 的路径与内容暂存两条异步分支都使用 drop 时目标快照。实际版本不提供可用 MIME 时停止并记录证据。
  - **允许的实现判断**：在上述契约内选择私有函数组织和测试 stub；修正现有未验收草稿。
  - **优先级**：未指定，按依赖执行。
  - **前置任务**：1
  - **依赖产物与契约**：任务 1 产物 parseLocalUploadSources + enqueueUriUpload 和 requestUploadResources 消息已经编译、UT通过；消息接收固定 connectionId/targetDirectory 和 paths 或 uris。
  - **下游交付**：3：提供已测试拖放入口及集成验证工具，最终回归复用。
  - **确认依赖**：无
  - **参考文档**：所列 REF-002 外部源码固定 VS Code 1.90，其余无。
  - **参考代码**：以上 REF 以原 prompt 编号沿用，行号为草稿当前定位；稳定锚点为所列函数名。用途为本任务现状核对；原文事实见执行依据事实；推导为复用已有执行链并补齐入口；适用范围为桌面扩展，真实 UI 效果需实测；核实状态：源码已核实，草稿未验收。
  - **预期产出**：生产拖放脚本和测试；任务 2 交付结论记录真实跨 Webview 数据格式、单/多选/目录覆盖及限制。
  - **完成标准**：普通 UT 覆盖格式优先级、畸形回退、非本地拒绝、异步切换目标、Files 两分支和落点规则。真实 VS Code 使用隔离测试来源验证 Explorer → Webview，不能以合成 DataTransfer 代替。

  - **阻塞记录（2026-09-27 21:52）**：普通 UT 7/7 通过，但原生拖放失败。用户在隔离 Upload Smoke 中将 folder 拖入后触发 VS Code 打开目录；results.jsonl 未出现 requestUploadResources。
  - **交付结论：宿主拦截**：VS Code 1.90 WebviewWindowDragMonitor 构造器监听 window dragstart 并调用 windowDidDragStart；WebviewElement.windowDidDragStart 调用 _startBlockingIframeDragEvents 设置 iframe pointerEvents=none；WebviewEditor 注册独立编辑器 drop target。生产 Webview 内部 MIME 解析不能接收这条拖放链路。
  - **证据位置**：https://github.com/microsoft/vscode/blob/1.90.0/src/vs/workbench/contrib/webview/browser/webviewWindowDragMonitor.ts 18–36；webviewElement.ts 525–534、700–708；webviewPanel/browser/webviewEditor.ts 181–186。隔离测试：C:/Users/tzrae/AppData/Local/Temp/remoteedit-upload-smoke-wh0eoi/results.jsonl。
  - **实际产出**：DragDropUpload.test.ts、test-upload-entrypoints.cjs 及原上传草稿仍保留；未宣称原生拖放入口完成。
  - **下一步**：方案回流，等待用户决定是否仅继续独立右键入口并移除无效 MIME 适配；任务 3 当前未执行。

### 3. 本地 Explorer 右键上传与状态生命周期

- [ ] **实现与验证** - 本地 Explorer 右键上传与状态生命周期
  - **执行前上下文**：全局契约和本任务完整块。
  - **目标文件**：修改 src/extension.ts、package.json 的命令/菜单；RemoteEditPanel.ts 的目标状态、命令、panel 生命周期；PanelMessages.ts、PanelHandlerTypes.ts、handlers/fileActionMessageHandlers.ts 的目标报告消息；webview/scripts/DragDropUpload.ts、EventBindings.ts、FileBrowser.ts、ServerJobsPortsActions.ts、TransfersStatus.ts、LayoutSessions.ts、StateDialogs.ts 中目标报告所需部分；新建 src/test/UploadContext.test.ts；必要修改 src/test/helpers/ConnectionUiHarness.ts；复用/补充 scripts/test-upload-entrypoints.cjs、src/test/UploadEntrypoints.integration.ts；package.json 测试命令。
  - **范围说明**：命令 remoteedit.uploadToCurrentWebviewDirectory，标题 Upload to Remote Edit；explorer/context 条件 resourceScheme == file && remoteedit.canUploadToWebview。仅真实可见 panel、有效活动连接、已显示目录可用；焦点不影响。多选中包含右键项则使用多选，否则用右键项。
  - **硬边界**：不能以原生复制或独立传输绕过队列。
  - **执行依据事实**：REF-004：RemoteEditPanel attachPanel/handlePanelDisposed 645–674、listDirectoryForConnection 1840–1890，PanelState 23–49；路径可能提前写入，headless 单例非可见 panel。REF-005：WebviewPanel.visible/onDidChangeViewState（类型定义按符号定位）。草稿通过前端已确认目录报告补充宿主状态。
  - **规范清单**：创建、隐藏、关闭、切换连接、断开、导航中及失败均同步 context。命令执行时重新核对状态并同步固定目标，不回退默认 /。成功目录响应或已加载快照才能报告目录；导航失败可恢复仍显示的旧目录。旧连接重连不能继承失效目标。普通打开文件/下载不改动。
  - **允许的实现判断**：在上述契约内选择私有函数组织和测试 stub；修正现有未验收草稿。
  - **优先级**：未指定，按依赖执行。
  - **前置任务**：1, 2
  - **依赖产物与契约**：任务 1 的选择器与入队函数已验证；任务 2 的来源适配和目标固定测试通过，集成入口可用于真实 VS Code 综合检查。
  - **下游交付**：无
  - **确认依赖**：无
  - **参考文档**：所列 REF-002 外部源码固定 VS Code 1.90，其余无。
  - **参考代码**：以上 REF 以原 prompt 编号沿用，行号为草稿当前定位；稳定锚点为所列函数名。用途为本任务现状核对；原文事实见执行依据事实；推导为复用已有执行链并补齐入口；适用范围为桌面扩展，真实 UI 效果需实测；核实状态：源码已核实，草稿未验收。
  - **预期产出**：命令菜单与确认目录生命周期；任务 3 交付结论记录自动测试、真实 UI 验收、剩余限制。
  - **完成标准**：npm test 全量回归；菜单生命周期、点击选择集合、导航失败/缓存恢复、断开重连、无焦点可见、操作后切换的 UT。隔离 VS Code 验证真实菜单显示/隐藏及命令进入 Upload 队列；原生拖放与菜单端到端结论如实记录。

  - **先行可行性试验（2026-09-27 22:14，用户要求试验右键入口）**：隔离 VS Code 1.90.0 中，Webview 可见、本地树获得焦点时右键 one.txt 显示 Upload to Remote Edit；点击后生产命令、选择器、enqueueUriUpload 生成 Upload 作业，目标 /upload/one.txt。右键 folder 生成目标 /upload/folder，生产收集器展开 folder/nested/中文.txt。切换本地编辑器隐藏 Webview 后菜单消失。
  - **试验边界**：使用模拟连接、真实 VS Code 菜单/命令与生产来源解析/入队入口；enqueueTransferJob 被测试回调接收，未运行生产队列调度和网络传输，因此不是完整上传验收。尚未验证实际连接导航/断开、批量命令、冲突及取消。任务 3 仍未标完成。
  - **试验证据**：scripts/test-upload-entrypoints.cjs；C:/Users/tzrae/AppData/Local/Temp/remoteedit-upload-smoke-B5afga/results.jsonl 的 queued、collected 记录。
