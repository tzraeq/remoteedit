# 远程文件编辑器 root 显示方式设置

## 业务背景与目标

多个远程连接的目标主机可能都是 `127.0.0.1`，通过不同端口访问不同目标。当前 VS Code 编辑器面包屑和同名文件 tab 的路径描述无法清楚区分这些连接。

用户已明确：连接名称 `[group/]connection` 更适合区分这些文件，同时希望尊重原作者的设计意图，增加设置供用户选择 root 显示 host 还是 connection name，而不是整体替换原有行为。默认继续使用 host；选择 connection name 后，有分组时显示分组名称和连接名称，无分组时显示连接名称。这里的方括号表示分组可选，不是要显示的字符。

目标是让用户按偏好选择文件来源的显示方式：提供 host 显示，也可以根据已保存的连接组织方式识别文件。用户补充确认（2026-09-19）：侧边栏与 Webview 的 host 显示也应统一，两个入口均使用目标主机名。文件名保持真实文件名，同名文件的补充描述继续由 VS Code 根据路径差异生成。

用户最新确认（2026-09-19）：“好，已打开的就先不管了”。设置只影响之后的打开动作，已打开文件保持原 URI、显示和编辑状态，关闭并重新打开后采用当前设置。此前文件夹移动的例子只是显示效果类比，真实远程文件位置不变。

用户已授权先实现本文件所述虚拟路径功能，并接受上述生效方式；直接检查当前显示链路，无需追溯 jump 的开发历史。上传入口增强仍是独立需求，本轮不实施。

## 可行性结论（2026-09-19）

**设置选项及新打开文件的显示模式可以实现。用户已接受不刷新已打开文件，原自动刷新阻塞 F1 不再阻塞本次实施。**

核实包含项目现状、VS Code 1.90.0 源码及该版本原版扩展宿主中的隔离内存文件实验。结论限定为当前普通文件编辑器架构、公开稳定扩展 API 和已验证的切换方式，不宣称所有可能的架构改造都不可能。

| 核实路径 | 结果 | 对方案的影响 |
| --- | --- | --- |
| 为新打开文件生成不同显示前缀 | 现有 URI 构造与解析机制可承载 | 设置本身可做，仍需业务实现与验收 |
| 保持文档 URI，仅动态修改两处显示 | 面包屑节点直接读取 URI 路径段；普通扩展也不能调用动态 formatter 提案 API | 不能按单纯修改标签的方式实现完整目标 |
| `WorkspaceEdit.renameFile` 配合纯内存虚拟别名切换 | 新 URI 保留未保存文字和 dirty 状态，但 Ctrl+Z 先撤回 URI，后续撤销未恢复原文字编辑过程 | 可以更新编辑器资源，但不是无副作用的显示刷新 |
| `workspace.fs.rename` 配合纯内存虚拟别名切换 | 新编辑器显示存储内容，未保存文字未迁移，dirty 变为 false | 不符合保留未保存编辑状态的要求 |

这些实验说明不能用 URI 迁移实现无副作用的显示刷新。本次采用用户已确认的“之后打开生效”契约，不迁移已有文档，保留实验依据供查阅。[REF-007～REF-011]

## 原显示链路与本次变更

- 两个 URI scheme（`remoteedit`、`remoteedit-readonly`）的资源标签均使用 `${path}`，以 `/` 为分隔符。[REF-001]
- `buildRemoteEditUri` 将显示标识作为虚拟根目录加到真实远程路径前；query 中的 `connectionId` 和 `remoteRoot` 用于还原连接与真实路径。host 模式仍经过主机名截短和 ASCII 字符替换；connectionName 模式新增独立路径段处理。[REF-002]
- host 模式中，Webview 与侧边栏均提供目标主机名，不再由侧边栏拼接 `username@host:port`。两个入口共用 `resolveEditorRootSegments` 解析可选分组/连接前缀。[REF-003、REF-004、REF-012]
- 保存配置已有连接 `name`、`groupId`，分组已有 `id`、`name`。活动连接有 `name` 和 `isQuickConnect`，但没有分组字段。分组名称需要根据保存配置关联取得。[REF-005]
- 现有设置分为 `Remote Edit: UI`、`Remote Edit: Webview`、`Remote Edit: Sidebar` 等类别；UI 类别已有编辑器标题按钮及状态栏显示选项。侧边栏和 Webview 的 Settings 入口都打开按扩展筛选的 VS Code 原生设置。[REF-006]

由此判断：现有虚拟路径机制可以承载可选的连接名称显示层级。host 模式统一输入后沿用原有构造逻辑；connectionName 模式需要独立的名称处理，不能直接套入现有主机名截短和 ASCII 替换逻辑，否则中文、空格、句点会丢失或被截断。

## 范围与显示规则

覆盖从侧边栏和 Webview 打开的远程文件，包括普通编辑、只读打开，以及现有共用 URI 构造器的比较文件入口。两种 scheme 均受该设置控制；connectionName 模式使用一致的连接路径规则，host 模式也统一使用目标主机名。[REF-001、REF-003、REF-004]

该设置控制 VS Code 文件编辑器中的显示前缀，不控制 Webview 内部的 Remote Path 导航或侧边栏文件树布局；这些已有各自的设置。[REF-006]

### 设置位置与选项

在现有 **VS Code 设置 → Remote Edit: UI** 类别中增加一个枚举选项，沿用原生设置入口。这里是两个打开入口共同使用的编辑器显示偏好，放在 UI 类别便于发现，无需为了单个选项另建设置分类。[REF-006]

配置契约：

| 项目 | 采用方案 |
| --- | --- |
| 配置键 | `remoteedit.editorRootLabel` |
| 设置名称 | `Editor Root Label`（按现有英文设置风格） |
| 类型 | `string` 枚举 |
| 默认值 | `host` |
| `host` | 沿用当前基于主机信息的 root 显示 |
| `connectionName` | 使用 `[group/]connection` 作为显示前缀 |
| 位置 | `Remote Edit: UI`，现有选项之后，`order: 50` |
| 配置范围 | VS Code 窗口级显示偏好，可在用户或工作区设置中配置；不写入单个连接配置 |

英文描述：`Controls the root label shown in breadcrumbs and tab descriptions for remote files opened in the editor.`

选项说明分别为 `Use the existing host-based root label.` 和 `Use the connection name, prefixed by its group name when available.` 设置描述补充：`Changes apply when opening files; already open editors keep their current labels until closed and reopened.`

### host 模式

未配置该选项或选择 `host` 时，侧边栏和 Webview 均将 `connection.host` 传入共用 URI 构造器，保留既有主机名截短和字符转换规则。相同连接和远程路径在两个入口生成相同的 authority 与显示路径，`openSource` 仍保留各入口原有语义。例如目标 `127.0.0.1` 均显示为 `/127.0.0.1/真实远程路径`。现有端口区分不足通过用户选择 connectionName 模式解决。旧 URI 仍按自身 query 解析，已打开文档不迁移。

### connectionName 模式

显示路径结构：

```text
有分组：/group/connection/真实远程路径
无分组：/connection/真实远程路径
```

示例（远程文件都是 `/etc/app/config.yml`）：

| 分组 | 连接名称 | 面包屑对应路径 |
| --- | --- | --- |
| 生产 | 应用 A | `/生产/应用 A/etc/app/config.yml` |
| 生产 | 应用 B | `/生产/应用 B/etc/app/config.yml` |
| 测试 | 应用 A | `/测试/应用 A/etc/app/config.yml` |
| 无 | 本地转发 | `/本地转发/etc/app/config.yml` |

`group` 和 `connection` 是可见的路径层级。连接名称不同的同名文件应能在连接层区分；跨分组同名连接的同名文件应能在分组层区分。tab 主标题仍为 `config.yml`，补充描述由 VS Code 原生同名文件消歧机制决定，不固定要求它总是完整展示上述路径。

这里的分组和连接是显示用前缀，不是远程服务器上的真实目录。

## 方案与端到端流程

### 名称来源

1. 打开动作读取有效设置。host 模式进入现有生成链路；connectionName 模式按以下步骤解析名称。
2. 取得目标 `connectionId` 后，始终按这个 ID 解析显示信息，避免异步读取期间切换活动连接导致名称和文件错配。
3. 对保存连接，从对应配置取得当前连接名称，根据 `groupId` 查找分组名称；没有分组时只使用连接名称。
4. 对 Quick Connect，使用活动连接名称，不关联同 ID 的保存配置。保存配置已不存在但会话仍有效时，同样使用活动连接名称作为无分组回退。
5. connectionName 模式下，两个入口共用名称解析和 URI 构造规则，避免各自拼接出不同的显示前缀。名称解析属于打开文件的准备过程，文件系统解析 URI 时不重新查询名称或当前设置。

以上处理集中在 `resolveEditorRootSegments`；数据来源见 REF-005、REF-006、REF-012。通过 VS Code configuration 保存显示偏好，连接和分组的数据模型沿用现状。

### URI 与真实路径

以下 URI 构造方式用于之后的打开动作。既有文档不做 URI 迁移，不订阅配置变化来关闭、重命名或重开编辑器。

- 将显示前缀与内部连接标识分开处理：显示名称允许重名，文件访问始终由原始 `connectionId` 定位，不能用名称反查连接。
- connectionName 模式下，URI 的 `path` 使用上述显示前缀加真实远程路径；query 的 `remoteRoot` 保存完整前缀（包括可选分组层级），`connectionId` 保存原始连接 ID。
- `parseRemoteEditUri` 应准确剥离整个虚拟前缀，得到原始远程路径。名称中有中文、空格、句点、`#`、`?`、`%` 时使用 URI/query 的正规编码，不让名称污染 URI 结构。
- connectionName 模式的 authority 使用稳定的连接身份构造，显示名称不承担身份唯一性；query 中的原始连接 ID 保持权威。host 模式的 authority 统一由目标主机名按既有规则生成。[REF-002]
- 保持 `readOnly` 和 `openSource` 的既有语义。普通编辑、只读和比较文件都要遵循选定模式的显示规则。[REF-002、REF-003]
- 已有旧格式 URI 仍按自身记录的 `remoteRoot` 解析，不能改为按当前名称或固定层数剥离前缀；这样存量 tab 的真实路径仍可还原。[REF-002]

### 已打开文件的更新时机

Q1 已按用户最新确认收敛：切换 `host` / `connectionName` 后，已有远程文件的面包屑和 tab 路径描述保持原样；之后的打开动作采用新模式。用户关闭并重新打开文件后生效。升级保持默认 host，之后从侧边栏打开的文件也使用统一的主机名显示。

设置变化不操作已有文档，因而不得自动保存、丢弃未保存内容或发起远程文件移动、重命名、删除。设置影响新 URI 的显示前缀，连接 ID 与真实远程路径仍按 URI 自带元数据还原。

连接或分组改名、移动分组时，之后打开文件读取最新配置；不额外实时刷新已有文档。

### 既有文档自动更新的技术核实记录

- 已核实：公开 API 中 `TextDocument.uri`、`Tab.label`、`Tab.input` 为只读；配置变更可以监听，但不能直接给 tab 的标签或文档 URI 赋值。[REF-007]
- 已核实：VS Code 1.90.0 的文件面包屑根据资源 URI 的路径逐层取父目录形成，各节点名称直接取 `basenameOrAuthority(resource)`；tab 自定义名称不影响此链路。因此，仅修改 tab 名称或资源描述格式并不足以更新面包屑。[REF-008、REF-010]
- 已核实：当前扩展的文件系统 `rename`、`delete` 会调用远程会话的实际操作；VS Code 文件移动的覆盖分支可能先删除已存在目标。不能把 `WorkspaceEdit.renameFile` 当作纯标签刷新接口直接使用。[REF-009]
- 已实测：动态 formatter 注册被稳定 API 权限检查拒绝；纯内存虚拟别名切换虽能更新 URI，但 WorkspaceEdit 路径干扰撤销，直接 fs 路径未迁移未保存内容。第二次独立运行排除了前一次失败移动对撤销行为的影响。[REF-011]
- F1：核实的自动刷新路径不符合无损刷新要求，本次不采用。
  - 影响：已有普通、只读和比较文件保持原有 URI 和状态。
  - 状态：**已解除阻塞**。用户于 2026-09-19 明确接受已打开文件暂不处理；本次实现不含自动刷新。
  - 核实边界：实际运行版本为原版 VS Code 1.90.0；实验使用隔离 profile、内存文件和纯虚拟别名，没有连接远程服务器。已发现的普通文本编辑反例足以否定这些候选的整体交付可行性；未进一步实施只读、比较文件和多编辑器组的完整迁移，不声称这些场景已经通过。

## 验收标准

以下为验收标准。实现验证结果见文末；独立 API 能力验证结果 REF-011 仅解释自动刷新方案的取舍。

1. 原生设置中能在 `Remote Edit: UI` 找到 `Editor Root Label`，可选 `host`、`connectionName`，默认 `host`。不设置或显式选择 host 时，同一连接和路径从侧边栏、Webview 打开，应生成一致的 authority、显示路径及对应 scheme，覆盖普通和只读打开；已有用户升级后不自动采用连接名称显示。原有侧边栏格式 URI 仍能正确还原真实路径。
2. 选择 connectionName 后，两个目标均为 `127.0.0.1`、端口不同、连接名称不同的连接，打开同一路径的同名文件，面包屑出现各自连接名称，tab 的原生补充描述能够区分来源。
3. 两个不同分组中同名的连接，打开同一路径文件后，完整显示路径包含各自分组；在 VS Code 默认 tab 标签设置下核实分组层级可参与同名文件消歧。
4. 无分组连接只显示连接名称层级，不产生空目录层级。有分组和无分组均保留真实文件名与目录结构。
5. 侧边栏、Webview 的普通打开与只读打开使用一致的分组和连接前缀；比较文件使用同样的 URI 路径规则，保留现有比较标题。
6. 使用 `生产环境`、`应用 A.v2` 等名称，以及包含 `#`、`?`、`%` 的名称，显示信息不再受主机名截短或 ASCII 白名单破坏；URI 序列化后再解析仍得到正确连接 ID 和远程路径。
7. 对有分组、无分组及旧格式 URI，验证构造/解析或存量解析得到的 `connectionId` 与真实路径正确；读取和保存访问原文件，虚拟前缀不进入远程文件系统请求。两份同名配置也不得导致文件访问串连。
8. Quick Connect 和保存配置已移除的有效会话按活动连接名称回退，仍能正确打开文件。
9. 已有多个远程文件打开时，从 host 切换到 connectionName，再切回 host，已有 URI、面包屑和 tab 显示保持原样，之后打开的文件采用当前模式；关闭并重新打开后采用当前模式。普通、只读和比较入口均应验证。
10. 存在未保存修改时切换设置，原文档内容、dirty 状态及撤销/重做保持可用；不自动保存、丢弃修改或发起远程文件移动、重命名、删除。配置变化本身不关闭或重开编辑器。

上述第 3–6、8 项针对 connectionName 模式；路径还原及文件访问正确性适用于两个模式。

验证结合 URI 层回归检查与真实 VS Code 扩展宿主中的文件打开、保存和比较操作。自动测试已核对实际文档与 tab 输入，面包屑可见文本和 tab 描述的最终布局仍需 UI 验收；仅 URI 字符串断言不能证明 tab 消歧效果。

## 默认决策与执行假设

- 默认 host，保留主机名构造规则；依据用户最新确认，侧边栏与 Webview 统一传入目标主机名。已有文档保留原 URI，不自动采用新显示。
- 设置放在现有 `Remote Edit: UI`，使用 `remoteedit.editorRootLabel` 与原生设置入口。依据是现有分类和入口，以及本项属于跨打开入口的编辑器显示偏好。[REF-006]
- connectionName 模式下，分组名称和连接名称作为两个独立路径段；依据用户指定的 `[group/]connection` 和现有 `/` 路径标签规则，影响面包屑层级与 tab 消歧。
- 保留普通名称的中文、空格和句点。名称内部的 `/`、`\` 默认分别显示为全角 `／`、`＼`；名称恰为 `.`、`..` 时默认使用全角句点，避免把名称误当路径层级或相对路径。此为显示用转换，不改保存配置。依据是现有名称不是文件系统路径段（REF-005），成立条件是在真实 VS Code 中验证这些字符的 URI 往返与显示效果。
- Quick Connect 或保存配置缺失时使用活动连接名称作为无分组回退。依据是活动连接已有名称，且无分组字段；不增加分组持久化数据。[REF-005]
- 之后打开文件时读取最新保存名称，不依赖活动会话名称及时同步。设置变化、单独改名或移动分组均不刷新已有文档。
- 保持原生 tab 文件名及路径缩短规则，不修改用户的编辑器标签设置。UI 验收先以默认设置为基准。

## 需用户确认的问题

无。Q1 最新确认：已打开文件暂不处理，设置只在之后打开时生效。用户已授权先实现虚拟路径功能，F1 因本次不再要求自动刷新而解除阻塞。

## 参考资源

本地路径均相对项目 `E:/workspace/vscode/remoteedit`。行号核对日期：2026-09-19；源文件变化时按稳定锚点重新定位。外部 VS Code 源码固定引用项目最低支持版本 1.90.0，其他编辑器版本仍须在实际环境验证。

- REF-001
  - 来源：`package.json`。
  - 用途：确认面包屑和 tab 路径标签使用的数据。
  - 参考范围：729–750 行。
  - 稳定锚点：`contributes.resourceLabelFormatters` 中 `remoteedit`、`remoteedit-readonly` 两项。
  - 原文事实：两项都使用 `${path}`、`/` 分隔符，并分别设置 workspaceSuffix。
  - 推导判断：把分组/连接放入 URI 路径可复用现有路径标签机制。
  - 适用范围与限制：标签来源配置；实际 tab 消歧与面包屑显示还需在 VS Code 验证。
  - 核实状态：已核实。

- REF-002
  - 来源：`src/filesystem/RemoteEditFileSystemProvider.ts`。
  - 用途：确认虚拟前缀、连接身份和真实路径还原契约。
  - 参考范围：226–303 行（构造、解析与 query）；309–353 行（前缀剥离及显示字符串转换）。
  - 稳定锚点：`buildRemoteEditUri`、`parseRemoteEditUri`、`buildRemoteEditUriQuery`、`stripVirtualRoot`、`normalizeUriAuthority`、`normalizeUriPathSegment`、`shortenHostname`。
  - 原文事实：host 显示标识经截短和替换后成为虚拟根；rootSegments 分支保留名称字符并规范化路径分隔符；query 保存连接 ID 和完整根；解析优先使用 query 的连接 ID，按根字符串剥离路径前缀。
  - 推导判断：新方案应保存完整分组/连接前缀并继续按 URI 自带元数据解析；旧 URI 可沿用原始元数据解析。
  - 适用范围与限制：两种远程文件 scheme 的构造与解析；不证明编辑器支持原地更新已有 tab 的 URI。
  - 核实状态：已核实。

- REF-003
  - 来源：`src/panel/RemoteEditPanel.ts`。
  - 用途：确定 Webview 中需要贯通的文件 URI 入口。
  - 参考范围：2027–2041 行（文件打开）；2133–2138 行（比较）；2201–2212 行（路径打开）。
  - 稳定锚点：调用 `buildRemoteEditUri(connectionId, entry.path, ...)` 的循环、`leftUri`/`rightUri`、`openFile`、`resolveEditorRootSegments`。
  - 原文事实：入口共用 URI 构造函数与名称解析；先按本次动作的 connectionId 捕获连接再异步解析名称；打开与比较分别交给 `vscode.open`、`vscode.diff`。
  - 推导判断：connectionName 模式统一名称来源，异步解析时绑定打开动作原始连接 ID；host 模式与侧边栏统一使用目标主机名。
  - 适用范围与限制：Webview 发起的远程文件编辑器 URI。
  - 核实状态：已核实。

- REF-004
  - 来源：`src/sidebar/SidebarController.ts`。
  - 用途：确认侧边栏入口与 Webview 使用相同的显示来源。
  - 参考范围：1209–1219 行。
  - 稳定锚点：`openRemoteFileInEditor`。
  - 原文事实：检查已连接状态后，取得目标连接的 host 并解析可选名称前缀，传给同一 URI 构造函数，再执行 `vscode.open`。
  - 推导判断：两个模式的显示来源均与 Webview 一致。
  - 适用范围与限制：侧边栏普通与只读文件打开。
  - 核实状态：已核实。

- REF-005
  - 来源：`src/connection/ConnectionManager.ts`；`src/remote/RemoteSessionTypes.ts`。
  - 用途：确认名称、分组与活动连接的数据来源。
  - 参考范围：ConnectionManager 12–40 行（配置和分组类型）、247–253 行（分组读取）、390–409 行（配置读取）、1611–1613 行（分组名规范化）；RemoteSessionTypes 77–99 行（活动连接类型）。
  - 稳定锚点：`ConnectionProfile`、`ConnectionGroup`、`ConnectionManager.listGroups`、`ConnectionManager.listProfiles`、`ConnectionManager.getProfile`、`normalizeGroupName`、`ActiveConnection`。
  - 原文事实：保存配置具有 `name`、`groupId`，分组具有 `id`、`name`；活动连接具有 `name` 和 `isQuickConnect`，无 `groupId`；分组名规范化修剪并合并空白，不限制为 ASCII 路径段。
  - 推导判断：名称解析应关联保存配置与分组，活动连接名称用于回退；显示字符串需要独立于主机名清洗规则处理。
  - 适用范围与限制：现有元数据足以生成名称前缀；不表示连接名称具有唯一性。
  - 核实状态：已核实。

- REF-006
  - 来源：`package.json`；`src/sidebar/SidebarController.ts`；`src/panel/RemoteEditPanel.ts`。
  - 用途：选择设置归属与访问入口，避免将编辑器 root 与文件浏览导航设置混淆。
  - 参考范围：package.json 462–600 行（UI、Webview、Sidebar 三个分类及属性）；SidebarController 451–453 行（设置入口）；RemoteEditPanel 724 行（`showSettings` 回调）。
  - 稳定锚点：`contributes.configuration` 下 `Remote Edit: UI`、`Remote Edit: Webview`、`Remote Edit: Sidebar`；`SidebarController.openSettings`；`showSettings: () =>`。
  - 原文事实：UI 类别有编辑器标题按钮与状态栏显示选项，order 为 10、20、30、40；本次 root 设置为 order 50、window 范围；Webview 和 Sidebar 类别配置各自的浏览导航显示；两个 Settings 入口均调用 `workbench.action.openSettings`，过滤条件为 `@ext:josegrabelha.remoteedit`。
  - 推导判断：新增枚举复用现有 UI 分类和两个 Settings 入口。
  - 适用范围与限制：设置组织与导航方式。
  - 核实状态：已核实。

- REF-007
  - 来源：https://github.com/microsoft/vscode/blob/1.90.0/src/vscode-dts/vscode.d.ts 。
  - 用途：核实普通文本文件与 tab 的公开修改能力，以及设置变更通知。
  - 参考范围：88–99 行（文档 URI）、18093–18109 行（tab 标签与输入）、13642 行（配置变更事件）。
  - 稳定锚点：`TextDocument.uri`、`Tab.label`、`Tab.input`、`workspace.onDidChangeConfiguration`。
  - 原文事实：上述文档和 tab 属性均为只读；存在配置变更事件。
  - 推导判断：监听设置变化可触发处理，但不能直接通过这些属性改写已有文档或标签。
  - 适用范围与限制：VS Code 1.90.0 稳定 API；不能据此断言所有间接显示刷新方案均不可行。
  - 核实状态：已核实。

- REF-008
  - 来源：https://github.com/microsoft/vscode/blob/1.90.0/src/vs/workbench/browser/parts/editor/breadcrumbsModel.ts 。
  - 用途：核实文件面包屑层级来源。
  - 参考范围：125–146 行。
  - 稳定锚点：`BreadcrumbsModel._initFilePathInfo` 中的 `uriPrefix` / `dirname(uriPrefix)` 循环。
  - 原文事实：按资源 URI 向上取父路径，并生成各级 `FileElement`。
  - 推导判断：新旧 root 的层级变化需要能够影响面包屑的数据来源，仅修改 tab 标题不够。
  - 适用范围与限制：内置编辑器实现证据，不是扩展可直接调用的公开接口。
  - 核实状态：已核实。

- REF-009
  - 来源：`src/filesystem/RemoteEditFileSystemProvider.ts`；https://github.com/microsoft/vscode/blob/1.90.0/src/vs/platform/files/common/fileService.ts 。
  - 用途：确认真实文件移动接口不能直接作为显示刷新接口。
  - 参考范围：本地 114–140 行；外部 773–824 行。
  - 稳定锚点：`RemoteEditFileSystemProvider.delete`、`RemoteEditFileSystemProvider.rename`、`FileService.doMoveCopy`。
  - 原文事实：本地 provider 调用 `sessions.delete`、`sessions.rename`；外部移动实现覆盖已有目标时可能先 `del(target)`，同 provider 移动再调用 `rename`。
  - 推导判断：对映射到同一真实文件的两个虚拟 URI，直接走真实文件移动可能产生远程文件操作，不符合本需求。
  - 适用范围与限制：约束未经适配直接使用文件移动的方案；不代表已验证其他虚拟 URI 切换方式。
  - 核实状态：已核实。

- REF-010
  - 来源：https://github.com/microsoft/vscode/blob/1.90.0/src/vs/workbench/browser/parts/editor/breadcrumbsControl.ts ；https://github.com/microsoft/vscode/blob/1.90.0/src/vs/workbench/browser/labels.ts ；https://github.com/microsoft/vscode/blob/1.90.0/src/vscode-dts/vscode.proposed.resolvers.d.ts 。
  - 用途：核实 formatter 或 tab 名称能否替代面包屑节点名，并确认动态 formatter 的 API 状态。
  - 参考范围：breadcrumbsControl 133–144 行；labels 406–434 行；resolvers 提案 436–457 行。
  - 稳定锚点：`FileItem.render`、`ResourceLabelWidget.setFile`、`workspace.registerResourceLabelFormatter`。
  - 原文事实：面包屑调用 `setFile` 并隐藏路径描述；普通节点名称取 URI 的 basename；动态 formatter 声明位于 `resolvers` 提案中。
  - 推导判断：修改路径描述不能替代面包屑节点文本；动态 formatter 也不是可直接使用的稳定公开能力。
  - 适用范围与限制：VS Code 1.90.0 的普通文件面包屑；工作区根目录显示另有分支，把连接添加成工作区目录会改变工作区结构，不等同于本需求的显示偏好。
  - 核实状态：已核实。

- REF-011
  - 来源：隔离实验目录 `C:/Users/tzrae/AppData/Local/Temp/remoteedit-label-feasibility-0397ea3de4a24f478b892b4ee4c44b74/` 下 `probe/tests.js`、`result.vscode-run1.json`、`result.vscode-run2.json`。
  - 用途：实际检验虚拟 URI 切换能否保留未保存内容和原有撤销行为。
  - 参考范围：`result.vscode-run2.json` 1–187 行，覆盖环境、对照、迁移、撤销、直接 fs 切换和内存操作记录；稳定锚点可重定位单项结果。临时文件若被清理，应按下面复现条件重做实验，不把丢失的日志当作仍可读取。
  - 稳定锚点：`environment`、`dynamicFormatterRegistration`、`controlUndo`、`controlRedo`、`afterVirtualRename`、`undo1`、`undo2`、`undo3`、`afterDirectFsRename`、`operations`。
  - 复现条件：原版 VS Code 1.90.0 官方 Windows x64 archive，隔离用户目录和扩展目录，不启用 proposed API；注册内存文件系统，存储内容为 `base\n`，以旧/新虚拟 URI 映射同一存储；两次文本插入后先确认普通 undo/redo 有效，再分别测试 WorkspaceEdit 和 fs 的 rename；provider 的 rename 只改变内存别名，记录并阻止删除/保存。
  - 原文事实：对照 undo 将 `second\nfirst\nbase\n` 变为 `first\nbase\n`，redo 恢复；WorkspaceEdit 切换成功并保留 dirty 与文字，但第一次 undo 将 URI 从 `/group/connection/file.txt` 撤回 `/host/file.txt`，后两次 undo 文字保持不变；直接 fs 切换后新编辑器为 `base\n`、dirty=false；formatter 注册报错 `CANNOT use API proposal: resolvers`。
  - 补充事实：第一次实验中，两种别名均被认为存在时，overwrite 移动尝试调用 delete，已由内存 provider 阻止。第二次独立运行跳过该失败实验，仍复现 WorkspaceEdit 的撤销副作用与直接 fs 切换的内容丢失。Cursor 独立窗口先前在编辑器初始化时报错，其运行结果未用于判断可行性。
  - 结果校验：`result.vscode-run2.json` SHA-256 为 `54D6D3F1A2D29880E59F06A5FD58892D00ECC3465C20D7E3DF26FDAEF1E0648D`。
  - 推导判断：这些 API 能更换编辑器资源，但已实测不满足纯显示切换的完整要求；不能把它们直接作为实现方案。
  - 适用范围与限制：独立内存实验，不是 Remote Edit 业务实现；未验证所有可能的替代架构，也不声称其他版本必然具有完全相同的行为。
  - 核实状态：已核实。

- REF-012
  - 来源：`src/filesystem/EditorRootLabel.ts`。
  - 用途：核对两个入口共用的设置读取与名称解析。
  - 参考范围：5–21 行。
  - 稳定锚点：`resolveEditorRootSegments`。
  - 原文事实：默认 host 返回 undefined；connectionName 读取保存配置和关联分组；Quick Connect 跳过保存配置，配置缺失时回退活动连接名称。
  - 推导判断：新打开时读取最新名称，文件系统解析旧 URI 无需查询配置。
  - 适用范围与限制：打开时名称解析，不订阅已有文档刷新。
  - 核实状态：已核实。

## Review 状态与交接说明

状态：**已确认，已实现并通过自动验证；UI 外观待验收**。

用户已明确增加 host / connection name 设置的方向，connection name 使用 `[group/]connection`。2026-09-19 用户授权先实现虚拟路径，并接受“已打开的就先不管了”，后续明确要求两个入口统一显示。按本文方案实施：默认 host，设置位于 `Remote Edit: UI`，侧边栏和 Webview 共用目标主机名或分组/连接名称，新打开时读取当前配置，已有文档保持原样。原 F1 不再阻塞此范围。

2026-09-19 实施验证：

- `npm test`：编译及现有 130 项测试通过。
- `npm run test:editor-root-label`：原版 VS Code 1.90.0 隔离扩展宿主通过。设置 `VSCODE_EXECUTABLE` 指向 VS Code 可执行文件后运行；测试启动器为 `scripts/test-editor-root-label.cjs`，测试入口为 `src/test/RemoteEditorRootLabel.integration.ts` 的 `run`。
- 已覆盖 host 基线、中文与保留字符 URI 往返、分组与无分组名称、Quick Connect/缺失配置回退、连接 ID 隔离、Webview/Sidebar 普通与只读打开、比较 tab 输入、原路径读写、切换设置后 dirty 与撤销/重做保留、关闭重开后采用新模式。另补充默认值及显式 host 模式下两个入口的显示路径/authority/scheme 一致性检查，修复前已复现 `root-127` 与 `127.0.0.1` 不一致。
- 使用内存会话替代远程服务器，没有连接真实服务器；面包屑可见文本和原生 tab 描述缩短结果未进行 UI 外观验收。

前期 REF-011 API 实验仅解释自动刷新为何不在本次实现范围，不作为本次功能通过的证据。
