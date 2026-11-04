---
id: "fb29bce3-6d46-42df-9c1e-ffa3e690acc0"
title: "gt_svn 本地复现失败根因：被测树不是 r87083（整树对齐/重新 configure/客户端同源）"
description: "gt_svn 本地全不过、远程全过的真正原因：本地树是分叉的混合快照而非 r87083（149 文件校验：31 版本不同 + 2 缺失）。含三个必要动作（整树对齐、删 CMakeCache 重新 configure、客户端同源重编）、undefined reference 判读、假成功机制、以及性能 34→8.07µs 的防污染配置缺口。"
status: "active"
created_at: "2026-11-04T15:21:34.671Z"
updated_at: "2026-11-04T15:21:34.671Z"
content_hash: "74d3498f9fb8d04db29eaf5779230b377466d835cf2b19a8ee8e314dd6a300c6"
source_paths:
  - "svn_work/47_本地gt_svn树校验收紧_根因与对齐_20261104.md"
  - "study/knowledge_base_system/14_线上代码复测与问题定位.md"
session_ids:
  - "af12362a-b805-42d1-aa07-4c729bb6d223"
memory_body_ids:
  []
---

# gt_svn 本地复现失败的真正根因：被测代码树不是 r87083（2026-11-04）

本文为 gt_svn 本地复测系列的收尾文档，**推翻并替代**前序文档中"改报文号 / 给新代 API 加 `process_etf_order` / 改路由分支"一带的排障方向。
前序基线（仍有效的环境与工具链细节）：
- `ac5aee7f-793f-41d9-a598-0e8950cccb73`（登录/委托链路排障基线）
- `9e861d20-7da8-4980-a9c5-34e5fcbae404`（ETF 能力缺失与性能 341 上限，含已排除假设）
- `b2d3cf84-f7c9-4ab9-af5b-36af0dd745ea`（341 根因：FTE 停读交易所 socket；ETF 分支存在）

对照环境 `gt_trunk`（= r87083 镜像，同机同 docker 同脚本），问题树 `/mnt/work/gt_svn`，运行根 `/mnt/work/gt_test_svn/run`。

## 1. 决定性结论

**本地"线上代码"根本不是 r87083**，而是一棵**分叉的混合快照**。因此"本地全不过、远程全过"不是测试方法或环境问题，是被测对象拿错了。

用 `verify_baseline.py` 对 149 个关键文件做基线校验（约 5 秒出结果）：

| 代码树 | 一致 | 仅换行差异 | 版本不同 | 缺失 | 判定 |
|---|---|---|---|---|---|
| `gt_trunk` | 138 | 3 | 8（均为本地补丁） | 0 | ≈ r87083 ✅ |
| `gt_svn`（本地） | 107 | — | 31 | 2 | ≠ r87083 ❌ |

**关键缺口**（行数差即可看出缺失功能）：

| 文件 | 本地 | 基线 | 后果 |
|---|---|---|---|
| `api/src/ute_client_handler_impl.cpp` | 2216 | 3506 | **整条 ETF 发送路径不在** |
| `fte/src/business/ccu_trade_impl.cpp` | 141 | 214 | ETF 分发分支不在 |
| `fte/src/business/ins_handler.cpp` | 3254 | 3895 | 少 641 行 |
| `fte/src/tech/float_helper.cpp` | **缺失** | 190 | 文件不存在 |

反向证据说明"不是整体旧一版"：本地 codec 头文件**比基线更大**（49311 vs 38451），即某些模块更新、某些缺失 → 分叉快照，靠补个别文件永远补不齐。

## 2. A/B 对照实验（决定性验证）

同一机器、同一 docker、同一套脚本，仅替换被测代码树：

- `gt_trunk`（= r87083）→ 深市 8/8、沪市 12/13，与远程报告逐项一致。

## 3. 让 gt_svn 跑通的三个必要动作（缺一不可）

| # | 动作 | 不做的后果 |
|---|---|---|
| 1 | **整树对齐 r87083**（fte / api / gw_external_msg / Common / tech / msg_parser） | 缺 ETF 整条路径 |
| 2 | **删 CMakeCache 后重新 configure** | 旧源文件列表让新文件不参与编译 → 大量 `undefined reference` |
| 3 | **客户端与 FTE 同源重编** | 只重编 `ute`、沿用旧 `libagw_ute_api.so`，ETF 仍发不出去 |

### 3.1 动作 1 的实况：链接期才暴露的缺件

整树同步后先出现的是一批 `undefined reference`，全部指向"新消息类/新 manager"未编译进来：

```
data_manager::BusinessRightsRuleManager::Instance() / Init()
share_common::LockFileManager::Init(char const*, std::function<...>, int)
message::PktExtFundRightSet::decode/dump
message::PktExtETFModifyPDBrokerConfigIns::decode/dump
message::PktRTControlInstructionGateWayInfo::dump
message::PktRTControlInstructionGwWeightingInfo::dump
```

判读：缺的是 `gw_external_msg` 一类新消息实现（同步时漏项），而不是链接参数问题。
补齐后仍需动作 2，否则 CMake 用旧的源文件列表继续报同一批 undefined reference。

### 3.2 动作 3 是最隐蔽的坑

```
FTE 已对齐、有 ETF 处理              ✓
客户端库 libagw_ute_api.so 仍旧、无 ETF 实现  ✗
  → SendMessage(ETF) 落到 default 分支，只打一句 warning 就返回 kSuccess
  → 客户端以为发送成功，FTE 一条报文都收不到
  → 现象：等待回报超时
```

**教训**：跨树复用已编译二进制的唯一合法前提是"源码已完全一致"。本次两棵树源码对齐后，直接复用 `gt_trunk` 已编译的客户端（`libagw_ute_api.so`、`mock_client`）即通过；否则要等 8.5MB 大文件重编十余分钟。

## 4. 最终结果（gt_svn 本地达标）

| 市场 | 套件 | 通过/总数 |
|---|---|---|
| 深圳 | fte_combo / etf_combo | 4/4 + 4/4 = **8/8** ✅ |
| 上海 | fte_combo / etf_combo / fill | 4/5 + 4/4 + 4/4 = **12/13** ✅ |

唯一失败：「FTE 撤单测试」——用例间回报串扰，**远程同样失败**，非 FTE 缺陷。
沪市曾出现"期望值口径不同"（`ord_type`/`time_in_force` 期望非空而实际为空、`cum_qty`/`leaves_qty` 期望差异），解决方式是把已验证通过的 `gt_test` 配置整体同步过来，而不是逐个改期望值。

## 5. 性能：先怀疑防污染配置

首轮 P50 34.36 µs（远程 10.44、gt_trunk 基线 7.99）。根因：`gt_test_svn` 缺防污染配置——
mock 未绑核、`start_all.sh` 无 `taskset`、`<reader_cpu>/<biz_cpu>/<writer_cpu>` 等绑核项缺失。
从 `gt_test` 补 `mock_exch_sz.xml`、`mock_exch_server`、`mock_sqlserver`、`start_all.sh` 后复测：

| 指标 | 补前 | 补后 | 远程 |
|---|---|---|---|
| 出入口包 | 10000/10000 | 10000/10000 | 6384/5461 |
| P50 | 34.36 µs | **8.07 µs** | 10.44 µs |
| P90 | 42.87 µs | **8.89 µs** | 10.59 µs |

结论：性能达标，优于远程记录，与 gt_trunk 基线持平。
**但** outlier 3591 个（约 36%）超过自定 5% 阈值 → 按协议该轮 P50 不宜对外引用，需同配置连跑 3 轮取稳定值。

## 6. 可复用教训

1. **开测第一步就跑基线校验**——5 秒能省掉几天弯路。
2. **"缺失文件"比"版本不同"更致命**；行数差是最快的功能缺口信号。
3. **树可能分叉而非单调旧版**，不能用"补几个文件"解决。
4. **对齐必须整树做**，.cpp 与 .h 必须同源。
5. **本地全不过时，先怀疑被测对象**，别急着改测试方法、报文号或路由分支。
6. **客户端与 FTE 必须同源**；复用旧二进制的输出是"假成功"（返回 kSuccess 但报文没发）。
7. **整树同步后要重新 configure**，否则旧源文件列表伪装成"链接错误"。
8. **性能异常先查防污染配置**（mock 绑核、taskset），再怀疑代码。

## 7. 收尾与建议

- 汇报文档：`svn_work/47_本地gt_svn树校验收紧_根因与对齐_20261104.md`（10 节）
- 知识库补充：`study/knowledge_base_system/14_线上代码复测与问题定位.md` §12.7/12.8
- 提交：`ac9219822`（定位根因）、`a9618aa4e`（本地 100% 复现）、`acdac9c20`（性能复测）
- 源码改动前备份：`/tmp/gt_svn_src_backup_2142.tgz`
- **建议**：从 SVN 重新拉一份干净的 r87083 工作副本，替代这棵分叉树，避免后续反复踩同一坑。
