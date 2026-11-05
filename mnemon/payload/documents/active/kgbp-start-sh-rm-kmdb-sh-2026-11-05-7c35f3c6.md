---
id: "7c35f3c6-2386-4c59-8c16-6ee1fe06b62f"
title: "KGBP 内存库清理职责拆分：start.sh 不再删库 + 新增 rm_kmdb.sh（2026-11-05）"
description: "otcsrc 四节点脚本改造记录：start.sh 全场景不再删除内存库落盘与 memdb 信号量，清理收敛到新 rm_kmdb.sh（6 条路径 + 信号量 + 审计日志），含回归证据、AS 信号量 _1.._N 编号陷阱、kill -9 后 32006 为既有 SysV 残留缺陷、共享 bin 部署风险与文档同步范围。"
status: "active"
created_at: "2026-11-05T07:30:53.506Z"
updated_at: "2026-11-05T07:30:53.506Z"
content_hash: "8af17cb587908e4f07ed324671f2f73faa4314ce9b188cd3090992c53423012d"
source_paths:
  - "otcsrc/kgbp_x64_linux/bin/start.sh"
  - "otcsrc/kgbp_x64_linux/bin/rm_kmdb.sh"
  - "otcsrc/kgbp_arm/bin/rm_kmdb.sh"
  - "otcsrc/kgbp_arm_global/bin/rm_kmdb.sh"
  - "study/knowledge_base_system/05_部署.md"
  - "study/knowledge_base_system/07_环境与排障.md"
  - "work_log/20261105.md"
session_ids:
  - "1b1cd9da-f0a3-4738-a067-71e3d8aa677d"
memory_body_ids:
  []
---

# KGBP 内存库清理职责拆分：start.sh 不再删库 + 新增 rm_kmdb.sh（2026-11-05）

面向 otcsrc 四个部署节点（x64 业务 `kgbp_x64_linux`、x64 全局 `kgbp_x64_linux_global`、arm 业务 `kgbp_arm`、arm 全局 `kgbp_arm_global`）的一次脚本改造，目标是让「停/启」绝不丢内存库数据，把清理收敛为一个显式运维动作。相关提交审阅基线见 `27e439e6-262d-4c7b-82ee-669d9e65375a`。

## 1. 需求与落地方式

| 需求 | 实施 |
|---|---|
| start.sh **任何命令场景都不删内存库** | 删除 `rm_memdb_data_file()`、`rm_memdb_shared_file()`、`get_memdb_name()` 三个函数（约 130 行）；`clean_environment()` 去掉 `clean_memdb_data` 参数与内存库分支，调用点统一为无参；头部注释、两处 help、stop 分支注释同步改写。4 节点全部改造 |
| 删除功能补进同级 `rm_kmdb.sh` | 原脚本仅 arm 两节点各约 30 行（只 `rm -rf ./kmdb`），x64 根本没有。新版约 350 行，承载原 `rm_memdb_data_file()` + `rm_memdb_shared_file()` 的全功能 |
| 回归 | 业务节点矩阵 20/20 PASS、全局节点矩阵 18/18 PASS（含跨节点隔离用例） |

停止/启动语义现在明确为：`stop` / `stop-init` / `restart` / `status` **只保留**落盘数据与 memdb 信号量；需要冷启动时显式 `stop` → `rm_kmdb.sh -y` → `start` 重建。

## 2. rm_kmdb.sh 设计要点

- **落盘 6 条路径**（与旧 `rm_memdb_*_file()` 等价，`find … -mindepth 1 -delete` 保留目录本身）：
  - x64：`../memory_database/data`、`../memory_database/arhievelog`（业务与全局各自相对 `bin/` 解析）
  - arm 业务：`bin/kmdb`、`bin/kmdb/arhievelog`
  - arm 全局：`bin/kmdb_global`、`bin/kmdb_global/arhievelog`
- **memdb 信号量**：从 `../conf/kgbp.xml` 读 memdb 名与 AS 数，枚举 `/dev/shm/sem.<memdb名>.init` 与 `sem.<name>_<N>.init`。
- **UI/安全**：`-n` 预览（dry-run）、`-y` 免确认、`-f` 强制、`-h` 帮助；检测到本节点 KGBP 在跑时默认拒绝（exit=2）；非交互无 `-y` 也拒绝；重复执行提示「无需清理」（幂等）。
- **审计**：日志写 `~/log/rm_kmdb_<YYYYMMDD>.log`，并追加同级 `stop_delete.log`，与旧 `start.sh` 删除动作同文件，运维查询路径不变（第一轮复查发现的唯一功能缺失就是这条）。
- 四个节点的 `rm_kmdb.sh` md5/哈希完全一致，靠分发而非各改一份。

## 3. 关键坑（值得复用的经验）

1. **AS 信号量编号从 `_1` 起，不是从 `_0`**：初版按 `as_num` 从 0 推算，实测为 `sem.2mdb.init` + `_1.._4` 共 5 个，推算式会漏删 `_N`。正解是按实际文件名枚举，而不是按数量推导。
2. **前缀匹配会误伤**：`sem.2mdb` 的规则必须不匹配 `sem.2mdbglobal`，靠完整 `.<name>.` / `.<name>_<N>.` 形式约束。
3. **`kill -9` 后重启的 32006 是既有缺陷，不是本次引入**：报 `Kocadb Start 2mdb Failed: Failed to create database shared memory`，根因是 **SysV 共享内存段残留**（nattch=0），与落盘数据、POSIX 信号量无关。把改造前脚本从 git 取出在同目录跑同一场景，双方都报 32006，行为一致。`killipc.sh -f` 只清 POSIX 信号量（15.8.1 起有意移除 `ipcrm -m` 防误删），恢复须手工 `ipcrm -m <shmid>`。
4. **同目录共享 bin 的部署风险**：两节点若部署在同一目录，`rm_kmdb.sh` 会清掉对方的内存库数据文件；生产部署必须目录隔离（此条已写入知识库 01_系统架构）。
5. **回归矩阵用例间必须彻底 reset**：kill 进程 + `ipcrm -a` + 清 `/dev/shm`，否则上一用例的信号量/共享内存残留会污染下一个用例的判定。

## 4. 验证证据（可复现）

- 4 节点 `grep` 内存库删除代码命中数 = **0**；哨兵法：在 `memory_database/data`、`arhievelog` 放哨兵文件，跑 `stop`/`stop-init`/`restart` 后全部存活。
- 信号量枚举实测：启动后 `/dev/shm` 有 `sem.2mdb.init` + `_1.._4`，`rm_kmdb.sh -n` 全部 5 个命中、无遗漏。
- 跨节点隔离用例（G8）：全局节点运行中清理业务节点，全局信号量（5）与落盘（25 项）零变化。
- 审计实测：造数据 → `rm_kmdb` 后 `stop_delete.log` 有 2 条目录清空记录、落盘归零、重建后恢复 25 项。
- 4 节点 `bash -n` 语法通过、权限 775、`clean_environment` 四节点同文（同哈希）。

## 5. 同步的仓库内文档

`study/knowledge_base_system/` 下 6 篇 + 当日工作日志：`05_部署.md`、`07_环境与排障.md`（含「删 kmdb：rm -rf ./kmdb」旧指引改写为 `./rm_kmdb.sh -y`）、`06_测试.md`、`01_系统架构.md`（两节点端口/节点 ID/内存库路径必须不同，共享 bin 会互清数据）、`MEMORY.md`、`Prompt.md`、`work_log/20261105.md`。

改造前知识库里把「启动自动清库」写成了正确行为，共 7 处旧描述会误导后续操作——**改脚本必须同步改文档**，否则下一个会话照旧文档操作会再次丢数据。
