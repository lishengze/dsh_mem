---
id: "9edbc756-0ad0-4081-81e8-475f7983f83e"
title: "GT 三仓 `policy_id` 字段贯通修改方案（TCP API + 共享内存 API，2026-11-12）"
description: "gt_trunk 工作区 `svn_work/108_policy_id贯通修改方案_20261112.md` 的知识摘要：三处结构体（gw_head.h / message:: / share_common::）尺寸与偏移实测、policy_id 三断点、偏移 106 撞车根因、SHM 无垫片必须同批升级的兼容性结论、六项测试方案与 T5 观测点、实施与三仓同步约束。"
status: "active"
created_at: "2026-11-12T10:24:33.708Z"
updated_at: "2026-11-12T10:24:33.708Z"
content_hash: "ca6874369ce92975b56a16b73b7c7d379a6edbbb16bdc59a3602b6941369c7c2"
source_paths:
  - "svn_work/108_policy_id字段贯通修改方案_20261112.md"
  - "work_log/20261112.md"
  - "DYS-FRAMEWORK/fte/fte/include/share_communication/share_comm_external_message.h"
  - "DYS-FRAMEWORK/fte/include/message/gw_head.h"
  - "DYS-FRAMEWORK/api/gw_api_boost/api/src/ute_client_handler_impl.cpp"
  - "DYS-FRAMEWORK/api/gw_api_no_boost/api/src/ute_client_handler_impl.cpp"
  - "DYS-FRAMEWORK/fte/dse/dse_client/src/dse_client.cpp"
  - "DYS-FRAMEWORK/fte/test_all/strategy_test/src/shm_test_client.cpp"
  - "DYS-FRAMEWORK/api/gw_api_boost/ute_mock_client/src/perf_runner.cpp"
  - "DYS-FRAMEWORK/api/gw_api_boost/ute_mock_client/api_net_time_design.md"
session_ids:
  - "bed849ee-ab12-4319-9a56-95c24cd0e06e"
memory_body_ids:
  []
---

# GT 三仓 `policy_id`（策略编号）字段贯通修改方案

**一句话**：需求方要求"FTE 内部 `TradeOrderInfo` 增加 `policy_id`，因此要改 API"，实测该内部字段 **2026-11-02 已完成**（提交 `ad3b0f9ba`，含 `reset/decode/encode/dump` + 128 字节整包 `static_assert`）；真正缺的是 **三个入口断点**，其中共享内存通道**没有长度垫片**，必须与策略端同批升级。

交付物：`svn_work/108_policy_id字段贯通修改方案_20261112.md`（535 行 / 8 章 + 附录，位于 `/home/lsz/code/work/gt_trunk`）；提交 `b02da5345` 已推送，三仓 `0 0`。**本次只产出方案，未改任何源码。**

## 1. 三断点（链路图结论）

| 断点 | 位置 | 后果 |
|:-:|---|---|
| ① AGW API → FTE 转发未赋值 | `DYS-FRAMEWORK/api/gw_api_{boost,no_boost}/api/src/ute_client_handler_impl.cpp`，各 4 处 `process_*` | `info.trade_order_info.policy_id` 是**未初始化内存**（结构体无构造函数且未 memset）→ 随机值进入 FTE/DSE |
| ② 共享内存结构缺字段 | `DYS-FRAMEWORK/fte/fte/include/share_communication/share_comm_external_message.h`，`share_common::TradeOrderInfo` = 36B | FTE 按 `message::TradeOrderReq` 解读 SHM 缓冲，偏移 106 实际读到 `ulStrategyKey` 低 2 字节 = 策略 Key 低位 |
| ③ 对外 TCP API 头缺字段 | `DYS-FRAMEWORK/fte/include/message/gw_head.h`，`TradeOrderReq` = 106B | < FTE `TradeOrderReqLayout` = 108B → `prep_order` 判 `kInvalid`，报 `order req length 106 too short` **直接拒单** |

两条通道：A = TCP（端口 33001/33002，`AgwMsgOrder`），B = 共享内存（`Strategy_UTE_ReqOrderUTE_<inst>.queue`，`share_common::TradeOrderReq`）。两通道都汇入 FTE 的 `CheckBasicInfo → UtilityCheck`。

## 2. 结构布局实测（合并探针，可复现）

修复前（修复前探针输出）：

| | `gw_head.h`（对外 TCP） | `message::`（FTE/服务端） | `share_common::`（共享内存） |
|---|--:|--:|--:|
| `TradeOrderInfo` | 内联 | 38 | **36** ❌ |
| `TradeOrderReq` | **106** ❌ | 116（layout 108 + reserved 8） | **114** ❌ |
| 整包 | 118 ❌ | 128 | — |

**根因一句话**：`message::TradeOrderInfo::policy_id` 在偏移 **106**，而 SHM 的 `ulStrategyKey` 也恰在 **106** —— 偏移 106 具双重语义，撞车。

修复后目标值（方案期望）：

| | `gw_head.h` | `message::` | `share_common::` |
|---|--:|--:|--:|
| `TradeOrderInfo` | — | 38 | **38**（+`policy_id`）|
| `TradeOrderReq` | **116**（+`policy_id`+`reserved[8]`）| 116 | **116** |
| 整包 | 128 | 128 | — |
| `TradeOrderER` / `OrdERInfo` | — | 332 / 262 | 332 / 262（一致，不回显 `policy_id`）|
| `CancelOrderReq` | 86 | 116 | 86（无 `policy_id`）|

关键偏移：`message::TradeOrderReq` 的 `trade_order_info` @70、`policy_id` @36（绝对 106）、`reserved` @108。

## 3. 方案要点

1. **SHM `share_comm_external_message.h`（必改）**：在 `stop_px` **之后**加 `unsigned short policy_id;` → 落在偏移 106，`ulStrategyKey` 顺势移到 108（正对 FTE 的 `reserved`，不被读取）。**SHM 处理接口一处都不用改**（策略端写 Key / 服务端取 Key / 长度校验全部随结构体自洽），另加 4 条 `static_assert` 固化字节契约（尺寸/偏移）。
2. **`gw_head.h`（必改，5 处）**：字段 + `reserved[8]` + `reset`/`dump`/`encode`/`decode` → 116B 与服务端逐字节一致（走 `kPass`）。只补到 108B 也能被 `kReuse` 接受，但不推荐。
3. **API 两棵树 × 4 个 `process_*`（必改，8 处）**：各补 1 行赋值，并 `memset(&info, 0x00, sizeof(info));` 消除未初始化 `reserved`（结构体无构造函数，不推荐改对外契约头加构造函数）。
4. **兼容性（决定性）**：
   - **TCP 有垫片** —— `prep_order` 接受 ≥108（108~115 自动补零），可灰度发布；
   - **SHM 没有垫片** —— `mpmc_queue<T>` 的槽步长与共享内存段大小都由 `sizeof(T)` 推导（依据 `queue_manager.cpp` 的 `CreateShareMemory`：`uiMemorySize_ = sizeof(mpmc_queue<T>) + uiElementSlotBlocksSize + 128`，`create_shared` 手动映射 slot 到 `addr + sizeof(mpmc_queue<T>) + 32`）→ **FTE 与策略端必须同批升级**，此点必须写进发布说明。
5. **不改清单（已核实，避免误改）**：`fte/uteapixn/**`（遗留 API，未纳入任何构建）、`fte/api_demo/`（使用 `gw_head.h`，自动受益）、V1 `agw_external.xml`（当前统一用 V2）、`fte/include/message/trade_order_type.h`（死代码）、`agw_client_impl.cpp:1104`（Gone 路径已正确赋值）、`message_manager`。
6. **撤单路径无需改动**：`process_cancel_order` 构造的 `message::CancelOrderReq` 本身不含 `policy_id`；FTE 撤单按 `orig_clordno` 查原委托并继承（`uplink_biz_processor.cpp` `internal_order->orig_clordno = orig_order->clordno;`）。

## 4. 顺带发现（待业务确认）

`DYS-FRAMEWORK/fte/dse/dse_client/src/dse_client.cpp` 的 `FillCancelOrderReq` 中 `//target->policy_id = src->;` 被注释且**表达式残缺**；而目标结构 `atp::gw::AgwMsgCancelOrderSync` **确有** `policy_id` 字段 → 撤单报文到 DSE/AGW 时不带策略编号。本方案不改（通常由网关按原委托推导），已列入文档 §8 待确认项 6。

（注：`dse_client.cpp` 其余 `FillOrderReq` 等路径已正确赋值 `target->policy_id = src->external_order.trade_order_info.policy_id;`，FTE→DSE 委托段不用改。）

## 5. 测试方案（T1~T6）

- **T1** 编译 FTE 与 SHM 客户端（两棵 API 树）。
- **T2** 结构布局单测（最快最有价值，不需要起服务）：把"两条通道字节对齐"变成可执行断言，后续字段调整会立刻暴露。
- **T3** SHM 功能回归：`dse_stack/start_dse_stack.sh sz -trunk` + `run_shm_test.sh --market {sz,sh} --mode func -trunk`，沪深深各 **5/5 PASS**（登录/买入/静默/撤单/卖出），与 `svn_work/106` 基线一致；**先清 `/dev/shm`**。
- **T4** TCP API 功能回归：`run_test.sh --market {sz,sh} --type func --local`，用例全绿且**不得出现** `order req length ... too short`。
- **T5** `policy_id` 端到端透传（核心）：`policy_id` **不随回报回显**，交易所报文也不含该字段，且当前环境 DSE→AGW（`10.187.143.51:32001`）本就超时 → **唯一可靠观测点在 FTE 内部解析处**。以 **Debug** 级别启动 FTE（`start_dse_stack.sh` 的 `-l Info` 临时改 `-l Debug`），注入已知值（SHM `4321` / API `1234`），抓 `TradeOrderReq::dump()` 打印的 `policy_id[<值>]`（`uplink_biz_processor.cpp` `LOG_DEBUG`，`CheckBasicInfo → UtilityCheck`，两通道都会走到），`grep -o "policy_id\[[0-9]*\]"` 后比对。**负向判据（修复前基线）**：SHM 通道应等于 `ulStrategyKey & 0xFFFF`，不等于注入值；API 通道修复前为随机值。
- **T6** 性能回归（建议）：`run_shm_test.sh --market sz --mode perf --tps 1000 -trunk` 等；判据为吞吐不下降（基线 SHM 1000TPS 5000/5000 样本、P50 ≈ 0.5 ms 量级），结构体 +2 字节对 2048B 队列槽无实质影响。

## 6. 实施顺序与提交约束

| 序 | 步骤 | 依赖 | 判据 |
|:-:|---|---|---|
| 1 | 改 `share_comm_external_message.h`（+静态断言） | — | T1 |
| 2 | 改 `gw_head.h`（5 处） | — | 探针 116 |
| 3 | 改 `ute_client_handler_impl.cpp` × 8 处 | 步骤 2 | T1 |
| 4 | 改测试客户端与用例 | 步骤 1、3 | T3/T5 |
| 5 | 改文档 `api_net_time_design.md`（尺寸 106→116 + 字段表） | — | 尺寸描述一致 |
| 6 | 全量编译 + T2 + T3 + T4 + T5 | 1~5 | 全绿 |
| 7 | 三仓同步评估与执行 | 6 | 三仓 `0 0` |

**纪律（沿用既有约束）**：SVN 源码提交**由用户手动执行**，Agent 只整理清单与提交信息（`task/task_svn/task_svn.txt`）；提交前必须**多轮全量复查直至零修改轮**，工作日志写 `复查记录`；`share_comm_external_message.h`、`gw_head.h`、`ute_client_handler_impl.cpp` 属源码 → **三仓（gt_trunk/gt_svn/gt_sit）必须同改**。

## 7. 待用户/需求方拍板

1. `policy_id` **由谁填**？策略端显式填，还是 `StrategyMessageManager` 按策略 ID 自动填？（方案按"显式填"写，`policy_id` 是业务策略编号，与 `StrategyKey` 非同一语义）
2. `gw_head.h` 是否要求整包 **128 字节强一致**（补 `reserved` 到 116）？建议补。
3. **撤单是否也要带 `policy_id`**？需要则要补 `dse_client.cpp` 那处残缺注释。
4. 本期 **不回显** 回报（`TradeOrderER` 无该字段，如需回显属新增字段、影响 SHM 回报结构，需单独立项）。
5. FTE 侧不做 `policy_id` 校验（仅透传），以规避拒单风险；AGW API 可选加 2 字节范围检查。

## 8. 复查与交付状态

- 文档共 19 张表列数一致、`check_doc_refs.sh` 校验 `257` 处路径引用 / `29` 处章节号引用、豁免 18 处、**问题 0**；`file:line` 引用文件全部存在。
- 工作日志 `work_log/20261112.md` 记录**六轮复查**（前 5 轮均有修正，第 6 轮零修改通过），用单个合并探针复验全部尺寸/偏移数字。
- 提交 `b02da5345`（`docs(api): policy_id 贯通 TCP API / 共享内存 API 的修改方案`）已推送，trunk 与 gt_svn/gt_sit 均 `0 0`。
- **下一步待用户确认**是否按方案实施（改代码 + 全量编译 + 跑 T2~T5）。
