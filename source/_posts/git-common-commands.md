---
title: Git 常用命令速查手册
date: 2026-04-21 10:30:00
tags: [Git, 版本控制]
categories: 工具
description: 日常开发中最常用的 Git 命令整理，涵盖配置、提交、分支、合并等操作
top_img:
cover:
---

## 基础配置

```bash
# 设置全局用户名
git config --global user.name "你的名字"

# 设置全局邮箱
git config --global user.email "你的邮箱"

# 设置默认编辑器
git config --global core.editor vim

# 查看所有配置
git config --list
```

## 仓库操作

```bash
# 初始化一个新仓库
git init

# 克隆远程仓库
git clone <url>

# 克隆指定分支
git clone -b <branch> <url>
```

## 日常提交流程

```bash
# 查看当前状态
git status

# 添加文件到暂存区
git add <file>
git add .              # 添加所有变更

# 提交到本地仓库
git commit -m "提交说明"

# 推送到远程仓库
git push origin <branch>
```

## 分支管理

```bash
# 查看所有分支
git branch
git branch -a          # 包含远程分支

# 创建并切换分支
git checkout -b <branch>

# 切换分支
git checkout <branch>

# 删除本地分支
git branch -d <branch>

# 删除远程分支
git push origin --delete <branch>
```

## 合并与变基

```bash
# 合并分支（将 feature 合并到当前分支）
git merge <feature>

# 变基（更整洁的提交历史）
git rebase <branch>

# 终止合并
git merge --abort
```

## 撤销操作

```bash
# 撤销工作区的修改
git checkout -- <file>

# 撤销暂存区的文件
git reset HEAD <file>

# 撤销最近一次提交（保留修改）
git reset --soft HEAD~1

# 撤销最近一次提交（丢弃修改）
git reset --hard HEAD~1
```

## 查看历史

```bash
# 查看提交日志
git log
git log --oneline --graph   # 简洁图形模式

# 查看某文件的修改历史
git log -p <file>
```

## 远程仓库

```bash
# 查看远程仓库
git remote -v

# 添加远程仓库
git remote add origin <url>

# 修改远程仓库地址
git remote set-url origin <url>

# 拉取并合并
git pull origin <branch>
```

## 实用技巧

### 临时存储修改

```bash
git stash              # 暂存当前修改
git stash list         # 查看暂存列表
git stash pop          # 恢复最近一次暂存
```

### 查找引入某 Bug 的提交

```bash
git bisect start
git bisect bad         # 标记当前版本有 Bug
git bisect good <commit>  # 标记某个正常版本
```

## 总结

Git 是开发者必备的工具，熟练掌握这些命令能大幅提升日常开发效率。建议将常用命令整理成自己的速查手册，随时翻阅。