---
id: "27412262-3b0e-4cc0-9cee-98c11555b960"
title: "OrdERInfo 解析/输出完整性补全（share_util/json_struct.h）与 json helper 缺省值约定（2026-11-17）"
description: "gt_trunk test/share_test 中 OrdERInfo 的字段覆盖契约（34 字段中除 constituent_stock 需全覆盖，且解析顺序与结构体顺序一致）、本次补的 business_type/no_security/available_t1、重复 market_id 解析修复、GetJsonUnsignedCharField 缺字段取 '-'=45 的既有约定、test_data.json 按 cwd 相对路径读取需同步到 build_20261117/，含提交号与复测方法。"
status: "active"
created_at: "2026-11-17T05:27:02.330Z"
updated_at: "2026-11-17T05:27:02.330Z"
content_hash: "c98db6f67426ef87e8ef0fe52bf83a9514794f6f9ed9e8e4bd127a05167c64e9"
source_paths:
  - "test/share_test/share_util/json_struct.h"
  - "test/share_test/test_data.json"
  - "test/share_test/share_util/json_util.hpp"
  - "svn_work/132_共享内存测试客户端升级_20261117.md"
  - "work_log/20261117.md"
session_ids:
  - "d1777508-1ea6-4acf-a34c-a22e2048f880"
memory_body_ids:
  []
---

# OrdERInfo 解析/输出完整性补全（2026-11-17）

承接同日 `test/share_test` 升级工作（见文档 `98683cb9-f567-47ff-b952-2ebc8a769a36`，那次补的是 `TradeOrderInfo.policy_id`）。本次处理另一处同类缺口：`OrdERInfo`。

代码：`test/share_test/share_util/json_struct.h`（`ParseOrdERInfo` / `OrdERInfoStr`）与 `test/share_test/test_data.json`。
提交 `050946a89`（fix(test)），已推送，三仓 `0 0`，工作树干净。文档记录 `svn_work/132` §6.8~6.10，工作日志 `work_log/20261117.md`。

## 1. 字段覆盖契约（本项目应沿用的判定方式）

- `OrdERInfo` 共 **34** 个字段；`constituent_stock` 是变长区（`char[0]`），**按约定不解析**，因此 `ParseOrdERInfo` 与 `OrdERInfoStr` 必须各覆盖 **33** 个。
- 判定必须**以结构体定义为权威**逐字段比对，且 `ParseOrdERInfo` 的解析语句顺序、`OrdERInfoStr` 的打印顺序要与结构体声明顺序一致（校验脚本要比对集合 **和顺序**，不能只比集合）。
- 比对脚本的坑：用窄正则（只匹配 `GetJson...Field(...)` 单 token 类型）会漏掉 `unsigned short` / `long long` / `GetJsonArrayStringField<16>` 模板实参、以及 `"\norder_id:"` 这类带前缀的打印行，从而误报"不对称"。稳健做法是取每行行尾标识符，并保证对比前**不对输出行排序**（排过序的顺序比对无效）。

## 2. 本次改动

缺口 3 个字段（改动前只覆盖 30/33）：

```cpp
// ParseOrdERInfo：插在 total_value_traded 之后
GetJsonUnsignedCharField(jsOrdERInfo, "business_type", stOrdERInfo.business_type, sErrMsg);
GetJsonUnsignedIntField (jsOrdERInfo, "no_security",   stOrdERInfo.no_security,   sErrMsg);
GetJsonLongLongField    (jsOrdERInfo, "available_t1",  stOrdERInfo.available_t1,  sErrMsg);
```

`OrdERInfoStr` 同步补 3 行；注意 `business_type`（`uint8_t`）打印要 `int(obj.business_type)`，否则会输出成字符。

顺带修掉一个真实 bug：`market_id` 的解析语句**重复写了两遍**（相邻两行完全相同），删除一条后仍正常解析（101/102）。

`test_data.json` 的 `OrderReqRsp`、`CancelOrderReqRsp` 两处 `OrdERInfo` 各补**非零值** `business_type=1`、`no_security=0`、`available_t1=1234567890123`——用非零值才能证明"真解析到"而不是恰好为 0。

## 3. json helper 的缺省值约定（易被误判为脏值）

`share_util/json_util.hpp` 中 `GetJson...Field(..., bAllowDefaultValue=true)` 缺字段时的行为：

| 目标类型 | 缺字段取值 |
|---|---|
| `char` / `unsigned char` | `'-'`，即 ASCII **45** |
| `unsigned int` / `long long` 等整型 | 0 |

所以负向用例（从 json 里删掉新字段）中 `business_type` 读到 **45** 是**既有约定**，与 `exec_type`/`side`/`cancel_flag` 完全一致，不是未初始化脏值。反之 `bAllowDefaultValue=false` 时置 `sErrMsg` 并返回 false。

## 4. 复测口径

1. `make clean` 后全量重编 5 个 target，0 error。
2. 端到端跑委托 + 撤单：**两条回报各 33 个字段**，新字段共出现 6 次，取值与 `test_data.json` 一致（服务端日志同样打印）。
3. 字段完整性脚本比对：结构体 33 ↔ 实际输出 33，**集合与顺序均一致**。
4. 负向用例：删掉 3 个字段 → 程序不崩，char 取 45、整型取 0，其余 30 字段不受影响。

## 5. 部署注意（cwd 相对路径）

测试程序按**当前工作目录**读取相对路径的 `test_data.json`。因此改完源码目录的 json 后，必须再拷一份到运行目录：

- 运行/部署目录：`test/share_test/build_20261117/`（`client_strategy` / `server_ute` 已用新代码干净重建并复测）。

否则端到端仍读到旧数据，表现为"新字段恒为 0/45"，容易误判为解析没生效。
