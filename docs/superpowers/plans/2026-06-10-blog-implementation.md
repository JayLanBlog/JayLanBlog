# JayLanBlog 博客站实施计划

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 基于 Astro + Fuwari 主题搭建综合博客，部署到 GitHub Pages，支持数学公式、评论系统和访问统计。

**Architecture:** 使用 Astro 静态站点生成器 + Fuwari 博客主题模板。内容以 Markdown 文件存储在 `src/content/posts/`，通过 GitHub Actions 自动构建并部署到 GitHub Pages。额外集成 KaTeX（数学公式）、Giscus（评论）、Umami（统计）三个功能。

**Tech Stack:** Astro, Fuwari theme, Tailwind CSS, Svelte, Pagefind, Expressive Code, remark-math, rehype-katex, Giscus, Umami, pnpm, GitHub Actions

---

## 文件结构映射

| 文件 | 职责 | 操作 |
|------|------|------|
| `astro.config.mjs` | Astro 核心配置（站点URL、base、插件） | 修改 |
| `src/config.ts` | 博客站点信息、导航、社交链接、Giscus/Umami 配置 | 修改 |
| `src/layouts/PostLayout.astro` | 文章详情页布局（注入 KaTeX CSS、Giscus 组件、Umami 脚本） | 修改 |
| `src/components/GiscusComment.astro` | Giscus 评论组件 | 创建 |
| `src/components/UmamiScript.astro` | Umami 统计脚本组件 | 创建 |
| `src/content/posts/hello-world.md` | 示例文章：技术教程 | 创建 |
| `src/content/posts/my-first-post.md` | 示例文章：生活随笔 | 创建 |
| `.github/workflows/deploy.yml` | GitHub Actions 自动部署工作流 | 创建 |
| `package.json` | 添加 remark-math、rehype-katex 依赖 | 修改 |

---

### Task 1: 基于 Fuwari 模板初始化项目

**Files:**
- Create: 整个项目目录（通过 `pnpm create fuwari`）
- Modify: `e:\AI\bockwork\JayLanBlog\`（将模板内容合并到现有仓库）

- [ ] **Step 1: 在临时目录创建 Fuwari 项目**

```powershell
cd 'c:\Users\86178\.trae-cn\work\6a296902e71cd1e58afe8e41'
pnpm create fuwari@latest jaylanblog-temp
```

Expected: 在 `c:\Users\86178\.trae-cn\work\6a296902e71cd1e58afe8e41\jaylanblog-temp` 生成 Fuwari 模板项目

- [ ] **Step 2: 将模板文件复制到 JayLanBlog 仓库**

```powershell
# 复制所有文件（排除 .git 目录）到 JayLanBlog 仓库
$source = 'c:\Users\86178\.trae-cn\work\6a296902e71cd1e58afe8e41\jaylanblog-temp'
$dest = 'e:\AI\bockwork\JayLanBlog'
# 使用 robocopy 排除 .git 目录
robocopy $source $dest /E /XD .git /XF .git
```

Expected: Fuwari 模板的所有文件（src/、public/、astro.config.mjs 等）出现在 JayLanBlog 目录中

- [ ] **Step 3: 安装依赖**

```powershell
cd 'e:\AI\bockwork\JayLanBlog'
pnpm install
```

Expected: 依赖安装成功，无报错

- [ ] **Step 4: 验证开发服务器启动**

```powershell
cd 'e:\AI\bockwork\JayLanBlog'
pnpm dev
```

Expected: 开发服务器在 `http://localhost:4321` 启动成功，页面可访问

- [ ] **Step 5: 停止开发服务器，提交初始化**

```powershell
cd 'e:\AI\bockwork\JayLanBlog'
git add -A
git commit -m "feat: initialize blog with Fuwari template"
```

---

### Task 2: 配置站点信息

**Files:**
- Modify: `src/config.ts`

- [ ] **Step 1: 编辑 src/config.ts，配置站点基本信息**

打开 `e:\AI\bockwork\JayLanBlog\src\config.ts`，修改以下字段：

```typescript
// 站点标题和描述
const title = "JayLanBlog";
const titleSvg = `<svg xmlns="http://www.w3.org/2000/svg" width="24" height="24" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"><path d="M12 20h9"/><path d="M16.5 3.5a2.121 2.121 0 0 1 3 3L7 19l-4 1 1-4L16.5 3.5z"/></svg>`;
const description = "图形学、AI 渲染、自动驾驶技术博客";
const author = {
  name: "Jaylan",
  avatar: "./avatar.png",
  url: "https://github.com/JayLanBlog",
};

// 社交链接
const socialLinks = [
  { name: "GitHub", href: "https://github.com/JayLanBlog" },
  { name: "Email", href: "mailto:your-email@example.com" },
];

// 导航菜单
const navItems = [
  { label: "首页", href: "/" },
  { label: "文章", href: "/posts" },
  { label: "项目", href: "/projects" },
  { label: "归档", href: "/archive" },
];
```

注意：具体字段名和结构以 Fuwari 模板实际的 `config.ts` 为准。上述为预期结构，实际执行时需对照模板源码调整字段名。

- [ ] **Step 2: 启动开发服务器验证配置生效**

```powershell
cd 'e:\AI\bockwork\JayLanBlog'
pnpm dev
```

Expected: 浏览器访问 `http://localhost:4321`，页面标题显示 "JayLanBlog"，导航菜单包含首页、文章、项目、归档

- [ ] **Step 3: 停止开发服务器，提交**

```powershell
cd 'e:\AI\bockwork\JayLanBlog'
git add src/config.ts
git commit -m "feat: configure site info, navigation and social links"
```

---

### Task 3: 配置 Astro 站点 URL 和 base 路径

**Files:**
- Modify: `astro.config.mjs`

- [ ] **Step 1: 编辑 astro.config.mjs，设置 site 和 base**

打开 `e:\AI\bockwork\JayLanBlog\astro.config.mjs`，修改：

```javascript
export default defineConfig({
  site: "https://jaylanblog.github.io",
  base: "/JayLanBlog",
  // ... 其余配置保持不变
});
```

说明：`base` 设置为 `/JayLanBlog` 是因为仓库名为 `JayLanBlog`，GitHub Pages 的项目站点路径会包含仓库名。如果后续配置自定义域名（如 `jaylan.blog`），需将 `base` 改为 `/`，`site` 改为自定义域名。

- [ ] **Step 2: 验证构建正常**

```powershell
cd 'e:\AI\bockwork\JayLanBlog'
pnpm build
```

Expected: 构建成功，`dist/` 目录生成静态文件，资源路径包含 `/JayLanBlog/` 前缀

- [ ] **Step 3: 提交**

```powershell
cd 'e:\AI\bockwork\JayLanBlog'
git add astro.config.mjs
git commit -m "feat: configure site URL and base path for GitHub Pages"
```

---

### Task 4: 集成数学公式支持（KaTeX）

**Files:**
- Modify: `package.json`（添加依赖）
- Modify: `astro.config.mjs`（添加插件配置）
- Modify: `src/layouts/PostLayout.astro`（引入 KaTeX CSS）

- [ ] **Step 1: 安装 remark-math 和 rehype-katex**

```powershell
cd 'e:\AI\bockwork\JayLanBlog'
pnpm add remark-math rehype-katex
```

Expected: 依赖安装成功

- [ ] **Step 2: 在 astro.config.mjs 中配置插件**

在 `astro.config.mjs` 的 `defineConfig` 中添加 remark 和 rehype 插件：

```javascript
import remarkMath from "remark-math";
import rehypeKatex from "rehype-katex";

export default defineConfig({
  // ... 已有配置
  markdown: {
    remarkPlugins: [remarkMath],
    rehypePlugins: [rehypeKatex],
  },
});
```

注意：如果 Fuwari 模板已有 remarkPlugins/rehypePlugins 配置，需合并而非覆盖。

- [ ] **Step 3: 在文章布局中引入 KaTeX CSS**

在 `src/layouts/PostLayout.astro`（或 Fuwari 实际使用的文章布局文件）的 `<head>` 中添加：

```html
<link rel="stylesheet" href="https://cdn.jsdelivr.net/npm/katex@0.16.11/dist/katex.min.css" />
```

注意：需确认 Fuwari 的文章布局文件实际路径（可能是 `src/layouts/PostLayout.astro` 或其他名称），执行时以实际文件为准。

- [ ] **Step 4: 创建测试文章验证数学公式**

创建 `src/content/posts/math-test.md`：

```markdown
---
title: "数学公式测试"
published: 2026-06-10
description: "测试 KaTeX 数学公式渲染"
tags: [测试]
category: 测试
draft: true
---

## 行内公式

质能方程 $E = mc^2$ 是物理学中最著名的公式之一。

## 块级公式

$$
\int_{-\infty}^{\infty} e^{-x^2} dx = \sqrt{\pi}
$$

## 矩阵

$$
\begin{bmatrix}
a & b \\
c & d
\end{bmatrix}
\begin{bmatrix}
x \\
y
\end{bmatrix}
=
\begin{bmatrix}
ax + by \\
cx + dy
\end{bmatrix}
$$
```

- [ ] **Step 5: 启动开发服务器验证**

```powershell
cd 'e:\AI\bockwork\JayLanBlog'
pnpm dev
```

Expected: 访问测试文章页面，行内公式和块级公式均正确渲染为 KaTeX 格式

- [ ] **Step 6: 删除测试文章，提交**

```powershell
cd 'e:\AI\bockwork\JayLanBlog'
Remove-Item 'src\content\posts\math-test.md'
git add -A
git commit -m "feat: integrate KaTeX math formula support"
```

---

### Task 5: 集成评论系统（Giscus）

**Files:**
- Create: `src/components/GiscusComment.astro`
- Modify: `src/config.ts`（添加 Giscus 配置项）
- Modify: 文章布局文件（嵌入评论组件）

- [ ] **Step 1: 在 src/config.ts 中添加 Giscus 配置**

在 `src/config.ts` 中添加（使用占位值，用户后续自行替换）：

```typescript
// Giscus 评论系统配置
// 请访问 https://giscus.app 生成实际配置值
const giscusConfig = {
  repo: "JayLanBlog/JayLanBlog",           // GitHub 仓库
  repoId: "",                                // 仓库 ID（从 giscus.app 获取）
  category: "Announcements",                // Discussion 分类
  categoryId: "",                            // 分类 ID（从 giscus.app 获取）
  mapping: "pathname",                       // 页面与 Discussion 的映射方式
  reactionsEnabled: "1",                     // 启用 Reactions
  emitMetadata: "0",                         // 不发送元数据
  inputPosition: "top",                      // 评论框位置
  lang: "zh-CN",                             // 语言
};
```

注意：字段名和结构以 Fuwari 模板实际的 config.ts 导出方式为准。如果 Fuwari 已有 Giscus 配置结构，直接填入值即可。

- [ ] **Step 2: 创建 GiscusComment 组件**

创建 `src/components/GiscusComment.astro`：

```astro
---
// 从 config 导入 Giscus 配置
// 注意：实际 import 路径以 Fuwari 的 config 导出方式为准
import { giscusConfig } from "../config";
---

<div id="giscus-container" class="giscus"></div>

<script
  src="https://giscus.app/client.js"
  data-repo={giscusConfig.repo}
  data-repo-id={giscusConfig.repoId}
  data-category={giscusConfig.category}
  data-category-id={giscusConfig.categoryId}
  data-mapping={giscusConfig.mapping}
  data-reactions-enabled={giscusConfig.reactionsEnabled}
  data-emit-metadata={giscusConfig.emitMetadata}
  data-input-position={giscusConfig.inputPosition}
  data-theme="preferred_color_scheme"
  data-lang={giscusConfig.lang}
  data-loading="lazy"
  async
></script>
```

注意：需确认 Fuwari 是否已有 Giscus 集成。如果 Fuwari 已内置评论系统支持，则此步骤应改为启用内置功能而非创建新组件。

- [ ] **Step 3: 在文章布局中嵌入评论组件**

在文章布局文件（PostLayout 或等效文件）的文章内容末尾、页脚之前添加：

```astro
---
import GiscusComment from "../components/GiscusComment.astro";
---

<!-- 文章内容之后 -->
<GiscusComment />
```

- [ ] **Step 4: 提交**

```powershell
cd 'e:\AI\bockwork\JayLanBlog'
git add -A
git commit -m "feat: integrate Giscus comment system"
```

---

### Task 6: 集成访问统计（Umami）

**Files:**
- Create: `src/components/UmamiScript.astro`
- Modify: `src/config.ts`（添加 Umami 配置项）
- Modify: 基础布局文件（注入脚本）

- [ ] **Step 1: 在 src/config.ts 中添加 Umami 配置**

```typescript
// Umami 访问统计配置
// 请访问 https://umami.is 注册账户获取实际值
const umamiConfig = {
  websiteId: "",           // Umami 网站 ID（注册后获取）
  src: "https://analytics.umami.is/script.js",  // Umami 脚本地址（自托管则改为自己的地址）
};
```

- [ ] **Step 2: 创建 UmamiScript 组件**

创建 `src/components/UmamiScript.astro`：

```astro
---
import { umamiConfig } from "../config";
---

{umamiConfig.websiteId && (
  <script
    async
    src={umamiConfig.src}
    data-website-id={umamiConfig.websiteId}
  ></script>
)}
```

- [ ] **Step 3: 在基础布局中注入 Umami 脚本**

在基础布局文件（Layout.astro 或等效的根布局）的 `<head>` 中添加：

```astro
---
import UmamiScript from "../components/UmamiScript.astro";
---

<head>
  <!-- 已有内容 -->
  <UmamiScript />
</head>
```

- [ ] **Step 4: 提交**

```powershell
cd 'e:\AI\bockwork\JayLanBlog'
git add -A
git commit -m "feat: integrate Umami analytics"
```

---

### Task 7: 创建示例文章

**Files:**
- Create: `src/content/posts/hello-world.md`
- Create: `src/content/posts/my-first-post.md`

- [ ] **Step 1: 创建技术教程示例文章**

创建 `src/content/posts/hello-world.md`：

```markdown
---
title: "Hello World — 博客开篇"
published: 2026-06-10
description: "JayLanBlog 博客正式上线！这里将分享图形学、AI 渲染、自动驾驶等技术内容。"
image: ""
tags: [博客, 开篇]
category: 随笔
draft: false
---

## 欢迎来到 JayLanBlog

这是我的技术博客的第一篇文章。在这里，我将分享以下方向的内容：

- **图形学与渲染**：OpenGL、Vulkan、实时渲染技术
- **AI 与视觉**：PyTorch、可微渲染、NeRF
- **自动驾驶**：感知、规划、仿真

## 关于本博客

本博客基于 [Astro](https://astro.build/) 静态站点生成器搭建，使用 [Fuwari](https://github.com/saicaca/fuwari) 主题，部署在 GitHub Pages。

文章使用 Markdown 编写，支持代码高亮和数学公式。

### 代码高亮示例

```cpp
#include <iostream>
int main() {
    std::cout << "Hello, Graphics World!" << std::endl;
    return 0;
}
```

### 数学公式示例

渲染方程：

$$
L_o(p, \omega_o) = L_e(p, \omega_o) + \int_{\Omega^+} f_r(p, \omega_i, \omega_o) L_i(p, \omega_i) (n \cdot \omega_i) d\omega_i
$$

## 期待你的关注

如果你对图形学、AI 渲染或自动驾驶感兴趣，欢迎关注我的博客！
```

- [ ] **Step 2: 创建生活随笔示例文章**

创建 `src/content/posts/my-first-post.md`：

```markdown
---
title: "搭建博客的旅程"
published: 2026-06-10
description: "记录从零搭建这个博客的过程，以及选择 Astro + Fuwari 的原因。"
image: ""
tags: [博客, 技术]
category: 随笔
draft: false
---

## 为什么搭建博客

作为一个技术人，拥有一自己的博客一直是我的目标。博客不仅是记录学习笔记的地方，也是与社区交流的窗口。

## 技术选型过程

在选择博客框架时，我对比了以下方案：

| 方案 | 优点 | 缺点 |
|------|------|------|
| Astro + Fuwari | 性能极佳，UI 精美 | 生态相对较小 |
| Next.js | React 生态完整 | 对纯博客偏重 |
| Hexo | 中文生态成熟 | 技术栈较老 |

最终选择了 **Astro + Fuwari**，因为：

1. 零 JS 默认加载，访问速度极快
2. Fuwari 主题开箱即用，功能全面
3. 支持数学公式渲染，适合技术文章
4. GitHub Pages 免费部署，成本低

## 搭建过程

整个过程非常顺利：

1. 使用 `pnpm create fuwari@latest` 初始化项目
2. 修改配置文件，设置站点信息
3. 集成 KaTeX、Giscus、Umami
4. 配置 GitHub Actions 自动部署
5. 推送到 GitHub，博客上线！

如果你也想搭建自己的博客，希望这篇文章能帮到你。
```

- [ ] **Step 3: 启动开发服务器验证文章显示**

```powershell
cd 'e:\AI\bockwork\JayLanBlog'
pnpm dev
```

Expected: 首页显示两篇文章卡片，点击可进入文章详情页，代码高亮和数学公式渲染正常

- [ ] **Step 4: 提交**

```powershell
cd 'e:\AI\bockwork\JayLanBlog'
git add src/content/posts/
git commit -m "feat: add sample blog posts"
```

---

### Task 8: 配置 GitHub Actions 自动部署

**Files:**
- Create: `.github/workflows/deploy.yml`

- [ ] **Step 1: 创建 GitHub Actions 工作流文件**

创建 `.github/workflows/deploy.yml`：

```yaml
name: Deploy to GitHub Pages

on:
  push:
    branches: [main]
  workflow_dispatch:

permissions:
  contents: read
  pages: write
  id-token: write

concurrency:
  group: "pages"
  cancel-in-progress: true

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

- [ ] **Step 2: 验证构建流程（本地模拟）**

```powershell
cd 'e:\AI\bockwork\JayLanBlog'
pnpm build
```

Expected: 构建成功，`dist/` 目录包含完整的静态站点文件

- [ ] **Step 3: 本地预览构建结果**

```powershell
cd 'e:\AI\bockwork\JayLanBlog'
pnpm preview
```

Expected: 预览服务器启动，页面可正常访问，资源路径正确（包含 `/JayLanBlog/` 前缀）

- [ ] **Step 4: 提交**

```powershell
cd 'e:\AI\bockwork\JayLanBlog'
git add .github/workflows/deploy.yml
git commit -m "feat: add GitHub Actions deployment workflow"
```

---

### Task 9: 最终验证与推送

**Files:**
- 无新文件，验证所有功能

- [ ] **Step 1: 完整构建验证**

```powershell
cd 'e:\AI\bockwork\JayLanBlog'
pnpm build
```

Expected: 构建成功，无错误和警告

- [ ] **Step 2: 检查所有提交历史**

```powershell
cd 'e:\AI\bockwork\JayLanBlog'
git log --oneline
```

Expected: 看到以下提交记录（按顺序）：
1. `feat: initialize blog with Fuwari template`
2. `feat: configure site info, navigation and social links`
3. `feat: configure site URL and base path for GitHub Pages`
4. `feat: integrate KaTeX math formula support`
5. `feat: integrate Giscus comment system`
6. `feat: integrate Umami analytics`
7. `feat: add sample blog posts`
8. `feat: add GitHub Actions deployment workflow`

- [ ] **Step 3: 推送到 GitHub**

```powershell
cd 'e:\AI\bockwork\JayLanBlog'
git push origin main
```

Expected: 推送成功

- [ ] **Step 4: 在 GitHub 仓库设置中启用 GitHub Pages**

1. 进入 GitHub 仓库 → Settings → Pages
2. Source 选择 "GitHub Actions"
3. 保存设置

Expected: GitHub Actions 工作流自动触发，构建并部署

- [ ] **Step 5: 验证线上效果**

等待 GitHub Actions 完成后，访问 `https://jaylanblog.github.io/JayLanBlog/`

Expected:
- 首页正常显示，标题为 "JayLanBlog"
- 导航菜单包含首页、文章、项目、归档
- 示例文章可正常访问
- 代码高亮和数学公式渲染正常
- 暗色模式切换正常
- 搜索功能可用
