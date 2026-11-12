---
id: "5efa9314-4724-45e0-a46f-2f1ad6add9ad"
title: "DYS-FRAMEWORK r87468：agw_client_impl.cpp 冲突裁决 + 两处连带编译断裂根因（2026-11-12）"
description: "可复用记录：agw_client_impl.cpp 更新到 r87468 后的单块冲突根因（duplicate case value 双方各修一次，取线上语义等价）与 md5 逐字节校验法；同时定位两处\"与本次合并无关\"的既存编译断裂根因（strategy_trade_busi 5 参 vs 4 参回调、ins_message.h 6 个风控字段被 git 提交 23c690c0a 删除但 dse 仍引用）及恢复办法、全量编译证据与待人工处理清单。"
status: "active"
created_at: "2026-11-12T09:52:25.359Z"
updated_at: "2026-11-12T09:52:25.359Z"
content_hash: "5fe6069b053d04435efcd60d5a8cb6ca80a6f3f232727323bb5bbac580fa91de"
source_paths:
  - "svn_work/107_agw_client_impl冲突分析与合并_20261112.md"
  - "DYS-FRAMEWORK/api/gw_api_boost/api/src/agw_client_impl.cpp"
  - "DYS-FRAMEWORK/fte/src/business/strategy_trade_busi.cpp"
  - "DYS-FRAMEWORK/fte/include/message/ins_message.h"
  - "DYS-FRAMEWORK/fte/dse/src/data_sync_engine.cpp"
  - "work_log/20261112.md"
session_ids:
  - "39fe0cd8-b63d-4a2f-9415-7f4c12e09967"
memory_body_ids:
  []
---

仓库 `/mnt/work/gt_trunk/DYS-FRAMEWORK`，工作副本 update 至 **r87468**（git HEAD `a3b7652fa`，仓库 Revision 87478）。同类冲突的通用口径见 `a7e662dd-488d-462b-9523-ba4004d5ebd7`（备份本地 → 取线上 → md5 校验 → 本地改动后补）；CRLF 假冲突判别见 `30120f0a-95c5-4559-9dc8-55141c5e41eb`。仓内成文版：`svn_work/107_agw_client_impl冲突分析与合并_20261112.md`。

## 1. 冲突：1 块（3582~3592，11 行），四份文件都是 LF

不是行尾假冲突。**根因：同一个 `duplicate case value` 缺陷被双方各修了一次**，落点相同但上文不同 → diff 无法对齐成更小块。

| 侧 | 修法 |
|---|---|
| 线上 r87468 | `#endif` 上移关闭 UTE 段；case 标签**只留一个**（无条件编译），两套逻辑下沉到 `#ifdef AGW_UTE_API` / `#ifndef AGW_UTE_API` 的**语句体** |
| 本地 `.mine` | 保留两个 case 标签，第二个整体用 `#ifndef AGW_UTE_API` 包住 |

语义等价，**取线上不损失功能**（线上写法更彻底）。

## 2. 合并与一致性校验（只认 md5 + svn diff）

| 步骤 | 证据 |
|---|---|
| 备份本地 | `agw_client_impl.cpp.bak_20261112`，181,950 B，md5 `64c669202ca59ceee1cd321c11c92bec` |
| 覆盖为 r87468 | 181,111 B |
| 与线上逐字节一致 | WC md5 `ff6a075f727a330ac68de118339bdd11` == `svn cat -r 87468` md5 ✅ |
| 冲突标记 / `svn diff` | 0 / 0 行 |

**坑**：内容一致后 `svn status` 仍显示 **C**（冲突记录留在 `.svn` 元数据），须由用户执行 `svn resolve --accept working api/gw_api_boost/api/src/agw_client_impl.cpp`。

取线上后会丢的两处本地改动：`Connect()` 里包住 UTE 直连初始化段的 `/* */` 被删（P0-1，线上 r87468 仍未修，不恢复则 UTE 直连登录超时）；`#if 0` 包住 mac/ip 填充（P0-2，当前 codec 已含 mac/ip，无需该守卫）。

## 3. 两处"与本次合并无关"的既存编译断裂

### B1 `fte/src/business/strategy_trade_busi.cpp` — 原型不匹配

本地 cpp 是 **5 参**（多 `int buffer_size`，用于 `buffer_size < sizeof(TradeOrderReq)` 长度校验）；而 `strategy_trade_busi.h`(L17)、`main.cpp`(L487 的 `std::bind` 4 个占位)、`share_comm_external_message.h` 回调类型都是 **4 参**（线上版）。svn 状态显示该 cpp 为 `M`。
处置：备份 `strategy_trade_busi.cpp.bak_20261112` 后**对齐线上 4 参**（md5 `96ecbb2d87552411521a6865cbb82590`，`svn status` 已干净）。
要恢复校验必须**成套改 4 处**：头文件加参 + cpp 加参并使用 + `main.cpp` bind 加第 5 占位 + `share_comm_external_message.h` 回调类型加 `int`。

### B2 `fte/include/message/ins_message.h` — 风控字段被 git 提交删除但仍有引用

`dse/src/data_sync_engine.cpp`（L403~406、L1075~1080）引用 `PktRTControlInstructionFundInfo` 的 `sh/sz_check_order_per_sec_ / _per_day_ / _cancel_per_order_` 共 6 个字段，头里没有（本地与 r87468 都缺）。
根因：git 提交 **`23c690c0a「D:更新svn线上代码」** 在 `ins_message.h` 中**删掉 6 个字段声明 + 6 行 reset 赋值**（`@@ -1124` 与 `@@ -1172` 两个 hunk），而 dse 未同步 → 一直靠旧产物在跑。
处置：备份 `ins_message.h.bak_20261112`，按删除前版本恢复 6 字段 + 6 行 reset。
风险提示：这是**本地补丁**，下次 `svn update` 会被再覆盖 → 建议整理成 patch 交客户，随 P0-1 一起。

## 4. 全量编译证据（容器内 `make -j6`，0 error）

`ute` ✔ `dse` ✔ `utedatainit` ✔ `agwapi` ✔ `agw_ute_api` ✔ `agwapi_no_boost` ✔ `agw_ute_api_no_boost` ✔ `mock_client` ✔ mock 组件 ✔
产物：`ute` 168,904,512 B / `dse` 5,907,600 B / `libagwapi.so` 17,062,912 B / `libagw_ute_api.so` 23,319,136 B / `mock_client` 5,723,912 B。

## 5. 待人工处理

1. P0-1：恢复 `Connect()` 里 UTE 直连的 `/*`(~2086) 与 `*/`(~2141)。
2. 若需 SHM 报文长度校验：按 §3-B1 成套改 4 处。
3. `ins_message.h` 6 字段补丁整理为 patch。
4. `svn resolve`（文件级已一致，元数据未清）。

## 6. 教训

- 冲突块少不代表冲突简单：先看两侧是否在同一缺陷上各写一版，语义等价时取更彻底的一侧。
- 编译断裂要分清「本次合并引入」与「既存断裂」：本次两处都能用 `svn status` / `git log -S` 定位到来源（未提交的本地改动 / 一次「更新线上代码」的提交）。
- 老仓库常有「头文件被更新、引用方未跟、靠旧 .o 运行」的隐性债，全量重建才会暴露。
