---
id: "008c711d-2f32-4853-ad4d-729bf5339feb"
title: "FTE / DSE 本地测试端口全链路对照（ute.xml ServerPort、dse.xml 32001、共享内存 N/A）2026-11-18"
description: "本地 dse_stack 测试栈六段链路的端口与来源逐项对照：测试客户端→FTE 的 SHM 无端口与 TCP 33001/33002、FTE→仿真交易所端口、domain_server 5221/5222、DSE→AMI mempipe 32001，并澄清 ServerPortMin/Max 与 UnifiedServerPort 的非客户端语义。"
status: "active"
created_at: "2026-11-18T02:48:36.392Z"
updated_at: "2026-11-18T02:48:36.392Z"
content_hash: "e226861705d3e663bf235bb3554022e0c5d5f00b1c73afac4ac56bc8be6e2922"
source_paths:
  - "DYS-FRAMEWORK/fte/test_all/etf_test_sh/cfg/ute.xml"
  - "DYS-FRAMEWORK/fte/test_all/etf_test_sz/cfg/ute.xml"
  - "DYS-FRAMEWORK/fte/test_all/etf_test_sh/cfg/dse.xml"
  - "DYS-FRAMEWORK/fte/test_all/etf_test_sz/cfg/dse.xml"
  - "DYS-FRAMEWORK/fte/test_all/dse_stack/start_dse_stack.sh"
  - "DYS-FRAMEWORK/fte/dse/src/dse_config.cpp"
  - "DYS-FRAMEWORK/fte/dse/dse_client/src/dse_client.cpp"
  - "DYS-FRAMEWORK/fte/ftedatainit/src/ftedata_init.cpp"
  - "DYS-FRAMEWORK/fte/fte/src/data_manager/config_manager.cpp"
  - "test/api_client/agw_api_demo/etf_demon/main.cpp"
session_ids:
  - "ecc5c6f9-078c-4e94-8e60-391aa4535a23"
memory_body_ids:
  []
---

# FTE / DSE 本地测试端口全链路对照（2026-11-18）

回答「客户端连 UTE 的端口从哪来」时的静态代码与配置核对结论（本地 `start_dse_stack.sh` 那套测试栈）。
**本条为静态核对，未在容器内跑 `ss -tln` 实测**；要实测监听需先起 `otc` 容器并按 `MKT` 跑启动脚本。

相关工作区文档：`db5c633e-d03f-4e9f-bca9-f0a0dbac21a4`（启动顺序契约与踩坑速查）、
`c2a4915e-4e7b-44f8-afc7-84be6b4b5fec`（DSE↔FTE SPSC 共享内存队列）。

## 1. 六段链路

```
测试客户端 ──(①共享内存 / ②TCP:ServerPort)──> FTE 柜台(ute) ──(③共享内存)──> DSE
                                                    │
                                                    └──(④TCP: trade_port)──> 仿真交易所(mock)
      ──(⑤TCP:5221/5222)──> domain_server
      ──(⑥mempipe TCP:32001)──> AMI
```

| # | 链路 | 协议 | 端口 | 来源 |
|:-:|---|---|---|---|
| ① | 测试客户端 → FTE（**本套件实际走的路径**） | SHM，**无端口** | `/dev/shm/Strategy_UTE_*.queue`、`*.lck` | `meta_data.json` 的 `ute_name` |
| ② | 测试客户端 → FTE（TCP 通道，如 `test/api_client` 的 AgwClient） | TCP | **33001**（上海）/ **33002**（深圳） | `ute.xml` 的 `<ServerIP>127.0.0.1</ServerIP>` + `<ServerPort>` |
| ③ | FTE → DSE | **共享内存，无端口** | `/dev/shm/dse_ute*`、`ute_dse*` | — |
| ④ | FTE → 仿真交易所 | TCP | 上海 **38140/38141**（SSETDGW）+ **39144**（SSESTEP）；深圳 **39142/39143**（SZSETGW） | `ute.xml` 的 `<trade_port>` / `<port>` |
| ⑤ | DSE / FTE → domain_server | TCP | **5221**（`DS_WEB`）/ **5222**（`DS_DATA`，etcd 数据口） | `start_dse_stack.sh` L62–63 |
| ⑥ | DSE → AMI mempipe | TCP | **32001** | `dse.xml` `<TargetIP>127.0.0.1</TargetIP>` + `<TargetPort>` |

## 2. 本地回环实例（`127.0.0.1`）

| 市场 | FTE 实例名 | 客户端连入端口 | 出到仿真所端口 |
|---|---|---|---|
| 上海 | `UTE_61_611_11` | **33001** | 38140、38141（SSETDGW）+ 39144（SSESTEP→`mock_sqlserver`） |
| 深圳 | `UTE_84_842_21` | **33002** | 39142、39143（SZSETGW） |

`start_dse_stack.sh` 默认实例与之对应（L50–57）：`MKT=sz → UTE_84_842_21`，`MKT=sh → UTE_61_611_11`；
`DS_NAME=DOMAINSERVER_1_11_11`；`RUN_DIR` 按 `FTE_BRANCH` 推导，**不回落到三仓共用的 `run/`**。

| 端口 | 程序 | 备注 |
|---|---|---|
| 38140 | `mock_exch_server --config mock_exch.xml` | SSETDGW row#1 |
| 38141 | `mock_exch_server` | SSETDGW row#2，**缺了 FTE 会 SIGBUS** |
| 39144 | `mock_sqlserver --port 39144 --pbu 21085 --setid 1 --fill` | SSESTEP，成交回报匹配关键 |
| 39142 | `mock_exch_server --config mock_exch_sz.xml` | 深圳 ETF/非交易平台 |
| 39143 | `mock_exch_server --config mock_exch_sz.xml` | 深圳现货平台（可选） |

## 3. `ute.xml` 中其它实例（消费端可见的外网仿真所，非本机）

| 实例 | ServerIP | ServerPort | 交易所端口 |
|---|---|---|--:|
| `UTE_81_811_11` | 10.187.111.210 | 1199 | 9902/9910/10648/10649/10653/10654（+1433 SSESTEP） |
| `UTE_82_822_21` | 10.187.143.4 | 1199 | 8604/8609/8621 |
| `UTE_61_611_11` | 10.187.111.40 | 1199 | 9516/9728/10650/10651/10655/10656 |
| `UTE_71_712_21` | 10.187.143.30 | 1199 | 8193/8553 |
| `UTE_83_831_11` | 10.187.111.246 | 1199 | 9904/9911 |

这些 `ServerPort=1199` 是同名实例在另一套环境（UAT/联调）的值，本机测试不能沿用。

## 4. 语义澄清（源码证据）

- **`ServerPortMin/Max` 与客户端连入无关**：`ftedatainit/src/ftedata_init.cpp:1234-1235` 仅读取为
  `server_port_min_/max_`，`fte/src/data_manager/config_manager.cpp` 只做 `LOG_INFO` 打印，未见监听用途；
  实配 8081/8082（多数实例）与 8061/8071（`UTE_61_611_11`、`UTE_71_712_21`）。`UnifiedServerPort`（35001）
  深圳为 35001、上海为 0。真正客户端落点是 `ServerPort`（`ftedata_init.cpp:1232`），
  上海整测实例 `etf_test_sh` 的 `ServerPort=33001`，深圳整测实例 `etf_test_sz` 的 `UTE_84_842_21` 为 33002，
  同文件其它实例为 1199。
- **`dse.xml` 的 TargetIP:TargetPort 是 DSE 连出去的目标，不是监听口**：
  `dse/src/dse_config.cpp:46/56-58` 拼成 `ip:port` 塞入 `vLocations_`，
  `dse/dse_client/src/dse_client.cpp:53` 以 `kLocations` 传给 agw client 的 `DoConnect()`。即 32001 是 AMI 内存管道端点。
- **`FTE(ute)` 侧未发现 TCP server 起监听使用 `ServerPortMin/Max` 的路径**；`server_port_` 来源是
  DS Context（缺 `--domain-server` 或 `-n` 时读成 0，日志 `local_port: 0`，客户端必然超时）。
- **AgwClient 侧端口来源**：`test/api_client/agw_api_demo/etf_demon/main.cpp:1064-1065`（另见 `cash_etf/main.cpp:663`）
  把 `login_info.ute_ip/ute_port` 写成 `kUTEServerIp/kUTEServerPort`；同文件默认值
  `ute_ip("10.189.111.35")`、`ute_port("33001")` 属外部环境，本机需改。

## 5. 踩坑要点

1. FTE 必须带 `--domain-server {127.0.0.1:5222}` 与正确 `-n`，否则 `server_port_` 为 0 不监听客户端口。
2. 共享内存测试客户端根本不碰 33001/33002 —— 靠 `meta_data.json` 的 `ute_name` 挂 `/dev/shm/Strategy_UTE_<ute_name>.queue`。
   因此「SHM 模式下客户端→FTE 的端口号」是 **N/A**；只有 TCP 通道（`agw_api_demo`、`no_boost probe`）才用端口。
3. `config_manager` 只有 `FTE_DEBUG` 下才打印 `server_port`，Release 排障不能靠该日志。
