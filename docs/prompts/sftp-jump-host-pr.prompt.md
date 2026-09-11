# SFTP Jump Host：面向上游 issue #36 的 PR 准备方案

## 1. 背景、目标与本轮产物

用户已在 `josegrabelha/remoteedit#36` 提出 SFTP Jump Host，并在 fork 的 `dev` 实现。维护者认可方向，邀请向上游 `main` 提交 PR，明确关注 connection cleanup、credential handling、backward compatibility、connection references、import/export behavior。维护者建议先用一个 PR，因为模型、UI、凭据、测试和文档相互关联；必要时再拆分。这是愿意审查，不是已经认可实现质量或承诺合并。[REF-001]

本轮用户要求：分析维护者关注点，在 fork 的 `dev` 准备有关 PR 的 prompt。当前文件是后续生成 checklist 的方案来源，本轮不实现修复、不改原执行清单、不创建或提交 PR。

本次后续工作的目标：在保留已实现功能契约的基础上，修正有证据的生命周期与取消反馈缺口，补齐可重复执行的回归证据，将改动整理为维护者能够审查的单个 PR。成功标准是五个关注点都有明确行为、源码依据和验证结果，而不是继续扩大 Jump 功能。

## 2. 已核实基线

- 核实日期：2026-09-11。实际仓库根目录为 `E:\workspace\vscode\remoteedit`，remote 为 `https://github.com/tzraeq/remoteedit.git`。操作前工作区干净，当前分支 `dev`，HEAD 为 `4c83099`。
- GitHub API 返回的上游 `main` 为 `6d443738d7c05ebce1b1934d820a88df258d698c`，与本地 `main` 一致。`dev` 和 `main` 的共同祖先为 `a18ed11f4f11580bffe87b35a17b16979305416a`；`dev` 尚未包含上游新增的贡献指南和许可贡献条款。[REF-002]
- `git diff main...dev --stat` 为 27 个文件、3193 行新增、177 行删除，其中原 prompt 和 checklist 共新增 706 行。三点 diff 是以共同祖先计算的 PR 变更，不能将其当作 `dev` 已包含上游最新提交的证据。
- 本轮执行 `npm ci --no-audit --no-fund` 后，执行 `npm test`，15/15 通过；该命令先执行 `npm run compile`，TypeScript 编译通过。环境为 Windows、Node v24.5.0、npm 11.5.1。没有修改依赖声明或 lockfile。
- 自动测试仅有链解析与 `SshJumpChain` 两个测试文件。直连 SFTP、FTP/FTPS 的隔离项是读取源码做正则断言，未运行完整会话管理器、连接管理器或真实 FTP/SFTP 网络操作。[REF-011]
- README 已提供隔离多主机验证步骤，并明确截至原实现时未执行；本轮也没有执行该拓扑或 VS Code 双 UI 手工回归。[REF-012]

旧 `docs/prompts/sftp-multi-hop-jump.md` 中的“当前尚不支持 Jump”等描述属于实现前背景；现有 checklist 的完成记录属于历史证据。后续工作应以本文件的现状为准，不重做已完成的七项功能，也不将历史验证记录当作本轮运行结果。

## 3. 维护者关注点分析

| 关注点 | 维护者需要确信什么 | 当前证据与缺口 | 本次处理方向 |
| --- | --- | --- | --- |
| 连接清理 | 成功移交、失败、取消、远端断开和重连都有明确资源所有者，关闭一条目标链不影响另一条 | 已有逆序幂等 `dispose`；但销毁调用后立即移除 error 监听，测试替身不模拟异步关闭。最终客户端 close 仅清理 Jump 链，未同步移除 session/connection；未完成连接也尚未进入 `disconnectAll` 枚举集合。[REF-004][REF-005] | 修正状态与所有权缺口；用接近真实时序的事件验证清理，不以调用过 destroy 作为唯一完成标准 |
| 凭据处理 | 每跳使用自己的秘密，原生输入可取消，秘密不会随 profile、会话快照、日志或普通备份外泄 | SecretStorage 分层与白名单快照已存在。Jump 密码提示发生在两套 UI 建立取消进度之前，取消会进入普通错误分支。现有测试未覆盖 ConnectionManager 的 SecretStorage 交互和输出边界。[REF-006][REF-007] | 保留存储模式，修正取消分类，补充逐跳取密、临时输入和输出边界的行为测试 |
| 向后兼容 | 旧 profile、旧备份以及直连 SFTP、FTP/FTPS 的行为可解释且可验证 | 无 Jump 的 profile 继续直连；导出 v3，v1/v2 导入按直连。直连 SFTP 也使用了新认证 helper，因此源码隔离检查不足以证明运行时不回归。[REF-005][REF-009][REF-011] | 保留既有兼容策略，用旧数据 fixtures 和真实方法调用证明；文档明确兼容方向 |
| 配置引用 | 保存、删除、协议转换、组删除、导入都维护同一张合法引用图 | 纯解析器和 mutation 前校验已实现；缺少持久化入口的可重复测试。UI 候选过滤不能代替宿主校验。[REF-003][REF-008][REF-013] | 测试最终集合、拒绝前零写入、稳定 ID 和批量删除边界；已正确的代码保持不动 |
| 导入导出 | 引用及各跳凭据往返后仍对应，合并与替换不会意外形成悬空引用或半套配置 | 已在最终图校验及解密成功后写设置、组、profile 和 secrets；写入仍分步进行，不是跨存储事务。缺少公开测试覆盖这些约束。[REF-009][REF-010] | 明确前置失败的无写入保证、merge/replace 语义和加密往返；不把既有存储 API 扩成通用事务系统 |

### 3.1 已核实、需要处理的具体缺口

1. **最终远端关闭后的状态不完整。** `attachFinalClientCloseListener()` 的回调删除 close listener 记录与 `jumpChains`，但不删除 `sessions`、`connections` 或缓存；`getConnection()`/`listConnections()` 仍读这些 map。Router 也没有订阅 SFTP 远端关闭通知。需要完成“传输关闭 → 清理当前会话 → Router 通知 → 两套 UI 刷新”的闭环。[REF-005][REF-014]
2. **建立中的连接不在 `disconnectAll()` 所有权集合中。** SFTP 客户端和隐藏链要到平台及起始目录初始化后才注册，`disconnectAll()` 仅枚举已注册 map。必须让建立中的尝试可被显式断开或整体释放取消，迟到的成功不能重新注册为活动连接。[REF-005]
3. **Jump 密码输入取消被显示为失败。** `buildJumpConnectOptions()` 正确抛出取消异常，但 Panel 与 Sidebar 的前置 `buildConnectOptions()` catch 都按普通错误处理，尚未进入后面的 canceled 分支。[REF-006][REF-007]
4. **关键模块缺少自动回归。** 当前 `npm test` 没有执行 ConnectionManager 的公共保存/导入/导出方法，也没有执行 SftpSessionManager 的完整连接与关闭流程。这是验证缺口，不等于这些方法全部有 bug。[REF-011]

### 3.2 待复现的时序风险

- `SshJumpChain.dispose()` 在 `destroy()` 后立即移除自身 error 监听；真实 ssh2 将 socket error 继续转发到 Client。应覆盖 destroy 已调用但终止事件尚未到达时的 error，确认不会成为未处理异常。现有 ControlledStream.destroy 仅计数，未模拟 Node stream 终止时序。[REF-004][REF-011][REF-015]
- 取消路径先 await 最终 SFTP `end()`，再 dispose 隐藏链，没有采用正常 disconnect 路径已有的超时边界。真实 `end()` 在活动 SFTP 状态下等待 close；应注入迟到 close/不收敛 end 的场景，验证中间链不会被无限拖住。[REF-005][REF-015]
- `forwardTo()` 主要依赖回调或 token 完成。ssh2 的实际 socket close 会清理挂起 channel 请求并回调，不能仅凭没有显式 close listener 就断言它必然挂起。应模拟真实依赖的关闭回调与取消竞态，再决定是否需要补充当前阶段监听。[REF-004][REF-015]

这些风险尚未通过本轮专门的运行时复现实验证实。后续先补充能区分正确/错误实现的场景；只有失败或资源生命周期契约确实缺失时才修改对应代码。

## 4. 功能范围与保留契约

- 已保存的 SFTP 连接与 Quick Connect 可选择 Direct 或已保存 SFTP profile；临时连接不成为候选。每个 profile 保留至多一个 `jumpProfileId`，允许有限无环多跳，不设人为深度上限。[REF-001][REF-003]
- 宿主先完成全链结构校验，再按最外层到最终目标建立；每个目标拥有独立隐藏链。中间层用 `ssh2.Client.forwardOut()` 返回的 channel 作为下一 SSH 的 `sock`，最终用 `ssh2-sftp-client` 建立 SFTP。只有最外层从本地做 TCP 探测。[REF-004][REF-005]
- 不创建 Jump 本地监听或可配置端口映射，不要求预先打开可见 Jump 会话；FTP/FTPS 维持直连。终端、普通端口转发、远程命令和文件操作继续使用最终目标会话。[REF-001][REF-012]
- 本次后续变更覆盖上述功能对应的清理、取消反馈、引用、备份、凭据边界、测试与说明。具体修复限定在已核实缺口和能够复现的风险；不重新设计认证方式、链路复用或备份加密体系。

## 5. 方案与关键取舍

### 5.1 从连接尝试到会话结束的所有权

SftpSessionManager 应在首个异步建连工作前登记当前尝试，登记项拥有取消/关闭能力和尝试身份。成功完成最终初始化后，将资源移交给活动会话；失败、取消或 disconnectAll 进入同一幂等清理路径。建立中的终止与已建立会话的终止都必须使结果收敛，不能只删除 map 而让仍在执行的 Promise 继续注册会话。

沿用“最终 SFTP → 最近 Jump 流/客户端 → 最外层”的释放顺序。正常断开与取消共用有界的最终客户端关闭策略，优先复用现有 `SSH_DISCONNECT_TIMEOUT_MS`，并以 finally 确保中间链仍被释放。不要新增用户可配置的清理超时。

资源结束以最终关闭状态为依据：业务监听和取消订阅在当前阶段结束时释放；防止未处理错误的监听保留到终止事件收敛后再解除。已关闭/未成功启动的资源也必须能完成清理。计时器、error/close 监听和迟到 forward 回调均纳入测试；不得通过永久保留空监听来掩盖泄漏。

远端关闭的处理应校验当前客户端/尝试身份，清理当前目标的 session、connection、缓存与隐藏链，避免旧客户端迟到的 close 删除同 ID 的新连接。中间传输失败也应使最终目标关闭或连接尝试失败；如果实测底层无法及时传播，则由链所有者通知终止。

拟议增加一个内部、可选的 SFTP 连接关闭通知契约，例如 `onDidCloseConnection: Event<string>`；Router 订阅后移除匹配的 route，并触发现有 `onDidChangeConnections`，供 Panel/Sidebar 现有订阅刷新。事件在管理器清理当前连接后发出，显式 disconnect 与远端关闭去重。它只传 connection ID，不传运行时链或凭据。现有接口尚未提供该通知，不能当作已存在能力。[REF-014]

### 5.2 凭据与取消反馈

继续由 ConnectionManager 从每个 profile 自己的 SecretStorage key 读取密码/口令。Jump 缺少密码时仍在网络建立前使用原生密码框补齐；Esc 表示取消整次尝试，Panel 和 Sidebar 的前置 catch 应识别取消异常，清除 pending/busy 状态并给出 canceled 反馈，不显示认证失败。此时尚无网络资源，不必为修复提示分类重构整套凭据准备流程。[REF-006][REF-007]

私钥口令在确实需要时由运行时原生输入补齐，沿用连接 token 取消。临时输入只用于当前尝试，不自动回写 secrets，不更改其他跳的 remember 选项。目标连接原有凭据输入方式和保存偏好保持不变。

持久化只保存 `jumpProfileId`；runtime `jumpChain` 可含当前尝试的秘密，但不得流入 profile、outbound Webview 消息、ActiveConnection、普通备份连接项或日志。pending/active 快照继续采用字段白名单；不要直接序列化 ConnectOptions。路线摘要使用名称，错误可包含节点/阶段/host/port，不打印含密配置或私钥内容。[REF-006][REF-007][REF-010]

验证使用为每跳生成的不同 synthetic secret，检查各客户端收到正确值，同时捕获 globalState、outbound 消息、日志和普通备份 JSON，断言这些秘密不出现。允许用户选择的加密凭据块，不把“可以加密导出秘密”误述为泄漏。

### 5.3 引用图及配置变更

保留宿主解析器作为最终可信边界。保存后候选集合、导入后最终集合都通过相同引用校验，UI 筛选只是提前反馈。[REF-003][REF-008]

具体行为保持如下：

- 重命名或移动分组不改变 profile ID，目标仍引用同一 Jump。
- 单个删除或改成 FTP/FTPS：存在直接依赖时拒绝，并列出可理解的依赖名称。拒绝发生在 globalState/secrets 写入前。
- 删除组但保留连接：引用继续有效。连组内连接一起删除：只要组外剩余连接引用待删除集合就拒绝；若引用双方都在同一删除集合，允许整组删除。
- SFTP 输入省略 `jumpProfileId` 表示沿用已保存选择；显式空字符串表示切回 Direct。FTP/FTPS 保存和普通连接参数构建清除该字段，维持既有协议行为；v3 备份中的非 SFTP Jump 引用按导入校验拒绝。这两个入口的处理差异应分别验证，不将它们混写为“一律报错”。[REF-008][REF-009]
- 缺失或循环的 SFTP 引用明确拒绝，不能静默变成 Direct。候选列表更新后两套 UI 都保持 ID 语义与宿主一致。[REF-003][REF-013]

### 5.4 备份与兼容语义

保留当前版本 3，不新增备份版本或加密算法。公开文档解释兼容方向：新版本接受旧 v1/v2；旧版本读取 v3 会因较新格式而拒绝，不能承诺旧扩展能够读取新备份。[REF-009]

| 场景 | 明确采用的行为 |
| --- | --- |
| 历史 profile 没有 Jump 字段 | 继续直连，不做一次性数据迁移 |
| v1/v2 导入 | Jump 字段按旧版本语义解释为无 Jump；恢复为直连配置 |
| v3 导出再导入 | 保持 profile ID 与 `jumpProfileId`；校验与连接数组排列顺序无关 |
| merge | 同 ID 按现有导入覆盖规则合并，最终集合为保留的现有配置与导入配置；导入项可引用保留下来的现有 Jump |
| 旧备份 merge 覆盖同 ID 的现有 Jump 目标 | 沿用当前完整配置覆盖语义，该目标恢复成备份中的 Direct；文档明确这个可观察结果，不默认为“只补缺失字段” |
| replace | 最终图只来自导入集合；引用未导入的旧配置会拒绝；成功替换按原有语义清除旧凭据，再恢复所选择的加密凭据 |
| includeCredentials / restoreCredentials | 沿用显式选项；普通 connection 项没有明文密码/口令；加密块按 profile ID 恢复，不按名称或数组位置匹配 |
| 图校验失败、备份密码错误、缺少所选凭据块 | 整次导入在设置、组、profile、secrets 及附属数据写入前失败 |

合并时删除/替换一个被现有配置引用的 Jump，必须以最终图判断；不能只验证导入文件自身。重复 ID 和不支持的连接类型仍由当前归一化处理，再验证最终图，避免被过滤的 Jump 留下悬空引用。

这里的“失败不改状态”限定为可预检的引用/版本/解密失败。现有 API 分步写入 settings、globalState、SecretStorage，没有跨存储事务；本 PR 不承诺磁盘或 VS Code 存储故障下自动回滚。这是现有架构边界，不是新增一个事务改造需求。[REF-009][REF-010]

### 5.5 测试方式与 PR 组织

沿用 Node 内置 `node:test` 和现有编译入口；新增测试必须被 `npm test` 实际执行。ConnectionManager 用内存 globalState/secrets、原生输入替身、可观测写入记录，调用真实公共方法。会话测试调用真实 SftpSessionManager/Router 方法，以受控 client、stream、平台探测等边界替换外部依赖；仅增加必要的测试接入点，不建立通用插件/依赖注入框架。

保留有价值的纯图测试及 no-listener 架构约束，但行为保证不能仅以源码正则证明。清理测试至少有真正模拟异步 error/close 的资源，测试结束检查 listener、timer、pending attempt 和活动状态；不把同步计数替身的“15 tests passed”扩大为无泄漏证明。[REF-011]

后续 PR 推荐从届时最新上游 `main` 创建独立分支，带入相关实现、修复、测试和英文用户文档，以单个 PR 提交。fork 的 `dev` 和规划记录保留，避免重写其已发布历史；本轮不执行 Git 合并、挑选提交或建 PR。上游代码若有新变化，重新核对本方案引用后再整合。[REF-001][REF-002]

最终上游 diff 聚焦功能代码、相关自动测试、依赖声明与 lockfile、README/CHANGELOG；原 prompt、完成清单及本文件作为 fork 的工作资料，默认不纳入 PR。源码、测试和说明按最终 diff 组织，去掉与功能无关的空白变更。新增测试编译产物也应核对是否被 `.vscodeignore` 排除，避免随扩展发布；当前 ignore 只排除了 src，未单独排除 `out/test`。[REF-016]

PR 使用英文，说明触发场景与可观察行为，链接 #36，按维护者五点给出设计和验证摘要；附双 UI 示例及备份兼容说明。列明真实执行的命令、隔离拓扑结果和未完成验证。README/CHANGELOG 描述最终用户行为，具体开发验证记录放在 PR，不把历史未验证说明写成永久产品限制。遵守上游 CONTRIBUTING 与 LICENSE 的贡献条款。[REF-001][REF-002]

## 6. 端到端验收场景

以下为后续验收标准，除第 2 节已明确记录的基线测试外，本轮尚未完成这些验证。

| 场景与输入 | 可观察结果 | 验证方式 |
| --- | --- | --- |
| 历史无 Jump 的 SFTP profile，分别使用密码、普通私钥、需口令私钥 | 仍走直连；配置、认证、取消与错误反馈符合原有用户行为，不误建隐藏链 | 旧 profile fixtures + 真实方法调用；VS Code 冒烟 |
| A→B→C→D、三跳混合认证 | 实际建立顺序 Local→D→C→B→A；每跳收到自己的认证和 keepalive；仅 D 本地探测，最终只注册 A | runtime 行为测试 + 隔离拓扑 |
| 自引用、目标循环、中间循环、缺失引用、FTP/FTPS Jump、512 层有限链 | 非法图在网络前拒绝；合法长链解析通过，没有深度阈值 | 保留图测试，并经 save/buildConnectOptions/import 公共入口验证 |
| Jump 密码框 Esc；中间口令框取消；SSH/forward/最终 SFTP/初始化期间取消 | 两套 UI 反馈 canceled；不注册会话、不保存临时秘密；已创建资源全部释放，重复取消幂等 | 输入替身 + 生命周期测试 + 双 UI 冒烟 |
| 某跳认证失败、forward 失败、最终 SFTP 失败或目录初始化失败 | 错误能定位节点与阶段，所有已拥有资源释放，秘密不进入输出 | 分阶段故障注入，输出捕获 |
| 远端关闭、销毁后迟到 error/close、取消后迟到 forward stream | 无未处理异常；最终 Promise 收敛；活动连接与两套 UI 状态更新；迟到 stream 被释放 | 模拟异步终止事件；Router 通知断言；真实依赖 smoke |
| 建立中 disconnectAll；同 ID 重连后旧客户端 close；两个目标使用同一个 Jump profile | 没有迟到注册；旧清理不删除新会话；关闭一个目标不影响另一条私有链 | manager 级时序与资源隔离测试 |
| 临时密码、已存密码、私钥口令分别用于不同 Jump | secrets key 按 ID 对应；临时输入不自动保存；globalState、outbound 快照、日志和普通 backup JSON 不含秘密 | 内存存储写入记录 + synthetic secret 检查 |
| 重命名、删除、协议切换、删组保留连接、组内整体删除及存在组外依赖 | ID 稳定；保护只阻止会破坏剩余图的操作；拒绝前零写入 | ConnectionManager 公共方法行为测试 |
| v1/v2 导入；v3 打乱数组顺序后往返；merge 引用现存 Jump；replace 缺少 Jump | 结果符合第 5.4 节版本与最终集合语义，非法图无写入 | 版本 fixtures + 结果图断言 + mutation spy |
| v3 含密/无密备份；错误密码；不恢复凭据；同名不同 ID profiles | 仅显式选择时加密导出与恢复，秘密按 ID 配对；解密失败无副作用 | 调用真实 buildBackupFile/importBackupFile，使用 synthetic credentials 和真实加解密 |
| 双 UI 保存/编辑/Quick Connect、Direct 清除与 FTP/FTPS 切换 | 候选、dirty 状态、payload 与宿主一致；保存跳转字段只传 ID，协议隔离不回归 | 小范围自动行为验证 + 双 UI 手工记录 |
| 隔离网络 Local→D→C→B→A | 本机不能直连 C/B/A，A 主机名只由 B 解析；目录浏览、读写、上传下载成功，断链/取消后服务器侧会话关闭 | 使用可销毁主机或容器，记录路由/访问限制与服务端连接证据 |
| 最终目标的 SSH Terminal、既有端口转发、远程命令和文件操作；直连 FTP/FTPS | 使用正确最终会话；既有操作可用且不误入 Jump runtime | 针对已有功能做冒烟，不扩成全产品测试矩阵 |

提交前证据至少包含更新后的 `npm test` 结果、可解释的最终 diff、双 UI 记录和隔离拓扑状态。隔离拓扑尚不可用时，准确标记未完成，不把 mocks 当作替代；完整拓扑验收的执行条件见第 9 节。

## 7. 参考资源

本地行号以 `dev@4c83099` 为准，根目录为 `E:\workspace\vscode\remoteedit`。除明确标为待复现的行为外，以下文本均已读取核实；行号漂移时按稳定锚点重定位。

- REF-001
  - 来源：<https://github.com/josegrabelha/remoteedit/issues/36>；维护者回复 <https://github.com/josegrabelha/remoteedit/issues/36#issuecomment-5470368754>。
  - 用途：需求及维护者审查范围。
  - 参考范围：issue 的 Summary、Expected behavior、Connection behavior、Validation；2026-08-30 维护者整条回复。
  - 稳定锚点：`Feature request: SFTP Jump Host support`；`I'll pay particular attention to connection cleanup, credential handling, backward compatibility, connection references, and import/export behavior.`
  - 原文事实：邀请向 main 提 PR、阅读 CONTRIBUTING、先提交单 PR；作者列出五项关注点。
  - 推导判断：本次重点是审查证据与可靠性；回复不等于实现已获认可。
  - 适用范围与限制：公开 issue 截至核实日仅一条评论；实现和测试声明需要独立核对。
  - 核实状态：已核实，使用 GitHub API 读取正文与完整评论。

- REF-002
  - 来源：<https://github.com/josegrabelha/remoteedit/blob/6d443738d7c05ebce1b1934d820a88df258d698c/CONTRIBUTING.md>；同提交的 `LICENSE`；本地 Git refs。
  - 用途：PR 目标、贡献约定及整合基线。
  - 参考范围：CONTRIBUTING 的 Pull Requests、Contribution License；LICENSE 的 Contributions；`git log dev..main`、`git merge-base dev main`。
  - 稳定锚点：`Pull requests should target the main branch`（原文 main 带反引号）；`perpetual, worldwide, non-exclusive, irrevocable, royalty-free license`。
  - 原文事实：要求聚焦改动并补相关测试/文档；提交贡献授予作者列明的使用、修改、分发等许可；dev 缺少新增指南和条款的提交。
  - 推导判断：后续基于最新 main 整理 PR，保留其贡献文件；当前 prepare 不发生贡献提交。
  - 适用范围与限制：许可是仓库原文事实，不在本次改写；不能把审查邀请等同于发布保证。
  - 核实状态：已核实；上游 main SHA 经 API 确认，指南经 API 读取，LICENSE 经对应本地 main 读取。

- REF-003
  - 来源：`src/connection/JumpChain.ts`。
  - 用途：链方向与图校验的现有契约。
  - 参考范围：40–129 行。
  - 稳定锚点：`resolveJumpProfileChain`、`visitedProfileIds`、`return nearestToOutermost.reverse()`。
  - 原文事实：迭代解析，目标/中间自引用、循环、缺失和非 SFTP 引用报错；输出最外层到最近层。
  - 推导判断：应复用现有宿主校验，不重写另一套持久化图算法。
  - 适用范围与限制：纯函数行为不等于所有保存/导入入口已经充分测试。
  - 核实状态：已核实。

- REF-004
  - 来源：`src/ssh/SshJumpChain.ts`。
  - 用途：隐藏链建立、终止与认证时序。
  - 参考范围：117–179 行认证；201–291 行建链；293–343 行资源销毁；345–397 行 SSH 阶段监听；400–465 行 forward 与迟到流。
  - 稳定锚点：`resolveSshAuthentication`、`SshJumpChain.open`、`dispose`、`connectClient`、`forwardTo`。
  - 原文事实：只探测最外层，通过 forwardOut/sock 接力；dispose 逆序调用 end/destroy 后移除 error listener；forward 取消后销毁迟到流。
  - 推导判断：所有权方向可保留；异步关闭收敛需要更接近真实依赖的验证。
  - 适用范围与限制：源码能证明调用顺序，不能单独证明 OS socket 已关闭或延迟 error 一定出现。
  - 核实状态：已核实源码；延迟事件风险待复现。

- REF-005
  - 来源：`src/ssh/SftpSessionManager.ts`。
  - 用途：最终 SFTP 的资源移交、取消、断开与活动状态。
  - 参考范围：109–124 行 maps；126–170 行尝试创建；172–260 行认证到初始化及注册；262–300 行活动信息/失败清理；310–340 行输入与取消关闭；342–455 行断开及 final close；457–463 行活动查询。
  - 稳定锚点：`connect`、`cleanupAttempt`、`closeClientForCancellation`、`disconnectAll`、`attachFinalClientCloseListener`、`listConnections`。
  - 原文事实：注册发生在初始化完成后；取消关闭 await end；正常关闭已有有界策略；final close 仅移除链相关记录。
  - 推导判断：建立中资源需可取消；远端关闭需同步 session/connection 状态；关闭策略应统一。
  - 适用范围与限制：当前分析针对 Jump 相关生命周期；直接认证 helper 的变更仍需兼容回归。
  - 核实状态：已核实源码；end 不收敛的具体运行场景待测试。

- REF-006
  - 来源：`src/connection/ConnectionManager.ts`；`src/remote/RemoteSessionTypes.ts`。
  - 用途：凭据与 runtime/持久化边界。
  - 参考范围：ConnectionManager 389–407 行 profile 快照；1257–1341 行连接参数；1345–1411 行逐跳取密与输入；1415–1446 行保存偏好；1961–1963 行 secret key。RemoteSessionTypes 7–45 行 runtime 链及 token。
  - 稳定锚点：`listProfiles`、`buildConnectOptions`、`buildJumpConnectOptions`、`applyCredentialPreferences`、`secretKey`、`JumpConnectOptions`。
  - 原文事实：每跳按 ID 读 SecretStorage；Jump 缺失密码原生提示且取消抛取消异常；该输入没有 token 参数；临时 Jump 输入不写入 secrets。
  - 推导判断：测试必须走真实 ConnectionManager，覆盖读取身份与未自动保存，而不仅测试 SSH config。
  - 适用范围与限制：含密 runtime 对象允许存在于宿主当前尝试中；禁止向普通持久化或 outbound 消息扩散。
  - 核实状态：已核实。

- REF-007
  - 来源：`src/panel/RemoteEditPanel.ts`；`src/sidebar/SidebarController.ts`。
  - 用途：取消异常的用户反馈与 Webview 输出边界。
  - 参考范围：Panel 945–987 行 profiles/pending 快照；1336–1409 行建连前后 catch 与 token；Sidebar 3055–3114 行 buildConnectOptions、进度和错误处理。
  - 稳定锚点：`buildPendingConnectionSnapshot`、`private async connect(payload: any)`、`private async connectWithPayload`。
  - 原文事实：两者先 buildConnectOptions 再建立进度取消源；前置 catch 直接报错；Panel 白名单快照只映射 Jump ID 和名称。
  - 推导判断：修正前置取消分类；保留白名单并加入输出泄漏回归。
  - 适用范围与限制：本次不将原有目标密码表单改为全新的交互模式。
  - 核实状态：已核实。

- REF-008
  - 来源：`src/connection/ConnectionManager.ts`。
  - 用途：配置引用保护的 mutation 边界。
  - 参考范围：316–373 行组删除；411–479 行保存；555–573 行单个删除；1192–1205 行统一校验；1273–1275 行普通连接协议归一化；1871–1880 行依赖提示。
  - 稳定锚点：`deleteGroup`、`saveProfile`、`deleteProfile`、`validateProfileJumpReferences`、`findDirectJumpDependents`。
  - 原文事实：校验先于写入；整组删除针对剩余 profiles 判断外部引用；非 SFTP 普通保存/建连清空 Jump。
  - 推导判断：测试观察真实写入行为，并区分单项删除与集合删除语义。
  - 适用范围与限制：无需新增自动重定向引用或级联清空策略。
  - 核实状态：已核实。

- REF-009
  - 来源：`src/connection/ConnectionManager.ts`。
  - 用途：备份版本、最终集合和导入副作用顺序。
  - 参考范围：670–695 行导出结构；748–787 行版本和导入准备；795–868 行 merge/replace、校验、解密及写入；915–940 行备份字段；1112–1162 行版本归一化；1459–1475 行旧 profile；1857–1869 行版本校验。
  - 稳定锚点：`buildBackupFile`、`importBackupFile`、`toBackupConnection`、`normalizeBackupConnections`、`normalizeStoredProfile`、`validateBackupVersion`。
  - 原文事实：导出使用当前版本 3；旧版本 Jump 归为 undefined；同 ID merge 覆盖导入字段；先验证最终图、解密，再写设置和存储。
  - 推导判断：保留版本和覆盖语义，补往返与零写入测试；无事务保证不应夸大。
  - 适用范围与限制：导入声明必须区分 merge、replace 与 settings-only；v3 向旧版本兼容不成立。
  - 核实状态：已核实。

- REF-010
  - 来源：`src/connection/ConnectionManager.ts`。
  - 用途：加密凭据导出及恢复契约。
  - 参考范围：697–705 行导出选项；843–889 行解密及按 ID 恢复；1164–1189 行收集/删除凭据；1891–1937 行加解密。
  - 稳定锚点：`collectStoredCredentials`、`encryptCredentials`、`decryptCredentials`、`restoredCredentials[profileId]`。
  - 原文事实：显式选择时用 scrypt/AES-256-GCM 加密；replace 清除旧凭据；只向导入 ID 恢复秘密。
  - 推导判断：用真实加密往返与不同 ID 的 synthetic secrets 验证，不引入新加密设计。
  - 适用范围与限制：普通备份结构与加密凭据块的验收规则不同；后者允许承载加密秘密。
  - 核实状态：已核实。

- REF-011
  - 来源：`src/test/JumpChain.test.ts`；`src/test/SshJumpChain.test.ts`；`package.json`。
  - 用途：现有验证能力与缺口。
  - 参考范围：JumpChain.test 45–145 行各图测试；SshJumpChain.test 59–78 行 stream 替身、80–160 行 client 事件调度、297–378 行建链/销毁断言、380–526 行错误/取消/认证测试、529–548 行源码断言；package.json 2201–2217 行脚本及依赖。
  - 稳定锚点：`ControlledStream`、`ControlledClient`、`source contract keeps Jump internal while Direct SFTP and FTP/FTPS remain separate`、`scripts.test`。
  - 原文事实：npm test 显式指定两个测试文件，15 个测试；清理替身主要计数；协议隔离读取源码匹配正则。
  - 推导判断：新增测试必须接入单一命令，并覆盖真实管理器行为；通过的基线不能替代集成证据。
  - 适用范围与限制：本轮已跑 15/15，但未完成此方案新增验收。
  - 核实状态：已核实源码并实际运行基线。

- REF-012
  - 来源：`README.md`；`CHANGELOG.md`；原 `docs/prompts/sftp-multi-hop-jump.md`。
  - 用途：现有公开使用说明、隔离拓扑与保留需求。
  - 参考范围：README 317–351 行；CHANGELOG 3–14 行；原 prompt 153–170 行验收标准。
  - 稳定锚点：`SFTP Jump Hosts`、`Isolated topology verification`、`Verification status`、原 prompt `8. 验收标准`。
  - 原文事实：已描述双 UI、多跳和隔离 D→C→B→A；明确原开发环境未运行隔离验证；原需求包含最终目标工具兼容。
  - 推导判断：复用已有拓扑；实际执行后按事实更新验证说明。
  - 适用范围与限制：说明书不是网络验证结果；原 prompt 的实现前现状不能当作今天的现状。
  - 核实状态：已核实文本，外部拓扑未执行。

- REF-013
  - 来源：`src/panel/webview/scripts/RemoteCommandActions.ts`；`src/sidebar/SidebarController.ts`。
  - 用途：双 UI 候选与宿主校验的一致性。
  - 参考范围：RemoteCommandActions 855–915 行；SidebarController 2679–2727 行。
  - 稳定锚点：`analyzeJumpProfileCandidate`、`getJumpProfileSelectionError`、`promptSidebarJumpProfileId`。
  - 原文事实：Webview 有迭代候选分析；Sidebar 构建 Direct-first picker 并利用 buildSidebarJumpDisplay 判断可用性。
  - 推导判断：需要候选和 Direct 清除的跨入口回归，无须重写现有 UI。
  - 适用范围与限制：前端校验不替代 ConnectionManager 的图校验。
  - 核实状态：已核实源码，UI 操作未在本轮运行。

- REF-014
  - 来源：`src/remote/RemoteSessionRouter.ts`；`src/remote/RemoteSessionManager.ts`；Panel/Sidebar 的连接变化订阅。
  - 用途：远端关闭到活动状态/UI 的通知契约。
  - 参考范围：Router 20–53 行事件与构造、56–106 行断开/查询；RemoteSessionManager 75–81 行连接接口；Panel 587–591 行订阅；Sidebar 110–126 行订阅与刷新。
  - 稳定锚点：`onDidChangeConnectionsEmitter`、`sessionRoutes`、`RemoteSessionManager`、`connectionChangeEvent`。
  - 原文事实：Router 维护 route 并主动发出连接变化事件；现有接口未提供 SFTP 远端关闭通知；UI 已订阅 Router 的变化。
  - 推导判断：需要最小内部关闭通知，使清理结果复用现有 UI 刷新链路。
  - 适用范围与限制：拟议事件只传连接身份，不改变 FTP 功能或构造新的通用事件总线。
  - 核实状态：已核实。

- REF-015
  - 来源：lockfile 安装的 `node_modules/ssh2/lib/client.js`、`node_modules/ssh2-sftp-client/src/index.js`。
  - 用途：检查真实依赖对错误和 close 的语义，约束测试替身。
  - 参考范围：ssh2 client 797–829 行；ssh2-sftp-client index 1454–1485 行。
  - 稳定锚点：`Socket error:`、`Notify outstanding channel requests of disconnection`、`End the SFTP connection`。
  - 原文事实：socket error 转发为 Client error；socket close 清理挂起 channel 请求；SFTP end 在存在 sftp 时等待 close，无活动连接则直接 resolve。
  - 推导判断：不要凭简化替身断言 forward 必挂起；测试需模拟延迟终止事件。
  - 适用范围与限制：以本轮 lockfile 实际版本为准；这是依赖源码事实，不是完整网络验证。
  - 核实状态：已核实安装后的依赖源码。

- REF-016
  - 来源：`.vscodeignore`。
  - 用途：新增测试的发布包边界。
  - 参考范围：1–16 行源文件与构建排除；17–42 行依赖与 extras 排除。
  - 稳定锚点：`TypeScript sources and build metadata`、`Dependency files not needed at runtime`。
  - 原文事实：排除了 src、TypeScript、map 等，没有项目级 `out/test` 排除。
  - 推导判断：后续针对编译后的测试做最小排除并检查包内容。
  - 适用范围与限制：本轮未制作 VSIX，未声称已观察到实际发布包内容。
  - 核实状态：已核实规则，包内容待执行时复核。

## 8. 默认决策与执行假设

- **单个 PR，目标上游 main。** 依据维护者明确回复；后续在独立分支整合，fork dev 保留，避免为缩小审查差异删除本地历史资料。
- **保留已有 Jump 数据与网络模型。** 依据 issue 和当前实现；本轮后续修复集中在生命周期、取消反馈及测试，不借机新增认证或代理功能。
- **保持 v3 与既有 merge/replace 语义。** 依据 ConnectionManager 的明确实现；旧备份覆盖同 ID 后恢复 Direct 必须写入说明和回归用例。若用户决定改为保留现有 Jump，这将改变导入契约，应回到 prepare 修订。
- **取消分类采用最小修复。** 原生 Jump 密码框仍在网络前提示，两套 UI 正确识别 Esc 为 canceled；运行时口令框与网络阶段继续接入已有 token。
- **生命周期验证允许最小测试接入点。** 依据目前缺少 manager 级测试；不为可测试性重构与 Jump 无关的模块。
- **前置错误零写入，不承诺跨存储故障事务。** 依据现有写入顺序和存储架构；这一保证范围必须同时体现在测试、README 和 PR 描述中。
- **内部规划文档留在 fork。** 作者要求聚焦改动，原规划已有 706 行；上游 diff 默认带用户说明和测试说明，本文件及原执行记录保留供本地工作使用。

## 9. 需用户确认的问题

无阻塞性的业务选择需要本轮确认；以上默认方案可作为生成 checklist 的输入，不能据此将 review 状态写为“用户已确认”。

执行时须复核的条件（均为 **不阻塞，仅执行时复核**）：

- 上游 main 和 dev HEAD 是否变化；变化后重读受影响 REF，再判断整合冲突。
- 可销毁的隔离多主机/容器环境及双 UI 测试环境是否可用；不可用时其他修复与自动验证可以继续，但隔离拓扑验收保持未完成，PR 说明准确披露限制，不冒充验证通过。
- `npm test` 在后续采用的开发环境是否可运行；当前只有 Node v24.5.0 的基线结果，不能扩展为所有 Node/VS Code 版本都通过。

## 10. Review 状态与交接说明

状态：**待 review**。

本轮已依据 issue、上游贡献指南、实际 dev 源码和依赖完成方案分析，并重跑现有 15 项测试。可供 review 的内容是五项关注点的处理方式、明确缺口、待复现风险、备份语义、生命周期通知与验收边界。用户授权的是 prepare，尚未确认本文件全部设计，也未授权本轮进入 create/execute 或提交 PR。

后续 create 应将“当前已正确、需要证明”的行为与“需要修复”的缺口区分，保留 REF 的读取边界及执行条件；不得把历史完成清单重新置为待办，也不得将本方案的验收标准抄成已通过结果。运行时风险经复现若不成立，就保留相关回归证据，不为符合推测而修改正常代码。
