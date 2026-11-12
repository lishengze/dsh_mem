---
id: "53eb4bc5-3b14-4695-a999-a43d9baf6dc7"
title: "NewAPI 委托报文体长修复（policy_id 106→108B）与 git/SVN 双轨提交处置（2026-11-12）"
description: "FTE 拒单 \"order req length 106 too short\" 的根因与修复、验证链路，以及同一批改动在 git（origin master）与 SVN（r24~r30 残留 2 文件）双轨提交时的分流与踩坑记录。"
status: "active"
created_at: "2026-11-12T04:34:03.643Z"
updated_at: "2026-11-12T04:34:03.643Z"
content_hash: "01b8f4fb0fbf0d89fc1762824539cab94382dcd0c3313a7f45172ceb0b78857c"
source_paths:
  - "trunk/NewAPI/gone/api/include/gw_head.h"
  - "trunk/NewAPI/gone/api/src/gw_head.h"
  - "trunk/NewAPI/gone/api/src/gw_counter_direct.cpp"
  - "trunk/NewAPI/gone/api/src/single_socket_engine.cpp"
  - "trunk/NewAPI/gone/api/include/order_trade_type.h"
  - "task/svn/task_svn.txt"
  - "work_log/20261112.md"
  - ".gitignore"
session_ids:
  - "297296d5-7f40-49bb-b1af-13db9648489b"
memory_body_ids:
  []
---

# NewAPI 委托报文体长修复与 git/SVN 双轨提交处置（2026-11-12）

工作副本：docker 容器内 `/mnt/work/api_trunk/trunk`（SVN URL `https://jyjs2.svn.gtja.net/svn/NewAPI/trunk`，基线 r23）；
git 远端 `git@github.com:lishengze/server_9.git`（origin/master）。两者跟踪同一源码树，制度不同，见第 3 节。

## 1. 根因：委托请求体少 2 字节被 FTE 丢单

| 项 | 值 |
|---|---|
| FTE 侧期望体长 | `FTE TradeOrderReqLayout = 108B` |
| API 侧实际组包 | 106B（`TradeOrderReq` 缺 `PolicyID_def policy_id`，2B） |
| FTE 侧行为 | `ccu_trade_impl` 判 `order req length 106 too short` 直接丢单，客户端侧只看到超时 |

修法：在 `NewAPI/gone/api/{include,src}/gw_head.h` 的 `TradeOrderReq` 补回 `policy_id` 字段（两处头文件必须同步），
使组包体长对齐 108B。补字段后 GW 用例的委托/成交回报由 FAIL 转 PASS。

**通用教训**：跨进程定长报文出现"静默丢单 + 超时"而无错误码时，先核对双方结构体布局与逻辑体长，而不是先怀疑网络或时序。
该结论也修正了 r21 迁移时"`OrderReq` 已下线、`req.policy_id` 可删"的判断——`policy_id` 在委托报文层仍然必需。

## 2. 同批改动与验证

同批内容（共 21 个已版本化文件，SVN 侧全为 `M`，无需 `svn add`）：

1. 测试数据/配置 9 个：`config/cases/*.json` 6 个 + `connection_config_gw_{multi,single}.json` + `test_plan_gw.json`
2. 测试客户端 2 个：`mock/client/CMakeLists.txt`（修正 API 头文件搜索路径）、`src/test_case_runner.cpp`（`rtn_type` 按字符输出，与 `side`/`order_type` 口径一致）
3. 报文体长修复 2 个：`include/gw_head.h`、`src/gw_head.h`
4. 字典与注释 3 个：`include/{api_interface,common_struct,order_trade_type}.h`（补齐 98 CTE 市场/侧别/委托类型，含 105/106 港股通）
5. gw/counter98 映射 4 个：`src/{gw_counter_direct,counter98}.cpp`、`src/counter98.h`、`src/single_socket_engine.cpp`
6. mock 98 柜台 1 个：`mock/98_counter/src/client_session.cpp`

验证：`g1api` + `mock_client` 重编通过（exit 0）→ GW 用例 **5/5 PASS**（登录、委托买入、成交回报、撤单、心跳 30s）。

## 3. git / SVN 双轨分流规则

- 进 SVN：仅 `trunk/NewAPI` 下代码与配置；`task/`、`work_log/`、`study/` 属 git 体系，不进 SVN。
- SVN 必须 `LANG=zh_CN.utf8`，否则中文文件名触发 `E000022`。
- SVN 未版本化项一律不提交：`lib/{mock_client,perf_client}`、`mock/*/build/`、`mock/client/api_log/`、`mock/client/result/`、`*.log`。
- git 侧则跟踪文档与任务单，并把运行期产物写入 `.gitignore`：
  `trunk/NewAPI/gone/api/mock/client/result/`、`trunk/NewAPI/gone/api/mock/client/api_log/`。

**关键坑**：`result/` 下的 `result_analysis.txt`、`test_report.txt` 早已被 git 跟踪，只加 ignore 规则**不会生效**；
必须 `git rm --cached` 把历史误跟踪文件移出版本库（磁盘文件保留），规则才对后续产物起作用。

## 4. 临时诊断块误入库的教训

本轮调试加在 `gw_counter_direct.cpp`（`DIAG_ORDER`、`DIAG_LOGIN`）、`single_socket_engine.cpp`（`DIAG_SEND`）的
`info_log` 诊断块，在未清理的情况下被打进首个提交。处置：删除 3 处 `[临时诊断]` 块 → 重编 → 用例复测 5/5 →
`git commit --amend`（`single_socket_engine.cpp` 因此无净改动，移出该提交）。

建议：诊断代码统一带 `[临时诊断]` 标记，提交前用
`grep -rn "DIAG_\|临时诊断" <src>` 自检为 0 再提交。注意 `DIAG_LOGIN` 会打印密码明文与账号字段，属敏感信息，不能入库。

## 5. 两套 VCS 的收尾状态（易错点）

- git：4 个原子提交 `d7acaa3`（代码修复+删诊断）→ `e8202cc`（文档）→ `e354592`（gitignore+产物移出）→ `86b5771`（补记日志），
  推送后 `HEAD` 与 origin 一致（ahead/behind 0 0），工作区干净。
- SVN：21 个文件已被用户提交为 **r24~r30**，但当时未删诊断块，因此 r24~r30 含 3 处 `[临时诊断]`；
  删除后工作副本仍有 `gw_counter_direct.cpp`、`single_socket_engine.cpp` 两个 `M`（纯删除）需补提交，
  命令与校验步骤见 `task/svn/task_svn.txt` 的"第零步"（按纪律由用户执行写操作）。

**教训**：同一批改动并行推进 git 与 SVN 时，两个版本库的"已提交"进度会分叉；
收尾前必须分别在两侧核对 `svn status` / `git status` 与远端 revision，不能以一侧的结论推断另一侧。
