---
id: "08328407-794a-4cf6-b5be-7719a9ad3a99"
title: "trade_trunk Git 上游迁移：origin 由 server_10 改为 server_11 并推送 dev（2026-11-11）"
description: "解决 2026-11-10 遗留的 origin 无关历史阻塞：用户改设 origin=server_11（空仓库）、本地切到 dev 分支，推送 2 个提交（43851 文件 / 1752 MB，最大单文件 36.9 MB）成功并与 origin/dev 同步 0 0；含推送前三项门禁、换 remote 必须重新探测的教训与仍未推送的 master。"
status: "active"
created_at: "2026-11-11T01:53:23.768Z"
updated_at: "2026-11-11T01:53:23.768Z"
content_hash: "1f0071562077fdd2a2f88c98a78b2a1c1fd58a5dbf55d7c8dacbb18fb80d3f70"
source_paths:
  - "work_log/20261111.md"
  - "work_log/20261110.md"
  - ".gitignore"
session_ids:
  - "f0400b09-cc28-4605-bf64-7b8cbaa5696f"
memory_body_ids:
  []
---

# trade_trunk Git 上游迁移：origin 由 server_10 改为 server_11 并推送 dev（2026-11-11）

> 工作区：`/home/lsz/code/work/trade_trunk`。本文是既有文档 `c6b211b1-0b3d-4f19-9897-1d7ea08ce511`（体系落地，其第 4 节记录了 server_10 推送被拒、结论「先不推送」）的**后续解决篇**，只记上游迁移与推送结论。
> 仓内权威正文：`work_log/20261111.md`（当日日志）、`work_log/20261110.md`、`.gitignore`。

## 1. 结论（现状口径）

| 项 | 值 |
|---|---|
| origin | `git@github.com:lishengze/server_11.git`（fetch/push 均已更换） |
| 推送前远端状态 | **空仓库**：`git ls-remote origin` 无任何引用（无 heads、无 tags） |
| 工作分支 | 本地 `dev`（用户此前已切换），上游跟踪 `origin/dev` |
| 推送结果 | `* [new branch] dev -> dev`，退出码 0；`6519c0c8` 与远端 `refs/heads/dev` 一致 |
| 同步核对 | `git rev-list --left-right --count HEAD...origin/dev` → **`0 0`** |
| 远端现有引用 | 仅 `refs/heads/dev`；`master` **未推送** |
| 本地分支 | `dev`(6519c0c8, 跟踪 origin/dev)、`master`(a98ba3c2)、`backup-local-20261111-0918`(07ee9bb0) |

已推送提交（自旧到新）：`07ee9bb0`（体系建立）→ `a98ba3c2`（复查收尾）→ `6519c0c8`（补记 2026-11-11 工作日志）。

旧 origin `server_10` 是 ADK/AAF 基础库知识库（1373 文件、6 提交、与本地无共同祖先、12 个同名文件冲突），**已弃用**；本地 `backup-local-20261111-0918` 备份分支仍保留，确认稳定后可删。

## 2. 推送前门禁（三项只读核查，均通过）

1. **体积**：43,851 文件 / 1,752 MB；最大单文件 **36.9 MB**（`DYS/ors_impl/tool/encrypt/lib64/libadk.so`），无超过 GitHub 100 MB 上限的文件。
2. **敏感/元数据**：入库内容中 `.svn/` 0 个、`.git/` 0 个；构建产物与大安装包由 `.gitignore` 排除（>80 MB 第三方安装包/依赖/运行期数据不入库，留盘）。
3. **工作区**：干净，无未提交改动。

推送在后台 job 中执行（1.75 GB），完成后按上述同步核对确认。

## 3. 可复用教训

- **更换 remote 后必须重新探测远端，不能沿用上一次的判断。** 本次先例（server_10）是"非空、无关历史、推送被拒"；server_11 则是空仓库，直接推送即为初始内容——两次的处置完全不同。探测顺序：`git remote -v` → 当前分支与本地提交 → `git ls-remote origin`（只读、可设超时）→ `git rev-list --left-right --count HEAD...origin/<branch>` 核对同步。
- **大仓首次推送走后台 job**，不要同步阻塞；推送后必须做一致性断言（本地 HEAD 与远端 ref 同一 SHA，左右计数 0 0）。
- 工作日志按规范落在 `work_log/YYYYMMDD.md`，日期取系统 `date`，跨午夜新建文件而不追加到昨天。

## 4. 遗留项

- `master` 未推送，需要时再推（用户未要求）。
- `/home` 磁盘此前已用约 91%（剩约 22 GB），大体积产物入库策略仍需维持。
