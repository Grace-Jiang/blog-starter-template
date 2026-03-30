---
title: Git面试必备 - 分支管理、合并策略与常用命令
description: Git三个区域、分支操作、merge vs rebase、cherry-pick、reset vs revert
pubDate: '2025-02-16'
categories:
- Python
tags:
- Git
- 版本控制
- 面试
---
# Git 面试重点速查

## 核心概念（必考）

### 三个区域
```
工作区 → git add → 暂存区 → git commit → 本地仓库 → git push → 远程仓库
```

### 文件状态
- **Untracked** → **Staged** → **Committed** → **Modified**

## 必会命令

```bash
# 基础操作
git init / git clone <url>
git add . / git commit -m "msg"
git status / git log --oneline --graph

# 分支
git branch <name>              # 创建
git checkout <name>            # 切换
git checkout -b <name>         # 创建并切换
git merge <branch>             # 合并
git branch -d <name>           # 删除

# 远程
git pull                       # 拉取并合并
git push                       # 推送
git fetch                      # 只拉取不合并

# 撤销
git reset --soft HEAD~1        # 移动当前分支指针，保留暂存区和工作区
git reset --hard HEAD~1        # 移动当前分支指针，重置暂存区和工作区
git revert <commit>            # 创建新提交来撤销

# 储藏
git stash                      # 储藏当前修改
git stash pop                  # 恢复储藏
```

## 高频面试题（必背）

### 1. git pull vs git fetch
```bash
git fetch        # 只拉取，不合并
git pull         # = git fetch + git merge
git pull --rebase # = git fetch + git rebase
```

### 2. git reset 三种模式

**原理**：移动当前分支指针（HEAD 跟随分支移动）

```bash
# 初始状态：HEAD → main → C

--soft   # 移动分支指针到 B，保留暂存区和工作区
         # HEAD → main → B（C 的修改在暂存区）

--mixed  # 移动分支指针到 B，重置暂存区，保留工作区（默认）
         # HEAD → main → B（C 的修改在工作区）

--hard   # 移动分支指针到 B，重置暂存区和工作区
         # HEAD → main → B（C 的修改完全丢失）
```

**注意**：正常情况下 `HEAD → 分支 → 提交`（两层引用），reset 移动的是分支指针

### 3. merge vs rebase
```bash
# merge: 保留历史，创建合并提交
git merge feature
#   main: A---B---C---M
#                    /
#   feature:    D---E

# rebase: 线性历史，不创建合并提交
git rebase main
#   main: A---B---C---D'---E'

# 原则：公共分支用 merge，个人分支用 rebase
```

### 4. 如何撤销已 push 的提交？
```bash
# 方法1：revert（推荐，不改变历史）
git revert <commit>
git push

# 方法2：reset + force push（危险，改变历史）
git reset --hard <commit>
git push --force-with-lease
```

### 5. 解决冲突
```bash
# 1. 发现冲突
git merge feature  # CONFLICT

# 2. 手动编辑文件，删除冲突标记
<<<<<<< HEAD
当前分支内容
=======
要合并分支内容
>>>>>>> feature

# 3. 标记已解决
git add <file>
git commit
```

### 6. HEAD、工作区、暂存区
- **HEAD**: 指向当前分支的指针（通常是 `HEAD → 分支 → 提交`）
- **工作区**: 实际文件目录
- **暂存区**: 下次提交的快照（索引）

**特殊情况**：detached HEAD 时，HEAD 直接指向提交（`HEAD → 提交`）

### 7. 合并多个提交
```bash
# 交互式 rebase
git rebase -i HEAD~3
# 将 pick 改为 squash 或 fixup
```

### 8. 找回删除的内容
```bash
git reflog              # 查看所有操作
git reset --hard <hash> # 恢复到某个状态
```

## 工作流（了解）

### Git Flow
```
master (生产) ← release ← develop ← feature
                ↑ hotfix
```

### GitHub Flow（更简单）
```
main ← feature (通过 PR 合并)
```

## .gitignore 常用

```bash
# Python
__pycache__/
*.pyc
.env
venv/

# Node.js
node_modules/
.env

# IDE
.vscode/
.idea/
```

## 配置（常用）

```bash
# 用户信息
git config --global user.name "Name"
git config --global user.email "email@example.com"

# 别名
git config --global alias.st status
git config --global alias.co checkout
git config --global alias.lg "log --oneline --graph --all"

# 默认分支
git config --global init.defaultBranch main
```

## 提交规范

```bash
<type>: <subject>

# type 类型：
feat:     新功能
fix:      修复 bug
docs:     文档
refactor: 重构
test:     测试

# 示例：
feat: add user login
fix: resolve memory leak in cache
```

## 常见场景

### 提交到错误分支
```bash
git reset --soft HEAD~1  # 撤销提交
git checkout correct-branch
git commit -m "msg"
```

### 忘记切换分支
```bash
git stash               # 储藏修改
git checkout correct-branch
git stash pop           # 恢复修改
```

### 修改最后一次提交
```bash
git commit --amend      # 修改提交信息或添加文件
```

### 查看某个文件的历史
```bash
git log -p <file>       # 详细历史
git blame <file>        # 每行的修改者
```

## 面试问答要点

**Q: Git 和 SVN 的区别？**
A: Git 是分布式，每个人都有完整仓库；SVN 是集中式，只有服务器有完整历史。Git 可离线工作，分支操作更快。

**Q: 什么时候用 rebase，什么时候用 merge？**
A: 个人分支更新用 rebase（保持线性历史），合并到主分支用 merge（保留完整历史）。公共分支永远不要 rebase。

**Q: 如何处理大文件？**
A: 使用 Git LFS (Large File Storage)，或者不要提交大文件到 Git。

**Q: detached HEAD 是什么？**
A: 直接 checkout 到某个 commit 而不是分支时的状态。需要创建分支保存工作：`git checkout -b new-branch`

**Q: cherry-pick 的作用？**
A: 将某个提交应用到当前分支：`git cherry-pick <commit>`

## 记忆口诀

- **三区域**：工作区 → 暂存区 → 本地仓库
- **三模式**：soft 软（保留全部）、mixed 混（保留工作区）、hard 硬（全删除）
- **两合并**：merge 保历史、rebase 变线性
- **两拉取**：fetch 只拉、pull 拉合

## 实用技巧

```bash
# 查看简洁日志
git log --oneline --graph --all

# 查看某次提交的改动
git show <commit>

# 搜索提交信息
git log --grep="keyword"

# 临时忽略文件修改
git update-index --assume-unchanged <file>

# 恢复跟踪
git update-index --no-assume-unchanged <file>

# 清理已删除的远程分支
git remote prune origin

# 查看配置
git config --list
```

## 面试准备清单

✅ 能画出三个区域的流转图  
✅ 能解释 reset 三种模式的区别  
✅ 能说出 merge 和 rebase 的使用场景  
✅ 能演示如何解决冲突  
✅ 能解释 pull 和 fetch 的区别  
✅ 知道如何撤销已 push 的提交  
✅ 了解至少一种工作流（Git Flow 或 GitHub Flow）  
✅ 能写出规范的提交信息  

---

**重点中的重点**：三个区域、reset 三种模式、merge vs rebase、冲突解决
