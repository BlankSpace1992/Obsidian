---
title: Claude Code 降本利器：RTK 代理工具
author: 苏三说技术
source: https://mp.weixin.qq.com/s/ZMfictVdEKT_N9U6Jyc-cw
date: 2025-05-14
tags:
  - Claude-Code
  - AI工具
  - 开源
  - 效率
---

# Claude Code 降本利器：RTK 代理工具

> 原文链接：[苏三说技术](https://mp.weixin.qq.com/s/ZMfictVdEKT_N9U6Jyc-cw)

---

## 概述

使用过 Claude Code 的小伙伴应该有所了解，Claude Code 有个 **200k 的 LLM 上下文**。如果我们在命令达到 LLM 上下文之前，不做过滤和压缩的话，上下文很快就会占用过高了，不仅会导致 AI 推理能力变差，而且会消耗大量的 token。

今天给大家分享一款高性能 CLI 代理 **RTK**，能大大降低 token 的消耗！

---

## RTK 简介

RTK 是一款高性能的 CLI 代理，它能在命令输出到达 LLM 上下文之前进行**过滤和压缩**，能将 token 消耗降低 **60-90%**，目前在 Github 上已有 **25k star**！

以 `git status` 命令为例，RTK 的工作原理如下：

**没有 rtk：**

```
Claude  --git status-->  shell  -->  git
  ^                                   |
  |        ~2,000 tokens（原始）       |
  +-----------------------------------+
```

**使用 rtk：**

```
Claude  --git status-->  RTK  -->  git
  ^                      |          |
  |   ~200 tokens        | 过滤     |
  +------- （已过滤）-----+----------+
```

下面是某开发者使用 RTK 几周后的真实反馈，节约了接近 **89%** 的 token。

---

## 安装

以 Windows 环境为例：

1. 首先在 RTK 的 release 页面下载安装包
   - 下载地址：[https://github.com/rtk-ai/rtk/releases](https://github.com/rtk-ai/rtk/releases)

2. 下载成功后解压会得到一个 `rtk.exe` 可执行程序，将路径添加到**环境变量 → 系统变量 → Path** 中：

```
Path = D:\developer\tools\rtk
```

3. 在命令行中使用以下命令验证安装：

```bash
rtk --version
```

如果输出了版本号，就代表 RTK 已经安装成功了！

---

## 使用

以 Claude Code 为例，讲解 RTK 的使用：

### 1. 全局初始化

```bash
rtk init -g
```

此时 Claude Code 的 `CLAUDE.md` 配置文件中会自动添加 RTK 的使用说明。

### 2. 自动命令转换

打开 Claude Code 的 CLI，通过 `git status` 命令测试，该命令会自动转换为 `rtk` 开头的命令。

### 3. Token 节约效果

转换后的命令能大大降低命令传入 LLM 上下文的 token 大小。

### 4. 支持的命令

RTK 支持转换的命令有 **30 多个**。

### 5. 查看节约情况

使用一段时间后，可以使用以下命令查询 token 的节约情况：

```bash
rtk gain
```

---

## 总结

RTK 能让我们在使用 Claude Code 时获得：

- **更好的推理** — 上下文更精炼，AI 推理质量更高
- **更长的会话** — 节省 token 意味着可以进行更多对话
- **更低的成本** — Token 消耗降低 60-90%

---

## 项目地址

[https://github.com/rtk-ai/rtk](https://github.com/rtk-ai/rtk)
