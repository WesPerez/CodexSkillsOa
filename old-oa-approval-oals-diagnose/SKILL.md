---
name: old-oa-approval-oals-diagnose
description: 诊断老 OA 审批页空白、审批环节列表无数据、OALS001 流水号不一致、业务表指向新流水但 OA001/OA003/OA010 缺失、已处理记录 DBCL002=02 但 DBCL003/DBCL004 为空、移动端显示已处理但 PC 不显示等审批流断链问题。用于合同/预算协议/支付/借支等老 OA 业务审批数据排查、生产只读 SQL 生成、原因判定和修复前证据链整理。
---

# 老 OA 审批流水诊断

## 使用原则

先判断“页面查哪个流水”，再判断“审批数据在哪个流水”，最后判断“处理痕迹是否完整”。不要只看业务表状态，也不要只看 OA001 的 `DBCL002`。

生产排查默认只读。需要修复数据时，必须先让用户执行只读 SQL 并回传结果，再给带备份、预期行数、回滚点、校验 SQL 的 DML。

## 可执行闭环原则

本技能必须给出下一条可执行诊断 SQL 或修复前证据链，不能只说“审批链断了”。

默认链路：

1. 定位业务类型和业务表。
2. 查业务表当前 `OALS001` / 状态 / 主键。
3. 查当前流水的 `OA001/OA003/OA010`。
4. 用业务主键反查历史流水。
5. 查异常已办和 PC 展示函数。
6. 输出原因标签和最小修复目标。
7. 生产库只生成只读 SQL 或待审批 DML 草案；开发/测试库也要先备份、预期行数、回滚点和校验 SQL。

如果用户只给页面现象或截图，先生成只读 SQL 包让用户在目标库执行；如果缺少业务主键，则给出从页面 URL、隐藏域、业务表、`SP001` 反查主键的 SQL。只有业务类型和主键都无法通过现有材料定位时，才要求用户补充。

## 调用契约

本技能是老 OA 审批流水诊断专用入口，通常独立使用；当被其他 OA 技能引用时，按以下契约交接：

- 输入：业务类型、业务表/主键、页面 URL/截图现象、当前 `OALS001`、用户可执行的目标库范围，或足以反查这些信息的线索。
- 输出：只读诊断 SQL 包、当前/历史流水对比、`OA001/OA003/OA010` 证据、PC 展示函数证据、原因标签、修复前最小证据链。
- 状态：`diagnosis-ready`、`needs-business-key`、`needs-user-readonly-sql-result`、`repair-dml-draft-only`。
- 生产库只输出只读 SQL 和待审批 DML 草案；不把诊断结果直接交给浏览器验收作为“已修复”证据。
- 如果后续需要新 OA 页面或迁移一致性判断，交给 `oa-business-logic-compare`；如果需要开发/测试库 DML 验证，交给对应 SQL 安全门禁，不在本技能内执行写入。

## 必查源码链路

优先读取这些对象，按本地实际路径：

- `cpzx-oa/sqlFiles/procedures/P_XTGL_COMM_SH_INIT.sql`
- `cpzx-oa/sqlFiles/procedures/P_KSXT_XYYSSP_TJ.sql`
- `cpzx-oa/sqlFiles/procedures/P_OA_DB_SHOW_SPLCTJ.sql`
- `cpzx-oa/sqlFiles/procedures/P_XTGL_COMM_SH_TJ.sql`
- `cpzx-oa/sqlFiles/procedures/P_OA_DB_SET.sql`
- `cpzx-oa/sqlFiles/procedures/P_OA_DB_FQ_SET.sql`
- `cpzx-oa/sqlFiles/procedures/P_XTGL_COMM_SH_PASS.sql`
- `cpzx-oa/sqlFiles/procedures/P_OA_DB_SET_D.sql`
- `cpzx-oa/sqlFiles/procedures/P_OA_DB_SHOW_PASS.sql`
- `cpzx-oa/sqlFiles/procedures/P_SH_MOBILE_YSXY_SH_PASS.sql`
- `cpzx-oa/sqlFiles/functions/F_DB_COMMON_SPLC.sql`

如果是 `YWLX001='140'` 预算协议/合同预算审批，还必须读取：

- `cpzx-oa/sqlFiles/procedures/P_KSXT_XYYSSP_TJ.sql`
- `cpzx-oa/sqlFiles/procedures/P_OA_DB_SHOW_SPLCTJ.sql`

## 关键机制

`OALS001` 新流水由 `P_XTGL_COMM_SH_INIT` 生成：

```sql
substr(f_sysdate_l, 0, 2) || '-' || in_curr_ywlx001 || '-' ||
lpad(seq_oa001_oals001.nextval, 10, 0)
```

其中 `substr(f_sysdate_l, 0, 2)` 取的是日期字符串前两位；例如 `2026-...` 得到 `20`，所以流水形如 `20-140-0000653193`。

业务提交过程通常是：

1. 业务提交过程调用 `p_xtgl_comm_sh_init`，生成新 `OALS001`、流程串 `SHLC001`、下一状态。
2. 业务表更新为新 `OALS001`、新 `PHZT001`、新 `SHLC001`。
3. `p_xtgl_comm_sh_tj` 调 `p_oa_db_set` 插入当前/下一环节 `OA001`。
4. `p_oa_db_fq_set` 基于第一条待办补 `OA003` 申请头。
5. 页面审批环节函数 `F_DB_COMMON_SPLC` 按传入 `OALS001` 查 `OA001`，且已办显示要求 `DBCL003 IS NOT NULL`。

因此，`OALS001` 只存在于业务表不代表审批链完整。至少要同时核对：

- 业务表当前 `OALS001`
- `OA001` 待办/已办
- `OA003` 申请头
- `OA010` 操作日志
- PC 展示函数返回值

## 140 预算协议高风险点

`P_OA_DB_SHOW_SPLCTJ` 的 `YWLX001='140'` 分支在重新提交时会：

1. 按当前 `OALS001` 找 `KS069`
2. 调 `p_xtgl_comm_sh_reset('140', fjid001, old_oals001, ...)`
3. `UPDATE ks069 SET phzt001='000', shlc001='', oals001=''`
4. 再调 `p_ksxt_xyyssp_tj`，生成并绑定新 `OALS001`

如果 reset/重提后 `OA001/OA003/OA010` 未完整落库，就会出现“业务表指向新流水，旧流水保留审批痕迹，新流水页面空白”。

## 判定规则

用以下结论标签：

- `业务表空壳流水`：业务表当前 `OALS001` 有值，但 `OA001/OA003/OA010` 全部或关键记录缺失。
- `旧流水残留审批`：同一 `ZJID003/FJID001` 在旧 `OALS001` 下存在 `OA001/OA003/OA010`。
- `半成品已办`：`OA001.DBCL002='02'`，但 `DBCL003` 或 `DBCL004` 为空。
- `PC 不显示原因明确`：`F_DB_COMMON_SPLC` 查当前流水为空，或对应环节 `DBCL003 IS NULL`。
- `移动端/PC 落库不一致`：移动端显示已处理，但缺 `OA010` 通过日志或缺 `DBCL003/DBCL004`。

## 固定排查顺序

1. 定位业务类型：查 `SP001`，确认业务表、主键字段、业务类型。
2. 查目标业务表当前状态：当前 `OALS001`、`PHZT001`、`SHLC001`、创建人、更新时间。
3. 用当前 `OALS001` 查 `OA001/OA003/OA010` 行数和明细。
4. 用业务主键 `ZJID003/FJID001` 反查所有历史 `OALS001`。
5. 对比当前流水与历史流水：哪个有申请头，哪个有审批行，哪个有日志。
6. 查异常已办：`DBCL002='02' AND (DBCL003 IS NULL OR DBCL004 IS NULL)`。
7. 查 PC 展示函数：`F_DB_COMMON_SPLC(ywlx, oals, '')` 返回长度/HTML。
8. 按规则输出原因，不把缓存、浏览器或页面刷新作为首因。

## SQL 模板

需要生产只读 SQL 时，读取 `references/diagnostic-sql.md`。

## 输出要求

回答必须包括：

- 当前业务表指向哪个 `OALS001`
- 哪些 `OALS001` 下有 `OA001/OA003/OA010`
- PC 页面为什么空
- 已处理但空时间/意见的具体字段原因
- 是否存在重新提交/重置造成的流水分裂证据
- 下一步修复应补哪几类数据，不直接给无验证 DML
