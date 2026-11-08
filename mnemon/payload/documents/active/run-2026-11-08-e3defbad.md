---
id: "e3defbad-a13b-483d-afdb-df64e78c6be0"
title: "三仓部署隔离改造：run_<分支> 独立目录、分支探测与跨仓污染修复（2026-11-08）"
description: "gt_trunk/gt_svn/gt_sit 部署隔离改造记录：FTE_BRANCH 三级解析与 run_trunk/run_svn/run_sit 独立目录、三处跨仓硬编码污染、被 source 的 env.sh 不能读 $1 的陷阱、dse 编译 ATP util/include 遮蔽 BizParam 的修复、--deploy-only 构建目录、repos.py 分支适配、端到端 4/4 复测与遗留项。"
status: "active"
created_at: "2026-11-08T10:07:02.755Z"
updated_at: "2026-11-08T10:07:02.755Z"
content_hash: "ebd546cf71ca1897ac95f05b58de6770ae7965e5291617ade361fb3d373e5c85"
source_paths:
  - "DYS-FRAMEWORK/fte_all/compile_fte_all.sh"
  - "DYS-FRAMEWORK/fte/test_all/webui/repos.py"
  - "DYS-FRAMEWORK/fte/dse/CMakeLists.txt"
  - "task/task_rule.md"
  - "svn_work/66_三仓完全独立改造记录_20261108.md"
session_ids:
  - "1f995e01-3a62-462a-b01d-fb6fecd9228e"
memory_body_ids:
  []
---

# 三仓部署隔离改造：run_<分支> 独立目录与分支探测（2026-11-08）

> 背景：gt_trunk / gt_svn / gt_sit 三仓共用一个 `${WORK_ROOT}/run` 部署目录，互相覆盖产物，且脚本中存在跨仓硬编码导致"在 A 仓跑脚本、实际编/跑 B 仓代码"。
> 关联文档：`d2d8435f-21da-4ebb-a207-25a1799cd569`（三仓编译与部署口径，描述的是改造前的共享 run/ 模型，本文为其演进）。
> 仓内成文版：`svn_work/66_三仓完全独立改造记录_20261108.md`（三仓各一份）。

## 1. 目标与结果

| 项 | 改造前 | 改造后 |
|---|---|---|
| 运行目录 | `${WORK_ROOT}/run`（三仓共用） | `${WORK_ROOT}/run_trunk` / `run_svn` / `run_sit` |
| 分支标识 | 无 | `FTE_BRANCH`，三级解析：环境变量 → `-trunk/-svn/-sit` 入参 → 按本仓路径自动探测 |
| 控制台解析 | 固定 run/ | `work_root/run_<branch>`，缺失时回退旧 run/ 并在 note 提示 |
| gt_trunk dse | 借用 gt_svn 产物 | 本仓独立编译 5,712,624 B，0 error |

## 2. 三处隐蔽的跨仓污染（本次最重要发现）

脚本看着在正确仓库里跑，实际用的是另一个仓的代码/产物：

1. **gt_svn 脚本在 docker 内 `cd /mnt/work/gt_trunk`** → 在 gt_svn 跑脚本实际执行 gt_trunk 代码。改为指向本仓。
2. **`compile_mock.sh` 硬编码 `cmake /home/lsz/.../gt_trunk/...`** → 无论哪个仓编译，都编 gt_trunk 源码。改为 `SRC_DIR`（cd 前固化路径）。
3. **`compile_fte_only.sh` 从 `$WORK_ROOT/run/env.sh` 取环境** → 依赖运行目录，形成循环依赖。改为从本仓 `test_all/env.sh` 取。

教训：多仓共用脚本时，任何绝对路径、`WORK_ROOT/run` 引用、`cd` 目标都需按"本仓"语义复核；仅看脚本所在目录不足以判断实际执行对象。

## 3. 陷阱：被 source 的 env.sh 不能读 `$1`

`env.sh` 是 `source` 进来的，`$1` 是**调用方的参数**。若用 `$1` 判分支，`run_test.sh --market sh` 会把 `--market` 当分支名报错。

落地做法：`env.sh` **只用环境变量 + 本仓路径探测** 决定 `FTE_BRANCH`；分支开关只放在入口脚本（`compile_fte_all.sh` 等）解析后 export。

回归验证（全部通过）：

| 场景 | 预期 | 结果 |
|---|---|---|
| 调用方带参（`set -- --market sh; source env.sh`） | 不把 `--market` 当分支 | BRANCH=trunk |
| 显式 `FTE_BRANCH=sit` + 调用方参数 | 用 sit | BRANCH=sit |
| 非法分支 `FTE_BRANCH=xxx` | 报错退出 | 退出码 2 |

## 4. dse 编译修复：ATP 头目录遮蔽

现象：`dse_data_loader.cpp:20: 'BizParam' has not been declared`（gt_trunk 与 gt_sit 同类）。

根因：include 顺序里 `${ATP_RES_ROOT}/common/include/util/include` 中存在同名 `biz_param.h`，遮蔽 dse 自身头。

修复：
- `dse/CMakeLists.txt`（trunk/svn）移除 ATP `util/include` 整目录，改用 `atp_override` 精确覆盖（三仓 `dse_deps` 均为 `atp_override=7`）；
- 同步移除 `THIRD_INCLUDE_DIR_SRC` 里的 ATP `util/include`（仅剩注释引用，无实际使用）；
- 重编结果：`build_root/build_release/dse/dse` = 5,712,624 B，make=0 error。

注意：gt_sit 的对应锚点文本缺失，未能自动修（该仓走 `build_dse_sit` 独立构建）。

## 5. `compile_fte_all.sh` 增强

- 新增 `FTE_BUILD_DIR` 覆盖构建目录；
- `--deploy-only` 默认按 **Release** 取产物（此前落到 `build_fast`，导致"dse 未找到"WARN）；
- 新增 `dse_stack` 部署：`dse`（${BUILD_DIR}/dse/dse）+ `domain_server`（${AMI_RES_ROOT}/bin，dse 启动前必须先起）+ 按市场复制 `dse.xml` / `domainserver_98.config` 到 `dse_cfg_sh|sz`。

## 6. 控制台适配（webui/repos.py）

- 每仓新增 `branch` 字段，`run_dir = work_root/run_<branch>`；
- 兼容：分支目录不存在但旧 `run/` 存在时回退，并在 note 标 `[提示] 未找到分支目录`；
- gt_sit 的 `code_dir` 由 `gt_sit_src` 改回本仓 `gt_sit`，部署项改为 `run_{branch}/` 与 `build_sit_dse`。
- 静态验证：三仓分别解析为 run_trunk / run_svn / run_sit，无回退（运行态页面复测未做）。

## 7. gt_sit 资产补齐

- 运维脚本 5 个 + `env.sh`；
- 上场数据：`etf_test_sh`（9 bin + 44 xml）、`etf_test_sz`（13 bin + 47 xml）；
- mock_client 源码 604 KB（移除 135 MB 预编译 .so 与日志）、mock_exch_server 源码 736 KB（已验证可独立编译，产物 130 KB；mock_sqlserver 134 KB）；
- 部署脚本重写：ute / lib / dse / domain_server / mock / 数据 / 脚本全部取自本仓。

## 8. 复测结论

| 项 | 结果 |
|---|---|
| 部署隔离 | 三套独立目录均部署成功 |
| 分支探测 | trunk / svn / sit 各自正确 |
| 陷阱回归 | 3/3 通过 |
| gt_trunk 端到端 | 栈启动（domain_server/dse/ute/etcd）+ 深市功能 4/4 通过 |
| gt_sit 独立编译 | mock_exch_server、mock_sqlserver、dse（1.78 MB） |

遗留：
1. `run_trunk` 下 dse↔fte 共享内存通道未建立（`DSE attach ute_dse: 0`，FTE 功能仍 4/4 通过），需单独排查；
2. gt_svn / gt_sit 端到端复测待补；
3. gt_sit 的 mock_client 尚无独立编译链路（源码就位，缺 agw 库构建）。

## 9. 提交

| 仓库 | 提交 | 与上游 |
|---|---|---|
| gt_trunk | `d91389bd8` | 0/0 |
| gt_svn | `f9da074e6` | 0/0 |
| gt_sit | `2c4b2aa8a` | 0/0 |

工作规范侧：`task/task_rule.md` 第 14 章「跨仓同步考察规则」——任一仓对"源代码之外的部分"升级后必须考察另两仓是否需同步，并写明结论，禁止默认不同步。
