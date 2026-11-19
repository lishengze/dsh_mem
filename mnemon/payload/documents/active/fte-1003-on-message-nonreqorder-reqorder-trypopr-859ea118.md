---
id: "859ea118-22d3-4a54-bd0e-944131326bbc"
title: "FTE 共享内存委托(1003)静默丢失排查结论：on_message 日志只覆盖 NonReqOrder 队列，ReqOrder 队列走 TryPopReqOrder（2026-11-19）"
description: "线上 FTE 收不到策略委托但两侧日志无异常的分层定界方法与结论：登录/登出(1001/1002)走 NonReqOrder 队列并打 strategy_trade_busi on_message，委托(1003)走独立 Strategy_UTE_ReqOrder 队列经 TryPopReqOrder 直投 DealTradeOrderReq 而无通用日志，故 on_message 记录不能证明委托通道通；含 SendReqOrder/push_share 静默失败点、排查步骤与后续验证建议。"
status: "active"
created_at: "2026-11-19T08:45:13.281Z"
updated_at: "2026-11-19T08:45:13.281Z"
content_hash: "e907f5ca2277ee986e32d0917564269bc1b511fd9f8007ba358150229756f600"
source_paths:
  - "DYS-FRAMEWORK/fte/src/business/strategy_trade_busi.cpp"
  - "DYS-FRAMEWORK/fte/src/business/tcpserver_handler.cpp"
  - "DYS-FRAMEWORK/fte/fte/include/share_communication/share_comm_external_message.h"
session_ids:
  - "8878f1b0-39d3-4bd8-9cad-349adbcf0298"
memory_body_ids:
  []
---

# FTE 共享内存委托（1003）不到达业务层的排查结论（2026-11-19 线上排查）

实例：`UTE_83_831_11`，FTE 14:40:28 由旧 PID 5007 重启为新 PID 42787；
策略 key `38430075080`，客户端 15:01:43 登录、15:01:46 发委托。
相关文档：`85c276bd-e6a0-4cd2-956a-139ae6487dfa`（方案Q：ulStrategyKey 前置 + 业务体 116 零拷贝）、
`008c711d-2f32-4853-ad4d-729bf5339feb`（本地测试栈端口/共享内存链路）、`c2a4915e-4e7b-44f8-afc7-84be6b4b5fec`（DSE↔FTE SPSC）。

## 1. 观测事实（可复用为排查清单）

| 观测点 | 结果 |
|---|---|
| 客户端 attach | 三个队列（NonReqOrder、ReqOrder、回报队列）全部 PASSED |
| 登录 | 15:01:43 FTE `uplink_biz_processor::DealLogon` → SHM `DealLogonAns` 正常 |
| 发委托 | 15:01:46 客户端发出，**无任何发送失败日志** |
| 回报队列 | FTE 已创建 `UTE_Startegy_<key>.queue`，客户端 `Attach Rsp Queue SUCCESS` → `kPktStrategyInit` 链路正常 |
| FTE `strategy_trade_busi.cpp on_message` 50 日志 | 20 条，全部 `iMsgID=1001(LoginReq)` / `1002(LogoutReq)`，**0 条 1003** |
| FTE `DealTradeOrderReq` / `prep_order_shm` / `unexpect msgid` / `too short` | 全部 0 条 |
| 客户端回报 | 24 条全是 `iMsgID=2001`（登录应答），无委托回报 |

## 2. 关键结论：`on_message` 日志不能覆盖委托通道

FTE `ute_message_manager.cpp::StartListenQueue` 的消费循环分两条独立路径：

- `m_StrategyNonReqOrderQueue.TryPopShare(uteMsg)` → `m_pfnOnMessage(uteMsg.iMsgID, …)`；
  登录(1001)、登出(1002)、撤单等非委托请求走这里，因此会打印
  `strategy_trade_busi.cpp on_message 50 | Shm Msg: iMsgID:…`。
- `m_StrategyReqOrderQueue.TryPopReqOrder(reqOrder)` → 直接调用
  `m_pfnOnMessage(kPktOrderReq, order_body, kShmOrderReqBodyLen, ulStrategyKey, ClockNanoTime())`，
  `order_body = &reqOrder + sizeof(ulStrategyKey)`；**该路径没有通用入口日志**，只在
  `StrategyTradeBusi::on_message` 的 `kPktOrderReq` 分支里打 `FTE_SAVE_POINT(T1)` 并进入
  `uplink_biz_processor::DealTradeOrderReq`。

因此「20 条 on_message 全是 1001/1002」**只证明登录通道通，不证明委托通道通**；
这次现象的定性是：共享内存请求队列（`Strategy_UTE_ReqOrder*.queue`）里的 1003 没有被取出，
不是委托内容/字段长度/版本不匹配问题（新旧客户端表现完全相同）。

## 3. 推断的根因候选（按证据强度，未实锤）

1. **客户端 `QueueManager::SendReqOrder` 静默失败**（最可能）：
   ```cpp
   void QueueManager::SendReqOrder(const char* pBuffer) {
       pReqOrderMpmcQueue_->push_share(pReqOrderMpmcShareSlots_, pBuffer);  // 返回值被丢弃
   }
   ```
   `push_share` 在队列满或生产者/消费者对容量掩码理解不一致时失败，且无日志 →
   客户端“以为发成功”，FTE 侧零痕迹。
2. **队列几何不一致**：`/dev/shm` 残留让表头 `mask_` / 读写指针错位，双方对“第几个槽”理解不同，
   写 A 槽读 B 槽，双方都不报错。
3. **FTE 重启后未重新 attach/消费指针不一致**（PID 5007→42787 且新建了 `.queue`）。
   两侧均未打印 `queue init successed, capacity is …`，说明 attach 路径未走到该日志，无法用日志对齐容量。

## 4. 可复用取值/取证步骤

- `ls -l /dev/shm/Strategy_UTE_ReqOrder*.queue`，与 FTE 日志 `CreateShareMemory: Strategy_UTE_ReqOrder` 记录的容量/槽尺寸对比；
- `SendMsg` 之前确认 `m_bRspQueueInit` 与 `TryPopReqOrder` 两侧是否走同一 `pReqOrderMpmcShareSlots_`；
- 线上代码（`/home/lsz/code/src/gt_src`）核对：有 `policy_id`、有方案Q（`ulStrategyKey` 前置 + `reserved[8]` + `kShmOrderReqBodyOff/Min/Len`）、**没有 `FundChangeNotify`**——
  与「只算昨天 SVN 升级」一致，但源码 ≠ 运行中的二进制，需确认当日 FTE 二进制的来源。
- 排查时环境已无 FTE 进程、`.queue` 已被清理，因此当时无法读取表头取证。

## 5. 建议的根治与验证（当时提交给用户的下一步）

- 客户端 `SendReqOrder` 的 `push_share` 返回值透出并在失败时 `LOG_ERROR`（或在 `SendMsg` 内判一次），
  把“静默丢委托”变成显式错误日志；
- 本地最小复现：容器内起 mock 服务端 + 新客户端走“登录+委托”，确认本地委托队列通路（此前 09:30 测试是通的）。
