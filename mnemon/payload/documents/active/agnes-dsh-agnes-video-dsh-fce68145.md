---
id: "fce68145-1059-4e75-9f68-48f5ad8251dc"
title: "Agnes 视频生成 DSH 插件（agnes-video-dsh）实现与接入结论"
description: "把 Agnes Video 2.5 Flash 接入 DSH 的插件结构、双工具异步模型、503 重试与超时取舍、密钥解析顺序，以及真机验证证据。"
status: "active"
created_at: "2026-10-23T14:14:08.738Z"
updated_at: "2026-10-23T14:14:08.738Z"
content_hash: "ebfb574b42106f3f4c984f83dbd5f685d63b4fdad6e219c9763e72d136e3ed73"
source_paths:
  - "插件和软件安装与配置/agnes-video-dsh/index.js"
  - "插件和软件安装与配置/agnes-video-dsh/lib/agnes.mjs"
  - "插件和软件安装与配置/agnes-video-dsh/lib/agnes-video.mjs"
  - "插件和软件安装与配置/agnes-video-dsh/cli.mjs"
  - "插件和软件安装与配置/agnes-video-dsh/cordis.patch.yml"
  - "插件和软件安装与配置/agnes-video-dsh/README.md"
  - "插件和软件安装与配置/agnes-video-dsh/package.json"
session_ids:
  - "717aef80-36a3-4f5c-8ccf-ec790c1a1eed"
memory_body_ids:
  []
---

# Agnes 视频生成 DSH 插件（agnes-video-dsh）

面向 DSH 宿主的本地插件包，注册 `agnes_video`（建任务）与 `agnes_video_status`（查询并下载）两个工具，把 Agnes Video 2.5 Flash 的文生视频 / 首尾帧 / 多模态参考能力接进对话。结构参照同目录的 `agnes-image-dsh`（见文档 `57c4e049-ab98-46e4-b013-b3e85bc6549f`）。

插件目录：`插件和软件安装与配置/agnes-video-dsh/`

## 包结构

| 文件 | 职责 |
| --- | --- |
| `lib/agnes.mjs` | 共享层：HTTP 请求、鉴权头、有限重试、错误映射、任务轮询、文件落盘（网络与文件动作只在这里） |
| `lib/agnes-video.mjs` | 视频业务：create / status / download，含 `DEFAULT_MAX_WAIT_MS` 等常量 |
| `index.js` | 宿主层：`ctx.tools.register` 注册 `agnes_video`、`agnes_video_status` |
| `cli.mjs` | 命令行入口，与宿主工具共用同一套 core |
| `package.json` | `private: true`、`type: module`、`main`/`exports` 指向 `index.js`、版本 0.1.0 |
| `cordis.patch.yml` | 模型、`timeoutMs`、`pollMs`、`maxWaitMs`、`retries` 配置 |
| `README.md` | 参数、两种工具、CLI 示例、密钥配置方式 |

## 双工具异步模型（关键设计）

生成是异步且耗时的（实测单条 4s/720P 片段约 210–423s），因此**创建与等待必须拆成两个工具**：

1. `agnes_video`：`POST /v1/videos` 建任务后**默认立即返回** `{ videoId, status: queued, progress }`，不阻塞对话；`wait: true` 才同步等到底。
2. `agnes_video_status`：按 `video_id` 查询 `{ status, progress, url }`，完成时自动下载 MP4 到 `agnes-videos/` 并返回绝对路径；未完成只回报进度，可反复调用。

参数约束：`mode` = `text` / `keyframe`（`first_frame`、`last_frame`）/ `reference`（`images` ≤5、`audios` ≤3，**必须是公网 http(s) URL**，本地文件传不上去）；Flash 仅 720P；`seconds` 4–12；`aspect_ratio` 16:9 / 4:3 / 1:1 / 3:4 / 9:16 / 21:9；`seed` 可复现。

## 抗 503 与超时取舍（踩过的坑）

- `HTTP 503 视频队列已满，请稍后重试` 是高频瞬时错误，宿主工具层同样会撞到，所以**重试必须落在 `lib/` 的 core**，只在 CLI 或 shell 循环里加重试不够。
- 重试有限且可配置：`retries` 默认 3，退避 8s / 16s / 24s（503 走更长退避），条件 `retryable(status) && attempt <= retries`；**404 / `任务不存在` 绝不重试**，那是任务 id 错。
- 旧 `DEFAULT_MAX_WAIT_MS = 480_000` 太紧（单次生成已耗 423s），导致长任务在同步调用里被中断，表现为 `Error: [object Object]`。现为 `900_000`，且必须与 `cordis.patch.yml` 的 `maxWaitMs: 900000` **同步修改**，否则宿主侧仍按旧值超时。
- 长耗时联调放后台 job，用短而多次的 `job_output` 轮询；长时间 `wait` 会以 `wait aborted` 结束，那是等待本身死亡，不是任务失败。

## 密钥解析顺序

`config.apiKey` → `AGNES_API_KEY` → `<DSH_HOME>/agnes-video.json` → 回退 `<DSH_HOME>/agnes-image.json`（复用图片插件的 key）。**不把 key 写进 `cordis.patch.yml` 等可提交文件**；缺失时 core 抛出带三种配置方式的明确错误。

## 安装与验证

- 安装：`dsh.cmd plugin --profile desktop add "link:<插件绝对路径>"`，期望 `EXIT=0` 与 `Done`（`[WARN] Issues with peer dependencies found` 为既有告警，不阻断）；每次改 core 后需重装或重启宿主。
- 接线探测：用不存在的 id 调 `agnes_video_status`，远端返回 `任务不存在 (HTTP 404)` **即接线成功**（证明工具已注册、鉴权头已带、URL/方法正确），不要当失败去改代码。
- 真机证据（均为宿主工具三步闭环，非 CLI 兜底）：`task_WsG6tFJFmjhUbq3GuMp87xEuUxzr7lly` → `agnes-video-20261023-215132.mp4`（347,622 B）、`task_YGVrY1YqZiIjpo6JzaqpuhwY2BAQ2ykf` → `agnes-video-20261023-220213.mp4`（196,910 B）、竖屏请求 `task_ulP0v7oyTM3QVfwNc9t2XoEPyqYayWeI` → `agnes-video-20261023-221317.mp4`（2,977,768 B，9:16 / 5s / 720P）。产物校验用 `Get-ChildItem *.mp4 | Select Name,Length` 并确认 `magic=ftyp`（文件头 `ftypisom`），大小远大于 0。

## 用法

对话里直接描述需求（如「生成一段猫在雨夜街头行走的视频，5 秒，竖屏」），或命令行：`node cli.mjs -p "<prompt>" -t 5 -r 9:16`；只轮询既有任务并下载：`node cli.mjs -p "<prompt>" -t 4 --video-id task_xxx`。产物统一落在 `插件和软件安装与配置/agnes-videos/`。
