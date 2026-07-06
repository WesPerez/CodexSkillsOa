---
name: oa-real-sql-gate
description: 真实开发/测试数据库 SQL 门禁。用于修改 OA MyBatis XML、SQL、DAO 或数据库相关代码后，证明 SQL 能通过 MyBatis 解析并在真实开发/测试 Oracle schema 上执行，且不留下脏数据。
---

# OA 真实 SQL 门禁

## 目的

在宣布任何 OA SQL 改动完成之前，必须先通过本门禁。Mockito DAO/Service 测试不够：它们不会解析 MyBatis XML，不会生成 `BoundSql`，也不能证明 Oracle 表、字段、schema 路由、函数或动态 SQL 分支在真实开发库中存在。

本门禁只负责基础 SQL 安全和证据，不单独证明业务一致性。

- 判断数据逻辑是否符合老 OA 业务规则时，使用 `oa-business-logic-compare`。
- 需要真实页面、接口、下载或副作用验收时，使用 `oa-real-browser-driver`。
- 后端/Mapper 修复通过 SQL 门禁后，如仍需 IDEA/DevTools 重启和浏览器 Network/console 复验，使用 `oa-dev-verification-gate`。

## 可执行闭环原则

本门禁应产出下一步可执行的数据库动作，而不是只给一个 `blocked` 标签。

默认链路：

1. 执行任何 SQL 前，先确认数据库身份和 schema。
2. 为选中的 Mapper/XML 生成 MyBatis `BoundSql`。
3. 用安全参数在真实开发/测试 schema 执行 SELECT 路径。
4. 参数缺失时，从源码条件和只读数据库查询中寻找候选样例，再更新 `real-sql-gate-samples.yml`。
5. 检测到 DML 时默认阻断，但必须输出缺失项：环境身份、样例 id、原始行快照查询、计划变更、预期影响行数、读回查询、恢复 SQL、恢复验证 SQL。
6. 用户已授权精确开发/测试写入且恢复要素齐全时，才可使用 `-AllowDml` 执行，必须验证回滚/恢复，并把报告交给 `oa-real-action-evidence` 或 `oa-real-browser-driver`。
7. SQL/代码改动通过后，交给 `oa-dev-verification-gate` 编译/重载，再交给浏览器/动作证据做真实验收。

只有在生产/未知库写入、无界副作用、缺少授权、或已尝试样例发现但仍缺少恢复验证时，才使用 `blocked`。

## 跨技能交接契约

本门禁负责数据库执行安全和 SQL 证据。其他 OA 技能应把本报告视为数据库契约：

- 输入：Mapper id 或 XML 路径、必要的 DAO/Service 上下文、companyCode/数据源预期、参数来源、样例 id、SQL 类型、预期影响表，以及调用方是否准备浏览器动作、后端重载或业务一致性检查。
- 输出：报告路径、数据库身份/schema、选中数据源、生成的 `BoundSql`、参数集、对象元数据结果、执行/读回结果、副作用分类、DML 执行时的恢复证据、最终状态。
- 返回状态值：`pass`、`needs-data`、`blocked-production-or-unknown-db`、`blocked-missing-authorization`、`blocked-unbounded-side-effect`、`blocked-missing-restore-verification`、`write-readback-restored`、`failed`。
- SELECT 或安全 SQL 解析得到 `pass` 时，交回 `oa-business-logic-compare` 做一致性判断；如后端运行时需要重载，交给 `oa-dev-verification-gate`。
- 得到 `write-readback-restored` 时，把报告路径交回 `oa-real-action-evidence` 或 `oa-real-browser-driver`，让可见点击附上数据库证据。
- 得到 `needs-data` 时，必须写清所需表、字段、分支和样例条件，方便调用方继续只读找样例。
- 任何 `blocked-*` 都必须写清缺少的安全要素；阻断解除前，不要要求浏览器/动作技能点击副作用路径。

## 必用命令

从 `E:/IdeaProjects/oa` 运行共享工具：

```powershell
# 仅检查已变更 Mapper XML
.\.agents\tools\real-sql-gate\run-new-oa-mybatis.ps1 -Mode changed

# 检查一个精确 Mapper statement
.\.agents\tools\real-sql-gate\run-new-oa-mybatis.ps1 -Mapper 'com.nfrc.modules.commons.system.system.dao.SysCompanyDatabaseDao.selectAllList'

# 检查一个 XML 文件
.\.agents\tools\real-sql-gate\run-new-oa-mybatis.ps1 -Xml 'src/main/resources/mybatis/system/system/SysCompanyDatabaseMapper.xml'
```

`-Xml` 会校验该 XML 中的所有 statement。若包含 INSERT / UPDATE / DELETE，除非 DML 已被明确授权且有恢复验证，否则预期结果应为 `blocked`。

SQL 通常通过 `@ApiNeedCompanyCode` 到达租户数据源时，使用租户数据源：

```powershell
.\.agents\tools\real-sql-gate\run-new-oa-mybatis.ps1 -Mapper '<namespace.id>' -CompanyCode '010'
```

共享样例文件：

```text
E:/IdeaProjects/oa/.agents/skills/oa-real-sql-gate/real-sql-gate-samples.yml
```

必要时覆盖样例文件：

```powershell
.\.agents\tools\real-sql-gate\run-new-oa-mybatis.ps1 -Mapper '<namespace.id>' -Samples 'E:/IdeaProjects/oa/.agents/skills/oa-real-sql-gate/real-sql-gate-samples.yml'
```

报告输出到：

```text
E:/IdeaProjects/oa/.agents/reports/real-sql-gate/real-sql-gate-report.md
```

除非任务明确要求证据产物，否则不要提交生成的报告。

## 数据源规则

- 数据库 URL、用户、密码、driver、schema、companyCode 路由输入必须来自用户授权来源：当前会话输入、用户批准的 prompt/dialog、为本次运行设置的环境变量，或用户明确指定/确认的项目文件、配置文件、凭据文件。若用户已指定读取位置，直接读取，不要为了强制交互中断长任务。未经授权不得搜索任意已提交配置文件中的凭据。
- `sqlGate.companyCode=010` 会用默认连接从 `SYS_COMPANY_DATABASE` 解析真实动态数据源。
- 不得在回答、日志、报告、截图或文档中打印数据库用户名、密码或 token。
- `@ApiNeedCompanyCode` 链路中的普通租户 SQL 不得硬编码租户 schema 前缀。遇到 `ORA-00942` 时，先检查 schema 路由，不要直接补 schema 名。
- 任何可写检查前，必须记录真实数据库身份：`USER`、`CURRENT_SCHEMA`、数据库名/服务标签。看起来像正确用户表或权限表还不够；SQL 必须打到 Service/Mapper 路径实际使用的数据源。
- 权限和登录路径可能跨数据源：认证可能读新 OA 表，业务权限可能读老 OA 租户表。备份、DML、读回和恢复必须全部针对实际读取的 schema/table。

## SQL 类型策略

### SELECT

必须满足：

1. MyBatis XML 能解析。
2. statement 能生成 `BoundSql`。
3. SQL 能在真实开发/测试 Oracle 数据库执行。
4. 报告记录 Mapper id、XML 路径、数据源 key/schema、参数来源、行数和对象元数据检查。

如果没有安全参数集，先用与 Mapper where/join 条件一致的只读查询寻找样例。仍找不到时，标为 `needs-data`，写清需要的表/字段/分支。不要伪造参数来换绿色结果。

### INSERT / UPDATE / DELETE

默认阻断，除非明确启用。

执行必须同时满足：

1. `run-new-oa-mybatis.ps1` 使用 `-AllowDml`。
2. `E:/IdeaProjects/oa/.agents/skills/oa-real-sql-gate/real-sql-gate-samples.yml` 有样例。
3. 精确 Mapper id 上有 `allowDml: true`。
4. 有 `rollbackVerifySql` 证明恢复后回到原状态。
5. 使用真实、可控的测试数据。

缺少任一项时，标为 `needs-data` 或 `blocked`，列出缺少的恢复要素，方便调用方找更好样例或请求明确授权。不要在开发库留下脏数据。

### 存储过程 / 副作用

除非用户明确授权精确环境、样例 id 和恢复计划，否则不要执行包含 commit、审批流、打印完成、文件生成、下载记录、权限授予、状态流转等副作用的过程或 SQL。

无需额外审批即可做：

- `all_objects`、`all_source`、`all_arguments` 等元数据查询。
- 只读函数，但必须先检查函数体/源码确认无副作用。
- 序列元数据检查，例如通过 `all_sequences` / `all_objects` 确认 `SEQ_*` 存在。不要仅为证明存在而执行 `SEQ_*.NEXTVAL`；`NEXTVAL` 会推进数据库状态，不是只读探针，除非外层业务流已具备授权写入/恢复计划。

## 已变更 SQL 范围

`run-new-oa-mybatis.ps1 -Mode changed` 会检查新 OA git 仓库：

- `git diff --name-only`
- `git diff --name-only --cached`
- 未跟踪文件

它会选择 `src/main/resources/mybatis` 下变更的 `*Mapper.xml`。若无法安全隔离精确变更 statement，就校验该 XML 中所有 statement，这是有意设计。

新增 SQL 调用的 DAO 或 Service 变更，仍需要人工用 `-Mapper '<namespace.id>'` 精确选择。

## 样例文件

使用工作区级文件：

```text
E:/IdeaProjects/oa/.agents/skills/oa-real-sql-gate/real-sql-gate-samples.yml
```

最小结构：

```yaml
samples:
  com.example.Dao.selectSomething:
    companyCode: "010"
    params:
      param:
        ksid001: "real_sample_id"
```

DML 必须显式 opt in：

```yaml
samples:
  com.example.Dao.updateSomething:
    companyCode: "010"
    allowDml: true
    params:
      id: "real_test_row"
      status: "002"
    rollbackVerifySql: "select status from SOME_TABLE where ID = ?"
    rollbackVerifyParams:
      - "real_test_row"
```

样例选择规则：

- 优先选择一个能覆盖目标列表行、弹窗、下载、授权、完成和导出分支的代表性项目/账号。
- 一个样例无法覆盖全部分支时，添加多个具名样例，并记录每个样例证明哪个分支。
- 不要用编造 id 只为了生成 `BoundSql`。没有安全数据时，返回 `needs-data` 并写清需要的表/字段/样例。
- 临时开发/测试权限更新必须包含原始行快照查询、update/insert、预期影响行数、读回查询、恢复语句、恢复验证查询。缺任一项，SQL gate 保持 `blocked`。

## 报告模板

门禁报告必须包含：

- 结论：pass / failed / needs-data / blocked。
- 范围：变更 XML、显式 Mapper id 或显式 XML。
- 数据库标签和当前 schema，省略凭据。
- SQL 清单：Mapper id、SQL 类型、XML 文件、数据源/schema、判定。
- 执行细节：参数来源、执行模式、行数或影响行数。
- 可行时提供 `all_objects` / synonym 等对象元数据检查。
- 执行 DML 时提供回滚/恢复记录。
- 失败项和未验证项。

## 完成标准

- 相关 Mapper XML 已通过 MyBatis 解析。
- 每个选中 statement 已生成 `BoundSql`，或明确标为 `needs-data`。
- SELECT 已在真实开发/测试数据库执行。
- DML 已回滚/恢复，或明确标为 `blocked` / `needs-data`。
- 生成报告不含凭据。
- 业务正确性结论由 `oa-business-logic-compare` 支撑，不能只依赖 SQL Gate。
