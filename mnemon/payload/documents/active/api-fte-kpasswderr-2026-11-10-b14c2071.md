---
id: "b14c2071-d3d5-458d-80fb-1733405e5af3"
title: "网关 API 用例/配置合规改造与 FTE 登录 kPasswdErr 复现（2026-11-10）"
description: "网关配置与用例对齐 FTE 的改造项（market_type=101、need_multi_cust、order_qty=10000 口径）、用例 JSON 顶层 _待确认 约定与校验器 _ 前缀跳过规则、测试环境端口与 start_all.sh 日志级别约束、实测错误码判读（kPasswdErr=4 / -35 NOT_LOG_CUST 连锁）、登录失败的 4 项已排除项与剩余疑点（LogOnReq 报文偏移/填充）。"
status: "active"
created_at: "2026-11-10T11:25:59.436Z"
updated_at: "2026-11-10T11:25:59.436Z"
content_hash: "ee62b6ea56475d26df5531bbeff2fbedcd56a5d207f123f20ec0aee22e54fb49"
source_paths:
  - "task/api_dev/todo_review_20261110.md"
  - "work_log/20261110.md"
  - "trunk/NewAPI/gone/api/mock/client/src/test_case_runner.cpp"
  - "trunk/NewAPI/gone/api/mock/client/config/connection_config_gw_single.json"
  - "trunk/NewAPI/gone/api/src/gw_counter_direct.cpp"
  - "DYS-FRAMEWORK/fte/ftedatainit/src/ftedata_init.cpp"
  - "DYS-FRAMEWORK/fte/src/business/uplink_biz_processor.cpp"
session_ids:
  - "ac88ec76-9092-47cb-9799-197e725db419"
memory_body_ids:
  []
---

# 网关 API 用例/配置合规改造与 FTE 登录 kPasswdErr 复现（2026-11-10）

一轮"配置+用例对齐 FTE"的改造 + 实机复测记录。**配置侧改造已验证生效，但登录仍被 FTE 判 kPasswdErr，根因未定**（本节第 6 节为已排除项，属开放问题，不要当成结论引用）。

相关文档：
- `ac5aee7f-793f-41d9-a598-0e8950cccb73`（gt_svn 登录/委托排障基线：CheckError 错误码表、`CheckPasswd()` 查 `passwd_map_`、测试账号三件套教训、必须 `-l Debug`）——本次是该基线结论的**反向案例**：账号/密码/数据侧三件套全部核对无误，仍报 `kPasswdErr`，指向报文层。
- `0d85368b-f55c-4ebc-991f-d79e413a2c52`（mock_client 框架 / pad 填充差异导致登录失败与 50019）。
- `6cebe0bd-7ebc-44ee-8c94-55551983a76b`（案例补齐、`$last_order_sys_no`、SKIP 机制）。

## 1. 配置与用例改造清单（已验证生效）

| 对象 | 改动 | 依据 |
|---|---|---|
| `connection_config_gw_single.json` / `_multi.json` | `market_type` 1 → **101** | 101=上海（用例证券 600007）；FTE 应答已回填 101，说明取值被接受 |
| 同上 | `single_cust_per_link` → **`need_multi_cust`**（0/1） | 配置字段更名 |
| `cases/order_request.json` | market→101；`order_qty` 100 → **10000**；移除已下线字段 `policy_id` / `tgw_id` | 10000 = 100 股 × 100（RUL-35），且须为 FTE `buy_qty_unit` 的整数倍 |
| `cases/{order,trade,cancel,login}_expected.json`、`perf_order_request.json` | market→101；`qty`/`trade_qty`/`exec_qty` → 10000 | 与请求侧口径一致 |
| `test_plan_gw.json` | `perf_test.enable` → **false** | 功能先跑通，性能单独开 |
| `test_case_runner.cpp` | 校验器新增：**字段名以 `_` 开头视为注释/待确认，跳过校验** | 见第 2 节 |

## 2. 用例 JSON 的"待确认"约定（教训）

- 待确认标记统一放在用例 JSON **顶层 `_待确认` 对象**里，**不要放进 `fields`**；校验器同时把 `_` 前缀字段当注释跳过（双保险）。
- 踩坑：首版把标记放进 `fields`，校验时被当成真实期望字段 → 报"字段未找到"而判失败。凡"给人看的说明文字"都不能落在 `fields` 里。

```json
"fields": { "market_type": 101, "order_qty": 10000 },
"_待确认": { "order_qty": "10000 = 100 股 ×100（RUL-35），须为 FTE buy_qty_unit 整数倍" }
```

## 3. 测试环境拓扑与启动约束

| 组件 | 端口 |
|---|---|
| FTE（ute） | 33001 上海 / 33002 深圳 |
| mock_exch_server | 38140 沪 / 39142 深 / 39144 SSESTEP |
| 98 mock | 9003 |

- 必须用 `start_all.sh` 起环境：手动后台启动 `ute` **不常驻**（起来又退出，33001 不监听）；`start_all.sh` 可一次性恢复。
- `start_all.sh` 中日志级别**写死 `-l Info`**；要抓 `DealLogon` 全字段 dump 必须临时改成 `-l Debug`。Info 级只能看到客户端"登录失败"，看不到 `error_code`。

## 4. 实测结果与错误码判读

| 用例 | 结果 | 判读 |
|---|---|---|
| 登录 | ❌ | `on_login: err_code=4` = `CheckError::kPasswdErr` |
| 委托买入 | ❌ | `order_insert()` = **-35**（`NOT_LOG_CUST`）——**登录失败的连锁，不是独立问题** |
| 成交回报 | ❌ | 未收到 2005（委托根本没发出） |
| 撤单 | ❌ | `order_cancel()` = -35（同上连锁） |
| 心跳 30s（≈1000 次） | ✅ | 无断链 |

排障要点：登录失败后，委托/撤单/回报三类失败**全部是下游连锁**，先修登录再看它们，不要并行排查。

## 5. 登录链路的定长填充事实（本次核对，两侧一致）

- API 侧 `to_fte_str()`：**空格**填充定长字段（`array<char,N>`），`from_fte_str()` 裁尾空格。
- FTE 侧 `LoadLoginInfo()` 用 `CopyToArray()` 装载 fund/branch，同为**空格**填充；实测日志 `branch_id=[0001      ]`。
- `BranchID_def` 两侧均为 `std::array<char, 10>`；`password` 两侧比较前均 `AsStringTrim`（`uplink_biz_processor.cpp` `DealLogon`）。
- 密码链路完整：mock → `LoginReq.password` → counter98 `acc_login_event_info` → `gw build_login_msg`（截 99B、尾部保留空格）→ FTE `AsStringTrim` 后比对 `ConfigManager::passwd_map_`。

## 6. kPasswdErr 的 4 项已排除 + 剩余疑点（开放）

已排除：

1. **数据侧未加载**？—— `etf_test_sh/run/log/fteinit.txt` 有 `LoadLoginInfo insert: fund_account_id=[1000000000000001] branch_id=[0001      ] password=[TEST000000001`（与本次启动时间一致）。
2. **填充不一致**？—— 两侧均空格填充（第 5 节）。
3. **密码未透传**？—— 链路各环节均在。
4. **密码取值不符**？—— 用例 `TEST000000001` 与 XML `<Password>` 完全一致。

剩余疑点：`LogOnReq` 报文中 `password` / `branch_id` 的**实际落点**。若 API 组包与 FTE 解码的字段偏移不一致，FTE 读到的就不是这个密码（同类历史坑：pbu 字符集/填充差异）。

下一步（二选一，均不改上场数据）：
1. 改 `start_all.sh` 为 `-l Debug`，抓 `DealLogon message` 的 Request/Answer dump；
2. `tcpdump` 抓 33001 登录报文，按 `gw_head.h` 的 encode 顺序逐字段解码偏移。

## 7. 其他经验

- 复查价值：本轮 R1 抓到"待确认标记位置"与"qty 口径"两处，R2 抓到"章节落点错误"（同类第 4 次）→ 已把章节顺序检查固化为 `task/tool_check_chapter_order.py`。文档改动后跑一次脚本，比人眼复查可靠。
