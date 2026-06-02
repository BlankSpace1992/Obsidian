---
title: Claude Code 11 个常用 Skill 推荐（含各职业方案）
author: 萝卜啊
source: https://mp.weixin.qq.com/s/1UwdUwLYhAOPZ2jyNUjfJA
date: 2025-05-14
tags:
  - Claude-Code
  - Skills
  - AI工具
  - 效率
  - 职业方案
---

# Claude Code 11 个常用 Skill 推荐（含各职业方案）

> 原文链接：[萝卜啊](https://mp.weixin.qq.com/s/1UwdUwLYhAOPZ2jyNUjfJA)

---

我第一次用 Claude Code 的时候，以为这东西已经很聪明了——不就是个自带终端权限的 AI 编程助手吗？我只要把需求说清楚，它就能帮我干活。

干了两周，我发现不对。有三个问题反复出现，怎么调模型都解决不了。

**第一个问题：Token 烧得跟不要钱似的。**

我让它帮我写一份技术文档，要求排版整齐、层级分明、带页码和目录。它写完我一看，格式全对，但每个标题都是手动敲的空格对齐，文档结构用的是"一、""二、"而不是自动编号，目录那一页的数字跟正文对不上。

我问它为什么不直接用 Word 的自动目录功能？

它说：抱歉，我没有读取文档格式的能力，所有的格式判断都基于纯文本。

这一来一回重改了三轮，全是折腾一些没必要消耗 Token 的废话，最后那份八千字的文档烧掉我五十万 Token——三分之一是格式调整。

**第二个问题：遇到复杂任务就跳步。**

我让它拆解一个老项目的核心模块重构成可复用服务。它看了几眼仓库，直接开始写代码——没问我要留什么接口、没确认过数据库兼容、没见过一次测试。我赶紧叫停，让它先出个方案。方案倒是出了，但三行字就敷衍过去。最后我自己把执行过程拆成七个步骤，它才一步步老实做完。

**第三个问题：在专业领域像个实习生。**

我让它帮我设计一个 SaaS 仪表盘。它写的代码逻辑全对，功能也能跑。但界面惨不忍睹——默认的 Inter 字体、蓝白灰的卡片、列表间距全一样，看起来像 GitHub 上随手一翻就能找到的"AI 风格"产物。我不知道有多少人看过这类作品，如果看多了你甚至能一眼从审美上认出哪段代码是 AI 写的。

这三个问题其实都指向同一个根源：**Claude Code 裸用的时候是一个全能但粗糙的通用工具，它缺乏领域知识、缺乏工作流的纪律性、缺乏对专业质量的判断力。**

就像你雇了一个智商极高但完全没在你们行业干过的人，他什么都能聊，一上手就不对。

而那些真正的重度用户、那些能在生产环境里稳定出活的团队，其实早就在用一套东西把那个"粗糙"给治了。

它就是 **Skills**。

这篇文章就是写给还没有接触 Skills、或者刚接触但不知道从哪下手的读者看的。我会把这几个问题掰开讲清楚：Skills 到底是什么、怎么装、装多少、怎么查怎么更新、装完出问题怎么办、以及不同职业的人该怎么给自己定制一套 Skills 组合。

这是一篇基础教程，但也是一份实测清单——里面列出来的每一个 Skill，都是我自己或我身边的高频开发者真正在用的，没有任何凑数的。

---

## Skills 不是高级 Prompt，它是一个内置了工作流、参考文件和可执行脚本的文件夹

先讲清楚定义，因为误解这个定义的人比不会用 Skills 的人还多。

Anthropic 的官方工程师说过一句很直白的话：**Skills 是一个常见的误解，大家以为它们只是 Markdown 文件。**

这个误解的代价是巨大的——把它理解成 Markdown，就像把瑞士军刀当成开瓶器在用，没出错，但你只用了一个功能。

真正的 Skill 是一个文件夹，里面装着四样东西。一个标准的 Skill 文件夹结构长这样：

```
my-skill/
├── SKILL.md        # 必须：元数据 + 核心执行指令
├── scripts/        # 可选：可被直接调用的 Python/Bash 脚本
├── references/     # 可选：补充规则、格式标准、API 签名
└── assets/         # 可选：模板、样例数据、品牌色板
```

这四样东西各自解决一个 Agent 真实干活时的具体痛点。

SKILL.md 是指挥中心，脚本是动手能力，参考文件是随时可查的资料库，模板是每次产出的标准答案。它们不是同时被读进上下文的——这个后面会讲——而是只在需要的时候才出现。

这就直接回答了刚才说的第一个问题。没有装文档处理 Skill 的时候，Claude 面对"帮我生成一个带目录的 Word 文档"这个需求，它会用纯文本逻辑去模拟一个文档结构——所有排版都是手动的、所有的格式判断都基于它在文本训练里残留的经验。

但当你装了 Anthropic 官方的 docx Skill 之后，它读到的不再是"假设一个 Word 文档该怎么写"，而是一个由专业文档引擎支撑的执行逻辑——它能直接调用操作 OOXML 的格式指令，生成真正的自动目录、分页符和样式套用。

这就是"从纯文本猜测"到"具备真实格式控制能力"的区别。

而 Skills 在工程层面最大的突破，在于一个叫 **渐进式披露** 的加载机制。

传统 Agent 工作流是把所有工具说明书一次性塞进上下文——开发者实测 7 个 MCP 服务器就能在上下文里塞 9.87 万个 Token，模型还没干活，五分之一的 Token 预算已经租给了工具说明书。

而 Skills 的渐进式披露完全逆转了这个逻辑：Agent 启动时只加载每个 Skill 的 name 和 description（约 100 Token/个），相当于只拿了一份目录；当任务匹配到某个 Skill 的描述后，才加载 SKILL.md 正文（推荐控制在 5000 Token 以内）；正文里引用的外部资源——参考文档、模板、脚本——只在 Agent 实际执行到对应步骤时才被调用。

> 装了 50 个 Skill 也只在启动时占约 5000 Token，而不是 50 本手册同时摊开在上下文里等着被一页一页翻。

这就是为什么装了 Skill 之后大家会感觉 AI"变聪明了"——不是模型进化了，是上下文富裕了。Token 预算没有被操作手册吃掉，模型的注意力没有被稀释。

另外有一个很多人搞混的区域：**Skills 和 MCP（Model Context Protocol）不是竞争关系。**

它们分工完全不同。MCP 解决的是"我能调什么工具"，Skills 解决的是"怎么把这个工具用对"。Anthropic 的工程师用一个很形象的比喻把这件事说透了——MCP 是飞行员，Skills 是副驾驶。你不会因为副驾驶认识路，就把飞行员炒掉。

---

## 你一定要学会的三条命令：add、check、update

工欲善其事，先会命令行。Skills 的所有操作都通过一个叫 skills 的 CLI 工具完成，它由 Anthropic 官方维护，通过 npx 零安装调用。

### 安装一个 Skill

```bash
# 格式：npx skills add <GitHub 仓库地址> [--skill <具体技能名>] [选项]

# 最常用：从 skills.sh 注册表安装
npx skills add vercel-labs/agent-skills --skill frontend-design
```

这条命令会自动把 frontend-design 这个 Skill 下载到你的项目 `.claude/skills/` 目录下，Claude Code 下次启动时会自动扫描这个目录，不需要手动刷新。

如果想让一个 Skill 在所有项目里都能用，加上 `-g`（global）：

```bash
npx skills add vercel-labs/agent-skills --skill frontend-design -g
```

全局 Skill 的路径是 `~/.claude/skills/`，跟项目级的 `./.claude/skills/` 互不冲突。

### 查看已安装的 Skill

```bash
npx skills list -g
```

### 检查是否有更新版本

```bash
# 查看当前项目已安装 Skills 的版本状态
npx skills check
```

这个命令会对比你本地安装的 Skill 版本和远程仓库最新版，告诉你哪些 Skill 有可用更新。

### 升级 Skill 到最新版

```bash
# 一键升级所有可更新的 Skill
npx skills update
```

升级之后重启 Claude Code 或重新加载会话即可生效。建议每两到三周跑一次 `npx skills check`，Skills 生态更新频率很高，社区热门的 Skill 有时候一周就换个版本。

### 搜索 Skill

```bash
# 格式：npx skills find <搜索关键词>
npx skills find podcast
```

搜索结果会按安装量排序——安装量是最硬的质量指标，优先选安装量高的。

### 卸载一个 Skill

```bash
# 删除本地 Skill 目录即可
rm -rf .claude/skills/<skill-name>
```

除了命令行，也可以直接浏览 skills.sh 的市场网站，按分类和安装量排行搜索——这是目前公认的两条寻找 Skill 的首选渠道。

---

## 第一梯队：所有人必须装的生态基础设施（4 个 Skill）

第一梯队不是给某一个职业准备的。不管你写代码还是写文章，做前端还是做数据分析，这四个 Skill 迟早会救你的命。前三个现在就应该装，第四个先放着，等你遇到第八个想装的 Skill 的时候再回来——你会谢谢自己。

### 1. superpowers —— 没有纪律的 Agent 是开不出工地的坦克

**安装量：170K+ stars，Skills 生态里 GitHub 星标最高的项目之一**

superpowers 是 Claude Code 核心贡献者 Jesse Vincent 的开源项目。它不给你任何新能力，而是给 Claude Code 套上一整套强制执行的工程门禁。

裸用 Claude Code 的时候，你跟它说"帮我重构这个用户鉴权模块"，它的自然状态是立刻写代码。它不会先跟你讨论技术选型，不会先拆成原子任务，不会先写会失败的测试。Jesse Vincent 自己的代码在开源社区里有一个很出名的数字——他的 PR 被拒绝率高达 94%。一个对自己代码质量严苛到这种程度的人，设计 superpowers 的逻辑就是：**不让 AI 跳过任何一个他脑子里不可妥协的工程纪律。** 它把过去你在脑子里默念的"先想清楚再动手"，变成了一套 AI 绕不过去的强制流程。

它提供 9 大子技能，覆盖从头脑风暴到审查合并的全工作流——brainstorming 在写代码前强制先做苏格拉底式追问，writing-plans 把复杂需求拆成 5-10 个可验证的原子步骤，executing-plans 严格按步骤逐条执行且每步验证，test-driven-development 强制先写会失败的测试再写代码。如果你不做任何配置，裸用 Claude Code 写复杂功能时，AI 跳过前期讨论直接写代码的概率非常高，后期的返工和边界遗漏会让你怀疑人生。装了 superpowers 之后，这套门禁会把那种"想到哪写到哪"的惯性直接拦在第一关门口。

```bash
npx skills add https://github.com/obra/superpowers
```

> ⚠️ **注意**：superpowers 是全流程门禁，微型任务（单文件改动、半小时能搞定的）关掉它，只留原生 Claude Code 更省 Token。把小任务也过全套门禁就像你只是想修个按钮文案，AI 先拉你聊二十分钟这按钮的存在意义和用户动机——代价远大于收益。

### 2. find-skills —— 整个 Skills 生态的第一把钥匙

**安装量：上线 4 天破 5.2 万，目前 Skills 生态安装量最高**

find-skills 是这个生态里的"元技能"。它自己什么具体活都不干，但有了它，你再也不用手动去 skills.sh 翻页找技能。你只需要对 Claude Code 说一句"帮我找个做浏览器自动化的 Skill"，它会自动搜索注册表、按安装量排序、把最优的几个列出来让你选、然后自动帮你装好。社区实测，用 find-skills 之后，从"我需要一个某领域的 Skill"到"它已经在帮我干活的"这个时间，缩短了 20 倍以上。

```bash
npx skills add vercel-labs/skills --skill find-skills -g
```

> 建议全局安装——这个 Skill 跟具体项目无关，你会在所有项目里都需要它。装完后对 Claude Code 说"帮我搜索一个数据分析的 Skill"，它就会自动跑 `npx skills find` 搜索。

### 3. skill-creator —— 把你脑子里的经验变成可以让 AI 无限复用的资产

**安装量：Anthropic 官方出品，Skills 生态里的"母机"**

如果你不仅想消费别人的 Skill，还想把自己的经验沉淀下来，这个 Skill 就是你最值钱的资产。它引导你走完从捕捉需求到创建 SKILL.md 到测试用例到迭代优化的完整闭环，2026 年 3 月升级的 2.0 版本还加入了自动描述优化器——哪怕你写的 description 模糊得像"这东西帮我干点事"，它也能帮你改写成可以被 Agent 精确触发的版本。

```bash
npx skills add anthropics/skills --skill skill-creator -g
```

> 同样建议全局安装——你会在任何项目里用它来封装经验。

### 4. claude-mem —— 真正具备"长期记忆"的数字大脑

**当前状态：Claude Code 生态中最火的记忆增强插件，开发者社区实测好评如潮**

长任务最怕一件事：你今天刚带 Claude 理顺了复杂的业务逻辑，明天新开会话它就变回了"最熟悉的陌生人"。

claude-mem 彻底终结了这种挫败感。它不像传统的 session-manager 只是死板地备份 JSON，而是通过 SQLite + 向量数据库，将你的项目决策、代码变动和技术讨论转化为可检索的知识库。当你开启新会话时，它会自动同步之前的上下文；更离谱的是，你可以直接问它："上周重构支付模块时，我们最后决定的接口协议是什么？"——它会像一直在场的老搭档一样秒回你。

```bash
/plugin marketplace add thedotmack/claude-mem
```

> claude-mem 是中大型项目协作的"入场券"。它不仅省去了你重述背景的口舌，更赋予了 Claude 跨越时空的逻辑一致性，是你进行模块化重构、复杂 Debug 时的最强辅助。

---

## 第二梯队：文档与数据处理，所有职场人的共同底座

这个梯队不需要解释。只要你工作中碰过 PDF、Excel、Word 三种文件中的任何一种，这个梯队就是你挡掉那些重复性手工搬运劳动的第一道墙。Anthropic 官方出品的文档 Skill 在底层做了精确的格式控制，不是靠模型凭记忆猜出来的排版，而是能真正操作文件格式内部的文档结构。

### 5. pdf —— 让 AI 真正读懂 PDF 的内部表格和排版

**安装量：Anthropic 官方出品，Skills 生态下载量最大的文档类 Skill 之一**

传统 AI 在读取 PDF 时极其容易出幻觉——尤其是遇到表格或复杂排版，大面积抓偏和乱序是日常。这个 Skill 内置了专业 PDF 解析逻辑，能从一份 300 页的年报里自动抽出指定财务表格，保存为结构化数据。

```bash
npx skills add anthropics/skills --skill pdf
```

### 6. xlsx —— Excel 的全链路自动化处理

**安装量：Anthropic 官方出品，办公场景核心基础 Skill**

处理 .xlsx、.xlsm、.csv、.tsv 等表格文件——从读取修改到数据清洗到制作图表到格式化转换，全部原生支持。它被官方专门设计为"只输出真正的 Excel 可操作工作表"，不会偷偷换成一个 HTML 报表或 Python 脚本冒充 Excel。

```bash
npx skills add anthropics/skills --skill xlsx
```

### 7. docx —— 生成的 Word 有自动目录和样式

**安装量：Anthropic 官方出品，持续维护超过一年**

正因为这个 Skill 的存在，文章开头那个"调整格式就烧掉数十万 Token"的问题才被彻底终结。它能生成自动目录、统一样式、页码自动对齐，所有格式变更直接操作底层的 OOXML 指令，不经过文本转义层。

```bash
npx skills add anthropics/skills --skill docx
```

---

## 第三梯队：设计风格与视觉规范，告别"AI 味道"

这个梯队解决的就是文章开头第三个问题——AI 写的界面跑得通但眼光一过就知道是人还是机器写的。

### 8. frontend-design —— 用"强势风格"打穿万年蓝白灰

**安装量：skills.sh 排行榜前端大类第一**

它不教你写代码，而是逼 AI 在写代码之前先定设计方向：你要的是极简、复古未来还是粗野主义？定好方向之后，它在这个基调的约束下逐组件执行——专门跟"默认蓝白卡、菜单全对称、按钮永远灰"的 AI 惯性对着干。

```bash
npx skills add https://github.com/anthropics/skills --skill frontend-design
```

> 与 web-design-guidelines 一起装——一个管审美，一个管规范，互补不能互替。

### 9. web-design-guidelines —— 300 多条铁律

这个 Skill 不带任何风格决策，只做代码约束审查和专业交互规范——对比度合不合无障碍标准、响应式不同尺寸下组件是否叠合、按钮有无按压反馈。跟 frontend-design 放在一起用就是"一个有审美的上司"和"一个铁面无私的合规官"的组合。

```bash
npx skills add https://github.com/vercel-labs/agent-skills --skill web-design-guidelines
```

---

## 第四梯队：后端与工程自动化，开发者的生产力飞轮

进入真刀真枪的开发场景后，你会发现代码写得快和代码写得好之间，其实是靠一系列自动检查、自动验证和自动测试的步骤来闭环的。如果这些步骤你没制度化，那你就得靠自己的脑子和体力——而人的精力和注意力在过了半个晚上之后，早已不是第一天上午的那种度量。

### 10. gstack —— 23 个虚拟角色组成你一个人的工程团队

**安装量：YC CEO Garry Tan 开源，60 天内产出 60 万行生产代码、其中 35% 是测试**

Garry Tan 把二十年的工程管理经验压缩成了 23 个角色：产品经理帮你做需求澄清（`/office-hours`）、技术评审画架构图（`/plan-eng-review`）、代码审查员逐行挑逻辑隐患（`/review`）、QA 在真实浏览器里点你的页面并看交互是否完整。这些角色共享同一个模型引擎，但关注点彼此隔离——工程师不碰设计，审查不碰实现，QA 不修 Bug。背后的设计逻辑就是"注意力边界"——AI 不需要更聪明，它需要的是在不同阶段戴上不同的注意力透镜。

```bash
git clone https://github.com/garrytan/gstack ~/.claude/skills/gstack
cd ~/.claude/skills/gstack && chmod +x setup && ./setup
```

### 11. agent-browser —— 让 AI 拥有视觉，而不只是读代码

**安装量：浏览器自动化赛道安装量最高**

这个 Skill 是社区用于网页自动化测试的高频工具。AI 不再只能分析你的代码——它可以直接打开无头浏览器，真的去点击你的页面按钮、填写表单、做页面抓取。是前端的自动化回归与终端测试中最被低估的基础设施。

```bash
npx skills add anthropics/skills --skill agent-browser
```

---

## 第五梯队：发布、运维与持续监控

代码写得再好，最后一步的包装决定职业感的上限。GStack 内置的 `/land-and-deploy` 和 `/make-pdf` 就是为这一步而生的。

`/land-and-deploy` 自动合并分支、跑全部前置检查、打 PR 描述、部署，一气呵成；`/make-pdf` 则把你在 Claude Code 里产出的架构图、milestone 进度或研发文档一键转换成可以分发的最终交付文档。前者省掉的是合并漏错和部署步序颠倒的老毛病，后者堵住的是"代码写完了但附加材料东拼西凑"的专业形象缺口。

> 记住一个底层原则：**评判某个 Skill 是否值得安装，就看它是否能替代你日常重复执行的一套定型动作。** 如果这类动作每周出现两次以上，它被固化进 Skill 的直接收益远超新增的 Token 开销。

---

## 按角色定制：不同职业的 Skills 安装方案

这部分是真正的核心。不同职业的人，最需要的能力模块完全不同。以下方案对应的职业分布已经尽可能覆盖目前社区反馈和一线团队的实际配合测试。

### 前端开发

**核心痛点**：代码长得太像 AI 写的、没有交互规范、适配在一种屏幕下完美但换尺寸就崩坏。

**必装核心 5 件套**：frontend-design、web-design-guidelines、agent-browser、qa（gstack 内建命令）、gstack（核心全栈审查链）。前两个锁审美和规范，agent-browser 负责浏览器截图验证和 DOM 抓取，qa 负责在真实浏览器里跑完整的交互回归测试。建议加上 remotion-best-practices，给动画注入表现力。

### 后端开发

**核心痛点**：错误处理漏项、并发安全没有强制回归、API 设计缺乏标准化约束。

**必装核心 5 件套**：superpowers、gstack（工程评审器、代码审查员和安全官三个角色都必须跑）、claude-mem、docx、data-analysis。superpowers 负责 TDD 强制纪律和原子任务拆分，gstack 的执行流程重点在于审查和安全性——写代码和执行测试两个 Agent 的上下文被强制隔离，AI 不会自己给自己的代码打满分。data-analysis 是后端问题排查、日志分析和性能诊断的高效探针，绝不仅是给数据分析师用的。

### 全栈开发

**核心痛点**：模块边界模糊、前后端数据契约缺少强制约束。

全栈的 Skill 配置本质上是前端和后端核心组合的交集加一道对接铁桥。在已经完整部署前端和后端核心套件的前提下，加上 claude-mem 维持跨上下文一致性，再加上 gstack 的工程评审器做全栈架构评审——当全栈职责同时涉及前端、后端与数据库接口时，还需要增加 data-analysis 来做数据一致性交叉验证。

### 一人公司创始人

**核心痛点**：你是唯一的技术与产品负责人，没人替你想战略、做测试、做安全审计。

**必装 Skill 清单**：superpowers、gstack（重点跑 CEO 审查和安全审计角色）、claude-mem、excalidraw-diagram、data-analysis、agent-browser。CEO 审查角色负责在你兴奋地写代码之前拉你停下来想清楚"这个功能到底值不值得做"，安全审计角色负责在你一个人没有任何安全同事的前提下检索 OWASP 和 STRIDE 层面的结构性漏洞。excalidraw-diagram 可以把你的业务想法直接变成可嵌入 BP 或客户演示的架构图。

### 数据分析师

**核心痛点**：数据全是散乱的原始材料，分析流程没被标准化过，每次都是手工重新来。

**必装 Skill 清单**：data-analysis、xlsx、pdf、agent-browser、superpowers。优先级这样排：data-analysis 是雷达和探针，xlsx 是纯工作底稿的直接操作引擎，pdf 是材料入口关，agent-browser 提供第一线数据抓取与线上信息采集。superpowers 在这里的重点不是 TDD——而是把每次分析项目拆成固定的工作流模板，一旦固化，后面同样的分析流程基本接近自动化运转。

### 产品经理/技术写作

**核心痛点**：产出质量波动大，竞品信息搜集全靠手工。

**必装 Skill 清单**：docx、pdf、excalidraw-diagram、brainstorming（superpowers）、agent-browser、frontend-design。文档和图表是主要输出载体，docx/pdf 负责终稿级产物输出，excalidraw-diagram 负责快速画业务架构和泳道图，brainstorming 是前期发散思考时最可靠的机制。agent-browser 对竞品市场调研的价值在这个组合里非常关键。

### DevOps/安全工程师

**核心痛点**：部署链条断在没人盯的边缘环境上。

**必装 Skill 清单**：gstack（部署与安全角色）、superpowers、agent-browser。安全官角色在此获取完整的访问控制检查逻辑，部署角色用于流水线闭环，qa 配合 agent-browser 进行生产环境状态的基本冒烟测试。

### 在校学生/自学编程新手

**核心痛点**：没有工程习惯，写的代码全是"能跑就行"，学不到规范。

**必装 Skill 清单**：superpowers、data-analysis、find-skills。superpowers 在这里的价值跟前面讲的完全相同——它是一套让 AI 从"裸写代码"变成"按专业软件工程流程逐步推进"的强制纪律。data-analysis 提供了从零探索数据集和理解数据形态的完整路径。

---

## 避开这些坑，Skills 就能用到退休

### 坑一：安装越多的 Skill 并不代表生产力就越高

Anthropic 官方自身的建议是持有 20 到 30 个 Skill。实测下来，日常高频触发的 Skill 往往只有 5 到 7 个——其余的存在意义是为特殊项目准备的备弹，而非每周必用的默认装备。如果你一口气装了 50 个 Skill 却常把工具描述全部挤进上下文，Token 消耗会直接挤出模型的推理能力。

### 坑二：安装后找不到 Skill

你敲了 `npx skills add`，但 Claude Code 里怎么都叫不出这个 Skill。这是所有新手 100% 会踩的坑。原因是你把 Skill 装到了 `.agents/skills/` 目录下，但 Claude Code 只识别 `.claude/skills/`。解决方案很简单：装 Skill 的时候全程在项目根目录下操作，不要跳目录；装完以后用 `npx skills list` 检查，确认路径落在 `.claude/skills/` 下。

### 坑三：安装社区来路不明的 Skill 时不做背调

Skills 生态全部是开源的，这意味着任何人都能发布 Skill。2026 年已经有安全机构披露了公网 Skill 市场中存在恶意 Skill——内置窃密和挖矿逻辑。基本安全底线就三条：优先选择高安装量官方源；安装前检查仓库星标数和最后更新时间；不下载不明个人源文件。

### 坑四：装了 Skill 就跑，不更新

Skills 生态目前仍在高速进化：热门 Skill 几周到几个月就可能切版本。`npx skills check` 每两到三周做一次，发现标记了版本过期且已由官方或大社区接手维护的，用 `npx skills update` 一口气搞定。

### 坑五：Skill 装了但不用，占着上下文预算吃干饭

每个 Skill 在启动时都会占用少量元数据（约 100 Token/个），50 个 Skill 约 5000 Token——对宽裕的上下文窗口来说不算多，但如果里面有一半是你三个月才用一次的冷备件，它们就只是放着占地方的长期成本。每季度批量做一次 Audit——哪些 Skill 过去三个月 0 触发？直接删，或者移到 `~/.claude/skills-archive/` 里备用。

---

## 写在最后

去年我第一次接触 Skills 时的状态，跟你现在可能的状态一样——不知道从哪下手，不知道哪个该装，不知道装完是真变强还是在吃 Token。花了十几个晚上把高频 Skill 一个一个试完，现在剩下的结论就写在上面那几十行里。

**如果你只想记住三件事：**

1. 裸用 Claude Code 的时候赶紧找 **superpowers** 替你在门口问清楚那几个该回答的问题
2. 用 **find-skills** 能让你不至于在茫茫 Skill 海和排着队的"AI 专家推荐文"里扒拉一个晚上
3. Skill 不在于多，在于能在你**每周重复两次以上的工作流**里替你默默接住那些永远不变的琐碎

文章最开头那段话——我让 Claude Code 第一次裸批一份带目录的技术文档时烧掉的那数十万 Token——直到三个月后的今天想起还是会一阵心疼。但后来我学会了：当某个专业接口总有专业的执行脚本在背后扛着的时候，你就不会再把 Token 烧在调整文本对齐上——你会把注意力全部放回真正的产出内容。**这才是 Skills 体系最核心的价值。**
