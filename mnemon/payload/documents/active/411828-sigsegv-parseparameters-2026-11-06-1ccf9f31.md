---
id: "1ccf9f31-6114-4272-b2a8-bd1656c2f2db"
title: "411828 SIGSEGV 根因复盘：公共入参必须在 ParseParameters() 内调用（2026-11-06）"
description: "411828 账户转托管服务端 SIGSEGV 的根因（GetPerGeneralParam 放在 Execute() 开头，CA 读流未绑定导致 TryGetValue 空句柄野指针）、修复位置、Kafka 假线索的排除方法与平台日志归属不可信的教训、写死业务日期断言的修正；最终 13 业务 13/13 PASS，提交 8f3a4030 / 00309675。"
status: "active"
created_at: "2026-11-06T06:13:09.468Z"
updated_at: "2026-11-06T06:13:09.468Z"
content_hash: "a6eff09129a4e62f69eca8f04635b2d0a7ceb176b395d3fff4c4cd8702d48d0d"
source_paths:
  - "otcsrc/lbm/lbm_otc_counter/src/processors/risk/stradtransfer_jjtg_processor_411828.cpp"
  - "otcsrc/mock/mock_client/conf/req_70411828.xml"
  - "study/knowledge_base_system/13_公共入参与测试数据规范.md"
  - "work_log/20261106.md"
session_ids:
  - "d1269f80-8446-487d-a1ce-2ab3434a0290"
memory_body_ids:
  []
---

# 411828 SIGSEGV 根因复盘：公共入参必须在 ParseParameters() 内调用（2026-11-06 收口）

> 工作副本：`/home/lsz/code/work/otc_src`。本文是文档 `3f823932-136e-4daa-89c3-e8da39b87bea`（412232/410567/411828 测试数据自包含改造收口）中「411828 服务端 SIGSEGV，未解决」一项的**解决方案与复盘**，与该文的结论冲突时以本文为准；测试数据改造原则仍见该文。

## 1. 最终结果

| 项 | 值 |
|---|---|
| 411828 账户转托管 | 14/14 PASS，零 core |
| 全量回归 | 13 个业务 13/13 PASS，零 core |
| 提交 | `8f3a4030`（代码+文档），`00309675`（任务需求文件归档） |
| 推送 | `be7b49a9..8f3a4030..00309675` → `origin dev`，同步 0/0 |

## 2. 根因：公共入参解析位置错误（本轮改造自己引入）

崩溃栈（gdb）：

```
#0 CJstpReadStream::GetValueStr (this=0x20)      ← 野指针 SIGSEGV
#2 JstpPackGetValueStr(handle=0x0)               ← CA 读流句柄为空
#3 CJstpBase::TryGetValue(this=..., "funcid")
#4 CBaseProcessor::GetPerGeneralParam()
#5 CStrdTransferJJTGProcessor::Execute() 第 51 行
```

为了让 411828 用上老系统 4 开头外围接口的 10 项公共入参，`GetPerGeneralParam()` 被放在 `Execute()` 的**最开头**。该位置框架**尚未绑定 CA 读流**，`TryGetValue` 以空 handle 取到野指针并 SIGSEGV。

**修复**：把调用移到 `ParseParameters()` 内（此处 CA 读流已绑定），与 412230/412231 的既有写法一致。代码位置：`otcsrc/lbm/lbm_otc_counter/src/processors/risk/stradtransfer_jjtg_processor_411828.cpp`。

**规则（重要）**：`JstpBase::TryGetValue` / `GetPerGeneralParam` **只能在 CA 读流绑定之后调用**，即 `ParseParameters()` 内；`Execute()` 开头只能做线程池句柄之类的准备。旁证：原代码该位置本就有一条被注释掉的 `// GetGeneralParam();`，说明历史上从未成功调用过。

## 3. 排查中被排除的两个假线索（方法论）

| 假线索 | 排除方法 |
|---|---|
| 首个 core 栈显示崩在 `pthread_cond_timedwait` / librdkafka 内部 | 关掉 kgbp.xml 中两个 Kafka XA 配置后**仍然崩** → 排除 Kafka 为根因；随后恢复 Kafka 配置复测仍通过 |
| 疑似 Kafka 异步线程自发崩溃 | 空转 150 秒、连跑 5 次其它业务均无 core → 排除自发崩溃，确认由 411828 请求触发 |

**平台日志归属不可信**：日志中的 `lbm:70411828` 会误导——异步线程崩溃会被记到当时正在跑的 LBM 上。判定根因必须以 **core 栈 + 可复现触发路径**为准，不能只看日志归属。

## 4. 附带修掉一个「必然失败」的断言

`req_70411828.xml` 的 TC-001 原先写死 `orderdate=20261004`，而出参是**当天业务日期**（当日为 `20261106`），隔天运行必然失败。处理：移除该写死断言，改为人工核对「与 `sys_config.sys_date` 一致」。

**规范**：mock 用例不要对随运行日/环境变化的字段（业务日期、时间戳、流水号）写死期望值。

## 5. 知识沉淀位置

- 仓库内规范：`study/knowledge_base_system/13_公共入参与测试数据规范.md` 第八章「重要缺陷复盘」（含本次 4 条教训与 9 条踩坑清单）。
- 工作日志：`work_log/20261106.md`。
- 改动清单：5 个处理器（411828/412230/412231/410567/412232）、5 个 `req_70*.xml`、5 个业务各 6 类文档、知识库 13 号文档与 MEMORY/Prompt，共 52 文件 +2173/−371。

## 6. 环境收尾

测试完成后按「业务节点 start.sh stop → 全局节点 global_node.sh stop → KGMS → docker stop」顺序关停容器 `otc`；停止脚本后仍残留 140 个 kgbp（多为崩溃遗留），需直接 `docker stop`。重新开始时 `docker start otc`，服务需按业务节点→全局节点→KGMS 顺序拉起。
