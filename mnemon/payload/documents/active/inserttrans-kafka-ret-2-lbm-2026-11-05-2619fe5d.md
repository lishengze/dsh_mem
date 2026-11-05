---
id: "2619fe5d-6e91-4d71-828e-6d5d8f661c8b"
title: "InsertTrans Kafka 钩子修复与验证：ret=-2 根因、类型化序列化、LBM 测试模块加载（2026-11-05）"
description: "修正既有 InsertTrans 钩子文档的序列化方式：ForwardRecordRaw+JstpGetTableFieldInfo(hDb) 返回 KF_ERR_FIELD(-2) 的根因与类型化 ForwardRecord 修复、钩子最终形态、验证结果、LBM 测试模块必须导出 GetTableVersion、真实进程内端到端未跑通的三处卡点与提交范围建议。"
status: "active"
created_at: "2026-11-05T09:44:20.096Z"
updated_at: "2026-11-05T09:44:20.096Z"
content_hash: "36f770440cef14d1e255b2669d9e205c8e7a8e966b0b60a4f9c4dd3392424ea3"
source_paths:
  - "otcsrc/public/comm/kafka_insert_hook.h"
  - "otcsrc/public/include/jstp_head.h"
  - "otcsrc/kgbp_x64_linux/conf/kafka_forward.conf"
  - "otcsrc/public/include/jstp_table_versions.h"
  - "task/kafka/kafka_insert_hook_demo/test_insert_hook.cpp"
  - "study/knowledge_base_system/09_Kafka推送.md"
session_ids:
  - "3f5beb13-aaee-4d0b-89a0-f60dc8864b75"
memory_body_ids:
  []
---

# InsertTrans Kafka 钩子：ret=-2 根因修复、类型化序列化与验证结论（2026-11-05）

工作副本：`/home/lsz/code/work/otc_src`。本文是既有文档 `598a8376-dfc9-4868-bd41-749e42e66d6b`（InsertTrans 钩子并发冲突处置）的**后续与修正**：该文第 3 节记录的 `ForwardRecordRaw(hJstp, hDb, ...)` 方案在本轮实测中失败，已改为按表名分派 + 类型化 `ForwardRecord`。两文并存时，**序列化方式以本文为准**。

## 1. 根因：`ForwardRecordRaw` 在 InsertTrans 调用点不可用

- 症状：真实 kgbp 进程日志 `[KafkaInsertHook] forward failed table=trd_order/trd_appoint_order/sys_log_asset ret=-2 err=...`，`ret=-2` 即 `KF_ERR_FIELD`；同一条链路对白名单外表无任何打点（说明命中/过滤逻辑本身正确）。
- 原因：钩子原实现用 `ForwardRecordRaw + JstpGetTableFieldInfo(hDb, ...)` 现场查字段元数据，而 `hDb`（`CJstpDatabaseManager::GetDatabaseHandle()`）在该调用点取不到有效元数据，字段查询即失败。
- 修复：**按表名分派到具体记录类型**，用类型化 `ForwardRecord(..., DescribeField)` 序列化，完全不再依赖 `hDb`：

| 表名 | 记录类型 | 头文件（相对路径 include，避免依赖各模块 `-I`） |
|---|---|---|
| trd_order | `CTrd_order` | `../../dao/tables/trd_order.h` |
| trd_appoint_order | `CTrd_appoint_order` | `../../dao/tables/trd_appoint_order.h` |
| sys_log_asset | `CSys_log_asset` | `../../dao/tables/sys_log_asset.h` |

- 取舍理由：类型化后字段名/类型/偏移由 DAO 定义天然保证与内存库一致，代价是表加字段需重编（可接受，三表结构稳定）；`hDb` 参数保留在签名中但不再参与序列化（注释已更正，避免误导后续维护者）。

## 2. 钩子最终形态（`otcsrc/public/comm/kafka_insert_hook.h`）

- 挂载点：`otcsrc/public/include/jstp_head.h` 的 `InsertTrans()`，仅在插入成功（`currOid != 0`）后调用；不改变返回值与事务语义。
- 白名单编译期固定三表；非白名单表**零额外开销**（不取元数据、不序列化、不加锁）。
- 运行期开关：`CKafkaRecordForwarder::IsEnabled()` + `IsInterested(szTable, "I")`，配置 `otcsrc/kgbp_x64_linux/conf/kafka_forward.conf`（`tables=` 已追加三表）。
- 失败策略：整体 `try/catch(...)` 兜底，绝不抛异常出函数；`CKafkaInsertHookStats` 原子统计 hit/sent/skipped/failed；失败日志节流为"前 3 次 + 之后每 1000 次"，防刷爆 kgbp 日志。
- 本轮清理：删除未被使用的 `TargetTableArray()/TargetTableCount()` 死代码；修正 `hDb` 参数注释与实现不符。

## 3. 验证结论（可复跑）

- 离线验证程序：`task/kafka/kafka_insert_hook_demo/test_insert_hook.cpp`（dry_run，不依赖 kgbp 与业务数据）——**ALL PASS**：三表命中；`trd_order_un_confirmed`、`otc_seriesinfo`、空表名、NULL 均不命中；三表 JSON 全部生成（字段数 97/60/49）；直接调 `KafkaInsertHookOnInsert` 得"命中=3 转发=3 跳过=0"，三表各落盘 2 条 JSON。
- 编译：`atom_trade` / `atom_counter` / `lbm_otc_param` 均 0 error（改动后再编译一次确认无回归）。
- 旧功能回归：70200008 产品系列新增 → `kafka_record` 新增 `otc_seriesinfo`（offset 10），转发通道未被破坏。
- 5 轮复查（需求/代码/文档一致性）最终全过；中途出现的 FAIL 均系**校验脚本自身判据问题**（宿主机/容器路径混用、嵌套 `docker exec`、`sed` 转义、期望值不合理），非产品缺陷——写校验脚本时应先在小样本上验证判据。

## 4. 未完成项：真实进程内端到端未跑通（卡点在测试基础设施）

1. **自建 LBM 测试功能号 `.so` 始终 `can not load`**：kgbp 加载 LBM 模块除业务函数外**还需导出 `GetTableVersion`**（由 `otcsrc/public/include/jstp_table_versions.h` 的 `JSTP_IMPLEMENT_GETTABLEVERSION` 宏生成，签名 `int GetTableVersion(tagJstpTableVersion*, jstp_int_t*)`），且编译宏须与主程序一致（`IN_JSTP_PLATFORM`、`KCPD_ODBC_MYSQL`/`KCPD_ODBC_UNIX` 等）；否则报 `Lbm <funcid> ... can not load` 且 `dlerror` 为 null，极易误判为路径问题。本轮补宏未在测试件上生效（尺寸不变=编译未真正重跑），故未凑齐。
2. 真实业务入口 70411828（投顾转托管，会调 `Writetable_Trd_order`）在参数校验阶段即返回 `411828080 ofcode is empty`，未走到插入逻辑。
3. `WriteSysLogAsset` 在 atom 层**当前无调用者**，`sys_log_asset` 暂无活跃写入路径，无法自然触发。

## 5. 顺带发现与处置

- **并发会话冲突**：另一会话在同一工作副本并行实现同一钩子，两套实现一度并存于 `InsertTrans` 会导致**同一条记录转发两次**。处置：删除本会话重复实现，保留对方版本并在其上做 `ret=-2` 修复。
- 识别手法：编译报错指向"自己没创建过的文件"时，先看该文件 mtime 与 `git status` 中的 `??` 未跟踪文件，而非直接改代码。
- 测试临时物（`function.xml` 中 999997 死注册、测试 `.so`）已清理，工作区只保留真实改动。

## 6. 交付物与提交范围建议

产品：`otcsrc/public/comm/kafka_insert_hook.h`（新）、`otcsrc/public/include/jstp_head.h`、`otcsrc/kgbp_x64_linux/conf/kafka_forward.conf`。

测试件：`task/kafka/kafka_insert_hook_demo/test_insert_hook.cpp`、`task/kafka/kafka_ins_hook_test/`（源码 + `req_ins_hook.xml`，不含 `.so`）、`otcsrc/mock/mock_client/conf/req_ins_hook.xml`。

文档：`study/knowledge_base_system/09_Kafka推送.md` §8.11（4 个坑）、`MEMORY.md`、`Prompt.md`、`work_log/20261105.md`。

不提交：`task/kafka/kafka_task.txt` 的删除与 `task/kafka/task_kafka.txt` 的新增（属用户的新任务记录）。

## 7. 复用要点

- 内存表插入转发的通用路径已成立：`InsertTrans` 的 `lpDataBase` 即 `CJstpDatabaseManager*`，`GetPlatformHandle()` 返回 kgbp 句柄；三表 mgr 均有 `GetTableName()`，基类 `CJstpTable` 没有 → SFINAE 对其他表回退空串，钩子天然只对三表生效。
- 新增转发表的最小改动：白名单 + 类型化分派分支 + `kafka_forward.conf` 的 `tables`。
- 遇到`KF_ERR_FIELD(-2)`先怀疑元数据来源（`hDb`）而非配置；改为类型化 `DescribeField` 是确定性解法。
- 测试基础设施攻坚要有止损点：连续两轮测试件编译/加载失败且非产品代码问题，应转入收尾并如实记录未完成项。
