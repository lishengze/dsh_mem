---
id: "f3324711-b8f7-477e-94c5-3d06a1ee3d6d"
title: "gt_sit mock_client double free 根因复核：boost 1.68/1.62 混链（2026-11-08）"
description: "复核三仓独立改造收尾结论，用 ldd 证据推翻「agw 库 ABI 不配套」判断，定位 double free 真因为 build_mock_client.sh 的 boost 路径写错导致 1.68/1.62 混链；含修正后重编验证、文档/脚本不一致清单。"
status: "active"
created_at: "2026-11-08T16:05:48.818Z"
updated_at: "2026-11-08T16:05:48.818Z"
content_hash: "0304b9f88d9252b335fa3b6d50145e92d856435934c3c684996759682f1a8ac2"
source_paths:
  - "DYS-FRAMEWORK/fte/test_all/build_mock_client.sh"
  - "DYS-FRAMEWORK/fte/test_all/verify_branch_isolation.sh"
  - "DYS-FRAMEWORK/fte_all/compile_fte_only.sh"
  - "DYS-FRAMEWORK/fte/test_all/env.sh"
  - "svn_work/66_三仓完全独立改造记录_20261108.md"
session_ids:
  - "25fc27ac-693f-4731-9768-8351ef6fd5f8"
memory_body_ids:
  []
---

# gt_sit mock_client `double free` 根因复核：boost 1.68 / 1.62 混链

> 复核对象：三仓（gt_trunk / gt_svn / gt_sit）编译-部署-测试完全独立改造的收尾结论，
> 以及其中唯一未闭环项「gt_sit mock_client 运行期 double free」。
> 复核时间：2026-11-08 夜。相关既有文档 id：`e3defbad-a13b-483d-afdb-df64e78c6be0`（部署隔离改造）、仓内 `svn_work/66_三仓完全独立改造记录_20261108.md`。

## 1. 结论先行

仓库内文档 §4.5.7 / §4.5.8 给出的判断（"自建 libagw_ute_api.so 与 dse_deps 预编译 libagw_external_msg.so 报文结构体布局不一致"）**证据不足**，不是主因。

实测主因：**gt_sit 的 mock_client 由 `build_mock_client.sh` 编译时，boost 头/库路径写错，静默回退到系统 boost 1.68；该进程因此同时装载 boost 1.68（9 个）与 boost 1.62（12 个）两套同名库**，退出期重复释放是必然结果。

对照证据：gt_svn 的 mock_client 与它使用的 `libagw_external_msg.so`（md5 `51a9635c90a4…`，与 gt_sit 部署的是**同一个文件**）在同一次 4/4 通过里共同工作 —— 说明该 agw 库本身没问题，配错的只有 mock_client 自身。

## 2. 证据

### 2.1 混链事实（容器 otc 内 ldd）

`/mnt/work/gt_test_sit/run_sit/ute_mock_client/mock_client`：

- boost 1.68（`/usr/local/lib`）：system / thread / chrono / filesystem / regex / date_time / log / log_setup / program_options / atomic
- boost 1.62（`/mnt/work/gt_test_sit/work_lib/AMI/lib64`）：filesystem / program_options / date_time / system / locale / log_setup / log / thread / regex / chrono / atomic

即 system、thread、chrono、filesystem、regex、date_time、log、log_setup、program_options、atomic 十个模块在**同一进程内各装了两份不同版本**。

对照 `/mnt/work/gt_test_svn/run_svn/ute_mock_client/mock_client`：boost 只有 1.62 一套（来自 `work_lib/3rd/boost_1_62_0/stage/lib`），无 1.68。

### 2.2 根因在构建脚本的两处路径

`DYS-FRAMEWORK/fte/test_all/build_mock_client.sh`（gt_sit）：

| 脚本里写的 | 实际布局 | 后果 |
|---|---|---|
| `-I"${B}/include"` | `${B}/include` **不存在**（头在 `${B}/boost`，即 `-I${B}`） | 找不到 1.62 头 → 用 `/usr/local/include` 的 1.68 头 |
| `-L"${B}/lib"` | `${B}/lib` **不存在**（库在 `${B}/stage/lib`） | `-lboost_*` 落到 `/usr/local/lib` 的 1.68 动态库 |

其中 `B=${WORK_ROOT}/work_lib/3rd/boost_1_62_0`。且链接命令尾部带 `2>/dev/null`，把"找不到 1.62 库"的告警吞掉，问题被完全隐藏。

### 2.3 修正路径后重编的验证

同样的源码与链接库清单，只把两处路径改对（`-I${B}`、`-L${B}/stage/lib`，另加 `-L${AMI}/lib64`），产物 `/tmp/review_mc/mock_client`：

- 编译错误 0，链接 rc=0，二进制 341,480 B（原坏件 342,824 B，同量级）
- `ldd` 中 boost 1.68 计数 = **0**，全部 11 个 boost 模块均为 1.62
- `--help` 正常输出并干净退出（rc=0，无 double free）

> 备注：运行期仍需 `LD_LIBRARY_PATH` 含 `../boost_1_62_0/stage/lib` 与 `work_lib/AMI/lib64`（或在链接时加 `-Wl,-rpath`），因为部署目录内没有这两个库。

## 3. 建议的收口动作（未执行，属原任务范畴）

1. 修 `build_mock_client.sh`：`-I${B}`、`-L${B}/stage/lib`、去掉 `2>/dev/null`、加 `-Wl,-rpath,${B}/stage/lib`；链接后断言 `ldd | grep 1.68` 为空。
2. 用修好的 mock_client 跑一次 gt_sit 深市功能 4/4（这一步才能真闭环；本次只做了编译 + `--help` 冒烟）。
3. 若仍 double free，再回头看 §4.5.7 的 codec 假设 —— 届时才有意义，因为那时探针才是干净的。

## 4. 其余复核发现（非阻塞，但建议修）

| # | 发现 | 位置 |
|---|---|---|
| 1 | 文档 §4.5.3 表格里 gt_trunk 的 DSE 联通仍写「⏳ 待新一轮构建」、gt_sit 写「待验证」，与 §4.5.6 的「三仓均 ✅ 双向 attach」自相矛盾（§4.5.4 的 gt_trunk 4/4 也仍标「NO_DSE 构建时期」，未更新为 DSE 版复测） | 66 号文档 §4.5.3 / §4.5.4 |
| 2 | BUILD_ARM 的架构自适应修复**只落在 gt_sit** 的 `compile_fte_all.sh`；gt_trunk 的 `compile_fte_only.sh` 仍硬编码 `export BUILD_ARM=1`（在 x86 容器里会复现同类 `File in wrong format` 链接失败）；gt_trunk / gt_svn 的 `compile_fte_all.sh` 则完全不设 BUILD_ARM，在真实 ARM 生产机上会反向链接 x86 共享 boost | `DYS-FRAMEWORK/fte_all/compile_fte_only.sh`、三仓 `compile_fte_all.sh` |
| 3 | 22 项自检只扫 `*.sh`（且排除 `/bak/`、`/build`），不扫 `.py`；因此 gt_svn / gt_sit 的 `DYS-FRAMEWORK/fte/test_all/etf_test_sh/ai/expand_5_tables.py` 里仍写着 `/home/lsz/code/work/gt_trunk/...` 却判"跨仓路径干净" | `DYS-FRAMEWORK/fte/test_all/verify_branch_isolation.sh` |
| 4 | gt_svn 的 `run_svn/ute_mock_client` 只部署了 mock_client，未部署 `libagw_ute_api.so`（运行时按 RPATH 指向 `build_root/build_release/...`），属"部署不完整但当前可用"的隐患 | `/mnt/work/gt_test_svn/run_svn/ute_mock_client/` |

## 5. 已核实为真的部分

- 三仓工作区干净、与上游 0/0：gt_trunk `906899d19`(trunk)、gt_svn `39accc6b9`(svn_remote)、gt_sit `2ee31026d`(svn_sit)。
- 四类 ATP/biz 报文头三仓 md5 完全一致（messages_atp_te `425ec04bc9`、msg_struct `d8426392b8`、constants `a4ef3af6a6`、messages_biz_msg `3eb1d0452b`），即"报文头三版本"根因确已消除。
- `verify_branch_isolation.sh` 三仓 md5 相同；"22 项"与脚本结构吻合（1 自动探测 + 3 显式分支 + 1 非法分支 + 1 带参 + 1 运行目录 + 10 组件 + 2 上场数据 + 2 跨仓路径 + 1 汇总；`--all` 另加 3 项）。
- `FTE_WITH_DSE` 三仓均默认 `ON`（`-DFTE_WITH_DSE=${FTE_WITH_DSE:-ON}`），"本仓重编 ute 连不上 DSE"的默认值问题确已修。
- 三仓 `svn_work/66_*.md` 为同一份（md5 相同，497 行）；`task/task_rule.md` 第 14 章「跨仓同步考察规则」存在。
- 部署目录确为分支独立：`gt_test/run_trunk`、`gt_test_svn/run_svn`、`gt_test_sit/run_sit`，各自含 ute / dse / domain_server / libutedatainit.so / dse_cfg_* / etf_test_* 。
