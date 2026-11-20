---
id: "a4d51da0-5efc-41af-8a0f-2e44f521b13f"
title: "容器内重建 mock_client（GLIBCXX 3.4.19 上限 + boost 抢占 + sendlogin P0-1）与 ETF policy_id 端到端实测（2026-11-20）"
description: "gt_trunk 在容器 otc 内把 libmsg_proto/libagw_ute_api/mock_client 降到容器 GLIBCXX 3.4.19 以内可用的完整路径：ABI 自检、BOOST_ROOT 抢占坑、sendlogin() 注释态 P0、三组件联调栈起停、policy_id 观测口径（需 Debug 级），以及 build_in_container.sh 脚本固化与 15/15 回归结论。"
status: "active"
created_at: "2026-11-20T04:26:55.933Z"
updated_at: "2026-11-20T04:26:55.933Z"
content_hash: "763f5c4cd3bc2ff68d20e3d0f713e68787d337f158f1d64f3d2e07a4affbc804"
source_paths:
  - "DYS-FRAMEWORK/api/gw_api_boost/ute_mock_client/build_in_container.sh"
  - "DYS-FRAMEWORK/api/gw_api_boost/api/src/ute_client_handler_impl.cpp"
  - "DYS-FRAMEWORK/api/gw_api_boost/ute_api"
  - "DYS-FRAMEWORK/msg_parser"
  - "run_trunk/build_mock_client.sh"
  - "run_trunk/start_dse_stack.sh"
  - "svn_work/129_policy_id功能测试文档_20261116.md"
  - "work_log/20261119.md"
session_ids:
  - "fe90c7b1-76db-46c2-b826-d8ac383f59de"
memory_body_ids:
  []
---

# 容器内重建 mock_client 并端到端验证 ETF policy_id（2026-11-20）

**一句话**：官方 `run_trunk/build_mock_client.sh` 依赖分支自带的 `work_atp/dse_deps` 库，但在容器 `otc` 里旧库与新建二进制都有 GLIBCXX 版本墙；本次在容器内**从源码重建** `libmsg_proto.so` / `libagw_ute_api.so` / `mock_client`，把它们压到容器上限 `GLIBCXX_3.4.19` 以内，随后拉起 domain_server → dse → fte 三组件栈，实测 ETF 通道 `policy_id=1234`、现货 `policy_id=4321` 互不串味，深圳 15/15 PASS。

相关既有文档：`f3324711-b8f7-477e-94c5-3d06a1ee3d6d`（gt_sit `build_mock_client.sh` 的 boost 1.68/1.62 混链 double free 根因，2026-11-08）、`98683cb9-f567-47ff-b952-2ebc8a769a36`（`test/share_test` 侧 policy_id 与业务体偏移陷阱）。本文只记**容器内重建 + 端到端实跑**这条路，不重复上述两篇。

## 1. ABI 约束（容器 `otc` 的现实）

| 库 | 需求 GLIBCXX 上限 | 重建后 | 结论 |
|---|---|---|---|
| `libmsg_proto.so` | 3.4.14 | ✅（原 3.4.29 降下来） | 需重建 |
| `libagw_ute_api.so` | 3.4.19 | ✅ 恰好压线 | 需重建 |
| `libagw_external_msg.so` | 3.4.14 | — | **无需重建**（原库就够） |
| `mock_client` | 依赖上述 | 3,076,704 B | 需重建 |

容器硬上限就是 **3.4.19**，超过即 `GLIBCXX_3.4.2x not found`。重建后 `ldd` 自检应无 `not found`。

**顺带纠正一个既有印象**：`work_atp/dse_deps/libmsg_proto.so` 本来就是 3.4.14 —— 说明重建不是唯一可行路线，官方脚本那条路径在 ABI 上原本就通；本次重建版本同样可用。

## 2. 两个必踩的坑（都已固化进脚本）

### 2.1 boost 头文件被系统路径抢占

不显式指定 `BOOST_ROOT` 时，编译期会命中系统的 `/usr/local/include/boost`，与链接的 1.62 `stage/lib` 版本不一致 → 典型表现为 `boost::system::generic_category_ncx()` 未定义。

修法：`BOOST_ROOT=/mnt/work/gt_test/work_lib/3rd/boost_1_62_0`，并让 CMake 打印出生效值确认：

```
-- Boost include: /mnt/work/gt_test/work_lib/3rd/boost_1_62_0
-- Boost lib dir: /mnt/work/gt_test/work_lib/3rd/boost_1_62_0/stage/lib
```

> 与 `f3324711` 里 gt_sit 的混链坑同源不同面：那次是**链接期** `-L` 写错静默回退 1.68，这次是**编译期** `-I` 被系统头抢占。两者共用的判据都是「编译日志里的 boost 路径必须是 1.62，ldd 里 1.68 计数为 0」。

### 2.2 `sendlogin()` 仍是注释态（P0-1，源码未修）

`DYS-FRAMEWORK/api/gw_api_boost/api/src/ute_client_handler_impl.cpp:140` 当前是：

```cpp
// sendlogin();
```

直连模式（`ute_switch=1`）下 `sendlogin()` 是**唯一**发 1001（`kPktLoginReq`）的地方（另一个 1001 发送者在网关注册分支 `process_cust_logon_for_gateway`；`sendlogin_from_800006` 目前无调用点）。不启用的后果链：

TCP 建链成功 → 上报 `kUTEConnected` → 但 UTE 收不到登录请求、不回 `LogOnAns` → `status_` 停在 `kStatusConnected`、`direct_login_phase_` 停在 `kTcpConnected` → 之后任何委托/撤单在 `AgwClientImpl::SendMessage` 直连分支落到 `set_cust_info`（只认 800005）→ 类型转换失败 → 返回 `kSetCustInfoError`，报文根本不下发。

客户端表面症状就是：`等待连接/登录超时`（`mock_client.cpp:207`），`初始化失败, 错误码=-3`。启用后立即变为 `TCP 连接建立成功, UTE 客户登录成功`。

留档补丁在 `ute_client_handler_impl.cpp.bak_presendlogin_20261113`（1,422 B），内容就是恢复该行 + 上述因果注释。

## 3. 三组件联调栈（起停口径）

`run_trunk/start_dse_stack.sh <sz|sh> [stop]`，按 domain_server → 等 10s → dse → 等 10s → fte 顺序拉起。要点：

- 运行目录按分支独立：`${WORK_ROOT}/run_${FTE_BRANCH}`（`-trunk/-svn/-sit` 入参或路径推导），**绝不回落到三仓共用 `run/`**
- 前置：`echo 140 > /proc/sys/vm/nr_hugepages`
- 实例名必须与 domain_server 已导入的 Context 一致：sh → `UTE_61_611_11`，sz → `UTE_84_842_21`；`DS_NAME=DOMAINSERVER_1_11_11`
- 连通判据：33002（客户端连接口）监听；shm 出现 `dse_ute`/`ute_dse` 及 `*.lock`；`DSE attach ute_dse` 与 `FTE attach dse_ute` 计数同向增长
- 收尾：`... stop` 后必须核对 **残留 shm = 0、无存活残留进程**，再把 `nr_hugepages` 还原为 0
- `FTE_LOG_LEVEL=Debug` 仅排查字段透传时临时开（每笔委托都打印 InternalOrder dump，压测量级暴涨，勿常开）

## 4. 观测口径（本次最容易被误判为失败的点）

- `policy_id` 的观测点是 **DEBUG 级**日志（`uplink_biz_processor.cpp DealTradeETFOrderReqBusi` 的 `internal_order` dump）。Info 级下搜不到，**不要据此判定字段没透传**——必须 `FTE_LOG_LEVEL=Debug` 重启 FTE 再看。
- ETF 分支的专属证据是 `DealTradeETFOrderReqBusi`；现货走另一分支，因此「同一字段名在两个分支各自命中不同值」才是真正的隔离证据。
- 对照组设计：同一套用例分别注入 `policy_id=1234`（ETF）与 `policy_id=4321`（现货），要求
  - 未配置该字段的用例全部 `policy_id[0]`（证明**服务端不推导**）
  - 1234 只出现在 ETF 委托、4321 只出现在现货委托（证明**不串味**）
- 生成对照用例时注意容器内 python 的编码：直接 `python -c` 传中文会 `UnicodeEncodeError`（ascii/surrogates），改用写临时 py 文件或 heredoc 处理。

## 5. 固化产物：`build_in_container.sh`

新增 `DYS-FRAMEWORK/api/gw_api_boost/ute_mock_client/build_in_container.sh`（提交 `87339eeb4`），一脚本完成：

1. cmake + make 编 `libagw_ute_api.so`（`LIBRARY_PATH` 指向 `dse_deps`）
2. 逐文件 `c++ -std=c++11 -O2 -DAGW_UTE_API -DNDEBUG` 编 mock_client 的 obj，显式 include `api/include`、`api/include/agw`、`msg_parser/include`、`gw_external_msg/V2/external/include`
3. 内置 ABI 上限自检（对比 3.4.19）、`ldd` 无 `not found` 自检
4. `--with-sendlogin` 临时启用 `sendlogin()` 并**在结束时自动还原源码**（实测：脚本跑完 `git 改动=0`、无 `.bak_build` 残留、第 140 行仍为注释态）
5. `--deploy` 部署到 `$RUN_DIR/ute_mock_client`

脚本在 `ute_mock_client/` 下，`REPO` 需上溯 4 级到仓库根（实测推导为 `/mnt/work/gt_trunk`）。

**注意**：`run_trunk/ute_mock_client/` 部署目录里原本已有兼容库（`libagw_ute_api` 3.4.19 / `libmsg_proto` 3.4.11 / `libagw_external_msg` 3.4.11）与旧版 5,733,616 B 的 mock_client；覆盖前先备份（本次备份在 `/tmp/mc_deploy_bak`）。

## 6. 实测结果

| 项 | 结果 |
|---|---|
| 单用例 `FTE_ETF申购委托回报_深圳_policy_id透传` | PASS（50ms），FTE 侧 `policy_id[1234]` |
| 现货对照（4321） | PASS，FTE 侧 `policy_id[4321]` |
| 隔离性 | 无串味；未配置用例全为 `policy_id[0]` |
| 脚本产物复跑 | 同样 PASS（证明脚本可用，不是手工构建的偶然结果） |
| 深圳全量回归（15 个用例） | **15/15 PASS，0 FAIL** |
| 环境收尾 | shm 残留 0、hugepages 0、`sendlogin` 已还原、三仓 0 0 |

## 7. 未决事项

`sendlogin()` 只做了「构建时临时启用 + 自动还原」，**未改仓库源码**（沿用此前「先不提交」的约定）。代价是任何人重新编译 boost 树，得到的 mock_client 都登录不上。正式修复需改 `ute_client_handler_impl.cpp`（SVN 管理），提交清单与提交信息待整理。
