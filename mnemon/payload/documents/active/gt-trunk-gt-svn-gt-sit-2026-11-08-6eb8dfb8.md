---
id: "6eb8dfb8-9fb2-4eb2-84af-99641ec7da22"
title: "gt_trunk/gt_svn/gt_sit 三仓的测试资产与上场数据管理方案（2026-11-08）"
description: "三仓 schema 与测试资产盘点（atp_te.xml、agw_external.xml、mock_client/mock_exch_server、上场数据、用例、Row struct）；baseline+overlay 分层存储建议；结构变化 vs 语义变化（行宽校验挡不住 NUL/空格填充差异）与五级防护、8/7 环节影响矩阵；libutedatainit.so 单变量定位法；bin 格式与 bin_tool.py。"
status: "active"
created_at: "2026-11-08T02:58:31.042Z"
updated_at: "2026-11-08T02:58:31.042Z"
content_hash: "5e1b14633acf2762234b4fc0cb7267930950147d49655c1ab87c5d3c51c95ad6"
source_paths:
  - "svn_work/59_多分支代码与测试案例管理方案_20261108.md"
  - "DYS-FRAMEWORK/Common/dev_resource/atp_te.xml"
  - "DYS-FRAMEWORK/gw_external_msg/V2/buildmsg/agw_external.xml"
  - "DYS-FRAMEWORK/fte/ftedatainit/src/ftedata_init.cpp"
  - "DYS-FRAMEWORK/fte/ftedatainit/include/ftedatainit.h"
  - "DYS-FRAMEWORK/fte/include/common/uteloaddata.h"
  - "DYS-FRAMEWORK/tech/include/load_data.h"
  - "DYS-FRAMEWORK/fte/test_all/bin_tool/bin_tool.py"
  - "DYS-FRAMEWORK/api/gw_api_boost/ute_mock_client"
  - "test/mock_exch_server"
session_ids:
  - "37e4ffb9-9455-445d-85de-2a41e6a3d4d4"
memory_body_ids:
  []
---

# 三仓（gt_trunk / gt_svn / gt_sit）的代码、测试资产与上场数据结构管理方案

**一句话**：三个分支不是"同一份代码的三个副本"，而是三套 **schema + 生成物 + 测试资产** 的快照；测试资产按"分层 baseline + overlay"管理，数据变更的风险主体不是结构变化（行宽有校验）而是**语义变化（填充/编码/单位，零防护）**。

> 交付物：三仓各一份 `svn_work/59_多分支代码与测试案例管理方案_20261108.md`（464 行 / 10 章 + 附录，三仓字节一致 24586 / md5 `faed98c7`；提交 gt_trunk `3cfcb23c3`、gt_svn `5ccf752af`、gt_sit `836332ee3`）。本文是该文档的知识摘要。

## 1. 分支职责与现状盘点（实测）

| 分支 | 角色 |
|---|---|
| gt_trunk（trunk） | 本地开发 + 线上测试 |
| gt_svn（svn_remote） | 线上测试分支（本地开发完成 → svn 提交 → 充分测试） |
| gt_sit（svn_sit） | 线上生产 |

**两个贯穿三仓的固定数据源**（本次差异的根源，实测 md5）：

| 文件 | 作用 | gt_trunk | gt_svn | gt_sit |
|---|---|---|---|---|
| `DYS-FRAMEWORK/Common/dev_resource/atp_te.xml` | 上场数据类型定义 | 19,711 行 `09169caa` | 同左 | **19,261 行 `f7422f5b`** |
| `DYS-FRAMEWORK/gw_external_msg/V2/buildmsg/agw_external.xml` | AGW 报文 IDL | 8,744 行 `c35a1feb` | 同左 | **8,546 行 `b8cd4363`** |

trunk 与 svn 完全一致；**sit 明显落后 450 / 198 行** —— 这是 gt_sit 缺 FPGA 报文、缺新数据字段的直接原因（编译侧表现见关联文档 `5ae98d7d-73b9-4432-b1d8-1f1a28b52256`）。

**测试资产归属**：

| 资产 | gt_trunk | gt_svn | gt_sit |
|---|---|---|---|
| mock_client 源码（`DYS-FRAMEWORK/api/gw_api_boost/ute_mock_client`） | 有 | 有 | **无** |
| mock_exch_server 源码（`test/mock_exch_server`） | 有 | 有 | **无** |
| 上场数据（`DYS-FRAMEWORK/fte/test_all/etf_test_sz`） | 13 bin + 47 xml | 同左 | **0** |
| 测试用例（`config/test_cases`） | 6 json | 6 json | **0** |
| `ftedatainit.h` Row 结构体 | 4 | 4 | **3**（缺 `CreditDebtsStkLevelRow`） |

结论：**gt_sit 没有测试资产**，以往"gt_sit 测试"实际是借用 gt_svn 的运行目录，掩盖了分支差异。

## 2. 三个问题的结论

### 2.1 测试客户端 / 模拟交易所代码会随组件变化吗

| 组件 | 变化根因 | 幅度 |
|---|---|---|
| mock_client | 强依赖 `agw_external.xml` 的生成物（`libagw_external_msg` / `libmsg_proto`、`libagw_ute_api`），**IDL 变必须重编** | 大 |
| mock_exch_server | 依赖交易所协议（较稳定），但接入参数（端口/pbu/setid/offer_way）随 FTE 配置变化 | 中 |

实测 trunk vs svn：`ute_mock_client/src/main.cpp` 差 13 行，另有 `test_config_sh_fill_perf.json` 与 `build_svn`、`mock_client_docker` 等构建/运行差异；`mock_exch_server/src/session.cpp` 与 `cfg/mock_exch*.xml` 有差异。测试用例 json 三仓间（trunk vs svn）**完全相同**。

### 2.2 是否每分支单独存储

推荐 **分层（baseline + overlay）**，反对两个极端：

- 全量复制：维护量三倍，正是当前困境；
- 单点共享：掩盖差异，**已经出过事**（gt_sit 借用 gt_svn 运行目录）；
- 分层：通用资产（用例模板、模拟器 cfg 骨架）走 baseline，分支特有部分（schema、codec 生成物、上场数据、端口/账号）走 overlay，差异显式可审计。

关键：**codec 生成物与 schema 必须各分支独立**，不可跨分支复用二进制。

### 2.3 数据结构变化的影响面（核心）

| 类型 | 行宽变化 | 现有校验 | 表现 |
|---|---|---|---|
| **结构变化**（字段增删/宽度） | 会 | ✅ 能挡 | 数据"凭空消失" |
| **语义变化**（填充/编码/单位/枚举） | **不会** | ❌ **零防护** | 加载成功但业务报错 |

校验在 `ForeachBinaryRow`（`DYS-FRAMEWORK/tech/include/load_data.h`、`DYS-FRAMEWORK/fte/include/common/uteloaddata.h`）：`context_row_bytes < sizeof(T)` 直接拒绝加载，内容不是行宽整数倍也拒绝 —— 只覆盖结构，不覆盖语义。

实证：gt_sit 的 `ftedatainit/src/ftedata_init.cpp` 缺 3 处键归一化（`LoadFundInfo` 的 `cash_fund` 键、`LoadFundPartAsset` 键、`LoadAccount` 的 `branch_id`），二进制数据用 NUL 填充而 FTE 协议侧用空格填充，行宽未变、校验全过，却造成登录 `No fund_data`、下单 `50019`「委托账户与登录账户不符」，排查近 2 天。

**影响矩阵**：上场数据加字段 → 8 个环节（schema → struct → 装载 → 数据文件 → 业务 → 用例 → 模拟器 → DSE）；AGW 加报文 → 7 个环节（schema → codec 生成 → 库 → 客户端 → 模拟器 → FTE → DSE）。

### 2.4 五级防护（对语义变化）

1. schema 指纹（`atp_te.xml` / `agw_external.xml` 的 md5 + 行数）写入数据头或同目录元数据；
2. 启动自检：比对运行时 schema 指纹与数据文件指纹，不一致报警；
3. `manifest.json`：声明每个数据集的建设日期、来源分支、所需 struct 版本；
4. CI 四项校验：schema 指纹、struct 数量、数据行宽、用例引用完整性；
5. **语义变更清单**（最有价值）：字段的填充字符、编码、单位、枚举含义变更必须人工登记并回归对应业务。

## 3. 定位方法（可复用）

**单变量对照法**：在已知可用的 gt_svn 环境中**只替换 `libutedatainit.so`** 为 gt_sit 构建版本 → 登录立即失败、症状与 gt_sit 完全一致 → 锁定在数据初始化库，一次性排除 ute 二进制、数据文件（md5 相同）、mock_client 二进制与配置（md5 相同）、`domainserver_98.config`（md5 相同）。此前两次误判（"分支版本不齐""二进制问题"）都源于没做这个对照。

## 4. 二进制上场数据格式与工具

```
文件 = [BinHeader 53B] + [Row0][Row1]...[RowN-1]   // #pragma pack(1)
BinHeader: first_row_total_bytes int64(8) | partition uint8(1) | PartitionName char[32]
           | context_row_bytes int64(8) | context_number int32(4)
行数 = (文件大小 - 53) / context_row_bytes
```

`DYS-FRAMEWORK/fte/test_all/bin_tool/bin_tool.py` 提供 info / dump / query / insert / update / delete / create，字段类型从 `DYS-FRAMEWORK/Common/dev_resource/atp_te.xml` 的 `<Types>` 动态采集（`--schema` / `ATP_SCHEMA` 可覆盖），按 `context_row_bytes` 自动识别 struct，无法识别时退化为原始字节模式。其他造数入口：`DYS-FRAMEWORK/test_support/include/atp_data_file_generator.hpp`、`DYS-FRAMEWORK/Common/tools/data_generate`。

## 5. 复查与遗留

文档经 6 轮复查：3 轮零修改（数据 md5/行数/资产数/struct 数逐项核对 0 偏差、章节与表格结构、9 个引用路径全部存在），3 轮改动（补"用例与协议耦合"章节与《数据结构变更自查清单》附录、调整附录顺序）。交付过程中出现过一次 `git add -A` 把用户 GBK 编码任务笔记 `task/task_test/task_test.txt` 混入功能提交，已用后续提交说明，未做 force-push。
