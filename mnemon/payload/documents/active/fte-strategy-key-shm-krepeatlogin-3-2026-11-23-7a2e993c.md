---
id: "7a2e993c-939c-4c17-b7b8-7aa6349d7866"
title: "FTE 策略死亡事件 strategy_key 丢失导致 SHM 登录态残留（重登录报 kRepeatLogin=3）定因、修复与验证（2026-11-23）；附「资金变动无回报」判定为功能覆盖不足"
description: "可复用排查结论：FTE on_event 回调形参个数不匹配（std::bind 只绑 _1.._3）致死亡检测传给 DoLogOut 的 strategy_key 实为 buffer_len(=0)，登出空转使 SHM 登录态残留、同账号再登录报 kRepeatLogin(3)；含证据链、4 处修复、修复前后实测对照，以及资金变动 3001 不推送的推送条件分析（非 bug，3 个真实缺口）。"
status: "active"
created_at: "2026-11-23T10:48:11.750Z"
updated_at: "2026-11-23T10:48:11.750Z"
content_hash: "b39a0af9e0b079df33402f453479df31807c45ba2d329d565981d41540f15ace"
source_paths:
  - "DYS-FRAMEWORK/fte/fte/src/main.cpp"
  - "DYS-FRAMEWORK/fte/include/business/strategy_trade_busi.h"
  - "DYS-FRAMEWORK/fte/src/business/strategy_trade_busi.cpp"
  - "DYS-FRAMEWORK/fte/src/business/uplink_biz_processor.cpp"
  - "DYS-FRAMEWORK/fte/unittest_google/work/business/ute_process.cpp"
  - "svn_work/04-功能与业务缺陷/01-缺陷排查/183_共享内存重登录失败与资金变动无回报分析_20261123.md"
  - "work_log/20261123.md"
session_ids:
  - "b64947e0-43c8-4186-beb2-1959f99ffc8f"
memory_body_ids:
  []
---

# 现象1：客户端异常退出后同账号重登录失败 —— 真实代码 bug（已修复并实测）

工作副本 `/home/lsz/code/work/gt_trunk`（SVN trunk / git 镜像 `gt_trunk`）。提交：`59b109640`（修复）+ `d96c9953d`（合并远程并归位文档）。

## 1. 故障签名（可直接用于判型）

```
15:01:58  LogOnAns session_status=1                    ← 登录成功
15:03:54  Lock file [34480723439] is dead, remove it    ← 死亡检测生效
15:03:54  DoLogOut SHM 0 enter                          ← ★ strategy_key 退化为 0
15:04:05  LogOnAns error_code=3                         ← kRepeatLogin
```

判据：**`DoLogOut SHM 0`**（登出键为 0）+ 日志中**完全没有** `SHM DoLogOut fund_account_id ...` ⇒ `strategy_custMap_` 里 `it->second == 0` 匹配不到任何账户，整个清理循环空转 ⇒ SHM 登录态残留 ⇒ 再登录得到 `error_code=3`（`CheckError::kRepeatLogin = 3`，定义于错误码头 `static const CheckError_def kRepeatLogin = 3;`）。

## 2. 根因：回调形参个数不匹配，`std::bind` 少给占位符**不报编译错**

```cpp
// 回调类型 UteGetStrategyReqCallBackFuncType —— 5 参
void(int iMsgID, const char* buf, unsigned int buffer_len,
     unsigned long long strategyKey, unsigned long long popTime);

// 修复前：StrategyTradeBusi::on_event(int, const char*, size_t strategy_key) —— 只有 3 参
// 修复前：main.cpp  std::bind(..., _1, _2, _3)
```

`lock_file_manager` 的调用是：

```cpp
UteOnEventFunc_(kUteFailed, "<key> is Dead!", /*buffer_len=*/0, /*strategyKey=*/stoull(key), 0)
```

`buffer_len` 恒为 0，因此 3 参版 `on_event` 收到的 `strategy_key` 其实是**第 3 参 buffer_len(=0)**，真正的 key（第 4 参）被静默丢弃。

**为什么长期未暴露**：`shm_test_client.cpp:539` 早有绕行注释「不登出直接退出会留下已登录状态，下次登录得到 kRepeatLogin(3)，所以每次跑完都要主动登出」——测试脚本每次都主动登出，把 bug 藏住了；只有异常退出/直接关窗口才命中。

## 3. 修复（4 处，最小闭环）

| 文件 | 改动 |
|---|---|
| `DYS-FRAMEWORK/fte/include/business/strategy_trade_busi.h`、`src/business/strategy_trade_busi.cpp` | `on_event` 改 **5 参**，显式取第 4 参 `strategyKey` 作登出键（注释写明「不能取第 3 参 buffer_len」） |
| `DYS-FRAMEWORK/fte/fte/src/main.cpp` | `std::bind` 占位符 `_1.._3` → **`_1.._5`** |
| `DYS-FRAMEWORK/fte/src/business/uplink_biz_processor.cpp` | **附带修复**：`strategy_custMap_.emplace(...)` → 赋值语义；`emplace` 在 key 已存在时**不覆盖**，同账户再登录会残留旧 `strategy_key` ⇒ 旧会话死进程事件匹配不到、登出清不掉 ⇒ 同类残留复发 |
| `DYS-FRAMEWORK/fte/unittest_google/work/business/ute_process.cpp` | 占位符同步（测试侧） |

## 4. 修复前后实测对照（构造「客户端异常退出后重新登录」）

| 版本 | FTE 日志 | 同账号再登录 |
|---|---|---|
| 修复前 | `DoLogOut SHM **0** enter`（无 fund_account_id 行） | ❌ `error_code=3` |
| 修复后 | `DoLogOut SHM **34493531130** enter` → `SHM DoLogOut fund_account_id 1000000000000001, branch_id 0001` | ✅ `session_status=1, error_code=0` |

构建：容器内重建 **0 error**（utedatainit / ute / agwapi / mock_client 全绿）；环境收尾 `hugepages=0`、残留 shm=0、无存活残留进程；三仓 `0 0 改动=0`。

## 5. 全仓复核要点

改签名后需确认没有其它绑定点遗漏：全仓 `on_event` 定义/声明中，`strategy_trade_busi.h` 是唯一与 `UteGetStrategyReqCallBackFuncType` 对应的那个；`tcpserver_handler.h`、`ccu_trade_impl.h`、`ccu_base.h`、`tech/network/tcp_endpoint.h` 的同名函数是 tcp 事件重载，不相关。

# 现象2：资金变动没有回报 —— **不是 bug**，但背后有 3 个真实缺口

推送条件（`ins_handler.cpp`）要求四者同时满足，第 4 条是 `vertical_table_items[kT1]`（T+1 可用资金）**真的发生变化**。当日数据：

| 观测 | 结果 |
|---|---|
| 资金指令 `PktRTControlInstructionFundInfo` | 16 条，全部 `action_=Q`（查询）、`t1_=0` |
| 资金调整 `PktExtAdjustSyncIns` | 4 条，`dys_t0/t1/t2/t3/tn_effect` 全 0，只 `stk_effect=1`（改股份） |
| 日志 | `No changed for available_T1` × 16；`PktFundChangeNotify` 下发 **0** |

查询指令走 `FillIns(res)` 只读填响应、不改 `fund_data`；调整指令只动股份 ⇒ **本来就不该触发 T+1 推送，行为正确**。反面验证（真实 T+1 变化触发 3001）需 DSE 下发 `dys_t1_effect=1` 的指令，本环境未构造，已在文档中标注。

真实缺口（建议排期）：

1. **结构只表达 T+1**：`FundChangeNotify` 只有 `available_t1`，无法表达 T0/T2/T3/TN、信用可用、冻结；
2. **触发点只覆盖 DSE 下行指令**：成交回报引起的资金变动（手续费扣减等）不推送；
3. **多数消息 `fund_data_scope` 恒为 `Initial`**（日志统计 `scope[0]=5805 / scope[1]=24 / scope[2]=2`）→ 推送分支直接跳过，容易被误判成「修复没生效」。

# 可复用教训

- **回调签名与 `std::bind` 占位符数量不一致是静默 bug 源**：形参少一个就整体错位（后一个参数顶上来），编译期零告警。改回调签名后必须同时改 bind 占位符，并用「日志打印键值」实测验证传递是否正确，而不是只看能编过。
- 容器/`Map` 用 **`emplace` 表达「新建」、用赋值表达「覆盖」**；登录态这类「同 key 应刷新为本轮值」的映射一旦用 `emplace` 就会残留旧值，并使已修的 bug 复发。
- 排查「登出/清理不生效」先看**传入的键值本身**（`DoLogOut SHM 0`）而不是清理逻辑；键为 0 通常意味着上层参数错位，而非容器没数据。
- 现象定级要分清 **bug（现象1）** 与 **功能覆盖不足（现象2）**；后者不要改代码「让推送出现」，先对齐推送条件的业务语义。

# 相关文档与路径

- 现场文档（落库 183 号）：`svn_work/04-功能与业务缺陷/01-缺陷排查/183_共享内存重登录失败与资金变动无回报分析_20261123.md`（7 章 233 行）
- 过程日志：`work_log/20261123.md`（6 轮复查，末轮零修改）
- 相关 Managed Document：`859ea118-22d3-4a54-bd0e-944131326bbc`（on_message 只覆盖 NonReqOrder 队列，委托走 TryPopReqOrder）、`8c2e87c5-a86e-4234-8edc-0d0c551f6b64`（policy_id 布局 108→106 与通道下限 108）

## 编号与登记（工程约定，避免重号）

`svn_work` 已按 6 大类目录重组（`01-环境与部署` … `06-总结与过程记录`，根目录只允许 README.md），并要求在 `svn_work/00_文档导引_按序号总览.md` 登记；`task/review/check_doc_guide.sh`（导引数==实际文档数、每篇必须登记）与 `check_doc_placement.sh`（必须归类到 6 大类子目录）已挂 pre-commit。本次原拟 135 号，但 `135_复测全过程总结_20261117.md` 已占用、编号可重复（同号不同篇），最终取空闲号 **183** 并归入 `04-功能与业务缺陷/01-缺陷排查/`。合并时另发现远程 177/178/179 在 `Prompt.md` 中的引用缺子目录前缀（3 处失效路径），已修正，`check_doc_refs.sh` 问题数 3 → 0。
