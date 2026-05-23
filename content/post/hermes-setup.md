---
title: "我用上了 Hermes —— 开源 AI 助手安装全记录"
description: "记录在本地部署 Hermes Agent 的过程与初体验"
keywords: "hermes,hermes-agent,ai助手,nous-research,deepseek"

date: 2026-05-23T16:00:00+08:00
lastmod: 2026-05-23T16:00:00+08:00

math: false
mermaid: false

categories:
  - 技术
tags:
  - AI
  - Hermes
  - 开源
---

最近用上了 [Hermes Agent](https://github.com/NousResearch/hermes-agent)，一个由 Nous Research 开发的开源 AI 助手框架。它可以接入多种 LLM 提供商，支持终端、浏览器、文件操作、GitHub 集成等工具，还能通过 WeChat 等平台直接对话，体验相当流畅。

<!--more-->

## 什么是 Hermes

Hermes 是一个本地运行的 AI Agent，你可以把它看作一个能力远超普通聊天机器人的"终端里的 AI 同事"。它能执行命令、读写文件、搜索网页、管理 GitHub 仓库，甚至可以定时运行任务。

## 安装步骤

以下是部署过程的记录，供参考：

### 1. 克隆仓库

```bash
git clone https://github.com/NousResearch/hermes-agent.git
cd hermes-agent
```

### 2. 运行安装脚本

```bash
bash scripts/install.sh
```

脚本会自动检测你的系统类型并完成依赖安装，包括 Python 环境和必要的系统包。

### 3. 配置模型提供商

安装完成后，编辑 `~/.hermes/config.yaml`，配置你使用的 LLM 提供商。我使用的是 DeepSeek：

```yaml
model_provider: deepseek
model: deepseek-v4-pro
```

Hermes 支持多种提供商：OpenAI、Anthropic、DeepSeek、OpenRouter 等，按需选择即可。

### 4. 设置 API Key

将 API Key 写入 `~/.hermes/.env`：

```
DEEPSEEK_API_KEY=你的密钥
```

### 5. 启动

```bash
hermes
```

首次启动会进入 TUI 界面，你就可以和 Hermes 对话了。

## 连接 WeChat

Hermes 的一大亮点是支持多平台消息接入。我用 `hermes setup` 配置了 WeChat 通道，绑定后就能在微信里直接和 Hermes 聊天，随时随地让它帮忙处理任务。

## 初体验

上手之后发现它的几个实用场景：

- **写博客**：就像这篇文章，直接告诉它"帮我在 Hugo 博客里创建一篇新文章并推送"，它就能自己找到项目目录、理解主题结构、写好 markdown、甚至帮你 commit 和 push
- **执行任务**：代码搜索、文件操作、运行脚本都不在话下
- **定时任务**：可以设置 cron job 定时执行检查或发送通知

总体来说，Hermes 的本地化部署 + 多平台接入的设计让 AI 助手真正融入了日常工作流。后续计划探索更多自动化场景。
