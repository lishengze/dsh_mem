---
id: "bc5440de-3af8-4dfb-85b5-2bf7ee1674bc"
title: "Excel sheet 排序约定与 svn 提交命令覆盖核对（2026-11-10）：4 开头业务优先、每个改动文件只出现在一组"
description: "两项可复用约定：①templata.xlsx / template_1.xlsx 的 sheet 顺序（Sheet1 → 4 开头业务按功能号升序 → 其余升序）固化在两个生成脚本的 BIZ 列表而非手工拖拽；②task_svn.txt 的 svn commit 命令分组必须「并集=svn status 的 M 集合、组间零重复」，411833 因依赖框架头改并组4 的判定理由与集合核对脚本写法。"
status: "active"
created_at: "2026-11-10T04:42:08.799Z"
updated_at: "2026-11-10T04:42:08.799Z"
content_hash: "e90475933dfde4caafa1db81331a586d2fc3305b7852bdec973bd1331a7b51bd"
source_paths:
  - "study/excel/build_param_xlsx.py"
  - "task/params_stats/build_stats_xlsx.py"
  - "task/svn_conflict/task_svn.txt"
  - "study/excel/templata.xlsx"
  - "task/params_stats/template_1.xlsx"
  - "work_log/20261110.md"
session_ids:
  - "b163d7a9-a6e9-4274-9798-f0e9b7a9daf9"
memory_body_ids:
  []
---

# Excel sheet 排序约定与 svn 提交命令覆盖核对（2026-11-10）

> 工作副本 `/home/lsz/code/work/otc_src`。两个产物：sheet 顺序统一 `6b95cd3f`、提交命令覆盖校正 `787e7266`（均已 push `origin/dev`）。
> 相关既有文档：`a09a0e67-6e2b-4e64-a153-999429b6372d`（两个 Excel 的入参去重规则与校验，本文不重复）、`5d13fd14-8697-419b-8332-de3bebc6094a`（4 开头 14 业务公共入参复用落地，含组3/组4 待提交清单）、`8f722b2a-1a8e-4954-be70-20d895ae7be9`（重复入参的代码层依据）。

## 1. 两个 Excel 的 sheet 排序约定

用户口径：**两个 Excel 的 sheet 顺序要一致，且 4 开头的业务排在前面。**

最终顺序（22 个 sheet，两文件逐项相等）：

```
Sheet1（模板/说明页）
── 4 开头业务 14 个，按功能号升序 ──
410233 410497 410553 410567 410685 411535 411827
411828 411830 411833 411834 412230 412231 412232
── 其它业务 7 个，按功能号升序 ──
240178 240690 811509 811523 811526 811528 831509
```

**关键做法：顺序写进生成脚本，不手工拖 sheet。** 两个生成器的 `BIZ` 列表都改成「4 开头升序 → 其余升序」，再**用脚本重建 workbook**，使顺序与内容来自同一份可复现源，避免下次重建回退：

- `study/excel/build_param_xlsx.py` → `study/excel/templata.xlsx`
- `task/params_stats/build_stats_xlsx.py` → `task/params_stats/template_1.xlsx`

校验：`openpyxl.load_workbook(...).sheetnames` 两文件逐项相等；前 14 项全部以 `4` 开头；重建后 13 个业务的重复参数名计数仍为 0、内容行数未丢（行数示例 78 / 86，均无重复）。

坑：WPS/Excel 打开着文件时会留锁文件（如 `task/params_stats/.~template_1.xlsx`），重建后需用户关闭再打开才能看到新顺序与内容。

## 2. svn 提交命令分组的铁律与核对法

`task/svn_conflict/task_svn.txt` 里的 `svn commit` 命令按「组」切分（组1 整型默认值守卫、组2 字符型默认值守卫、组3 公共入参复用、组4 411833 g_* 去除）。要求：

1. **并集自洽**：所有组列出的文件路径并集 == `svn status otcsrc/lbm | grep '^M'` 的集合（0 遗漏、0 多余）。
2. **组间零重复**：同一文件只能出现在一组。若文件同时属于两个语义组，按**依赖顺序归入后需执行的那组**，并在注释里说明为何不重复列出。
3. 命令中每条路径做存在性校验，并把命令拼接后用 `bash -n` 做语法检查。

**本轮发现的缺陷（已修）**：`per_cust_risk_ctrl_set_processor_411833.cpp` 同时出现在组3 和组4——组3 先提交后，组4 再指定该文件会「无内容可提交」而报错。判据是它的三块改动（custid/trdpwd 复用、6 个 `g_*` 去除、设置校验位语义恢复）**都依赖框架头 `base_processor.h` 改动**，因此统一并入组4，组3 收缩为 `query 9 + counter 3 = 12` 个文件。

修正后规模：组1 16 / 组2 15 / 组3 12 / 组4 3（411833 业务 + `lbm_otc_param/include/base_processor.h` + `param_param.h`），并集 30 == 实际 M 30。本轮源码改动 15 文件（`a460cfab` + `a86d0cea`）恰等于组3(12)+组4(3)。

不属于本批、未列入任何组：`otcsrc/kgbp_arm/conf/kgbp.xml`（Kafka topic 统一轮次）。
提交后自检：`svn status otcsrc/lbm | grep '^M'` 应为空。

## 3. 复查纪律

文档/命令改动本身也要复查到「零修改轮」才算完成：sheet 顺序做了 3 轮复查，提交命令做了覆盖集合比对 + 语法/路径校验 + 与实际 diff 的一次性复核（`组2+3+4=30 / 实际M=30 / 遗漏无 / 多余无 / 组间重复无`）。SVN 提交仍由用户手动执行，脚本只负责文档与命令清单。
