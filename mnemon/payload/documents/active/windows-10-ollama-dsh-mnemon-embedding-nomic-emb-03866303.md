---
id: "03866303-cfe2-4a88-aa12-603abb3aedce"
title: "Windows 10 上安装 Ollama 并为 DSH/mnemon 打通本地 embedding（nomic-embed-text + 隐藏自启动）"
description: "Ollama 0.40.9 安装、嵌入模型拉取与三维度验证、ollama app.exe 起不来服务的坑、HKCU Run 隐藏自启替代计划任务与启动文件夹冲突的处理，含验收命令。"
status: "active"
created_at: "2026-10-23T16:48:00.663Z"
updated_at: "2026-10-23T16:48:00.663Z"
content_hash: "7d7f92b77b8795d77ed6cdb2ed7564322b64df431bc0d266283f86bbdf400d4d"
source_paths:
  - "ollama-install/start-ollama-hidden.vbs"
  - "ollama-install/OllamaSetup.exe"
session_ids:
  - "89df850f-fe4a-4895-8a7f-2247212587a5"
memory_body_ids:
  []
---

# Windows 10 上安装 Ollama 并为 DSH/mnemon 打通本地 embedding

目标：让 dsh-mnemon 记忆空间使用真实的本地嵌入（而非降级），使写入返回 `embedded: true`、相似度与 `semantic` 边可用。前置结论（见文档 `75ad464f-1af0-4059-9bae-eec491d6c90e`）：Ollama 不是 mnemon 的必需依赖，缺失时写入仍成功，只是没有向量与语义边。

## 最终成果

| 项 | 值 |
|---|---|
| Ollama | 0.40.9，安装在 `C:\Users\XYD\AppData\Local\Programs\Ollama`（`ollama.exe`、`ollama app.exe`、`lib\`） |
| 安装包 | `ollama-install/OllamaSetup.exe`，1,662,271,480 字节；Inno Setup 6.7.0，manifest `requestedExecutionLevel level="asInvoker"` |
| 嵌入模型 | `nomic-embed-text:latest`，274 MB（274,302,450 B），digest `0a109f422b47e3a30ba2b10eca18548e944e8a23073ee3f3e947efcf3c45e59f`，`family=nomic-bert`，137M 参数，F16，`embedding_length: 768`，context 2048，`capabilities: ["embedding"]` |
| 服务端点 | `127.0.0.1:11434`，三个 API 均返回 768 维 |
| mnemon 侧 | `mnemon embed --status` → `protocol: "ollama"`, `ollama_available: true`, `embedding_available: true`, `coverage: 100%` |

## 端到端验收（可复用命令集）

1. 三种嵌入 API 都能用，说明不依赖单一协议分支：
   - `POST /api/embed` → 返回 keys `model,embeddings,total_duration,load_duration,prompt_eval_count`，`count=1 dims=768`
   - `POST /api/embeddings`（legacy）→ `dims=768`
   - `POST /v1/embeddings`（OpenAI 兼容）→ `dims=768`
2. mnemon 真实链路：连续 `mnemon remember` 两条相关事实，返回 `embedded: true` 且 `semantic_candidates` 给出相似度 `0.7903`（真实向量算出，非同句捷径）；再写一条近似重复句，相似度 `≈1.0` 时 `auto_linked: true` 并建出 `semantic` 边 —— 证明向量确实落库并参与图构建。
3. DSH 设置页「测试状态」走的是真实数据目录 `C:\Users\XYD\.mnemon`：`coverage: "0%", embedded: 0, total_insights: 0` 是空集占空集，不是故障；`embedding_available` 与 `ollama_available` 为 `true` 即代表探测通过，存第一条记忆后才会变成 `1/1`。

## 坑 1：`ollama app.exe`（托盘）拉不起服务

现象：开机或手动启动托盘后，`app.log` 反复出现
`msg="ollama server not ready, continuing anyway" error="timeout waiting for Ollama server to be ready"`，
最终 `msg="ollama server not ready after retries"`；`server.log` 一个字节都没写；进程列表只有 `ollama app.exe`，没有 `ollama serve` 子进程；11434 未监听。

结论：该版本托盘程序负责 UI（`ui.go` 起本地 web 端口）但没有成功 spawn 服务端，所以「靠官方自启」在每个开机日都白启动一次。直接 `ollama serve` 则完全正常（启动后 `/api/version` 返回 `{"version":"0.40.9"}`，端口 LISTENING）。

诊断要点（下次同类问题照做）：`netstat -ano | findstr 11434`、`Get-Process ollama*`、`app.log`/`server.log` 尾部、`ollama --version` 会打印 `Warning: could not connect to a running Ollama instance`。注意 `server.log` 为空比报错更有信息量——说明服务进程压根没启动。

## 坑 2：非管理员无法注册计划任务

`Register-ScheduledTask -TaskName OllamaEmbedServer` 报 `Access is denied. HRESULT 0x80070005`（当前会话非管理员）。替代方案是等价的 HKCU Run 键（无需提权）：

```
HKCU\Software\Microsoft\Windows\CurrentVersion\Run
  OllamaEmbedServer = wscript.exe "D:\DeepSeek Harness\插件和软件安装与配置\ollama-install\start-ollama-hidden.vbs"
```

`ollama-install/start-ollama-hidden.vbs` 以隐藏窗口方式执行 `ollama serve`，避免每次开机弹黑框。

## 坑 3：启动文件夹里的官方快捷方式会打架

安装程序在「启动」文件夹放了 `Ollama.lnk`，会和上面的 Run 键抢着起服务，并留下一个永远「未就绪」的托盘。处理方式是把扩展名改成 `Ollama.lnk.disabled`（改回来即可恢复），让唯一的启动路径是隐藏 `ollama serve`。改完后启动文件夹为 `HuyaExternal.lnk / Ollama.lnk.disabled / 发送至 OneNote.lnk`。

## 验证「模拟重启」

杀掉全部 ollama 进程 → 端口 11434 = False → 执行 logon launcher（等价于开机 Run 键）→ 端口 = True、仅一个 `ollama` 进程、`embed_dims=768`。这条闭环比「装完即宣称成功」可靠，因为它验证的是开机路径而不是当次会话的后台作业。

## 坑 4：模型下载卡死与断点续传

`registry.ollama.ai` 的速度从 2.4 MB/s 掉到 96 KB/s 后停滞在 44%。Ollama 的分块下载会保留 `sha256-<digest>-partial*` 文件，取消作业后重新 `ollama pull` 会从断点续传（日志显示从 264 MB/274 MB 的 96% 直接收尾，3.3 MB/s），最终 `verifying sha256 digest / writing manifest / success`。因此卡住时不要删 blobs 目录重下，直接重启 pull。

进度可通过 `~/.ollama/models/blobs` 里 partial 文件大小 + `GET /api/tags`（下载中 `models: []`）判断。

## 已知琐碎事项

- `MNEMON_EMBED_PROTOCOL=auto` 会触发 `warning: invalid MNEMON_EMBED_PROTOCOL "auto", falling back to auto-detect`（该值不在 `MNEMON_EMBEDDING_PROTOCOLS` 白名单内）。DSH 在 `protocol === "auto"` 时**不注入**该环境变量（`index.js` 中的条件展开 `...config.embedding.protocol === "auto" ? {} : {...}`），所以真实调用不会出现此警告；仅手动测试时才会。
- `ollama-install/OllamaSetup.exe`（1.5 GB）与已验证的 `ollama-install/OllamaSetup.exe` 同目录产物可在确认无问题后删除回收空间。
- 之前尝试静默安装均失败：`/VERYSILENT` 之类调用 3–5 秒退出（EXIT=1）、`/LOG=` 零字节、`install.log missing`、`installed? False`；GUI 子系统进程立即返回后随 pwsh 结束被杀。最终由用户手动双击图形安装完成 —— 这类 Inno GUI 安装包在本机不要指望非交互脚本化。
