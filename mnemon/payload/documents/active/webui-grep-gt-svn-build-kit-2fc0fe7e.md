---
id: "2fc0fe7e-5b0a-4499-9580-f0948d6cc3ad"
title: "webui 代码浏览搜索子系统（文件名模糊/正则 + 全局内容 grep）与 gt_svn build_kit 存档"
description: "记录 /code 左栏搜索为何改为服务端递归、全字/大小写/正则/包含/排除语义、性能限额实测（22k 文件 4s）、按行渲染+整段高亮切分 span 平衡、跳转行与 Ctrl+F 交互、无浏览器验证手法；以及 tools/build_kit（60KB，仅存 gt_svn 特化的 2 个 fte_all 文件 + check/restore 脚本，其余从 gt_trunk borrow）的取舍与 17 项核验结果。"
status: "active"
created_at: "2026-11-05T01:36:46.012Z"
updated_at: "2026-11-05T01:36:46.012Z"
content_hash: "0c03eb7f296de840554131367be758a57388d4c1c0e75b87ccf9c451710199ce"
source_paths:
  - "DYS-FRAMEWORK/fte/test_all/webui/codeview.py"
  - "DYS-FRAMEWORK/fte/test_all/webui/server.py"
  - "DYS-FRAMEWORK/fte/test_all/webui/static/code.js"
  - "DYS-FRAMEWORK/fte/test_all/webui/static/code.html"
  - "tools/build_kit/README.md"
  - "tools/build_kit/check_gt_svn.sh"
  - "tools/build_kit/restore_gt_svn.sh"
  - "svn_work/50_gt_svn工程所需内容清单.md"
session_ids:
  - "191eec6d-1d9a-478f-8bae-4a3294bcc933"
memory_body_ids:
  []
---

# gt_trunk 控制台 webui：代码浏览搜索子系统与 gt_svn 构建套件存档

来源：2026-11 一次维护会话，两次提交
- `0a36902b7` feat(webui): 代码浏览支持文件内搜索与跳转行
- `97b228405` feat(webui): 代码浏览左栏搜索重构 —— 文件名递归搜索(模糊/正则) + 全局内容搜索(全字/大小写/正则/包含/排除)

相关既有文档：`d2d8435f-21da-4ebb-a207-25a1799cd569`（三仓编译部署口径、构建套件归属、BUILD_ARM 坑）。

## 一、左栏文件名搜索：为什么必须服务端递归

**症状**：用户反馈"过滤文件名只过滤了目录，搜不到文件"。

**根因**：代码树是懒加载的，前端过滤只作用于**当前已渲染的那一层**，深层文件/目录根本没进过 DOM。所以问题不是"只匹配目录"，而是"只匹配顶层"。

**改法**：改为服务端递归遍历整个仓库、按相对路径匹配，结果**平铺返回**（目录与文件都返回，点目录仍可展开/懒加载）。

两种模式（按钮切换，正则态高亮）：

| 模式 | 行为 |
|---|---|
| 模糊 | 子串匹配，忽略大小写 |
| 正则 | 标准正则；**非法正则自动退化为字面量**，避免输错就无结果或报错 |

实测（gt_svn）：`CMakeLists` → 8 项（含根目录）；`f[f]?t[e]_perf` → 命中 `fte_perf_test`。

## 二、全局内容搜索（含可展开过滤）

搜索框下方：全字 / 大小写 / 正则 三个独立开关，右侧 `▾` 展开**包含文件**与**排除文件**两个输入框 —— 两者都支持正则、逗号分隔多个。点结果直接打开文件并跳到对应行。

- 全字实现为 `(?<!\w)…(?!\w)`（注意：不是简单的 `\b`，以兼顾中文/下划线边界）。
- 实测（gt_svn）：`q=kPktETFReq, include=\.h$, exclude=gw_external_msg` → 4 条命中 / 扫描 5361 文件。

**性能保护**（防大仓库卡死）：

| 限制 | 值 |
|---|---|
| 单文件 | 1 MB（超过跳过） |
| 二进制判定 | NUL/控制字符占比 > 10% 跳过 |
| 扫描文件数 | 50,000 |
| 时间预算 | 12 秒 |
| 命中上限 | 200 |
| 噪声目录 | `.git/.svn/.cache/__pycache__/CMakeFiles/node_modules/webui_results/build*` |

实测 gt_trunk 全仓 **22,056 文件 / 4.0 秒 / 不截断**。调参经过：初版上限过小导致 68 文件就 trunc，放宽后再补噪声目录才稳定。

新增接口：

```
GET /api/code/find?repo=&pattern=&regex=&case=&limit=
GET /api/code/grep?repo=&q=&whole=&case=&regex=&include=&exclude=&limit=
```

## 三、渲染与跳转的两个关键实现点

1. **每行渲染成独立 `div.cl`** —— 否则无法精确定位/滚动/局部高亮。
2. **高亮仍整段做**：块注释、多行字符串只有整篇高亮才正确；切行时自动**补齐 / 重开 `<span>`**，保证每行 HTML 标签自平衡。这是最容易翻车的地方（此前用行内独立高亮会出现跨行注释错乱）。

跳转行：`Ctrl+G` 或行号框回车 → 居中滚动 + 蓝色闪现 1.2s；文件内搜索 `Ctrl+F`，`Enter`/`Shift+Enter`、`F3`/`Shift+F3` 上下导航，`Esc` 清除，计数显示"第 n / 共 m 处 · 第 X 行"。

## 四、无浏览器环境下的验证手法（可复用）

- Node 单测核心算法：跨行注释切分后**每行 open/close span 数相等**；`'a\n\nb'` → `["a","","b"]`；偏移→行号 `0→1, 6→2, 18→3, 30→4`。
- **HTML/JS 元素 ID 交叉核对**：脚本比对 JS 里 `getElementById` 引用的全部 id 与 HTML 中 `id=` 集合，差集必须为空（本次 16 个 id 全命中）。改动前端时这一步最省时间。
- 起服务后 curl `/code`、`/static/code.js`、`/api/code/find`、`/api/code/grep` 均 200。

## 五、gt_svn 构建套件存档 `tools/build_kit`（60 KB）

**为什么只存 2 个文件**：其余脚手架（mock 交易所、测试脚本、抓包工具、模拟客户端）在 gt_trunk 有原版，脚本用 `borrow` 现拉，避免仓库重复几百 MB；而 `DYS-FRAMEWORK/fte_all/` 的 `CMakeLists.txt`（与 gt_trunk 差 30 行）和 `compile_fte_all.sh`（差 10 行）是 **gt_svn 特化版，两仓不能互通**，必须独立存档（另有 `.gitignore`）。

```
tools/build_kit/
├── README.md                    为什么需要 / 用法 / 四层判断依据
├── gt_svn_patch/DYS-FRAMEWORK/fte_all/{CMakeLists.txt,compile_fte_all.sh,.gitignore}
├── check_gt_svn.sh              17 项逐层核验
└── restore_gt_svn.sh            一键补齐（borrow 缺失脚手架）
```

验证结果：真实 gt_svn **17/17 全 OK**；合成干净树 3 OK / 14 MISS 并给出修复命令；restore 后仅剩 4 个真实 SVN 才有的依赖子工程（msg_parser、gw_external_msg/des、gw_external_msg/V2、api/gw_api_no_boost）缺失。

**关键提醒**：不要靠"基线清单里有没有"判断 SVN 是否自带 —— 149 条基线只是子集，**没收录 ≠ 不在 SVN**。正确做法是 `svn status <path>`（空=SVN 原生）或 `svn status --no-ignore | grep '^?'` 列本地新增。

运行环境坑：宿主机重启后 HugePages 归零（`nr_hugepages=0`）会导致"启动链路"失败，需 `sudo sh -c 'echo 64 > /proc/sys/vm/nr_hugepages'`。远程机器 `lib/x86` 归档异常时需 `export BUILD_ARM=1` —— 那是该机器特例，不是通用要求。
