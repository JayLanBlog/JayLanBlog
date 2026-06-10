# JayLanBlog 博客站设计文档

> 日期：2026-06-10
> 状态：待用户审阅

## 1. 概述

### 1.1 项目目标

基于 `JayLanBlog/JayLanBlog` GitHub 仓库搭建一个综合博客网站，涵盖技术教程、生活随笔、项目展示等多种内容。博客面向技术社区，作者背景为图形学/AI 渲染/自动驾驶方向。

### 1.2 约束条件

- 部署在 GitHub Pages（免费托管）
- 内容管理方式：Markdown + Git（无 CMS 后台）
- 仓库已有 GitHub Profile README，需保留或整合
- 作者熟悉前端开发（React/Vue/HTML）

### 1.3 成功标准

- 博客可通过 GitHub Pages 正常访问
- 支持分类/标签/归档功能
- 支持 Markdown 写作，代码高亮和数学公式渲染正常
- 评论系统和访问统计功能可用
- GitHub Actions 自动部署流程正常工作

---

## 2. 技术选型

### 2.1 核心技术栈

| 层级 | 选型 | 理由 |
|------|------|------|
| 静态站点生成器 | Astro | 零 JS 默认加载，性能最优；原生支持 MDX；支持 React/Svelte 组件混用 |
| 博客主题 | Fuwari | 功能全面（搜索、暗色模式、RSS、响应式）；UI 精美现代；社区活跃 |
| 样式框架 | Tailwind CSS | Fuwari 内置，实用优先的 CSS 框架 |
| 组件框架 | Svelte | Fuwari 默认组件框架，轻量高效 |
| 搜索引擎 | Pagefind | Fuwari 内置，静态全文搜索，无需后端 |
| 代码高亮 | Expressive Code | Fuwari 内置，支持语法高亮、文件名、行高亮、diff 等 |
| 包管理器 | pnpm | Fuwari 官方推荐 |

### 2.2 需额外集成的功能

| 功能 | 选型 | 理由 |
|------|------|------|
| 数学公式 | remark-math + rehype-katex | Astro 官方推荐，KaTeX 渲染速度快 |
| 评论系统 | Giscus | 基于 GitHub Discussions，免费、无需数据库、支持 Markdown |
| 访问统计 | Umami | 开源、隐私友好、支持自托管或云服务 |

---

## 3. 架构设计

### 3.1 项目结构

```
JayLanBlog/
├── .github/
│   └── workflows/
│       └── deploy.yml           # GitHub Actions 自动部署
├── docs/
│   └── superpowers/specs/        # 设计文档
├── public/
│   ├── favicon/                  # 网站图标
│   └── assets/                   # 静态资源（图片等）
├── src/
│   ├── components/               # Svelte 组件
│   │   ├── Card.ts               # 文章卡片
│   │   ├── Navbar.ts             # 导航栏
│   │   ├── Footer.ts             # 页脚
│   │   └── ...
│   ├── content/
│   │   ├── posts/                # 博客文章 (Markdown/MDX)
│   │   │   └── *.md
│   │   └── projects/             # 项目展示页（可选）
│   ├── config.ts                 # 博客核心配置
│   ├── constants.ts              # 常量定义
│   ├── layouts/
│   │   ├── Layout.ts             # 基础布局
│   │   └── PostLayout.ts         # 文章布局
│   ├── pages/
│   │   ├── index.astro           # 首页
│   │   ├── posts/
│   │   │   ├── [slug].astro      # 文章详情页
│   │   │   └── index.astro       # 文章列表
│   │   ├── archive.astro         # 归档页
│   │   └── projects.astro        # 项目展示页
│   └── styles/
│       └── global.css           # 全局样式
├── astro.config.mjs              # Astro 配置（站点URL、集成等）
├── tailwind.config.cjs            # Tailwind 配置
├── tsconfig.json                 # TypeScript 配置
├── package.json                   # 依赖管理
└── pnpm-lock.yaml                 # 锁文件
```

### 3.2 内容管理流程

```
作者编写 Markdown 文章
  → 放入 src/content/posts/ 目录
  → Git commit + push 到 main 分支
  → GitHub Actions 触发构建
  → Astro 生成静态 HTML 到 dist/
  → 自动部署到 GitHub Pages
  → 博客上线
```

### 3.3 文章 Frontmatter 规范

```yaml
---
title: "文章标题"
published: 2026-06-10
description: "文章简短描述（用于 SEO 和列表展示）"
image: ./cover.jpg          # 封面图片（可选）
tags: [图形学, OpenGL, 渲染]
category: 技术教程           # 文章分类
draft: false                # 是否为草稿
lang: en                    # 仅当文章语言与站点默认语言不同时设置
---
```

---

## 4. 功能设计

### 4.1 开箱即用功能（Fuwari 内置）

| 功能 | 说明 |
|------|------|
| 文章列表与详情 | 首页展示文章卡片，点击进入详情 |
| 分类/标签/归档 | 通过 frontmatter 定义，自动生成分类页、标签页、时间线归档 |
| 代码高亮 | Expressive Code 引擎，支持多语言语法高亮 |
| 全文搜索 | Pagefind 静态搜索索引，客户端零依赖 |
| 暗色模式 | Light/Dark 主题切换，跟随系统或手动切换 |
| RSS 订阅 | 自动生成 RSS feed |
| 响应式设计 | 移动端、平板、桌面端自适应 |
| 页面过渡动画 | Astro View Transitions 平滑切换 |
| Markdown 扩展语法 | Admonitions（提示框）、GitHub 仓库卡片等 |

### 4.2 需额外配置的功能

#### 4.2.1 数学公式（KaTeX）

- 安装依赖：`remark-math`、`rehype-katex`
- 在 `astro.config.mjs` 中配置 remark/rehype 插件
- 在页面中引入 KaTeX CSS
- 支持行内公式 `$...$` 和块级公式 `$$...$$`

#### 4.2.2 评论系统（Giscus）

- 在 `src/config.ts` 中添加 Giscus 配置项
- 创建 GiscusComments 组件，嵌入文章详情页
- 需要用户提前配置 GitHub App（通过 giscus.app）
- 配置项：repo、repoId、category、categoryId

#### 4.2.3 访问统计（Umami）

- 注册 Umami 账户（自托管或云服务）
- 获取 tracking script URL 和 website ID
- 在布局组件中注入 Umami 跟踪脚本
- 配置为 `async` 加载，不影响页面性能

---

## 5. 部署方案

### 5.1 GitHub Pages 配置

- 站点输出为纯静态文件（Astro `static` 模式）
- 部署目标：`gh-pages` 分支 或 GitHub Actions artifact
- 站点 URL：`https://jaylanblog.github.io/JayLanBlog/`
- `astro.config.mjs` 中设置 `site` 和 `base` 路径

### 5.2 GitHub Actions 工作流

```yaml
# .github/workflows/deploy.yml
name: Deploy to GitHub Pages
on:
  push:
    branches: [main]
  workflow_dispatch:
permissions:
  contents: read
  pages: write
  id-token: write
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: pnpm/action-setup@v4
        with:
          version: 9
      - uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: pnpm
      - run: pnpm install
      - run: pnpm build
      - uses: actions/upload-pages-artifact@v3
        with:
          path: dist/
  deploy:
    needs: build
    runs-on: ubuntu-latest
    environment:
      name: github-pages
      url: ${{ steps.deployment.outputs.page_url }}
    steps:
      - id: deployment
        uses: actions/deploy-pages@v4
```

---

## 6. 定制化计划

### 6.1 站点配置（`src/config.ts`）

- 博客标题：JayLanBlog
- 博客描述：图形学、AI 渲染、自动驾驶技术博客
- 作者头像、社交链接（GitHub、Email、Twitter）
- 导航菜单项：首页、文章、项目、归档
- 主题色和 banner 图片

### 6.2 个人信息整合

- 保留原 README.md 中的个人信息（技能、项目列表）
- 可在博客中创建「关于」页面，整合 Profile 信息
- 项目展示页可引用 README 中的精选项目

---

## 7. 实施步骤概览

1. 基于 Fuwari 模板初始化项目（`pnpm create fuwari@latest`）
2. 配置 `src/config.ts`（站点信息、导航、社交链接）
3. 配置 `astro.config.mjs`（站点 URL、base 路径）
4. 集成数学公式支持（remark-math + rehype-katex）
5. 集成评论系统（Giscus 组件）
6. 集成访问统计（Umami 脚本）
7. 创建示例文章（技术教程、生活随笔各一篇）
8. 配置 GitHub Actions 自动部署
9. 测试构建和部署流程
10. 推送到 GitHub，验证线上效果
