# SFTP 多级 Jump 连接链

## 1. 业务背景与目标

当前 Remote Edit 的连接配置只能从 VS Code 扩展宿主直接连接目标服务器。部分目标服务器只允许经由堡垒机或中间 SSH 主机访问，因此需要让每个 SFTP 连接可以选择一条已经保存的 SFTP 连接作为 jump。

这里的“一条 jump”是单个连接配置至多保存一个 `jumpProfileId`，不是整条链只能有一层。被选作 jump 的连接可以继续配置自己的 jump，从而形成任意深度的有向连接链；本功能不设置人为最大深度，但必须在建连前可靠识别自引用和循环引用。

完成后，用户只需连接最终目标，扩展应自动解析并建立完整 SSH 链，不要求用户预先手动打开任何 jump 连接。

## 2. 当前现状与问题

- 连接资料由 `ConnectionManager` 统一保存到 VS Code `globalState`，密码和私钥口令保存在 `SecretStorage`。
- `ConnectionManager.buildConnectOptions()` 当前只解析最终目标的连接参数和凭据。
- `SftpSessionManager.connect()` 当前先从本机探测目标 TCP 地址，再由 `ssh2-sftp-client` 直接连接目标。
- 项目已经通过 `ssh2` 的 `Client.forwardOut()` 实现活动 SSH 会话上的端口转发，证明底层依赖具备 SSH connection hopping 所需能力。
- `forwardOut()` 是 `ssh2` 库提供的 API；现有源码只是将其用于端口转发功能，并未把它接入连接配置、连接表单或 SFTP 建连流程，因此当前 UI 没有 Jump Host 选项。
- `ssh2` 当前由 `ssh2-sftp-client` 间接依赖；实现 jump 时会在运行时直接创建 `ssh2.Client`，因此应改为项目的直接依赖。
- FTP/FTPS 使用独立控制连接和动态被动数据连接。现有 `basic-ftp` 会为数据传输直接创建新 socket，单独转发控制端口无法可靠覆盖完整 FTP 流程。

## 3. 本次范围

### 3.1 范围内

- 为保存的 SFTP 连接增加可选 jump 配置，值来自现有的已保存 SFTP 连接。
- 允许 Webview 中的新建/快速连接选择已保存 SFTP 连接作为 jump。
- 允许侧边栏中的保存连接和快速连接查看、选择、修改或清除 jump。
- 允许 jump 连接继续引用另一个 jump，递归形成不设最大深度的多级链。
- 在扩展宿主中解析整条 jump 链，读取每一跳的认证配置和安全凭据，并按从最外层 jump 到最终目标的顺序建立连接。
- 对自引用、循环引用、缺失引用、非 SFTP jump、无效认证和连接失败进行明确校验与错误反馈。
- 在取消、失败、断开或扩展释放资源时，完整关闭最终 SFTP 会话、各级转发流和中间 SSH 客户端。
- 在连接详情、活动连接状态或日志中提供足够的 jump 链信息，便于用户确认实际连接路径和定位失败节点。
- 将 jump 引用纳入连接备份导入/导出，并兼容没有 jump 字段的历史连接数据和旧版备份。
- 保持现有直接 SFTP 连接、SSH Terminal、端口转发、文件操作和服务器工具行为不回归。

### 3.2 非范围

- FTP 或 FTPS 目标通过 jump 连接。
- FTP 或 FTPS 连接充当 jump。
- 支持 OpenSSH `~/.ssh/config` 中的 `ProxyJump`、`ProxyCommand` 或主机别名解析。
- 新增当前项目尚未支持的认证方式，例如 SSH Agent、keyboard-interactive 或证书认证。
- 多个最终目标共享同一个隐式 jump SSH 客户端或对中间连接做连接池复用。
- 将隐式建立的中间 jump 会话作为普通活动连接展示或开放文件浏览。

## 4. 业务流程与场景

### 4.1 配置连接

- 当连接类型为 SFTP 时，用户可以在连接表单或侧边栏详情中选择“Direct”或一条已保存的 SFTP 连接作为 jump。
- 候选项不得包含当前连接自身、FTP/FTPS 连接，以及选择后会形成循环的连接。
- 候选 jump 即使已经配置了自己的 jump 也仍可选择；UI 可展示其链路摘要，帮助用户理解最终路径。
- 当目标连接改为 FTP/FTPS 时，jump 不可编辑，并在保存时清除 `jumpProfileId`。

### 4.2 建立多级连接

- 连接前先在扩展宿主中解析完整链并完成结构校验，不应边连接边发现循环。
- 对于 `目标 A -> jump B -> jump C`：先从本机直连 C；通过 C 的 `forwardOut()` 创建到 B 的双向流，并以该流作为 B 的 `sock` 建立 SSH；再通过 B 创建到 A 的双向流，最终以该流作为 `ssh2-sftp-client` 的 `sock` 建立 A 的 SFTP 会话。
- 中间 jump 只需建立原始 SSH 客户端，不执行 SFTP 初始化、远端平台探测、起始目录解析或普通活动连接注册。
- 目标主机名和端口应由其前一跳发起连接，不能要求 VS Code 所在机器能够直接解析或访问目标地址。
- 最终目标成功连接后，扩展对外仍只注册最终目标这一条活动连接。

### 4.3 凭据处理

- 每一跳均使用其自身保存的用户名、密码或私钥配置。
- 密码和私钥口令只在扩展宿主中从 `SecretStorage` 读取，不发送到 Webview，也不持久化进普通连接数据或日志。
- 如果某一跳使用密码但未保存密码，连接时通过 VS Code 原生密码输入框提示用户；私钥确实需要口令但未保存时，也按失败节点提示输入口令后重试该次建连。
- 用户取消任一凭据输入或连接进度时，应取消整条链并释放已经建立的资源。

### 4.4 配置变更与删除

- 重命名连接不影响引用，因为 jump 使用稳定的 profile ID。
- 被其他连接引用为 jump 的连接默认不允许直接删除；错误信息应列出或概述依赖它的连接，用户先移除引用后再删除。
- 被其他连接引用为 jump 的 SFTP 连接默认不允许改成 FTP/FTPS。
- 如果因备份、历史数据或异常状态出现缺失 jump 引用，连接时必须明确报错，不能静默退化为直连。

### 4.5 失败、取消与断开

- 任一中间 SSH 连接、`forwardOut()` 或最终 SFTP 握手失败时，错误信息应指出失败的 profile 名称、主机和所处阶段。
- 建连失败或取消时，从最终尝试向最外层 jump 逆序关闭已创建的 SFTP 客户端、SSH 客户端和转发流。
- 最终目标主动断开时，同时释放其专属的完整 jump 链；断开某个普通可见连接不应误伤其他目标的隐式链。

## 5. 数据、接口与契约

### 5.1 数据结构

- `ConnectionProfile.jumpProfileId?: string`
  - 保存被引用 jump 的稳定 profile ID。
  - `undefined` 或空值表示直连。
- `ConnectionProfileInput.jumpProfileId?: string`
  - 接收 Webview、侧边栏草稿、快速连接和保存流程传入的 jump 选择。
- `RemoteEditBackupConnection.jumpProfileId?: string`
  - 备份连接之间的引用关系。
- `ConnectOptions`
  - 最终建连参数需要携带已解析的 jump 链，建议使用专用的内部 `JumpConnectOptions[]`，不要让会持久化或发往 Webview 的对象包含明文秘密。
  - 链条顺序必须在类型注释或命名中固定，避免调用方对“目标到外层”与“外层到目标”的方向产生歧义。
- `ActiveConnection`
  - 可携带不含凭据的 `jumpProfileId`、jump 名称列表或链路摘要，用于状态和展示；不得包含密码、私钥内容或私钥口令。
- SFTP 会话管理器需要按最终 `connectionId` 保存该目标专属的中间 SSH 客户端和转发流集合，用于生命周期清理。

### 5.2 接口契约

- 本功能不新增网络服务接口或外部 API。
- Webview 保存/连接 payload 增加可选 `jumpProfileId`，ProfilesLoaded 中现有 profile 快照可作为候选数据源。
- 侧边栏草稿与详情字段增加 jump 选择，但只传递 profile ID。
- 扩展宿主负责将 profile ID 解析为连接参数和凭据；Webview 不负责构造 jump 链。

### 5.3 转换与兼容策略

- 没有 `jumpProfileId` 的现有 profile 继续按直连处理，不需要一次性迁移。
- 读取持久化数据时，将空字符串、只含空白或非字符串 jump 值规范为 `undefined`。
- 保存 SFTP profile 时校验引用存在、类型正确且不会形成循环。
- 保存 FTP/FTPS profile 时将 jump 规范为 `undefined`。
- 备份格式版本默认从 2 提升到 3；新版继续接受旧版本备份，并将缺失 jump 字段解释为直连。
- 导入时应在所有 profile 完成规范化和合并后统一校验引用，避免因文件中的排列顺序误判合法链路。
- 合并导入需要基于最终 profile 集合校验 jump；替换导入只允许引用同一份最终导入集合中的 profile。

## 6. 实现约束与规则

- 不设置 jump 最大深度常量。实际链长仅受有限 profile 数量和系统资源约束。
- 链解析应使用 `visited` 集合检测循环；优先采用迭代方式，避免极长但合法的链造成 JavaScript 调用栈溢出。
- jump 必须是已保存的 SFTP profile；快速连接可以使用保存的 profile 作为 jump，但不能被其他 profile 引用为 jump。
- 中间连接使用 `ssh2.Client`；最终文件会话继续使用 `ssh2-sftp-client`。
- 将 `ssh2` 以与 lockfile 已解析版本兼容的版本加入直接运行时依赖，不依赖传递依赖的存在。
- 只有最外层 jump 执行本机 TCP 可达性探测。后续节点已经通过 `sock` 建立，不能再调用当前的本机直连探测。
- 每一级应应用其自身的 SSH ready timeout、认证信息和 keepalive 配置。
- 每个最终目标默认拥有独立的隐式 jump 链，不复用 UI 中已打开的 jump 会话，避免可见连接的断开行为影响其他目标。
- 所有事件监听器、取消订阅、SSH 客户端和流都必须在成功移交或异常路径中有明确所有者和释放点。
- 日志可以记录 profile 名称、host、port、层级和阶段，但不得记录密码、私钥内容、私钥口令或完整含密配置对象。
- 错误归一化需要区分：最外层 TCP 不可达、某级 SSH 认证失败、某级 `forwardOut` 失败、最终 SFTP 初始化失败。
- UI 与后端都要校验候选，但以后端校验为最终可信边界。
- FTP/FTPS 的 UI 不显示或禁用 jump；即使收到手工构造的 `jumpProfileId`，后端也不得尝试为其建立 jump。

## 7. 参考资源

- `package.json` - 当前 `basic-ftp`、`ssh2-sftp-client` 依赖以及需要补充的 `ssh2` 直接依赖。
- `package-lock.json` - 当前已解析的 `ssh2` 版本和依赖关系。
- `src/connection/ConnectionManager.ts` - profile 类型、持久化、SecretStorage、连接参数构建、删除以及备份导入导出。
- `src/remote/RemoteSessionTypes.ts` - `ConnectOptions` 和 `ActiveConnection` 契约。
- `src/remote/RemoteSessionRouter.ts` - SFTP/FTP 会话路由和活动连接生命周期。
- `src/remote/ConnectionProbe.ts` - 当前从本机直接进行 TCP 可达性探测的实现。
- `src/ssh/SftpSessionManager.ts` - 当前 SFTP 直连、认证、取消、断开和底层 `ssh2.Client` 获取方式。
- `src/ssh/PortForwardManager.ts` - 项目中现有的 `ssh2.Client.forwardOut()` 调用和流清理参考。
- `src/ftp/FtpSessionManager.ts` - FTP/FTPS 直连和多数据 socket 行为，作为本期不支持 FTP jump 的依据。
- `src/panel/webview/markup/Body.ts` - Webview 连接表单入口。
- `src/panel/webview/scripts/RemoteSearch.ts` - 表单校验、变更快照与连接 payload 收集。
- `src/panel/webview/scripts/RemoteCommandActions.ts` - profile 填充、清空和 profile 到连接 payload 的转换。
- `src/panel/RemoteEditPanel.ts` - ProfilesLoaded、保存和连接消息处理。
- `src/sidebar/ConnectionDraftStore.ts` - 侧边栏保存连接与快速连接草稿转换。
- `src/sidebar/ItemHelpers.ts` - 侧边栏连接详情字段和显示内容。
- `src/sidebar/SidebarController.ts` - 侧边栏连接字段编辑、保存与建连流程。
- `ssh2` 官方 Connection hopping 示例 - `forwardOut()` 生成的 stream 可作为下一 `Client.connect({ sock })` 的传输通道。

## 8. 验收标准

- 没有配置 jump 的历史 SFTP 连接行为与修改前一致。
- 用户能在 Webview 和侧边栏为 SFTP 连接选择、查看、更换和清除 jump。
- 单级 jump 在最终目标无法从本机直连、但能从 jump 访问时可以成功连接并完成目录浏览、读取、写入、上传和下载。
- 至少用三层 jump 链验证多级逻辑；实现中不存在拒绝更深合法无环链的固定深度判断。
- 每一级均能分别使用密码或私钥认证；明文凭据不出现在 profile 快照、Webview 消息、普通持久化数据或日志中。
- 缺少已保存密码/口令时能安全提示；取消输入会取消整个连接尝试且不留下活动 socket。
- `A -> A`、`A -> B -> A` 以及更长循环在网络连接开始前被拒绝，并给出可理解的链路错误。
- 引用不存在的 profile、引用 FTP/FTPS profile 或手工向 FTP/FTPS 目标传入 jump 时均明确失败或在保存阶段被拒绝，不能静默直连。
- 最终目标主机名只需能被其前一跳解析，不要求本机 DNS 可解析。
- 任一中间节点连接或转发失败时，用户可以从错误信息定位具体失败节点和阶段。
- 连接取消、连接失败、正常断开和扩展关闭后，中间 SSH 客户端及转发流全部释放，不继续发送 keepalive，也不残留活动连接状态。
- 删除或改变一个正被引用的 jump profile 时会被阻止，并能告知存在依赖连接。
- 备份导出后再导入可以恢复完整 jump 引用；旧版备份和旧 profile 数据仍能作为直连连接使用。
- FTP/FTPS 连接继续按现有方式工作，界面中不提供 jump 能力。
- SSH Terminal、已有端口转发、远程命令、服务器工具和文件系统操作在直连与经 jump 的最终 SFTP 连接上保持可用。
- TypeScript 编译通过；对链解析、循环检测、导入引用校验和资源清理补充可自动验证的覆盖，并通过隔离网络拓扑完成至少一次集成验证。

## 9. 默认决策与执行假设

- **链深度**：不设置人为最大深度。
  - **依据**：用户已明确要求不要设置最大深度；profile 集合本身有限。
  - **影响范围**：链解析必须防循环并避免递归栈限制，测试需覆盖多级链而非只覆盖一层。
- **协议范围**：只有 SFTP 目标和 SFTP jump 参与 jump 链。
  - **依据**：SSH 的 `forwardOut`/`sock` 能直接承载后续 SSH；FTP/FTPS 动态数据连接需要额外代理机制。
  - **影响范围**：连接模型可保留通用可选字段，但 UI、保存校验和会话路由只对 SFTP 启用。
- **中间会话所有权**：每个最终目标建立并拥有独立的隐式 jump 链。
  - **依据**：避免复用可见活动会话后产生断开依赖、引用计数和跨目标故障传播。
  - **影响范围**：SFTP 会话管理器需要维护按最终 connection ID 隔离的客户端和流集合。
- **缺失凭据**：连接时使用 VS Code 原生安全输入提示，不要求 jump 必须预先连接或强制保存密码。
  - **依据**：与“选择已配置连接”及现有 SecretStorage 模型兼容，同时不把凭据暴露给 Webview。
  - **影响范围**：连接参数解析/发起流程需要支持异步凭据补全和用户取消。
- **被引用 profile 的删除或协议变更**：默认阻止操作，而不是自动清空依赖或允许形成悬空引用。
  - **依据**：自动清空可能让后续连接静默退化为直连，存在行为和安全风险。
  - **影响范围**：删除及保存逻辑需要反向依赖检查，UI 需要呈现明确错误。
- **备份格式**：导出格式升级为版本 3，新版继续导入旧版本。
  - **依据**：jump 是跨 profile 的语义引用；显式版本升级可避免旧版本工具静默丢失该关系。
  - **影响范围**：备份类型、版本校验、导入合并/替换和验收用例。
- **快速连接**：最终目标可以是快速连接，并选择已保存 SFTP profile 作为 jump。
  - **依据**：用户要求每个连接可指定 jump，快速连接已有完整的临时连接模型。
  - **影响范围**：Webview 和侧边栏草稿/payload 需要携带 jump ID，但快速连接自身不会成为候选 jump。

## 10. 需用户确认的问题（待确认问题）

无阻塞性待确认问题。以上默认决策足以生成 task checklist；若后续希望支持 FTP/FTPS jump、共享 jump 会话或改变删除策略，应作为独立范围重新评估。

## 11. 面向 task-checklist 的生成提示

- 后续拆分应覆盖数据模型与兼容、链解析与校验、凭据解析、SSH hopping、资源生命周期、两套配置 UI、备份契约、错误与日志、自动验证和隔离网络集成验证等维度。
- 任务依赖应体现先稳定数据与链方向契约，再接入连接管理和会话层，随后完成 UI 与备份，最后进行跨入口回归和资源泄漏验证。
- 对 `forwardOut`、`sock`、取消和断开清理相关工作应保留紧密的源码引用，避免后续执行时把现有普通端口转发误当成可直接复用的完整 jump 生命周期。
- 验证设计需要确保最终目标无法从测试进程直接访问，以证明连接确实经过 jump，而不是因本机网络可达产生假阳性。
