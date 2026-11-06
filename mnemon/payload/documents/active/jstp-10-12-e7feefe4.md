---
id: "e7feefe4-7a8e-46a8-83bc-edb19185c0d0"
title: "JSTP 报文头长度域：定长不可行与下游免算宽度的解析法（修正 10~12 字节）"
description: "修正 JSTP 头部长度 11~13 → 10~12 字节；记录长度域无法定宽（SDK 无选项）、跳过续字节的免算宽度解析法及其实测脚本。"
status: "active"
created_at: "2026-11-06T05:36:15.851Z"
updated_at: "2026-11-06T05:36:15.851Z"
content_hash: "350897ad8f3f9d5273a1cd2e68a0123823225a96e1e06a3a261ad09770d1254f"
source_paths:
  - "task/kafka/parse_jstp_record.py"
  - "task/kafka/test_parse_jstp_record.py"
  - "task/kafka/annotate_header.py"
  - "task/kafka/test_skip_continuation.py"
  - "task/kafka/emit_pack.cpp"
  - "task/kafka/emit_tricky.cpp"
  - "task/kafka/build_kafka_sub_doc.py"
  - "study/knowledge_base_system/09_Kafka推送.md"
  - "otcsrc/library/lbmapi/include/jstp_pack_type.h"
session_ids:
  - "15bc245e-bcf5-46f4-9182-18d386999c7d"
memory_body_ids:
  []
---

# JSTP 报文头长度域：定长不可行 + 下游免算宽度的解析法（修正 11~13 → 10~12）

本文修正并扩展已有文档 `12e794ba-aff6-4507-b5a0-a69b9fb58eb3`（JSTP 字段体逐字节格式）。该文档中「小报文头部 11 字节、中大报文 12~13 字节」的数值有误：按实测应为 **10 / 11 / 12 字节**（旧结论是按 981/1953 字节样本这一特例归纳的）。背景实现见 `2619fe5d-6e91-4d71-828e-6d5d8f661c8b`。

## 1. 精确布局与三档头长

```
偏移 0..1   ca 01                固定 2 字节标识（先校验）
偏移 2      varint v             名字长度 = v >> 3，类型 = v & 7（当前恒 0x34 → 名字 6 字节、STRING=4）
偏移 3..8   "RECORD"            跳过「名字长度」个字节
偏移 9      varint L             JSON 字节长度，LEB128，宽度 1~3 字节
偏移 9+Lw   JSON 正文起点        Lw = L 的 varint 宽度
```

头部总长 = 2 + 1 + 6 + Lw = **10 / 11 / 12 字节**（JSON ≤127 / ≤16383 / ≤2097151）。

实测对照（`annotate_header.py` 标注真实报文）：27→`1b`（1 字节，起点 10）、83→`53`（1 字节，10）、609→`e1 04`（2 字节，11）、5002→`8a 27`（2 字节，11）、20002→`a2 9c 01`（3 字节，12）。

## 2. 长度域无法改成固定 3 字节（设计结论）

- `jstp_pack_type.h` 仅暴露 5 个选项（`JSTP_PACK_OPTION_PACK_HEAD_SIZE` / CHARSET / VERSION / HANDLE / CALLBACK），**没有长度域定宽选项**；宽度由 `libjstp_pack` 内部 LEB128 自适应。
- 自行拼非标准字段体（定宽长度）会被 `kafka_xa` 的 `JstpPackPutPacket` 拒绝，且破坏标准 JstpPack 兼容性。

## 3. 下游无需判断宽度（推荐做法）

varint 除最后一个字节外其余字节最高位（bit7/0x80）均为 1，最后一个为 0：

```python
p = 9
while msg[p] & 0x80:   # 最高位为 1 → 还有续字节
    p += 1
json_body = msg[p + 1:]  # 下一个字节即 JSON 起点；JSON 延伸到报文末尾，长度值都不用算
```

C 同构：`while (msg[p] & 0x80) p++;`。前提是字段名恒为 `RECORD`（本 Topic 成立）。

三种做法对比：跳过续字节（2~3 行，推荐）／完整 varint 解析（约 10 行，通用且可拿长度做校验）／查找首个 `{"schema"`（1 行，但要求 schema 是第一个键）。

关于 `{"schema"` 标记法：JSON 字符串内引号必须转义为 `\"`，故原始字节序列 `{"schema"` 只会出现在正文起点；已用「字段值里含 schema 字样」的构造报文（`emit_tricky.cpp`）验证不会误命中，但 schema 非首键时该法失效（返回 -1）。

## 4. 验证与参考实现

- `task/kafka/parse_jstp_record.py`：三步 varint 解析（`parse_fields` / `extract_record_json` / `locate_json_offset` / `extract_by_marker`），可作命令行工具。
- `task/kafka/test_parse_jstp_record.py`：7 样本（1/2/3 字节宽度、含中文多字节），校验「JSON 逐字节一致且恰好用完报文」，实测 ALL PASS。
- `task/kafka/annotate_header.py`：逐字节标注头部（上表数据来源）。
- `task/kafka/test_skip_continuation.py`：8 样本（含 tr ic ky 构造样本）验证免算宽度法，ALL PASS。
- `task/kafka/emit_pack.cpp` / `emit_tricky.cpp`：生成 1/2/3 字节长度域、中文、含 schema 字样的真实样本。
- 文档同步点：`task/kafka/build_kafka_sub_doc.py` 生成 `kafka_sub_template.xlsx`（幂等哈希校验通过）；`study/knowledge_base_system/09_Kafka推送.md` §8.15（含 §1.2 免算宽度节）。

## 5. 易错点

- 不要硬编码「跳过 11 字节」，也不要写死 10 或 12；按 bit7 循环或完整解析。
- 长度域宽度与 JSON 字节数绑定，不是「报文总长」的函数。
- 头长旧值 11~13 仅在 2 字节长度域时成立，勿再沿用。
