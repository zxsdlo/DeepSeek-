---
id: "75ad464f-1af0-4059-9bae-eec491d6c90e"
title: "DSH 记忆空间「未安装 CLI」排查：安装 mnemon CLI 与 embedding 可选性结论"
description: "dsh-mnemon 记忆空间显示「未安装 CLI (可选)」的真实原因、mnemon CLI 安装位置与解析顺序、无需重启宿主的依据，以及 embedding 端点缺失时的降级行为；附 memos.exe 与 Mnemon 无关的取证。"
status: "active"
created_at: "2026-10-23T15:55:23.845Z"
updated_at: "2026-10-23T15:55:23.845Z"
content_hash: "03e965b452365e823c79221ede492210b89586d705a8cddd0c65c2080494cb6f"
source_paths:
  []
session_ids:
  - "d9a5e381-b2e7-430d-b852-da3fa36f09bc"
memory_body_ids:
  []
---

# DSH 记忆空间「未安装 CLI」排查与 mnemon CLI 接入

## 结论摘要

1. **`D:\memos.exe` 与记忆空间无关**：该二进制是 [usememos/memos](https://github.com/usememos/memos)（自托管笔记应用），二进制内 `usememos` 字符串 8678 处、`mnemon` 与 `MemOS` 均 **0 处**；`dsh-mnemon` 及 8 个子 provider 的 `lib/` 中搜 `memos` 也是 0 命中。
2. **真正的问题是缺原生 CLI**：设置页 Mnemon Native 那行的「未安装 CLI (可选)」表明宿主需要 `mnemon` 可执行程序。npm 包 `@mnemon-dev/mnemon` 只是 JS 驱动器，二进制在 optionalDependencies 的平台子包（`@mnemon-dev/mnemon-win32-x64` 等）里。
3. **不需要重启 DSH**：`commandFound` 是每次读取都重新探测的 getter，关闭再打开设置页即可刷新状态。
4. **Ollama 不是必需的**：embedding 端点不可用时写入依然成功，只是返回 `"embedded": false`，检索退化为关键词 + 图谱边。

## 安装与落地位置

```
npm install --global @mnemon-dev/mnemon      # 实测 v0.2.12
```

| 项目 | 实测值 |
| --- | --- |
| 启动器 | `C:\Users\XYD\AppData\Roaming\npm\mnemon.cmd` |
| 包根 | `C:\Users\XYD\AppData\Roaming\npm\node_modules\@mnemon-dev\mnemon` |
| 原生二进制 | 包内 `node_modules\@mnemon-dev\mnemon-win32-x64\bin\mnemon.exe`（约 15.6 MB） |
| 默认数据目录 | `C:\Users\XYD\.mnemon`（可用 `--data-dir` / `MNEMON_DATA_DIR` 覆盖） |

该 npm 全局 bin 目录本就在用户 PATH 中，因此 `where mnemon` 能直接找到，无需额外配置。

## 宿主如何发现 CLI（dsh-mnemon-source-memory-spaces）

- `lib/native-cli.js` 的 `findMnemonCommand()` 在 PATH 上按 `mnemon.exe` → `mnemon.cmd` 顺序查找，并校验 `.cmd` shim 内容是否为官方 npm 启动器（`mnemonNpmLauncher`）。
- `lib/index.js:1149` 的 `currentCommand()` = `findMnemonCommand(config) ?? config.cliPath ?? "mnemon"`。
- `lib/index.js:1199-1202` 的 `commandFound` getter 每次调用都重新执行 `findMnemonCommand()` + `isMnemonExecutable()`，**不缓存**，所以安装后无需重启宿主进程。
- 启动失败时的提示只在 `reason === "launch"` 时附带安装指引；超时、取消、输出超限各有独立文案。

## embedding 的可选性（重要）

dsh-mnemon 会把保存的配置通过环境变量注入子进程：`MNEMON_EMBED_ENDPOINT` / `MNEMON_EMBED_MODEL` / `MNEMON_EMBED_API_KEY` / `MNEMON_EMBED_PROTOCOL`（`lib/index.js:1138-1146`，会先过滤宿主同名变量，保证配置优先；协议为 `auto` 时不注入）。

实测（本机未装 Ollama，端点 `http://localhost:11434` + `nomic-embed-text` 不可达）：

- `mnemon remember` 仍返回 `"action": "added"`、`"embedded": false`，退出码 0；
- `mnemon search` 仍能返回结果，靠关键词与图谱；
- 图谱边照常建立：`edges_created` 中 `temporal` 递增，`semantic_candidates` 出现 `similarity: 1` 但 `auto_linked: false`（无向量时相似度不可信）。

即：装 Ollama + `nomic-embed-text`（约 1 GB）只是「打开向量语义召回」，不是可用性前提。

## Windows 沙箱带来的坑

在 DSH workspace-write 沙箱下，`mnemon` 打不开工作区外的数据目录：

```
open database: create data dir: mkdir C:\Users\XYD\.mnemon\data: Access is denied.
```

同样地，在 workspace 内 `D:\DeepSeek Harness\插件和软件安装与配置\.tmp\mnemon-probe` 初次 `mkdir` 也可能被拒（`.tmp` 的 ACL 给出 Authenticated Users `Modify`，但 `New-Item` 仍报访问被拒；先写文件再建目录可绕过）。

排查要点：任何 `mnemon` 探针命令都显式传 `--data-dir <工作区内路径>`，否则容易被误判为「CLI 坏了」。

## 验证清单

1. `mnemon version` → `mnemon version 0.2.12`；
2. `mnemon remember "..." --cat fact --data-dir <工作区内目录>` → `"action": "added"`；
3. `mnemon search "..."` 或 `mnemon status` 返回 JSON；
4. 重开 DSH 设置页 → 记忆空间 → Provider，确认「未安装 CLI」标记消失，再新建记忆空间。

## 未决事项

- 是否安装 Ollama + `nomic-embed-text` 以启用向量召回（用户尚未决定）。
- 是否单独部署 memos（默认端口 5230，与 DSH 记忆系统不互通）。
- `D:\Docker Desktop Installer.exe`（641 MB）疑似冗余，本机 `docker --version` 已可用（29.9.2）。
