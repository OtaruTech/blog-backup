---
title: OpenClaw 安装部署指南
date: 2026-02-24 14:50:00
tags:
  - OpenClaw
  - 部署
  - AI Assistant
categories:
  - 工具部署
cover: /img/openclaw-deploy.jpg
---

# OpenClaw 安装部署指南

OpenClaw 是一个多通道 AI 助手网关，支持 WhatsApp、Telegram、Discord、飞书等多种消息平台。本文详细介绍如何在 macOS/Linux 系统上安装和部署 OpenClaw。

<!--more-->

## 环境要求

- **Node.js**: 22.x 或更高版本
- **操作系统**: macOS / Linux / Windows
- **网络**: 能够访问外部 API（用于连接 AI 服务）

## 安装步骤

### 1. 安装 OpenClaw

<Tabs>
  <Tab title="macOS / Linux">
    在终端执行以下命令：

    ```bash
    curl -fsSL https://openclaw.ai/install.sh | bash
    ```

    或者使用 npm 全局安装：

    ```bash
    npm install -g openclaw
    ```
  </Tab>
  <Tab title="Windows (PowerShell)">
    在 PowerShell 中执行以下命令：

    ```powershell
    iwr -useb https://openclaw.ai/install.ps1 | iex
    ```

    或者使用 npm 全局安装：

    ```powershell
    npm install -g openclaw
    ```
  </Tab>
</Tabs>

### 2. 运行初始化向导

```bash
openclaw onboard --install-daemon
```

初始化向导会引导你完成：
- 配置 AI 服务（如 Anthropic、Moonshot、Minimax 等）
- 设置网关参数
- 配置消息通道（飞书、Telegram 等）
- 安装系统服务（可选）

### 3. 检查网关状态

```bash
openclaw gateway status
```

如果一切正常，会显示网关正在运行。

### 4. 打开控制面板

```bash
openclaw dashboard
```

或者直接在浏览器访问：http://127.0.0.1:18789/

## 配置飞书通道

### 1. 创建飞书应用

1. 访问 [飞书开放平台](https://open.feishu.cn/)
2. 创建企业自建应用
3. 获取 `App ID` 和 `App Secret`

### 2. 配置权限

需要以下权限：
- `im:chat:readonly` - 读取群信息
- `im:message:readonly` - 读取消息
- `im:message:send_as_bot` - 发送消息

### 3. 添加到配置

编辑 `~/.openclaw/openclaw.json`：

```json
{
  "channels": {
    "feishu": {
      "enabled": true,
      "appId": "你的App ID",
      "appSecret": "你的App Secret"
    }
  }
}
```

### 4. 重启网关

```bash
openclaw gateway restart
```

## 配置 AI 模型

OpenClaw 支持多种 AI 提供商。在配置文件中添加：

```json
{
  "auth": {
    "profiles": {
      "anthropic": {
        "provider": "anthropic",
        "apiKey": "sk-ant-..."
      },
      "moonshot": {
        "provider": "moonshot",
        "apiKey": "你的Moonshot API Key"
      }
    }
  }
}
```

## 常用命令

<Tabs>
  <Tab title="macOS / Linux">
    | 命令 | 说明 |
    |------|------|
    | `openclaw gateway start` | 启动网关 |
    | `openclaw gateway stop` | 停止网关 |
    | `openclaw gateway restart` | 重启网关 |
    | `openclaw status` | 查看状态 |
    | `openclaw dashboard` | 打开控制面板 |
    | `openclaw message send` | 发送消息 |
  </Tab>
  <Tab title="Windows">
    | 命令 | 说明 |
    |------|------|
    | `openclaw gateway start` | 启动网关 |
    | `openclaw gateway stop` | 停止网关 |
    | `openclaw gateway restart` | 重启网关 |
    | `openclaw status` | 查看状态 |
    | `openclaw dashboard` | 打开控制面板 |
    | `openclaw message send` | 发送消息 |
  </Tab>
</Tabs>

## 常见问题

### Q: 网关无法启动？

A: 检查端口是否被占用：`lsof -i :18789`

### Q: 无法发送消息？

A: 确认通道配置正确，权限已开通

### Q: 如何查看日志？

A: `tail -f ~/.openclaw/logs/openclaw.log`

## 总结

OpenClaw 部署简单，功能强大，是构建个人 AI 助手的优秀选择。通过本文档，你可以快速搭建自己的 AI 助手，并通过飞书等消息平台随时随地与之交流。

---

*参考资料：OpenClaw 官方文档*
