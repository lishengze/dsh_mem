---
id: "90434fbc-1037-4476-a8ec-06a5ce1add4a"
title: "DSH 上下文溢出诊断：profile 虚报 contextWindow 致压缩不触发（2026-11-04）"
description: "报错 1,576,974 > 1,048,576 的根因：cordis.patch.yml 把 lang/cloud 路由 contextWindow 写成 2,000,000，压缩触发点被算到 1,600,000 永不触发；含触发公式、建议改法、旧会话交接与 /compact 不分块的坑。"
status: "active"
created_at: "2026-11-04T13:09:56.258Z"
updated_at: "2026-11-04T13:09:56.258Z"
content_hash: "a96f0de60ff13bdffb953127b03e08d3cc2bafb38cdaf35ab294761214b5f3a0"
source_paths:
  []
session_ids:
  - "faf940a4-3452-487e-8b74-8718938fd2ca"
memory_body_ids:
  []
---

# DSH 上下文溢出诊断：profile 虚报 contextWindow 导致压缩永不触发（2026-11-04）

## 现象

其他会话报错：

```
This model's maximum context length is 1048576 tokens. However, you requested 1576974 tokens
(1320974 in the messages, 256000 in the completion). (request_id: de5bc3a9-...)
```

关键推断：1320974（messages）+ 256000（completion）= 1576974；即使把输出预算压到 0，消息本身仍超出上游硬上限 27 万 token。**单纯调小 max_tokens 救不回来，必须先削减消息历史。**

## 根因：配置窗口是上游实际上限的近两倍

用户 profile 补丁层 `/home/lsz/.dsh/profiles/web/cordis.patch.yml` 中，多条路由把窗口写成 2,000,000：

- 第 73/86 行（provider `cloud`，https://aifw.ucloud2026.cn/v1）`contextWindow: 2000000`，`maxTokens: 512000`
- 第 118/130 行（provider `lang`，https://ailanglang.top/v1）`contextWindow: 2000000`，`maxTokens: 256000`

从报错里的 256000 completion 可反推：**出问题是 `lang` 组的 deepseek-v4.1-flash-1017 / deepseek-v4.1-flash（maxTokens 256000）**；`cloud` 组是 512000，与报错不符（若 ucloud 确实给到 2M，则不必改）。

而 `@deepseek-ai/dsh-compaction-basic` 的自动压缩触发点公式（见包 README，`/home/lsz/.npm-global/lib/node_modules/@deepseek-ai/dsh/node_modules/@deepseek-ai/dsh-compaction-basic/README.md`）：

```
floor(min(W × thresholdRatio, W − O − headroomTokens))
默认 thresholdRatio = 0.8，headroomTokens = 65536，retainRatio = 0.16
```

以 W=2,000,000、O=256,000 计算，触发点 = 1,600,000。会话长到 1,320,974 时**离触发线还差 28 万，压缩一次都不会跑**，但上游在 1,048,576 就直接 400。这是"模型说能装 2M、服务商只给 1M"造成的错位，调 thresholdRatio 治不了——窗口被虚报成 2M，任何比例都会算出过高的触发点。

## 建议修复

1. 把实际会用到的路由改回真实窗口，并降低过大的输出预留：

```yaml
contextWindow: 1048576   # 原 2000000
maxTokens: 32768         # 原 256000 / 512000
```

改后触发点 ≈ floor(min(1048576×0.8, 1048576−32768−65536)) ≈ 727,040，距 1M 留约 30 万缓冲，压缩来得及介入。改 profile 需重启 dsh 才生效，会影响其他在跑的会话，改动前先确认。

2. 已经超限的那个会话：先试 `/compact`，但 `dsh-compaction-basic` **不分块**，摘要请求会把约 1.19M token 历史整体发出，大概率再撞同样的 400。更稳的做法是开新会话 + 交接摘要；旧会话数据在 `~/.dsh/sessions/--home-lsz-code-work-gt_trunk--/session-47ad2301-a1d4-44be-83ec-4512ec797454/session.v4.jsonl.zstd`（约 14M，可用 zstd 解压后提炼目标/已完成改动/未决问题）。

3. 日常习惯：不要把整份日志或大文件直接打印进对话（用 head/tail/grep 或写文件再读片段）；长任务中途主动 `/compact`，别等自动触发。

## 相关环境事实

- DSH 安装位置：`/home/lsz/.npm-global/lib/node_modules/@deepseek-ai/dsh/`（约 862M），压缩相关包：`dsh-compaction`、`dsh-compaction-basic`、`dsh-compaction-tool-result-pruner`、`dsh-compaction-image-offload`、`dsh-command-compact`、`dsh-token-meter`。
- 手动压缩命令实现：`dsh-command-compact/lib/index.js`，失败码 busy/cancelled/changed/summary/commit/persistence 各自对应不同处置。
- 会话存储：`~/.dsh/sessions/<workspace 转义名>/session-<id>/session.v4.jsonl.zstd`。截至 2026-11-04 最大的是 gt_trunk 下 14M 的那个会话。
- profile 补丁层位置：`/home/lsz/.dsh/profiles/web/cordis.patch.yml`（191 行），`cordis.yml` 仅 4 行；`/home/lsz/.dsh/.credentials.yaml` 存凭据（勿外泄）。

## 状态

诊断已完成并有上述证据；配置修改**尚未执行**，等用户确认后再动 profile。
