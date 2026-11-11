---
id: "3fff2be4-655c-4416-973f-77c5cf229cc1"
title: "rm_kmdb.sh 默认改为「强制清理、不交互确认」（2026-11-11，含 -c 新增与 4 副本同步）"
description: "otcsrc 内存库清理脚本 rm_kmdb.sh 于 2026-11-11 的行为修订：默认直接强制清理不确认、新增 -c 人工确认、-y/-f 降级为兼容别名；含沙箱逐项实测证据、知识库 5 文件同步范围、SVN 仅跟踪 arm 两份的提交边界，并说明其取代 2026-11-05 文档中的旧默认口径。"
status: "active"
created_at: "2026-11-11T01:36:14.337Z"
updated_at: "2026-11-11T01:36:14.337Z"
content_hash: "8acbe79622702fa923990bc83ccc2456c85436470aa3867499552939b2d8ed46"
source_paths:
  - "kgbp_x64_linux/bin/rm_kmdb.sh"
  - "kgbp_x64_linux_global/bin/rm_kmdb.sh"
  - "kgbp_arm/bin/rm_kmdb.sh"
  - "kgbp_arm_global/bin/rm_kmdb.sh"
  - "study/knowledge_base_system/07_环境与排障.md"
  - "study/knowledge_base_system/MEMORY.md"
  - "task/svn_conflict/task_svn.txt"
session_ids:
  - "0aab0078-0c6d-41c2-865b-83ab7f540276"
memory_body_ids:
  []
---

# rm_kmdb.sh 默认改为「强制清理、不交互确认」（2026-11-11）

用户原话诉求：「对 `rm_kmdb.sh` 的优化有些需要改进的地方，默认执行的时候直接进行强制清理，不要交互式确认。」
这是对 2026-11-05 职责拆分（`7c35f3c6-2386-4c59-8c16-6ee1fe06b62f`：start.sh 任何命令不删库 + 清理收敛到 rm_kmdb.sh）的一次**默认口径修订**。该 2026-11-05 文档中「非交互无 `-y` 也拒绝」「有进程即拒绝」的描述已被本次改动取代；如需该脚本的完整落盘 6 路径与 AS 信号量编号陷阱等设计背景，仍读那份文档。

## 1. 变更前后对照

| 用法 | 2026-11-05 版 | 2026-11-11 起 |
|---|---|---|
| 默认（无参数） | 交互确认；无终端必须 `-y`，否则拒绝 | **直接强制清理、不确认**，即使本目录有 kgbp 运行也继续（仅 WARNING） |
| `-y / --yes` | 跳过确认 | **兼容别名**（等同默认），旧运维脚本无需改写 |
| `-c / --confirm` | 无 | **新增**：清理前人工确认；无终端直接拒绝并 `exit 1` |
| `-n / --dry-run` | 仅预览 | 不变 |
| `-p / --protect` | 有进程即拒绝（exit=2） | 不变（现在唯一的「保守」入口） |
| `-f / --force` | 强制 | 兼容保留（等同默认） |
| 审计日志字段 | `dry_run= force= assume_yes=` | 增加 `confirm=`，可区分是否人工确认过 |

实现方式：新增 `CONFIRM=0` 变量与 `-c|--confirm` 分支，把原「`ASSUME_YES=0` 且非 dry-run 就进确认流程」改为「`CONFIRM=1` 才进确认流程」，无终端时拒绝；`usage()`、头部注释、未知参数提示里的用法行同步改为 `[-n] [-c] [-p] [-y] [-f] [-h]`。

## 2. 副本与「改一处等于改四处」

内容完全相同的 4 份副本（改后 md5 一致 `1fced645`，`bash -n` 全通过）：

- `kgbp_x64_linux/bin/rm_kmdb.sh`、`kgbp_x64_linux_global/bin/rm_kmdb.sh`
- `kgbp_arm/bin/rm_kmdb.sh`、`kgbp_arm_global/bin/rm_kmdb.sh`

提交边界不对称，改脚本时必须两边都看：**SVN 只跟踪 arm 两份，x64 两份不在 SVN 工作副本内（`svn status` 报 W155007 not a working copy），随 git 走**。

## 3. 实测证据（沙箱 /tmp/kmdbtest，未碰真实内存库）

| 场景 | 期望 | 实测 |
|---|---|---|
| 默认执行 + stdin 非终端（模拟无人值守） | 不提示、直接清空、exit 0 | 2 个目录清零，剩余文件 0，exit 0 |
| `-n` | 只预览不删 | 剩余 1 项保留，exit 0 |
| `-c` + 非终端 | 拒绝、exit 1、不删 | 打印「-c 需要终端交互」，剩余 1 项，exit 1 |
| `-y` | 兼容删除 | 清空 1 个目录（1 个空目录跳过），exit 0 |
| 真实节点 `kgbp_x64_linux/bin/rm_kmdb.sh -n` | 预览范围正确 | data/arhievelog + `/dev/shm/sem.2mdb{,_1.._4}.init` 共 4 个信号量，未删除，exit 0 |

注意：`-c` 无终端时脚本打印错误后**显式 `exit 1`**，判定要以退出码为准，不要只看 stdout（首次测试输出里错写过「退出码=0」，复测已纠正为 1）。

## 4. 文档同步范围

知识库 5 个文件共 9 处，把标准操作从 `rm_kmdb.sh -y` 改为 `rm_kmdb.sh`，并写明 `-c` 新语义：

- `study/knowledge_base_system/Prompt.md`（1 处）、`05_部署.md`（1 处）、`06_测试.md`（1 处）、`07_环境与排障.md`（3 处）、`MEMORY.md`（2 处）。

刻意**不改**历史复盘/案例文档中的 `-y` 用法：命令仍然有效（兼容别名），改写历史记录反而失真。

## 5. 顺带的 SVN / Git 状态

- 组5（240178 源码 + 6 个 query 业务）已由用户提交为 **r77842**，覆盖核对后剩余待提交 = 组2(15) + 组6(2) = 17 个 M；组6 即 arm 两份 `rm_kmdb.sh`，已写入 `task/svn_conflict/task_svn.txt`。
- git 侧 `0b3a105f`（脚本 + 文档 + 组6 记录）、`200c1b96`（SVN 状态刷新）已 rebase 后推送 `origin/dev`，与远程 0/0 同步。

## 6. 可复用教训

1. 运维脚本的「默认最危险动作」是刻意设计的结果，改动时要同时改 `usage()`、头部注释、错误提示与日志字段——四处漏改任何一处，运维人员就会按文档误判行为。
2. 保留 `-y`/`-f` 作兼容别名、新增 `-c` 而不是复用 `-y` 反向语义，能让存量脚本与文档继续可用，避免大面积改写。
3. 多副本脚本先比对 md5 再改，改后再次比对 md5，是防止「只改了 4 份中的 3 份」的最省事手段。
