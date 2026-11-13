---
id: "85c276bd-e6a0-4cd2-956a-139ae6487dfa"
title: "FTE 共享内存通道兼容改造（方案Q）提交清单与状态：8 文件最小闭环、CRLF 归一、main.cpp 混装剥离（2026-11-13）"
description: "可复用提交清单：方案Q（key 前置 + 业务体 116 零拷贝）在 DYS-FRAMEWORK/fte 下的 8 文件最小闭环、3 文件必须先转 CRLF、main.cpp 混装 hunk 剥离与还原补丁、policy_id 已在 r87543 提交的事实，以及尚未验证的回归与工单适用性风险。"
status: "active"
created_at: "2026-11-13T08:55:46.617Z"
updated_at: "2026-11-13T08:55:46.617Z"
content_hash: "6e829a51cc64c26c3b3caa6ad1e647bbe455fd785719831bc73388b82ef874c2"
source_paths:
  - "task/task_svn/task_svn.txt"
  - "svn_work/111_策略端共享内存委托兼容方案_20261112.md"
  - "svn_work/112_策略端共享内存兼容_实施与验证总结_20261113.md"
  - "svn_work/113_长度契约去魔数与字节自检接入_20261113.md"
  - "svn_work/114_回报通道缓冲上限编译期检查_20261113.md"
  - "svn_work/115_共享内存兼容性测试文档_20261113.md"
  - "svn_work/main_dse_changes_restore_20261113.patch"
  - "DYS-FRAMEWORK/fte/fte/include/share_communication/share_comm_external_message.h"
  - "DYS-FRAMEWORK/fte/src/business/uplink_biz_processor.cpp"
  - "DYS-FRAMEWORK/fte/fte/src/main.cpp"
  - "work_log/20261113.md"
session_ids:
  - "a38605bf-d905-42fc-afe8-6af28d33ca87"
memory_body_ids:
  []
---

# FTE 共享内存通道兼容改造（方案Q）— 提交清单与状态

工作副本：`/mnt/work/gt_trunk/DYS-FRAMEWORK`（SVN `02Trunk`，容器内路径）。
依据文档：`svn_work/111`（方案）、`112`（实施与验证总结）、`113`（长度契约去魔数）、`114`（回报通道缓冲上限编译期检查）、`115`（测试文档）。
相关 Managed Document：`9edbc756-0ad0-4081-81e8-475f7983f83e`（policy_id 三通道方案，含 106 偏移撞车根因）、`30120f0a-95c5-4559-9dc8-55141c5e41eb`（CRLF 整文件假冲突处置手册）、`c2a4915e-4e7b-44f8-afc7-84be6b4b5fec`（DSE SPSC 双队列，解释被剥离的 DSE hunk 来源）。

> 本文件是「待提交清单 + 已知风险」。证据来自一次完成度检查点，未逐条在会话内重跑；引用数字均为该检查点实测值。

## 1. 改造要点（一句话链路）

改造前 SHM 通道结构 114B，FTE 内部按 `message::TradeOrderReq` 116B 解读，且 `ulStrategyKey` 位置压在内部 `reserved` 区 → 长度不足被拒单或字段静默错位。方案 Q：`ulStrategyKey` 前置到偏移 0（业务体 = 对象地址 + 8）；业务体 108（70+38，与 `TradeOrderReqLayout` 同构）+ `reserved[8]` = 116 与内部等长 → 走 `kPass` 零拷贝；构造函数 `memset(this,0,sizeof(*this))` 保证补齐区为 0。配套：回调链增 `buffer_len`（0=未提供）、`prep_order_shm()` 三分支归一化（kPass / kReuse 补零 / kInvalid 拒单，后者用 `kIllegalFormat 50009` 且不读不可信数据）、回报侧 `UteMsg::strMsgBuf[2048]` 编译期上限断言。**策略端客户端一行未改。**

## 2. 8 文件最小闭环（缺任一个都编不过）

| 文件（相对 DYS-FRAMEWORK） | svn | 真实改动（转 CRLF 后） | 内容 |
|---|:-:|--:|---|
| `fte/fte/include/share_communication/share_comm_external_message.h` | M | +99 −9 | key 前置 + `reserved[8]` + memset + `kShmOrderReqBody*` 常量 + 关系式断言 + 回报缓冲断言 + 回调加 `buffer_len`；**同时含 SHM 通道 policy_id** |
| `fte/fte/src/share_communication/ute_message_manager.cpp` | M | +8 −2（2 hunk） | 普通消息传 `iMsgLen`；委托 `body = &reqOrder + sizeof(ulStrategyKey)` |
| `fte/fte/src/share_communication/lock_file_manager.cpp` | M | +2 −1 | 策略死亡事件回调 `buffer_len` 传 0 |
| `fte/include/business/strategy_trade_busi.h` | M | +3 −1 | `on_message` 增 `unsigned int buf_size` |
| `fte/src/business/strategy_trade_busi.cpp` | M | +5 −3 | 签名同步 + 透传 `DealTradeOrderReq` |
| `fte/include/business/uplink_biz_processor.h` | M | +15 −1 | `length` 形参、`LenCheckResult`、`prep_order_shm()`、`order_reuse_[256]` |
| `fte/src/business/uplink_biz_processor.cpp` | M | +46 −2 | 三分支归一化；`UtilityCheck` 第三参改用 `length` |
| `fte/fte/src/main.cpp` | M | +2 −1（1 hunk） | `std::bind` 增第 5 个占位符（回调 4→5 参） |

合计真实改动约 **+180 / −20**。
注意路径**分叉**：`share_communication/` 与 `main.cpp` 在 `fte/fte/`，`business/` 在 `fte/` —— 写提交命令时极易写错前缀。
`strategy_message_manager.cpp/.h` 与 `ute_message_manager.h` 经核对为 clean → 客户端零改（仅因头文件变化重编即通）。

## 3. 三个文件必须先转 CRLF

SVN 里是 CRLF、工作副本被 git `.gitattributes eol=lf` 转成 LF，直接提交会整文件重写（`+408/−377`、`+558/−620`、`+8897/−9612`）。

```bash
perl -i -pe 's/\r?\n/\r\n/' \
    fte/fte/include/share_communication/share_comm_external_message.h \
    fte/include/business/uplink_biz_processor.h \
    fte/src/business/uplink_biz_processor.cpp
# 校验：file 应含 "with CRLF line terminators"；svn diff 收敛到 +99-9 / +15-1 / +46-2
```

## 4. main.cpp 混装剥离

| hunk | 内容 | 归属 |
|:-:|---|---|
| 1 | `FTE_DSE_FIFO_LEN/MD_LEN` 8192→65536 | ❌ DSE 队列尺寸，与本改造无关 |
| 2 | `#ifdef NO_DSE` 新增投放线程（防队列满忙等） | ❌ DSE 联调，与本改造无关 |
| 3 | `std::bind` 第 5 占位符 | ✅ 必需 |

已剥离 1、2（文件 567→550 行），只留 hunk 3；剥离内容存 `svn_work/main_dse_changes_restore_20261113.patch`（37 行，回贴后 md5 与备份一致）。剥离后 `make ute` 0 error。

## 5. 已发生的提交（勿重复提交）

`r87543 | 122098 | 2026-11-13 08:40 | [NGTP-T202613076]`
- `M api/gw_api_boost/api/src/ute_client_handler_impl.cpp`
- `M api/gw_api_no_boost/api/src/ute_client_handler_impl.cpp`
- `A fte/include/message/gw_head.h`（**新增入库**）

→ TCP API 通道的 policy_id 已生效。SHM 通道 policy_id 仍在第 2 节那个头文件里，因此**只有完成第 2 节提交，两个通道的 policy_id 才同时生效**；在此之前 SHM 通道仍读到策略 Key 低位。

## 6. 提交命令（8 文件同一提交）

```bash
cd /mnt/work/gt_trunk/DYS-FRAMEWORK
# 先做第 3 节 CRLF 归一
svn commit -m "[NGTP-T202613076] 共享内存通道兼容性改造（方案Q：key前置+业务体116零拷贝）" \
    fte/fte/include/share_communication/share_comm_external_message.h \
    fte/fte/src/share_communication/ute_message_manager.cpp \
    fte/fte/src/share_communication/lock_file_manager.cpp \
    fte/include/business/strategy_trade_busi.h \
    fte/src/business/strategy_trade_busi.cpp \
    fte/include/business/uplink_biz_processor.h \
    fte/src/business/uplink_biz_processor.cpp \
    fte/fte/src/main.cpp
```

## 7. 未纳入 / 不在 SVN 的对象

- `fte_all/` 整目录**未纳管**（`svn: E200009` / status `?`），含 `compile_fte_all.sh` 里 `check_wire_abi.sh` 的接入 → 无需提交。构建集成改动只存在于工作目录。

## 8. 已知风险（提交前须处理）

1. **DSE 改动的唯一副本是 git 未跟踪文件**。剥离后工作副本里已无 `FTE_DSE_FIFO_LEN=65536` 与 NO_DSE 投放线程；`svn_work/main_dse_changes_restore_20261113.patch` 在 git 中仍是 `??`。应 git add/commit 该补丁，否则一旦丢失需重写；若 DSE 改进本身仍要上线，需在 SHM 提交后回贴。
2. **工单号适用性未证实**。`NGTP-T202613076` 在 r87543 的提交说明是「增加策略编号 policy_id 字段」，方案 Q 属另一项改造（文档 111–115 自成体系）。沿用同一工单便于追溯的一致性，但需确认工单范围是否覆盖；否则应换工单。
3. **提交说明未含 policy_id**。第 2 节头文件同时携带 SHM policy_id，提交说明只写「兼容性改造」→ 追溯缺失。建议说明中补一句 SHM 通道 policy_id 一并生效。
4. **CRLF 归一会让 git 镜像侧再次出现整文件差异**。git `.gitattributes eol=lf` 会把 3 个文件视为全量改写；提交后需在 gt_trunk/gt_svn/gt_sit 三仓复核并归位，别把噪声当成新改动。
5. **功能回归未取证**。检查点只证明 `make ute` 通过；任务要求里的「用老 API 的 mock_client 对 fte 做全面功能回归」没有对应证据。上线前应补跑。
6. **上线约束**：段布局变为 124B（key 前置），FTE 与所有策略端**必须同版本同批升级**，并清空 `/dev/shm` 重启。
