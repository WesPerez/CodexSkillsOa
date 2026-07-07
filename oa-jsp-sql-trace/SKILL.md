---
name: oa-jsp-sql-trace
description: "老 OA JSP / Struts FRAME / Oracle SQL 证据链追踪技能。用于从老 OA Struts forward、JSP URL、JSP 文件、nf 自定义标签、common_save/common_ajax_save/frame_list、FRAME_QUERY/FRAME_LIST/FRAME_TASK_QUERY/FRAME_TASK_QUERY_WHERE/FRAME_COMMIT、Oracle 函数/过程出发，独立完成老 OA 取证、只读数据库校验和迁移风险报告；不负责新 OA MyBatis 校验。"
---

# 老 OA JSP / FRAME / Oracle SQL 证据链追踪

## 1. 定位

只服务老 OA：`E:/IdeaProjects/oa/cpzx-oa`。

本技能负责独立完成老 OA 证据链：入口 -> JSP/JS/标签 -> 公共 FRAME 入口 -> FRAME 配置 -> SQL 展开 -> 函数/过程源码 -> 真实数据库只读校验 -> 副作用清单。

不做新 OA MyBatis 校验，不调用 `oa-real-sql-gate`。新老 OA 总体业务对比由 `oa-business-logic-compare` 调度：老 OA 取证用本技能，新 OA SQL 校验用 `oa-real-sql-gate`，两者不可互相替代。

主路径：`E:/IdeaProjects/oa/.agents/skills/oa-jsp-sql-trace/SKILL.md`。

本技能已从 `cpzx-oa` 子项目迁移到当前工作区上级技能目录；不要再维护子项目内的旧副本。

## 1.1 可执行闭环原则

本技能只做老 OA 取证，但取证本身必须能往前推进：

1. 入口不完整时，从 forward、JSP 文件名、按钮文字、FRAME id、URL 参数、业务表字段逐项反查，而不是直接停止。
2. 本地导出文件缺失时，先用 `rg --files` 找真实路径；仍缺失时，用只读数据库元数据 `all_source` / `all_objects` / `all_tab_columns` 兜底。
3. FRAME 配置过大时，定向检索 id、版本号、SQL_NAME、TASK001，不全量读大文件。
4. 函数/过程不能执行时，查源码、参数、对象和 `WHERE 1=0` 解析；把副作用列清楚交给新 OA 对比或 SQL gate，而不是猜。
5. 数据库不可连接时，输出已还原的文件级 SQL 和精确待执行只读校验 SQL；不要给一致性结论，但要给下一步可执行查询。

只有生产/未知库写入请求、会改变状态的过程执行请求、或入口完全无法定位且没有可检索关键词时，才停止并要求用户补充。

## 1.2 跨技能调用契约

本技能只产出老 OA 证据链，供业务对比或迁移判断消费：

- 输入：老 OA forward/JSP/URL、按钮/表单/下载/保存动作、FRAME id、业务表字段、样例业务主键或用户提供的页面现象。
- 输出：老 OA 证据报告，包含入口、JSP/JS/标签、FRAME 配置、展开 SQL、函数/过程源码、只读数据库元数据/样例校验、副作用清单和未验证项。
- 返回状态：`old-oa-traced`、`needs-entry-data`、`needs-readonly-db-result`、`side-effect-not-executed`、`blocked-entry-not-locatable`。
- 交给 `oa-business-logic-compare` 时，必须同时提供老 OA 基线和副作用清单；不要在本技能内判断新 OA 是否一致。
- 如果本地导出或数据库不可用，输出精确待执行的只读 SQL 和缺失路径，让 `oa-business-logic-compare` 把该项标成未验证，而不是猜测一致性。
- 不调用 `oa-real-sql-gate`；新 OA MyBatis/SQL 校验由业务对比技能转交给 SQL gate。

## 1.3 老 OA 真实库与运行时快速路径

老 OA 不是 MyBatis 项目；不要把 `oa-real-sql-gate` 当作老 OA 的数据库验证工具。遇到老 OA JSP、FRAME、函数、过程或运行目录问题时，按下面路径推进，避免在错误工具链上反复试错：

1. 先定位当前老 OA 运行实例实际使用的数据源。优先读取该实例的 `WEB-INF/proxool.xml` 和运行目录 classpath，通过老 OA 同源 JDBC/Proxool 建连；不要凭连接名、IP、schema 名或导出文件里的 schema 猜库身份。
2. 第一条 SQL 必须只读确认身份：`select user, sys_context('USERENV','CURRENT_SCHEMA'), sys_context('USERENV','DB_NAME') from dual`。未确认前按生产库处理；确认是开发/测试库后，才允许在用户授权范围内做受控 DDL/DML/编译。
3. 老 OA Proxool/解密类依赖旧 JDK 行为。若 JDK 17 等新运行时出现 `sun.misc.BASE64Decoder`、驱动加载、连接池初始化或解密相关错误，立即切换到项目/IDEA 老 OA 使用的 JDK 8 和 `WEB-INF/classes`、`WEB-INF/lib/*` classpath；不要继续试无关 JDBC 工具或新 OA 脚本。
4. 编译老 OA 函数/过程到开发库时，使用当前运行实例确认过的 schema。若本地导出 SQL 带旧 schema 或 `EDITIONABLE` 导致编译失败，先按目标开发 schema 做最小替换并去除不兼容关键字，再查 `USER_OBJECTS` / `USER_ERRORS` 验证 `VALID` 和错误为空。
5. 新增或修改 JSP 入口后，除文件存在外还必须验证 FRAME 路由配置：使用 `GlobalUtil.encrypt("页面名")` 打开的页面，应确认运行库 `FRAME_PAGE` 有对应 `PAGE_NAME`、`PAGE_PATH`、有效标志，且 JSP 已同步到当前运行的 exploded artifact/部署目录。
6. 浏览器点击只证明入口可触发，不证明闭环完成。点击后必须检查弹窗或 iframe 的可见正文、Network/console、服务器日志和必要的数据库读回；只要页面出现“服务器程序出现错误”、空白 iframe、跨 frame 加载失败或错误页，立即按 `oa-dev-verification-gate` 的 `browser-failure` 分支读取老 OA 运行控制台/日志和浏览器 Network/console，再回到 FRAME_PAGE/JSP 编译/参数传递/SQL 解析。不要只凭数据库探针猜原因，也不得继续宣称页面已连通。
7. 对会写库的按钮、保存、确认、下载记录、状态流转，只在确认开发/测试库、样例可控、影响字段明确、读回/恢复方案明确且用户授权后执行；否则只验证到按钮可见、请求可达、SQL/过程源码和副作用清单。

## 2. 输入

接受任一入口：

- Struts forward 名称或 URL 参数 `forwardName=...`
- JSP URL 或 JSP 文件路径
- JSP 页面里的 `nf:*` 标签 id、`taskStr`、`listData id`、`commitIDArrStr`
- FRAME 配置 id：`SQL_NAME`、`TASK001`、`TASK002`、`commit id`
- 业务按钮、表单、弹窗、下载、打印、保存动作描述

入口不唯一时，先写明默认假设；高风险动作、库身份不明、生产库写入请求必须停下询问。

## 3. 必读真实路径

从真实文件开始，不假设路径存在。用 `Test-Path` 或 `rg --files` 先确认。

公共入口：

- `web/WEB-INF/struts-config.xml`
- `web/WEB-INF/tld/<legacy-taglib>.tld`
- `web/WEB-INF/jsp/common/common_save.jsp`
- `web/WEB-INF/jsp/common/common_ajax_save.jsp`
- `web/WEB-INF/jsp/frame/frame_list.jsp`
- `web/js/frame/frame_ajax_save.js`

FRAME Java 工具类：

- `src/com/<legacy-package>/dbTool/tool/FrameList.java`
- `src/com/<legacy-package>/dbTool/tool/FrameQuery.java`
- `src/com/<legacy-package>/dbTool/tool/FrameCommit.java`
- `src/com/<legacy-package>/dbTool/tool/CommonSave.java`
- `src/com/<legacy-package>/dbTool/tool/DatabaseTool.java`

FRAME 配置导出：

- `sqlFiles/frame_tables/FRAME_QUERY.sql`
- `sqlFiles/frame_tables/FRAME_LIST.sql`
- `sqlFiles/frame_tables/FRAME_TASK_QUERY.sql`
- `sqlFiles/frame_tables/FRAME_TASK_QUERY_WHERE.sql`
- `sqlFiles/frame_tables/FRAME_COMMIT.sql`

函数/过程本地导出：

- `sqlFiles/functions`
- 需要时同时查 `sqlFiles/procedures`、`sqlFiles/packages`，前提是路径真实存在。

只定向检索大文件，不全量阅读 `FRAME_LIST.sql`、`FRAME_TASK_QUERY_WHERE.sql` 等超大导出。

## 4. 追踪流程

### 4.1 从入口还原页面

1. 如果输入是 forward，先在 `struts-config*.xml` 中定位 `<forward name="...">` 和 JSP path。
2. 如果输入是 URL，解析 `forwardName`、查询参数、表单目标、父页面 include。
3. 如果输入是 JSP 文件，先找调用方和 include 方：`rg -n "jsp文件名|forwardName|include"`。
4. 记录入口证据：文件、行号、forward 名、URL 参数、默认假设。

### 4.2 逐行解析 JSP

逐行读 JSP，不能只 grep id 后下结论。记录：

- `include`、`taglib`、公共 JS/CSS、弹窗 JSP、iframe、下载 JSP
- `<form>`、隐藏域、默认值、只读/禁用、校验、字典、上传字段
- 按钮、链接、onclick、onchange、弹窗打开函数、提交函数
- JS 引用、局部函数、全局函数、AJAX `.load()`、`forwardName`
- 请求参数来源：URL、session、request、隐藏域、列表当前行、父窗口

必须识别并映射这些老 OA 标签和参数：

| JSP/JS 项 | 追踪目标 |
|---|---|
| `nf:dst` | 数据源容器、`dsID`、alias/cfalias |
| `nf:dstData id` | `FRAME_QUERY.SQL_NAME` |
| `nf:listDst listID` | 列表容器、分页、alias/cfalias |
| `nf:listData id taskStr where` | `FRAME_LIST.SQL_NAME` + `FRAME_TASK_QUERY.TASK001` + `FRAME_TASK_QUERY_WHERE` |
| `nf:list` | 渲染列、分页、AJAX 刷新目标 |
| `nf:form` | 保存表单、action、upload、edit、param |
| `nf:input` | 入参字段、显示字段、字典、校验、是否可编辑 |
| `commitIDArrStr` | 加密后的 `FRAME_COMMIT` 配置串 |
| `commitIDArrStr_i` / `commitIDArrStr_u` | 新增/修改保存链路 |
| `GlobalUtil.encrypt(...)` | 被加密的 query/list/task/commit id |
| `a_save(this)` | 普通公共保存入口 |
| `jq_commsave(...)` | AJAX 公共保存入口 |
| `l_para` | 列表刷新和保存后的列表参数 |
| `frame_list_curr_page` | 列表 AJAX 翻页/刷新 |

### 4.3 追公共入口和 Java 实现

把 JSP 事件接到真实公共入口：

- `common_save.jsp` / `common_ajax_save.jsp` -> `CommonSave.save()` -> `DatabaseTool.save()` / `batchSave()`
- `frame_list.jsp` / `frame_ajax_save.js` -> `FrameList` / `ListDao` / `DatabaseTool.changeListSql(...)`
- `nf:dstData` -> `FrameQuery.execute()` / `QueryDao`
- 直接 Java 调用 -> `FrameQuery`、`FrameList`、`FrameCommit`

保存链路只做取证，不执行。`FrameCommit.execute()`、`CommonSave.save()`、`DatabaseTool.save()`、`batchSave()`、`commitSQL(...)` 都会改库或提交事务，禁止作为验证手段调用。

### 4.4 展开 FRAME 配置

对 JSP 中每个 id 建立映射表：

- `dstData id` -> `FRAME_QUERY.SQL_NAME`
- `listData id` -> `FRAME_LIST.SQL_NAME`
- `listData id` -> `FRAME_TASK_QUERY.SQL_NAME` -> `TASK001`
- `taskStr` -> 拆成 `TASK001` + `#XH00001`
- `commitIDArrStr*` 解密来源或明文配置 -> `FRAME_COMMIT`

记录字段：

- `FRAME_QUERY`：`SQL_NAME`、`SQL_STR`、where/入参、返回列、alias/cfalias
- `FRAME_LIST`：`SQL_NAME`、`SQL_STR`、`SQL_STR_COL`、`ALIAS`、分页、排序、显示列、函数列
- `FRAME_TASK_QUERY`：`SQL_NAME`、`FORM001`、`FORM002`、`TASK001`、`FORM003`、`DB`
- `FRAME_TASK_QUERY_WHERE`：`TASK001`、`XH00001`、`TASK002`、`SQL_WHERE`、`BACK001`
- `FRAME_COMMIT`：`SEQ_ID`、commit id、`SQL_STR`、`OUT_STR`、alias/cfalias、执行顺序

版本条件必须逐项展开：`#001`、`#002`、`#003` 等每个版本都写完整 where、排序、入参和语义差异，禁止写“同上”。

### 4.5 展开 SQL

还原最终 SQL 时写清：

- 原始配置 SQL
- JSP/请求入参替换规则：`?XXX*`、`#XXX*`、`#L_1*`、`?L_1*`
- where 条件、动态花括号条件、默认条件
- order by、分页包装、rownum 范围
- SELECT 返回列、字典/函数列、隐藏业务列
- 保存 SQL 的插入/更新/删除目标表、字段和值
- 执行顺序、异常分支、`OUT_STR`/返回参数

如果配置、JSP、Java 实现互相冲突，列冲突证据，不要猜。

### 4.6 递归追函数/过程

遇到以下项必须递归追踪：

- `F_*`、`f_*`
- `P_*`、`p_*`
- `call ...`
- 包调用：`pkg.proc(...)`、`schema.pkg.func(...)`
- `FRAME_COMMIT` 中以过程/函数形式出现的配置

优先本地导出：`sqlFiles/functions`、真实存在的 `sqlFiles/procedures` / `sqlFiles/packages`。本地没有时，只读查询 `all_source`、`all_arguments`、`all_objects`、`all_synonyms`。

函数/过程报告必须写：

- 入参来源、类型、示例值来源
- 变量含义
- 查询表、更新表、删除表、插入表
- 条件分支、循环、游标、动态 SQL、异常分支
- 返回值、`OUT` 参数、错误码/错误消息
- 是否可能有副作用；不确定时按有风险处理

不要为了看结果而执行过程。不要执行未确认纯只读的函数；含函数的 SELECT 样例优先用 `WHERE 1=0` 解析，或只查源码。

## 5. 数据库安全门槛

执行任何 SQL 前，先确认目标库身份；不能只凭连接名、IP、schema 名或注释判断。

第一条只读校验必须记录：

```sql
select user as current_user,
       sys_context('USERENV','CURRENT_SCHEMA') as current_schema,
       sys_context('USERENV','DB_NAME') as db_name
from dual
```

身份不清按生产库处理。生产库只读。禁止 DDL/DML，包括 `INSERT`、`UPDATE`、`DELETE`、`MERGE`、`TRUNCATE`、`DROP`、`ALTER`、`CREATE`、`GRANT`、`REVOKE`。

允许的查询只限：

- 元数据：`all_objects`、`all_tab_columns`、`all_source`、`all_arguments`、`all_synonyms`、`user_*`
- 解析：`SELECT ... WHERE 1=0`
- 小样例：有明确业务样例且 `rownum <= N`
- 统计：带明确过滤条件的 `COUNT(*)`
- 表/字段注释：`all_tab_comments`、`all_col_comments`

禁止：

- 执行有业务副作用的过程、函数、下载、打印完成、审批流转、状态变更
- 调用老 OA 工具的更新能力：`DBQueryTool` / `DatabaseTool` 的 `commitSQL`、`save`、`batchSave` 等
- 输出账号、密码、token、完整连接串
- 把生产库只读查询写成大范围全表扫描或长事务

## 6. 报告模板

输出中文 Markdown。证据必须带文件/行号或数据库对象/配置 id。

```markdown
# 老 OA JSP / FRAME / Oracle SQL 证据链追踪报告

## 1. 输入入口与假设
- 用户入口：
- 默认假设：
- 未确认事项：

## 2. JSP / Struts 入口证据
| 层级 | 文件/forward/URL | 行号/参数 | 结论 |
|---|---|---|---|

## 3. JSP 标签与按钮事件证据
| JSP 元素 | id/name/taskStr/commit | 参数来源 | 触发事件 | 证据 |
|---|---|---|---|---|

## 4. 公共保存 / 列表入口证据
| 入口 | 公共 JSP/JS/Java | 调用路径 | 风险 |
|---|---|---|---|

## 5. FRAME 配置映射表
| JSP 来源 | FRAME 表 | 配置 ID | 版本 | 关键字段 |
|---|---|---|---|---|

## 6. SQL 展开结果
| 配置 ID | 最终 SQL/where/order/page | 入参来源 | 返回列/保存字段 | 副作用 |
|---|---|---|---|---|

## 7. 函数/过程递归追踪
| 对象 | 类型 | 入参来源 | 查询表 | 更新表 | 返回/OUT | 风险 |
|---|---|---|---|---|---|---|

## 8. 数据库真实只读校验
| 库身份确认 | 当前 USER | CURRENT_SCHEMA | 校验 SQL | 结果 |
|---|---|---|---|---|

## 9. 表字段和注释
| 表 | 字段 | 类型 | 注释 | 来源 |
|---|---|---|---|---|

## 10. 副作用清单
| 动作 | 表/过程 | 条件 | 副作用 | 是否执行 |
|---|---|---|---|---|

## 11. 未验证项
| 项 | 原因 | 需要什么 |
|---|---|---|

## 12. 风险与迁移提示
- （按追踪结果填写）
```

## 7. 完成检查

- [ ] 已确认真实入口，不只按文件名判断。
- [ ] 已逐行解析 JSP、include、taglib、隐藏域、按钮、弹窗、JS。
- [ ] 已追 `common_save`、`common_ajax_save`、`frame_list`、`frame_ajax_save.js`。
- [ ] 已读相关 FRAME Java 工具类。
- [ ] 已展开 `FRAME_QUERY`、`FRAME_LIST`、`FRAME_TASK_QUERY`、`FRAME_TASK_QUERY_WHERE`、`FRAME_COMMIT`。
- [ ] 已对 `#001/#002/#003` 等版本逐项展开，未写“同上”。
- [ ] 已记录 `SQL_STR`、`SQL_NAME`、`OUT_STR`、`ALIAS`、`SEQ_ID`、入参、where、排序、分页、返回列、保存副作用。
- [ ] 已递归追 `F_*`、`P_*`、包调用、`call`。
- [ ] 已确认数据库身份、`USER`、`CURRENT_SCHEMA`。
- [ ] 数据库只读；未执行 DDL/DML；未执行副作用过程/函数。
- [ ] 未输出账号、密码、token、完整连接串。
- [ ] 已列未验证项和副作用风险。
