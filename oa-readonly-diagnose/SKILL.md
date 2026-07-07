---
name: oa-readonly-diagnose
description: 为 OA 页面、弹窗、列表、按钮禁用、审批页空白、数据少/查不到、权限过滤、生产现象不一致等问题生成最少步骤的只读诊断 SQL。用于 Codex 从老 OA JSP/FRAME/函数/过程或新 OA Vue/API/Mapper 源码还原真实查询、过滤、脱敏、分页、按钮状态和副作用边界，再给用户一段可直接在生产/目标库执行的只读 SQL 包；用户回传结果集后由 Codex 判定根因。适用于“我只想执行 SQL 给你结果”“步骤越少越好”“生产库只读排查”“把页面逻辑变成诊断 SQL”“审批 OALS 断链只读诊断”等场景。
---

# OA 只读 SQL 诊断

## 定位

本技能是 OA 生产/目标库只读排查的统一入口。目标是把一次排查压缩成最少用户动作：

1. Codex 读源码、配置和参考案例，还原真实页面/接口逻辑。
2. Codex 输出一段可复制执行的只读 SQL 包，参数集中在顶部。
3. 用户在目标库执行并回传结果集。
4. Codex 根据结果集给出根因、证据和下一步最小动作。

不要为每个业务特例新建独立技能。业务特例沉淀到 `references/`；只有出现全新的工具链、安全边界或验证模式时，才考虑独立技能。

## 安全边界

- 生产库和身份不明库只允许只读：`SELECT`、`WITH`、数据字典/元数据查询。
- 禁止输出可直接执行的 `INSERT`、`UPDATE`、`DELETE`、`MERGE`、DDL、写入型过程、审批流转、下载完成、打印完成、状态变更。
- 第一段 SQL 必须返回库身份列：`USER`、`CURRENT_SCHEMA`、`DB_NAME`。如果是多结果集脚本，第一组也必须返回。
- 不输出账号、密码、token、完整连接串。
- 调用函数前必须从源码、导出或既有证据判断其无明显写副作用；不确定时只查 `ALL_SOURCE/USER_SOURCE` 或把函数列为未验证项。
- SQL 必须有业务边界：主键、编号关键词、登录人、日期范围、状态或 `FETCH FIRST`/`ROWNUM` 上限，避免生产大范围扫描。
- 不让用户执行“先更新再查”“临时改状态验证”“跑过程看看”的动作。

## 输入处理

接受任一线索并自行补链路：

- 页面截图、页面标题、按钮文字、弹窗标题、字段名、列表列名。
- 老 OA `forwardName`、JSP、FRAME id、`taskStr`、`commitIDArrStr`、业务过程/函数名。
- 新 OA 路由、Vue 文件、API URL、Controller、Service、DAO、Mapper id。
- 业务编号、项目编号、客户名、流水号、当前登录用户、公司/部门、状态、时间范围。

如果缺关键参数，最多问一次。低风险缺口使用 SQL 参数占位，例如 `替换为当前登录YHID001`、`CAST(NULL AS VARCHAR2(64))`。

## 参考库选择

按需读取以下参考，不要一次性加载无关案例：

- `references/case-old-oa-project-handover-popup.md`：老 OA 项目交接“项目编号选择”弹窗，`KS009` 候选、负责人有效性过滤、`XM041` 已选禁用状态。
- `references/case-old-oa-approval-oals.md`：老 OA 审批页空白、OALS 流水分裂、`OA001/OA003/OA010` 断链、PC/移动端审批痕迹不一致。
- `references/approval-oals-diagnostic-sql.md`：OALS 审批断链只读 SQL 模板。只有遇到审批/OALS 线索时读取。

需要老 OA JSP/FRAME 证据链时，可调用 `oa-jsp-sql-trace` 的方法还原 JSP -> `nf:*` 标签 -> `FRAME_QUERY/FRAME_LIST/FRAME_TASK_QUERY_WHERE/FRAME_COMMIT` -> SQL/函数/过程。需要新老 OA 行为一致性审计时，交给 `oa-business-logic-compare`。需要真实浏览器动作验收或开发库 SQL 门禁时，交给对应专用技能；本技能不替代那些验证模式。

## 诊断模型

把页面逻辑拆成结果集可判定的列：

- 候选全集：按用户输入的最宽但有边界条件查出原始候选。
- 页面过滤：把每个 `WHERE`、函数判断、状态判断转成 `CASE` 原因。
- 权限/登录人：显式列出当前用户、部门、角色、负责人、创建人、组织条件。
- UI 状态：可见、不可见、禁用、已选中、可勾选、按钮可点、按钮隐藏。
- 关联状态：是否已有主表/明细/日志/关联表，是否被其他流程占用。
- 汇总计数：候选总数、页面应显示数、异常数，尽量用窗口函数放到每行。
- 排序分页：输出页面排序字段，解释“查到了但不在当前页”的可能性。

结果集必须让 Codex 能直接判断“为什么页面看到的是这个样子”，而不是只证明某张表有没有数据。

## SQL 包格式

默认输出一个脚本。优先一个结果集；确实需要多组明细时，用 `TAG` 字段区分，或说明“同一脚本会返回多组结果”。

推荐骨架：

```sql
WITH p AS (
    SELECT
        '替换为关键词/编号' AS p_key,
        '替换为当前登录YHID001' AS p_user,
        CAST(NULL AS VARCHAR2(64)) AS p_optional_id
    FROM dual
),
raw_data AS (
    SELECT ...
    FROM ...
    CROSS JOIN p
    WHERE ... -- 最宽但有边界的候选条件
),
diag AS (
    SELECT
        r.*,
        CASE WHEN ... THEN 'Y' ELSE 'N' END AS will_show,
        TRIM(BOTH '；' FROM
            CASE WHEN ... THEN '原因1；' ELSE '' END ||
            CASE WHEN ... THEN '原因2；' ELSE '' END
        ) AS not_show_reason,
        CASE WHEN EXISTS (...) THEN '页面显示为已勾选且禁用' ELSE '页面显示为可勾选' END AS ui_state
    FROM raw_data r
    CROSS JOIN p
)
SELECT
    USER AS db_user,
    SYS_CONTEXT('USERENV','CURRENT_SCHEMA') AS current_schema,
    SYS_CONTEXT('USERENV','DB_NAME') AS db_name,
    COUNT(*) OVER () AS candidate_count,
    SUM(CASE WHEN will_show = 'Y' THEN 1 ELSE 0 END) OVER () AS page_count,
    will_show,
    NVL(not_show_reason, '满足全部页面过滤条件') AS diagnosis_reason,
    ui_state,
    ...
FROM diag
ORDER BY ...
```

输出给用户时只保留必要说明：

- “只需要改这几个参数”
- “执行后把完整结果集发我”
- “如果结果集太大，先发前 50 行和汇总列”

不要把源码推导过程转嫁给用户，也不要让用户分多步手工判断。

## 回传结果判定

先读库身份和汇总列，再读原因列：

- `page_count = 0`：按 `diagnosis_reason` 归因到具体过滤、权限、状态或断链条件。
- 有 `will_show='Y'` 但页面无数据：优先核对登录人参数、库/schema、代码版本、分页排序、前端二次过滤。
- 显示为禁用/已选中：不是查询不到，而是关联表或状态函数导致 UI 不可操作。
- 结果证明当前业务主键无主表/无候选：给一个最小定位 SQL，不扩散到全库扫描。
- 结果集缺关键字段：只追加一个最小增量 SQL，不让用户重跑全套。

结论必须包含：命中的过滤条件、证据字段、是否符合老/新 OA 真实逻辑、下一步只读查询或待审批修复方向。

## OALS 审批断链特别规则

遇到审批页空白、审批环节列表无数据、`OALS001` 不一致、已处理但无时间/意见、移动端和 PC 审批痕迹不一致时，读取 OALS 参考文件。

判定顺序固定为：

1. 页面/业务表当前查哪个 `OALS001`。
2. 当前流水下是否有 `OA003` 申请头、`OA001` 待办/已办、`OA010` 操作日志。
3. 同一业务主键是否存在旧流水审批痕迹。
4. 是否存在 `DBCL002='02'` 但 `DBCL003/DBCL004` 为空。
5. `F_DB_COMMON_SPLC(ywlx, oals, '')` 对 PC 页面是否返回空。

只读结果确认前，不给可执行修复 DML。确认后也只输出待审批修复草案和校验 SQL，不替用户改生产。

## 状态标签

- `readonly-sql-ready`：已给出一段可执行只读 SQL 包。
- `needs-user-result`：等待用户回传结果集。
- `diagnosed-from-result`：已根据结果集给出根因。
- `needs-one-delta-sql`：只缺一个最小增量 SQL。
- `blocked-needs-source-entry`：缺入口线索且无法从现有材料定位。
- `blocked-unsafe-write`：用户要求生产写入或副作用动作，已停止在安全边界。
