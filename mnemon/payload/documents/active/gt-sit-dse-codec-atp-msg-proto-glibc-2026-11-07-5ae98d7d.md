---
id: "5ae98d7d-73b9-4432-b1d8-1f1a28b52256"
title: "gt_sit dse 编译阻塞：分支内 codec/ATP 生成物陈旧与 msg_proto 的 GLIBC 约束（2026-11-07）"
description: "gt_sit 分支 dse 编译 345→10 处错误的根因链：agw_external.xml 已含 FPGA 报文但生成物陈旧、msg_proto 需 GLIBC 2.34 无法重生成、dse_deps 精确头覆盖做法与三条踩坑教训、优先向分支维护者索取配套生成物。"
status: "active"
created_at: "2026-11-07T03:08:36.982Z"
updated_at: "2026-11-07T03:08:36.982Z"
content_hash: "7a86c793e33ac42fd61b88f21470ffd766a562a54c941e6f0f6a9becde73ec42"
source_paths:
  - "svn_work/57_gt_sit_DSE联调与测试总结_20261107.md"
  - "DYS-FRAMEWORK/fte/dse/CMakeLists.txt"
  - "DYS-FRAMEWORK/gw_external_msg/V2/buildmsg/agw_external.xml"
  - "DYS-FRAMEWORK/gw_external_msg/V2/external/include/gw/external/codec/types_agw_external.h"
  - "DYS-FRAMEWORK/fte/dse/dse_client/include/dse_client.h"
session_ids:
  - "99f38c26-e508-42c8-bcad-a93002578143"
memory_body_ids:
  []
---

# gt_sit 的 dse 编译阻塞：分支内自动生成物陈旧（2026-11-07）

**一句话**：gt_sit 的 dse 编译失败不是构建方式问题，而是该分支快照自身不一致——`agw_external.xml` 已含新报文，但配套的自动生成 codec / ATP 报文头仍是旧版；唯一的代码生成器 `msg_proto` 依赖 GLIBC 2.34，容器为 2.17，无法原地重生成。

## 1. 症状

`dse` 编译报 `has not been declared`，累计 345 处错误，集中在两类：

- `atp::gw::AgwMsg*FPGAOrder`、`AgwMsgCancelOrderSync`、`AgwMsgMarginTradingExtOrder`、`AgwMsgCreditAuctionExtOrder` 未声明（引用处为 `dse_client.h`）
- `class abf::util::PktCashAuctionTER has no member named 'skip_cal' / 'stk_effect_type1' / ...`

## 2. 根因链（证据）

| 环节 | 结论 | 证据 |
|---|---|---|
| codec 头是否缺文件 | 不缺失，6 个头文件都在，但**内容旧** | `types_agw_external.h`：gt_svn 1171 行 / gt_sit 930 行，`FPGAOrder` 出现 1 / 0 次 |
| 报文定义 | gt_sit 的 `buildmsg/agw_external.xml` **已含** FPGA 报文 | 与 gt_svn 归一化行尾后真实差异仅 295 行，且多为元素顺序/空白，不是缺报文 |
| 生成器能否重跑 | 不能 | `msg_proto` 在共享 ATP 资源包内；拷到 /tmp 执行报 `GLIBC_2.34 not found`；源目录无 FPGA 模板，生成物只能来自该二进制 |
| ATP 报文头 | 与所需字段无关 | ATP 资源包与仓库 Common/util 的 `types_atp_te.h`/`messages_atp_te.h` 均无 `skip_cal`；`PktCashAuctionTER` 正定义在 `DYS-FRAMEWORK/Common/biz_msg/include/types_biz_msg.h`，两条构建路径都未将其纳入 include |

结论：**dse 源码（新版）与 codec / Common 生成物（旧版）在同一分支内版本不一致**。

## 3. 已落地的构建适配（可复用做法）

前提：FTE 侧不引用 AGW codec 头，仅 dse 引用。因此不覆盖分支文件，而是在 `DSE_PREBUILT_LIB_DIR`（`dse_deps`）内做"精确头覆盖"，用 `include_directories(BEFORE)` 注入。

| 覆盖目录 | 内容 | 作用 |
|---|---|---|
| `agw_codec_include/`（嵌套目录 + 根层各一份） | 6 个配套 codec 头 | 补齐 FPGA 等新报文声明 |
| `atp_override/` | 仅 7 个 `*_atp_te.h`（**不含** `biz_param.h`） | 提供 ATP 侧新字段，同时不遮蔽 dse 自身头 |
| 库目录 | `libagw_external_msg.so` 等 | 该 .so 内已含 43 个 FPGA 符号，与配套头匹配 |

错误数收敛：**345 → 1 → 10**（最后 10 处均为 `PktCashAuctionTER` 新字段）。

### 三条踩坑教训

1. `dse_client.h` 用**裸 include**（`#include "agw_client.h"`），只把配套头放进嵌套目录无效，必须同时在覆盖目录根层放一份。
2. 整目录引入 ATP 资源包的 `common/include/util/include` 会让 ATP 的 `biz_param.h` 遮蔽 dse 自己的 `md/include/biz_param.h`（同名不同 `namespace dse`），必须改为**按文件名逐个覆盖**。
3. `DSE_PREBUILT_LIB_DIR` 必须在 CMake 使用点**之前** set，否则 `EXISTS()` 判断为假、覆盖目录静默不生效。

## 4. 未解决与下一步（按优先级）

1. 首选：向分支维护者索取与 `agw_external.xml` 配套的生成物（codec 头 + ATP 报文头 + 库）。
2. 其次：在 GLIBC ≥ 2.34 环境用 `buildmsg/Jamfile.v2` 的 `msgproto_ex` 规则重生成 codec。
3. 最后：从 gt_svn 整体回移 codec 与 biz_msg——但 `messages_agw_external.h` 已有 6332 行既有差异，需完整回归，不宜作为第一手段。
4. 若继续推进，先确认 `PktCashAuctionTER` 的 `skip_cal`/`stk_effect_type*` 应由哪个头提供（当前三份副本都没有），这是 10 处残留错误的唯一来源。

## 5. 本轮其它成果（上下文，细节见 DSE 全链路文档）

- gt_svn：dse 与 ute 编译 0 error；上海功能 14 用例全通过、TPS 989.26，深圳 8 用例全通过、TPS 1000.10，均发送失败 0。
- gt_sit：FTE 去 `NO_DSE` 编译成功（171,698,280 字节，0 error），七项环境适配 + 头覆盖完成；功能/性能测试因依赖 dse 未执行。
- 附带修复：验证脚本"计数受历史日志干扰"缺陷，改为增量统计，已回同步 gt_trunk。

## 6. 交付物

- `svn_work/57_DSE联调问题与解决方案全记录_20261107.md`（gt_trunk，411 行，21 个问题）
- `svn_work/57_gt_svn_DSE联调与测试总结_20261107.md`
- `svn_work/57_gt_sit_DSE联调与测试总结_20261107.md`（§6 为下一步建议）
- 提交：gt_trunk `84f7c7e5e`、gt_svn `07586c74c`、gt_sit `8ef7e56dd`，三仓与上游均 0/0

相关文档：`db5c633e-d03f-4e9f-bca9-f0a0dbac21a4`（DSE × DomainServer × FTE 全链路联调打通，2026-11-07，覆盖 gt_trunk 侧的 SIOF、dse_deps、启动顺序与双市场测试）。
