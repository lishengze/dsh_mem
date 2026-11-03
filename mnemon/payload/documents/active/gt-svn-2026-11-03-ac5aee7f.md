---
id: "ac5aee7f-793f-41d9-a598-0e8950cccb73"
title: "gt_svn 本地复测：登录与委托链路排障基线（2026-11-03）"
description: "gt_svn FTE 本地复测的可复用基线：编译宏与 key 归一化改动、CheckError 错误码判读、组件同源约束、委托超时已排除项、远程性能基线与 --fuzzy 取数要求。"
status: "active"
created_at: "2026-11-03T14:50:49.670Z"
updated_at: "2026-11-03T14:50:49.670Z"
content_hash: "b5919f75832af65af73d7edfb307f2b83807fc5e63b5b6a842712201576ff77c"
source_paths:
  - "DYS-FRAMEWORK/fte/include/common/ute_types.h"
  - "DYS-FRAMEWORK/fte/src/business/uplink_biz_processor.cpp"
  - "DYS-FRAMEWORK/fte/fte/CMakeLists.txt"
  - "svn_work/34_控制台交互优化复盘_20261103.md"
  - "svn_work/35_gt_svn本地复测_进行中_20261103.md"
  - "svn_work/36_gt_svn登录打通与委托排查_20261103.md"
session_ids:
  - "fdccfaab-ad30-46ea-852a-3ea7a6fd6a3e"
memory_body_ids:
  []
---

# gt_svn 本地复测：登录与委托链路排障基线（2026-11-03 进行中）

适用：在本地 docker 环境复测 gt_svn（DYS-FRAMEWORK）的 FTE 功能/性能用例时，先按本文核对，
避免重复"改坏测试账号""混搭二进制""info 日志看不到错误码"这几类弯路。

## 1. 环境与关键位置

- 源码工作副本：`/mnt/work/gt_svn`（gt_svn 仓库，基线 `f99cc22de`，本地提交 `8c075dd30`）
- 构建目录：`DYS-FRAMEWORK/fte_all/build_root/build_release/fte`
- 部署/运行根：`/mnt/work/gt_test_svn/run`（`ute`、`ute_mock_client`、`etf_test_sz`）
- FTE 运行日志：`run/etf_test_sz/run/log/log_UTE_<..>_<date>.log`
- 客户端配置：`run/ute_mock_client/config/`（`connection/sz.json`、`test_cases/sz/*.json`）

## 2. 必须落地的两项编译/代码改动

| 改动 | 位置 | 原因 |
|---|---|---|
| key 归一化 | gt_svn `ftedata_init.cpp`（建 key 约 266 行、账户行约 1246 行） | 原先对 `std::array<char,N>` 整块赋值，源数据 5 字符定长、结构体 NUL 初始化 → key 带 NUL，与 FTE 侧空格填充永不相等 → 报 `No fund_data`。改为 `common::CopyToArray` + `strnlen` 空格填充 |
| `FTE_NETCARD_TEST` | `fte/fte/CMakeLists.txt`（`add_definitions(-DFTE_NETCARD_TEST)`） | 不带该宏时抓包无法识别业务报文，capture 样本为 0 |

## 3. 登录失败判读（最高价值的一条经验）

登录失败先看 FTE 侧 `DealLogon` 的 `error_code`，映射表在
`DYS-FRAMEWORK/fte/include/common/ute_types.h` 的 `struct CheckError`：

```
kSuccess=0  kFundAccountErr=1  kRepeatLogin=3  kPasswdErr=4  kGatewayModeErr=5
kConnectErr=100 kIllegalFormat=101 kGetSUnitErr=102 kGetShmRspQueueManager=103 kUserError=104
```

本次实况：`LogOnAns ... session_status[0] error_code[4]` → `kPasswdErr`，
说明归一化修复已生效（不再是 `No fund_data`），仅密码校验失败。
根因是排查 gt_trunk 资金账号映射时把客户端账号改成了数据里另一个账号，密码与 `TEST000000001` 不匹配；
换回远程验证过的原始测试账号（fund `1000000000000001` / branch `0001`）后立即登录成功。

校验处代码：`DYS-FRAMEWORK/fte/src/business/uplink_biz_processor.cpp` 的
`CheckPasswd()`（查 `ConfigManager::passwd_map_`，命中且相等才通过）。

**教训**：测试账号是三件套（fund_account_id / branch_id / password），换号必须整体换；
改过账号要在提交前复原。

## 4. 为什么必须开 `-l Debug`

info 级日志只显示客户端"登录失败"，看不到 `error_code`；`-l Debug` 才输出
`DealLogon` 的 Request/Answer 全字段，本次一行日志直接定位到密码错，省掉长时瞎猜。
开 Debug 后还可用关键字计数判定链路走到哪：

```bash
grep -acE 'OrderReq|DealOrder|UtilityCheck' <FTE 日志>   # 本次 = 0
```

## 5. "组件同源"原则（用户明确要求）

源码、`ute`（FTE 服务端）、`mock`（mock_exch_server / mock_sqlserver）、
`ute_mock_client` 必须同源编译。本次只换了源码与 ute，mock 与测试数据仍来自 gt_trunk。

已确认的约束：
- gt_svn 基线 `f99cc22de` **不含** `test/mock_exch_server`（NOT_IN_BASE），mock 本是 gt_trunk 的工具；
- `gt_svn/test/mock_exch_server/src/session.cpp` 是早期拷贝，缺少后来的自旋优化，与 gt_trunk 版本不同；
- 因此"用 gt_svn 源码重编 mock"这条路走不通，重编结果仍是同一份（且更旧的）源码。

## 6. 当前的委托失败现象与已排除项

现象：登录/心跳通过，但 `fte_combo` 2/4、`etf_combo` 0/4，失败用例均为约 8s 超时
（ETF 申购委托回报、篮子回报）；手动起与 `start_all` 起结果一致 → 确定性失败。

已排除：
- 运行的不是新二进制（`readlink /proc/<pid>/exe` + md5 与构建产物一致）；
- 部署未同步（构建产物与部署的 `ute`/`mock_client`/`libagw_ute_api.so` 三者 md5 两两一致）；
- 归一化修复未生效（`strnlen` 已生效、编译通过）；
- FTE 拒绝委托（Debug 日志中 `OrderReq|DealOrder|UtilityCheck` 计数为 0，且有 `Read failed ... End of file` 反复断连）。

结论方向：委托要么没发出，要么在进入 FTE 委托处理逻辑前连接就断开 → 优先看客户端侧发送返回码
（`SEND-DEBUG` / `ec=`），再排查 mock 报文版本混搭（r87082 有报文变更）。

## 7. 远程服务器已确立的基线与结论（本地需对齐）

| 项 | 远程结果 |
|---|---|
| 深圳功能 | 8/8 |
| 上海功能 | 3/5 + 4/4 |
| 深圳 1000 TPS | P50 10.70 µs |
| 深圳 2000 TPS | P50 9.95 µs |
| 5000 TPS | 客户端卡死 → 定位 P0-4 对象池容量（`fte_report` 2048 对每秒约 1.5 万回报） |
| 性能 capture 取数 | **必须加 `--fuzzy`**，精确匹配样本为 0 |

本地 FTE 日志里出现 `object_pool.h create ... fte_reject init successed, capacity is 2048`、
`unbound_object_pool.h get ... account_share produce too slow` 时，即对应上述容量问题。

## 8. 相关仓库内文档（gt_trunk 仓库）

- `svn_work/31_*`、`svn_work/32_*`：远程服务器对 gt_svn 的功能/性能复测结论
- `svn_work/34_控制台交互优化复盘_20261103.md`：控制台交互优化交付清单、6 个设计决策、8 个坑
- `svn_work/35_gt_svn本地复测_进行中_20261103.md`：本地复测首轮（编译/部署/登录阻塞）
- `svn_work/36_gt_svn登录打通与委托排查_20261103.md`：登录打通根因 + 委托失败定位
- 知识库 `13_FTE测试控制台.md`：新增「交互设计」章节（布局、状态色语义、门禁表、展开数据来源）
