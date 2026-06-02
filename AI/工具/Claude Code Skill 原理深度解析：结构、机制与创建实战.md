---
title: Claude Code Skill 原理深度解析：结构、机制与创建实战
author: 萝卜啊
source: https://mp.weixin.qq.com/s/lo8sscfBGmRl-nu8yMhlKQ
date: 2025-05-14
tags:
  - Claude-Code
  - Skills
  - AI工具
  - Agent
  - Prompt
  - MCP
  - 渐进式披露
---

# Claude Code Skill 原理深度解析：结构、机制与创建实战

> 原文链接：[萝卜啊](https://mp.weixin.qq.com/s/lo8sscfBGmRl-nu8yMhlKQ)

---

## Part.01 Skill 到底是什么

Anthropic 官方工程师 Thariq Shihipar 说过："Skills 是一个常见的误解，大家以为它们只是 Markdown 文件。这个误解的代价是巨大的——把它理解成 Markdown，就像把瑞士军刀当成开瓶器在用，没出错，但你只用了一个功能。"

**Skill 是包含可组合程序性知识的文件集合，它是一个标准化、可移植、具备可组合性的文件夹。**

关键概念是**程序性知识（Procedure Knowledge）**——你让 AI"做个漂亮的网页"，它能做，但做出来一股模板味。因为它缺的不是审美能力，是"你先理解这个页面的语境，然后选一个 BOLD 的设计方向，不要选安全的字体"这种程序性知识。Skill 就是把这种知识封装成结构化的操作手册。

### Skill 的定位：Tool、Skill、Agent 三层

| 层级 | 比喻 | 职责 |
|------|------|------|
| **Tool** | 螺丝刀 | 能干一件事，但你得告诉它往哪拧 |
| **Skill** | 电工手册 | 告诉学徒面对配电箱先试电、再关开关，顺时针三圈半 |
| **Agent** | 学徒 | 会调度、会判断，但离开指令指导容易出错 |

> 没有手册，学徒拿着螺丝刀可能捅到带电的端子上。没有 Skill，Agent 用 Tool 就是盲调。

---

## Part.02 拆开一个 Skill 文件夹：四层结构

```
my-skill/
├── SKILL.md          # 必须：元数据 + 核心指令
├── scripts/          # 可选：可执行代码
├── references/       # 可选：参考文档
└── assets/           # 可选：素材模板
```

一个 Skill 具备三个特性：**可复用**（一次编写，反复使用）、**可组合**（多个 Skill 随意搭配）、**可移植**（同一套 Skill 在所有支持标准的平台上都能跑）。

### 第一层：SKILL.md — 指挥中心

SKILL.md 分为两块：**YAML 元数据 + Markdown 正文**。

**YAML 必填字段：**

| 字段 | 约束 | 说明 |
|------|------|------|
| `name` | 最多 64 字符，小写字母+数字+连字符，必须与父目录名一致 | 技能名称 |
| `description` | 最多 1024 字符，应包含触发关键词 | AI 用它判断是否激活 |

> ⚠️ **description 是最容易踩坑的字段。** Anthropic 的建议：把 description 当作激活这个 Skill 的 **if 语句**，而不是摘要。

对比示例：

```
# ❌ 差的 description
"Helps with PDFs."

# ✅ 好的 description
"Extract text and tables from PDFs, fill forms, merge documents.
Use when the user mentions PDFs, forms, scanning, or document extraction."
```

差的只写了它是什么，好的写了**什么时候触发它**。

**Markdown 正文建议结构：**
1. 这个技能做什么
2. 什么场景用（以及什么场景不用）
3. 需要什么输入
4. 分步骤的执行流程
5. 怎么判断"做完了"
6. 常见的失败模式和修复方法

> **最重要的建议：把 SKILL.md 控制在 500 行以内。** 超出 500 行的内容就是冗余的上下文，会拖慢 Agent 响应。详细内容拆到 `references/` 里。

### 第二层：scripts/ — Skill 的"动手能力"

放的是可执行代码（Python、Bash、JavaScript 都行）。

**关键设计：Skill 的脚本不直接触碰底层系统。**

```
Agent → 调度 Skill → 调用脚本 → 通过操作系统执行动作
```

每一步都可以被观测和测试。脚本只能做 Skill 声明要做的事。每个 Skill 可以在 YAML 里通过 `allowed-tools` 字段声明需要哪些工具权限。

> 实测对比：加载 100 个 Skill 的总 Token 消耗，仅为对接单个 MCP 服务的一半左右——因为脚本在需要时才执行，平时不占上下文。

### 第三层：references/ — 只在需要时才翻的补充材料

放额外的 Markdown 文档——规则、规范、API 签名、常见 FAQ、格式标准。

**SKILL.md 是指挥中心，references/ 是资料库。指挥中心只负责指路，不负责灌知识。**

> 如果把所有参考资料全部塞进 SKILL.md 正文，Agent 每次激活这个 Skill 都会被迫读一遍——哪怕这次任务根本用不上。这就是为什么很多人的 SKILL.md 越写越长，AI 反而越来越"笨"。

### 第四层：assets/ — 输出模板和静态资源

放模板文件、图片、样例数据这类静态资源。Agent 在需要产出特定格式的输出时，直接从 `assets/` 里拽模板。

> 四层结构的设计哲学就一句话：**不是让 AI 更聪明，是让你把经验从脑子里搬到文件系统里，AI 按你的标准干活。**

---

## Part.03 渐进式披露：Skill 设计的灵魂

这是整个 Skill 设计最天才的部分。四个阶段：

| 阶段 | 做什么 | Token 消耗 |
|------|--------|-----------|
| **广告** | 启动时只加载所有 Skill 的 name 和 description，注入系统提示词 | 约 100 token/技能 |
| **触发加载** | Agent 判断任务匹配某个 Skill 描述时，完整加载 SKILL.md 正文 | 推荐 ≤5000 token |
| **按需读资料** | 只在 Agent 主动调用时才加载 references、templates、assets | 按需 |
| **按需跑脚本** | Agent 调用脚本工具来执行 bundled scripts | 按需 |

### 与传统 MCP 方案的对比

```
传统 MCP：7 个 MCP 服务器 → 塞进上下文 98,700 token → 模型还没干活，1/5 预算已耗尽

Skills 渐进式披露：50 个 Skill 启动只占约 5000 token → 按需加载 → 上下文富裕
```

> 这不是 Workaround，是一步升维。从"给 Agent 所有工具说明书让它判断"变成了"让 Agent 先粗筛一遍再给完整指令"。
>
> 前者是目录 + 整个图书馆，后者是目录 + 你点的书从库里送过来。

---

## Part.04 Skills 和 MCP 的真实关系

Anthropic 的 Pedro Rodrigues 在 MCP Dev Summit 上给出了经典比喻：**MCP 是飞行员，Skills 是副驾驶。你不会因为副驾驶认识路，就把飞行员炒掉。**

| | MCP | Skills |
|---|-----|--------|
| **解决什么** | N 个 Agent 如何跟 M 个服务对话（连接问题） | 怎么把做一件事的正确上下文给到 Agent（饱和问题） |
| **定位** | 能力协议 | 操作手册 |
| **关系** | 互补，各司其职 | 互补，各司其职 |

### 现场演示案例

Pedro 让 Claude 通过 Supabase MCP 服务器创建团队任务摘要视图，跑了四个版本：

- **MCP-only**：Agent 拿到了 Supabase 的完整工具箱，但忽略了 `search_docs`，用训练数据里陈旧的 Postgres知识写了一个视图，绕过了 Row-Level Security——团队 A 看到了团队 B 的任务
- **Skill + MCP**：Agent 多做了一个动作——读了文档，知道了要用 `security_involver`，视图被放在安全护栏之内

> MCP 让 Agent 有能力建视图，Skills 让 Agent 有上下文**建对**视图。

---

## Part.05 手把手创建自己的 Skill

### 路径一：从零手写

**第一步**，建文件夹。在项目根目录或 `~/.claude/skills/` 下创建文件夹，name 必须是小写字母、数字、连字符组合。

**第二步**，写 SKILL.md：

```yaml
---
name: weekly-report-generator
description: Generate weekly progress reports from Git commit logs.
  Use when the user asks to generate a weekly report, create a
  progress summary, or compile work done this week. This skill
  synthesizes commit history into structured report formats.
license: Apache-2.0
---
```

**第三步**（可选），添加 `references/`、`scripts/`、`assets/`。

**关键原则：**
- description 要写**触发条件**，不要写摘要
- 正文要写**步骤和坑**，不要写抽象建议
- 细节往外拆，正文保持在 500 行以内

### 路径二：用 skill-creator

Anthropic 官方工具，支持四种模式：

| 模式 | 功能 |
|------|------|
| **Create** | 描述想要的能力，自动生成 Skill 初版结构和内容 |
| **Eval** | 写测试 prompt、定义"好的输出"，跑结构化评估，给 pass/fail 结果 |
| **Improve** | 根据评估结果自动给出优化建议 |
| **Benchmark** | 内置 comparator agent，做盲态 A/B 对比（装前 vs 装后） |

> skill-creator 的关键提醒：你写的 Skill 是给另一个 Agent 实例用的。**重点应该放在"对 Agent 有用但不是显而易见的信息"上。** 不要写 AI 已经知道的东西——它不需要你告诉它 React 是什么，它需要你告诉它"我们这个项目里 React 组件用哪种状态管理方案"。

---

## Part.06 设计 Skill 的三条铁律

### 铁律一：一个 Skill 只做一件事

职责越单一，触发越精准，可组合性越强。Anthropic 设计指南核心原则：*"Design Skills as composable building blocks that do one thing well."*

不要做一个"万能的办公助手 Skill"，拆成 PDF 处理、周报生成、邮件草拟三个。未来可以任意组合。

### 铁律二：description 决定一个 Skill 会不会被用

description 写糟了，就算正文藏着全部心血，Agent 也不会触发。把它当 **if 语句**写，写出用户会用什么词、在什么场景下提需求，然后倒推 description。

### 铁律三：把所有"常见坑"写进 Gotcha 部分

Anthropic 的建议："在任何 Skill 中，最具价值的部分是 Gotcha 部分——也就是 Agent 可能踩的坑。这些部分应基于 Agent 在使用你的 Skill 时遇到的常见失败点构建。"

> Agent 不是人，它不会自己领悟"哦这里可能会出错"，除非你明确告诉它。

---

> Skill 是你把经验从脑子里搬出来、放到文件系统里的方式。它不是让 AI 更聪明，是让你不再需要重复教同一个东西。你做 Skills 的能力，就是未来十年你的 AI 协作能力。

> 相关笔记：[[Claude Code Skills 完全指南]] · [[Skill 创建指南：把经验变成 AI 指令]] · [[Claude Code 11 个常用 Skill 推荐（含各职业方案）]]
