---
id: "12f5d1df-be46-4b11-8eb6-30f64e0da7bd"
title: "otcsrc 非必填入参 TryGetValue 覆盖默认值：根因、守卫模式与回归陷阱（2026-11-09）"
description: "lbm_otc_query 16 个业务非必填 offset/pageSize/count/bjzdydh 被空串冲掉默认值的排查与修复记录：框架行为实测、统一守卫模式、负值必须放行的回归教训、验证用例与提交状态，供后续同类业务接入复用。"
status: "active"
created_at: "2026-11-09T07:45:48.869Z"
updated_at: "2026-11-09T07:45:48.869Z"
content_hash: "12152f20992189d8a80e973e63c32d70e46233fcc0a836a46d8e61ddbac5799f"
source_paths:
  - "otcsrc/lbm/lbm_otc_query/src/query_per_cust_risk_ctrl_business_411834.cpp"
  - "otcsrc/lbm/lbm_otc_query/src/query_otc_get_bjhq_business.cpp"
  - "otcsrc/mock/mock_client/conf/req_omit_defaults_20261109.xml"
  - "study/410233_留存业务-客户资产总值分类查询/410233_参数对比.md"
session_ids:
  - "0ce1e82a-6af4-4311-94ea-031adcde6012"
memory_body_ids:
  []
---

# otcsrc 非必填入参 TryGetValue 覆盖默认值（2026-11-09 收口）

仓库：`/home/lsz/code/work/otc_src`（`otcsrc/lbm/lbm_otc_query`）。提交 `840a3ed9`（`0b7d6303..840a3ed9` → `origin/dev`）。
相关文档：`otcsrc SVN 改动审阅与提交基线`（id `27e439e6-262d-4c7b-82ee-669d9e65375a`）第 3.1 节 offset off-by-one；本文补充其"默认值被空值冲掉"的根因与守卫写法。

## 1. 根因（实测确认，非推断）

框架 `TryGetValue()` 把 `JstpPackGetValueX(&Value)` 的输出参数**直接传给成员变量**（`jstp_base_for_jstp.h` L317-364），因此：

| 场景 | 实测返回 | 结果 |
|---|---|---|
| 字段完全不传 | `TryGetValue("bjzdydh",-1)=50005`（非成功） | 成员保持默认 ✅ |
| 字段存在但值为空串 | `TryGetValue("offset",0)=0`、`("pageSize",0)=0` | **默认值被冲成 0/空** ❌ |

→ 真正风险是「字段存在但值为空」，会让 `pageSize=100` 退化为取全部、`count=50` 失效、`bjzdydh=-1`（查全部）失效。

## 2. 受影响业务（16 个）

- `offset` + `pageSize`：412232、410685、240690、411827、411535、411834、811526、831509、811509、410567、811523、410553、410233、811528
- 另加 `count`：411830
- `bjzdydh`（默认 -1 = 全部）：410497
- 无风险（`GetValue` 必填解析）：412230、240178、412231、411828、411833

## 3. 统一修复模式

```cpp
{
    int nTmpPageSize = 默认值;
    if (TryGetValue("pageSize", nTmpPageSize) && nTmpPageSize != 0)   // 0 视为空/未给
        成员 = nTmpPageSize;
}
```

约定（四态行为：不传 / 空串 / 0 / 负数）：

- `pageSize`、`count`：**只拦 0**，负值放行给业务校验。
- `offset`：**无条件下赋值**（0 合法，负值必须保留以触发 `803002001` 分页参数错误），仅字段缺失时保默认。
- `bjzdydh`：只拦 0，保持 -1。
- 顺带把 9 个 `offset` 默认值 1 → 0（0-based 语义）。

## 4. 回归陷阱（本轮最重要的教训）

首版守卫写成 `offset >= 0`、`pageSize > 0`，**把负值也吞掉**了：
411834 TC-18「分页参数为负应报 803002001」由报错变成成功，分页校验被静默关闭。

→ 守卫的正确边界是"**只拦空/0，不拦负数**"。任何"合法性过滤"放进默认值守卫都会吃掉下游校验用例。
复查第 1 轮即因此返工；第 2 轮重编重测恢复，第 3 轮零修改。

## 5. 验证方式与结果

| 项 | 结果 |
|---|---|
| 新增省略字段用例 `otcsrc/mock/mock_client/conf/req_omit_defaults_20261109.xml`（14 业务完全不传 offset/pageSize/count） | 14/14 PASS |
| 空值专项（`offset=""` / `pageSize=""`） | 70410567 返回 1 条（pageSize 保持 100）PASS |
| 410497 全局节点（bjzdydh 不传 / 传空） | 23/23 PASS（保持 4 条） |
| 411834 回归（含 TC-18 分页为负） | 17/17 PASS |
| 全量回归（原用例 + patch_verify + 新用例 + 411834） | 75/78 PASS |

3 个失败（410553 TC-12、811509 TC-16、240690 TC-05）均为「客户不存在」的既有数据前置问题：用 `git stash` 回改动前基线重编同库跑同 3 例，失败现象完全一致 → 非本次引入。
重复执行结果一致（可重复性复查通过）。

## 6. 文档与提交状态

- 16 篇《参数对比》（`study/<funcid>_*/<funcid>_参数对比.md`）各新增 **§2.4 非必填默认值保持（2026-11-09 修复）**：问题描述、修复模式、四态行为约定、验证结果。
- 改动文件：16 个 `otcsrc/lbm/lbm_otc_query/src/query_*_business_*.cpp`（含 `query_otc_get_bjhq_business.cpp`）+ 16 篇文档 + 1 个新用例 + 工作日志。
- **SVN 侧未动**：新用例文件需 `svn add`，16 个源码改动待按 SVN 流程提交（git 已存档）。

## 7. 环境小坑

`/usr/local/python3.6` 的 `subprocess.run` 不支持 `capture_output=`（3.7+ 才有），脚本需用 `stdout=subprocess.PIPE, stderr=subprocess.PIPE`，否则 `TypeError: __init__() got an unexpected keyword argument 'capture_output'`。
