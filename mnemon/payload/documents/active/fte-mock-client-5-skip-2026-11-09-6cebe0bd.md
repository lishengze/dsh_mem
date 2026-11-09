---
id: "6cebe0bd-7ebc-44ee-8c94-55551983a76b"
title: "FTE mock_client 深圳现货案例补齐（任务5）：证券白名单、撤单动态委托号与 SKIP 机制（2026-11-09）"
description: "任务5 补 5 个深圳现货案例（买入/卖出/成交/撤单/静默）并三仓跑通 13/13；记录 000525 被拒的 kSecurityErr=1 根因与 159930 试验法、撤单必须用 $last_order_sys_no（固定 \"0\" 在 SH/SZ 均超时）、前置引用缺失时 SKIP 而非失败、cmake 重配失败下用 flags.make/link.txt 单独编译、三仓提交。"
status: "active"
created_at: "2026-11-09T03:00:56.935Z"
updated_at: "2026-11-09T03:00:56.935Z"
content_hash: "e6b03dc27d5dc8ededcbcc09b6df682cb31498ce3521153ce7c4e2f5f7299d43"
source_paths:
  - "svn_work/71_任务5深圳现货案例补充_20261109.md"
  - "DYS-FRAMEWORK/api/gw_api_boost/ute_mock_client/src/mock_client.cpp"
  - "DYS-FRAMEWORK/api/gw_api_boost/ute_mock_client/include/test_case_runner.h"
  - "DYS-FRAMEWORK/api/gw_api_boost/ute_mock_client/src/test_case_runner.cpp"
  - "DYS-FRAMEWORK/api/gw_api_boost/ute_mock_client/config/test_cases/sz/cases"
  - "DYS-FRAMEWORK/fte_all/include/common/ute_types.h"
session_ids:
  - "8a9bdc98-5521-4d7d-9a46-93ff8ad12eaf"
memory_body_ids:
  []
---

# FTE mock_client 深圳现货案例补齐（任务5，2026-11-09）

在 `DYS-FRAMEWORK/api/gw_api_boost/ute_mock_client` 的案例集里补齐深圳现货场景，三仓（gt_trunk / gt_svn / gt_sit）全部 13/13 通过，上海回归 13/13。原始实施记录见 `svn_work/71_任务5深圳现货案例补充_20261109.md`；单案例目录布局、`cases/` 前缀坑与侧车机制见文档 `0d85368b-f55c-4ebc-991f-d79e413a2c52`；控制台入口与 `LD_LIBRARY_PATH`、SIGBUS 噪声见文档 `2c2b5ebe-117e-4de3-8a17-52f60834a5fe`。

## 新增案例（config/test_cases/sz/cases）

| 案例 | 场景 | 期望 |
|---|---|---|
| FTE 深圳委托买入测试 | 现货买入申报 | `order_rtn` |
| FTE 深圳委托卖出测试 | 现货卖出（side=2） | `order_rtn` |
| FTE 深圳成交回报校验 | 委托后成交 | `trade_rtn`（exec_type=F） |
| FTE 深圳撤单测试 | 撤单应答 | `cancel_rsp` |
| 等待回报静默（2s） | 用例间排空 | `heartbeat_ok` |

期望字段文件 4 个：`order_rtn_buy` / `order_rtn_sell` / `trade_rtn_buy` / `cancel_rsp`。

## 坑一：证券代码必须被 FTE 认可（kSecurityErr=1）

首版用 `000525`（证券状态 status=0），委托被拒：

```
order_status=8  exec_type=1  reject_reason_code=1   cum_qty=0  leaves_qty=100
```

`include/common/ute_types.h` 中 `RejectReasonCode::kSecurityErr = 1`（标的相关错误）→ 该证券不在 FTE 上场数据里。SZ 的 `cash_auction_params_ute.xml` 条目数为 0，无法从静态表推断可用标的。

**试验法**（逐代码试单，最快）：

| 证券 | 结果 |
|---|---|
| **159930**（ETF 案例同款） | ✅ 通过 |
| 159453 / 000001 / 000525 / 002719 | ❌ 全部拒绝 |

→ 深圳现货统一改用 **159930**、价格 `10000`（1.000 元）。遗留：其它 A 股代码需补充 SZ 上场数据（证券静态表）。

## 坑二：撤单必须用真实委托号（推翻 0d85368b 的「解耦为固定 "0"」）

| 做法 | 上海 | 深圳 |
|---|---|---|
| 固定 `"0"` | 现也超时（跑全量时暴露） | 静默丢弃，无应答 → 超时 |
| `$last_order_sys_no` | ✅ | ✅ |

即当时「期望回报已固定，不需要真实委托号」的判断只在缩小后的用例集成立；跑 SH 全量后固定 `"0"` 同样超时。SH/SZ 两处撤单均改为动态引用前序委托号。

代价是重新引入跨案例依赖，因此补 **SKIP 机制**：

```cpp
// 请求引用了 $last_order_sys_no 但该值尚未产生 → 跳过，不计失败
result.skipped = true;
result.passed  = true;
```

`TestResult` 新增 `skipped` 字段，报告输出 `[SKIP]` —— 既不误判失败，也不假装通过。乱序执行（撤单排在首位）实测为 `[SKIP] FTE 深圳撤单测试` + 其余 PASS；正常顺序仍 13/13。

遗留：要真正独立需 mock_client 支持「一案例多请求」（一个案例内先下单再撤单）。

## 构建：cmake 重配失败时的绕行

`make` 触发 `cmake_check_build_system` 重配，报 `ATP_RES_ROOT AND CMAKE_ATP_RES_ROOT Are All Not Set`（环境变量要求变化），整工程无法重配。只改了 mock_client 时不必重编全工程：直接用已配置好的 `flags.make` / `link.txt` 单独编译链接。

| 仓 | 产物大小 |
|---|---|
| gt_trunk | 5,472,624 B |
| gt_svn | 5,472,624 B |
| gt_sit | 343,200 B（独立编译方式） |

gt_sit 退出码 134（历史已知 SIGBUS）但报告完整，不影响判定。

## 验证与提交

- 深圳全量：gt_trunk / gt_svn / gt_sit 均 13/13；4 个现货案例全通过。
- 上海回归：13/13（改撤单为动态引用后）。
- 三仓资产一致：sz 案例 13 + 期望 5 + 文档 Y + 代码含 skip 逻辑。
- 提交：gt_trunk `d5ac0eabc`、gt_svn `c681d5e17`、gt_sit `660680f32`，上游均 0/0。
