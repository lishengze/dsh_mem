---
id: "c53c5b3f-0494-4597-8cf6-068fcefd8c04"
title: "共享内存测试客户端支持 FundChangeNotify（资金变化通知 3001）解析：结构同构核验、正向/负向实测与模拟推送（2026-11-19）"
description: "gt_trunk test/share_test 套件新增 kPktFundChangeNotify(3001) 客户端解析分支的可复用知识：内部 PktFundChangeNotify 与通道 FundChangeNotify 的 56 字节同构对照、iMsgLen 长度校验依据、ute_server 主动推送验证机制、json_struct FundChangeNotifyStr、顺带修复登录应答长度 bug、CRLF 与 eol=lf 的 git 提示解释；含提交 3c80bf4e9。"
status: "active"
created_at: "2026-11-19T01:36:18.395Z"
updated_at: "2026-11-19T01:36:18.395Z"
content_hash: "b4fe541fa59e013ba5f5f64e0d9cbac189509831e74ebd6ee05c0a13d5b0136c"
source_paths:
  - "test/share_test/pack_for_src/share_comm_external_message.h"
  - "test/share_test/pack_for_src_upgrade/share_comm_external_message.h"
  - "test/share_test/share_util/json_struct.h"
  - "test/share_test/test_client/strategy_client.cpp"
  - "test/share_test/test_server/ute_server.cpp"
  - "DYS-FRAMEWORK/fte/include/business/uplink_biz_processor.h"
  - "DYS-FRAMEWORK/fte/include/common/external_types.h"
session_ids:
  - "5be78af5-8a5a-49a7-b0d5-e17ccfb85e2b"
memory_body_ids:
  []
---

# test/share_test 支持 `kPktFundChangeNotify`（资金变化通知 3001）

**一句话**：FTE 用 `kPktFundChangeNotify`(3001) 向策略端**主动推送**资金变化，测试客户端原先无此分支；本次补齐解析，并用字节同构核验 + 正向/负向实测自证，顺带修掉模拟服务端一处长度 bug。

产出：代码提交 `3c80bf4e9`（已推送，三仓 `0 0`）；文档 `svn_work/132_共享内存测试客户端升级_20261117.md` 新增 §6.11~6.14；日志 `work_log/20261119.md`（6 轮复查记录，末轮零修改通过）。
前序背景见 [共享内存测试客户端 test/share_test 升级（policy_id + 方案Q）](98683cb9-f567-47ff-b952-2ebc8a769a36)。

## 1. 两个最容易写错的语义

| 特点 | 影响 |
|---|---|
| **结构里没有 `TradeOrderUser` 子结构**，账户字段平铺 | 不能套用 `ParseTraderOrderUser` / `TraderOrderUserStr`；照抄其他回报的写法会读错 |
| **不是请求-应答，而是柜台主动推送** | 与“先发请求再等应答”的时序无关，随时可能到达；本地常规流程不会触发，需要专项造包才能验证 |

## 2. 关键核验：FTE 下发的是内部结构，不是通道结构

`DYS-FRAMEWORK/fte/include/business/uplink_biz_processor.h` 中 `SendTradeOrderER(const FundChangeNotifyContext&, msg_type)` 组装的是 **内部 `message::PktFundChangeNotify`**，发送长度取 `sizeof(PktFundChangeNotify)`，并写入 `++report_index_`：

```cpp
shm_rsp_queue_manager->SendMsg(msg_type, (char*)&fund_change_notify, sizeof(PktFundChangeNotify));
```

因此客户端判断长度前必须先证明内部结构与通道 `share_common::FundChangeNotify` **逐字节同构**。类型来源 `DYS-FRAMEWORK/fte/include/common/external_types.h`：

| 字段 | 类型定义 | 内部 offset | 通道 offset |
|---|---|--:|--:|
| `fund_account_id` | `std::array<char,16>` | 0 | 0 |
| `branch_id` | `std::array<char,10>` | 16 | 16 |
| `account_id` | `std::array<char,12>` | 26 | 26 |
| `market_id` | `uint16_t` | 38 | 38 |
| `available_t1` | `int64_t`（`Amt_def`） | 40 | 40 |
| `index` | `int64_t`（`SeqNum_def`） | 48 | 48 |
| **sizeof** | | **56** | **56** |

结论：客户端 `iMsgLen < sizeof(FundChangeNotify)` 的长度校验**有实测依据**，不是猜测。凡是“FTE 内部结构 vs 通道结构”的新消息，都应先做这一张 `offsetof` 对照表，再写长度判断。

## 3. 改动清单

| 文件 | 内容 |
|---|---|
| `test/share_test/pack_for_src{,_upgrade}/share_comm_external_message.h` | 重新执行 `sync_from_fte.sh` 同步；同步后与 FTE 源 **md5 逐字节一致**（`ce8e94d4ad6805044d21698ed75f4d7f`） |
| `test/share_test/share_util/json_struct.h` | 新增 `FundChangeNotifyStr`，输出全部 6 个字段 |
| `test/share_test/test_client/strategy_client.cpp` | `StrategyOnMessage` 新增 `case kPktFundChangeNotify`：长度不足先 `LOG` 告警再跳过（防越界读），否则解析并打印 |
| `test/share_test/test_server/ute_server.cpp` | 委托应答后**主动推一条**通知，便于本地验证；顺带修 bug |

**顺带的 bug**：登录应答用了 `sizeof(LogOutAns)`，改为 `sizeof(LogOnAns)`（复制粘贴漏改）。注意 `g_fund_change_index_` 这类全局变量必须定义在使用点**之前**（首次编译即因定义在文件尾部而报 `not declared`）。

## 4. 验证方法（可复用）

- **正向**：模拟服务端在 `kPktOrderReq` 应答后跟发一条 `kPktFundChangeNotify`（`available_t1=9876543210` 故意用非零值，便于确认真解析到）。委托 4 次收到 4 条，6 字段全部正确，`index` 依次 1/2/3/4。
- **负向**：故意发 `sizeof(FundChangeNotify) - 8`（48 字节），客户端打印 `length mismatch, iMsgLen=48 (expect 56)` 后跳过，不崩溃、不越界。
- **回归**：`make clean` 全量重编 0 error（5 个 target）；ABI 契约仍 `User=70 / Info=38 / policy_id@36 / Req=124(key@0 user@8 info@78) / body=116`；`OrdERInfo` 每条 33 字段；`policy_id` 回显 `4321/0/1/65535`。
- **数条数的教训**：撤单**不推送**资金变化，所以“委托 4 次 + 撤单 1 次”只会收到 4 条。核对条数要以下发端日志（`Send FundChangeNotify`）为准，别按客户端输入命令次数数。

## 5. CRLF 与 `eol=lf` 的 git 提示

同步后快照文件在工作区是 **CRLF**（与 FTE 源逐字节一致），而仓库属性为 `text: auto / eol: lf`，故 `git diff`/提交会提示 `CRLF will be replaced by LF`。这是**预期行为**：工作区保留原样，入库为 LF。复查 diff 时应用 `diff -w`/忽略行尾，不要为消除提示而改文件行尾。

## 6. 复查要点

- `StrategyOnMessage` 覆盖的消息 ID：`kPktLoginAns / kPktLogoutAns / kPktOrderAns / kPktCancelOrderAns / kPktOrderMatch / kPktRejectMsg / kPktFundChangeNotify`。
- `FundChangeNotifyStr` 输出字段必须与结构字段**一一对应**（自检：集合比较无缺无多）。
- 快照 diff 只应含新增两块（两文件各 11 行），确认无其它漂移。
