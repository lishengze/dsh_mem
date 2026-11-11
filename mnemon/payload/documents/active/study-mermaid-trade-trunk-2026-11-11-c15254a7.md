---
id: "c15254a7-e612-4475-a211-e8f21a30514b"
title: "study/ 知识库 Mermaid 图形配色统一与解析失败修复（trade_trunk，2026-11-11）"
description: "trade_trunk 的 study/ 下 14 个文件 / 49 个 Mermaid 块的莫兰迪配色统一、3 个解析失败的精确根因与转义规则、9 类语义配色与对比度依据、自建解析校验器门禁，以及 KGBP_TOOLS/AMO安装包.zip 于会话期间从磁盘消失导致引用门禁 6 处告警的环境异动。"
status: "active"
created_at: "2026-11-11T10:30:56.081Z"
updated_at: "2026-11-11T10:30:56.081Z"
content_hash: "94cca9eb277e5a0d048b233a7a09ecde143e07e66a02a03bf20160946fbda883"
source_paths:
  - "study/framework_mastery/01_框架掌握总览.md"
  - "study/framework_mastery/04_CTE业务实现基准.md"
  - "study/business_learning/hk_connect/01_港股通公司行为.md"
  - "study/business_learning/hk_connect/02_港股通转托管.md"
  - "study/business_learning/hk_connect/03_港股通交易.md"
  - "study/business_learning/hk_connect/05_深港通转托管与DCOM非交易报盘.md"
  - "study/knowledge_base_system/07_环境与排障.md"
session_ids:
  - "afaed6fe-4b29-4509-b81e-792ff0a98793"
memory_body_ids:
  []
---

# study/ 知识库 Mermaid 配色统一与解析失败修复

工作区：`/home/lsz/code/work/trade_trunk`（分支 `dev`）。提交 `5b20277b`（`5b20277b D：按莫兰迪色卡重制全部 49 个 Mermaid 流程图配色，并修复 3 个解析失败图`），已推送到 `github.com:lishengze/server_11.git`，同步核对为 `0 0`。
覆盖范围：14 个 Markdown 文件 / 49 个 Mermaid 代码块，其中 32 块原先完全没有配色。

## 1. 问题分两类（用户只报了「字看不清」）

| 问题 | 实测 |
|---|---|
| 32/49 块没有任何配色 | 走 Mermaid 默认主题（浅黄底 + 默认字色），浅色背景上对比度不足 |
| 3 块根本无法解析 | 渲染成报错而非图形：`01_港股通公司行为.md` 块4、`02_港股通转托管.md` 块1、`03_港股通交易.md` 块3 |

## 2. 三个解析失败的精确根因（可复用）

| 图类型 | 破坏字符 | 结论 |
|---|---|---|
| `stateDiagram-v2` | 标签中 `ExecType::kNew` 的 `::` | **加引号无效**，必须转义为 `&#58;&#58;` |
| `sequenceDiagram` | 消息与 `alt`/`else` 标签中的 `;` | **加引号无效**，必须转义为 `&#59;` |
| `flowchart` | 未加引号的 `()`，如 `P[Rollback RoundAmtExchangeRate2(GetExchangeRate(true))]` | 补引号即可 |

要点：先按图类型分流，序列图/状态图类标签不能靠加引号解决。

## 3. 配色方案（基于用户提供的莫兰迪色卡）

色卡是配色设计稿、不是按「可作填充」筛过的，因此先用 WCAG 对比度计算再定用途：取明度较高的色作填充、统一配深灰墨字 `#2F3438`，全部达 AA（≥4.5:1）；偏深的 3 个色降级为只作描边（作填充时对比度不达标：灰酒红 2.65、灰橄榄绿 3.49、灰褐咖 3.89）。

9 类语义配色（填充 / 对比度 / 色卡名）：

| 语义 | 填充 | 对比度 | 色卡名 |
|---|---|:---:|---|
| base 既有/不变 | `#B9B9B9` | 6.41 | 中性浅灰 |
| platform 平台/底座 | `#94A8B5` | 5.11 | 雾霾蓝 |
| store 存储/内存表 | `#94A58D` | 4.81 | 灰苔绿 |
| data 数据/报文 | `#D9C6C2` | 7.67 | 石英裸粉 |
| biz 业务逻辑 | `#D9CFC1` | 8.18 | 燕麦米 |
| framework 框架/能力 | `#A89BA8` | 4.74 | 灰紫 |
| new 本次新增 | `#B89A94` | 4.85 | 灰玫瑰 |
| target 目标形态 | `#CFA89E` | 5.84 | 莫兰迪豆沙粉 |
| todo 待实现 | `#BFAF9E` | 5.90 | 浅驼咖 |

其他固定值：文字 `#2F3438`、连接线 `#5F574F`、背景 `#FBFAF9`。

落地方式：
- `flowchart`：逐节点 `classDef`（保留原有类归属，只换色）。
- `sequenceDiagram` / `stateDiagram-v2` / `classDiagram`：用 `%%{init: {"theme":"base","themeVariables":{...}}}` 统一注入。
- init 行首版冗长（49 次重复、每处近 5 KB），已压缩到 **1216 字符**，只保留生效键。

## 4. 验证与门禁

- 自建 Mermaid 解析校验器作为门禁：**改造前 3 块失败 → 改造后 49/49 全部解析通过**。
- 剔除 Mermaid 代码块后的正文：15 个文件逐字节零改动（只改图内内容）。
- 抽查 5 个修复块，箭头/参与者计数完全一致（未增删节点或边）。
- 代码围栏配对；每块恰增 1 行 init，行数变化与块数吻合（如 `01_框架掌握总览.md` 2 块 → +4/-2）。
- **视觉终验未做**：本机无法渲染。`~/.cache/puppeteer/chrome/*` 的 zip 全部损坏/截断（131/155/157 三个版本解压均报 `cannot find zipfile directory`），系统只有 `/usr/bin/firefox`，与 puppeteer 不兼容。只做了「解析 + 配置值 + 对比度」的文本级验证，视觉确认需人工打开文档。

## 5. 环境异动（非本次改动造成）

`KGBP_TOOLS/AMO安装包.zip`（4.0 GB，未入库、在 `.gitignore` 内）在会话期间从磁盘消失，全盘查找为空。本次只修改 `study/` 下的 Markdown，未写入 `KGBP_TOOLS/`。由此 `study/knowledge_base_system/07_环境与排障.md:279,307` 出现失效路径引用，仓库引用门禁共报 6 处问题（2309 处路径引用 / 2309 检查中），均为该文件、与本次改动无关。是否改文档还是等文件恢复，待用户确认。

## 6. 复用经验

- `/tmp/mmdval/fix_and_color.py` + `palette.py` 为本次的一次性工具。中途靠零散补丁修补脚本，先后丢失 `split_blocks`、`FLOW_CLASSES`、`diagram_kind`、`ESCAPE_BY_KIND` 等函数，反复失败；**最终改为整文件重建才通过**——工具脚本局部损坏时应整体重写而非逐段打补丁。
- 改动前先备份全部目标文件、改动后立即用 `git status` 确认「只有预期的 N 个文件被改、无游离文件」，是这类批量文本改写的有效护栏。
