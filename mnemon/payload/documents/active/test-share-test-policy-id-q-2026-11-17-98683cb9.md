---
id: "98683cb9-f567-47ff-b952-2ebc8a769a36"
title: "共享内存测试客户端 test/share_test 升级（policy_id + 方案Q 适配）：结构、四类阻塞、业务体偏移陷阱与同步语义（2026-11-17）"
description: "gt_trunk 工作区 test/share_test 测试套件升级为 v3/方案Q 口径的可复用知识：目录分工、5 个 target 编译修复、policy_id 恒读 0 的 8 字节偏移根因、buffer_len=业务体长度语义、sync_from_fte.sh 预检与快照专属改动、CRLF→LF 决策与 review 用 diff -w、干净重建+端到端验证，以及同日 work_log 合并冲突处理；含提交号与路径。"
status: "active"
created_at: "2026-11-17T03:28:48.070Z"
updated_at: "2026-11-17T03:28:48.070Z"
content_hash: "39991a4c1bcdb0a492019f970bfb0b37c8fcbd1eb9dca99cc06b11356698cc2f"
source_paths:
  - "svn_work/132_共享内存测试客户端升级_20261117.md"
  - "work_log/20261117.md"
  - "test/share_test/CMakeLists.txt"
  - "test/share_test/sync_from_fte.sh"
  - "test/share_test/share_util/json_struct.h"
  - "test/share_test/test_client/strategy_client.cpp"
  - "test/share_test/test_server/ute_server.cpp"
  - "test/share_test/test_data.json"
  - "test/share_test/pack_for_src_upgrade/share_comm_external_message.h"
  - "test/share_test/pack_for_src_upgrade/ute_message_manager.cpp"
  - "test/share_test/pack_for_src/strategy_message_manager.cpp"
session_ids:
  - "9f6c4fda-1188-437b-a9f9-4024eba41c1e"
memory_body_ids:
  []
---

# test/share_test 测试套件升级（policy_id + 方案Q / v3 口径）

**一句话**：`DYS-FRAMEWORK/fte` 侧完成方案Q（`ulStrategyKey` 前置、业务体 116 零拷贝）与 SHM 通道 `policy_id` 之后，仓库内自带的共享内存测试套件 `test/share_test` **无法编译、且按旧口径解读**；本次把它升级到 v3 口径并端到端自证（客户端发 0/1/4321/65535，服务端原样收到）。

产出：`svn_work/132_共享内存测试客户端升级_20261117.md`（10 表 / 286 行，含同步语义与复验说明）、`work_log/20261117.md`；提交 `a3682c8a8`（升级）+ `4d72bf6b8`（合并远程同日日志冲突）。仅走 git：**`test/` 不在 SVN 内**，gt_svn/gt_sit 无该目录，未同步（若要让外部测试人员复测，需手动同步整个 `test/share_test/`）。

前序背景见 [GT 三仓 policy_id 字段贯通修改方案](9edbc756-0ad0-4081-81e8-475f7983f83e) 与 [FTE 共享内存通道方案Q 提交清单](85c276bd-e6a0-4cd2-956a-139ae6487dfa)；本文只记**测试套件侧**的知识。

## 1. 目录分工

| 路径 | 角色 |
|---|---|
| `test_client/` | 被测客户端（策略端），交互式命令循环 `h/q`、`1 登录`、`2 登出`、`3 委托`、`4 撤单`，本次新增 `5/6/7`（`policy_id`=0/1/65535 边界）与 `a`（重打印 ABI 契约） |
| `test_server/` | 模拟 UTE 服务端（本地自测，不连真 FTE） |
| `test_client_by_so/`、`test_server_by_so/` | 同上，但链接 `pack_for_so` 编出的 `libute_share_comm_lib.so` |
| `pack_for_src/`、`pack_for_src_upgrade/` | 共享内存通道源码副本（“脱离 FTE 主体独立编译”的适配快照） |
| `pack_for_so/` | 把副本编成 `.so` |
| `share_util/` | json 解析（`json_struct.h`）、logger、spdlog |
| `test_func/` | 更细的单测，**未加入构建** |

两个易踩的事实：`test_client/strategy_client.cpp` 是 **GB18030** 编码（其余 UTF-8）；target 原先**直接引用 FTE 主体源码目录**，而 FTE 的 `ute_message_manager.h` 依赖 `ute_singleton.h`（实现依赖 `global_obj`），套件里没有。

## 2. 四类阻塞与修法

| # | 阻塞 | 修法 |
|:-:|---|---|
| R1 | 5 个 target 全部 `fatal error: ute_singleton.h` | CMakeLists（顶层 + 5 子工程）改用 `pack_for_src_upgrade` 适配快照，不再引 FTE 主体源码 |
| R2 | `pack_for_src*` 副本停留在 v3 之前（无 `policy_id`、无方案Q） | 新增 `sync_from_fte.sh` 从 FTE 同步 11 个文件 |
| R3 | `share_util/json_struct.h` 不解析/不打印 `policy_id` | `TradeOrderInfo` 增字段解析 + 打印；`test_data.json` 增 `"policy_id":4321` |
| R4 | `test_server` 回调仍是 4 参旧签名（v3 已 5 参，增 `buffer_len`） | 回调加 `buffer_len`，委托按业务体范围校验 |

**教训**：遇到这类“套件编不过 FTE 源码”的问题，**先找现成适配层**（`pack_for_src*` + `sysc_share_comm_src.sh` 就是线索），不要急着往 CMake 里加 include 路径。

## 3. 两个 v3 “后遗症”（本次最贵的坑）

### 3.1 `buffer_len` 是业务体长度，不是结构长度

`kShmOrderReqBody*` 系列常量（Off=8 / Len=116 / Min=108）一律是**业务体口径**；与 `sizeof(TradeOrderReq)`（124）**差 8 字节**。把它当结构长度做校验，会把所有委托拦下。

### 3.2 回调给的是业务体指针，按完整结构强转会静默错位 8 字节

`UteMessageManager` 出队后：

```cpp
const char* order_body = reinterpret_cast<const char*>(&reqOrder) + sizeof(reqOrder.ulStrategyKey); // +8
m_pfnOnMessage(kPktOrderReq, order_body, kShmOrderReqBodyLen, reqOrder.ulStrategyKey, ClockNanoTime());
```

`test_server` 却按 `TradeOrderReq*` 强转并读 `info.policy_id`（绝对 78 偏移）→ 实际读到业务体偏移 70 之后错位 → **`policy_id` 恒为 0**。修正：按业务体布局解析（`user@0`、`info@70`）。

**通用规则**：v3 之后凡是 `(TradeOrderReq*)pMsgBuf` 的写法都要 review —— 要么先 `- sizeof(ulStrategyKey)`，要么直接按业务体字段序列读。**这种错不报错、不拒单，只静默读错**，只能靠“发送值 vs 解析值”对账（`grep code` 计数）才暴露；本次即用该方法定位。

另：`share_util/json_util.hpp` 的 `StdArrayToStr(std::array<char,Size>&)` 是**非 const 引用**，传临时/const 数组编不过 —— 先拷到局部数组再转字符串。

## 4. ABI 自检基线（客户端启动即打印 `PrintAbiContract()`）

```
User=70 Info=38 policy_id@36 Req=124 (key@0 user@8 info@78)
kShmOrderReqBodyOff=8 Len=116 Min=108 ; TradeOrderER=332
```

与 FTE 侧基线逐项一致。改动共享内存结构后**先跑这条自检**，再谈端到端。

## 5. 同步语义（`sync_from_fte.sh`，取代失效的 `update_test.sh`/`update_work.sh`）

脚本行为：从 `DYS-FRAMEWORK/fte/fte/{include,src}/share_communication` 拷 11 个文件到 `pack_for_src_upgrade`，**只做一处适配且显式打印**（`ute_singleton.h` → `singleton.h`），行尾随 FTE 源（LF）；末尾提示需人工跟进的三处（`json_struct.h` 字段同步、`test_data.json` 测试取值、`strategy_client.cpp` 菜单/发包）与编译命令。

**关键新事实：快照不是“FTE 的旧副本”**。它含快照专属本地改动 —— `queue_manager.h/.cpp` 多一个 `uiQueueBlockCount` 入参、`StartConsumerThread` 实现不同、`lock_file_manager.*` 细节不同（忽略行尾后真实差异：`lock_file_manager.cpp` 54 行、`queue_manager.cpp` 92 行等）。首次同步即把这些**静默覆盖**掉了。

因此脚本增加**拷贝前预检**：逐文件内容比较（忽略行尾），凡有差异打印 `[覆盖] <文件> 内容差异 N 行`，例：

```
[覆盖] ute_message_manager.h   内容差异 2 行（快照的本地改动将被 FTE 源码覆盖）
```

本次按 FTE 覆盖后编译 + 端到端均通过，改动无丢失；但**约定：预检告警出现时必须先确认“是真要覆盖”再继续**。

## 6. 行尾：CRLF → LF（review 必须用 `git diff -w`）

旧快照 14 个文件是 CRLF，FTE 源是 LF。直接同步会让 `git diff` 充满行尾噪声：`ute_message_manager.cpp` 原始 diff **457 行**，忽略行尾后真实差异仅 **20 行**。

决定**保持 LF**：①与 FTE 源一致，今后同步不再翻行尾；②仓库 `.gitattributes` 就是 `* text=auto eol=lf`；③脚本已打印“行尾随 FTE 源，为 LF”。代价是本次提交含一次性行尾变更 —— **审阅请用 `git diff -w`**。

## 7. 验证基线（复现步骤）

1. 干净目录从零重编：`cmake -S test/share_test -B /tmp/st_build && make -C /tmp/st_build -j4` → 0 error，5 个 target 全 Built（升级前全失败）：`ute_share_comm_lib`、`client_strategy`、`server_ute`、`so_client_strategy`、`so_server_ute`。
2. ABI 自检（§4）。
3. 端到端：客户端发 `0 / 1 / 4321 / 65535`，服务端 `policy_id` 全部原样收到。
4. 复查（末轮零修改）：表格列数一致、文档引用路径存在、`test/share_test` 各文件路径与所述一致、快照两份副本差异仅预期项、`help` 文本与 `switch` case 一一对应（`d` 是 “unused” 等英文冒号的误匹配）。

## 8. 同日 work_log 冲突（可复用处置）

远程也在同一天新建了 `work_log/20261117.md` → rebase/pull 出现 **add/add 冲突**。处置：保留**双方**章节（本次任务 + 远程“49 上重建 FTE”“复测全过程总结”），删净冲突标记，校验双方关键小节都在，再提交 `merge:` 说明。

注意：`git` 的 `<<<<<<< ours/theirs` 语义在 rebase 与 merge 下相反，**先确认方向再拼接**（本次第一版就取错了 theirs 段，需 `git checkout --conflict` 重做）。

## 9. 交付清单

- `test/share_test/sync_from_fte.sh`、`pack_for_src_upgrade/sync_from_fte.sh`（新增）
- `CMakeLists.txt`（顶层）+ `test_client/`、`test_client_by_so/`、`test_server/`、`test_server_by_so/`、`pack_for_so/` 的 CMakeLists
- `share_util/json_struct.h`、`test_data.json`、`test_client/strategy_client.cpp`、`test_server/ute_server.cpp`
- `pack_for_src*` 11 个文件按 FTE 重同步（含 CRLF→LF）
