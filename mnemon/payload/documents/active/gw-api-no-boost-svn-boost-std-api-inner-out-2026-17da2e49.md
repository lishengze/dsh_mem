---
id: "17da2e49-f417-4ac9-bd8f-c6c058eaf828"
title: "gw_api_no_boost 三处 SVN 未提交改动的成因盘点：boost→std 智能指针迁移、生成器不同步隐患、API_INNER_OUT 性能探针（2026-11-27）"
description: "gt_trunk 下 DYS-FRAMEWORK/api/gw_api_no_boost/api/src 三个 SVN 状态 M 文件（agw_event.cpp、agw_client_impl_auto_gen.cpp、ute_client_handler_impl.cpp）的改动点与原因：ExternalMessagePtr 为 std::shared_ptr 导致的 boost→std 替换、dump_agw_event.py 未同步会回退 178 处、SendMessage 三段打点探针，以及 branch_id 修复被回退的口径。"
status: "active"
created_at: "2026-11-27T06:32:45.414Z"
updated_at: "2026-11-27T06:32:45.414Z"
content_hash: "715d4e4413aad15a039daac863209e357da73c8f9280d7b87355ff8c3241c634"
source_paths:
  - "DYS-FRAMEWORK/api/gw_api_no_boost/api/src/agw_event.cpp"
  - "DYS-FRAMEWORK/api/gw_api_no_boost/api/src/agw_client_impl_auto_gen.cpp"
  - "DYS-FRAMEWORK/api/gw_api_no_boost/api/src/ute_client_handler_impl.cpp"
  - "DYS-FRAMEWORK/api/gw_api_no_boost/api/include/agw/types.h"
  - "DYS-FRAMEWORK/api/python/dump_agw_event.py"
  - "DYS-FRAMEWORK/api/python/dump_agw_client_file.py"
session_ids:
  - "02e547f5-5504-47e6-ac18-7927aef76faa"
memory_body_ids:
  []
---

# gw_api_no_boost/api/src 的 3 个 SVN `M` 文件：改动点与原因（2026-11-27 盘点）

工作副本 `/home/lsz/code/work/gt_trunk`，目录 `DYS-FRAMEWORK/api/gw_api_no_boost/api/src`。
`svn status` 的 `M` 只有 3 个（另有 1 个未纳管 `?` 备份文件）：

| 文件 | 变更量 | 一句话 |
|---|--:|---|
| `agw_event.cpp` | +178 / −178 | `boost::dynamic_pointer_cast` → `std::dynamic_pointer_cast` |
| `agw_client_impl_auto_gen.cpp` | +209 / −209 | `boost::make_shared` → `std::make_shared`（207 处）+ `new boost::asio::io_service::work` → `new asio::io_service::work`（1 处） |
| `ute_client_handler_impl.cpp` | +24 / −1（25 行） | 新增 `API_INNER_OUT` 环境变量控制的性能探针 |

`agw_client_impl.cpp` **SVN 干净**（不在上表中），虽然 git 侧显示 `M`（`TEST_UTE_API` 相关的 `#ifdef` 改动，9 行）——即 SVN 与 git 两套基线的差异，不要混淆。

## 1. 前两个文件：同一根因——消息容器是 std 智能指针

`api/include/agw/types.h:43`

```cpp
typedef std::shared_ptr<ExternalMessage> ExternalMessagePtr;  // 原 boost::shared_ptr 已注释
```

因此：

- `boost::make_shared<T>()` 造出 `boost::shared_ptr<T>`，赋给 `ExternalMessagePtr` 不成立 → 必须 `std::make_shared`。
- `boost::dynamic_pointer_cast<T>(message)` 返回 `boost::shared_ptr`，同样不能互转 → 必须 `std::dynamic_pointer_cast`。
- `BOOST_AUTO(...)` 宏**保留**（等价于 `auto`），只替换 cast/构造实现。
- `agw_client_impl_auto_gen.cpp` 里唯一非 `make_shared` 的改动：`ios_work_(new boost::asio::io_service::work(ios_))` → `new asio::io_service::work(ios_)`，因为 no_boost 变体走自带 asio 头（编译参数 `-I.../api/include/asio/1_12_2/include`）。

no_boost 变体并非"零 boost"：仍有 `boost::bind`(215)、`boost::placeholders`(207)、`boost::format`(79) 等，仅智能指针/asio 命名空间这一层改成 std 宏/自带头。

## 2. ⚠️ 真实隐患：生成器只同步了一半

两个文件都是脚本生成的，但只有构造侧脚本被改过：

| 生成器 | 现状 | 覆盖对象 |
|---|---|---|
| `api/python/dump_agw_client_file.py:598` | 已输出 `std::make_shared` | `agw_client_impl_auto_gen.cpp`（209 处安全） |
| `api/python/dump_agw_event.py:129` | **仍输出 `boost::dynamic_pointer_cast`** | `agw_event.cpp` |

**后果**：一旦重新生成 `agw_event.cpp`，那 178 处手改会全部回退。提交前应先把 `dump_agw_event.py:129` 的模板串改成 `std::dynamic_pointer_cast`。`gw_api_boost` 与 `gw_api_no_boost` 两份 `dump_agw_event.py` 都是旧串（第 129 行相同），一并处理。

## 3. `ute_client_handler_impl.cpp`：`API_INNER_OUT` 性能探针（+24/−1）

目的：把 no_boost 版 `SendMessage` 的耗时按段拆开（对应提交 `be3639b4f`「125 号 - no_boost 版 SendMessage 耗时段分析」的结论：总 3.83µs，其中约 87% 在 `sendmsg`，编码仅 80ns）。

形态要点：

- 文件顶部匿名命名空间：`nb_now_ns()`（`clock_gettime(CLOCK_MONOTONIC)`）+ `struct NbInnerWriter`，构造时 `getenv("API_INNER_OUT")`，未设置即不 `fopen`，**零开销**；命中则 `fopen(..., "w")` + 1MB `setvbuf`，每 256 行 `fflush`。
- `SendMessage` 内三处打点：`t_arrive`（进入函数）→ `t_leave`（编码完成、准备 `endpoint_->write`）→ `t_send`（`write` 返回），最后 `put(t_leave-t_arrive, t_send-t_leave, total)` 输出 `编码,发送,总耗时` 三列 CSV。
- 是否入库需用户决策：默认零开销，但属诊断代码，会进生产库；不留则在 SVN 提交前摘除。

## 4. 口径澄清：branch_id 修复已被回退，不要"再修一次"

此前一次修改曾把 `AgwMsgCustLoginResp` 顶层字段读法改成遍历 `fund_account_array` 取 `branch_id`（因为当时 800006 应答顶层没有 `branch_id`）。**后来报文结构体确实增加了顶层 `branch_id`**（`messages_agw_external.h` 中 `AgwMsgCustLoginResp` 第 21 行 `BranchID_def branch_id;`），原写法 `p->branch_id` 已合法，故该修复被回退、不再需要。当前 `ute_client_handler_impl.cpp:3296` / `:3546` 直接用 `p->fund_account_id, p->branch_id` 是正确的。

同源提醒：`62c52ec61 D:更新数据` 这个提交曾被一次带入冲突标记（见 `807216fd-8810-4b4e-8201-06318e73736a`），涉及 gw_api 两树的改动先确认基线与冲突标记再动手。

## 5. 杂项

- `ute_client_handler_impl.cpp.bak_premutex_20261113` 是未纳管（`?`）备份文件，**不要 `svn add`**，建议清理或 ignore。
- 宿主无 svn，容器内 `/usr/bin/svn` 1.7.14；所有 `svn status/diff` 需在容器 `otc` 内执行（`ATP_RES_ROOT: /mnt/work/gt_test/work_atp/cmake/` 为其环境特征）。
- 三个文件在 **git 侧已提交**（`git status` 干净），本次 SVN 提交属「git 已有、SVN 待补」。
- SVN 提交只出清单、由用户手动执行（铁律）；提交范围必须显式列文件，禁止整目录（同 `d2ca6007-c97e-49be-8225-ca2d451a7c8c`）。

## 相关文档

- `807216fd-8810-4b4e-8201-06318e73736a`：gw_api 两树 SVN 冲突的融合口径（P0-1 补丁化、日志注释粒度）
- `d2ca6007-c97e-49be-8225-ca2d451a7c8c`：API 性能优化三文件的 SVN 提交执行手册
- `a4d51da0-5efc-41af-8a0f-2e44f521b13f`：容器内重建 mock_client / GLIBCXX 与 boost 抢占
