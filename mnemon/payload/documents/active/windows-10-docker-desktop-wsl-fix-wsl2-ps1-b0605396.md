---
id: "b0605396-a324-4156-9f37-8acd8f6de68f"
title: "Windows 10 上 Docker Desktop 启动失败的 WSL 缺失诊断与修复（Fix-WSL2.ps1）"
description: "Docker Desktop 报“WSL needs updating”实为 WSL 完全未安装的取证过程、修复脚本流程、重启续跑与网络栈限制。"
status: "active"
created_at: "2026-10-23T15:06:37.574Z"
updated_at: "2026-10-23T15:06:37.574Z"
content_hash: "bb2638518913f242969827495d249dd7a28e765ea87f732a3d60607473dd4f70"
source_paths:
  - "wsl-docker-fix/Fix-WSL2.ps1"
session_ids:
  - "f5e470bf-7e76-4bb9-82be-8fad9d3cebd7"
memory_body_ids:
  []
---

# Windows 10 上 Docker Desktop 启动失败的 WSL 缺失诊断与修复

## 现象与结论

Docker Desktop 启动弹窗报 *"Your version of Windows Subsystem for Linux (WSL) is too old. Run the command below to update"*。

**实际根因不是版本旧，而是这台机器根本没有安装 WSL。** Docker 的提示文案具有误导性，直接照提示跑 `wsl --update` 不会解决问题。修完 WSL 才轮到讨论 Docker 版本兼容。

## 取证证据（本机实测）

| 检查项 | 结果 | 含义 |
| --- | --- | --- |
| 系统 | Windows 10 Pro 22H2，Build 19045.6466 | 满足 WSL 2 最低要求（x64 需 19041+） |
| `C:\Windows\System32\lxss` | 仅有一个 `lib` 子目录，无 `bash.exe`、无 kernel | WSL 可选功能未启用 |
| `LxssManager` 服务 | NOT FOUND | 功能启用后才会创建 |
| `WslService` 服务 | NOT FOUND | 新版 WSL 包未安装 |
| `C:\Program Files\WSL\wsl.exe` | 不存在 | 未安装 Store/MSI 版 WSL |
| `System32\wsl.exe` | 存在，但只支持 `--install/--list/--status/--help`，无 `--version`、`--update` | 空壳占位，非可用的 WSL 运行时 |
| `wsl --status` | 退出码 50 | 未安装 |
| `vmcompute` / `HvHost` | Running，StartType=3 | VirtualMachinePlatform 侧正常，虚拟化没问题 |
| Docker Desktop | 4.96.3.244472 | 要求 WSL 2.x |

## 修复方案

一键脚本：`wsl-docker-fix/Fix-WSL2.ps1`（已通过 `SYNTAX OK` 校验）。运行方式：

```powershell
powershell -NoProfile -ExecutionPolicy Bypass -File "D:\DeepSeek Harness\插件和软件安装与配置\wsl-docker-fix\Fix-WSL2.ps1"
```

脚本流程：

1. 自弹 UAC 提权后重新执行自身（避免用户在非管理员 shell 里反复试）。
2. `dism` 启用 `Microsoft-Windows-Subsystem-Linux` 与 `VirtualMachinePlatform` 两个可选功能；若需要重启则写入 `.reboot-required` 标记并退出。
3. 再次运行时检测到标记 → 跳过启用步骤，直接装 WSL 包：优先 `wsl --update`，失败则从 `github.com/microsoft/WSL` releases 里挑最新 **2.x** 的 `*.x64.msi` 静默安装（3.x 可能仅支持 Win11，Win10 上要固定 2.x）。
4. `wsl --set-default-version 2`，最后打印验证结果。日志写入同目录 `fix-wsl2.log`。

手动等价命令（管理员 PowerShell）：

```powershell
dism /online /enable-feature /featurename:Microsoft-Windows-Subsystem-Linux /all /norestart
dism /online /enable-feature /featurename:VirtualMachinePlatform /all /norestart
# 重启
wsl --set-default-version 2
wsl --install -d Ubuntu --no-launch
```

收尾：打开 Docker Desktop，确认 General 里 **Use the WSL 2 based engine** 已勾选；Docker 需要至少一个已注册发行版（所以脚本顺带装 Ubuntu，仅 `--no-launch`）。

## 关键取舍与限制

- **代理/网络栈限制**：该会话的 shell 拿不到 TLS 凭证，`curl` 报 `schannel: SEC_E_NO_CREDENTIALS (0x8009030E)`，对外 HTTPS（github、agnes）全部失败；系统代理配置为 `127.0.0.1:33210` 但 `ProxyEnable=0`。因此下载动作必须在用户自己的 PowerShell 里完成，Agent 只能生成并交付脚本、不能代跑需 UAC 的提权安装。
- `Get-AppxPackage` 在该会话报"拒绝访问"，所以无法用 Appx 查询判断 WSL 包状态，只能靠服务/目录/参数探测。
- 兼容性提示：Windows 10 22H2 已于 2025-10 结束支持，Docker 4.96 这类新版本对 Win10 不保证。若 WSL 装好后 Docker 仍报错，退到 ~4.35 版本或升级 Win11 更省事。

## 可复用经验

在 Windows 上排查"WSL 版本旧"类报错时，先判**是否存在**再判**是否旧**：看 `LxssManager`/`WslService` 服务、`System32\lxss` 内容、`wsl.exe --help` 暴露的参数集合、`wsl --status` 退出码——这四项比报错文案可靠。
