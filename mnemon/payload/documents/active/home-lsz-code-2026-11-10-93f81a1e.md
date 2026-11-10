---
id: "93f81a1e-5e16-4023-a414-28b68fcebbaa"
title: "/home/lsz/code 大文件清理盘点与风险分级（2026-11-10，只读审计）"
description: "/home/lsz/code 下 3044 个 >10MB 文件的四类分级（A 编译中间产物 / B 日志 / C VCS 元数据 / D 其他）、可删性风险规则（内存库 redolog、.svn-base/.pack 严禁删除）、方案与回收量估算、待用户确认项；配套统计 MD 与明细 CSV 路径。"
status: "active"
created_at: "2026-11-10T10:28:20.124Z"
updated_at: "2026-11-10T10:28:20.124Z"
content_hash: "abbc85134ac7bc2ee603e6c806994b79253e06a2a6a388c042bb447313dd1219"
source_paths:
  - "code_build_log_cleanup_stat_2026-11-10.md"
  - "code_build_log_cleanup_detail_2026-11-10.csv"
session_ids:
  - "d89e8947-a27f-4d8d-be25-6adb0ab8c379"
memory_body_ids:
  []
---

# /home/lsz/code 大文件清理盘点与风险分级（2026-11-10）

只读审计，**未执行任何删除**（复核时刻 `/home` 仍为 3.7 GB 可用 / 223 GB 已用 / 99%）。目的：在执行清理前把「看起来能删」和「删了会出事」的边界定清楚。

- 统计文档：[code_build_log_cleanup_stat_2026-11-10.md](code_build_log_cleanup_stat_2026-11-10.md)（1627 行，含逐条明细）
- 明细清单：[code_build_log_cleanup_detail_2026-11-10.csv](code_build_log_cleanup_detail_2026-11-10.csv)（3044 条数据行 + 表头）

## 1. 分级结果

阈值：单文件 > 10 MB。共 3044 个文件，分四类：

| 类别 | 构成 | 文件数 | 占用 | 是否清理范围 |
| --- | --- | ---: | ---: | --- |
| A 编译中间产物 | `.o` `.make` `.gch` `.pch` `.ipch` | 1270 | 21.4 GB | 是（回收主体） |
| B 日志 | `.log` | 120 | 7.2 GB | 需分级，见 §2 |
| C VCS 元数据 | `.svn-base` 280（17.1 GB）、`.pack` 41（7.4 GB）、`.idx` 1 | 322 | 24.4 GB | **严禁删除** |
| D 其他大文件 | `.so` 13.7 GB、无扩展名可执行 9.4 GB、`.xml` 7.4 GB、`.zip` 6.8 GB、`.jdb` 3.8 GB、`.lib` 2.9 GB 等 | 1332 | 54.9 GB | 本次不动 |

## 2. B 类日志三级风险（关键结论）

| 分级 | 说明 | 文件数 | 占用 | 处置 |
| --- | --- | ---: | ---: | --- |
| D1 | 内存库 redolog / 上场数据 / 测试数据集 | 68 | 6.1 GB | **禁止删除** |
| D2 | 服务运行日志（`bin/logs/`、`bin/global_logs/`） | 29 | 290.0 MB | 近 30 天保留，更早可选删 |
| D3 | 测试日志（test、bug_log、mock、api_log） | 23 | 855.3 MB | >30 天建议删除 |

B 类中**真正可删的只有 621.9 MB**；6.6 GB 必须保留。即回收空间几乎全部来自 A 类。

### 必须遵守的两条删除禁令

1. `memory_database/data/*redolog*.log` 是**内存数据库重做日志（redo log）**，属数据库数据文件而非运行日志；删除可能导致数据库无法恢复/损坏。同类还有 `online_data/`、`test/data/` 下的 `start_run.log`、`kgbp_gather*.log`、`log_UTE_*.log` 等上场数据。
2. `.svn-base`（SVN pristine）与 `.pack`/`.idx`（Git 对象包）是版本控制元数据，删除直接损坏工作副本。已从清理候选自动排除。
3. 末次校验：危险扩展名（`.d`/`.mod`）被误判为编译产物的数量 = 0；抽查 A 类最大文件确为 CMake 生成的 `depend.make`（`# CMAKE generated file: DO NOT EDIT!`）。

## 3. 预计回收（三种方案）

| 方案 | 内容 | 预计回收 |
| --- | --- | ---: |
| 保守 | A 类 + B 类仅 D3 且 >30 天 | 21.8 GB |
| 推荐 | A 类 + B 类 D2/D3 >30 天（D1 不动） | 22.0 GB |
| 激进 | A 类 + B 类除 D1 外全部 | 22.5 GB |

C 类、D 类在任何方案下都不触碰。

参考（本次未处理）：仅 `work/gt_trunk/build` 目录整体 31 GB，其下 >10MB 中间产物仅 11.5 GB；`gt_trunk/DYS/build` 3.6 GB、`DYS-FRAMEWORK/biz_framework/te/business/rzrq/build` 4.5 GB。若改按目录整体清理 build 输出，回收量会显著大于按文件清理。

## 4. 待用户确认（截至审计结束时未答复）

1. A 类 21.4 GB 是否全部清理（建议：全清）。
2. B 类选哪个方案（建议：方案二，D1 一律保留）。
3. 最近 7 天内的 39 个中间产物（约 1.1 GB）是否保留（建议：保留以免重复编译）。
4. D 类 54.9 GB 本次不动。

确认后才进入删除执行阶段；删除动作未在本会话发生。

## 5. 关联文档

- 构建与部署口径（gt_trunk/gt_svn/gt_sit，build 目录与产物归属）：`d2d8435f-21da-4ebb-a207-25a1799cd569`
- NewAPI mock 侧日志与产物背景：`64592da6-cecb-4bb5-ab4a-5ce1124761b0`
