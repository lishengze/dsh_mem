---
id: "598a8376-dfc9-4868-bd41-749e42e66d6b"
title: "内存表 InsertTrans 级 Kafka 转发钩子：并发会话冲突处置与验证（2026-11-05）"
description: "在 jstp_head.h 的 InsertTrans 挂 Kafka 插入转发钩子的实现取舍与实证：并发会话重复实现冲突及处置、GetTableName SFINAE 验证、受影响模块与编译验证、事务语义限制与待决策项。"
status: "active"
created_at: "2026-11-05T08:31:53.542Z"
updated_at: "2026-11-05T08:31:53.542Z"
content_hash: "d87771cd070a72c28f0a0d082847097d1cf4a3163bc6d293be984f87ed40bf52"
source_paths:
  - "otcsrc/public/comm/kafka_insert_hook.h"
  - "otcsrc/public/include/jstp_head.h"
  - "otcsrc/kgbp_x64_linux/conf/kafka_forward.conf"
  - "otcsrc/dao/tables_mgr/trd_order_mgr.h"
session_ids:
  - "a5721f05-16cf-41e2-a5dc-50741ff622d1"
memory_body_ids:
  []
---

# 内存表 InsertTrans 级 Kafka 转发钩子：并发冲突处置与验证（2026-11-05）

工作副本：`/home/lsz/code/work/otc_src`（kgbp / otcsrc 主线）。本文记录"在 DAO 插入公共入口挂 Kafka 转发钩子"这一改动的**实现取舍、验证结论与未决问题**，供后续接手者直接复用。

## 1. 目标与切入位置

业务要求"某些内存表插入新记录时同步转发到 Kafka"，插入动作绝大多数收敛在 `public/include/jstp_head.h` 的 `InsertTrans()`（`objTableMgr.Insert(pTableU)` 之后）。选择在**公共插入入口**挂钩子，使业务代码零改动。

白名单固定 3 张表：`trd_order`、`trd_appoint_order`、`sys_log_asset`。

## 2. 关键事件：两个会话在同一工作副本做了同一件事

| 时间(11-05) | 事件 |
|---|---|
| 16:15 | 本会话创建 `public/comm/kafka_insert_forwarder.h`（`KafkaForwardAfterInsert` 方案） |
| 16:16 | 本会话在 `conf/kafka_forward.conf` 加三表白名单 |
| 16:20 | **另一会话**创建 `public/comm/kafka_insert_hook.h`（184 行）并改 `jstp_head.h` 挂 `kafka_fwd::KafkaInsertHookOnInsert` |
| 16:21–16:24 | 对方文件出现 `missing terminating " character` 语法错误，随后**自行修复**（mtime 16:24:01）→ 证明对方仍在活跃编辑 |

**风险**：`InsertTrans` 内两套钩子并存（旧第 115 行 `KafkaInsertHookOnInsert` + 第 128 行 `KafkaForwardAfterInsert`）→ **同一条记录被转发两次**。

**处置**：删除本会话自己的 `kafka_insert_forwarder.h` 及 `jstp_head.h` 中的 include/调用，保留对方实现（更完善）。删除后 `InsertTrans` 只剩一处调用。

**识别手法（可复用）**：编译报错指向"自己没创建过的文件"时，先查该文件 mtime + `git status` 的 `??` 未跟踪文件，而不是急着改代码——这是**并发会话冲突**信号，不是自身缺陷。

## 3. 保留方案要点

文件：`public/comm/kafka_insert_hook.h`，命名空间 `kafka_fwd`。

- **表名获取**：`KafkaInsertHookTableName(objTableMgr)` 用 SFINAE——`GetTableNameImpl(TMgr&, int)` 优先匹配 `objMgr.GetTableName()`，否则回退 `(TMgr&, long)` 返回 `""`（等价"不转发"）。
- **白名单**：`KafkaInsertHookIsTargetTable()` 三次 `strcmp`，非目标表零额外开销（不取元数据、不序列化、不加锁）。
- **运行期开关**：`CKafkaRecordForwarder::Instance()` 的 `IsEnabled()` / `IsInterested(szTable,"I")`，配置来自 `conf/kafka_forward.conf`（`enabled` / `tables` / `ops=I`），改完需重启 kgbp。
- **转发**：`ForwardRecordRaw(hJstp, hDb, szTable, "I", pRecord, "")`，名称驱动（`JstpGetTableFieldInfo` 实时取字段元数据，表加字段无需改代码）。
- **失败策略**：整体 `try/catch(...)` 兜底，绝不抛异常出函数、不改 `InsertTrans` 返回值；`KF_OK`/`KF_ERR_TOOLONG` 计 sent，`KF_SKIPPED` 计 skipped，其余计 failed 并按"前 3 次 + 每 1000 次"节流打 stderr。
- **统计**：`CKafkaInsertHookStats`（`std::atomic<unsigned long>` hit/sent/skipped/failed）。

## 4. 已验证事实（免重复排查）

- `CTrd_orderManager::GetTableName()` → `"trd_order"`（`dao/tables_mgr/trd_order_mgr.h:117`）、`CTrd_appoint_orderManager` → `"trd_appoint_order"`（`trd_appoint_order_mgr.h:117`）、`CSys_log_assetManager` → `"sys_log_asset"`（`sys_log_asset_mgr.h:120`）✅
- 基类 `CJstpTable`（`public/include/jstp_table.h`）**没有** `GetTableName()` → SFINAE 对其他所有表回退空串，钩子天然只对三表生效 ✅
- `lpDataBase->GetPlatformHandle()` 返回 kgbp 句柄（`public/comm/kgbp_lbm_base.h:91` 的 `SetPlatformHandle(m_hJstp)`）✅
- 受影响模块目录（含 `jstp_head.h` 的模块）：`core/atom_counter/include`、`core/atom_include/include`、`core/atom_quota/include`、`core/atom_rts/src`、`core/atom_trade/{include,src}`、`lbm/lbm_jstp_system{,_global}/include`、`public/{busi_tools,comm,include,util/include}`。
- 插入点分布：`trd_order`/`trd_appoint_order` 的 4 处 `InsertTrans` 在 `atom_trade`；`sys_log_asset` 的 2 处在 `atom_counter`。

## 5. 编译验证结果（0 error）

```
build_kgms/build_fast/otcsrc/lbm/lbm_otc_param/liblbm_otc_param.so
build_kgms/build_fast/otcsrc/core/atom_trade/libatom_trade.so
build_kgms/build_fast/otcsrc/core/atom_counter/libatom_counter.so
```

顺序策略：先编"受影响最广"的 `lbm_otc_param`（公共头改动的最广消费者），再编真正承载插入点的 `atom_trade` / `atom_counter`。

## 6. 已知限制与未决事项

1. **事务语义**：钩子在"记录已插入、事务未 Commit"时触发；若业务随后 Rollback，Kafka 会多出一条未落库记录。严格一致需改为 Commit 成功后再转发（`base_processor.h` 的 `ForwardRecordToKafka` 路径）。相关提交前判定见文档 `27e439e6-262d-4c7b-82ee-669d9e65375a`（§3.4 Kafka 转发时机）。
2. **消息量**：`sys_log_asset` 是资产流水表、写入极频繁，全量开启可能产生巨量 Kafka 消息；建议评估是否先只开 `trd_order`/`trd_appoint_order`，或对该表单独限流/开关。
3. **待用户裁决（A/B/C）**：A 继续做完部署+重启 kgbp+端到端复测（前提：另一会话停手）；B 停手交由对方完成、本会话只做终审；C 只做只读评审出意见。
4. **并发治理**：同一工作副本上多会话并行改同一文件会导致互相覆盖、抢同一 kgbp 进程；接手前必须先确认无其它活跃会话。
5. 工作区状态（当时）：`M conf/kafka_forward.conf`、`M public/include/jstp_head.h`、`?? public/comm/kafka_insert_hook.h`；用户侧把 `task/kafka/kafka_task.txt` 重命名为 `task/kafka/task_kafka.txt`。新头文件若要提交，须随改动一并入库（未跟踪会导致他人 update 后编译失败）。
