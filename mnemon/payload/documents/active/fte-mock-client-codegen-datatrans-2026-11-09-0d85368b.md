---
id: "0d85368b-f55c-4ebc-991f-d79e413a2c52"
title: "FTE mock_client 测试框架实施记录：codegen、单案例目录、回报侧车与 DataTrans（2026-11-09）"
description: "31 个 codegen 文件清单与执行要点（python2.7 UCS4、-d 必给、-I 不能是目录）；config/test_cases/<market>/cases 单案例布局与 cases/ 前缀坑；actual_fields 侧车与展示；撤单案例解耦；datatrans.py 往返与 pad 填充；构建/内存/链接经验；三仓提交。"
status: "active"
created_at: "2026-11-08T23:17:00.686Z"
updated_at: "2026-11-08T23:17:00.686Z"
content_hash: "eb8ccfcd9dbdc9298afdea3fe505a7fc30cf123086ca72371b2322479c071656"
source_paths:
  - "svn_work/67_codec生成作业单_20261109.md"
  - "svn_work/68_DataTrans工具说明_20261109.md"
session_ids:
  - "98a45ca2-1697-40c0-b34d-250d0bd2b154"
memory_body_ids:
  []
---

# FTE mock_client 测试框架：codegen、单案例目录、回报侧车与 DataTrans（2026-11-09）

本记录来自一次已完成的实施任务（DYS-FRAMEWORK / fte_all 测试框架，三仓 gt_trunk / gt_svn / gt_sit）。所有结论均经实测：构建 rc=0、深圳 8/8 案例通过、13 个 .bin 往返字节级一致、三组件联通验证通过。

相关既有文档：`6eb8dfb8-9fb2-4eb2-84af-99641ec7da22`（三仓测试资产与上场数据管理方案，含 schema、Row struct、bin 格式与 bin_tool.py）。本记录只补充其中未覆盖的执行流程与本次实施细节。

## 1. 哪些文件需要 codegen（31 个）

文件名带 `_atp_te`、`_agw_external`、`_biz_msg` 后缀的 .h/.cpp 全部为生成物，共三套 31 个文件：

| 套件 | 输入 IDL | 产物 | 落地位置 |
|---|---|---|---|
| atp_te | `DYS-FRAMEWORK/Common/dev_resource/atp_te.xml` | 7 头 + 4 cpp | `Common/util/include/` |
| agw_external | `DYS-FRAMEWORK/gw_external_msg/V2/buildmsg/agw_external.xml` | 6 头 + 4 cpp | `V2/external/{include,src}/.../codec/` |
| biz_msg | 由 atp_te + 业务定义生成 | 6 头 | `Common/biz_msg/include/` |

**关键更正（重要教训）**：曾判断"需要在 glibc 2.34 的外部机器上生成"，该结论只看了 C 版 `msg_proto`。实测容器内 `python2.7` 为 **UCS4** 构建，`msg_proto.py` 可直接运行：atp_te、agw_external 两套生成均 rc=0，各 10 个文件。以后遇到"生成器跑不起来"，先核对 python 是否为 UCS4（`sys.maxunicode`），不要先怀疑 glibc/C 工具链。

命令要点（详细参数见 `svn_work/67_codec生成作业单_20261109.md`）：
- `-d` 必须显式给出（输出目录），不能省略；
- `-I` 不能传目录，必须传具体输入文件。

## 2. 单案例文件存储与载入

- 新布局：`config/test_cases/<market>/cases/<案例名>.json`，trunk 上 sh 13 例 / sz 8 例。
- 拆分工具：`split_test_cases.py`，两档（体检只报告 / 写盘），同名去重。
- 载入：`suites[].files` 列表；旧 `file` 字段保留兼容。
- 文件格式同时兼容 `{test_cases:[...]}` 与单案例对象。

**踩坑**：以目录方式执行时，配置路径必须带 `cases/` 前缀（如 `config/test_cases/sz/cases`）。漏掉时每个案例文件都命中但结果是 0/0 —— 表现为"静默通过 0 例"，极易误判为实现失败。

## 3. 回报校验与 actual_fields 侧车

- `validate_response()` 本来就只比对 `expected_response.fields` 中列举的字段，不是逐字段全比；本任务只是补展示，未改校验语义。
- 新增 `TestResult.actual_fields`：采集真实回报全字段，**不参与校验**。
- 每例生成 `*_actual.json` 侧车（`name` + 全部实际字段）；控制台下半部分以表格展示（差异红底），通过与失败都显示。
- 实测：sz 8 例中 6 例有真实回报，每例 31–36 个字段；心跳/登录类无真实回报，字段数为 0。

## 4. 案例解耦

扫描全量案例后只发现 1 处跨案例依赖：`FTE 撤单测试` 使用 `orig_cli_ord_no = $last_order_sys_no`，依赖前序委托用例。

解耦方式：固定为 `"0"`。依据是模拟环境的期望回报已固定为 `exec_type=1 / order_status=8 / 50046`，并不需要真实委托号；旧 `fte_combo.json` 同步修改。

资金/持仓：核查确认 mock 不扣减、上场数据只读，本轮无需另造数据集。

## 5. DataTrans 工具

- 位置：`DYS-FRAMEWORK/fte/test_all/datatrans/datatrans.py`
- 接口：`datatrans.py <input> <output>`，转换方向由文件名后缀判定
- 环境：python3.7.9 兼容，仅用标准库
- 验证：13 个 .bin 往返 **字节级一致**

设计要点：记录字符串的**填充字符**（`pad` 属性），避免 NUL 与空格被互相改写。填充差异是历史上"登录失败 / 下单 50019"的根源（行宽校验挡不住语义变化，参见 `6eb8dfb8`）。细节见 `svn_work/68_DataTrans工具说明_20261109.md`。

## 6. 构建与验证经验

- 构建目录：`DYS-FRAMEWORK/fte_all/build_root/build_release`；mock_client 产物约 5.48 MB。
- 链接期 `undefined reference` 的一种常见成因：匿名命名空间内函数在使用点之后定义（本次 `accumulate` 需前移到 `run_from_config` 之前）。
- 编译前建议先清理残留业务进程：曾因 `ute` 常驻占 4.3 GB 导致 cc1plus 无法推进、构建长时间停在某一百分比；清进程后即恢复。
- 端到端验证：启动 domain_server ↔ dse ↔ fte 三组件栈，确认 DSE attach ute_dse 与 FTE attach dse_ute 各 2 次，证明联通。

## 7. 三仓同步与提交

改动需同步到 gt_trunk / gt_svn / gt_sit 三份工作区（14 个文件 + 单案例目录），分别提交并推送，上游均为 0/0：`83a360cd2` / `d0528d52f` / `cecd701ba`。设计文档为 `svn_work/69` 号（任务3 设计与实施，258 行）。
