# SFTP 多级 Jump 连接链 - Checklist

## 概述

为 Remote Edit 的 SFTP 连接增加基于已保存 SFTP profile 的多级 Jump 链。每个连接最多引用一个 `jumpProfileId`，被引用连接可以继续引用另一连接，从而形成不设置人为最大深度的无环链。扩展自动解析链、补全各跳凭据、使用 `ssh2.Client.forwardOut()` 与 `sock` 逐级建连，并在失败、取消和断开时释放整条隐式链。

**主要功能点**：

- 保存、编辑、导入和导出 SFTP profile 的 jump 引用。
- 迭代解析任意深度无环链，拒绝自引用、循环、缺失和非 SFTP 引用。
- 通过独立中间 SSH 客户端逐级 `forwardOut()`，最终建立目标 SFTP 会话。
- 在 Webview 与原生侧边栏中选择、展示和清除 jump。
- 保护 SecretStorage 凭据、被引用 profile 和运行时资源生命周期。
- 保持 FTP/FTPS、直接 SFTP、终端、端口转发和文件操作行为不回归。

**关键文件路径**：

- `docs/prompts/sftp-multi-hop-jump.md`
- `src/connection/ConnectionManager.ts`
- `src/connection/JumpChain.ts`
- `src/remote/RemoteSessionTypes.ts`
- `src/ssh/SftpSessionManager.ts`
- `src/panel/webview/markup/Body.ts`
- `src/panel/webview/scripts/RemoteSearch.ts`
- `src/panel/webview/scripts/RemoteCommandActions.ts`
- `src/sidebar/ConnectionDraftStore.ts`
- `src/sidebar/ItemHelpers.ts`
- `src/sidebar/SidebarController.ts`

**清单状态说明**：

- `- [ ]` 未开始
- `- [-]` 进行中
- `- [x]` 已完成
- `- [?]` 待确认（发现问题需要用户确认）
- `- [!]` 阻塞中（依赖其他任务或外部条件）

## 清单执行规则

1. **按优先级执行**：优先完成 P0 任务，确保核心功能可用。
2. **按依赖关系执行**：严格按任务编号顺序执行，先稳定类型和数据契约，再接入运行时和 UI。
3. **验证标准**：每个任务完成后执行其“完成标准”中的验证；无法运行外部集成环境时不得伪造结果。
4. **状态更新**：开始任务时改为 `[-]`；验证成功后改为 `[x]`，追加完成内容和资源列表。
5. **提交边界**：每个完成任务将代码与对应 checklist 状态一体提交；提交消息遵守 `.cursorrules`，使用 `✨ feat(jump): 完成任务 N：...`。
6. **连续执行**：当前任务完成并提交后，自动进入下一个无确认依赖的任务，直到全部完成或出现真实阻塞。

## 执行前强制上下文

执行任一任务前，必须先读取并遵守 `概述`、`清单状态说明`、`清单执行规则`、`执行前强制上下文`、`全局规范清单`、`全局确认依赖`、`全局参考` 和当前任务完整任务块；如果当前上下文未包含这些内容，先回读 checklist 文件顶部。

**硬约束**：

- 只处理当前任务声明的目标文件和预期产出；不要顺手修改无关文件、主索引、状态文件或后续任务产物，除非当前任务明确列出。
- 严格按照当前任务的前置任务、范围说明、规范清单和完成标准执行。
- 证据不足、路径缺失或需求不明确时，将任务标记为 `[?]` 并写明待确认问题，不要靠猜测补实现。
- 任务状态为 `[?]` 或任务存在未解决确认依赖时，不得直接执行；必须先提示用户确认具体问题。
- `.cursorrules` 是用户现有未跟踪文件，只读取并遵守；除非用户另行要求，不把它加入任何提交。

## 全局规范清单

- 仅 SFTP 目标可以配置 jump，且只有已保存的 SFTP profile 可以充当 jump；FTP/FTPS 始终直连。
- 不定义 jump 最大深度；使用迭代遍历和 `visited` 集合检测自引用及任意长度循环。
- 统一规定运行时 `jumpChain` 顺序为“最外层、可从本机直连的 jump”到“最靠近最终目标的 jump”。例如 `A -> B -> C` 解析为 `[C, B]`。
- Webview、日志、普通 profile、备份和 `ActiveConnection` 不得包含密码、私钥内容或私钥口令。
- 中间 SSH 会话为最终目标专属，不复用可见活动连接，不注册成普通活动连接。
- 只有链的最外层节点执行本机 TCP 可达性探测；经 `sock` 建立的后续节点不得执行本机直连探测。
- 缺失、非法或循环引用不得静默退化为直连。
- 被其他 profile 引用的 jump 不得直接删除或改为 FTP/FTPS。
- 没有 `jumpProfileId` 的历史 profile 保持直接连接，不进行破坏性迁移。
- 备份格式升级到版本 3，并继续接受旧版本备份。
- 所有失败、取消和断开路径都必须释放已创建的客户端、forward stream、监听器和取消订阅。
- 每项实现完成后先运行差异检查与对应验证，再更新 checklist 状态并提交；不得提交 `.temp/` 执行日志。

## 全局确认依赖

无未解决确认依赖。

## 全局参考

- `docs/prompts/sftp-multi-hop-jump.md` - 业务范围、默认决策、数据契约和验收标准。
- `.cursorrules` - Git 提交消息与代码/checklist 一体提交规则。
- `package.json` - 构建脚本与运行时依赖。
- `package-lock.json` - `ssh2` 当前解析版本与锁定依赖。
- `src/connection/ConnectionManager.ts` - profile、SecretStorage、备份和连接参数构建。
- `src/remote/RemoteSessionTypes.ts` - `ConnectOptions` 与 `ActiveConnection`。
- `src/remote/ConnectionProbe.ts` - 本机 TCP 探测。
- `src/ssh/SftpSessionManager.ts` - SFTP 建连与资源生命周期。
- `src/ssh/PortForwardManager.ts` - 已有 `forwardOut()` 与 stream 清理用法。
- `src/ftp/FtpSessionManager.ts` - FTP/FTPS 不纳入 jump 的依据。
- `src/panel/RemoteEditPanel.ts` - profile 快照、保存和建连入口。
- `src/panel/webview/markup/Body.ts` - Webview 连接表单。
- `src/panel/webview/scripts/StateDialogs.ts` - Webview DOM 引用与状态。
- `src/panel/webview/scripts/RemoteSearch.ts` - 表单校验、dirty snapshot 与 payload。
- `src/panel/webview/scripts/RemoteCommandActions.ts` - profile 填充、清空和转换。
- `src/sidebar/ConnectionDraftStore.ts` - 侧边栏草稿模型。
- `src/sidebar/ItemHelpers.ts` - 侧边栏连接详情字段。
- `src/sidebar/SidebarController.ts` - 侧边栏编辑、保存和连接流程。

---

## 核心契约

### 1. 建立 Jump 链类型与无深度上限解析器

- [x] **类型定义** - Jump 链 - 建立稳定的链方向契约和纯解析逻辑
  - **执行前上下文**：执行前必须读取并遵守 `执行前强制上下文`；若当前上下文未包含该段，先回读 checklist 文件顶部。
  - **目标编号**：1
  - **目标名称**：建立 Jump 链类型与无深度上限解析器
  - **所属界面组**：无
  - **导航分组**：无
  - **目标文件**：
    - `src/connection/JumpChain.ts` - 新增纯 profile 图解析与校验能力。
    - `src/remote/RemoteSessionTypes.ts` - 定义运行时 jump hop、chain 与安全展示字段。
  - **硬边界**：只处理本任务目标文件和预期产出；不修改持久化、会话连接、UI、备份或测试文件。
  - **确认依赖**：无
  - **范围说明**：定义可供连接管理器和 SFTP 会话复用的 jump 描述类型；使用迭代遍历从最终目标沿 `jumpProfileId` 收集链，输出固定为最外层到目标最近层；验证自引用、任意长度循环、缺失 profile 和非 SFTP jump，且不设置最大深度。
  - **优先级**：P0
  - **前置任务**：无
  - **参考文档**：
    - `docs/prompts/sftp-multi-hop-jump.md` - 第 1、4.2、5.1、6、8、9 节。
  - **参考代码**：
    - `src/connection/ConnectionManager.ts` - 现有 profile 结构。
    - `src/remote/RemoteConnectionTypes.ts` - SFTP 类型判断。
    - `src/remote/RemoteSessionTypes.ts` - 现有连接运行时契约。
  - **接口依赖**：无
  - **类型依赖**：
    - `RemoteConnectionType` - profile 协议类型。
    - `ConnectOptions` - 最终连接参数。
  - **规范清单**：
    - 纯解析模块不得依赖 `vscode` 或 SecretStorage。
    - 错误必须包含可读链路或相关 profile 名称，便于上层展示。
    - 不使用递归深度限制或固定层数数组。
    - 空 jump 表示合法直连并返回空链。
  - **预期产出**：
    - `src/connection/JumpChain.ts` - 可复用的迭代链解析器。
    - `src/remote/RemoteSessionTypes.ts` - 明确的 `JumpConnectOptions` 与安全链摘要契约。
  - **完成标准**：
    - 代码可表示任意有限无环 jump 链，输出方向有注释和类型约束。
    - 自引用、循环、缺失与非 SFTP jump 均产生区分明确的错误。
    - `npm run compile` 通过。
  - **完成时间**：2026-08-27 08:27 CST
  - **实际产出**：新增纯迭代 `resolveJumpProfileChain()`、四类结构校验错误、运行时 `JumpConnectOptions` 与不含秘密的活动连接链摘要；未设置最大深度。
  - **资源列表**：`src/connection/JumpChain.ts`、`src/remote/RemoteSessionTypes.ts`。
  - **验证结果**：`npm ci` 成功；`npm run compile` 成功；`git diff --check` 成功；人工核对空链、链方向、迭代遍历与四类错误分支通过。

---

### 2. 扩展连接配置、凭据解析与建连参数

- [x] **数据与服务** - ConnectionManager - 持久化 jump 并构造完整安全连接链
  - **执行前上下文**：执行前必须读取并遵守 `执行前强制上下文`；若当前上下文未包含该段，先回读 checklist 文件顶部。
  - **目标编号**：2
  - **目标名称**：扩展连接配置、凭据解析与建连参数
  - **所属界面组**：连接配置
  - **导航分组**：无
  - **目标文件**：
    - `src/connection/ConnectionManager.ts` - 扩展 profile/input、规范化、保存校验和完整 `ConnectOptions` 构建。
    - `src/connection/JumpChain.ts` - 接入任务 1 类型所需的最小调整。
    - `src/remote/RemoteSessionTypes.ts` - 接入已解析 chain 所需的最小调整。
  - **硬边界**：只处理连接配置、凭据和建连参数；本任务不实现备份版本、删除保护、SSH `forwardOut` 运行时或 UI。
  - **确认依赖**：无
  - **范围说明**：为 `ConnectionProfile` 与 `ConnectionProfileInput` 增加可选 `jumpProfileId`；历史空值规范为直连；SFTP 保存时基于最终 profile 集合校验引用，FTP/FTPS 保存时清除 jump；`buildConnectOptions()` 解析最外层到最近层 chain，并只在扩展宿主读取每跳 SecretStorage 凭据。缺失 jump 密码或必要口令时使用 VS Code 原生安全输入，取消输入则取消整次建连。
  - **优先级**：P0
  - **前置任务**：任务 1
  - **参考文档**：
    - `docs/prompts/sftp-multi-hop-jump.md` - 第 4.1、4.3、5、6、8 节。
  - **参考代码**：
    - `src/connection/ConnectionManager.ts` - `saveProfile()`、`listProfiles()`、`buildConnectOptions()`、SecretStorage helper。
    - `src/utils/localPathUtils.ts` - 私钥路径展开参考。
    - `src/utils/progressUtils.ts` - 取消错误类型。
  - **接口依赖**：
    - `vscode.window.showInputBox` - 未保存 jump 密码/口令的安全输入。
  - **类型依赖**：
    - `ConnectionProfile`
    - `ConnectionProfileInput`
    - `JumpConnectOptions`
    - `ConnectOptions`
  - **规范清单**：
    - profile 快照只暴露 `hasSavedPassword` / `hasSavedPassphrase`，不得附带秘密。
    - jump chain 中每跳使用自身 host、port、username、authType、key path、keepalive 和秘密。
    - 快速连接可以引用保存的 jump，但不能成为被引用对象。
    - 无效引用不得在连接时静默忽略。
  - **预期产出**：
    - `src/connection/ConnectionManager.ts` - 完整 jump 持久化与安全 chain 参数构建。
  - **完成标准**：
    - 旧 profile 读取后仍为直连，保存/读取 jump ID 可稳定往返。
    - 多跳链凭据仅存在于扩展宿主内的 `ConnectOptions`。
    - FTP/FTPS 输入不会产生 jump chain。
    - 缺失凭据和取消输入具有明确行为。
    - `npm run compile` 通过。
  - **完成时间**：2026-08-27 08:44 CST
  - **实际产出**：
    - `src/connection/ConnectionManager.ts` - 增加 SFTP-only jump ID 归一化与往返、保存前最终图校验、saved/quick 目标链解析、逐跳 SecretStorage 补全及缺失密码安全提示。
    - `src/connection/JumpChain.ts` - 允许轻量 quick target 作为解析起点，同时保持返回的已保存 profile 强类型。
  - **验证结果**：
    - ✓ `npm run compile` 与 `git diff --check` 成功。
    - ✓ 隔离验证覆盖三跳顺序、旧值直连、FTP 强制直连、quick 引用、秘密不进入 profile、一次性密码不落盘、取消输入及无效保存不写入。
    - ✓ 未保存私钥口令保持未填充，由任务 4 加载密钥确认确实加密后再安全提示，避免未加密私钥误弹窗。

---

### 3. 完成引用保护与备份版本兼容

- [x] **数据兼容** - ConnectionManager - 保护反向依赖并升级备份契约
  - **执行前上下文**：执行前必须读取并遵守 `执行前强制上下文`；若当前上下文未包含该段，先回读 checklist 文件顶部。
  - **目标编号**：3
  - **目标名称**：完成引用保护与备份版本兼容
  - **所属界面组**：连接管理、备份
  - **导航分组**：无
  - **目标文件**：
    - `src/connection/ConnectionManager.ts` - 删除/协议变更保护、备份 v3 导入导出和最终集合校验。
  - **硬边界**：只处理反向引用和备份契约；不修改 SSH 会话、Webview、侧边栏或测试文件。
  - **确认依赖**：无
  - **范围说明**：阻止删除仍被其他 profile 引用的 jump，阻止被引用 SFTP profile 改为 FTP/FTPS；扩展 `RemoteEditBackupConnection`，将备份版本提升到 3；旧版本缺失 jump 时按直连导入；合并和替换均在最终 profile 集合形成后统一校验引用、协议和循环。
  - **优先级**：P0
  - **前置任务**：任务 2
  - **参考文档**：
    - `docs/prompts/sftp-multi-hop-jump.md` - 第 4.4、5.3、8、9 节。
  - **参考代码**：
    - `src/connection/ConnectionManager.ts` - `deleteProfile()`、`buildBackupFile()`、`importBackupFile()`、`normalizeBackupConnections()`。
  - **接口依赖**：无
  - **类型依赖**：
    - `RemoteEditBackupConnection`
    - `RemoteEditBackupFile`
    - `ConnectionProfile`
  - **规范清单**：
    - 删除或协议变更错误应列出或概述直接依赖 profile。
    - 导入不得依赖备份数组顺序。
    - 合并导入依据合并后的最终集合，替换导入依据导入后的最终集合。
    - 导入无效引用不得自动改为直连。
  - **预期产出**：
    - `src/connection/ConnectionManager.ts` - 安全的引用变更与 v3 备份往返。
  - **完成标准**：
    - 被引用 jump 的删除和协议降级被明确拒绝。
    - v3 备份保留 jump，旧备份可作为直连导入。
    - 合并/替换导入均能发现缺失、非 SFTP 和循环引用。
    - `npm run compile` 通过。
  - **完成时间**：2026-08-27 08:58 CST
  - **实际产出**：
    - `src/connection/ConnectionManager.ts` - 增加直接依赖名称提示与删除/协议降级/组批量删除保护；升级无密备份 v3；按版本归一化 jump；在 merge/replace 最终集合形成后、任何导入写入前统一校验。
  - **验证结果**：
    - ✓ `npm run compile` 与 `git diff --check` 成功。
    - ✓ v3 无序往返及 v2 直连兼容通过；备份连接不含密码或口令。
    - ✓ merge/replace 均拒绝缺失引用、FTP jump 和循环，失败前后 profile、group 与 SecretStorage 保持不变。
    - ✓ 被引用 profile 的删除、协议降级和组级批量删除均被拒绝并显示依赖连接。

---

## SSH 运行时

### 4. 实现多级 SSH ForwardOut 会话与清理

- [x] **运行时** - SSH/SFTP - 逐级建立、取消和释放专属 Jump 链
  - **执行前上下文**：执行前必须读取并遵守 `执行前强制上下文`；若当前上下文未包含该段，先回读 checklist 文件顶部。
  - **目标编号**：4
  - **目标名称**：实现多级 SSH ForwardOut 会话与清理
  - **所属界面组**：无
  - **导航分组**：无
  - **目标文件**：
    - `package.json` - 将 `ssh2` 声明为直接运行时依赖。
    - `package-lock.json` - 锁定直接依赖关系。
    - `src/ssh/SshJumpChain.ts` - 新增可测试的中间 SSH 建连、forward 与资源容器。
    - `src/ssh/SftpSessionManager.ts` - 接入 jump runtime、最终 `sock`、取消和断开清理。
    - `src/remote/RemoteSessionTypes.ts` - 运行时资源接入所需的最小类型调整。
  - **硬边界**：只实现 SSH/SFTP 运行时和依赖；不修改连接 UI、侧边栏、备份或文档。
  - **确认依赖**：无
  - **范围说明**：使用原始 `ssh2.Client` 直连最外层 jump，随后逐级 `forwardOut()` 到下一跳并以 stream 作为下一 `Client.connect({ sock })`；最终将最近一跳 stream 作为 `ssh2-sftp-client.connect()` 的 `sock`。直接连接保持现有探测；jump 连接只探测最外层。按最终 connection ID 持有中间客户端和 stream，并覆盖成功、失败、取消、重连、断开和 disconnectAll 清理。
  - **优先级**：P0
  - **前置任务**：任务 3
  - **参考文档**：
    - `docs/prompts/sftp-multi-hop-jump.md` - 第 4.2、4.5、6、8、9 节。
  - **参考代码**：
    - `src/ssh/SftpSessionManager.ts` - 当前直连、认证和关闭逻辑。
    - `src/ssh/PortForwardManager.ts` - `forwardOut()` 与 stream 生命周期参考。
    - `src/remote/ConnectionProbe.ts` - 本机 TCP 探测。
    - `package-lock.json` - 当前 `ssh2@1.17.0` 解析结果。
  - **接口依赖**：
    - `ssh2.Client.connect()`
    - `ssh2.Client.forwardOut()`
    - `ssh2-sftp-client.connect()`
  - **类型依赖**：
    - `JumpConnectOptions`
    - `ConnectionCancellationToken`
    - `ActiveConnection`
  - **规范清单**：
    - 中间节点不注册活动连接、不运行平台探测、不打开 SFTP。
    - keepalive、ready timeout 与认证配置应用到每一级 SSH。
    - 日志和错误指出失败 profile、host 和阶段，且不包含秘密。
    - 清理应幂等并按目标向外层逆序进行。
    - 直连 SFTP 路径不得产生行为变化。
  - **预期产出**：
    - `src/ssh/SshJumpChain.ts` - 独立 jump runtime。
    - `src/ssh/SftpSessionManager.ts` - 支持直接和多跳的统一最终 SFTP 会话。
    - `package.json` / `package-lock.json` - 显式 `ssh2` 依赖。
  - **完成标准**：
    - `A -> B -> C` 按 C、B、A 顺序建立 SSH/SFTP，且最终只暴露 A。
    - 目标和中间 host 不需要本机可达或可解析。
    - 任一阶段取消或失败均无残留客户端、stream 或活动状态。
    - 直接连接、终端与已有端口转发仍能取得最终目标底层 SSH client。
    - `npm install` 不产生意外依赖漂移，`npm run compile` 通过。
  - **完成时间**：2026-08-27 09:23 CST
  - **实际产出**：
    - `src/ssh/SshJumpChain.ts` - 新增独立、可注入测试依赖的多级 SSH runtime；仅探测最外层，使用 `forwardOut()` direct-tcpip 内部流逐级传递 `sock`，不监听本地端口、不创建可配置端口映射；支持逐跳认证、加密私钥口令提示、取消及严格逆序幂等清理。
    - `src/ssh/SftpSessionManager.ts` - 最终 SFTP 接入最近跳 stream，按最终 connection ID 持有隐藏链；覆盖初始化失败、取消、重连、远端关闭、单次断开和全部断开的资源释放；安全填充 jump 摘要，并保持最终目标 raw SSH client 的终端/端口转发访问路径。
    - `package.json` / `package-lock.json` - 增加直接 `ssh2@^1.17.0` 运行时依赖，复用既有锁定的 `ssh2@1.17.0`。
  - **资源列表**：`src/ssh/SshJumpChain.ts`、`src/ssh/SftpSessionManager.ts`、`package.json`、`package-lock.json`。
  - **验证结果**：
    - ✓ `npm run compile`、`git diff --check` 通过；`npm install --package-lock-only --ignore-scripts` 未产生额外漂移，依赖树只有一个去重后的 `ssh2@1.17.0`。
    - ✓ fake-client 隔离验证覆盖 C→B→A 建连顺序、前一跳 stream 作为下一跳 `sock`、仅 C 本机探测、逐跳 keepalive/认证、最终流交付和目标侧向外层严格逆序清理。
    - ✓ 覆盖中间连接失败、转发期间取消、迟到 stream、重复 dispose、错误不含凭据、缺口令按需提示与非法 key 不提示；无客户端、stream、监听器或取消订阅残留。
    - ✓ 64 跳有限无环链迭代验证通过；源码扫描确认新 runtime 不调用 `createServer` / `listen`，不引用端口映射配置或 `PortForwardManager`。
    - △ 当前工作区没有可用的多主机隔离 SSH 拓扑，因此未声称真实网络集成已执行；任务 7 保留外部拓扑验收入口。

---

## 用户界面

### 5. 在 Webview 连接表单接入 Jump 选择

- [x] **Webview** - 连接配置 - 添加候选过滤、链摘要和 payload 往返
  - **执行前上下文**：执行前必须读取并遵守 `执行前强制上下文`；若当前上下文未包含该段，先回读 checklist 文件顶部。
  - **目标编号**：5
  - **目标名称**：在 Webview 连接表单接入 Jump 选择
  - **所属界面组**：Remote Edit Webview
  - **导航分组**：Connection details
  - **目标文件**：
    - `src/panel/webview/markup/Body.ts` - 增加 Jump Host 表单控件。
    - `src/panel/webview/scripts/StateDialogs.ts` - 注册 DOM 引用和交互状态。
    - `src/panel/webview/scripts/RemoteSearch.ts` - dirty snapshot、校验和 payload 增加 jump。
    - `src/panel/webview/scripts/RemoteCommandActions.ts` - 候选渲染、profile 填充/清空和类型切换。
    - `src/panel/webview/scripts/TransferContextActions.ts` - Jump 控件事件与关闭行为。
    - `src/panel/webview/scripts/LayoutSessions.ts` - 表单输入状态与 profile 列表变化同步。
    - `src/panel/webview/scripts/TransfersStatus.ts` - 连接中/已连接时的控件锁定。
    - `src/panel/webview/styles/Styles.ts` - Jump 选择及链摘要样式。
    - `src/panel/RemoteEditPanel.ts` - 连接/活动状态所需的安全 jump 摘要传递与日志。
  - **硬边界**：只处理 Webview 入口和安全展示；不修改侧边栏、SSH runtime、备份或测试文件。
  - **确认依赖**：无
  - **范围说明**：SFTP 表单显示 Direct/已保存 SFTP profile 候选；排除自身和会形成循环的候选，但允许候选自身包含 jump；显示链摘要；保存与快速连接 payload 只发送 `jumpProfileId`。FTP/FTPS 隐藏或禁用并清空 jump。profile 更新、选择、清空、连接锁定和 dirty comparison 都需保持一致。
  - **优先级**：P1
  - **前置任务**：任务 4
  - **参考文档**：
    - `docs/prompts/sftp-multi-hop-jump.md` - 第 3.1、4.1、5.2、6、8 节。
  - **参考代码**：
    - `src/panel/webview/markup/Body.ts` - 现有连接类型和认证 picker。
    - `src/panel/webview/scripts/RemoteCommandActions.ts` - profile picker 模式。
    - `src/panel/webview/scripts/RemoteSearch.ts` - payload 与 snapshot。
    - `src/panel/RemoteEditPanel.ts` - ProfilesLoaded 和 connect。
  - **接口依赖**：
    - `profilesLoaded` Webview 消息。
    - `saveConnection` / `connect` payload。
  - **类型依赖**：
    - `ConnectionProfile.jumpProfileId`
    - `ActiveConnection.jumpProfileIds`
  - **规范清单**：
    - 候选可用性由 UI 提示，但扩展后端保持最终校验。
    - 不把 chain 的任何凭据发送给 Webview。
    - Direct 作为明确选项，空值必须稳定往返。
    - 锁定和 dirty 状态包含 jump，避免误保存或漏提示。
  - **预期产出**：
    - Webview 可完整配置并观察 SFTP jump 链。
  - **完成标准**：
    - 新建、编辑、快速连接、类型切换和 profile 刷新均正确处理 jump。
    - 自身、非 SFTP 和循环候选不可选；合法多跳候选可选。
    - 连接后可看到不含秘密的链路摘要或 via 信息。
    - `npm run compile` 通过。
  - **完成时间**：2026-08-27 09:57 CST
  - **实际产出**：
    - 新增全宽 Jump Host picker 与 Direct 空值；仅展示合法 SFTP profile，使用无深度上限的迭代 `visited` 分析排除自身、缺失、非 SFTP 和任意循环，同时允许候选自身继续 Jump。
    - Jump ID 已覆盖保存/快速连接 payload、profile/form dirty 快照、profile 与 quick-session 往返、连接匹配、类型切换、profile 刷新、dropdown 互斥及连接状态锁定；失效 ID 保留并阻止建连，不静默退化为直连。
    - 表单、活动文本和 session tooltip 使用最外层到最近层的安全名称显示 `Local → ... → Target` / `via ...`；Webview 不接收凭据或运行时 `jumpChain`，宿主日志只增加名称级 `Via`。
    - 该 UI 只提交 `jumpProfileId` 给任务 4 runtime；未引入本地监听、可配置端口映射或 `PortForwardManager` 接入。
  - **资源列表**：`src/panel/RemoteEditPanel.ts`、`src/panel/webview/markup/Body.ts`、`src/panel/webview/scripts/StateDialogs.ts`、`src/panel/webview/scripts/RemoteSearch.ts`、`src/panel/webview/scripts/RemoteCommandActions.ts`、`src/panel/webview/scripts/TransferContextActions.ts`、`src/panel/webview/scripts/LayoutSessions.ts`、`src/panel/webview/scripts/TransfersStatus.ts`、`src/panel/webview/styles/Styles.ts`。
  - **验证结果**：
    - ✓ `npm run compile`、完整生成脚本解析和 `git diff --check` 成功。
    - ✓ 聚焦验证覆盖嵌套链方向、合法 quick 多跳、自身/缺失/非 SFTP/循环过滤、首次 profile 填充、失效值保留、FTP 清空及 128 跳有限无环链。
    - ✓ 新增行安全扫描未发现凭据 payload、本地监听、端口映射或 `PortForwardManager`；工作树范围仅为本任务九个源码文件与 checklist。
    - △ 未使用真实多主机 SSH 拓扑；不虚报外部网络验证，留待任务 7 的可复现验收步骤。

---

### 6. 在原生侧边栏接入 Jump 详情与编辑

- [x] **Sidebar** - 连接配置 - 扩展草稿、详情项和 QuickPick 编辑流程
  - **执行前上下文**：执行前必须读取并遵守 `执行前强制上下文`；若当前上下文未包含该段，先回读 checklist 文件顶部。
  - **目标编号**：6
  - **目标名称**：在原生侧边栏接入 Jump 详情与编辑
  - **所属界面组**：Native Sidebar
  - **导航分组**：Saved Connections / Quick Connect
  - **目标文件**：
    - `src/sidebar/ConnectionDraftStore.ts` - 草稿、合并、快速连接和类型切换保留 jump。
    - `src/sidebar/ItemHelpers.ts` - 增加 Jump Host 详情字段和安全显示。
    - `src/sidebar/Items.ts` - tooltip/详情摘要接入 jump 信息。
    - `src/sidebar/TreeProviders.ts` - 为保存连接和快速连接呈现 jump 详情。
    - `src/sidebar/SidebarController.ts` - 使用 QuickPick 选择 Direct 或合法 jump，保存并建连。
  - **硬边界**：只处理原生侧边栏；不修改 Webview、SSH runtime、备份或测试文件。
  - **确认依赖**：无
  - **范围说明**：将 `jumpProfileId` 纳入新建/已有/快速连接草稿与保存 payload；SFTP 详情增加 Jump Host 行，点击后通过 QuickPick 选择 Direct 或合法候选，并展示候选完整链摘要；过滤规则和后端一致；切到 FTP/FTPS 时清空；活动连接 tooltip 可显示 via 链。
  - **优先级**：P1
  - **前置任务**：任务 5
  - **参考文档**：
    - `docs/prompts/sftp-multi-hop-jump.md` - 第 3.1、4.1、5.2、6、8 节。
  - **参考代码**：
    - `src/sidebar/ConnectionDraftStore.ts` - 现有草稿规范化。
    - `src/sidebar/ItemHelpers.ts` - `ConnectionDetailField` 与详情构建。
    - `src/sidebar/SidebarController.ts` - 连接字段编辑和保存。
  - **接口依赖**：
    - `vscode.window.showQuickPick` - Jump Host 选择。
  - **类型依赖**：
    - `ConnectionDetailField`
    - `ConnectionProfileInput`
    - `ConnectionProfile`
  - **规范清单**：
    - Quick Connect 可以选择保存 profile，但自身不能成为候选。
    - 未保存草稿不能形成可被其他连接引用的 jump。
    - 候选标签区分名称与 host，并展示已有链摘要。
    - 草稿 dirty 状态和保存/放弃行为覆盖 jump 字段。
  - **预期产出**：
    - 原生侧边栏可完整配置、保存、查看和使用 jump。
  - **完成标准**：
    - 保存连接与 Quick Connect 均能从侧边栏通过合法多跳链连接。
    - Direct、切换协议、保存、放弃和重载行为一致。
    - 详情与 tooltip 不泄露凭据。
    - `npm run compile` 通过。
  - **完成时间**：2026-08-27 10:31 CST
  - **实际产出**：
    - `jumpProfileId` 已覆盖新建、已有和 Quick Connect 草稿的规范化与重建；Direct 使用显式空串清除，FTP/FTPS 强制直连，保存、放弃、重载、私钥认证以及 Connect Without Saving 的完整 profile 合并保持一致。
    - SFTP 详情新增 Jump Host 行；guided add 与详情编辑使用 Direct-first QuickPick，只列出解析后合法的已保存 SFTP profile，并区分名称、endpoint 与 `Local → ... → Target` 完整路线；解析复用无深度上限的迭代 resolver。
    - 保存/Quick tooltip 展示安全 route，活动连接只使用 names-only `jumpProfileNames` 展示 `Via`；动态 Markdown 保持不可信并转义，侧边栏只传 ID，不接收凭据或运行时 chain。
    - 未增加本地监听、可配置端口映射、`PortForwardManager` 接入或可见 Jump 会话；选择 Jump 后仍由现有 SSH runtime 自动建立内部 `forwardOut()`/`sock` 链。
  - **资源列表**：`src/sidebar/ConnectionDraftStore.ts`、`src/sidebar/ItemHelpers.ts`、`src/sidebar/Items.ts`、`src/sidebar/TreeProviders.ts`、`src/sidebar/SidebarController.ts`。
  - **验证结果**：
    - ✓ `npm run compile` 与 `git diff --check` 成功。
    - ✓ 46 项隔离断言覆盖保存/Quick/新草稿、Direct、协议往返、保存基础值合并、放弃/重载、私钥上下文、自身/缺失/非 SFTP/循环过滤、嵌套 route、128 跳无环链及 tooltip 转义。
    - ✓ 精确 scope 与新增行安全扫描未发现本地监听、端口映射、runtime `jumpChain`、凭据数据字段或深度上限；`.cursorrules` 与 `.temp/` 未纳入提交。
    - △ 未虚报真实 VS Code 点击流程或多主机 SSH 拓扑；可复现自动测试和外部拓扑说明留待任务 7。

---

## 验证与交付

### 7. 补充自动验证、回归说明与最终验收

- [x] **测试与文档** - Jump 链 - 覆盖核心边界并完成全量回归
  - **执行前上下文**：执行前必须读取并遵守 `执行前强制上下文`；若当前上下文未包含该段，先回读 checklist 文件顶部。
  - **目标编号**：7
  - **目标名称**：补充自动验证、回归说明与最终验收
  - **所属界面组**：无
  - **导航分组**：无
  - **目标文件**：
    - `package.json` - 增加可重复执行的测试脚本或测试入口。
    - `src/test/JumpChain.test.ts` - 覆盖解析、方向、无限深语义和错误边界。
    - `src/test/SshJumpChain.test.ts` - 使用受控替身覆盖逐跳顺序、失败、取消和幂等清理。
    - `README.md` - 说明 SFTP Jump Host 配置、协议限制和多跳行为。
    - `CHANGELOG.md` - 记录新增能力与兼容边界。
  - **硬边界**：只增加验证、必要测试入口与用户文档；发现实现缺陷时可回到对应任务目标文件做最小修复，但必须在完成记录中列明。
  - **确认依赖**：无
  - **范围说明**：建立不依赖真实生产服务器的自动测试，覆盖空链、单跳、至少三跳、较长无环链、自引用、长循环、缺失/非 SFTP引用、顺序、错误阶段、取消和资源清理；完成 compile/test/diff 检查；记录可复现的隔离网络手工验证步骤，真实环境不可用时明确标记未执行，不虚报成功。
  - **优先级**：P1
  - **前置任务**：任务 6
  - **参考文档**：
    - `docs/prompts/sftp-multi-hop-jump.md` - 第 8 节验收标准。
  - **参考代码**：
    - `src/connection/JumpChain.ts` - 纯解析逻辑。
    - `src/ssh/SshJumpChain.ts` - SSH runtime 和资源所有权。
    - `package.json` / `tsconfig.json` - 当前构建方式。
  - **接口依赖**：
    - Node.js `node:test` / `assert`，优先避免引入额外测试框架。
  - **类型依赖**：
    - `JumpConnectOptions`
    - Jump runtime 的可注入客户端/stream 边界。
  - **规范清单**：
    - 自动测试不得连接真实外部主机或读取真实 SecretStorage。
    - “无最大深度”通过较长有限链证明算法无固定阈值，并检查源码无深度常量。
    - 资源清理验证覆盖连接中途失败、用户取消和正常断开。
    - 文档明确 FTP/FTPS 不支持 jump。
    - 最终检查不提交 `out/`、`.temp/`、凭据或 `.cursorrules`。
  - **预期产出**：
    - `src/test/JumpChain.test.ts` - 图解析回归测试。
    - `src/test/SshJumpChain.test.ts` - 运行时顺序和清理回归测试。
    - `README.md` / `CHANGELOG.md` - 用户可见说明。
  - **完成标准**：
    - `npm run compile` 通过。
    - 新增自动测试全部通过并能通过单一 npm script 重复执行。
    - `git diff --check` 通过，工作区只包含预期文件。
    - 直接 SFTP 与 FTP/FTPS 路径没有 jump 行为回归。
    - 手工/隔离拓扑验证结果如实记录；没有可用 SSH 环境时明确列为未执行的外部验证，而不影响自动验证结论。
  - **完成时间**：2026-08-27 11:01 CST
  - **实际产出**：
    - `package.json` 新增单一 `npm test` 入口，先编译再显式运行两份输出测试；未增加测试框架、依赖或 lockfile 变更。
    - 图解析测试覆盖 Direct、单跳、`D → C → B` 三跳方向、512 跳有限无环链、自引用、目标/内部/96 节点长循环、缺失和 FTP/FTPS 引用，并检查迭代 `visited`/`while` 与无深度上限常量。
    - 运行时测试使用受控 client/stream/token 驱动真实 `SshJumpChain`，覆盖最外层探测、三跳 `sock` 顺序、`127.0.0.1:0` 内部 direct-tcpip 流、逐跳认证/keepalive、阶段错误、连接/转发取消、延迟 stream、逆序幂等清理以及 Direct/FTP/FTPS 隔离；仅使用明确标注的 synthetic fixture，不读取真实 SecretStorage。
    - README 新增双 UI 配置、Direct、多跳顺序、无可见预连接/本地监听/可配置端口映射、SFTP-only 限制及 `D → C → B → A` 隔离拓扑步骤；CHANGELOG 新增 Unreleased 说明，并如实标注外部拓扑未执行。
  - **资源列表**：`package.json`、`src/test/JumpChain.test.ts`、`src/test/SshJumpChain.test.ts`、`README.md`、`CHANGELOG.md`。
  - **验证结果**：
    - ✓ `npm test` 成功，15 项测试全部通过、0 失败；`npm run compile` 独立复跑成功。
    - ✓ `git diff --check`、精确 scope、lockfile 与生成物检查成功；`package-lock.json` 未变化，`out/` 未跟踪。
    - ✓ 静态契约检查确认无本地 listener、可配置 Jump 端口映射、`PortForwardManager` 耦合或深度上限；直接 SFTP 保持原探测分支，FTP/FTPS 不进入 Jump runtime。
    - ✓ 凭据扫描只发现 synthetic fixture 与既有安全文档，错误断言确认不泄露 fixture 密码；`.cursorrules` 与 `.temp/` 不纳入提交。
    - △ 当前环境没有可用的隔离多主机 SSH 拓扑，因此真实网络与文件操作验证未执行；README 已提供可复现步骤且未虚报结果。

---
