---
id: "a7e662dd-488d-462b-9523-ba4004d5ebd7"
title: "SVN 冲突最终处置口径：本地日期备份 + 两文件全部取远端 r87459 + 本地改动补丁重放（DYS-FRAMEWORK，2026-11-12）"
description: "可复用操作规程：用户确定的 SVN 冲突处理原则（备份本地原文件加日期后缀、冲突文件全部以远端为准与线上逐字节一致、本地改动生成补丁后续手动重放），含 md5/svn diff 校验法、svn status 静默但 svn info 仍记冲突的坑、参照副本 .mine/.r87082/.r87459 重建方法，以及 4 组本地改动的入库建议。是 30120f0a（CRLF 假冲突诊断 + diff3 三方合并）的最终处置替代口径。"
status: "active"
created_at: "2026-11-12T07:00:49.552Z"
updated_at: "2026-11-12T07:00:49.552Z"
content_hash: "27524b7f06222cde308fdc83570c2ef6d6705c154e8026f0116c1857be769f06"
source_paths:
  - "svn_work/81_SVN更新冲突分析与合并方案_20261112.md"
  - "work_log/20261112.md"
  - ".tmp_merge_20261112/local_changes_20261112.patch"
  - "DYS-FRAMEWORK/fte/src/business/uplink_biz_processor.cpp"
  - "DYS-FRAMEWORK/fte/fte/include/share_communication/share_comm_external_message.h"
session_ids:
  - "2df1e0fe-2fbf-4898-b289-77faae850c9b"
memory_body_ids:
  []
---

# SVN 冲突最终处置：以远端为准 + 本地日期备份 + 补丁重放

仓内权威正文：`svn_work/81_SVN更新冲突分析与合并方案_20261112.md` 的 **§6.5 最终处置**（一页速查已同步）；工作日志 `work_log/20261112.md`；提交 `3dacdf141`（`trunk` 0 0）。
前置诊断（为何是「1 个冲突块覆盖整文件」＝CRLF 行尾假冲突，19141 → 163 行差异落差）见既有文档 `30120f0a-95c5-4559-9dc8-55141c5e41eb`。**本文档覆盖其「diff3 三方合并、双方全保留」的推荐做法**：用户最终选择更保守的口径。

## 1. 用户确定的原则（后续同类冲突照此办理）

> 先把当前冲突的本地原文件做备份、以日期为后缀；然后这些文件**全部以远端为准**，保证与 SVN 线上代码一致；剩下的本地改动**由用户手动修改重放**。

落地顺序：①备份 → ②远端版覆盖 → ③与线上逐字节校验 → ④清冲突标记 → ⑤生成本地改动补丁供重放 → ⑥（用户执行）`svn resolve`。
本次对象：`fte/src/business/uplink_biz_processor.cpp`、`fte/fte/include/share_communication/share_comm_external_message.h`，工作副本 `02Trunk` 已 update 至 **r87459**。

## 2. 已执行的处置与校验（文件级，未碰 SVN 元数据）

| 步骤 | 动作 | 校验 |
|---|---|---|
| ① | 本地原文件改名备份：`.cpp.bak_20261112`（417,787B）、`.h.bak_20261112`（13,034B） | 备份 md5 `d1ac10f2f1b89dbc93435add383cc8da`（cpp）＝更新前本地版本 `.mine` |
| ② | 用 `*.r87459` 覆盖两文件 | 行尾回归 **CRLF**（与线上提交一致） |
| ③ | WC vs `svn cat -r 87459` | **md5 完全一致**：`a380bfc2621535dace9b40aa857e0fe2`（cpp）、`c9c59013b750a5bb894d088d09c72faa`（h） |
| ④ | 冲突标记 / `svn diff` | 标记 **0**；`svn diff` **无输出**（＝与仓库版本逐字节相同） |

判定「是否真的与线上一致」只认 **md5 比对 + `svn diff` 无输出**，不要用「grep 宏计数」之类内容旁证推断（本次第一轮复查即因此被纠正）。

## 3. 两个必踩的坑

1. **`svn status` 静默 ≠ 冲突已解决。** 内容与 pristine 逐字节相同后，`svn status <file>` 对这两个文件**无任何输出**（视为未修改），但 `svn info` 里 `Conflict Previous Base File` / `Conflict Previous Working File` / `Conflict Current Base File` 三条仍在。因此 `svn resolve --accept working` 不能省，否则后续 `svn commit` 状态不确定。
2. **参照副本不能提前删。** 删掉 `.mine` / `.r87082` / `.r87459` 后 resolve 会引用缺失文件；需按 `svn info` 的三条元数据用 `svn cat -r 87082/87459` 重新生成 `.r87082`/`.r87459`、以备份复制成 `.mine`。resolve 成功后才可清参照副本，`.bak_YYYYMMDD` 建议保留到本地改动重放完成。

## 4. 本地改动留档与重放

- 补丁：`.tmp_merge_20261112/local_changes_20261112.patch`（101 行 / **6 hunk**）、`local_changes_header_20261112.patch`（14 行 / 1 hunk）。
- 可重放性证明：对 LF 归一后的远端版 `patch --dry-run -p0` → **6/6 hunk 全成功**，偏移 +3/+3/+30/+51/+89/+89（说明远端新增的 URC 通道等改动没有顶掉本地落点）。
- 重放顺序建议：先 `sed 's/\r$//'` 归一 LF 再 `patch -p0`，否则 CRLF 下 hunk 匹配会失败。

4 组本地改动及入库建议：

| # | 位置 | 改动 | 建议 |
|:-:|---|---|---|
| 1 | `uplink_biz_processor.cpp`（2 处 `static_assert`） | GCC5+ 分支改 `offsetof` + `std::is_standard_layout` | ✅ 真实可移植性修复，建议入库 |
| 2 | 同上（`SendOrderToExch`） | `#ifdef FTE_LATENCY` → `FTE_NETCARD_TEST`（抓包关联写 `user_info`） | ⚠️ 测量设施，入库前与客户确认 |
| 3 | 同上（对象池创建） | `etf_sync 1000→10000`、`fte_report`/`fte_reject 2000→150000`、`*_fte_etf_report 2000→20000`、`sz_internal_etf_report 8→2000` | ❌ 测试取数专用，提交 SVN 必须改回 1000/2000 |
| 4 | `share_comm_external_message.h` | 回调签名加第 5 个参数 `int`（buffer size）+ 2 行字段注释 | ⚠️ 接口签名变更，需调用方同步 |

远端侧新增（供重放时避让）：URC 现券通道 `sz_cash_urc_`/`sh_cash_urc_`（委托与撤单同会话、失败直接拒单不回退轮转）、QFII/额度 `StrongCheck` 系列、`personalized_flag_ != 2` 收敛、`SendTradeOrderER` 的 `available_t1` 回填 + `PktFundChangeNotify` 重载，头文件新增 `business_type`/`no_security`/`constituent_stock[0]`/`available_t1`。

## 5. 后续待办与防复发

- **待用户执行**：两条 `svn resolve --accept working`（SVN 写操作按铁律不由 agent 执行）；本地 4 组改动重放；`svn commit` 前的 14 个无关 `M` 文件按 `svn_work/04_沙箱临时改动清单.md` 区分保留/还原。
- 防复发：建议对这两个文件设 `svn:eol-style native`（仓库抽样 200 个 `.cpp/.h` 仅 3 个 CRLF，属提交侧行尾污染）；git 侧 `.gitattributes` 已是 `* text=auto eol=lf`，故 `DYS-FRAMEWORK` 这两个 CRLF 文件在 git 工作树恒显 `M`，且 `.bak_20261112`/`.mine`/`.r87082` 会以未跟踪文件出现——**这些参照副本与备份不要提交进 git**。
- 整树 `svn status` 会报 `E000022`（非 UTF-8 文件名），需按子目录/按文件查状态规避。
