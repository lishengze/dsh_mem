---
id: "55159cea-4ff5-44cd-b310-53c99d424c29"
title: "非 4 开头 7 业务公共入参改 g_ 口径（2026-11-10）：GetGeneralParamOper 迁移、JSTP_LBM_TRY 时序段错误坑与回归结论"
description: "7 个非 4 开头业务（240690/240178/811523/811526/811528/831509/811509）公共入参改 g_ 口径的实施记录：GetGeneralParam 与 GetGeneralParamOper 两套口径判定、GetGeneralParam 必须在 JSTP_LBM_TRY 内否则空句柄段错误、清库导致种子数据丢失的误报排除、A/B 与 165 用例回归结果、参数对比文档/Excel 同步与 SVN 组5 待提交清单。"
status: "active"
created_at: "2026-11-10T06:19:46.779Z"
updated_at: "2026-11-10T06:19:46.779Z"
content_hash: "df48e22063b431500fd16ab8dead10783cae1f71d36996f6b9b566b19ad3f1e9"
source_paths:
  - "study/240690_基金投顾子账户资产查询/240690_参数对比.md"
  - "study/240178_基金投顾策略解约/240178_参数对比.md"
  - "study/811528_留存业务客户资产总值分类查询/811528_参数对比.md"
  - "study/811523_客户资产总值分类查询/811523_参数对比.md"
  - "study/811526_客户级别客户资产总值分类查询/811526_参数对比.md"
  - "study/831509_客户总资产查询/831509_参数对比.md"
  - "study/811509_客户总资产明细查询/811509_参数对比.md"
  - "lbm/lbm_otc_query/src/query_strategy_qryasset_business_240690.cpp"
  - "lbm/lbm_otc_counter/include/processors/risk/strategy_cancel_processor.h"
  - "study/excel/templata.xlsx"
  - "task/params_stats/template_1.xlsx"
  - "task/svn_conflict/task_svn.txt"
  - "work_log/20261110.md"
  - "study/knowledge_base_system/02_业务开发模式.md"
session_ids:
  - "6cdc7f97-8435-4eae-a822-9d329a1f850f"
memory_body_ids:
  []
---

# 非 4 开头 7 业务公共入参改 g_ 口径（2026-11-10）

> 工作副本 `/home/lsz/code/work/otc_src`；Git `ceced6ec`「fix(param): 7 个非4开头业务公共入参改回 g_ 口径（GetGeneralParamOper / GetGeneralParam）」已推 `origin/dev`（e02609b4..ceced6ec）。
> 范围是**非 4 开头业务**：240690 / 240178 / 811523 / 811526 / 811528 / 831509 / 811509（查询类），此前分析轮把它们列为「范围外、未动」（见文档 `8f722b2a-1a8e-4954-be70-20d895ae7be9` §8）。4 开头 14 业务的改造与验证见 `5d13fd14-8697-419b-8332-de3bebc6094a`；参数对比文档/Excel 去重规则见 `a09a0e67-6e2b-4e64-a153-999429b6372d`。

## 1. 两套公共入参口径（不可混用）

| 口径 | 接口 | 前缀 | 目标结构体 | 适用 |
|---|---|---|---|---|
| 4 开头外围 | `CQueryBase::GetGeneralParam()` | 无前缀（`funcid/custid/custorgid/trdpwd/netaddr/orgid/operway/ext/operwayext/userinfo`，即 `GETVALUE_PERPARAM` 10 项） | `m_QryGeneralParam` | 4 开头外围接口 |
| 非 4 开头（BOS/老系统） | `CQueryBase::GetGeneralParamOper()` / `CBaseProcessor::GetGeneralParam()` | **`g_` 前缀**（`g_funcid`/`g_operid`/`g_opbrhid`/`g_operway`/`g_operwayext`/`g_stationaddr`/`g_userinfo`/`g_operpwd`） | `tagQryGeneralParamOper`（`lbm/lbm_otc_query/src/...`；counter 侧对应 `tagParamParam` 新增 `m_nOperId`/`m_cEstimated`） | 非 4 开头业务 |

两者解析逻辑与字段集合不同。全部用 `TryGetValue()` 解析，缺省空/0，**框架侧不做必填校验**（4 开头侧另有「必填性由 GetValue 抛错承担」的历史隐式契约，见 `8f722b2a` §3）。

## 2. 7 个业务的现状与修正

| 业务 | 原状况 | 修正 |
|---|---|---|
| 811523 / 811526 / 831509 / 811509 / 811528 | 误调 4 开头专用 `GetGeneralParam()` | 改 `CQueryBase::GetGeneralParamOper()`（`g_` 前缀，填 `tagQryGeneralParamOper`） |
| 240690 | 完全没有公共解析 | 补 `GetGeneralParamOper()` |
| 240178 | `GetGeneralParam()` 调用被注释（commit `ee0f7dde`） | 恢复调用，并**移入 `JSTP_LBM_TRY` 宏内** |

## 3. 关键坑：公共解析必须在 `JSTP_LBM_TRY` 内（否则段错误）

恢复 240178 调用后，70240178 每个请求直接 `signal 11`。抓 core + gdb：

```
CStrategyCancelProcessor::Execute → GetGeneralParam → TryGetValue("g_funcid")
→ JstpPackGetValueStr(handle=0x0) → 段错误
```

**根因**：参数包句柄为空。`GetGeneralParam()` 写在 `JSTP_LBM_TRY` **之前**，而参数包是在宏内 `Init(pCA, …)` 才初始化的。

**修法**：把公共解析移到 `JSTP_LBM_TRY` 内、`GetThreadPoolHandle()` 之后（与 `dzzh_manual_adjust` 等既有业务一致）。修好后 70240178 从 4/11 → 6/11。

**通则**：非 4 开头业务的公共入参解析位置与 `Init()` 同属宏内生命周期，放在宏外必然拿到空句柄。该坑与本轮其余结论已记入知识库 `study/knowledge_base_system/02_业务开发模式.md`（§5.3「非 4 开头业务的公共入参口径 + `JSTP_LBM_TRY` 时序坑」）。

## 4. 环境故障：内存库 DDL 与新表定义不一致

节点重启时报 `32006 Module [kocadb] start failed: Failed to create table [trd_oftafilec1] 字段数不匹配`。

- **原因**：仓库在 12:55 被更新（commit `e02609b4` 改了 `dao/tables/trd_oftafilec1.h`），内存库仍是旧 DDL。
- **处置**：`bin/rm_kmdb.sh -y` 清库重建后启动正常。
- **副作用**：**库内种子数据被清空**——本轮后续失败的测试用例大多与此有关，判断失败原因时务必先排除这一项。

## 5. 验证与 A/B

- **14 个已优化业务**：136/136 全绿（回归中一并复跑）。
- **本轮 7 个业务**：29 条用例，两轮分别 22 / 21 通过。

| 业务 | 用例 | 通过 | 失败原因 |
|---|:--:|:--:|---|
| 70811528 / 70811523 / 70811526 / 70831509 | 2/3/3/3 | 全过 | — |
| 70240690 | 3 | 2 | TC-05 客户状态非正常：库中无该客户（2026-11-09 已记录为改动前既有失败，期望 `240690006` 实得 `240690005` 客户信息不存在） |
| 70811509 | 4 | 3 | TC-16 客户不存在（同为改动前既有失败） |
| 70240178 | 11 | 5~6 | 无签约记录 / 客户 0005、0006 与资金账户 0003 不存在；该业务无 `testcase_num` 造数能力 |

- **A/B 验证**：临时注释掉 240690 的 `GetGeneralParamOper()` 重建复跑 → 仍 2/3、同一 TC-05 失败，证明**解析口径修正不改变业务行为**，剩余失败全部是测试数据前置缺失（清库后种子数据丢失）。
- **末轮最终回归**：TOTAL=165 PASS=157，失败项与上表同源。
- **部署一致性**：`liblbm_otc_query.so`（`0dc545e9`）与 `liblbm_otc_counter.so`（`61ad6d0f`）在 `lbm_*/build/` 与 `kgbp_x64_linux/bin/` 两侧哈希一致，且已部署 lib 内含 `GetGeneralParamOper()` 调用字符串。

## 6. 文档与 Excel 同步

- **7 份 `study/<funcid>_*/<funcid>_参数对比.md`**：重写 §1.1/§1.2/§1.3（旧系统集中营运 `g_*` ↔ 新框架 `GetGeneralParamOper()` / counter `GetGeneralParam()`），§5 全面对比表的公共行由「GETVALUE_PERPARAM 10 项」替换为 `g_*` 行（查询 7 行、240178 8 行），业务参数行保留并去重。
- **校验**：7 份文档均 table 1/1、§1.2 标题正确、无残留「10 项」字样；入参段无重复参数名，首个参数均为 `g_funcid`。
- **两个 xlsx 重建**：`study/excel/templata.xlsx`（21 sheet）与 `task/params_stats/template_1.xlsx`（21 sheet，含各业务 公共/业务/出参/错误 行数统计）；sheet 顺序不变（Sheet1 + 14 个 4 开头 + 7 个其它）。

## 7. 提交状态

- **已入库（SVN）**：组1 → `r77803`；组3 → `r77827`；组4 → `r77828`（2026-11-10 04:44 UTC，提交人 122098）。
- **新增组5**：7 文件（6 个 query + `lbm/lbm_otc_counter/include/processors/risk/strategy_cancel_processor.h`），已写入 `task/svn_conflict/task_svn.txt`。
- 待提交 = 组2(15) + 组5(7) = **22 个 M**，与 `svn status otcsrc/lbm` 完全一致（0 遗漏 / 0 多余 / 组间无重复）。

## 8. 遗留与待决策

- 70240178 等 7 条失败需**前置种子数据**（签约记录、特定状态客户/资金账户）才能闭环；可选方案是按场景补「先造数再跑」的用例，但会改变这些业务的测试用例组织方式，**待用户拍板**。
- 清库后所有依赖种子数据的用例都可能误报，重跑前先确认库内数据，或保留一份可重放的造数脚本。
