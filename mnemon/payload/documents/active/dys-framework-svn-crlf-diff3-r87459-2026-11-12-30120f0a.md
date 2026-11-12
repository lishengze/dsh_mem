---
id: "30120f0a-95c5-4559-9dc8-55141c5e41eb"
title: "DYS-FRAMEWORK SVN 整文件假冲突（CRLF 行尾）诊断与 diff3 三方合并处置（r87459，2026-11-12）"
description: "可复用手册：svn update 后出现「1 个冲突块覆盖整文件」时，如何判定为 CRLF/LF 行尾假冲突，用 strip-trailing-cr + diff3 一次干净合并，并用行数自洽性验证双方改动均未丢失。"
status: "active"
created_at: "2026-11-12T06:47:28.793Z"
updated_at: "2026-11-12T06:47:28.793Z"
content_hash: "6689f0abf2c40e6fa4405145a33d6d83f13e92d7a4a20ab7c41397619af4cd01"
source_paths:
  - "svn_work/81_SVN更新冲突分析与合并方案_20261112.md"
  - "work_log/20261112.md"
  - "DYS-FRAMEWORK/fte/src/business/uplink_biz_processor.cpp"
  - "DYS-FRAMEWORK/fte/fte/include/share_communication/share_comm_external_message.h"
session_ids:
  - "b9b11207-586f-4fc9-9aa8-f465fb188ef7"
memory_body_ids:
  []
---

# DYS-FRAMEWORK SVN 整文件假冲突（CRLF 行尾）处置手册

仓库内完整报告：`svn_work/81_SVN更新冲突分析与合并方案_20261112.md`；当日工作日志 `work_log/20261112.md`；
提交 `33dbb1d58`（三仓 gt_trunk / gt_svn / gt_sit 均 0 0）。对象：`/mnt/work/gt_trunk/DYS-FRAMEWORK`（`02Trunk`，update 至 **r87459**）。

## 1. 症状与判别标准

`svn update` 后出现 **1 个冲突块覆盖整文件**（而不是若干局部 `<<<<<<<` 块），基本可判为行尾差异而非内容冲突：

| 文件 | 冲突块 | 行数 |
|---|---|---|
| `fte/src/business/uplink_biz_processor.cpp` | 1 块，1~19179 | 19179 |
| `fte/fte/include/share_communication/share_comm_external_message.h` | 1 块，1~754 | 754 |

判别：冲突块数 = 标记行数 2（仅 `<<<<<<< .mine` 与 `=======`），且块覆盖首行到末行 → 整文件一坨。
注意 SVN 三份参照 `*.r87082`(BASE) / `*.mine` / `*.r87459`(THEIRS) 必须保留齐全，否则无法三方合并。

## 2. 根因量化（CRLF vs LF）

| 比较方式 | 差异行数 |
|---|--:|
| `diff base.r87082 theirs.r87459`（含行尾） | 19141（文件共 9529 行，几乎全不同） |
| 同上加 `--strip-trailing-cr` | **163** |
| 头文件（忽略 CR） | **5** |

- `.r87459`（远端版本）是 **CRLF**，本地工作副本是 **LF** → 每行都判为“不同” → SVN 只能吐出整文件一个冲突块。
- 抽样 `fte`/`Common`/`api` 下 200 个 `.cpp/.h`：**CRLF=3，LF=197**，这 3 个 CRLF 正好包含这两个冲突文件与 `uplink_biz_processor.h` → 属于**提交侧行尾污染**。
- 教训：遇到整文件冲突先做一次 `--strip-trailing-cr` 对比，用差异行数落差（19141 → 163）即可在五分钟内定性，不必逐行读 1.9 万行。

## 3. 重叠判定：按函数归类不等于逐行重叠

按 hunk 归属函数求交集只得 `DealTradeOrderReqBusi`、`SendOrderToExch` 两个“重叠函数”，但落点不同区段：

- 本地：函数后段 `static_assert` 改写、`FTE_NETCARD_TEST` 抓包关联块、URC 网关选择之后的发单/撤单后置逻辑；
- 远端：函数前段 URC 现券通道网关选择（`sz_cash_urc_`/`sh_cash_urc_`）、QFII/额度 `StrongCheck` 系列、`personalized_flag_ != 2` 收敛、`SendTradeOrderER` 的 `available_t1` 回填与 `PktFundChangeNotify` 重载。

结论：**零真实重叠，无需人工逐行取舍**。方法要点：用「冲突 hunk → 所在函数」映射做初步交集，再回读交集函数的行号区间确认落点。

## 4. 推荐处置流程（双方全保留）

```bash
M=/tmp/svn_merge; mkdir -p $M
# ① 远端先归一成 LF（关键：不归一就一定会再制造整文件冲突）
sed 's/\r$//' uplink_biz_processor.cpp.r87459 > $M/theirs.cpp
cp uplink_biz_processor.cpp.mine $M/mine.cpp
cp uplink_biz_processor.cpp.r87082 $M/base.cpp
# ② 三方合并（头文件同理）
cd $M && diff3 -m mine.cpp base.cpp theirs.cpp > merged.cpp
# ③ 落盘 + 标记已解决（写操作由用户执行）
cp merged.cpp <wc>/fte/src/business/uplink_biz_processor.cpp
svn resolve --accept working <wc>/fte/src/business/uplink_biz_processor.cpp
```

自洽性验证（比“看起来对”可靠）：

| 对照 | 期望 |
|---|---|
| `merged` 冲突标记数 | 0 |
| `merged − theirs` | 45 行（= 本地改动量，证明本地没丢） |
| `merged − mine` | 163 行（= 远端改动量，证明远端没丢） |

关键内容旁证：`is_standard_layout<InternalOrder>`=8、`FTE_NETCARD_TEST`=3、`create("fte_report", 150000)`=1；`use_cash_urc`=8、`available_t1`=3。全部命中即视为合并内容正确。

**不推荐** `svn resolve --accept theirs-full / mine-full`：分别丢掉本地 45 行或远端 163 行；也不推荐手工挑块——整文件块里行尾差异与真实改动肉眼无法分辨。

## 5. 落盘后必须做的验证

1. 冲突标记清零（`grep -c '^<<<<<<< \|^=======$\|^>>>>>>> '` = 0）；
2. 双方改动都在（第 4 节计数）；
3. 头文件新字段 `business_type` / `no_security` / `constituent_stock[0]` / `available_t1` 到位；
4. **全量编译 0 error**（`compile_fte_all.sh -r`）。`-fsyntax-only` 单文件预检**不可作为判据**：临时 include 树无法还原跨目录相对包含（`../../../include/common/ute_types.h`、`./../../../../Common/util/include/abf_math.h`），会报 `No such file` 假失败——真实树里这些头是从 `DYS-FRAMEWORK/fte/../Common`（即 `DYS-FRAMEWORK/Common`）找到的；
5. 对象池容量口径确认；6. 功能回归（沪 14/14、深 8/8）；7. 性能抽测（`识别业务报文>0`、无 `capacity should resize`）。

## 6. 顺带发现与复发预防

- 整树 `svn status` 报 `E000022`（非 UTF-8 文件名，如 `msg_sender_release/Release/*.docx`、`fte_tcp_通信链路分析.md`）→ 改为按子目录/按文件查状态规避。
- 同次 update 另有 14 个无关 `M` 文件（`Common/biz_msg/*`、各 `CMakeLists.txt`、`business_type` 相关头），提交前需按 `svn_work/04_沙箱临时改动清单_20261104.md` 区分保留/还原。
- **复发预防：对这两个文件设 `svn:eol-style native`**；新增/修改文件避免以 CRLF 提交。
- 临时合并产物放 `.tmp_merge_20261112/` 并加入 `.gitignore`（已确认未被 git 跟踪）。

## 7. 需用户决策的三处口径

| 项 | 建议 |
|---|---|
| 对象池容量放大到 150000/10000 | 仅测试取数用（见知识库 17 号），**提交 SVN 前需改回 1000/2000** |
| `FTE_LATENCY` → `FTE_NETCARD_TEST` 抓包关联（`client_seq_id` 写入 `user_info`） | 测量设施改动，入库前与客户确认 |
| GCC5+ `static_assert` 改用 `offsetof` + `std::is_standard_layout` | 真实可移植性修复，建议入库 |

## 8. 环境坑

- 容器内 Python 是 **3.6**：`subprocess.run(..., capture_output=True)` 直接 `TypeError`，需用 `stdout=PIPE, stderr=PIPE`。
- 该容器默认 locale 为 ascii：脚本里打印中文/非 ASCII 会 `UnicodeEncodeError`（`'ascii' codec` / `surrogates not allowed`）；分析脚本改用英文输出或显式 `sys.stdout` 编码，命令行解析也避免传中文。
