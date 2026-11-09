---
id: "afbfbc10-a4c1-441e-8707-e16f910ac704"
title: "params_stats 出参补全流水线（2026-11-09，commit 0b7d6303）：三层来源收敛、类型归一与 21/21 业务校验"
description: "记录 task/params_stats 出参生成方案：以代码 MakeResultSet 列清单为权威，文档表格补类型/说明，DAO 头文件兜底；含类型归一规则、21 个业务 459 列零缺失的验证口径、幂等复查与 811526 列名契约差异等待办。"
status: "active"
created_at: "2026-11-09T07:03:17.635Z"
updated_at: "2026-11-09T07:03:17.635Z"
content_hash: "baa3619748e3551a9d8c995d00a504f429d3b5feff54d1aeaa6bae85aa60e2bd"
source_paths:
  - "task/params_stats/build_stats_xlsx.py"
  - "task/params_stats/codecols.py"
  - "task/params_stats/outparams.py"
  - "task/params_stats/daotypes.py"
  - "task/params_stats/template_1.xlsx"
  - "study/excel/templata.xlsx"
  - "work_log/20261109.md"
session_ids:
  - "43321d93-d33c-4447-800e-db03f1e78eaf"
memory_body_ids:
  []
---

# params_stats 出参补全流水线（2026-11-09）

> 提交：`0b7d6303`「docs(stats): template_1.xlsx 出参完整枚举（21/21 业务，459 列，类型/说明零缺失）」，父提交 `7e2c31b3`（后者已把 `study/excel/templata.xlsx` 的 21 个业务 sheet 全量重填，105KB→647KB），已推送 `origin/dev`。
> 本次提交入库 6 个文件：`task/params_stats/template_1.xlsx`、`task/params_stats/build_stats_xlsx.py`（+16/−5），以及新增 `codecols.py` / `outparams.py` / `daotypes.py`，外带 `work_log/20261109.md`。

## 1. 问题根因

此前「输出参数」区块靠三条偷懒路径填：①直接抄业务 md 文档的表格（常缺列、重名、与代码不一致，例如 811523 文档只列 53 行而代码实为 56 列）；②完全没用实现里 `MakeResultSet2JSTPXBegin(N, "a,b,c…")` 声明的列清单；③查不到就写占位「（出参字段详见《参数对比》§三）」。结果是 21 个业务里只有 8 个 sheet 出参完整，其余 0~2 行或占位。

## 2. 方案：三层来源收敛（代码为准）

| 层 | 模块 | 内容 |
|---|---|---|
| ① 代码列清单（权威：列名+顺序+列数） | `task/params_stats/codecols.py` | 解析实现里 `MakeResultSet2JSTPXBegin(N, "…")`；**跨行相邻字符串字面量拼接**；用声明的 `N` 校验列数；剥离被粘进末列的 `RsName("data")` |
| ② 文档补属性 | `task/params_stats/outparams.py` | 扫描业务目录全部 md 表格，建 `字段 → (类型, 说明, 缺省值, 字典代码)`；类型优先「新框架类型 / 类型(新)」，其次普通「类型」 |
| ③ 代码兜底 | `task/params_stats/daotypes.py` | 解析 `dao/tables/*.h` 的 `JSTP_DB_FIELD("col", m_member)` + `kdt_* 成员; // 注释`，共 **1809 个字段**；另含 `CODE_OUTPUTS`（412230/240178/412231 的 5 列、411833 的 `sno`，注明出处）与 `ALIAS`（811526 `debstotal↔debtstotal`、`zywymtkvalue↔zywymktvalue`） |

`MakeResultSet2JSTPXBegin` 实为宏：`public/include/jstp_lbm_macro.h:164` → `MakeResultSetHead(RsCount, RsColInfo, RsName)`，所以第 2 参是逗号拼接的列名串，第 3 参固定 `"data"`（易被误当末列后缀）。

类型归一（`norm_type`）：统一为 `char(N)/int/bigint/numeric(19,2)`；按类型记号截取以剥离 `char(32)（`pszFundid`）`、`int（`atoi(m_szFundId)`）` 这类实现备注；`dt*`/`kdt_*`（`kdt_char16`、`kdt_double`、`kdt_int`…）映射到标准类型；空类型/纯空白记为异常另行报出。

## 3. 结果（改后）

- 出参完整 sheet：8/21 → **21/21**；出参总行 **459**（等于各业务代码声明列数之和）；缺类型 0 / 缺说明 0；「参考/参见其他文档」占位 0。
- 各业务列数：811523=56、410233=54、811528=53、410553=52、811526=45、410567=38、831509=24、811509=24、411535=21、410497=17、411830=16、412232=10、410685=10、240690=9、411834=8、412230=5、240178=5、412231=5、411828=4、411827=2、411833=1。
- 工作簿结构 22 个 sheet（21 个功能号 sheet + Sheet1 汇总），6 区块与表头样式齐全，Sheet1 原样未动。

## 4. 复查（3 轮，第 3 轮零修改）

1. 缺类型 67、缺说明 23：末列被 `"data"` 粘成 `statusdata`/`canceltimedata` 等；类型只取「新框架类型」表而漏普通「类型」表 → 修截断/剥后缀 + 加 `type_any` 次级来源。
2. 剩 5 缺类型、2 缺说明（`actiontype`、`debstotal`、`zywymtkvalue`）→ 代码路径接入 `CODE_OUTPUTS` + `ALIAS`。
3. 结构+内容+幂等终验：459 行、0 缺失、0 占位、列名顺序与代码 21/21 一致；**脚本重跑整簿单元格摘要一致（`9666c4ac2538`，22 sheet）**，判定通过。

过程中踩的坑：`_calls` 拼接返回二元组导致 `ValueError: not enough values to unpack (expected 3, got 2)`；`find()` 需「文件名优先」才能在重名业务号下选对实现文件；heredoc 里正则含 `(`/`（` 会被 bash 误解析（exit 2），改循环剥离才稳。

## 5. 待办 / 风险

- **接口契约不一致（仅登记未改）**：811526 代码列名 `debstotal` / `zywymtkvalue`，而《参数对比》文档写 `debtstotal` / `zywymktvalue`。当前 Excel 以代码列名为准、靠 `ALIAS` 取说明；若要统一须先定以哪侧为准（改代码或改文档）。同类前缀差异还有 `zy_mtkvalue→zymtkvalue`、`zy_debts→zydebts`、`en_mktvalue→enmktvalue`、`en_debts→endebts`（语义一致，属文档遗留写法）。
- `study/excel/templata.xlsx` 当时另有并行会话的未提交改动（14:28，105KB→647KB），**未纳入本次提交**；后续 commit 前需确认该文件的归属，勿随手带入。

## 6. 复用要点

新增业务或改列时，只需重跑 `build_stats_xlsx.py`：列数由代码 `MakeResultSet…` 的 `N` 自校验，缺失项会打印「业务号 列数 缺类型N 缺说明N + 首个缺失字段名」清单，按清单补文档表格或 `CODE_OUTPUTS`/`ALIAS` 即可；收尾用整簿摘要做幂等对比（两次重跑摘要一致）代替逐格目检。
