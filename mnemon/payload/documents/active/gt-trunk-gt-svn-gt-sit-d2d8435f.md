---
id: "d2d8435f-21da-4ebb-a207-25a1799cd569"
title: "三仓（gt_trunk/gt_svn/gt_sit）编译与部署口径：构建套件归属与最小构建法"
description: "澄清 gt_svn 并不缺 CMake 构建套件（fte_all + compile_fte_all.sh 属本地适配、不在 SVN 基线），gt_sit 才真缺（仅 bjam）并靠沙箱 rsync + 从 gt_svn 拷 3 个 CMakeLists 编出 ute 与 libutedatainit.so；含 8 子工程结构、7 类部署清单、GT_TRUNK 变量遗留命名、BUILD_ARM 坑与重拉 SVN 后构建套件丢失风险。"
status: "active"
created_at: "2026-11-05T00:54:42.583Z"
updated_at: "2026-11-05T00:54:42.583Z"
content_hash: "bd8d39fba52772335835cad83d0936d44202c3decc6e5dd26ee40f7bea273d22"
source_paths:
  - "DYS-FRAMEWORK/fte_all/CMakeLists.txt"
  - "DYS-FRAMEWORK/fte_all/compile_fte_all.sh"
  - "DYS-FRAMEWORK/fte/CMakeLists.txt"
  - "DYS-FRAMEWORK/fte/fte/CMakeLists.txt"
  - "DYS-FRAMEWORK/fte/ftedatainit/CMakeLists.txt"
  - "DYS-FRAMEWORK/fte/test_all/webui/repos.py"
  - "svn_work/49_三仓构建与部署说明.md"
session_ids:
  - "cf5ea124-292f-44bf-b32c-dc887ad5f826"
memory_body_ids:
  []
---

# 三仓（gt_trunk / gt_svn / gt_sit）编译与部署口径

> 来源：2026-11-05 复测收尾时对「gt_svn、gt_sit 的代码怎么编译部署」的逐项核对（基线清单 + git ls-files + 产物比对）。
> 关联文档：`97e362f5-6c02-45e0-9835-a66136108bcb`（FTE 测试控制台多代码库切换；那里讲控制台怎么调，这里讲构建本身）。
> 仓内成文版：`svn_work/49_三仓构建与部署说明.md`（144 行）。

## 1. 结论先行

- **gt_trunk / gt_svn 不缺编译套件**：聚合工程 `DYS-FRAMEWORK/fte_all/` + `compile_fte_all.sh` 都在，一条脚本完成编译+部署。
- **只有 gt_sit（生产分支 SVN 工作副本）真的缺**：它只有 `Jamroot` / `Jamfile.v2`（bjam 构建），服务器无 bjam；靠从 gt_svn 拷 3 个 CMakeLists 在沙箱副本里补出来。
- **"看不到编译套件"是观察视角问题**：`fte_all/` 与脚本是**本地适配资产，不在 SVN**，用 `svn` 视角查自然查不到；它们提交在 gt_svn 的 git 镜像提交 `8c075dd30`（"提交本地复测全部内容"）中。

## 2. 原件归属判定（关键，避免误判"缺文件"）

| 内容 | 来源 | 证据 |
|---|---|---|
| `DYS-FRAMEWORK/fte/CMakeLists.txt`（125 行）及 `fte/fte/`、`fte/ftedatainit/` 下的 | **SVN 原生**（r87083 基线即有） | 基线清单 `baseline_r87083.txt` 可 grep 到 |
| `DYS-FRAMEWORK/api/gw_api_boost/ute_mock_client/`（含其 CMakeLists） | **SVN 原生** | 同上 |
| `DYS-FRAMEWORK/fte_all/`（整个聚合工程，含 CMakeLists + `.gitignore`） | **本地自建** | 基线里 `grep fte_all` = 0 条 |
| `fte_all/compile_fte_all.sh` | **本地自建** | 同上 |

判定手法：**以 SVN 基线清单（`baseline_r87083.txt`）为准做 grep，而不是以"目录里有没有文件"为准**。

## 3. gt_trunk / gt_svn：一条脚本

```bash
bash DYS-FRAMEWORK/fte_all/compile_fte_all.sh -r                # 编译 + 部署
bash DYS-FRAMEWORK/fte_all/compile_fte_all.sh -r --deploy-only  # 只部署已有产物
```

编译主体：

```bash
cmake -DCMAKE_BUILD_TYPE=Release DYS-FRAMEWORK/fte_all
make -j${CPUS}
```

`fte_all/CMakeLists.txt` 用 `add_subdirectory` 串起 8 个子工程：
`msg_parser`（libmsg_proto.so）、`gw_external_msg/des`、`gw_external_msg/V2`、
`api/gw_api_boost`、`api/gw_api_no_boost`、`fte`（内含 ftedatainit + fte）、
`mock_exch_server`、`fte_perf_test`。

一次 make 产出：`ute`、`libutedatainit.so`、`mock_client`、`mock_exch_server`、`mock_sqlserver`、`fte_perf_capture`。

### 部署（`deploy_to_run`）拷 7 类

| 产物 | 目标 |
|---|---|
| `ute` | `$RUN_DIR/ute` |
| `libutedatainit.so` | `$UTE_LIB_PATH/`（= RUN_DIR；ute 运行期 dlopen 加载） |
| `mock_client` | `$RUN_DIR/ute_mock_client/` |
| `mock_exch_server` / `mock_sqlserver` | `$RUN_DIR/` |
| `mock_exch*.xml` | `$RUN_DIR/` |
| `ute_mock_client/config` + `test_config*.json` | `$RUN_DIR/ute_mock_client/` |
| `etf_test_sh` 等测试数据 | `$RUN_DIR/` |

要点：

- `RUN_DIR` 来自 `env.sh` 的 `$WORK_ROOT/run` —— **同一份脚本，换 `WORK_ROOT` 就是给不同测试根部署**，这是它同时服务 gt_trunk 与 gt_svn 的原因。
- 脚本里的 `GT_TRUNK` 变量是**历史遗留命名**，实际取 `$(cd $SCRIPT_DIR/../..)` = 本仓库根，**没有跨仓依赖**。
- `libutedatainit.so` 必须与 `ute` 同目录（`UTE_LIB_PATH`），否则运行时 dlopen 失败；脚本对变量缺失是直接报错退出。

## 4. gt_sit：沙箱补 CMake 的最小构建

SIT 生产分支无 CMake 工程且无 bjam，做法是**SVN 工作副本零修改，全部改动落在沙箱副本** `gt_sit_src/`：

```bash
# 1) 沙箱副本
rsync -a --exclude=.svn gt_sit/DYS-FRAMEWORK/ gt_sit_src/DYS-FRAMEWORK/

# 2) 从 gt_svn 拷 3 个 CMakeLists
for f in CMakeLists.txt fte/CMakeLists.txt ftedatainit/CMakeLists.txt; do
  cp gt_svn/DYS-FRAMEWORK/fte/$f gt_sit_src/DYS-FRAMEWORK/fte/$f
done

# 3) 编译（本地不要设 BUILD_ARM，设了会走 ARM 库链接失败）
cmake -S gt_sit_src/DYS-FRAMEWORK/fte -B gt_sit_src/build_sit \
      -DCMAKE_BUILD_TYPE=Release -DCMAKE_CXX_FLAGS=-DFTE_NETCARD_TEST
make -C gt_sit_src/build_sit -j10
```

- 顶层只有两个子目录：`add_subdirectory(ftedatainit)` + `add_subdirectory(fte)`。
- **只出两个产物**（实测：`ute` 171184072 B、`libutedatainit.so` 25072000 B），部署就是两条 cp 到 `$WORK_ROOT/run/`。
- SIT **不编译** mock_client / mock_exch_server，这两者复用 gt_svn 那套（`gt_test_sit/run` 由 `gt_test_svn/run` 复制并剔除二进制而来）。
- `fte/CMakeLists.txt` 用 `get_filename_component(... PATH)` 反推 `DYS_FRAMEWORK_DIR`，所以源码树必须保持 `DYS-FRAMEWORK/fte` 的相对布局，单拷 `fte/` 出去编不过。

## 5. 控制台侧的对应关系（细节见关联文档）

`repos.py` 为每个仓库声明 `kind`：

| 仓库 | build | deploy | code_dir | work_root |
|---|---|---|---|---|
| gt_trunk | `script`（compile_fte_all.sh -r） | `script`（--deploy-only） | /home/lsz/code/work/gt_test | 同左 |
| gt_svn | `script` | `script` | /home/lsz/code/work/gt_test_svn | 同左 |
| gt_sit | `cmake_make` | `copy`（两条 cp） | /home/lsz/code/work/gt_sit_src | /home/lsz/code/work/gt_test_sit |

两个已修过的坑（复现即可验证）：构建基准必须是**源码树**而非 work_root；容器内命令须用 `$CODE_ROOT`/$WORK_ROOT` 展开，写死绝对路径会报"目录不存在"。

## 6. 风险与待办

- **gt_svn 的 `fte_all/` + `compile_fte_all.sh` 是本地资产**：一旦从 SVN 重拉干净的 r87083 工作副本（遗留 P0），这套构建工程就消失，需要重建。建议单独存档（如 `gt_trunk/tools/build_kit/`）。
- gt_svn 工作区当时仍有 3 个本地修改未归位：`DYS-FRAMEWORK/Common/CMakeLists.txt`、`Common/Jamroot`、`Common/biz_msg/CMakeLists.txt` —— 属于构建适配，重拉后需重新施加。
- `BUILD_ARM` 在 x86 本地环境必须不设；这是 SIT 编译失败最常见的一步。
