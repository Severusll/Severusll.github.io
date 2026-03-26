---
title: "OpenClaw 部署与 QQ 接入实战记录（Windows）"
date: 2026-03-16 20:10:00
tags:
  - openclaw
  - ai工具
  - 部署
  - qq机器人
categories:
  - 开发工具
thumbnail: "/images/thumbnails/openclaw.png"
---

---

## 一、整体流程先看一眼

我把安装过程拆成 4 步：

1. 安装运行环境（Node.js、Git）
2. 安装 OpenClaw
3. 运行引导并完成基础配置
4. 接入 QQ 手机对话

---

## 二、步骤 1：安装运行环境

OpenClaw 依赖 Node.js 和 Git，建议先确认版本是否可用。

### 1) 检查是否已安装

```powershell
node -v
git --version
```

- 如果两个命令都能输出版本号，可以直接进入下一步。
- 如果命令不存在，先安装再继续。

### 2) 可选：检查 npm / pnpm

```powershell
npm -v
pnpm -v
```

`pnpm` 不是每台机器默认都有。如果后面命令提示 `pnpm` 不存在，可以先安装：

```powershell
npm install -g pnpm
```

---

## 三、步骤 2：安装 OpenClaw

### 1) 以管理员身份打开 PowerShell

Windows 搜索 `PowerShell` -> 右键 -> **以管理员身份运行**。

### 2) 执行一键安装命令

```powershell
iwr -useb https://openclaw.ai/install.ps1 | iex
```

安装完成后，按官方指引批准构建脚本：

```powershell
pnpm approve-builds -g
```

### 3) 验证安装

```powershell
openclaw -v
```

如果能看到版本号，说明 OpenClaw 本体安装成功。

---

## 四、步骤 3：运行引导并完成配置

安装完本体后，进入 OpenClaw 新手引导：

```powershell
openclaw onboard --install-daemon
```

- `onboard`：进入交互式引导
- `--install-daemon`：安装后台服务（关掉终端也能持续运行）

> 安全提醒：OpenClaw 是能操作你电脑的 AI 工具，具备执行终端命令的能力。建议在虚拟机或备用机先体验，避免误操作影响重要数据。

### 1) 选择安装模式

引导会让你选：

- `Quickstart`（快速开始）
- `Manual`（手动配置）

我建议新手先选 `Quickstart`，先跑通，再慢慢细调。

### 2) 配置大模型（关键步骤）

向导会列出模型平台（例如 Anthropic / OpenAI / Qwen 等）。

我的建议：

- 想快速体验：选支持 OAuth 登录的方案（流程更省事）
- 想长期稳定：用你已有的 API 平台账号和配额方案

### 3) 配置聊天渠道

向导可能会问你是否立即接 Telegram / Discord / 飞书等。

为了先跑通主流程，我建议：

- 先跳过第三方聊天渠道
- 先用网页/本地方式验证 OpenClaw 可用
- 再单独接 QQ（更清晰）

### 4) 安装 Skills 技能包

Skills 可以理解成 OpenClaw 的能力扩展模块。

建议至少安装官方市场能力（例如 ClawHub），后续按需再加：

- 网页搜索
- 浏览器操作
- 文档/PPT 处理
- 自动化任务类技能

### 5) 启动网关服务

向导后续会安装并启动 Gateway。它负责：

- 接收你来自网页/QQ 等渠道的消息
- 调度 AI 执行
- 回传结果

这一段通常引导会自动处理，注意看终端提示是否有报错。

---

## 五、步骤 4：接入 QQ 手机对话

现在可进入最实用的一步：手机 QQ 直接和 OpenClaw 对话。

### 1) 打开 QQ 机器人 OpenClaw 主页

```text
https://q.qq.com/qqbot/openclaw/index.html
```

使用 QQ 扫码登录。

### 2) 执行页面给出的 3 条配置命令

页面会给你三条专属命令（包含你的密钥信息）。

按顺序复制到 PowerShell 执行即可。

> 重要：这些命令里有敏感信息，别发到群里，别截图外传。

### 3) 手机上验证

在手机 QQ 给机器人发一条简单消息，例如：

- "你好，先做个自我介绍"
- "帮我列出 Windows 查看系统配置的命令"
- "帮我写一段 Markdown 周报模板"

如果能正常回复，说明 QQ 通道已经接通。

---

## 六、我自己遇到的几个常见问题

### 问题 1：`openclaw` 命令不可用

先检查：

```powershell
where openclaw
```

如果查不到，通常是安装未完成或 PATH 未生效。可尝试：

- 重新开一个 PowerShell
- 重新执行安装脚本
- 再次验证 `openclaw -v`

### 问题 2：`pnpm` 命令不存在

```powershell
npm install -g pnpm
pnpm -v
```

### 问题 3：引导卡住或服务异常

先做最小重试：

```powershell
openclaw onboard --install-daemon
```

如果仍有问题，重点看终端报错关键词（鉴权失败、网络超时、权限不足等）。

### 问题 4：QQ 接入后没反应

优先检查：

- 三条配置命令是否完整执行
- 是否复制了最新页面生成的命令
- 本机网关服务是否还在运行

---
