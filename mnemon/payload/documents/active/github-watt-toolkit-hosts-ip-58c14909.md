---
id: "58c14909-a097-41cc-af90-509624583416"
title: "GitHub 登录彻底打通（第二阶段）：Watt Toolkit 驱动/计划任务残留、hosts 归零与固定 IP 兜底"
description: "接续 e467ec7b 的修复记录：Watt Toolkit 卸载后仍由计划任务+内核驱动 WFP 劫持 DNS、进程被 explorer 反复拉起、hosts 被写成 0 字节、国内 DNS 返回的 github.com IP 不可达；给出提权清理脚本、hosts 固定可达 IP、node 端到端验证判据与 DSH 必须完全重启的结论。"
status: "active"
created_at: "2026-10-24T08:03:53.974Z"
updated_at: "2026-10-24T08:03:53.974Z"
content_hash: "a67891d46fda6b9e7e9da341e52190842b6a9d2f18218843dbf2a5e52dbdf0c4"
source_paths:
  - "cleanup-watt-toolkit.ps1"
  - "cleanup-watt-toolkit-2.ps1"
  - "finalize-hosts.ps1"
  - "hosts-diag.ps1"
  - "pin-github-hosts.ps1"
  - "update-github-pin.ps1"
  - "final-hosts2.ps1"
session_ids:
  - "4c85831b-89bc-4194-bb70-13e870afdd15"
memory_body_ids:
  []
---

# GitHub 登录彻底打通（第二阶段）：Watt Toolkit 残留与 hosts 固定 IP

> 前置记录见文档 `e467ec7b-1942-46b3-96f5-ad3ce6c2d4d4`（第一阶段：Watt Toolkit 自签 CA 让 Node 不信任 GitHub、PortableGit 2.56 便携安装、PowerShell 5.1 无 BOM 坑）。
> 本文记录用户「已经删掉 Watt Toolkit」之后仍复现失败的第二阶段：**证书不再是根因**，真实原因是卸载不彻底 + hosts 被改坏 + 国内 DNS 给出的 GitHub 边缘 IP 被阻断。
> 与第一阶段的关系：第一阶段加的 `NODE_EXTRA_CA_CERTS`（指向 Watt Toolkit 假 CA）在本文末尾已被**删除**，现在走真实 Sectigo 证书。

## 结论摘要

1. **Watt Toolkit 卸载不彻底的三层残留**才是主因：内核驱动 `steamcommunity_302_wfp` 仍 RUNNING、两个计划任务 `Steamcommunity302 V15` / `Steamcommunity302 V15 UI` 仍存在、三个进程被 `explorer.exe` 在杀完后重新拉起。
2. 残留驱动会**劫持 DNS**：即使 `nslookup github.com 223.5.5.5` 也返回 `127.0.0.1`，本地有进程在 443 冒充 GitHub 并出示假证书。所以「公共 DNS 解析结果」不能作为判断依据，必须看驱动/服务/任务。
3. **`hosts` 曾被写坏成 0 字节**（上一轮清理 `# S302 BEGIN…END` 段时整段写空），导致之后任何 hosts 修正都不生效——这是一度「改了也没用」的直接原因。
4. 清理干净后，国内 DNS 对 `github.com` 返回的 `20.205.243.166` **TCP 443 全部超时**（换 119.29.29.29 / 114.114.114.114 / 223.6.6.6 都是同一 IP），而 `api.github.com` 的 `20.205.243.168` 正常（~831ms）。故需要 hosts 把 `github.com` 固定到**实测可达**的 IP。
5. 最终判据：node（与插件同一套 fetch）`GET https://github.com/` → **200**、`POST https://github.com/login/device/code` → **200** 且返回真实 `device_code`/`user_code`。
6. 用户点「重新检查」仍报黄条的原因与网络无关：**DSH 主进程自 10-23 23:10 起从未重启**，新装的 git PATH 没进进程。

## 提权清理步骤（cleanup-watt-toolkit.ps1 → cleanup-watt-toolkit-2.ps1）

必须在管理员上下文执行（脚本自身 UAC 提权，`-Verb RunAs`；注意本机 `pwsh` 不在 PATH，提权要用 `powershell.exe`）。第一版脚本跑完后进程被 explorer 复活，第二版加了「杀完观察 45 秒」的守护检测，一次通过：

1. `schtasks /End` + `/Delete` 两个计划任务 → 均成功删除；
2. `Stop-Process -Force` 杀掉 `Steamcommunity_302.exe`、`steamcommunity_302.caddy`、`steamcommunity_302.cli`（非提权 taskkill 会被拒，进程属于 XYD 但已提权）；
3. `sc stop` + `sc delete steamcommunity_302_wfp` → 再查询报 `1060 指定的服务未安装`；`C:\Windows\System32\drivers\steamcommunity_302_wfp.sys` 已删除；
4. `ipconfig /flushdns`，然后**观察 45 秒**确认无复活（日志 `no respawn in 45s`）。

判据：`302 进程: 无` / `302 任务: 无` / `驱动文件: False` / `driver query: 1060`。

## hosts 最终内容（已稳定，8 秒复查无回写）

```
# hosts file for the github edge pins; localhost is handled by DNS itself.
127.0.0.1 localhost
::1 localhost
20.27.177.113 github.com
20.205.243.168 api.github.com
```

- `github.com` → `20.27.177.113`（实测 TLS 146ms，证书 `authorized=true`，issuer 为 Sectigo Public Server Authentication CA DV E36）；
- `api.github.com` → `20.205.243.168`（国内 DNS 原生可达，~831ms，保留原值）。
- 实测可达的其他候选（换 IP 时备选）：`20.200.245.247`(89ms)、`20.233.83.145`、`4.237.22.38`、`20.29.134.23`、`20.199.39.232`、`20.26.156.215`、`140.82.112.3`、`20.201.28.151`。实测不可达：`20.205.243.166`、`140.82.121.4`、`20.255.62.204`。
- 同时删除了用户级 `NODE_EXTRA_CA_CERTS`（假 CA 源已不存在）。

## 可复用的诊断顺序（下次 GitHub 又不通时照此走）

1. **先分清是 hosts 问题还是 DNS 劫持**：`nslookup github.com 223.5.5.5`。若公共 DNS 也返回 `127.0.0.1`，说明有本地驱动/进程冒充，先查驱动与服务：
   `Get-CimInstance Win32_SystemDriver | ? { $_.ImagePath -match '302|steam|watt' }`、`Get-ScheduledTask | ? TaskName -match '302|Steam|Watt'`。
2. **杀进程后必须观察 45 秒以上**；复活则用 `Get-CimInstance Win32_Process` 查 `ParentProcessId`（本案是 `explorer.exe`，说明外壳自启）。
3. **写 hosts 用 tmp+move**（`Set-Content tmp` → `Move-Item -Force`），写完隔 8 秒再读一次，确认没有守护进程回写；直接 `Set-Content` 曾有整段被写空成 0 字节的先例。
4. **换固定 IP 前先用 `TcpClient` 试目标 443**，只钉实测连通且有正确证书的 IP；hosts 钉死会随 GitHub 边缘节点变动而失效，届时重跑 `final-hosts2.ps1`（或 `pin-github-hosts.ps1`）换 IP 即可。
5. **端到端验证用 node**（与插件同一 fetch 栈）：`GET https://github.com/` 与 `POST https://github.com/login/device/code` 都要 200；curl 的 `(28) connect timeout` 与 node 结论可能不一致，以 node 为准。
6. **最后才动 DSH**：改 PATH / 装 git 之后必须**完全退出 DSH 再从托盘重开**（不是关窗口），否则旧进程里仍是旧环境变量，设置页「重新检查」永远报黄条。

## 坑与边界

- Watt Toolkit（Steam++ / `Steamcommunity_302` 套件）不要重装：其 WFP 内核驱动会改写 DNS 并清空 hosts，卸载时还会留下计划任务与驱动。
- `hosts` 被清成 0 字节时，`127.0.0.1 localhost` 一起丢失，属于连本地回环都受影响的状态，务必先修好基础四行再谈钉 IP。
- 固定 IP 只是兜底；若 Watt Toolkit 彻底卸载干净，理论上国内直连应可通，但当前默认解析（`20.205.243.166`）确实被阻断。
