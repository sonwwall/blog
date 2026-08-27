---
title: 学习笔记：Git Worktree 的使用与理解
abbrlink: learning-note-20260827
date: 2026-08-27T16:39:46
updated: 2026-08-27T16:39:46
tags:
  - 学习笔记
  - 学习
  - Git
  - Worktree
categories:
  - 学习笔记
desc: 对 Git Worktree 的概念、使用场景、常用命令和易混点进行系统整理。
---

# 学习笔记：Git Worktree 的使用与理解

它解决的问题很具体：当一个分支开发到一半，不想破坏当前现场，但又需要马上处理另一个分支时，可以给同一个 Git 仓库创建多个独立工作目录。

可以先记住一句话：

> branch 是代码版本线，worktree 是这个分支在硬盘上展开出来的工作目录。

普通 Git 使用方式下，一个仓库通常只有一个工作目录。你要从 `main` 切到 `feature-login`，需要执行：

```bash
git switch feature-login
```

这意味着当前目录只能处于一个分支上。使用 worktree 后，同一个仓库可以同时拥有多个工作目录：

```text
project/          -> main
project-login/    -> feature-login
```

这样就可以同时打开两个目录，分别开发、运行、测试不同分支。

## 一、Worktree 是什么

`git worktree` 的作用是让同一个 Git 仓库同时对应多个工作目录，并让这些目录分别检出不同分支。

例如原本只有：

```text
Desktop/
└── project/       -> main
```

执行：

```bash
git worktree add ../project-login feature-login
```

之后硬盘上会真的多出一个目录：

```text
Desktop/
├── project/       -> main
└── project-login/ -> feature-login
```

这两个目录里都会有项目文件：

```text
project/
├── main.go
├── go.mod
└── internal/

project-login/
├── main.go
├── go.mod
└── internal/
```

你在 `project-login/main.go` 里修改代码，不会直接改到 `project/main.go`。它们是两个独立的工作目录。

## 二、Worktree 不是新的分支

这是最容易混淆的地方：worktree 本身不是分支。

分支表示代码版本线，比如：

```text
main
feature-login
hotfix-payment
```

worktree 表示某个分支在磁盘上的工作目录，比如：

```text
project/          -> main
project-login/    -> feature-login
project-hotfix/   -> hotfix-payment
```

所以二者关系可以理解为：

```text
branch   = 代码版本
worktree = 把某个代码版本展开出来进行开发的目录
```

如果没有 worktree，多个分支只能轮流占用同一个目录。使用 worktree 后，不同分支可以同时拥有自己的目录。

## 三、Worktree 和 Clone 的区别

表面上看，worktree 和多 clone 几份仓库都能得到多个目录，但本质不同。

如果执行两次 `git clone`，会得到两套独立仓库：

```text
project1/
└── 自己的一整套 .git 数据

project2/
└── 自己的一整套 .git 数据
```

这两个仓库之间互相独立，各自维护 Git 对象、分支、引用和历史。

worktree 则是同一个仓库下面挂出多个工作目录：

```text
              一个 Git 仓库
              /          \
             /            \
project-main/        project-feature/
main                 feature
```

多个 worktree 共享很多 Git 数据，例如：

- commit
- object
- branch
- refs
- Git 历史

但每个 worktree 拥有自己独立的：

- 工作目录
- `HEAD`
- index，也就是暂存区
- 未提交修改

因此可以把 worktree 理解成：一个 Git 仓库，多张独立开发桌。

## 四、为什么需要 Worktree

worktree 最适合解决的问题是：

> 我不想动当前分支现场，但又想马上操作另一个分支。

### 1. 开发到一半，突然需要修线上 Bug

假设当前正在开发：

```text
project/ -> feature-agent
```

代码已经改了一半，此时线上出现 Bug，需要从 `main` 拉一个热修分支。

没有 worktree 时，通常要：

```bash
git stash
git switch main
git switch -c hotfix
```

修完以后再：

```bash
git switch feature-agent
git stash pop
```

这个流程不但麻烦，`stash pop` 还可能产生冲突。

使用 worktree 可以直接创建一个新目录：

```text
project/          -> feature-agent
project-hotfix/   -> hotfix
```

原来的开发现场完全不用动。

### 2. 同时运行不同分支做对比

有些任务需要对比两个实现方案，例如缓存优化、UI 改版、接口返回差异。

可以这样安排：

```text
project-old/   -> main
project-new/   -> optimize-cache
```

然后分别启动：

```text
main           -> localhost:8080
optimize-cache -> localhost:8081
```

这样可以直接对比：

- API 返回结果
- 性能表现
- UI 展示
- 测试结果
- 不同实现方案的代码结构

### 3. Code Review

假设自己正在开发：

```text
project/ -> feature-agent
```

同事提交了：

```text
feature-payment
```

如果直接 `git switch feature-payment`，就要切走自己的开发现场。使用 worktree 可以创建：

```text
project/           -> feature-agent
project-payment/   -> feature-payment
```

这样可以在本地运行同事分支，自己的代码不受影响。

### 4. AI Coding 并行方案

worktree 也很适合 AI 编程工具。

例如：

```text
project/        -> 自己开发
project-ai-1/   -> Agent 方案 1
project-ai-2/   -> Agent 方案 2
```

不同 Agent 可以在不同 worktree 中改代码，互不污染。最后再通过测试、diff 和人工审查比较哪个方案更好。

## 五、常用命令

### 1. 从已有分支创建 worktree

假设已经存在 `feature-login`：

```bash
git worktree add ../project-login feature-login
```

含义是：

```text
创建 ../project-login 目录
在该目录中 checkout feature-login
```

之后进入目录：

```bash
cd ../project-login
```

就可以正常开发。

### 2. 创建新分支并创建 worktree

如果分支还不存在，可以一边创建分支，一边创建 worktree：

```bash
git worktree add -b feature-login ../project-login main
```

含义是：

```text
从 main 创建 feature-login
创建 ../project-login 目录
在该目录中 checkout feature-login
```

它相当于同时完成了创建分支、签出分支、创建工作目录三个步骤。

### 3. 查看已有 worktree

```bash
git worktree list
```

示例输出：

```text
/Users/me/project        abc123 [main]
/Users/me/project-login  def456 [feature-login]
```

意思是：

```text
project       当前对应 main
project-login 当前对应 feature-login
```

### 4. 删除 worktree

如果某个 worktree 已经不需要了：

```bash
git worktree remove ../project-login
```

这只是删除工作目录，不会自动删除分支。

如果对应分支也不需要了，再执行：

```bash
git branch -d feature-login
```

所以要区分：

```text
git worktree remove = 删除工作目录
git branch -d       = 删除分支
```

## 六、在 Worktree 中如何开发和合并

进入 worktree 后，Git 使用方式和平时一样：

```bash
cd ../project-login
git status
git add .
git commit -m "feat: add login"
git push
```

worktree 不改变基本 Git 工作流。它只是让你多了一个独立目录。

合并时要注意：Git 并不是把两个 worktree 合并，而是把两个 worktree 对应的分支合并。

假设：

```text
project/          -> main
project-login/    -> feature-login
```

在 `project-login/` 中提交代码：

```bash
git add .
git commit -m "feat: add login"
```

然后回到主目录：

```bash
cd ../project
git branch --show-current
```

确认当前是 `main` 后执行：

```bash
git merge feature-login
```

真正发生的是：

```text
main + feature-login -> merge
```

而不是：

```text
project/ + project-login/ -> merge
```

worktree 只是分支的工作目录，分支才是 Git 合并的对象。

## 七、完整使用流程

假设需要从 `main` 创建 `feature-login` 并开发，完整流程可以这样写：

```bash
# 当前位于 project，分支为 main

# 创建 feature-login，并创建新的 worktree
git worktree add -b feature-login ../project-login main

# 进入新的 worktree
cd ../project-login

# 开发并提交
git add .
git commit -m "feat: add login"

# 回到 main 对应的 worktree
cd ../project

# 合并 feature-login
git merge feature-login

# 删除已经不需要的 worktree
git worktree remove ../project-login

# 删除已经合并的分支
git branch -d feature-login
```

这个流程里最重要的是：创建、开发、合并、删除分别操作的是不同对象。

```text
创建 worktree：git worktree add
开发代码：在新目录里正常 git add / commit
合并代码：git merge 分支名
删除目录：git worktree remove
删除分支：git branch -d
```

## 八、一个重要限制

同一个分支默认不能同时被两个 worktree checkout。

例如当前已经有：

```text
project/ -> main
```

这时通常不能再执行：

```bash
git worktree add ../project2 main
```

因为 `main` 已经被 `project/` 使用。

更常见的做法是从 `main` 创建一个新分支：

```bash
git worktree add -b hotfix ../project-hotfix main
```

结果是：

```text
project/          -> main
project-hotfix/   -> hotfix
```

这样每个 worktree 都对应不同分支，职责更清楚，也更不容易误操作。

## 九、容易混淆的点

### 1. Worktree 不是复制一份仓库

worktree 会创建真实目录，但不是完整 clone 一份新仓库。它和原仓库共享 Git 对象、历史和引用。

### 2. Worktree 不是分支

分支是版本线，worktree 是目录。合并、删除分支、推送远端，操作对象仍然是 branch。

### 3. 删除 worktree 不等于删除分支

`git worktree remove` 删除的是目录。分支还在，需要时可以继续 checkout 或重新创建 worktree。

### 4. 每个 worktree 有自己的未提交修改

不同 worktree 的工作区、暂存区和未提交修改彼此独立。在一个 worktree 中 `git status` 看到的状态，不等于另一个 worktree 的状态。

### 5. 不是什么时候都需要 worktree

如果只是正常开发完一个分支，再切到另一个分支，直接使用 `git switch` 就够了。

worktree 的价值出现在并行场景：

- 当前修改不能被打断
- 需要临时修 Bug
- 需要本地运行别人分支
- 需要同时对比两个实现
- 需要让多个 AI Agent 并行尝试不同方案

## 十、最终理解

普通 Git 像是一张开发桌：

```text
一张桌子
↓
main
```

要做 feature 时，需要把当前桌面内容切换成另一个分支：

```bash
git switch feature
```

Git Worktree 则是给同一个仓库增加多张开发桌：

```text
桌子 1           桌子 2
↓                ↓
main             feature
```

每张桌子都有自己的工作现场，可以独立修改、暂存、提交和运行。它们共享同一个仓库历史，但不会互相弄乱未提交代码。

一句话总结：

> Git Worktree = 给同一个 Git 仓库创建多个独立工作目录，让不同分支可以同时被 checkout、开发和运行。

当脑子里出现“我不想动当前现场，但又必须马上处理另一个分支”时，就该想到 `git worktree`。
