---
id: "ff632d4e-d340-4c5c-9b3b-d0ab4b9efcf1"
title: "每日工作规划模板（work_task）：结构、内联样式约束与验证流程"
description: "work_task 每日规划模板 v1 的存放约定、四段结构、13 时段/6 列/下拉控件基线，以及 Markdown 预览内联样式约束、无头浏览器验证流程与已知局限。"
status: "active"
created_at: "2026-11-04T02:48:38.249Z"
updated_at: "2026-11-04T02:48:38.249Z"
content_hash: "313c52e461e9a8d4232685b6d4e63d26a9d0a94ad5af6b370b5e33f489cd38f9"
source_paths:
  - "work_task/template.txt"
  - "work_task/2026/11/20261104.md"
session_ids:
  - "deda77ac-6d65-4669-af9c-77ff114d4bd8"
memory_body_ids:
  []
---

# 每日工作规划模板（work_task）设计与落地约束

工作区：/home/lsz/code/work/work_store。本文记录 2026-11-04 生成"每日工作规划"可交互 Markdown 模板的设计结论与验证方法，供后续按日生成/修改模板时复用。

## 1. 产物与存放约定

- 模板源：`work_task/template.txt`（约 1.5 KB，结构骨架）
- 当日文档：`work_task/<YYYY>/<MM>/<YYYYMMDD>.md`，按"年/月"两级目录存放，例如 2026-11-04 → `work_task/2026/11/20261104.md`
- 首份文档约 35 KB（UTF-8，含大量内联样式，行很长属正常）

## 2. 文档结构（v1）

1. 标题 + 星期 + 领域副标题
2. **一、目标设置**：工作 / 学习 / 生活 / 育儿 四行，每行"今日目标""预期困难"两个输入框
3. **二、今日时间记录**：09:00–23:00 共 13 行，11:00–13:00 合并为"午间合并"时段；固定 6 列：时间、类型、工作性质分类、计划事项、实际经历、经验教训；类型（工作/学习/生活/育儿）、工作性质（重要紧急/重要不紧急/不重要紧急/不重要不紧急）、满意度（1–5 星）为下拉选择
4. **三、分类总结**：四个领域各一组"满意度下拉 + 大文本框"
5. **四、今日总评**：用户自我总结（下拉 + 文本框）与 AI 总结（文本框）
6. 尾部 HTML 注释说明：表单控件状态不会写回 .md，长期记录请直接编辑 Markdown 源文本

实测控件规模（用作回归基线）：47 个 input、44 个 select、6 个 textarea、233 个 option。

## 3. 关键实现约束（踩过的坑）

- **必须用内联 style**：Markdown 预览器普遍会剥离 `<style>` 块或其中的选择器（Typora 需走"自定义 CSS/主题"，VS Code 预览对内联 `<style>` 支持不稳定），因此每个控件都写成带 `style="..."` 的内联属性；校验时确认 `style` 属性全部可解析、无空值。
- **不要给 select 写 width**：显式 width 会挤压列宽，应改用各列自己的 min-width；同时避免残留的 `background`、`font-family` 覆盖（最终检查 backgrounds left: 0）。
- **时间列不换行**：时段用 `<span style="white-space:nowrap">` 包裹，合并时段另加小字"午间合并"。
- **列宽校验**：目标是表格宽度 == 内容宽度（860px），6 列约 89/101/121/182/182/182；重建后 tr/td 上不应再有内联样式残留（应全部收敛到 th/td 的统一样式）。
- **表格内 HTML 必须成对**：曾出现 `style="...;;"` 之类的重复分号，重建后以"double semicolon: 0"作为自检项。

## 4. 验证流程（可复用）

1. 生成/重建 .md（脚本方式批量替换表格行，避免手写 13 行 HTML）。
2. 用无头浏览器打开本地 file:// 预览（ego-browser，见 dsh-browser-cdp 运行时）截图并读取 DOM 统计。
3. 结构化自检脚本输出：sections / goal_rows（工作·学习·生活·育儿）/ time_rows（13，首行 09:00–10:00，末行 22:00–23:00）/ 六列 / textareas（6）/ rating 选项 / merged_lunch / ai 与 user_summary 段落 / 文件字节数与路径。
4. 交互与布局改动后重新加载页面，重新量列宽与行数再截图，确认渲染没有回退。

浏览器坑：ego-browser 的表达式求值包装不接受多语句（`window.scrollTo(0, 900); ({...})` 会报 SyntaxError: Unexpected token ';'），滚动请用独立语句或 IIFE 返回单个表达式；滚动后再截图才能看到下半部分表格。

## 5. 已知局限

- Markdown 预览里的下拉框/输入框填写内容不会写回源文件，不适合真正日常记录；若要可保存，需改为 HTML 版（localStorage 自动保存 + 导出 Markdown）或纯 Markdown 可直接编辑版本。
- 该文档为空白模板；用户可直接把当日目标/事项发来代填并生成 AI 总结。

## 6. 参考

- 模板文件：`work_task/template.txt`
- 首份产物：`work_task/2026/11/20261104.md`
- 预览截图：`/tmp/daily_preview_final_top.png`、`/tmp/daily_preview_v3.png`
