---
title: Skill 创建指南：把经验变成 AI 指令
author: 幽幽星光
source: https://mp.weixin.qq.com/s/sQkP2_ljdgqmiL9JMlHZ7w
date: 2025-05-14
tags:
  - Claude-Code
  - Skills
  - AI工具
  - Agent
  - 教程
---

# Skill 创建指南：把经验变成 AI 指令

> 原文链接：[幽幽星光](https://mp.weixin.qq.com/s/sQkP2_ljdgqmiL9JMlHZ7w)
>
> 上一篇：[[Claude Code Skills 完全指南]]

---

## 一、开篇

Skill 的本质——就是一个 `SKILL.md` 文件，加上一些辅助脚本和参考资料。但「简单」不意味着「随便写写」。

一句话理解：**写 Skill 就像写一份「新员工入职手册」**——先定好标题和简介（元数据），再写清楚做事步骤（指令），最后附上参考资料（资源文件）。

---

## 二、SKILL.md 的结构

SKILL.md 由两部分组成：**元数据** 和 **指令**。

```markdown
---
name: code-reviewer
description: 按照团队规范审查代码质量，检查安全漏洞、命名规范和测试覆盖。当用户提到代码审查、PR review 时触发。
---

# Code Reviewer Skill

## 审查标准

### 高优先级（必须修复）
- 安全漏洞：SQL 注入、XSS、硬编码密钥
- 逻辑错误：空指针、数组越界、并发问题

### 中优先级（建议修复）
- 性能问题：N+1 查询、不必要的循环

### 低优先级（可选优化）
- 命名规范、代码风格

## 输出格式
按优先级分组输出，每条包含：文件路径和行号、问题描述、修复建议
```

### 为什么要分开？

> 想象一个图书馆：**元数据 = 书脊**（书名和简介），**指令 = 书的内容**。AI 不需要把所有书都读一遍，只看书脊就知道哪本可能有用，找到后才翻开阅读。

- **元数据**：对话开始时就加载，AI 知道有哪些 Skills 可用
- **指令内容**：Skill 被触发时才读取（按需加载）

> 安装 100 个 Skills 也不会撑爆上下文——AI 只看「书脊」，不翻「书页」。

---

## 三、元数据字段详解

### 必填字段

| 字段 | 约束 | 说明 |
|------|------|------|
| `name` | 最多 64 字符，小写字母+数字+连字符，不能含 "anthropic"/"claude" | Skill 的唯一标识 |
| `description` | 最多 1024 字符，不能为空 | 功能描述，**AI 用它判断是否触发** |

### description 黄金法则

- ✅ 说清楚**做什么**："检查代码质量"
- ✅ 说清楚**什么时候触发**："当用户提到代码审查、PR review 时触发"
- ❌ 不要超过 1024 字符

```yaml
# ✅ 好的 description
description: 按照团队规范审查代码质量，检查安全漏洞、命名规范和测试覆盖。当用户提到代码审查、PR review、merge request 时触发。

# ❌ 差的 description
description: 一个代码审查工具
```

### 通用可选字段

| 字段 | 类型 | 说明 |
|------|------|------|
| `version` | 字符串 | 版本号 |
| `author` | 字符串 | 作者 |
| `tags` | 数组 | 标签，便于搜索 |
| `category` | 字符串 | 分类 |

### Claude Code 专属字段

| 字段 | 类型 | 说明 |
|------|------|------|
| `disable-model-invocation` | 布尔值 | `true` 后只能手动触发，不会自动触发 |
| `allowed-tools` | 数组 | 限制可用工具 |
| `context` | 字符串 | 额外上下文 |

> ⚠️ 高风险操作（部署、删除）一定要设 `disable-model-invocation: true`，避免 AI 自作主张。

---

## 四、命名规则和目录结构

### 命名规则

| 规则 | ✅ 正确 | ❌ 错误 |
|------|---------|---------|
| 文件夹名全小写 | `code-reviewer` | `Code-Reviewer` |
| name 用连字符 | `my-skill` | `my_skill`、`mySkill` |
| 无空格 | `pdf-processor` | `pdf processor` |
| 核心文件大写 | `SKILL.md` | `skill.md` |

### 完整目录结构

```
my-skill/
├── SKILL.md              # 核心指令（必需）
├── scripts/              # 可执行脚本
│   ├── test.sh
│   └── build.sh
├── references/           # 参考文档（按需加载）
│   ├── code-style.md
│   └── api-spec.md
├── assets/               # 资源文件
│   ├── template.py
│   └── config.json
├── examples/             # 示例代码
│   ├── good-example.py
│   └── bad-example.py
└── tests/                # 测试文件
    └── evals.json
```

> 💡 **脚本**不会加载内容，只会执行（10 万行脚本也不消耗 token）。**参考文档**被链接后会自动加载到上下文，注意控制大小。

### 存放位置

| 位置 | 路径 | 适用场景 |
|------|------|---------|
| 全局 | `~/.claude/skills/` | 通用 Skills，所有项目可用 |
| 项目 | `.claude/skills/` | 项目专用，提交到 Git |

> 项目级 Skills 优先级更高，会覆盖同名全局 Skills。

---

## 五、用 skill-creator 创建 Skill

### 安装

```bash
git clone https://github.com/anthropics/skills.git
cp -r skills/skills/skill-creator ~/.claude/skills/
```

### 使用

在 Claude Code 中输入 `/skill-creator`，AI 会像产品经理一样用问卷梳理需求：

```
你：我想做一个 Skill，把会议录音的文字稿整理成结构化的会议纪要。

Claude：好的，我来问几个问题帮你确认需求：
        1. 纪要里需要包含哪些内容？
        2. 输出格式是 Word、Markdown，还是直接回复？
        3. 有没有固定的纪要模板？

你：需要包含会议主题、时间、参与人、讨论要点、决议事项、
    下一步行动（含负责人和截止日期）。输出 Word 文档。

Claude：明白了。再确认几个细节：
        4. 如果文字稿中有多次会议，是分别生成还是一起处理？
        5. 讨论要点需要保留原文发言吗，还是只提炼摘要？
        6. 负责人字段如果原文没提到，是留空还是标记「待定」？
```

> 💡 这个阶段**不要跳过**，后面踩的坑大多源于这里没说清楚。

---

## 六、实战案例：代码提交规范检查

### 需求

检查提交信息是否符合 Conventional Commits 规范。

### 生成的 SKILL.md

```yaml
---
name: commit-checker
description: 检查代码提交信息是否符合 Conventional Commits 规范。当用户提到提交信息、commit message、git commit 时触发。
---
```

```markdown
# Commit Checker Skill

## 检查规则

### 允许的 type
feat | fix | docs | style | refactor | perf | test | chore | build | ci | revert

### 约束条件
- description 不能超过 72 个字符
- description 必须小写开头
- 不能以句号结尾

## 输出格式
✅ 符合规范 / ❌ 不符合规范 + 问题描述 + 修改建议
```

---

## 七、常见错误 Top 5

| 问题 | 原因 | 解决方案 |
|------|------|---------|
| Skill 不触发 | description 太笼统 | 写清「做什么」+「什么时候触发」 |
| 加载但找不到 | 目录结构放错位置 | 确保 SKILL.md 在正确位置 |
| 高风险操作被自动执行 | 未设 disable-model-invocation | 部署/删除等设为 true |
| 文件名报错 | SKILL.md 大小写写错 | 必须全大写 `SKILL.md` |
| 上下文被撑爆 | 指令内容太多 | 拆分到 references/ 目录按需加载 |

---

## 八、创建检查清单

- [ ] `name` 符合命名规则（小写、连字符、无空格）
- [ ] `description` 写清「做什么」和「什么时候触发」
- [ ] 高风险操作设置 `disable-model-invocation: true`
- [ ] 指令结构清晰（步骤、示例、输出格式）
- [ ] 参考文档放在 `references/` 目录
- [ ] 测试用例覆盖正常和异常场景

---

## 相关资源

- skill-creator：[https://github.com/anthropics/skills](https://github.com/anthropics/skills)
- Skills 生态排行榜：[https://skills.sh/](https://skills.sh/)
- 官方规范：[Anthropic - Agent Skills](https://www.anthropic.com/engineering/equipping-agents-for-the-real-world-with-agent-skills)
