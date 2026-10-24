---
id: "e467ec7b-1942-46b3-96f5-ad3ce6c2d4d4"
title: "Watt Toolkit 拦截证书导致 DSH 的 Node 进程不信任 GitHub（含 PortableGit 缺失）修复记录"
description: "DSH 记忆空间 GitHub 登录报 UNABLE_TO_GET_ISSUER_CERT_LOCALLY 的根因（Watt Toolkit/Steam++ 本地反代自签 CA，Node 不读 Windows 证书库）、NODE_EXTRA_CA_CERTS 修复步骤、PortableGit 2.56 便携安装与验证方法，以及 PowerShell 5.1 无 BOM 编码坑。"
status: "active"
created_at: "2026-10-24T07:41:44.745Z"
updated_at: "2026-10-24T07:41:44.745Z"
content_hash: "7d82c24871a364f4722c6c6d3ed6a5feea3310db847bc23b5321012cecdd0bf5"
source_paths:
  - "git-install/setup-git-and-ca.ps1"
session_ids:
  - "3424050a-6924-4f0e-96d5-26aa5789cd45"
memory_body_ids:
  []
---

# Watt Toolkit 拦截证书 + Node 信任链：DSH GitHub 登录/同步修复记录

## 现象

DSH 设置 → 记忆空间 → 同步，点「使用 GitHub 登录」失败；浏览器里 GitHub 登录完全正常。报错形如
`UNABLE_TO_GET_ISSUER_CERT_LOCALLY`，并附带一句
`could not reach github.com: ... ; this Host has no proxy for GitHub, so set HTTPS_PROXY to a proxy that can reach it`。
另有独立黄条提示「同步需要 Git 2.20 或更高版本」——那条**不是误报**，机器上当时确实没有 git。

## 根因链

1. 机器上运行着 **Watt Toolkit（原 Steam++）**：它把 `github.com` 等域名经 hosts 指向 `127.0.0.1`，用自签 CA 做本地 TLS 反代。
   实测证书链：`Steamcommunity302 - ECC Intermediate ← Steamcommunity302 - 2026 ECC Root`。
2. 该根证书被装进 **Windows 证书库**，所以浏览器信任、登录正常。
3. DSH 宿主是 **Node.js**，Node 只认自己内置（或 `NODE_EXTRA_CA_CERTS` 指定）的受信 CA 列表，**不读 Windows 证书库** → 判定签发者未知。
4. 插件把这个证书错误**误判成网络不可达**：`dsh-mnemon` 的 ambient `fetch` 失败后，错误文本含 `"fetch failed"`，命中 `isTransportFailure` 的 `TRANSPORT_MARKERS`，于是转入「从环境变量/Windows 系统代理里找代理」的分支；找不到就拼出那句 `set HTTPS_PROXY` 提示。
   → **那句提示是症状描述，不是病因；不要顺着它去配代理。**

## 修复（Node 侧）

1. 从证书库导出 Steamcommunity302 的 4 张根证书，合并为 PEM：`C:\Users\XYD\.dsh\certs\watt-toolkit-ca.pem`（5046 字节）。
2. 设**用户级**环境变量 `NODE_EXTRA_CA_CERTS` = 该 pem 路径。
3. 验证：在读取用户级环境变量的**新进程**里 `fetch("https://api.github.com")` → `200`；`fetch("https://github.com")` → `200`。
4. 必须**完全退出 DSH 再重开**，环境变量只对新进程生效。

**顺序陷阱**：先放好 pem，再写环境变量。变量指向不存在的文件会让此后所有新启动的 Node 进程直接启动失败（本会话第一次尝试就因为中文路径乱码复制失败、而环境变量已写入，出现过这个状态）。

## 修复（git 侧）

- 机器原本完全没有 git；MSI/安装包需要提权，改用**便携版**：Git for Windows `PortableGit-2.56.0.3-64-bit.7z.exe`（约 60 MB，自解压 7z）。
- 完整性核对：Authenticode `Status=Valid`，签名者 `CN=Johannes Schindelin`，签发者 `CN=Microsoft ID Verified CS EOC CA 03`，SHA256 `4FC27B779829A36DF77911E3F4DEA09C66C790DD6C5F7DFA3B6A5479F1FE354F`。
- 解压到 `%LOCALAPPDATA%\Programs\PortableGit`（约 388 MB / 9616 文件），`cmd\git.exe` 存在即成功；`post-install.bat` 在 `cmd` 目录下不存在，**跳过它也能正常工作**。
- 用户 `PATH` 追加 `%LOCALAPPDATA%\Programs\PortableGit\cmd`。
- 验证：`git version` → `2.56.0.windows.3`；`git ls-remote https://github.com/git-for-windows/git.git` → `exit 0`（走 Watt Toolkit 反代也通）。
- 提交身份沿用已有 `~/.gitconfig`：`xuzipeng <xu3101457963@163.com>`。
- 工作区里的 450 MB 解压副本与 60 MB 安装包在复制到 AppData 后已删除。

## 可复跑脚本与坑

脚本：[`git-install/setup-git-and-ca.ps1`](git-install/setup-git-and-ca.ps1)（ASCII-only）。

- **PowerShell 5.1 编码坑**：无 BOM 的 UTF-8 `.ps1` 会被按 GBK 读取，中文路径（如 `插件和软件安装与配置`）变乱码 → `Copy-Item` 找不到源文件。修复脚本一律写成纯 ASCII，或加 BOM。
- `pwsh`（PowerShell 7）在这台机器上不存在，只能用 `powershell.exe` 5.1。
- 写 `C:\Users\XYD\.dsh` 与修改用户级环境变量（`SetEnvironmentVariable`，注册表 `HKCU\Environment`）都需要提权/审批；在审批被自动拒绝的受限会话里这两步会失败，需要由有权限的会话执行。
- `robocopy` 正常成功也返回 exit 1，不要当失败。

## 复现判断口诀

重启后若仍失败：
- 报 `ECONNREFUSED` → Watt Toolkit 没在运行，但 hosts 仍指向 `127.0.0.1`。
- 仍报证书 → `NODE_EXTRA_CA_CERTS` 未被宿主进程读到（未完全重启，或 Watt Toolkit 换了 CA）。
- 报找不到 git → 重开 DSH 让新 `PATH` 生效，点同步页「重新检查」。
