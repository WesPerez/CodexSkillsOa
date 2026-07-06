---
name: oa-real-action-evidence
description: 捕获真实 OA 动作窗口证据：可见按钮点击、下载、搜索、弹窗、保存/完成/授权动作、延迟 Element UI 消息、Network/console 失败、下载文件、后端日志、数据库读回和回滚边界。用于必须通过真实 Edge 官方插件交互证明 OA 功能，或其他 OA 技能在验收前需要请求/响应证据时。
---

# OA 真实动作证据

本技能用于“点击到证据”的闭环。可见按钮本身不等于验收通过；只有浏览器、后端/API、下载文件和数据库副作用证据互相吻合，动作才算通过。

## 可执行闭环原则

本技能应把每个可见 OA 动作变成一行证据，而不是变成死胡同。

每个控件的默认链路：

1. 先从源码链路分类动作。
2. 只读或只读下载动作，可通过官方 Edge 点击一次并捕获动作窗口。
3. 可能写入的动作，先调用 `oa-real-sql-gate`；只有样例和恢复计划通过门禁后才点击。
4. 当前账号缺权限时，追踪权限门，通过 `oa-real-browser-driver` 尝试已授权开发/测试账号；只有边界清楚时才用 `oa-real-sql-gate` 做临时开发/测试权限数据调整。
5. 涉及共享组件时，为每个受影响调用上下文重复记录证据行。
6. 后端代码已变更，或点击暴露运行时陈旧证据时，先调用 `oa-dev-verification-gate` 再重跑点击。

只有在下一条可执行路径不安全、未授权或缺少外部数据时，才使用 `blocked-*`。可见错误、无请求、权限拒绝、空文件或陈旧 tab 都是继续追链的信号，不是最终结论。

## 跨技能交接契约

本技能只负责可见动作证明，应与 OA 链路其他技能交换精简证据行：

- 输入：调用方名称、需求/控件矩阵行、路由/tab、来自 `oa-real-browser-driver` 的 `browserContext`、样例 id、动作分类、源码追踪、预期可见/API/DB/文件结果、可能写入时的 SQL gate 报告、后端新鲜度相关的 runtime gate 状态。
- 输出：每个控件一行动作证据，包含浏览器表面、时间戳、点击控件、Network/API 结果、console/可见消息窗口、文件证据、相关 DB/读回/恢复引用、tab 健康观察和一个证据状态标签。
- 动作为 `read-only` 或 `download-readonly` 时，源码证明无隐藏写入后可以点击。返回 `real-verified read-only`、`real-verified`、`partial-verified` 或 `needs-data`。
- 动作可写但没有 SQL gate 放行时，返回 `blocked-needs-safe-sample-or-rollback-plan` 并交给 `oa-real-sql-gate`；不要在本技能里编造备份/恢复 SQL。
- 点击显示后端/运行时陈旧症状时，返回 `runtime-stale` 或 `restart-unproven`，并带上精确请求、日志线索和重跑动作交给 `oa-dev-verification-gate`。
- 权限阻断时，除非可以通过已授权浏览器/账号或可逆开发/测试权限数据路径继续，否则返回 `manual-auth-required`。
- 请求只是 API/runtime 探活而非可见用户控件时，向浏览器/业务矩阵返回 `not-applicable-no-browser-surface`，并单独附 HTTP/log 证据。不要称其为真实可见动作。
- 源码追踪证明动作没有 Mapper SQL、数据库读回、DML 或持久层时，SQL 证据列返回 `not-applicable-no-sql`，不要强行调用 `oa-real-sql-gate`。
- tab 不健康时，返回 `tab-unhealthy-needs-browser-driver`，带上症状和最后安全证据。本技能不打开、关闭或替换 tab；tab 生命周期归 `oa-real-browser-driver`。
- 状态标签原样返回给 `oa-real-browser-driver` 或 `oa-business-logic-compare`，不要压缩成泛泛的“已验收”。

## 前置条件

1. 在 `E:/IdeaProjects/oa` 默认使用 Microsoft Edge。
2. 使用 `chrome:control-chrome` 的官方 Codex 浏览器扩展路径；除非用户明确要求 Chrome DevTools，否则最终 OA 验收不要替换成 `mcp__chrome_devtools`。
3. 从 `oa-real-browser-driver` 接收健康的已接管 tab 或新建兜底 tab。若 tab 无响应、CDP 被阻断、字典未加载或刷新后仍残留陈旧弹窗，标为 `tab-unhealthy-needs-browser-driver` 并交回症状。不要在本技能里自行打开、关闭或替换 tab。
4. 登录阻断路由且用户授权时，按 `oa-real-browser-driver` 的认证浏览器流程处理。只通过可见登录表单交互；不要检查 cookies、local storage、密码库、profile 或 token。
5. 非查询业务动作前，必须追踪前端方法、API、Controller、Service、Mapper/过程、SQL 类型、影响表、预期 UI 结果和停止条件。动作会写入或需要临时权限数据变更时，调用 `oa-real-sql-gate` 负责数据库身份、样例选择、备份/读回/恢复 SQL、执行许可和恢复验证；本技能不复制 DML 机制。

## 点击前分类

每个可见动作都要分类：

- `read-only`：查询、打开详情、打开弹窗、加载列表、对比记录。源码追踪证明无隐藏 DML 后可点击。
- `download-readonly`：只涉及 SELECT/文件生成的下载。点击后捕获请求/响应并检查下载文件。
- `download-with-write`：下载会更新状态、审计、权限或下载记录表。只有 `oa-real-sql-gate` 批准精确开发/测试样例和恢复计划后才点击。
- `dml-side-effect`：保存、完成、删除、授权/撤权、上传、流程/状态流转。只在确认开发/测试数据且 `oa-real-sql-gate` 判定可执行或明确阻断原因后执行。
- `ddl-or-unknown`：schema 变更或未分类副作用。不要作为浏览器验收执行。

下载不能默认当成只读。点击前必须证明源码路径。

## 动作窗口

每次真实点击都要：

1. 捕获基线：路由、可见筛选条件、选中行/样例 id、活动弹窗、登录角色/账号标签、Network/console 游标。
2. 通过官方 Edge 扩展点击真实可见控件一次。
3. 捕获 0-5 秒窗口：
   - Network 请求 URL、method、payload 摘要、status、headers/content type、response body 摘要和加载失败。
   - Console 错误和未处理 promise 消息。
   - Element UI `.el-message`、modal 文本、行内校验、禁用状态变化，以及出现可见消息时的截图。
   - 适用时记录下载文件名、大小、类型和内容检查。
   - 路径写入状态时，记录后端日志行和 SQL gate 的前/后/读回/恢复证据。
4. 请求异步或文件生成可能延迟报错时，立即轮询可见消息，并在 1s、2s、3s、5s 等短延迟后再次轮询。
5. 分层解释：
   - 无请求：locator、禁用状态、校验、权限或前端 handler。
   - transport/status 失败：代理、后端、路由或服务器。
   - blob 路径返回 `200` JSON 错误：后端业务错误必须暴露给用户。
   - `200` 二进制加错误 toast：前端 blob/save/catch 处理问题。
   - 成功 toast 但无文件/无读回：验收不完整，继续查下载或 DB。

不要把失败简化成一张即时截图。目标是捕获真实错误信号，包括延迟消息。

## 范围内控件覆盖门禁

宣布 OA 功能验收前，必须从真实页面为请求或受影响范围建立可见控件矩阵：

- 列出控件：查询字段、重置/搜索/导出、表头排序、分页、行菜单、行链接、弹窗、tabs、上传/下载按钮、授权按钮、保存/确认/提交/完成/删除按钮、关闭/取消按钮，以及禁用/隐藏分支。
- 每个控件记录：样例 id、动作分类、源码追踪、是否点击、可见结果、Network/API 结果、DB/读回结果、下载产物结果，或精确安全阻断原因。
- 只验证可见性永远不能算范围内控件通过。用户请求外或变更路径外的可见控件可标为 `not in scope`；范围内但未点击的控件必须标为 `blocked`、`manual authorization required` 或 `needs data`。
- 优先找一个能覆盖所有分支的代表性项目。没有单一样例覆盖全部分支时，拆分样例并记录每个样例证明哪个分支。
- 写入型控件点击前必须有 `oa-real-sql-gate` 结果。SQL gate 无法界定样例、副作用、恢复 SQL 或恢复验证时，不要点击。
- 用户要求完整功能、完整页面、共享组件或模块级闭环时，只验证最明显按钮不够；需包含行菜单、编辑弹窗、授权按钮组、完成按钮、导出按钮、上传/下载控件、共享详情组件，以及范围内的启用和权限阻断分支。
- 普通窄修复只点击变更行为、相邻回归点和受影响共享组件上下文。除非用户要求或代码改动让整个模块成为合理回归面，否则不要扩成全模块验收。
- 页面共享组件时，矩阵必须为每个调用上下文单独建行。某控件集在一个调用方正确，不代表另一个调用方也正确。

## 证据状态规则

更新矩阵、台账或最终报告时使用精确状态标签：

- `real-verified`：官方 Edge 动作完成，且可见结果、Network/API 或后端日志、写入的 DB 读回/恢复、下载的文件检查等必需证据齐全。
- `real-verified read-only`：可见动作和 API/DB 样例证据证明查询/打开/搜索/筛选路径无写入。
- `real-verified write-readback-restored`：写入路径已备份、点击一次、读回、恢复并验证恢复。
- `partial-verified`：已有源码、SQL 或可见证据，但至少缺一层必需证据。
- `source-verified`：代码和 SQL 路径已追踪，但缺真实浏览器动作或真实 DB/文件证据。
- `blocked-needs-safe-sample-or-rollback-plan`：副作用动作已理解，但尚不能安全点击。
- `manual-auth-required`：权限门无法通过现有授权账号或可逆开发/测试数据调整满足。
- `not-applicable-no-sql`：源码证明动作没有 Mapper/数据库持久层可验证。
- `not-applicable-no-browser-surface`：被验证项是 API/runtime 探活或 source-only 路径，不是可见用户动作。

阶段总结中不要把缺口弱化。缺少精确副作用、样例、恢复 SQL 或浏览器动作时，保留更强的缺口标签。

阶段总结和 commit 决策中，每一行都必须保留这些标签之一。不要写泛泛的“已验证”，除非该行确实是 `real-verified`、`real-verified read-only` 或 `real-verified write-readback-restored`。

## 下载证明

每个下载都要：

1. 记录源码分类：`download-readonly` 或 `download-with-write`。
2. 捕获 request body 和 response status/content type。
3. 确认保存文件存在、大小大于 0、文件签名匹配类型。
4. 检查代表性内容，例如项目编号、主题、封面类型、条形码/二维码文本或预期 Excel/Word 单元格。
5. `download-with-write` 需附 `oa-real-sql-gate` 读回/恢复结果，除非用户明确要求保留变更状态。

## 共享组件保护

组件或端点被多个路由、菜单、tab 或调用上下文共享时：

1. 从 props、route、API 前缀或行操作识别调用上下文。
2. 用户请求或代码改动触达共享行为时，变更后验证每个受影响上下文。
3. 记录每个上下文的预期控件集，尤其是看起来相同但副作用、权限或数据范围不同的控件。
4. 除非源码证据、新老 OA 证据或更新需求证明所有调用方都需要同一行为，否则不要为了修一个调用方而全局改变共享行为。

## 权限门证据

点击被当前用户权限、角色、部门、登录侧或归属阻断时：

1. 捕获可见拒绝消息和后端日志/过程证据。
2. 追踪精确权限门到用户/权限源表或存储过程，再选择绕行方式。确认该门实际路由到的数据源/schema；OA 登录身份和业务权限检查可能读取不同的新/老 OA 表。
3. 用只读 SQL 在实际路由数据源中找满足权限门的开发/测试账号或当前用户字段。确认是开发/测试库时，可以检查加密密码/状态字段判断账号可用性，但不要打印密码、密码 hash、cookies、local storage、浏览器 profile 或 token。
4. 需要默认开发凭据时，只能使用用户授权来源：当前会话输入、用户批准的可见输入框、为本次运行设置的环境变量、用户指定的项目/配置文件，或可见浏览器记住的凭据。不要通过搜索任意代码、文档、配置、浏览器存储或终端历史重新发现它们。按 `oa-real-browser-driver` 只通过可见登录表单尝试。如果用户授权可见浏览器记住的凭据，也只能通过可见登录表单使用，不能挖浏览器或文件系统秘密。
5. 验收需要开发/测试权限数据变更时，视为 `dml-side-effect` 并调用 `oa-real-sql-gate`。本技能记录可见动作证据；SQL gate 负责数据库身份确认、精确行备份、可逆 update/insert、读回、立即恢复和恢复验证。
6. 权限门无法安全满足时，标为 `manual authorization required`，并写清缺少的账号/部门/字段/样例。
7. 多数据源边界提醒：
   - OA 登录认证可能读取配置的新 OA schema 中的用户表。
   - 业务权限门可能通过动态数据源路由、`@ApiNeedCompanyCode`、存储过程或 `HT002`、`KS009`、`DY002` 等旧表读取配置的租户 schema。
   - 临时权限数据更新前，必须从 source-to-SQL 追踪确认权限门实际读取的 schema 和表。不要假设“用户表”就是新 OA 用户表。
   - SQL gate 必须针对实际读取的 schema/table，并用服务使用的同一权限门查询验证恢复，而不是对相似表做泛泛 `SELECT *`。

## 完成标准

动作只有在证据记录包含以下内容时才算通过：

- 浏览器表面：官方 Edge 扩展，已接管用户 tab 或有理由的新建官方扩展 tab。
- 路由、样例 id、筛选条件、点击控件和动作分类。
- 动作窗口内的 Network/console/可见消息结果。
- 适用时的下载文件证明或 DB 读回/恢复证明。
- 所有跳过动作都有精确安全原因，例如 `blocked-needs-safe-sample-or-rollback-plan`。
- 来自 `oa-real-browser-driver` 的 tab 上下文：原 tab/新建兜底、owner class、点击时健康状态，以及动作期间观察到的 tab 健康阻断。
