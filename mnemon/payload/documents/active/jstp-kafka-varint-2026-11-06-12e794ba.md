---
id: "12e794ba-aff6-4507-b5a0-a69b9fb58eb3"
title: "JSTP 字段体（Kafka 消息体）逐字节格式实测：varint 长度域与解析纪律（2026-11-06）"
description: "kafka_record 消息体（JSTP 字段体）的字节布局实测：ca 01 标识、varint 字段描述符与长度域、封套字段清单、禁止硬编码 11 字节偏移的解析纪律，以及 probe_pack.cpp 复现方法与 Excel/知识库同步点。"
status: "active"
created_at: "2026-11-06T03:06:59.154Z"
updated_at: "2026-11-06T03:06:59.154Z"
content_hash: "076e057934cf5c042f40f2ef3bdc88153cd8626a8f614cf6c1b855baee96f2b6"
source_paths:
  - "task/kafka/probe_pack.cpp"
  - "otcsrc/library/lbmapi/include/jstp_pack_type.h"
  - "otcsrc/library/lbmapi/include/jstp_pack.h"
  - "otcsrc/public/busi_param/jstp_pack_business_manager.h"
  - "study/knowledge_base_system/09_Kafka推送.md"
  - "kafka_sub_template.xlsx"
session_ids:
  - "9f1e9959-8f43-424c-baa9-97a05ebae158"
memory_body_ids:
  []
---

# JSTP 字段体（Kafka 消息体）逐字节格式：实测结论与解析纪律（2026-11-06）

工作副本：`/home/lsz/code/work/otc_src`。本文记录 `kafka_record` 主题消息体的二进制格式实测结论，用于说明"包头每个字段是什么、哪些固定、哪些可变"。相关实现背景见文档 `2619fe5d-6e91-4d71-828e-6d5d8f661c8b`（InsertTrans Kafka 钩子修复与验证）。

## 1. 结论先行

- `kafka_xa` 剥掉 104 字节 KGBP 包头后，订阅方看到的就是**裸 JSTP 字段体**；`bos_sub` 则是完整 KGBP 包（前 4 字节长度 + `KGBP` magic）。
- 该字段体**只含一个字段**：`RECORD`，类型 = 字符串。
- **报文头不是定长**：长度域是 varint，小报文头部 11 字节、中大报文 12~13 字节。**订阅方绝不可硬编码"跳过 11 字节"。**

字节布局（★ = 变长）：

```
ca 01                       固定 2 字节，JSTP 字段体标识
★ varint(名字长度×8 + 类型码)  当前实现恒为 0x34（RECORD=6 字节、STRING=4 → 6*8+4=52=0x34）
  R E C O R D                 字段名，6 字节 ASCII
★ varint(JSON 字节长度)       1~3 字节
  JSON 正文                  UTF-8 编码的完整 JSON 对象
```

## 2. 各组成"固定/可变"判定与实测依据

| 组成 | 判定 | 依据 |
|---|---|---|
| `ca 01` | 固定 | 15 组不同长度/名字/类型的实验全部输出这两字节 |
| 字段描述符 | 当前实现固定（0x34） | 由 `名字长度*8 + 类型码` 决定：名字 `R`→`0x0c`、`RECORDX`→`0x3c`、int→`0x31`、char→`0x30`、binary→`0x35`；超长名字的描述符本身也会变成多字节 varint |
| 字段名 `RECORD` | 当前实现固定 | 转发器只写这一个字段；改名会同时改变描述符 |
| 长度域 | **可变** | varint：≤127 用 1 字节，≤16383 用 2 字节，更大 3 字节 |
| JSON 正文 | 长度可变 | 长度恰等于长度域的值 |
| 字段个数 | 当前恒为 1 | 若追加第二字段，会紧接 `2c "EXTRA" 01 78`（实测两字段串联） |

类型码枚举来源：`otcsrc/library/lbmapi/include/jstp_pack_type.h` —— CHAR=0 / INT32=1 / INT64=2 / DOUBLE=3 / STRING=4 / BINARY=5 / DATASET=6 / PACK=7。

长度域 varint 实测对照：`0→00`、`1→01`、`10→0a`、`100→64`、`300→ac 02`、`981→d5 07`、`1953→a1 0f`、`5000→88 27`。

## 3. JSON 封套（顶层字段）

`schema`(固定 `kgbp.record.v1`，可做版本校验) / `table` / `op`(I=新增, U, D；钩子路径当前只推 I) / `funcid`(手工转发=业务功能号，**插入钩子路径为空串**) / `sys_id` / `as` / `ts`(毫秒) / `ts_str` / `data`(记录本体，JSON 对象)。

必备字段：`schema`、`table`、`op`、`data`；其余为恒定输出但值可变。

## 4. 订阅方解析纪律

1. 推荐用 `JstpPack` 解包取字段 `RECORD`（`JstpPackPutPacket` + `JstpPackGetValueStr`）；`JstpPackInit()` 可独立使用，不依赖 kgbp。
2. 轻量做法：在字节流中定位首个 `{"schema"` 子串，其后全部按 JSON 解析；**不要硬编码偏移量**（字段名长度一改，偏移就变）。
3. `data` 内字段个数因表而异且随版本增减（实测 96/60/49；本版 `trd_order.first_buy_flag` 未输出），必须**按字段名取值**，不可按位置或个数解析。
4. 下游若拿到的是 `kafka_xa` 数据，注意它已是裸 JSTP 字段体，不要再剥 104 字节。

## 5. 复现方法（可随时重跑）

探针程序：`task/kafka/probe_pack.cpp`，链接 `jstp_pack`，用 `JstpPackInit/BeginWrite/SetValue*/GetPacket` 对不同长度、字段名、类型打包后打印头部字节。

编译期注意点（实际踩过的坑）：`otcsrc/library/lbmapi/include/jstp_pack.h` 会 `#include <jstp_config.h>`，只加 lbmapi/include 会报 `jstp_config.h: No such file or directory`；需同时加 `otcsrc/public/include`（或 `/opt/fs/kgbp/include`）等头文件目录。`jstp_pack.h` 本身不在仓库根，实际位于 `otcsrc/library/lbmapi/include/`。

## 6. 产出同步点

- 知识库：`study/knowledge_base_system/09_Kafka推送.md` §8.15（逐字段说明与实测方法），§8.14 中的重复表格已改为指向 §8.15。
- Excel：`kafka_sub_template.xlsx` Sheet2 由 §"报文头字段说明"（5 行：JSTP 标识 / 字段描述符 / 字段名 / 长度域 / JSON 正文，属性列填「—」）与 §"封套字段说明"（9 字段）两个新增区块承载；生成脚本 `build_kafka_sub_doc.py`、校验脚本 `verify_doc.py` 已同步升级并全部 PASS，且生成幂等（两次 md5 一致）。
- 校验脚本的历史缺陷：原先用"行号 + 固定偏移"定位表头，新增区块后错位；已改为按单元格内容扫描定位。
