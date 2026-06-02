---
title: Claude Code Skills 完全指南：什么是 Skills 以及怎么用
author: 数字生命卡兹克
source: https://mp.weixin.qq.com/s/nRVVqPaGxWdNqNrUcurSXg
date: 2025-05-14
tags:
  - Claude-Code
  - Skills
  - AI工具
  - Agent
  - Prompt
  - MCP
---

# Claude Code Skills 完全指南：什么是 Skills 以及怎么用

> 原文链接：[数字生命卡兹克](https://mp.weixin.qq.com/s/nRVVqPaGxWdNqNrUcurSXg)

---

## 一、Skills 是什么？

**Skills = 给 Agent 用的技能。**

2025 年 10 月，Anthropic 在 Claude Code 上支持了 Skills 特性。12 月 18 日将其作为标准开放，随后各大编程工具纷纷接入。

目前支持 Skills 的工具：Claude Code、OpenCode、Codex、Cursor、Codebuddy 等。

### Skills vs Prompt vs MCP

用一个比喻来理解：

| 概念 | 类比 | 特点 |
|------|------|------|
| **Prompt** | 当场口头交代任务 | 临时、反应式、只在当前对话生效 |
| **Skills** | 公司内部的 SOP 手册 | 可复用、渐进式加载、持久能力 |
| **MCP** | 门禁卡 | 让 AI 安全连接外部系统、调用外部能力 |

> Skills 在形式上是一个**文件夹**，不只是一个文本，里面可以包含 Prompt、参考文档、脚本等资源。

---

## 二、为什么 Skills 有价值？

### 核心设计思想：渐进式披露（Progressive Disclosure）

不是一上来给 Agent 大量信息让其"认知负荷爆炸"，而是：

1. **先加载元信息**：让模型知道"有这么个手册，适用范围是啥"
2. **需要时读取完整 SKILL.md**：判断任务确实需要时再加载
3. **按需读取附带文件**：还不够就继续读文件夹里的其他文件

> 这样不仅保证 Agent 准确执行任务，还能在长轮对话中**节省大量 Token**。

### 实际案例

**案例一：AI 选题系统**

- 1 个 Agent + 3 个 Skill
- 每天只需说"开始今日选题生成"
- 自动完成：热点采集 → 选题生成 → 审核 → 不通过则迭代修改

**案例二：整合包生成器**

- 给一个 GitHub 链接
- 自动打包成本地整合包，一键启动
- 带前端界面，小白开箱即用

---

## 三、Skill 文件结构

```
my-skill/              # 文件夹名：小写字母 + 连字符
├── SKILL.md           # 唯一必需文件
├── scripts/           # 可选：脚本
├── templates/         # 可选：模板
└── references/        # 可选：参考资料
```

### SKILL.md 结构

```markdown
---
name: 你的 skill 名称
description: 简要描述该技能的功能以及何时该使用它
---

# 你的技能名称

## 指令 (Instructions)
为 Agent 提供清晰、逐步的操作指南。

## 示例 (Examples)
展示使用该技能的具体代码或操作案例。
```

### description 字段要点

- ✅ **使用第三人称**："处理 Excel 文件并生成报告"
- ❌ 不要用第一人称："我可以帮助你处理 Excel 文件"
- ❌ 不要用第二人称："你可以使用这个来处理 Excel 文件"
- 尽量包含触发关键词
- SKILL.md 正文保持在 **500 行以内**效果最佳

---

## 四、安装 Skills

### 方法一：命令安装

在 Claude Code 或 OpenCode 中直接发送：

```
安装这个 skill，skill 项目地址为: https://github.com/anthropics/skills/tree/main/skills/skill-creator
```

### 方法二：手动拖入目录

| 工具 | 全局目录 |
|------|---------|
| Claude Code | `~/.claude/skills` |
| OpenCode | `~/.config/opencode/skill` |

> ⚠️ 初始没有 `skill` 文件夹，需手动创建。建议所有 skill 放全局目录，任何项目都能共享。

装完后：
- **OpenCode**：退出重进
- **Claude Code**：2.1.0 版本后支持热重载，无需重启

---

## 五、Anthropic 官方 Skills 仓库

地址：[https://github.com/anthropics/skills](https://github.com/anthropics/skills)

推荐安装的 Skills：

| Skill | 用途 |
|-------|------|
| **skill-creator** | 生成新的 Skill |
| **docx** | 处理 Word 文档 |
| **frontend-design** | 前端设计 |
| **pdf** | 处理 PDF 文件 |
| **xlsx** | 处理 Excel 文件 |

---

## 六、运行 Skills

直接通过对话，Agent 会根据需求自动调用对应的 Skill。

> 说实话，Skills 这波热度真不是圈内人又在发明新词。它做的就一件事：**把你的流程性知识变成可复用的能力包，在 Agent 需要时随叫随到，稳定发挥。**
