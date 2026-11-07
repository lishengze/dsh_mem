---
id: "db5c633e-d03f-4e9f-bca9-f0a0dbac21a4"
title: "DSE × DomainServer × FTE 全链路联调打通（2026-11-07）"
description: "SIOF 崩溃根因（跨 TU 静态初始化顺序，Meyers singleton 修复）、dse_deps 空归档 + -DNDEBUG 编译打通、四步启动顺序契约与 FTE 三必须项、沪/深端口 33001/33002 差异、关键踩坑速查，以及双市场功能 28PASS/4PASS 与性能 997/1000 TPS 零丢单证据。"
status: "active"
created_at: "2026-11-07T00:56:20.966Z"
updated_at: "2026-11-07T00:56:20.966Z"
content_hash: "1d3100426d6242f3e1ba3b7318b79857868f55ca7e243292708876a56c2cd605"
source_paths:
  - "DYS-FRAMEWORK/fte/test_all/dse_stack/start_dse_stack.sh"
  - "DYS-FRAMEWORK/fte/test_all/dse_stack/README.md"
  - "DYS-FRAMEWORK/fte/dse/doc/dse_启动依赖与FTE交互_20261106.md"
  - "svn_work/56_DSE与DomainServer接入进展_20261106.md"
  - "DYS-FRAMEWORK/fte/include/business/external_data_handler.h"
session_ids:
  - "2c0e90dd-eda7-449c-a1f8-cdac3b2ff5a0"
memory_body_ids:
  []
---

## 结论

`domain_server → dse → fte` 全链路在上海、深圳两个市场打通，功能测试与性能测试全部通过，且**零丢单**（发送失败 = 0）。
本文记录修复根因、构建适配、启动顺序契约与验证证据；DSE 与 FTE 的共享内存协议本身见 `c2a4915e-4e7b-44f8-afc7-84be6b4b5fec`。

## 1. SIOF 崩溃的真正根因（此前误判）

先前怀疑"同时编入 `ins_message.cpp` 又链接 `libmsg_proto.so` 造成双重注册"，删源文件后符号缺失，该方向错误并已回滚。
gdb 调用栈给出真相：

```
#8 _GLOBAL__sub_I_dse_md_process.cpp   ← 静态初始化期
#6 DseEngineMDProcess::GetQueryClass()
#5 DseEngineMDProcess::DseEngineMDProcess()
#4 MDMessage::RegisterVisitor()
#3 boost::unordered_map::operator[]
```

`OBSERVER_IMPLEMENT` 在 `dse_md_process.cpp` 的静态初始化期注册 visitor，而 `MDMessage::visitor_registry_` 定义在另一个编译单元
`ute_md_process.cpp`。跨 TU 静态初始化顺序在 C++ 中未定义，此时 boost 容器仍是全零 BSS，`add_node` 解引用未初始化指针 → SIGSEGV。

修法（C++11 保证首次使用时构造，与 TU 顺序无关）：

```cpp
static VisitorRegistry& GetVisitorRegistry() {
    static VisitorRegistry registry;
    return registry;
}
```

**一般化教训**：跨编译单元共享的全局注册表不能是裸全局对象；Meyers singleton 是消除 SIOF 的标准手段。

## 2. DSE 编译打通

| 手段 | 说明 |
|---|---|
| `dse_deps` 目录 | 只保留 DSE 真正引用符号的库（`abf_util`/`abf_tech`/`biz_msg`/`msg_proto`/`agw_external_msg`/`agwapi`，均 GLIBC_2.14），其余 8 个 `abf_*` 用**空归档**满足 `-l` |
| `-DNDEBUG` | `fte_all` 用 `string(REPLACE)` 把 Release 的 `-DNDEBUG` 剥掉了，boost assert 在启动时 abort；DSE 侧需自行补回 |

产物 5,741,984 字节，容器内可编译可运行。

## 3. 启动顺序契约与三必须项

顺序：`domain_server → 10s → dse → 10s → 模拟交易所 → 10s → fte`。**模拟交易所必须先起**，否则 FTE 在 `do_connect` 抛 `system_error` 并 SIGBUS。

FTE 启动三必须项：
1. `--domain-server {127.0.0.1:5222}` —— 缺失则进程秒退（exit 0 无进程，客户端侧表现为 `local_port: 0`）。
2. 正确实例名 `-n`（上海 `UTE_61_611_11`、深圳 `UTE_84_842_21`）。
3. `dse.xml` 与 `ute.xml` 的 `IsCreditNode` 必须一致（统一为 0），否则 DSE 卡在 `ute_dse_md`。

端口差异：上海客户端口 33001，深圳 33002（`ute.xml` `<ServerPort>1199</ServerPort>`），脚本须按市场识别，不能硬编码。

## 4. 踩坑速查（关键条目，完整 14 条见脚本同级 README）

| 现象 | 根因 | 解法 |
|---|---|---|
| `path to the web root directory invalid` | 缺 `-r` | `-r $AMI/bin/web` |
| `Import config failed: timeout` | 缺 `-D`，或把 `-i` 当启动参数 | `-D /dev/shm/ami_ds_xx`；`-i` 要单独对**运行中**的 DS 执行 |
| `-i` 返回 0 但 Context 数不变 | 没带 `-n` | `-n $DS_NAME -i <config> -m default` |
| DSE 反复 attach `ute_dse` 失败 | FTE 尚未建队列（属正常） | 等 FTE 对象池初始化完（约 90s），自动 attach |
| 上海 DSE `IP_ADD_MEMBERSHIP failed, No buffer space` | `igmp_max_memberships=20` 被 20 个组播组占满 | `echo 200 > /proc/sys/net/ipv4/igmp_max_memberships` |
| FTE 活着但客户端连不上 | 未带 `--domain-server` / `-n` | 见上三必须项 |

判据陷阱：不要用 `pgrep` 判断组件是否在跑（会匹配到僵尸进程导致误判 "已在运行"），统一用 `ss -tlnp` 看端口监听。

## 5. 验证证据（2026-11-07）

联通：上海 DS Context 730、`DSE_61_611_11`/`UTE_61_611_11`、`ute_dse`+`dse_ute` 双向、端口 33001；
深圳同理，端口 33002。FTE 侧 `sync_up_ready [true]` 表示已连上模拟交易所。

功能测试：上海 28 PASS / 0 FAIL，深圳 4/4 PASS，均 exit=0。
性能测试（ETF fill，目标 TPS 1000）：上海 997.04 TPS、收到回报 5446、P50 框架处理 2114 ns；
深圳 1000.08 TPS、收到回报 22163、P50 框架处理 1453 ns。

**本轮未绑核（`CPU 绑定 : 绑定失败`），分位数不可作性能基线**；可引用的只有"发送失败 = 0"，说明 DSE 链路未引入丢单。

## 6. 交付物与提交

- `DYS-FRAMEWORK/fte/test_all/dse_stack/start_dse_stack.sh`（264 行）：四步启动、自动修大页/igmp、清共享内存、按市场识别端口、幂等 stop
- `DYS-FRAMEWORK/fte/test_all/dse_stack/README.md`（261 行）：8 章 + 14 条踩坑速查 + 实测结果
- `DYS-FRAMEWORK/fte/dse/doc/dse_启动依赖与FTE交互_20261106.md`、`svn_work/56_DSE与DomainServer接入进展_20261106.md`（226 行，两阶段排障记录）
- 代码：`ute_md_process.h`、`dse/CMakeLists.txt`、`fte_all/CMakeLists.txt`、`fte/fte/CMakeLists.txt`、`compile_fte_all.sh`

复查 5 轮修 4 处（步骤序号 `[1/3]→[1/4]`、移除未用 `BUILD_DIR`、README 坑表 7→14 条、端口按市场识别），第 5 轮零修改。
提交：gt_trunk `45a7a5a9d`、gt_svn `7fe16813f`、gt_sit 无改动，三仓均与上游 0/0。
