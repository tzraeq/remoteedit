# SFTP Jump Host patch01：侧栏 Jump、连接 tab 名称与分组入口

## 业务背景与目标

用户在作者发布的版本中 Clone 一个带 Jump Host 的 SFTP 连接，然后在 Native Sidebar 修改端口，发现 Jump 显示为 Direct。目标是让侧栏编辑过程中未修改的 Jump 保持原值，显示、保存和连接使用一致的字段语义。

2026-09-19 用户明确要求：先将本方案写入并提交 `xxx.patch01.prompt.md`，再开始修复。本文按原 PR 文档命名为 `sftp-jump-host-pr.patch01.prompt.md`。

同日用户补充：rename 某个连接后，已打开的连接 tab 没有跟随改名。目标是重命名成功后立即更新对应 tab，无需重新连接。

同日用户继续补充并确认：在 Connections 标题栏新增分组入口，只创建一级分组；沿用左侧隐藏空分组、右侧 Webview 的 Manage Saved Connections 显示空分组的行为，不增加连接／分组节点右键入口或子组。

## 已核实的现状与问题

- 当前工作分支为 `dev`，源码基线已先同步到作者上游 v1.9.2 `f24b55b5fb58b6e70adfa580130114b986a13938`；提交 `3e3a467` 仅恢复 dev 专用的四份既有 prompt/checklist 文档，`src/`、包清单、README、CHANGELOG、LICENSE 与上游 `main` 零差异。
- 当前基线已包含作者新增的 Clone 和 Save As 功能。Clone 保留 `jumpProfileId`，并有对应断言；缺陷对 Clone 和普通已保存连接都成立，根因不在 Clone 实现。
- 侧栏第一次修改端口时只传入 `{ port }`。草稿规范化补出 `jumpProfileId: undefined`，随后 `{ ...profile, ...draft }` 覆盖已保存的 Jump，导致编辑中的显示为 Direct（REF-001）。
- 已在未修改的 v1.9.2 源码与现有 UI 测试桩上复现：Clone 后 Jump 正常，修改端口后草稿显示 Direct，持久化原值仍在，保存后恢复原 Jump。这里是草稿合并缺陷，并非作者有意清除 Jump。
- 保存及构造连接参数的现有契约：`undefined` 表示沿用已保存值，空字符串表示显式清除；非 SFTP 连接使用 Direct（REF-002）。
- 重命名更新保存的 profile，侧栏通过共享事件通知面板，面板重命名直接发送 profilesLoaded。该消息未重绘 tab，tab 又直接显示运行中 session 的名称快照，因此继续显示旧名（REF-004）。
- Native Sidebar 使用 VS Code TreeView；其空白处右键不进入扩展的节点菜单，官方菜单贡献点不能直接满足本次指定的空白处入口（REF-005）。
- 当前 ConnectionGroup 没有父组字段，已有 `createGroup` 可创建一级分组；侧栏只在移动连接时可顺带新建组，没有独立创建命令。TreeProvider 会过滤空组，并在没有任何连接时提前返回占位行；Webview 的 Manage Saved Connections 显式包含空分组，符合用户确认的显示规则（REF-006）。

## 范围与方案

适用范围为 Native Sidebar 对已保存连接的草稿合并，包括 Clone 产生的已保存连接。

在 `SidebarConnectionDraftStore.mergeProfileWithDraft` 中，合并原配置和草稿时显式保留 Jump 的三种语义，再调用现有协议规范化：

- 草稿 `jumpProfileId === undefined`：沿用原配置的 Jump。
- 草稿 `jumpProfileId === ''`：保留用户明确选择的 Direct。
- 草稿为非空 profile ID：采用用户新选的 Jump。

沿用现有 FTP/FTPS 清除 Jump 的规则。修复放在草稿合并处，使侧栏显示和使用合并配置的连接入口都获得正确值；保存仍使用现有契约。无需新增字段或迁移存储。

生产代码目标为 `src/sidebar/ConnectionDraftStore.ts`，回归测试放在现有 `src/test/ConnectionUi.test.ts`。直接使用当前 v1.9.2 基线的真实 `cloneProfile` 和侧栏入口验证 Clone 流程。

连接 tab 补充方案：`renderSessionTabs` 使用 session ID 查找最新保存的 profile 名称；Quick Connect 或不存在对应 profile 时沿用 session 名称。收到 profilesLoaded 后重新绘制 tab，后续 sessionsChanged 也沿用此显示规则，避免旧运行快照把名称改回去。仅更新标签显示，保留活动连接、顺序、路径及网络会话。

该补充修改 `src/panel/webview/scripts/LayoutSessions.ts`、`src/panel/webview/scripts/EventBindings.ts`；测试继续使用 `src/test/ConnectionUi.test.ts`，按需扩展 `src/test/helpers/ConnectionUiHarness.ts` 的最小 DOM 桩以验证实际生成脚本的 tab 内容。

一级分组入口方案（用户已确认）：在 Connections 标题栏增加 New Group 按钮，调用独立的侧栏创建流程。输入组名后复用 `ConnectionManager.createGroup`，刷新侧栏并发送 profiles 变更通知，让已打开的 Webview 收到新分组；输入取消不写入，空白或重复名称沿用现有校验。创建成功给予明确提示。新建组不移动已有连接。

该项修改 `package.json`、`src/sidebar/SidebarController.ts`，在现有 `src/test/ConnectionUi.test.ts` 中验证创建与变更通知。沿用 TreeProvider 隐藏空组和 Manage Saved Connections 显示空组的现有实现。新命令不加入节点右键菜单，不引入子组或调整空组显示规则。

## 参考资源

- REF-001
  - 来源：`src/sidebar/ConnectionDraftStore.ts`，上游 v1.9.2 / 修复前 dev 基线。
  - 范围与锚点：87–113 行 `mergeProfileWithDraft`；127–144 行 `updateDraftValue` 与 `updateConnectionDetailDraft` 的 port 分支；217–234 行 `normalizeDraftForType`。
  - 原文事实：首次字段编辑从 `{ id }` 建草稿；规范化产生 undefined Jump；合并时草稿覆盖原值；非 SFTP 规范化为显式空字符串。
  - 推导与用途：应在合并已保存配置时区分“未修改”与“明确清除”，不能用真值判断吞掉空字符串。
  - 核实状态：已核实；适用于当前 dev 和上游 v1.9.2 的同一文件。
- REF-002
  - 来源：`src/connection/ConnectionManager.ts`。
  - 范围与锚点：551–553 行 `saveProfile` 的 Jump 取值；1396–1398 行连接参数构造中的 Jump 取值。
  - 原文事实：两处都通过 `input.jumpProfileId !== undefined` 判断是否覆盖原配置。
  - 推导与用途：侧栏合并应与底层保持相同的 undefined / 空字符串语义。
  - 核实状态：已核实；沿用现有持久化和连接契约。
- REF-003
  - 来源：`src/test/ConnectionUi.test.ts`，67–86 行测试 `Sidebar draft editing, discard, Quick Connect and save preserve explicit Direct`。
  - 原文事实：现有测试覆盖显式 Direct、丢弃草稿及 Quick Connect 协议切换，但未覆盖仅修改已有连接端口的显示。
  - 推导与用途：增加真实侧栏编辑入口的回归用例，并保留显式清除及协议切换验证。
  - 核实状态：已核实；UI/存储边界由现有 harness 控制，不等同于实机 VS Code 验收。
- REF-004
  - 来源与范围（dev 修复前）：`src/sidebar/SidebarController.ts` 2815–2870 行 `renameSavedConnection`；`src/panel/RemoteEditPanel.ts` 538–556 行 profiles 事件订阅、947 行起 `sendProfiles`、1222–1231 行 `renameConnection`；`src/panel/webview/scripts/EventBindings.ts` 19–29 行 profilesLoaded 分支；`src/panel/webview/scripts/LayoutSessions.ts` 1483–1507 行 `renderSessionTabs`。
  - 原文事实：两种重命名入口均可向面板发送最新 profiles；tab 使用 `escapeHtml(session.name)`；profilesLoaded 仅刷新 profile 相关界面。
  - 推导与用途：在显示层按稳定 ID 解析名称并重绘，可修复两种入口且避免修改运行会话快照。
  - 核实状态：已核实；适用于面板内连接 tab，不扩展为其他窗口或文件编辑器标题重命名。
- REF-005
  - 来源：`https://github.com/microsoft/vscode/blob/main/src/vs/workbench/browser/parts/views/treeView.ts`，2026-09-19 读取的 844–849 行，稳定锚点 `AbstractTreeView.onContextMenu` 中 `if (node === null) { return; }`；同仓 `src/vs/workbench/services/actions/common/menusExtensionPoint.ts` 246–260 行菜单注册 `view/title`、`viewContainer/title`、`view/item/context`。
  - 原文事实：没有右键命中的树节点时直接返回；扩展节点右键使用 view/item/context。
  - 推导与用途：不能仅增加 view/item/context 条目就声称支持空白右键；采用用户已确认的标题栏入口。
  - 核实状态：已核实；来源为当日 VS Code main 源码，未进行用户安装版本的实机点击验证。
- REF-006
  - 来源与范围：`src/connection/ConnectionManager.ts` 34–40 行 `ConnectionGroup`、254–281 行 `createGroup`；`src/sidebar/TreeProviders.ts` 234–255 行无连接/无匹配提前返回、361–382 行 `groupProfiles`；`src/panel/webview/scripts/RemoteCommandOutput.ts` 1181–1225 行 `renderManageProfilesList`；`package.json` 的 contributes.commands 及 view/title、view/item/context 菜单。
  - 原文事实：分组结构为 id/name/order/时间戳，无 parentId；createGroup 校验非空与大小写不敏感的重名；groupProfiles 过滤零连接分组；现有菜单未提供独立新建分组。
  - 推导与用途：复用一级分组存储，保留左右两种显示规则；Webview 使用 `includeEmptyGroups: true`，新增后通过 profiles 通知获得完整分组列表。
  - 核实状态：已核实；本次不引入子组。

## 验收标准

1. 已保存的 SFTP 目标引用 Jump，侧栏只修改端口后，新端口可见且 Jump ID、路由显示仍正确；原持久化配置在保存前不变。
2. 保存端口后 Jump 保留；丢弃后恢复原端口与 Jump。
3. 明确选择 Direct 后再修改端口，仍显示 Direct，保存后清除 Jump；改选另一个有效 Jump 后再修改端口，保留新选择。
4. 已保存连接切换为 FTP/FTPS 时使用 Direct，再切回 SFTP 不恢复已显式清除的 Jump；现有 Quick Connect 行为通过回归测试。
5. 新增回归测试在修复前失败、修复后通过；执行包含编译的 `npm test`，检查 diff 仅包含本文及上述实现/测试文件。
6. 分别通过侧栏和面板重命名已有连接，处理最新 profilesLoaded 后，对应 tab 立即显示新名；其他 tab、活动连接、顺序与路径保持不变。
7. 随后收到仍携带旧名称的 sessionsChanged，tab 保持新名；Quick Connect 和没有对应已保存 profile 的会话仍显示原会话名。名称保持 HTML 转义。
8. Connections 标题栏可独立创建一级分组，空连接列表时也成立；创建成功后发出包含新组的 profiles 更新；取消不写入，空名/重名被拒绝，已有连接归属不变。
9. 新命令只加入标题栏视图菜单，不加入连接／分组节点右键；不创建子组。左侧继续隐藏空组，右侧 Manage Saved Connections 按现有规则显示空组。

## 默认决策与执行假设

- 用最小字段级合并修复，而不重构草稿规范化；依据是本次问题集中在 Jump 的未修改语义。
- 当前分支已先同步作者上游，再实施 patch；不在旧的 `5a77934` 源码上开发后移植。

## 需用户确认的问题

无。

## Review 状态与交接说明

已确认：用户授权针对已定位的三个问题，先同步作者远端，再单独提交本方案，之后应用代码 patch 并验证。本文记录已收敛范围与实现决策；不另设审批，不生成 checklist。

连接 tab 补充沿用同一授权与顺序：用户提出重命名同步缺陷后，先更新本文，再实施显示层修复与回归验证。

分组入口已确认，Q1 已解决（2026-09-19 用户答复）：仅使用标题栏新增，保留左侧隐藏空组、右侧管理窗口显示空组，不添加节点右键入口或子组。方案提交后按该范围实施。
