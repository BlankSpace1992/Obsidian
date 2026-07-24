---
title: Nano ID：轻量安全的分布式唯一 ID 生成方案
author: 开源推荐
source: https://mp.weixin.qq.com/s/IYSYaAJdVwbqTVQXkx4JJQ
date: 2025-07-24
tags:
  - NanoID
  - UUID
  - 分布式
  - ID生成
  - 开源工具
  - 前端
---

# Nano ID：轻量安全的分布式唯一 ID 生成方案

> 原文链接：[开源推荐](https://mp.weixin.qq.com/s/IYSYaAJdVwbqTVQXkx4JJQ)
>
> GitHub：[ai/nanoid](https://github.com/ai/nanoid)

---

## 一、是什么？

轻量级唯一 ID 生成开源库，专为前后端、分布式系统设计。核心压缩后仅 **118 字节**，零外部依赖。

与 UUID 对比：

| 指标 | Nano ID | UUID v4 |
|------|---------|---------|
| 体积 | 118 字节 | 423 字节 |
| ID 长度 | 21 字符 | 36 字符 |
| 存储节省 | — | 42% |
| 碰撞概率 | 万亿级才十亿分之一 | 相当 |
| 字符集 | URL 安全（无特殊字符） | 含 `-` |

> 已移植至 Java、Python、Go、Rust 等 20+ 编程语言。

---

## 二、核心功能

### 加密级安全随机

- 浏览器调用 Web Crypto API，Node.js 使用 crypto 硬件随机源
- 字符映射算法规避 `随机数 % 字符集` 导致的分布不均问题
- 每个字符出现概率完全均等，无法暴力枚举预测

### 高度自定义

- 长度 1–256 位
- 自定义字符集（纯数字、纯字母、无特殊符号）
- 非安全高性能分支 `nanoid/non-secure`（无加密开销，适合本地临时标识）

### 全环境兼容

- ESM / CommonJS / CDN 三种引入方式
- 原生 TypeScript 支持
- 内置 CLI 命令行工具
- `_` 与 `-` 完全兼容 URL，无需转义

### 分布式集群友好

不依赖机器 ID、时间戳，多服务多实例并发生成无冲突，无需分布式锁、雪花算法时钟回退修复等额外逻辑。

---

## 三、快速使用

### 安装

```bash
npm install nanoid
# 或 CDN 直接引入
# <script src="https://cdn.jsdelivr.net/npm/nanoid/nanoid.js"></script>
```

### 代码

```js
import { nanoid, customAlphabet } from 'nanoid'

// 默认 21 位安全 ID
const orderId = nanoid()

// 自定义 8 位短 ID（短链接）
const shortUrlId = nanoid(8)

// 自定义纯数字 10 位 ID
const numId = customAlphabet('0123456789', 10)()

// 非安全高性能模式（本地缓存 key）
import { nanoid } from 'nanoid/non-secure'
const tempKey = nanoid(6)
```

### CLI 批量生成

```bash
npx nanoid --size 10                # 10 位 ID
npx nanoid --alphabet ABC123 --size 6  # 自定义字符集 6 位
```

---

## 四、源码亮点

- **极致体积**：核心逻辑仅数十行，打包后几乎不增加前端产物体积
- **位运算优化**：位掩码批量映射随机字节到字符集，减少循环计算，比 UUID 逐字节拼接性能提升 30%+
- **分层模块化**：安全模块、非安全高速模块、自定义字符集模块解耦独立，按需导入
- **完备测试**：内置百万级随机分布测试，校验字符均匀性、碰撞概率

---

## 五、适用场景

| 场景 | 用途 |
|------|------|
| Web 前端 | React/Vue 列表临时 key、文件名、会话标识 |
| 后端数据库 | MySQL/PostgreSQL 分布式主键、订单流水、日志编号 |
| 短链接系统 | URL 唯一编码 |
| 多端跨平台 | 小程序、移动端离线本地数据 ID |
| 低性能设备 | 嵌入式、轻量化脚本 |

---

## 六、对比总结

| 方案 | 体积 | 分布式 | 安全性 | URL 友好 |
|------|------|--------|--------|---------|
| Nano ID | 118B | ✅ 无冲突 | ✅ 加密随机 | ✅ |
| UUID v4 | 423B | ✅ 无冲突 | ✅ 加密随机 | ❌ 含 `-` |
| 雪花 ID | 自己实现 | ⚠️ 需时钟同步 | ❌ 可推测 | ✅ 纯数字 |

> 对于中小型项目、前端主导系统、分布式微服务，Nano ID 可直接替换 UUID 作为全局唯一 ID 生成方案。
