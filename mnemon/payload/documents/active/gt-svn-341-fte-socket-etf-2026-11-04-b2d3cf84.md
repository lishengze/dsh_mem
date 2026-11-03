---
id: "b2d3cf84-f7c9-4ab9-af5b-36af0dd745ea"
title: "gt_svn 341 硬上限根因（FTE 停读交易所 socket）与 ETF 结论修正（2026-11-04）"
description: "修正此前“GT_SVN 无 ETF 分支”的误判：ETF 分支存在于函数前置 if（msgid 1007 vs 工具链 1010）；性能样本恒 341 的根因是 FTE 压测中停止读取交易所 socket（Recv-Q=1100）；并记录 flush_on(info)、Debug 日志、39142 端口、对象池四条排除项与两代 API 并存导致的 ETF 阻塞。"
status: "active"
created_at: "2026-11-03T23:15:50.558Z"
updated_at: "2026-11-03T23:15:50.558Z"
content_hash: "25e2d7e5c9f1a651cdaf3c9d1d7f75013fd53025e9ae290e15f549943a1a9439"
source_paths:
  - "svn_work/39_gt_svn深度排查_341机制与ETF报文号_20261104.md"
  - "svn_work/40_gt_svn_ETF复测结论与阻塞_20261104.md"
  - "study/knowledge_base_system/14_线上代码复测与问题定位.md"
  - "api/gw_api_boost/ute_mock_client/api_net_time_common.h"
session_ids:
  - "cf3808e4-20bf-4756-b7ce-73eea72ff35f"
memory_body_ids:
  []
---

# gt_svn 341 机制定位 + ETF 结论修正（2026-11-04）

**本文修正**文档 `9e861d20-7da8-4980-a9c5-34e5fcbae404` 中的两条结论，并补上性能"341 硬上限"的根因。
前一阶段基线见 `ac5aee7f-793f-41d9-a598-0e8950cccb73`（登录/委托链路）与
`9e861d20`（本地复测最终结论）。源码副本 `/mnt/work/gt_svn`，运行根 `/mnt/work/gt_test_svn/run`，对照环境 `gt_trunk`。
汇报文档：`svn_work/39_gt_svn深度排查_341机制与ETF报文号_20261104.md`、`svn_work/40_gt_svn_ETF复测结论与阻塞_20261104.md`；
提交 `7f93e6235`、`91d222d30`、`2b2191611`。知识库补充：`study/knowledge_base_system/14_线上代码复测与问题定位.md` §11。

## 1. 修正一：gt_svn 的 FTE **有** ETF 分支

旧结论"ccu_trade_impl 没有 ETF 报文分支"**错误**。真实代码（`DYS-FRAMEWORK/fte/src/business/ccu_trade_impl.cpp:44` 附近）：

```cpp
if (LIKELY(message::kPktETFReq == msg_id)) {
    FTE_SAVE_POINT(TimePoint::T1);
    uplink_biz_processor::Instance()->DealTradeETFOrderReq(endpoint, data, length);
    ...
    return;
}
```

ETF 分支写在**函数开头的前置 `if`**，不在 `switch` 里；当时只 `grep 'case.*kPkt'` 因此漏判。
教训：判断"某功能是否存在"必须读完整函数体。

## 2. 修正二：差异是消息号，不是能力

| 位置 | `kPktETFReq` |
|---|---|
| `gt_svn/DYS-FRAMEWORK/fte/include/message/gw_external_message.h:52` | **1007** |
| `gt_trunk/DYS-FRAMEWORK/fte/include/message/gw_external_message.h:54` | **1010** |
| 本地 `api/gw_api_boost/ute_mock_client/api_net_time_common.h:15`（从 gt_trunk 拷入） | **1010**（与 FTE 冲突） |

客户端发 1010 → FTE 报 `unexpect msgid 1010` → 14 个 ETF 用例全挂。
改回 1007 并重编后，`unexpect msgid` 新增计数归零（报文已被接受），但**仍无回报**——见第 4 节。
远程"深市 8/8"因此是**正确的**：远程用 gt_svn 自带（发 1007）的客户端；本地错在混用了 gt_trunk 的工具链。

## 3. 341 的机制：FTE 停止读取交易所 socket

决定性证据（压测结束后查连接）：

```
ESTAB 1100 0 127.0.0.1:42734 127.0.0.1:39143 users:(("ute",pid=86330,fd=15))
      ^Recv-Q=1100  ← 交易所已发出数据，FTE 未读取
```

因果链：FTE 不再读交易所连接 → 回报无法处理、订单对象不释放 → 不再转发新委托（出口包卡在 341）
→ 客户端发单在 TCP 缓冲积压 → 窗口结束客户端断开 → 积压委托全丢 → 抓包只见 341 笔走完流程。

因此 341 与 TPS、时长、对象池容量、风控均无关：是"FTE 卡住"，不是"资源不够"。
判决方法：`ss`/`netstat` 看收发两端 socket 的 `Recv-Q`，比翻几万行日志快。

### 已实测排除（补充 9e861d20 的四条）

| # | 假设 | 结果 |
|---|---|---|
| 5 | 日志 flush 过频（`flush_on(info)`，且用同步 `rotating_logger_mt`，对照 gt_trunk 是异步 `create_async` + 仅 err flush） | 改 `flush_on(err)` 后 341 **不变** |
| 6 | Debug 级日志拖慢 | Debug 下**连登录都 `kLocationsError`/超时**（`mock_client.cpp:182/187`），性能测试必须 Info |
| 7 | 订单走 39142 而非 39143 | `--exch-port 39142` 抓包 **0 包**，只走 39143 |
| 8 | 对象池 | 放大到 262144（`fte_report`/`fte_reject`）后发送失败 14658→0，341 不变 |

延迟口径仍可信：1000TPS/15s P50 9.97 µs、200/25s 9.09 µs、50/40s 9.65 µs，outlier 0（样本恒 341）；
远程基线 10.70 µs，差约 7%。样本少是 FTE 卡住所致，非测量问题。

## 4. ETF 复测：报文号对齐后暴露"两代 API"问题

- 客户端（mock_client）重编后 FTE 侧 **零条 ETF 报文**，只有心跳（`msg_id 3`）；
- 原因：gt_svn 内并存两代 API——
  - `DYS-FRAMEWORK/api/gw_api_boost/api`（**新代**，mock_client 链接的）：ETF 实现 **0 处**，走 `client_->SendMessage(type,msg)`；
  - `DYS-FRAMEWORK/fte/gw_ute_api/gw_api_boost/api`（**旧代**）：`ute_client_handler_impl.cpp:1314` 有 `package_new(..., kPktETFReq, ...)`。
- 所以改 `api_net_time_common.h` 的常量只影响客户端**调试打印**，真正决定"发不发"的是链接的 API 库；
- gt_svn 仓库**本身不含客户端**，`ute_mock_client` 是从 gt_trunk 拷入的。

本地最终可达：登录/心跳 ✅；深市功能 2/8（6 个 ETF 无法执行）；性能延迟可信；现货 342 笔后卡住（P0-5）。

### 打通 ETF 的三条路径（按性价比排序）

1. **向远程索要其客户端**（低，推荐）：远程已 8/8，拿到即可本地复现；
2. 用旧代 API 写最小 ETF 客户端（中，几百行）；
3. 给新代 API 补 ETF 分支（中高，改的是线上代码，需客户确认）。

## 5. 可复用教训

1. `grep 'case.*'` 会漏掉前置 `if` 分支——判断功能有无要读完整函数体；
2. 移植工具链会带旧语义常量（1007/1010 冲突即典型）；
3. "跑的是谁的二进制"要贯彻到最底层：源码、FTE、客户端、工具头文件四者同源；
4. socket 的 `Recv-Q` 是判断"谁卡住"最快证据；
5. 改常量 ≠ 改行为——要验证"报文是否真的发出"，看对端日志比看客户端超时提示有用；
6. 同一仓库可能并存两代 API，能力不一致，先确认组件齐备（含"仓库里没有客户端"）再开测；
7. Debug 级日志不可用于性能测试。

## 6. 遗留

- **P0-5**：压测中 FTE 停止读取交易所连接（Recv-Q 积压后完全停滞）——待 gdb attach 卡住后的 ute，看交易所读线程停在哪；
- **P0-4**：对象池容量过小（已确认存在，放大后发送失败归零）。
