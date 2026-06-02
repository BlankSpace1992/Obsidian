---
title: Git 实际开发使用流程指南
author: CodeHeart
source: https://mp.weixin.qq.com/s/V35U-jaRjrVLoejyRhrh1w
date: 2025-05-14
tags:
  - Git
  - 版本控制
  - 开发工具
  - 团队协作
  - 分支管理
---

# Git 实际开发使用流程指南

> 原文链接：[CodeHeart](https://mp.weixin.qq.com/s/V35U-jaRjrVLoejyRhrh1w)
>
> 从项目初始化到日常协作，一份覆盖全场景的 Git 实战手册。建议收藏，随时查阅。

---

## 一、初始化本地仓库（从零开始）

当你需要在一个全新的本地目录开始一个项目时：

```bash
# 1. 创建并进入工作目录
mkdir mythy
cd mythy

# 2. 初始化 Git 仓库
git init
```

> 💡 此时会在当前目录下生成一个隐藏的 `.git` 文件夹，标志着一个本地仓库的诞生。

---

## 二、关联远程仓库并提交代码

将本地代码首次推送到远程仓库（以 Gitee / GitHub 为例）：

```bash
# 1. 添加远程仓库地址（建议统一使用 origin 作为默认别名）
git remote add origin https://gitee.com/ywc0414/mythy.git

# 修改远程仓库的名称
git remote rename origin newname

# 2. 将所有文件添加到暂存区
git add .

# 3. 提交到本地仓库
git commit -m "first commit"

# 4. 推送到远程仓库的 master 分支
# 第一次推送需加上 -u 参数建立追踪关系
git push -u origin master
```

> 🔑 **关于回退**：如果刚提交完发现有问题，想回退到上一次提交（⚠️ 慎用，会丢失修改）：
>
> ```bash
> git reset --hard HEAD~1
> ```

---

## 三、克隆现有项目（加入团队）

大多数情况下，我们是加入一个已存在的项目，无需手动 `init`：

```bash
# 1. 克隆远程仓库到本地
git clone https://gitee.com/ywc0414/mythy.git

# 2. 进入目录
cd mythy

# 3. 查看状态
git status
```

输出示例：`Your branch is up-to-date with 'origin/master'.`

> ✅ 表示本地与远程同步，可以放心开始开发。

---

## 四、日常开发循环（增删改查）

这是开发者**每天重复频率最高**的操作：

```bash
# 1. 修改代码后，添加所有变更到暂存区
git add .
# 或者只添加特定文件：git add filename

# 2. 提交变更到本地仓库
git commit -m "feat: 添加新功能模块"

# 3. 推送到远程仓库
# 如果之前建立过追踪（-u），直接输入 git push 即可
git push origin master
```

> ⚠️ 推送前建议先拉取最新代码以防冲突：
>
> ```bash
> git pull origin master
> ```

---

## 五、分支管理（Branching）

> **核心原则**：永远不要直接在 master（或 main）分支上开发新功能。应为每个功能或修复创建一个独立分支。

### 5.1 创建与切换分支

```bash
# 方法 A（传统）：创建并切换
git checkout -b iss1

# 方法 B（新版推荐）：创建并切换
git switch -c iss1

# 基于特定 Commit ID 创建分支（用于回溯修复）
git checkout -b hotfix-branch <commit-id>
```

### 5.2 查看与推送分支

```bash
# 查看本地分支（* 表示当前所在分支）
git branch

# 将新分支推送到远程，并建立追踪关系
git push -u origin iss1
```

### 5.3 合并分支与清理

当功能开发测试完成后，将其合并回主分支：

```bash
# 1. 切换回主分支
git checkout master
# 或 git switch master

# 2.（可选）拉取远程最新代码
git pull origin master

# 3. 将 iss1 分支合并到当前分支
git merge iss1

# 4. 推送合并后的主分支到远程
git push origin master

# 5. 删除本地已合并的分支
git branch -d iss1

# 6. 删除远程分支（清理远程仓库）
git push origin --delete iss1
```

### 5.4 其他分支操作

```bash
# 修改分支名称（旧名 → 新名）
git branch -m oldName newName

# 同步远程分支列表（清理本地已不存在的远程分支记录）
git fetch -p
```

---

## 六、版本标签（Tag）

用于标记发布版本（如 v1.0.0），标签通常打在特定的 Commit 上。

```bash
# 1. 给当前提交打标签
git tag v1.0.0 -m "发布第一个正式版本"

# 2. 给历史某次提交打标签（需指定 commit-id）
git tag v0.9.0 -m "回溯版本" 039bf8b

# 3. 查看标签
git tag          # 查看本地所有标签
git show v1.0.0  # 查看标签对应的提交详情

# 4. 推送标签到远程
git push origin v1.0.0       # 推送单个标签
git push origin --tags       # 推送所有本地标签到远程

# 5. 获取远程标签
git fetch origin --tags

# 6. 删除标签
git tag -d v1.0.0              # 删除本地标签
git push origin --delete v1.0.0 # 删除远程标签
```

---

## 最佳实践（Best Practices）

| # | 实践 | 说明 |
|---|------|------|
| 1 | 提交信息规范化 | 推荐使用 `类型(范围): 描述` 格式，例如 `feat(user): 添加用户登录功能` |
| 2 | 频繁提交，少量推送 | 本地可以多 commit 保存进度，但推送到远程前确保代码可运行 |
| 3 | 解决冲突 | `git pull` 或 `git merge` 出现冲突时，手动编辑文件解决冲突标记（`<<<<<<<`、`=======`、`>>>>>>>`），然后重新 add 和 commit |
| 4 | 统一远程别名 | 建议全团队统一使用 `origin` 作为默认远程仓库别名，避免混淆 |

---

> Git 的学习曲线不算陡峭，但要真正用好它，需要理解**工作区 → 暂存区 → 本地仓库 → 远程仓库**这四个概念的关系，以及在团队协作中养成良好的分支管理习惯。
