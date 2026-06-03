---
title: OpenClaw WSL2 安装指南（Windows 10）
date: 2026-06-03 19:00:00
tags:
  - OpenClaw
  - WSL2
  - 教程
categories:
  - 技术
---

> 本指南详细介绍在 Windows 10 上通过 WSL2 安装 OpenClaw 的完整步骤，包含国内镜像源配置、飞书接入和大模型配置。

---

## 目录

1. [安装前提条件](#1-安装前提条件)
2. [WSL2 + Ubuntu 安装](#2-wsl2--ubuntu-安装)
3. [OpenClaw 安装](#3-openclaw-安装)
4. [Gateway 服务配置](#4-gateway-服务配置)
5. [飞书连接配置](#5-飞书连接配置)
6. [大模型配置](#6-大模型配置)
7. [常见问题与解决方案](#7-常见问题与解决方案)

---

## 1. 安装前提条件

### 1.1 系统要求

| 项目 | 要求 |
|------|------|
| 操作系统 | Windows 10 版本 2004 及以上（建议 Windows 10 22H2） |
| 内存 | 至少 4GB（推荐 8GB+） |
| 磁盘空间 | 至少 20GB 可用空间 |
| 处理器 | 64 位处理器 |

### 1.2 启用 Windows 虚拟化

**检查方法：**

1. 按 `Ctrl + Shift + Esc` 打开任务管理器
2. 切换到「性能」选项卡
3. 点击「CPU」，查看右下角「虚拟化：已启用」或「已禁用」

**如果显示「已禁用」：**

1. 重启电脑
2. 进入 BIOS（开机时按 `Del` 或 `F2`）
3. 找到「Virtualization Technology」或「Intel VT-x」选项
4. 设为 **Enabled**
5. 保存并退出 BIOS
6. 重新进入 Windows

### 1.3 启用 Windows 功能

以**管理员身份**打开 PowerShell，执行：

```powershell
# 启用相关 Windows 功能
dism.exe /online /enable-feature /featurename:Microsoft-Windows-Subsystem-Linux /all /norestart
dism.exe /online /enable-feature /featurename:VirtualMachinePlatform /all /norestart
dism.exe /online /enable-feature /featurename:Hyper-V /all /norestart
```

完成后**必须重启电脑**。

### 1.4 安装前的依赖准备

#### 1.4.1 安装 Node.js（版本 ≥ 22）

OpenClaw 需要 Node.js 22 及以上版本。

**方法一：直接下载安装包**

1. 访问 https://nodejs.org/
2. 下载 **LTS 版本**（推荐 v22.x 或 v24.x）
3. 运行安装包，全程默认下一步
4. **务必勾选**「Add to PATH」

**验证安装：**

打开 PowerShell，执行：

```powershell
node -v
npm -v
```

#### 1.4.2 安装 Git（用于源码和依赖管理）

1. 访问 https://git-scm.com/download/win
2. 下载 Windows 64 位安装包
3. 安装时保持默认选项，**确保勾选**：
   - 「Git Bash Here」
   - 「Add Git to PATH」
4. 完成安装

**验证安装：**

```powershell
git --version
```

---

## 2. WSL2 + Ubuntu 安装

### 2.1 安装 WSL2（自动方式）

以**管理员身份**打开 PowerShell，执行：

```powershell
wsl --install
```

> 此命令会自动安装 WSL2 内核和默认 Ubuntu 发行版。

**⚠️ 安装完成后必须重启电脑！**

### 2.2 首次设置 Ubuntu

重启后，Ubuntu 会自动启动，进入首次设置向导：

1. 输入用户名（建议使用英文，如 `openclaw`）
2. 输入密码（设置后确认）
3. 完成后进入 Linux 终端

### 2.3 配置 Ubuntu 国内镜像源（必须）

默认官方源在国内速度很慢，需要更换为国内镜像。

**备份原配置：**

```bash
sudo cp /etc/apt/sources.list /etc/apt/sources.list.bak
```

**替换为阿里云镜像（推荐）：**

```bash
sudo nano /etc/apt/sources.list
```

删除文件内容，粘贴以下内容：

```
# 阿里云镜像源（Ubuntu 22.04）
deb http://mirrors.aliyun.com/ubuntu/ jammy main restricted universe multiverse
deb http://mirrors.aliyun.com/ubuntu/ jammy-updates main restricted universe multiverse
deb http://mirrors.aliyun.com/ubuntu/ jammy-backports main restricted universe multiverse
deb http://mirrors.aliyun.com/ubuntu/ jammy-security main restricted universe multiverse
```

> 如果你使用的是 Ubuntu 24.04，将 `jammy` 替换为 `noble`。

保存并退出：`Ctrl + O` → 回车 → `Ctrl + X`

**更新软件包：**

```bash
sudo apt update && sudo apt upgrade -y
```

### 2.4 启用 systemd（OpenClaw 守护进程依赖）

OpenClaw 的 Gateway 服务依赖 systemd，必须启用。

```bash
sudo nano /etc/wsl.conf
```

粘贴以下内容：

```ini
[boot]
systemd=true
```

保存并退出，然后**重启 WSL**：

在 PowerShell 中执行：

```powershell
wsl --shutdown
```

重新打开 Ubuntu 终端，验证 systemd：

```bash
systemctl --version
```

如果看到版本号输出，说明 systemd 已启用成功。

---

## 3. OpenClaw 安装

### 3.1 安装方式一：一键脚本安装（推荐）

在 Ubuntu 终端中执行：

```bash
curl -fsSL https://openclaw.ai/install.sh | bash
```

### 3.2 安装方式二：npm 全局安装

如果已安装 Node.js 22+，可以直接用 npm：

```bash
# 设置国内 npm 镜像（加速下载）
npm config set registry https://registry.npmmirror.com

# 全局安装 OpenClaw
npm install -g openclaw@latest
```

### 3.3 网络超时问题解决方案

如果安装过程中网络超时（连接海外服务器困难），尝试以下方法：

**方法一：使用国内 npm 镜像**

```bash
# 设置淘宝镜像
npm config set registry https://registry.npmmirror.com

# 重新安装
npm install -g openclaw@latest
```

**方法二：使用代理（如果有）**

```bash
# 设置 HTTP 代理
export HTTP_PROXY=http://你的代理地址:端口
export HTTPS_PROXY=http://你的代理地址:端口

# 重新安装
npm install -g openclaw@latest
```

**方法三：手动下载预编译包**

如果网络实在不稳定，可以下载预编译的 Windows 一键安装包，跳过 WSL2 方式：

1. 下载地址：https://openclaw.ikidi.top/api/download/package/14
2. 解压到纯英文路径（如 `D:\OpenClaw`）
3. 双击 `OpenClaw Windows 一键启动.exe` 运行

### 3.4 验证安装

```bash
openclaw --version
```

正常输出应显示类似：`OpenClaw 2026.x.x`

---

## 4. Gateway 服务配置

### 4.1 执行初始化向导

```bash
openclaw onboard --install-daemon
```

跟随向导完成：
- 设置 Gateway 密码/Token
- 选择默认模型
- 配置渠道（可选）

### 4.2 启动 Gateway 服务

**前台运行（调试用）：**

```bash
openclaw gateway run --port 18789
```

**后台守护进程运行：**

```bash
openclaw gateway run --port 18789 --daemon
```

**查看服务状态：**

```bash
openclaw gateway status
```

### 4.3 设置开机自启（systemd）

创建 systemd 服务文件：

```bash
mkdir -p ~/.config/systemd/user
nano ~/.config/systemd/user/openclaw-gateway.service
```

粘贴以下内容：

```ini
[Unit]
Description=OpenClaw Gateway
After=network-online.target
Wants=network-online.target

[Service]
ExecStart=/usr/local/bin/openclaw gateway run --port 18789
Restart=always
RestartSec=5

[Install]
WantedBy=default.target
```

启用服务：

```bash
systemctl --user daemon-reload
systemctl --user enable --now openclaw-gateway.service
```

查看状态：

```bash
systemctl --user status openclaw-gateway.service
```

### 4.4 访问 Web 控制台

Gateway 启动后，在浏览器中打开：

```
http://localhost:18789
```

使用设置好的密码/Token 登录。

---

## 5. 飞书连接配置

### 5.1 创建飞书企业自建应用

1. 打开飞书开放平台：https://open.feishu.cn
2. 登录开发者后台
3. 点击「创建企业自建应用」
4. 填写应用名称（如 `OpenClaw 机器人`）
5. 点击创建

### 5.2 添加机器人能力

1. 进入应用详情页
2. 点击「添加应用能力」
3. 找到「机器人」，点击添加

### 5.3 配置应用权限

1. 进入「权限管理」
2. 点击「批量导入」
3. 粘贴以下 JSON：

```json
{
  "scopes": {
    "tenant": [
      "im:chat.access_event.bot_p2p_chat:read",
      "im:chat.members:bot_access",
      "im:message",
      "im:message:send_as_bot",
      "contact:user.employee_id:readonly"
    ]
  }
}
```

4. 确认开通权限

### 5.4 获取应用凭证

1. 进入「凭证与基础信息」
2. 记录：
   - **App ID**（格式：`cli_xxx`）
   - **App Secret**（点击眼睛图标查看）

### 5.5 配置事件订阅

1. 进入「事件与回调」
2. 设置请求地址为：
   ```
   http://你的服务器IP:18789/webhook/feishu
   ```
3. 启用以下事件：
   - `im.message.receive_v1`

### 5.6 发布应用

1. 进入「版本管理与发布」
2. 创建新版本 v1.0.0
3. 提交发布（个人版无需审核，自动通过）

### 5.7 OpenClaw 端配置飞书

编辑 `~/.openclaw/openclaw.json`，在 `channels` 中添加：

```json
{
  "channels": {
    "feishu": {
      "enabled": true,
      "appId": "cli_xxxxxxxxxxxx",
      "appSecret": "xxxxxxxxxxxxxxxx",
      "connectionMode": "websocket"
    }
  }
}
```

> ⚠️ 注意：如果通过 npm 安装了 `@openclaw/feishu` 插件，只需配置 `channels` 即可，不要重复配置插件。

重启 Gateway：

```bash
openclaw gateway restart
```

---

## 6. 大模型配置

### 6.1 支持的模型提供商

OpenClaw 支持多种大模型：

| 提供商 | 模型 | 说明 |
|--------|------|------|
| DeepSeek | deepseek-chat, deepseek-reasoner | 性价比高 |
| Kimi (Moonshot) | moonshot-v1-8k/32k/128k | 长上下文 |
| 智谱 GLM | glm-4/glm-4-flash | 中文理解强 |
| MiniMax | minimax-01-32k | 图文混合处理 |
| OpenAI | GPT-4o, GPT-4o-mini | 国际模型 |
| Anthropic | Claude 3.5/3.7 | 高质量推理 |

### 6.2 配置 DeepSeek（推荐国内用户）

1. 前往 https://platform.deepseek.com/ 注册账号
2. 在 API Keys 中创建新密钥
3. 复制保存 API Key

编辑 `~/.openclaw/openclaw.json`：

```json
{
  "models": {
    "providers": {
      "deepseek": {
        "apiKey": "sk-xxxxxxxxxxxxxxxxxxxxxxxx",
        "apiBase": "https://api.deepseek.com/v1"
      }
    },
    "primary": "deepseek-chat"
  }
}
```

### 6.3 配置七牛云 MaaS（国内加速）

七牛云提供国内优化的模型接入，无需翻墙：

```json
{
  "models": {
    "providers": {
      "qiniu": {
        "apiKey": "你的七牛云 API Key",
        "apiBase": "https://api.qnaigc.com/v1"
      }
    },
    "primary": "qiniu/deepseek-v3.2-251201"
  }
}
```

### 6.4 配置 Kimi（长上下文场景）

```json
{
  "models": {
    "providers": {
      "kimi": {
        "apiKey": "sk-xxxxxxxxxxxxxxxxxxxxxxxx",
        "apiBase": "https://api.moonshot.cn/v1"
      }
    },
    "primary": "moonshot-v1-128k"
  }
}
```

### 6.5 配置 OpenAI / Claude（需要代理）

```json
{
  "models": {
    "providers": {
      "openai": {
        "apiKey": "sk-xxxxxxxxxxxxxxxxxxxxxxxx",
        "apiBase": "https://api.openai.com/v1"
      },
      "anthropic": {
        "apiKey": "sk-ant-xxxxxxxxxxxxxxxxxxxxxxxx",
        "apiBase": "https://api.anthropic.com/v1"
      }
    },
    "primary": "gpt-4o"
  }
}
```

### 6.6 验证模型配置

```bash
# 测试模型连接
openclaw model test

# 查看当前配置
openclaw config show
```

---

## 7. 常见问题与解决方案

### Q1: WSL2 安装卡在 0x800701bc 错误

**原因：** WSL2 内核组件未更新

**解决：**

1. 下载 WSL2 内核更新包：https://wslstorestorage.blob.core.windows.net/wslblob/wsl_update_x64.msi
2. 双击安装
3. 重启 PowerShell，执行 `wsl --update`

### Q2: npm 安装网络超时

**解决：**

```bash
# 切换到国内镜像
npm config set registry https://registry.npmmirror.com

# 清理缓存后重试
npm cache clean --force
npm install -g openclaw@latest
```

### Q3: 飞书机器人无响应

**排查步骤：**

1. 确认 OpenClaw Gateway 正在运行：`openclaw gateway status`
2. 检查飞书应用是否已发布（个人版提交后自动通过）
3. 确认 App ID 和 App Secret 正确
4. 检查事件订阅 URL 是否可访问

### Q4: 模型 API 调用失败

**排查步骤：**

1. 检查 API Key 是否正确
2. 确认 API Base URL 格式正确（注意尾斜杠）
3. 检查网络是否能访问目标 API 服务器
4. 查看日志：`openclaw gateway --verbose`

### Q5: 端口 18789 被占用

**解决：**

```bash
# 查看端口占用
netstat -ano | findstr :18789

# 使用其他端口启动
openclaw gateway run --port 18790

# 或者强制占用端口
openclaw gateway run --port 18789 --force
```

### Q6: 升级 OpenClaw

```bash
# 检查当前版本
openclaw --version

# 升级到最新版本
openclaw update

# 或手动更新
npm install -g openclaw@latest
```

### Q7: 完全卸载重装

```bash
# 停止服务
openclaw gateway stop

# 卸载
npm uninstall -g openclaw

# 清理配置（谨慎）
rm -rf ~/.openclaw

# 重新安装
npm install -g openclaw@latest
```

---

## 附录：配置文件路径参考

| 系统 | 配置文件路径 |
|------|-------------|
| Windows (WSL2/Linux) | `~/.openclaw/openclaw.json` |
| Windows (PowerShell) | `C:\Users\你的用户名\.openclaw\openclaw.json` |
| macOS | `~/.openclaw/openclaw.json` |
| Linux | `~/.openclaw/openclaw.json` |

---

## 相关资源链接

- OpenClaw 官网：https://openclaw.ai
- OpenClaw 文档：https://docs.openclaw.ai
- 飞书开放平台：https://open.feishu.cn
- DeepSeek API：https://platform.deepseek.com
- Node.js 下载：https://nodejs.org/

---

*本文档最后更新时间：2026年5月*
