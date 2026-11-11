---
id: "f7e5426f-3c90-4e21-8fc2-3bf65ae9476b"
title: "Mermaid %%{init}%% 指令致 \"No diagram type detected\" — 根因、修复与验证同路径原则（trade_trunk，2026-11-11）"
description: "trade_trunk 知识库 Mermaid 配色改制的收尾阶段：49 个图中注入的 %%{init}%% 指令因 JSON 多余花括号非法，导致 VS Code/预览报 \"No diagram type detected matching given configuration\"；含精确因果链、json.dumps 生成+往返自检的修复、git revert 恢复策略，以及「校验方式必须与被校验对象一致」这一核心经验。"
status: "active"
created_at: "2026-11-11T10:39:27.923Z"
updated_at: "2026-11-11T10:39:27.923Z"
content_hash: "79373f2783b57406fc281e868acb45c9ab879a4772c42e452702ee2828cf3cb7"
source_paths:
  - "study/framework_mastery/01_框架掌握总览.md"
  - "study/business_learning/hk_connect/01_港股通公司行为.md"
  - "study/business_learning/hk_connect/02_港股通转托管.md"
  - "study/business_learning/hk_connect/03_港股通交易.md"
  - "study/knowledge_base_system/02_核心业务流程.md"
  - "work_log/20261111.md"
  - "task/review/check_doc_refs.allow"
session_ids:
  - "4a708a21-6f6a-47c2-b7f9-cbb940b9e1c4"
memory_body_ids:
  []
---

# Mermaid `%%{init}%%` 指令导致渲染报错的根因与修复

工作区：`/home/lsz/code/work/trade_trunk`（分支 `dev`，远程 `github.com:lishengze/server_11.git`）。修复提交 `9419610d`（`D：修复 Mermaid render error（init 指令 JSON 非法）+ 完成莫兰迪配色重制`），推送后同步核对 `0 0`。

**本文是 [study/ 知识库 Mermaid 图形配色统一与解析失败修复](c15254a7-e612-4475-a211-e8f21a30514b) 的后续纠正**。该文档记录的是配色改制阶段，其"用 `%%{init}%%` 注入配色"的做法在该阶段结束时是坏的；本文记录其后的报错、根因与最终修复。两文合起来才是完整结论。

## 1. 现象

用户在 Markdown 预览中打开文档，流程图位置显示：

```
mermaid render error: No diagram type detected matching given configuration for text
```

**关键理解**：这条报错**不是**"忘了写图类型"。图类型（`flowchart TB`、`sequenceDiagram` 等）都在。它表示 **Mermaid 读不懂代码块的开头**——它把无法识别的行当成了图内容，于是探测不到图类型。

## 2. 因果关系链（精确）

```
注入的 %%{init: ...}%% 指令里 JSON 非法（手工拼花括号多写一个 }）
        ↓
Mermaid 无法解析并剥离该指令
        ↓
指令文本被当成图内容
        ↓
探测不到图类型 → No diagram type detected matching given configuration
```

具体形态：指令以 `..."fillType7":"#BFAF9E"}}}}%%` 结尾，**3 个右花括号**，但该 JSON 只需要 2 个（一个闭合 `themeVariables`，一个闭合根对象）。JSON 解析器报 `Unexpected non-whitespace character after JSON at position 1206`。

**规模**：49 个 Mermaid 块全部中招（同一个 init 字符串被复制注入），即所有图都无法渲染。

## 3. 为什么上一轮"验证通过"了还是错（本文最重要的一条）

上一轮的校验器**在 `parse()` 之前先把 `%%{init}%%` 剥离掉**再解析。而 init 指令正是最容易出错、也正是本次报错根源的部分——**等于把出错的部分排除在校验之外，校验方式与被校验对象不匹配**。

正确做法：**校验路径必须与真实消费路径一致**——不剥离 init，直接 `parse()` 整块。改完后一次就复现了用户的报错，也证明新写法通过。

这条经验不限于 Mermaid：任何"自建窄化校验器"只要比真实使用者少读一部分输入，就会给出假绿。

### 相关的路径差异（实测）

- `parse()` 在**指令行在图类型之前**时会失败（Mermaid 的 `parse()` 不剥离指令）；`render()` 另有环境依赖（本机复现时报 `CSSStyleSheet is not defined`，属 jsdom 环境限制）。
- 因此判断"指令是否合法"应直接对指令内的 JSON 做校验，而不是依赖 `parse()` 单独通过。

## 4. 修复做法

1. **改用 `json.dumps` 生成 init 字符串**，不再手工拼花括号；并加往返自检 `json.loads(inner) == cfg`，不自洽就直接报错。
2. **校验器改为不剥离 init 直接 `parse`**（与预览同一条路径）。
3. **用 `git revert` 撤销含错误 init 的提交**（`5b20277b`），从干净原文重新注入——已推送的分支不重写历史。
4. 加入**幂等保护**，防止对同一块重复注入 init。
5. 顺带修掉 3 处既有语法错误（详见 `c15254a7` 及下方）。

最终 init 长度 **1218 字符**，`themeVariables` 共 **44 个键**，尾部为 `..."fillType7":"#BFAF9E"}}}%%`（3 个花括号——因为 JSON 嵌套深度确实需要 3 个，**括号数本身不是判据，JSON 合法性才是**）。

### 既有语法错误（加引号无效，必须用 HTML 实体）

| 图类型 | 破坏解析的字符 | 处理 |
|---|---|---|
| `stateDiagram-v2` | 标签中 `ExecType::kNew` 的 `::` | 转义为 `&#58;&#58;` |
| `sequenceDiagram` | 消息与 `alt`/`else` 标签中的 `;` | 转义为 `&#59;` |
| `flowchart` | 未加引号的 `()`，如 `P[Rollback RoundAmtExchangeRate2(GetExchangeRate(true))]` | 补引号即可 |

## 5. 修复后的验证结果

| 验证项 | 结果 |
|---|---|
| 严格解析（**不剥离 init**，与预览同路径） | **49/49 全部通过** |
| init 的 JSON 合法性 | **49/49 合法** |
| 剔除 Mermaid 块后的正文 | 15 个文件**逐字节零改动**（只改图内内容） |
| 图结构（箭头/参与者计数） | 抽查 5 个修复块计数完全一致，未增删节点或边 |
| 仓库引用门禁（本次改动文件） | 0 问题 |

最终提交 `9419610d` 含 16 个文件：14 个配色/修复文档 + `task/review/check_doc_refs.allow` 豁免清单 + `work_log/20261111.md`。

## 6. 环境限制（影响验证手段的选择）

本机**无法出图做像素级确认**：`~/.cache/puppeteer/chrome/` 下 131.0.6778.204 / 155.0.8059.39 / 157.0.8103.50 三个 zip **全部是截断的损坏下载**（头部魔数 `PK..` 正常，但 `unzip` 报 `cannot find zipfile directory`）；系统只有 `/usr/bin/firefox`，与 puppeteer 不兼容。这是**既有的机器状态，不是本次改动造成的**。

因此验证只能停留在文本级四重证据：**严格解析 + init JSON 合法性 + WCAG 对比度 + 结构计数**。视觉终验需人工在编辑器中打开确认（必要时重载窗口以丢弃缓存的渲染结果）。

## 7. 可复用结论

- Mermaid 预览报 "No diagram type detected"，**先查代码块开头是否有 `%%{init}%%`**，并直接对指令内的 JSON 做合法性校验。
- **不要手工拼 JSON 文本**（尤其是要重复注入 49 次的长字符串）——用序列化器生成并做往返自检。
- **验证路径要覆盖全部输入**；剥离掉"最容易错的部分"再校验会制造假绿。
- 已推送分支上的错误改动，用 `git revert` 而非重写历史。
- 相关工具脚本 `/tmp/mmdval/fix_and_color.py` 为一次性脚本；中途靠零散补丁导致反复失败，**最终整文件重建才通过**——工具脚本局部损坏时应整体重写。
