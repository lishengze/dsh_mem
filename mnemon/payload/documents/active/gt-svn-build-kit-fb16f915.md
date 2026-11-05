---
id: "fb16f915-131d-4f7a-889e-b0131072781f"
title: "gt_svn 构建套件存档（build_kit）与远程重建清单"
description: "fte_all + compile_fte_all.sh 属本地适配、不在 SVN 基线；build_kit 只存 2 个特化文件并用 borrow 补齐脚手架；check/restore 脚本 17 项核验与合成干净树验证结果；五层所需内容（构建系统/工程 CMakeLists/依赖子工程/测试脚手架/运行资产）与远程三步操作、svn status 判定法、6 个环境易错点。"
status: "active"
created_at: "2026-11-05T01:20:04.827Z"
updated_at: "2026-11-05T01:20:04.827Z"
content_hash: "3277cb296f895d291641f183b183570dc5b90dc265f6c95543e8335b1dba506b"
source_paths:
  - "tools/build_kit/README.md"
  - "tools/build_kit/check_gt_svn.sh"
  - "tools/build_kit/restore_gt_svn.sh"
  - "tools/build_kit/gt_svn_patch/DYS-FRAMEWORK/fte_all/CMakeLists.txt"
  - "svn_work/50_gt_svn工程所需内容清单.md"
  - "svn_work/49_三仓构建与部署说明.md"
session_ids:
  - "417c45ed-e4b9-46e4-8e70-adb1cd4773b0"
memory_body_ids:
  []
---

# gt_svn 构建套件存档（build_kit）与远程重建清单

> 来源：2026-11-05 收尾时「gt_svn 的 fte_all/compile_fte_all.sh 是本地资产、重拉 SVN 就会丢」这一风险的落地处置。
> 关联文档：`d2d8435f-21da-4ebb-a207-25a1799cd569`（三仓编译部署口径：讲怎么编、怎么部署）；本文讲**丢了怎么补**与**远程机器上要带什么**。
> 仓内成文版：`svn_work/50_gt_svn工程所需内容清单.md`、`tools/build_kit/README.md`。

## 1. 为什么要存档

`DYS-FRAMEWORK/fte_all/`（聚合工程）与 `compile_fte_all.sh` 是**本地适配产物，不在 SVN 基线**（r87083 基线清单里 `grep fte_all` = 0 条），只存在于 gt_svn 的 git 镜像提交 `8c075dd30`("提交本地复测全部内容")。一旦从 SVN 重拉一份干净工作副本，整套 CMake 构建工程就没了。

## 2. 存档结构（tools/build_kit，60 KB，纯 shell、自包含）

```
tools/build_kit/
├── README.md                          为什么需要、用法、判断依据、维护方式
├── gt_svn_patch/DYS-FRAMEWORK/fte_all/
│   ├── CMakeLists.txt                 gt_svn 特化版（与 gt_trunk 版差 30 行）
│   ├── compile_fte_all.sh             gt_svn 特化版（与 gt_trunk 版差 10 行）
│   └── .gitignore
├── check_gt_svn.sh                    17 项逐层核验，列出缺失项并打印修复命令
└── restore_gt_svn.sh                  一键补齐
```

**为什么只存 2 个文件**：其余脚手架（mock_exch_server、fte_perf_test、test_all、ute_mock_client）gt_trunk 里都有原版，restore 脚本用 borrow 从 gt_trunk 拉取，避免仓库重复占体积；而 `fte_all/` 这两个文件是 gt_svn 特化版（源码根路径、依赖目录、部署目标不同），**不能与 gt_trunk 版互换**，必须独立存档。

## 3. 验证结果（存档当时）

| 测试 | 结果 |
|---|---|
| `check_gt_svn.sh` 对真实 gt_svn | 17/17 全 OK |
| 合成"干净 SVN 树"（只有 SVN 原生文件） | 3 OK / 14 MISS，并给出修复命令 |
| `restore_gt_svn.sh` 补齐后复查 | 脚手架全部 borrow 成功，仅剩 4 个真实 SVN 才有的依赖子工程 |

补齐后仍缺的 4 项（只能从 SVN 取）：`DYS-FRAMEWORK/msg_parser`、`gw_external_msg/des`、`gw_external_msg/V2`、`api/gw_api_no_boost`。

## 4. 远程服务器操作三步

```bash
scp -r gt_trunk/tools/build_kit <user>@<host>:/home/lsz/
bash /home/lsz/build_kit/check_gt_svn.sh   /home/lsz/code/gt_svn
bash /home/lsz/build_kit/restore_gt_svn.sh /home/lsz/code/gt_svn /home/lsz/code/gt_trunk
```

先 `svn status DYS-FRAMEWORK/fte_all`（空=SVN 有，`?`=本地后加）、`svn status --no-ignore | grep '^?'` 列出全部本地文件再决定补什么。

## 5. gt_svn 工程所需内容的五层划分

| 层 | 内容 | 在 SVN？ |
|---|---|---|
| ① 构建系统 | `fte_all/CMakeLists.txt`、`compile_fte_all.sh` | ❌ 必须另外带（build_kit） |
| ② 工程 CMakeLists | `fte/CMakeLists.txt`、`fte/fte/`、`fte/ftedatainit/` | ✅ 自带 |
| ③ 依赖子工程 | msg_parser、gw_external_msg(des/V2)、api×2 | ✅ 自带 |
| ④ 测试脚手架 | mock_exch_server、fte_perf_test、test_all、ute_mock_client | ❌ 从 gt_trunk 借 |
| ⑤ 运行资产 | `env.sh`、`etf_test_sh` / `etf_test_sz` | ❌ 同 ④ |

**判定原则**：不要靠基线清单判断"SVN 有没有"——基线只是 149 个文件的**子集**，没收录 ≠ 不在 SVN。以 `svn status` 为准。

## 6. 远程环境易错点

- `BUILD_ARM`：仅某台机器 `lib/x86` 归档有问题时才需要 `export BUILD_ARM=1`，是机器特例、不是通用要求。
- 构建缓存过期（build 目录跨机器复用）会表现为链接失败，先清 build 目录。
- `libutedatainit.so` 必须与 `ute` 同目录（`UTE_LIB_PATH`），否则运行期 dlopen 失败。
- `env.sh` 里 `BOOST_ROOT` / `ATP_RES_ROOT` / `MP_RES_ROOT` 需改成本机实际路径（容器内外不同）。
- `FTE_NETCARD_TEST` 编译宏影响网卡相关路径；`NO_DSE` 会丢弃线程相关代码。
- `test/mock_exch_server` 在仓库根而非 `DYS-FRAMEWORK/` 下，`fte_all/CMakeLists.txt` 用 `${DYS_FRAMEWORK_DIR}/../test/mock_exch_server` 引用。
