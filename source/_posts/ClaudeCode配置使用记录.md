---
title: "Claude Code 配置使用记录"
date: 2026-02-14 18:20:00
tags:
  - claude-code
  - ai工具
  - 学习笔记
  - 开发效率
categories:
  - 开发工具
thumbnail: "/images/thumbnails/claude.png"
---

## 1. 环境准备

先确认基础工具可用。

```powershell
git --version
node -v
npm -v
python --version
```

如果某项命令不存在，先补齐环境，再继续后面的配置。

## 2. 安装 Claude Code CLI

> 这一节以 npm 全局安装为例。若你使用其他安装方式，请以官方文档为准。

```powershell
npm install -g @anthropic-ai/claude-code
```

安装后先确认命令是否可用（常见命令名为 `claude`）：

```powershell
claude --version
claude --help
```

如果 `claude` 无法识别，先检查 npm 全局安装目录是否在 PATH。

## 3. 登录与鉴权

按你的 CLI 版本执行登录（常见是网页授权或 API Key）。

```powershell
claude login
```

如果你的版本走环境变量方式，可在当前会话先设置：

```powershell
$env:ANTHROPIC_API_KEY="你的Key"
```

验证是否生效：

```powershell
claude whoami
```

## 4. 在项目中初始化使用

进入目标项目目录（一定要先切到正确目录）。

```powershell
cd D:\01_Software\Development_Tools\Hexo\blog
```

先让 CLI 识别项目上下文，再开始交互。

```powershell
claude
```

我自己固定使用这套提问模板，减少沟通偏差：

```text
任务目标：
当前状态：
涉及文件：
约束条件：
期望输出：
验收标准：
```

## 5. 首次验证（最小可用流程）

我会用一个最小任务验证链路，比如“新增/修改一篇文章并本地预览”。

### 5.1 让 Claude Code 只改指定范围

建议你在提示里明确：

- 只允许改 `source/_posts`；
- 不改主题和配置；
- 输出变更说明。

### 5.2 本地验证 Hexo 是否正常

```powershell
hexo clean
hexo generate
hexo server
```

如果是首次本地运行，还需要先安装依赖：

```powershell
npm install
```

## 6. 日常使用建议（流程化）

我现在固定按这个顺序：

1. 写清楚本轮唯一目标；
2. 限定文件范围；
3. 让它先给方案，再执行；
4. 本地验证结果；
5. 做一次 3 行复盘（有效点/问题点/下次改进）。

## 7. 常见问题与排查命令

### 7.1 命令不可用（`claude` 不是内部命令）

```powershell
npm config get prefix
npm list -g --depth=0
where claude
```

排查重点：CLI 是否安装成功、PATH 是否包含全局 bin 目录。

### 7.2 登录后仍提示未鉴权

```powershell
claude whoami
$env:ANTHROPIC_API_KEY
```

排查重点：当前终端会话是否有 Key、是否切换了新终端导致环境变量丢失。

### 7.3 改了文章但页面不更新

```powershell
hexo clean
hexo generate
hexo server
```

排查重点：缓存未清、front-matter 格式错误、文件路径不在 `source/_posts`。


