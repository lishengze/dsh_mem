---
id: "a8dbc49f-635b-4755-8136-5c257da8841d"
title: "gt_sit dse 残留 10 处 TER 字段编译错误的更正根因：include 路径移除了 ATP common/include（未验证）"
description: "gt_sit dse 最后 10 处 PktCashAuctionTER has no member named 'skip_cal'/'stk_effect_type*' 的更正根因：本地 CMake 适配把 ATP 资源包 common/include 整目录移出 include 路径，连带丢失新版 biz_msg 报文头；ATP 根层无 biz_param.h，故只加回 common/include 即可同时解决遮蔽；含证据链与待实测修复步骤。"
status: "active"
created_at: "2026-11-07T14:03:17.678Z"
updated_at: "2026-11-07T14:03:17.678Z"
content_hash: "931bec5d04e5f4905a86134f166a02907fadb348de4f34dc81f2382540ac8e0a"
source_paths:
  - "DYS-FRAMEWORK/fte/dse/CMakeLists.txt"
  - "svn_work/57_gt_sit_DSE联调与测试总结_20261107.md"
  - "svn_work/58_DSE与DomainServer联调总总结_20261107.md"
session_ids:
  - "b17ccc11-cb24-4041-ba39-a4d1e92c217a"
memory_body_ids:
  []
---

# gt_sit dse 残留 10 处编译错误：更正后的根因（未验证）

**一句话**：gt_sit dse 最后 10 处 `PktCashAuctionTER has no member named 'skip_cal' / 'stk_effect_type*'` 并非"分支内生成物陈旧、无法修复"，而是本地 CMake include 适配的副作用——为了让 ATP 的 `util/include/biz_param.h` 不再遮蔽 dse 自己的 `md/include/biz_param.h`，把 ATP 资源包的 `common/include` **整目录**移出了 include 路径，连带丢失了其中的新版 `biz_msg` 报文头。

> 状态：**证据充分、修复方案尚未实测**。上一轮已停机（docker 容器 otc 已停），未启动验证。

## 1. 与既有文档的冲突

- 既有文档：`5ae98d7d-73b9-4432-b1d8-1f1a28b52256`（gt_sit dse 编译阻塞：codec/ATP 生成物陈旧与 msg_proto GLIBC 约束）与其对应的 `svn_work/57_gt_sit_DSE联调与测试总结_20261107.md` 把残留错误归因为"分支自身 `agw_external.xml` 有新报文但生成物陈旧，需向分支维护者索取配套产物"。
- 该结论对**前 345→10 的收敛过程**成立，但对**最后 10 处 TER 字段错误**不成立：ATP 资源包内确实存在含 `skip_cal` 的新版报文头，是本地适配把它移出了搜索路径。

## 2. 证据链

| 环节 | 证据 |
|---|---|
| 报错 | `data_sync_engine.cpp` 约 2390 行起连续报 10 处 `class abf::util::PktCashAuctionTER has no member named 'skip_cal' / stk_effect_type1/3/4/5/10 / fund_unfrzn` |
| 该类有两份定义 | ATP 包 `work_atp/cmake/common/include/biz_msg/include/messages_biz_msg.h`（261,766 B，`skip_cal` 命中 3 次）vs 仓库 `DYS-FRAMEWORK/Common/biz_msg/include/messages_biz_msg.h`（类体 55 行，无这些字段） |
| include 顺序差异（决定性） | gt_svn（编译通过）第 1–2 位是 `work_atp/cmake/common/include` 与 `.../common/include/util/include`；gt_sit 前两位是 `dse_deps/agw_codec_include` 与 `dse_deps/atp_override`，**ATP `common/include` 已不在列表中** |
| 遮蔽源定位 | `BizParam has not been declared` 来自 ATP 的 `common/include/util/include/biz_param.h` 遮蔽 dse 的 `fte/dse/md/include/biz_param.h` |
| 关键事实 | ATP `common/include` 顶层只有 `biz_msg/` 和 `util/` 两个子目录，**根层没有 `biz_param.h`** |

## 3. 拟采用的修复（待实测）

```cmake
# 只加回 common/include，不加 util/include：
#   biz_msg 报文头命中 ATP 新版（含 skip_cal 等字段）
#   而 ATP 该路径下无 biz_param.h，不会遮蔽 dse 自身同名头
include_directories(${ATP_RES_ROOT}/common/include)
```

预期两个问题同时消除：`biz_param.h` → `fte/dse/md/include/biz_param.h`（`BizParam` 可解析）；`messages_biz_msg.h` → ATP 新版（字段齐全）。

**验证步骤**：启动 docker `otc` → 清 `build_dse_sit` → 重编 dse → 确认 0 error，再补跑 gt_sit 双市场功能/性能；若失败，回退到按文件名逐个覆盖（`dse_deps` 精确头覆盖思路）。

## 4. 方法论教训

1. 用"整目录移除"解决单点头文件遮蔽，会连带丢掉该目录下其它需要的头；应改为**保留目录、只按文件名覆盖冲突项**。
2. 下"分支生成物陈旧、无法修复"这类阻塞结论前，先逐项比对**两份同名头的内容与 include 搜索顺序**，确认不是自己适配造成。
3. 已发布的分支总结文档若要改口径，需另立勘误记录并互相引用，不能只改结论。

## 相关

- `5ae98d7d-73b9-4432-b1d8-1f1a28b52256`：gt_sit codec / msg_proto GLIBC 约束（前段根因，仍然有效）
- `d2d8435f-21da-4ebb-a207-25a1799cd569`：三仓构建套件归属与最小构建法
- 三仓总总结：`svn_work/58_DSE与DomainServer联调总总结_20261107.md`
