---
id: "ad9f0f1f-7f34-4dda-a911-83a9c8e2b8a3"
title: "start.sh rm_memhq_shared_file 逻辑审计与 /dev/shm 残留归属（2026-11-13）"
description: "otcsrc 启停脚本 /dev/shm 清理审计：rm_memhq_shared_file 的三重空转原因（配置注释掉 / .in≠.init / 模块未启用）、hare-default-* 无人清理、rm_kgbp_shared_file 模式已失效，含 /dev/shm 逐项归属表与待确认修复方案。"
status: "active"
created_at: "2026-11-13T01:59:28.341Z"
updated_at: "2026-11-13T01:59:28.341Z"
content_hash: "05fd3e288f0661b83d4472afcf6d10272e94f6986292004ded4a7c98c5bd089a"
source_paths:
  - "otcsrc/kgbp_x64_linux/bin/start.sh"
  - "otcsrc/kgbp_x64_linux/bin/killipc.sh"
  - "otcsrc/kgbp_x64_linux/bin/rm_kmdb.sh"
  - "otcsrc/kgbp_x64_linux/bin/stop_delete.log"
  - "otcsrc/kgbp_x64_linux/conf/kgbp.xml"
  - "task/global_node/global_test.md"
  - "study/knowledge_base_system/05_部署.md"
session_ids:
  - "db27e0d7-f8d1-404e-a34d-5382094170ac"
memory_body_ids:
  []
---

# start.sh `rm_memhq_shared_file` 逻辑审计与 `/dev/shm` 残留归属（2026-11-13）

用户提问：「start.sh 中的 `rm_memhq_shared_file` 逻辑是什么？今天发现好像没有正常的删除 `/dev/shm` 中的内容」。

结论先行：**该函数目前是死代码（配置层 + 代码层双重空转）**；而 `/dev/shm` 真正占大头的 `hare-default-*`（每节点约 82MB）**没有任何启停脚本负责清理**——这才是"没正常删除"的实际来源，与 `rm_memhq_shared_file` 无关。

## 1. 函数逻辑（`kgbp_x64_linux/bin/start.sh` 约 L378~L399）

```bash
rm_memhq_shared_file() {
    local keyword='load="libmemhq.so"'
    result=$(get_module_config "$keyword")            # ① 在 ../conf/kgbp.xml 找 load="libmemhq.so" 的 module 块
    if [ -n "$result" ]; then
        result=$(echo "$result" | grep -E '<memhq')   # ② 取块内 <memhq 行
        memhq_name=$(... awk -F'name="' ...)          # ③ 抽出 name="kmemhq" → kmemhq
        if [ -n "$memhq_name" ]; then
            rm -rf "/dev/shm/sem.${memhq_name}.in"    # ④ 删"信号量文件"
        fi
    else
        echo "[WARNING] 未找到 memhq 配置，跳过 memhq 信号量清理（避免误删其他节点 IPC）"
    fi
}
```

调用链：`clean_environment()` → 依次 `rm_kgbp_shared_file` → `rm_memhq_shared_file` → `clean_hare_persistent`（受 `CLEAN_HARE_PERSISTENT=1` 门控）→ `./killipc.sh`；`clean_environment` 在 **start / stop / stop-init** 三个分支都会执行。

## 2. 为什么今天什么都没删 —— 三重叠加

| # | 原因 | 证据 |
|:-:|---|---|
| 1 | 当前 `../conf/kgbp.xml` 里 memhq 模块**整块被注释**，`get_module_config 'load="libmemhq.so"'` 返回空 → 直接走 WARNING 分支 | 用同样 awk 实测输出为空；`kgbp_ori.xml:188` 保留原块 `<module name="memhq" ... load="libmemhq.so"><memhq name="kmemhq" size="512M"/>`，现用配置只剩注释里的 `libmemhq_xa.so` |
| 2 | 后缀写错：删 `.in`，系统实际是 **`.init`**，且有分区实例 `sem.<name>_1..N.init` | 现网实证：`sem.2mdb.init` / `sem.2mdb_{1..4}.init`、`sem.2mdbglobal{,_1..4}.init`。即函数被"修好配置"也永远匹配不到 → 死代码 |
| 3 | memhq 是内存行情缓存模块（`<memhq name="kmemhq" size="512M"/>`），本环境节点未启用 | 运行中的 kgbp（业务 42 / 全局 242）均未加载 libmemhq.so |

补充：即使修正后缀，该函数也只管 `sem.kmemhq*.init`，**不管**内存库 `sem.2mdb*.init`——后者职责在 `rm_kmdb.sh`（见 `7c35f3c6-2386-4c59-8c16-6ee1fe06b62f`、`3fff2be4-655c-4416-973f-77c5cf229cc1`）。

## 3. `/dev/shm` 逐项归属（实测 `du`，总量约 165MB）

| 内容 | 大小 | 创建者 | 负责清理 | 现状 |
|---|---|---|---|---|
| `hare-default-42/`（`counter_cnc_613565.dat`） | ~82MB | 业务节点 Hare（`<system id="0000000042">` 末两位 = 42） | **无任何脚本** | 业务节点运行中，正常占用 |
| `hare-default-242/`（`counter_cnc_187536.dat`） | ~82MB | 全局节点 Hare | **无任何脚本** | 全局节点运行中 |
| `sem.2mdb{,_1.._4}.init` | 5×4KB | kocadb（业务节点，`as_num="4"`） | `rm_kmdb.sh`（start/stop 有意不删，2026-11-05 政策） | 运行中，正常 |
| `sem.2mdbglobal{,_1..4}.init` | 5×4KB | kocadb（全局节点） | 同上 | 运行中，正常 |
| `hare-default-100/`、`hare-default-301/` | 8KB | 历史 root 残留 / mock_client 专用 | 仅 mock_client 脚本清 301 | 小残留 |

关键点：**`hare-default-*` 不随进程退出自动消失**；属主若为 root（全局节点以 root 启动过）会导致 kgms 写 `hare_map_new_file_ex(...) failed (13: Permission denied)` → `hare_init ret=70301`，需删残留目录让其重建。已知于 `task/global_node/global_test.md:1896`、`global_node_dev_log.md:290-293`、`study/knowledge_base_system/05_部署.md:171`。

另一个同样空转的函数：**`rm_kgbp_shared_file`** 找 `/dev/shm/kgbp-<cwd路径>` 与 `/dev/shm/kgbp_shared_memory`；`stop_delete.log` 显示 11-10/11-11 曾真实删过（"删除 kgbp 共享文件: /dev/shm/kgbp--mnt-work-otc_src…"），但这两个模式当前已不存在 → 现在的"删除成功"消息不可信。

`killipc.sh` 默认（无 `-f`）只删本节点 POSIX `sem.*.init` 文件 + `ipcrm -s`，**不碰 SysV 段**；`-f` 才强制 `ipcrm -m` 当前用户全部 SysV 段（先检查无 kgbp/kocadb 进程）。默认路径已沙箱复刻验证可删（`ztest` + `as_num=2` → `sem.ztest{,_1,_2}.init` 全删）。

## 4. 待实测确认（勿凭日志下结论）

`stop_delete.log` 中 `sem.init` 相关记录为 **0 条**（仅 `rm_kmdb.sh` 在 11-05 删过）。可能解释：stop 时 kgbp 已优雅退出自行 unlink，`killipc.sh` 判 `-e` 为假既不删也不记；或 start 时尚未创建。要确认需一次带日志的 **stop → 检查 `/dev/shm` → start**（需短暂停节点）。

## 5. 建议修复方案（未落地，需用户确认）

1. **修 `rm_memhq_shared_file`**：`.in` → `.init`，补分区通配 `sem.${memhq_name}_[0-9]*.init`，补 `ipcrm -s` 与 killipc 口径一致，保留"无 memhq 配置则跳过"的保护。**6 份副本同步**：`kgbp_x64_linux`、`kgbp_x64_linux_global`、`kgbp_arm`、`kgbp_arm_global`、`kgbp_x64_linux_global_test_remote`、`tools/OTC`（改前改后都比 md5）。
2. **新增 `rm_hare_shared_file()`**：由本节点 `kgbp.xml` 的 `<system id>` 推出 `hare-default-<末两位>`；**仅在确认本节点无 kgbp 进程时**删本节点目录，不碰其它节点/mock 的 100/242/301；支持 `-n` 预览 + `stop_delete.log` 记账。
3. **让 `rm_kgbp_shared_file` 说实话**：模式不存在时打 `[INFO] 未发现 kgbp 共享文件（已无需清理）`。

风险提示：方案 2 属**语义决策**——"stop 后即清、下次 start 重建"需用户确认可接受（Hare 共享内存常驻用于计数器，清理会丢在途计数状态）。
