---
id: "eced3297-383c-468a-9065-c918368b3c8f"
title: "sys_table_sync_cfg.sql（r78465）脚本体检：结论、残留风险与可复用的 SQL 脚本检查清单"
description: "对 OTC_BOS/SQL/sql-bos/2026/S1/sys_table_sync_cfg.sql r78465 的结构体检结论（无语法错、358 行/80 表的数据事实）、r78453→r78465 那个致命中分号的历史教训，以及仍未修的四类风险（COLLATE 不一致、缺 DEFAULT CHARSET、DROP TABLE 清线上运维行、注释条数与实际不符）和可复用的离线检查方法。"
status: "active"
created_at: "2026-11-26T05:29:12.355Z"
updated_at: "2026-11-26T05:29:12.355Z"
content_hash: "6dad87c93aa3ad98f2ca54bbdd9247fe5fd784827408e9b085816a039135267a"
source_paths:
  - "OTC_BOS/SQL/sql-bos/2026/S1/sys_table_sync_cfg.sql"
session_ids:
  - "3dfb5539-ae68-44c3-a806-11c8367e4094"
memory_body_ids:
  []
---

# sys_table_sync_cfg.sql 体检（r78465）

脚本：`OTC_BOS/SQL/sql-bos/2026/S1/sys_table_sync_cfg.sql`（SVN r78465，618 行 / 49921 B / UTF-8 无 BOM / LF）。
作用：重建 `tradingdb.sys_table_sync_cfg`（kgbp 内存表同步开关），DROP + CREATE + 7 条多值 INSERT。

## 1. 当前版本体检结论：语法层面干净

| 检查项 | 结果 |
|---|---|
| 语句切分 | 9 条（DROP + CREATE + 7×INSERT），全部以关键字开头 |
| 括号平衡 | 9 条语句括号差均为 0 |
| 数据行 | 358 行全部为 6 字段元组，异常行 0 |
| 语句内分号 | 恰好 7 个，与 7 个 INSERT 块一一对应（无中途截断） |
| 主键重复 | 0 组（`table_name+system_id+as_index` 全唯一） |
| 列宽 | `table_name` 最长 29 ≤ 32；`system_id` ∈ {3,10} ≤ 10；`as_index` ∈ {1,3,ALL} ≤ 4 |
| 其他 | 无 `;;`/空语句、无 tab、无全角逗号/分号/括号、文件以 `;` 正常收尾 |

数据分布（合计 358 行，涉及 80 张表）：块#1 210 行/35 表（每表 6 行，含 `sys_sno`）· 块#2 31 行/31 表 · 块#3 68 行/34 表（`0000001000`/`0000002000` 各 34）· 块#4 8 行（`trd_oftafilec1`）· 块#5 38 行/19 表（`0000001002`/`0000002002` 各 19）· 块#6 2 行 · 块#7 1 行。

## 2. 历史硬伤：一个分号毁掉 4 行配置（r78453/r78464，r78465 已修）

第 6 条 INSERT 倒数第二行误写成 `...);` 而非 `),`，导致下一行成为以括号开头的孤立语句：

```
ERROR 1064 (42000): ... near '('otc_bj','0000002002','3','1',20261002100000,20261002100000)'
```

按客户端逐条执行、遇错即停推演：该 INSERT 只落 37 行（丢 `otc_bj/0000002002/as3`），随后 `sys_log_user`/`sys_log_detail`/`trd_oftafile26` 三条共 3 行根本没跑 → 少 4 行而"看起来跑完了"。修复是单字符 `);`→`),`（`svn diff -c 78465` 只有这一行）。

**教训**：多值 INSERT 不能只靠肉眼；应按"语句切分后每条语句内的数据行是否都是 `,` 收尾、只有最后一行是 `;`"机械核对。

## 3. 仍未修的风险（按优先级）

1. **DDL 字符集不一致**：`table_name`/`system_id`/`as_index` 声明了 `COLLATE utf8mb4_bin`，而 `sync_flag`、`ext_kind1~3`、`ext_string1~2` 未声明 → 继承库/服务端默认字符集；同目录姊妹脚本（如 `acc_custofsigno32.sql`）对 `char(1)` 一律显式写 `COLLATE utf8mb4_bin`。若 `tradingdb` 默认字符集不是 utf8mb4，`sync_flag` 与前 3 列比较/关联会 `ERROR 1267 Illegal mix of collations`。
2. **表尾只有 `COMMENT = '...'`，缺 `DEFAULT CHARSET=utf8mb4`**：整表字符集跟随库；同目录仅 1 个脚本显式声明过。
3. **`DROP TABLE IF EXISTS` 会清掉线上已追加的运维行**：已上线环境用它等于丢增量配置；增量场景应改用 OTC86/OTC95/OTC100 那类编号 patch 脚本。
4. **无事务保护**：DDL 隐式提交不可回滚，7 条 INSERT 中途失败会留空表/半量数据（§2 的事故正是被这一点放大）；建议合并为单条 INSERT，或把"建表"与"数据"拆成两个脚本。
5. **注释条数与实际不符**（会误导"是否导全"的判断）：头部"共 339 条" vs 实际 358（八部分声明值相加也只有 347）；第 1 部分"34 张/每表 5 条/170 条" vs 实际 35 张/每表 6 条/210；第 3 部分"66 条" vs 实际 68（行 374 注释同样错）；第 5 部分"34 条(17+17)" vs 实际 38(19+19)（`otc_bj` 的 as3 两行补入后未同步改数字）。
6. 低优先：`int(11)`/`bigint(20)` 显示宽度在 MySQL 8.0.17+ 已弃用（仅 warning，OceanBase MySQL 模式可接受，同目录脚本皆此写法）；脚本依赖 `tradingdb` 已存在（无 `CREATE DATABASE`/`USE`，库缺失即 `ERROR 1049`）。

## 4. 可复用的离线检查方法（无需连库、无需 sqlglot）

`import sqlglot` / `sqlparse` 在本机不可用（ModuleNotFoundError），因此按字符状态机做离线体检即可覆盖上述判据：

- 去注释后先切分语句（`;` 为界），核对每条语句首关键字 + 括号差 = 0；
- 统计"数据行以 `;` 结尾的位置数 == INSERT 块数"，定位截断点；
- 逐块统计行数与去重后的表数，验证头部注释声称的条数；
- 用 `(table_name, system_id, as_index)` 做集合去重找主键重复；
- 列宽/取值域用 `length()` 与取值集合对照 DDL 的 `varchar(n)`/`char(1)`；
- `cat -A` 查 tab 与行尾、检查 BOM/CRLF。
