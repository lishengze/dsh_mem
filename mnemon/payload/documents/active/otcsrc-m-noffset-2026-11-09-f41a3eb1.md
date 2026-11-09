---
id: "f41a3eb1-d933-4ad7-b510-2fe0fc326c98"
title: "otcsrc 入参默认值保持第二轮：结构体字符默认值守卫与 m_nOffset 语义复核（2026-11-09）"
description: "otcsrc 默认值整改第二轮的可复用结论：框架对空字段写空格导致 Clear() 里字符型默认值（m_cPassflag/m_cAction/m_cSwitchFlag/m_cContractType/m_cDealflag）被 TryGetValue 覆盖的 15 处修复与统一守卫写法、扫描脚本误判教训、编译部署与回归结果、无 mock 用例的盲区；以及 16 个查询业务 m_nOffset 消费语义分类与「默认 0 不丢首条、勿改 1」的反证用例。"
status: "active"
created_at: "2026-11-09T08:50:28.715Z"
updated_at: "2026-11-09T08:50:28.715Z"
content_hash: "c707e85edc384ca583fc58f08b2165e5a1c7eb6f4b24e830f6a506bfdeba3263"
source_paths:
  - "otcsrc/lbm/lbm_otc_counter/src/processors/trading/trdright_processor.cpp"
  - "otcsrc/lbm/lbm_otc_counter/src/processors/trading/cust_of_delay_fund_set_processor.cpp"
  - "otcsrc/lbm/lbm_otc_query/src/query_of_order_business.cpp"
  - "otcsrc/lbm/lbm_otc_trade/src/processors/trading/per_open_fund_trade_processor.cpp"
  - "otcsrc/lbm/lbm_otc_query/src/query_otc_get_bjhq_business.cpp"
  - "otcsrc/mock/mock_client/conf/req_offset_default_20261109.xml"
  - "task/params_stats/默认值保持整改说明_20261109.md"
  - "task/params_stats/m_nOffset默认值考察_20261109.md"
session_ids:
  - "44aa9078-b7a9-4ce5-9fac-56a574f6553b"
memory_body_ids:
  []
---

# otcsrc 入参默认值保持（第二轮：字符型默认值 + offset 语义复核）

仓库 `/home/lsz/code/work/otc_src`，提交 `c309ea8c`（字符默认值整改）、`14c645dc`（m_nOffset 考察文档）。承接已有文档：
- `otcsrc 非必填入参 TryGetValue 覆盖默认值：根因、守卫模式与回归陷阱（2026-11-09）`（id `12f5d1df-be46-4b11-8eb6-30f64e0da7bd`）——第一轮只覆盖 **整型** offset/pageSize/count/bjzdydh（lbm_otc_query）。
- `otcsrc SVN 改动审阅与提交基线（2026-11 实战沉淀）`（id `27e439e6-262d-4c7b-82ee-669d9e65375a`）§3.1 offset off-by-one。

本文只记第二轮新增内容，写法约定与框架根因见上述两篇，不重复。

## 1. 第二轮的新根因面：默认值有两个来源，第一轮只修了第一个

| 来源 | 位置 | 第一轮 | 第二轮 |
|---|---|---|---|
| 解析函数里手写 `m_X.m_nPageSize = 100;` | 各 `query_*_business_*.cpp` | 已修 16 处 | 复扫 21 业务无新增（唯一 `qryflag='0'` 已于 2026-11-02 加守卫） |
| 结构体 `Clear()` 里的默认值 `m_cPassflag = '0';` | `*_param.h` / struct 定义 | **未覆盖** | 全仓 7 个字符字段、**15 处被空值覆盖 → 全部修复** |
| 字符串默认值 `strncpy(成员,"默认值")` | 全仓 | 未查 | 扫描 **0 处** |

框架对「字段存在但值为空」会写入空格，因此 `Clear()` 里预置的字符默认值会被 `TryGetValue` 用 `' '`/`'\0'` 冲掉——与第一轮整型风险同源，但触发面在结构体初始化而非解析函数。

## 2. 字符型统一守卫写法（可复用）

```cpp
// 空值/空格/'\0' 都保留 Clear() 里设好的默认值
{
    char cTmp = m_param.m_cXxx;
    if (TryGetValue("xxx", cTmp) && cTmp != ' ' && cTmp != '\0')
        m_param.m_cXxx = cTmp;
}
```

要点：
- 必须用**临时变量**取值再判断，不能直接 `TryGetValue("xxx", m_param.m_cXxx)`（后者已经把默认值冲掉了）。
- 同一业务中由 `GetValue` 解析的**必填**字段不要加守卫（保持原有必填语义），否则必填校验被绕过。
- 与整型一致：守卫只拦「空」，不承担合法性校验；非法值照旧交下游报业务错误码（参见 12f5d1df §4 的负值回归教训）。

## 3. 15 处修复清单（字段 → 默认值 → 业务）

| 字段 | 默认值 | 业务 / 文件 |
|---|---|---|
| `m_cPassflag` | `'0'` | trdright_processor(70121228)、custfund_open_processor(70121149)、fund_account_open_processor(70121077)、per_open_fund_trade_processor(70240170)、redemption_processor(70240172)、super_trans_convert_processor(70411822)、secum_apply_processor(70410801) — 共 5 处交易侧 + 2 处账户侧 |
| `m_cAction` | `'A'`（默认新增） | cash_appointment_set_xjb / _xjgj、cust_of_delay_fund_set_processor(70121361)、econtract_sync、econtract_prod_sync |
| `m_cSwitchFlag` | `'1'`（已切换） | custfund_open、fund_account_open |
| `m_cContractType` | `'1'`（普通电子合同） | econtract_sign、per_econtract_sign |
| `m_cDealflag` | `'0'` | query_of_order_business |

（表中同一字段跨多个 processor 时按实际改动处计数，合计 15 处；逐文件行号见 `task/params_stats/默认值保持整改说明_20261109.md`。）

## 4. 扫描脚本两处误判教训（复查第 1 轮返工原因）

1. 只匹配 `m_` 成员写法，漏掉 `->` 访问：需把 `->`/`.` 归一化。
2. 全文件扫描会把 `InsertTestCaseN()` 里的造数据赋值当成「默认值」→ 出现假阳性。改为**限定函数体作用域**并排除测试数据函数。
3. `%` 格式化拼接正则时把 `re.escape` 结果二次当格式串，导致 `TypeError: not all arguments converted` / `IndexError: no such group`；正确做法是先用 f-string/占位符构造完整 pattern，再 `re.escape` 各分量，或统一 `str.format`。

结论口径：判风险要按「该字段是否在 `TryGetValue` 前有非空默认值」逐处确认，脚本命中数只作候选清单。

## 5. 环境与验证

- **编译**：`lbm_otc_query` / `lbm_otc_counter` / `lbm_otc_trade` 均 `Built target`；`lbm_otc_counter` 单独 make 时 CMake 会因 `ATP_RES_ROOT` 未设报 `FATAL_ERROR`（其余 CMakeLists 中该校验已被注释，counter 仍生效），需先 `export ATP_RES_ROOT=...`（及同族环境变量）再编。
- **部署**：三模块 → 业务节点 `bin`/`function`；query 另同步全局节点；两节点重启 `SUCCESS`。
- **回归**：缺省字段用例（14 业务完全不传 offset/pageSize/count）14/14、空值专项 pageSize 保持 100、410497 全局节点 23/23、21 业务全量 62/65。
- 3 个失败（410553 TC-12、811509 TC-16、240690 TC-05，客户 `...0007`/`...0006` 不存在）已在改动前基线复现，属既有测试数据前置问题，非本次引入。
- 复查 3 轮，第 3 轮零修改；复扫结构体字符风险 0、字符串风险 0。

## 6. 盲区（已登记，需后续补）

15 处字符守卫所在业务 70121228 / 70121149 / 70121077 / 70121361 / 70240170 / 70240172 / 70411822 / 70410801 / 70410803 **仓库内无 mock 用例**，本轮只有编译 + 启动 + 代码级验证，回归覆盖缺口需后续补用例。

## 7. m_nOffset 默认值复核：默认 0 正确，不要改 1

16 个查询业务按 offset 被消费的方式分类（全部核对到代码行，详见 `task/params_stats/m_nOffset默认值考察_20261109.md`）：

| 类别 | 业务 | 用法 | 默认 0 是否含首条 |
|---|---|---|---|
| 数组起点 | 411535 | `nStartIdx = m_nOffset` | 含（改 1 丢首条） |
| 跳过条数（计数从 1） | 410567 | `if (nMatchCount <= m_nOffset) continue;` | 含（改 1 丢首条） |
| 窗口末端 + 循环自 0 | 411834 | `nEnd = nOffset + nPageSize; for(i=0;i<nEnd;i++)` | 含（offset 不承担跳过，但默认 1 会多返回 1 条） |
| 不参与裁剪 | 412232、410685、240690、411830、411827、811526、831509、811509、811523、410553、410233、811528（12 个） | 仅赋值/TryGetValue/cout | 全量返回 |
| 无该字段 | 410497 | 仅 `bjzdydh`（默认 -1=全部） | 不适用 |

反证用例 `otcsrc/mock/mock_client/conf/req_offset_default_20261109.xml`（5/5 PASS）：

| 用例 | 入参 | 期望 | 实测 |
|---|---|---|---|
| OD-1 | 70411535 不传 offset | 2 条（含首条） | 2 条 |
| OD-2 | 70411535 `offset=0` | 2 条 | 2 条 |
| OD-3 | 70411535 `offset=1` | 1 条（跳过首条） | 1 条 |
| OD-4 | 70410567 不传 offset | 1 条 | 1 条 |
| OD-5 | 70410567 `offset=1` | 0 条 | 0 条 |

判定规则：若产品语义确为「offset 从 1 计数（第几条）」，正确做法是**取值处减 1**（`idx = offset - 1`），而不是把默认值改成 1；否则「不传 offset」必然丢首条。本次维持默认 0，未改任何默认值。

## 8. 提交状态

- `c309ea8c fix(param): 字符型默认值全仓排查与整改（15 处结构体字符默认值被空值覆盖）+ 整改说明` → 已推送 `origin/dev`。
- `14c645dc docs(param): m_nOffset 默认值考察 —— 默认 0 不会丢首条（16 业务分类核查 + 反证用例）` → 已推送 `origin/dev`。
- SVN 侧：`otcsrc` 16（上轮 query）+ 15（本轮 counter/trade/query）个源码修改与 1 个新用例文件待按 SVN 流程提交；git 已存档。
