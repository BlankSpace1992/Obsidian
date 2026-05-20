---
title: Claude Code 接入 DeepSeek V4 实战教程
author: 苏三说技术
source: https://mp.weixin.qq.com/s/GtdfAbn8_Ef1jWnynfJnmg
date: 2025-05-14
tags:
  - Claude-Code
  - DeepSeek
  - AI工具
  - 教程
  - 性价比
---

# Claude Code 接入 DeepSeek V4 实战教程

> 原文链接：[苏三说技术](https://mp.weixin.qq.com/s/GtdfAbn8_Ef1jWnynfJnmg)

---

## 背景

DeepSeek 发了 V4，官网直接打 2.5 折。DeepSeek V4 性能对标 Claude Opus 4.6，兼容 Anthropic API 格式，价格直接打骨折——百万 token 的输入价格不到 Claude 的十分之一。

本文是把 DeepSeek 接入 Claude Code 的完整过程 + 实际使用体验。

---

## 一、获取 DeepSeek API Key

1. 去 DeepSeek 官网找到 **API 开放平台**
2. 注册账号，在后台创建 API Key
3. 复制保存好（只显示一次）

> 💡 价格方面：DeepSeek V4 目前官网直接八折，百万 token 输入价格不到 Claude 的十分之一

---

## 二、安装 Claude Code

**macOS / Linux / WSL：**

```bash
curl -fsSL https://claude.ai/install.sh | bash
```

**Windows PowerShell：**

```powershell
irm https://claude.ai/install.ps1 | iex
```

**验证安装：**

```bash
claude --version
```

---

## 三、通过 CC Switch 配置 DeepSeek

Claude Code 本身只支持 Anthropic API，切到 DeepSeek 需要改供应商配置。

### 推荐方式：CC Switch

CC Switch 是一个统一管理 Claude Code、Codex、Gemini CLI 供应商配置的桌面工具，不用记环境变量，切换模型点两下就行。

**GitHub 地址：** [https://github.com/farion1231/cc-switch](https://github.com/farion1231/cc-switch)

**配置步骤：**

1. 下载安装 CC Switch（macOS 选 dmg，Windows 选 msi）
2. 点 "+" 新建配置
3. Claude 供应商选 **DeepSeek**
4. 粘贴 API Key
5. 添加模型：
   - 主模型：`deepseek-v4-pro[1m]`
   - 子代理/轻量任务：`deepseek-v4-flash`
6. 保存后点启动

### 手动方式：环境变量

```bash
export ANTHROPIC_BASE_URL=https://api.deepseek.com/anthropic
export ANTHROPIC_AUTH_TOKEN=<你的 DeepSeek API Key>
export ANTHROPIC_MODEL=deepseek-v4-pro[1m]
export ANTHROPIC_DEFAULT_OPUS_MODEL=deepseek-v4-pro[1m]
export ANTHROPIC_DEFAULT_SONNET_MODEL=deepseek-v4-pro[1m]
export ANTHROPIC_DEFAULT_HAIKU_MODEL=deepseek-v4-flash
export CLAUDE_CODE_SUBAGENT_MODEL=deepseek-v4-flash
export CLAUDE_CODE_EFFORT_LEVEL=max
```

> Windows 用户把 `export` 换成 `$env:` 即可

---

## 四、验证接入

```bash
claude
# 启动后问："你是什么模型"
# 或用 claude status 查看当前模型状态
```

> ⚠️ 记得开 **Max 模式**，才能体验 DeepSeek V4 的完整能力

---

## 五、实测体验

### 测试一：从零生成 Flappy Bird 小游戏

- 自动调用 `superpowers brainstorming skill`，先确认需求再写代码
- Skills 体系完全兼容
- 功能完整：小鸟能飞，管道能碰，分数正常计

### 测试二：待办事项管理工具

- 增删改查 + 数据持久化
- 同样自动调用 Skills，一次跑通

### 费用

两个应用跑完，消耗 **1.46 元**。

### 综合感受

| 方面 | 评价 |
|------|------|
| 代码逻辑 | 跟原生 Claude Opus 4.6 基本同一水平 |
| 工具兼容 | Skills、Agent 子代理、文件编辑完全兼容 |
| 前端 UI | 精致度比原生 Opus 4.6 差一点 |
| 后端/脚本 | 差距完全不影响 |

---

## 结论

- **已有 Claude Code 用户**：切过去完全可行，Max 模式下编码体验和 Opus 4.6 基本持平
- **前端 UI 极致要求**：主逻辑用 DeepSeek，最后精调用 Claude 收尾
- **纯新手**：先把 Claude Code 用熟，切模型不急

> 说白了，哪个性价比高用哪个。
