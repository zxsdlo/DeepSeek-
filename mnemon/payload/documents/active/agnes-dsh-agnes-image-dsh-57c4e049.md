---
id: "57c4e049-ab98-46e4-b013-b3e85bc6549f"
title: "Agnes 图像生成 DSH 插件（agnes-image-dsh）实现说明与接入排错"
description: "将 Agnes 文生图/图生图接入 DSH 的插件包结构、Agnes API 请求体约束、profile 挂载步骤，以及 401 无效令牌的排查路径。"
status: "active"
created_at: "2026-10-23T12:42:15.912Z"
updated_at: "2026-10-23T12:42:15.912Z"
content_hash: "da6ac9a17c79986a6353a355f257579c3f12299aa09913a896202f00d99d3d0f"
source_paths:
  - "插件和软件安装与配置/agnes-image-dsh/index.js"
  - "插件和软件安装与配置/agnes-image-dsh/lib/agnes.mjs"
  - "插件和软件安装与配置/agnes-image-dsh/cli.mjs"
  - "插件和软件安装与配置/agnes-image-dsh/cordis.patch.yml"
  - "插件和软件安装与配置/agnes-image-dsh/README.md"
session_ids:
  - "2e2dc4f7-e894-457d-94df-514d35cffe49"
memory_body_ids:
  []
---

# Agnes 图像生成 DSH 插件（agnes-image-dsh）

面向在 DSH（DeepSeek Harness）中注册 `agnes_image` 工具的本地插件包，零依赖，只使用 `ctx.tools`。用途：对话里直接出图、图生图，并保存到会话工作目录。

## 包结构

插件目录：`插件和软件安装与配置/agnes-image-dsh/`

| 文件 | 职责 |
| --- | --- |
| `index.js` | Cordis 插件入口，注册 `agnes_image` 工具；导出 `apply` / `inject` / `name`，`inject: ['tools']` |
| `lib/agnes.mjs` | 请求构造、响应解析、下载保存、重试 |
| `cli.mjs` | 不重启 DSH 也能验证的命令行版 |
| `cordis.patch.yml` | 插件条目与配置（baseURL、model、key、超时、输出目录） |
| `README.md` | 安装、用法、排错 |
| `package.json` / `cordis.yml` | 包元数据与插件声明 |

## Agnes API 请求体约束（踩坑结论）

端点：`POST https://api.agnes-ai.cn/v1/images/generations`（`GET` 返回 404 `Invalid URL`，可用于确认路径存在且只接受 POST）。

确认可用的字段形状：

- 文生图：`{"model":"agnes-image-2.5-flash","prompt":"x","size":"1K","extra_body":{"response_format":"url"}}`
- 图生图：`{"model":"...","prompt":"edit","size":"1024x768","extra_body":{"response_format":"url","image":["https://.../a.png"]}}`
- 精确像素（`1920x1080` 同理，走 `size`）

三条必须遵守的规则：

1. 输入图只放 `extra_body.image`，不要放顶层 `image`。
2. `response_format` 只放 `extra_body`，不要放顶层。
3. 不发 `tags: ["img2img"]`。
4. `ratio` 仅与 `1K`/`2K`/`3K`/`4K` 档位同发；给精确像素尺寸（如 `1024x768`）时不要发 `ratio`，避免冲突。

本地输入图路径会自动转 Data URI 后放入 `extra_body.image`。输出默认写入会话工作目录的 `agnes-images/`，返回绝对路径，便于 `![](路径)` 预览。

## 挂载到 DSH profile

用官方 CLI 加本地包（软链，非拷贝）：

```
dsh plugin --profile desktop add "link:D:/.../agnes-image-dsh"
```

结果：`node_modules/agnes-image-dsh` 为 Junction，且 `package.json` 的 `dsh.profile.bundles` 里追加 `agnes-image-dsh`。改动 profile 配置前先备份其 4 个文件到 `.tmp/profile-backup/`。`pnpm install` 走 supply-chain 策略校验，成功提示 `Already up to date`。

## 阻塞点与排查记录

现象：直连官方端点返回 `HTTP 401`，消息 `无效的令牌 (request id: ...)`，`code: ""`，`elapsedMs` 约 169ms。

已排除的方向（全部仍 401）：`Bearer`、裸 key、`x-api-key` 三种鉴权写法；备用域名 `api.agnes-ai.com`（其根路径也是 404，非鉴权错误）；插件自身请求体形状已单独构造验证正确。官方错误码文档对 401 的解释为「API Key 不正确 / 已过期 / 配置不正确 / 属于其他账户」。

结论：属于 key 层面的问题，不是端点或请求体问题。恢复方式（均无需改代码，只需改配置来源）：

- 在 Agnes 控制台重新生成 key；
- 写入 `%DSH_HOME%/agnes-image.json` 的 `{"apiKey":"sk-..."}`；
- 或设环境变量 `AGNES_API_KEY`。

密钥不要写进版本库或 `cordis.patch.yml`；优先用 `agnes-image.json` 或环境变量。

## 验证清单

1. `node -e "import('./index.js').then(m=>console.log(m)"` 确认导出 `apply/inject/name` 且 `inject: ['tools']`。
2. 命令行：`node cli.mjs --prompt "..." --size 2K --ratio 16:9`。
3. 会话内：直接说「画一张……」，检查 `agnes-images/` 下生成的绝对路径与可预览图片。
