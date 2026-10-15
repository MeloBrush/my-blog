---
title: 使用 Hexo + Butterfly 搭建个人博客
date: 2026-04-20 09:00:00
tags: [Hexo, 博客, Butterfly]
categories: 工具
description: 从零开始使用 Hexo 框架和 Butterfly 主题搭建个人技术博客的完整教程
top_img:
cover:
---

## 前言

搭建个人博客有很多方案，如 WordPress、Typecho、Halo 等。本文介绍如何使用 **Hexo** —— 一个快速、简洁且高效的静态博客框架，配合 **Butterfly** 主题搭建一个美观的个人技术博客。

## 环境准备

### 安装 Node.js

Hexo 基于 Node.js，首先需要安装 Node.js（建议使用 LTS 版本）。

前往 [Node.js 官网](https://nodejs.org/) 下载并安装。

### 安装 Git

Git 用于版本管理和部署博客。

前往 [Git 官网](https://git-scm.com/) 下载并安装。

### 安装 Hexo

打开终端，执行以下命令全局安装 Hexo：

```bash
npm install -g hexo-cli
```

## 搭建博客

### 初始化项目

```bash
hexo init my-blog
cd my-blog
npm install
```

### 本地预览

```bash
hexo server
```

浏览器访问 `http://localhost:4000` 即可看到博客。

## 安装 Butterfly 主题

```bash
npm install hexo-theme-butterfly --save
```

在 `_config.yml` 中设置主题：

```yaml
theme: butterfly
```

## 常用命令

| 命令 | 说明 |
|------|------|
| `hexo new "文章标题"` | 新建文章 |
| `hexo server` | 启动本地服务器 |
| `hexo generate` | 生成静态文件 |
| `hexo clean` | 清除缓存 |
| `hexo deploy` | 部署博客 |

## 写作流程

1. 使用 `hexo new "标题"` 创建新文章
2. 在 `source/_posts/` 目录下编辑 Markdown 文件
3. `hexo server` 本地预览效果
4. `hexo generate` 生成静态文件
5. 部署到 GitHub Pages 或其他平台

## 总结

Hexo + Butterfly 是一个轻量且强大的博客方案。Markdown 写作、静态部署、主题丰富，非常适合技术博客使用。