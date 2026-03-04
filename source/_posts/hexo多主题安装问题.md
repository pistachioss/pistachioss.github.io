---
title: Hexo多主题完整部署指南
date: 2026-03-01
categories:  Hexo
tags: wiki
toc: true
description: "Hexo 博客 + Wiki 双主题完整部署指南"

---

# Hexo 博客 + Wiki 双主题完整部署指南

> 本文档记录了如何为 Hexo 博客添加 Wiki 功能，使用两个独立主题（Freemind + Wixo），实现类似 wzpan 博客的效果。
> 
> **最终效果**：
> - 博客：https://blog.xiangyang.xyz （Freemind 主题）
> - Wiki：https://blog.xiangyang.xyz/wiki （Wixo 主题）
> - 自动部署：推送任一仓库，两个站点同步更新

---

## 📋 目录

1. [项目背景](#项目背景)
2. [技术方案](#技术方案)
3. [完整实施步骤](#完整实施步骤)
4. [常见问题与解决方案](#常见问题与解决方案)
5. [日常使用流程](#日常使用流程)
6. [高级配置](#高级配置)

---

## 项目背景

### 需求

参考 wzpan 的博客（https://www.hahack.com），实现：
- **博客**：时间线组织，用于发表文章和想法
- **Wiki**：主题组织，用于系统化的知识管理
- **统一域名**：通过子路径访问（/wiki）
- **不同主题**：博客和 Wiki 使用不同的视觉风格

### 技术挑战

Hexo 原生只支持**单主题**，无法在同一个项目中使用多个主题。

### 解决方案

**双项目 + GitHub Actions 自动合并部署**

```
博客项目（Freemind）  ─┐
                      ├─→ GitHub Actions ─→ 合并 ─→ 部署
Wiki 项目（Wixo）    ─┘
```

---

## 技术方案

### 方案架构

```
本地开发
├── myblog/                    # 博客项目
│   ├── themes/freemind/      # Freemind 主题
│   └── source/_posts/        # 博客文章
│
└── myblog-wiki/               # Wiki 项目
    ├── themes/wixo/          # Wixo 主题
    └── source/_posts/        # Wiki 笔记

GitHub
├── pistachioss.github.io     # 博客仓库
└── pistachioss-wiki          # Wiki 仓库

部署
└── gh-pages 分支
    ├── /                     # 博客内容（来自 myblog）
    └── /wiki/                # Wiki 内容（来自 myblog-wiki）

访问
├── https://blog.xiangyang.xyz/          # 博客
└── https://blog.xiangyang.xyz/wiki/     # Wiki
```

### 核心技术点

1. **双仓库管理**：博客和 Wiki 各自独立
2. **GitHub Actions**：自动构建和合并
3. **Repository Dispatch**：Wiki 更新触发博客部署
4. **路径配置**：Wiki 的 root 设置为 /wiki/

---

## 完整实施步骤

### 前置条件

- ✅ 已有 Hexo 博客项目
- ✅ 博客使用 GitHub Pages 部署
- ✅ 熟悉基本的 Git 操作

### 步骤 1: 创建 Wiki 项目

#### 1.1 初始化项目

```bash
# 在博客项目同级目录创建 Wiki 项目
cd D:\web
mkdir myblog-wiki
cd myblog-wiki

# 初始化 Hexo
npm install hexo-cli -g
hexo init .
npm install
```

#### 1.2 安装 Wixo 主题

```bash
cd D:\web\myblog-wiki

# 克隆主题（作为普通文件，不使用 submodule）
git clone https://github.com/wzpan/hexo-theme-wixo.git themes/wixo
```

**⚠️ 重要提示**：
- 不要使用 `git submodule add`，直接克隆即可
- 后续会将主题文件直接提交到仓库

#### 1.3 配置 Wiki 项目

创建 `D:\web\myblog-wiki\_config.yml`：

```yaml
# Hexo Configuration - Wiki Project
# Docs: https://hexo.io/docs/configuration.html

# ==================== 站点信息 ====================
title: 向阳的知识库
subtitle: Wiki & Notes
description: 个人笔记和技术文档
keywords: 编程,笔记,文档,Wiki
author: shineXGO
language: zh-CN
timezone: 'Asia/Shanghai'

# ==================== URL 配置（重要！）====================
url: https://blog.xiangyang.xyz/wiki
root: /wiki/                    # ← 关键配置：告诉 Hexo 部署在 /wiki 子路径
permalink: :category/:title/
permalink_defaults:
pretty_urls:
  trailing_index: true
  trailing_html: true

# ==================== 目录配置 ====================
source_dir: source
public_dir: public
tag_dir: tags
archive_dir: archives
category_dir: categories
code_dir: downloads/code
i18n_dir: :lang
skip_render:

# ==================== 写作配置 ====================
new_post_name: :title.md
default_layout: post
titlecase: false
external_link:
  enable: true
  field: site
  exclude: ''
filename_case: 0
render_drafts: false
post_asset_folder: false
relative_link: false
future: true
syntax_highlighter: highlight.js
highlight:
  line_number: true
  auto_detect: false
  tab_replace: ''
  wrap: true
  hljs: false
prismjs:
  preprocess: true
  line_number: true
  tab_replace: ''

# ==================== 首页设置 ====================
index_generator:
  path: ''
  per_page: 10
  order_by: -date

# ==================== 分类和标签 ====================
default_category: uncategorized
category_map:
tag_map:

# ==================== 元数据 ====================
meta_generator: true

# ==================== 日期格式 ====================
date_format: YYYY-MM-DD
time_format: HH:mm:ss
updated_option: 'mtime'

# ==================== 分页 ====================
per_page: 10
pagination_dir: page

# ==================== 包含/排除文件 ====================
include:
exclude:
ignore:

# ==================== 主题 ====================
theme: wixo

# ==================== 部署（由主博客的 GitHub Actions 处理）====================
deploy:
  type: ''
```

#### 1.4 配置 Wixo 主题

创建或编辑 `D:\web\myblog-wiki\themes\wixo\_config.yml`：

```yaml
# Wixo 主题配置

# 网站标语
slogan: "Writing codes & growing tao."

# 颜色主题（Bootstrap 主题）
theme: cerulean  # 可选: flatly, journal, simplex, spacelab 等
inverse: true    # 深色导航栏

# 菜单配置
menu:
  - title: 首页
    url: /wiki
    intro: "Wiki 首页"
    icon: "fa fa-home"
  - title: 归档
    url: /wiki/archives
    intro: "所有 Wiki 文章"
    icon: "fa fa-archive"
  - title: 分类
    url: /wiki/categories
    intro: "所有分类"
    icon: "fa fa-folder"
  - title: 标签
    url: /wiki/tags
    intro: "所有标签"
    icon: "fa fa-tags"
  - title: 博客
    url: /
    intro: "返回博客"
    icon: "fa fa-book"
  - title: 关于
    url: /about
    intro: "关于我"
    icon: "fa fa-user"

# 侧边栏组件
widgets:
  - category
  - tag
  - recent_posts
  - archive

# 摘要链接
excerpt_link: 阅读全文...

# 社交链接
social:
  GitHub: https://github.com/pistachioss

# Fancybox
fancybox: true

# RSS
rss: atom.xml

# 评论系统（可选，使用 Gitalk）
# comment_gitalk:
#   enable: true
#   client_id: "your_client_id"
#   client_secret: "your_client_secret"
#   repo: "pistachioss-wiki"
#   owner: "pistachioss"
#   admin: ["pistachioss"]
#   language: zh-CN
```

#### 1.5 创建测试文章

```bash
cd D:\web\myblog-wiki
hexo new "getting-started"
```

编辑 `source/_posts/getting-started.md`：

```markdown
---
title: 欢迎来到知识库
date: 2026-03-04 12:00:00
categories:
  - 使用指南
tags:
  - Wiki
  - 入门
---

欢迎来到我的知识库！


## 什么是 Wiki

这里是我的个人知识库，用于记录：

- 📚 技术笔记
- 💡 学习心得
- 📝 项目文档
- 🛠️ 工具使用指南

## 目录组织

知识库按照以下方式组织：

- **编程** - 各种编程语言和框架的笔记
- **工具** - 开发工具使用技巧
- **算法** - 算法学习笔记
- **其他** - 其他杂项

## 与博客的区别

- **博客**：时间线组织，记录想法和经验
- **Wiki**：主题组织，系统化的知识管理

欢迎探索！
```

#### 1.6 本地测试

```bash
cd D:\web\myblog-wiki
hexo clean
hexo server -p 4001

# 访问 http://localhost:4001
# 确认页面样式正常，文章显示正确
```

---

### 步骤 2: 初始化 Wiki 的 Git 仓库

#### 2.1 在 GitHub 创建新仓库

1. 访问 https://github.com/new
2. Repository name: `pistachioss-wiki`
3. Description: `Wiki for my blog`
4. Visibility: **Public**（推荐，避免权限问题）
5. **不要**勾选任何初始化选项
6. 点击 "Create repository"

#### 2.2 本地初始化并推送

```bash
cd D:\web\myblog-wiki

# 初始化 Git
git init

# 创建 .gitignore
echo node_modules/ > .gitignore
echo .DS_Store >> .gitignore
echo *.log >> .gitignore
echo public/ >> .gitignore
echo .deploy_git/ >> .gitignore
echo db.json >> .gitignore

# 添加所有文件（包括主题）
git add .

# 提交
git commit -m "Initial wiki setup with wixo theme"

# 关联远程仓库
git remote add origin https://github.com/pistachioss/pistachioss-wiki.git

# 设置默认分支为 main
git branch -M main

# 推送
git push -u origin main
```

**⚠️ 重要**：确保 `themes/wixo` 目录的所有文件都被提交了。

---

### 步骤 3: 修改博客的 GitHub Actions

#### 3.1 修改部署 Workflow

编辑 `D:\web\myblog\.github\workflows\deploy.yml`，**完全替换**为：

```yaml
name: Build and Deploy Blog with Wiki

on:
  push:
    branches:
      - main
  repository_dispatch:       # ← 新增：允许 Wiki 触发
    types: [wiki_updated]    # ← 新增：监听 wiki_updated 事件
  workflow_dispatch:

jobs:
  build-and-deploy:
    runs-on: ubuntu-latest
    permissions:
      contents: write

    steps:
      # ========================================
      # 步骤 1: 检出博客仓库
      # ========================================
      - name: Checkout Blog Repository
        uses: actions/checkout@v4
        with:
          path: blog
          fetch-depth: 0

      # ========================================
      # 步骤 2: 检出 Wiki 仓库
      # ========================================
      - name: Checkout Wiki Repository
        uses: actions/checkout@v4
        with:
          repository: pistachioss/pistachioss-wiki  # ← 你的 Wiki 仓库
          token: ${{ secrets.GITHUB_TOKEN }}
          path: wiki
          fetch-depth: 0

      # ========================================
      # 步骤 3: 设置 Node.js 环境
      # ========================================
      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'
          cache-dependency-path: |
            blog/package-lock.json
            wiki/package-lock.json

      # ========================================
      # 步骤 4: 缓存依赖（加速构建）
      # ========================================
      - name: Cache Hexo
        uses: actions/cache@v4
        with:
          path: |
            blog/node_modules
            blog/db.json
            wiki/node_modules
            wiki/db.json
          key: ${{ runner.os }}-hexo-${{ hashFiles('**/package-lock.json') }}
          restore-keys: |
            ${{ runner.os }}-hexo-

      # ========================================
      # 步骤 5: 安装 Hexo CLI
      # ========================================
      - name: Install Hexo CLI
        run: npm install -g hexo-cli

      # ========================================
      # 步骤 6: 构建博客
      # ========================================
      - name: Build Blog
        run: |
          cd blog
          npm install
          npx hexo clean
          npx hexo generate
          echo "✅ Blog built successfully"

      # ========================================
      # 步骤 7: 构建 Wiki
      # ========================================
      - name: Build Wiki
        run: |
          cd wiki
          npm install
          npx hexo clean
          npx hexo generate
          echo "✅ Wiki built successfully"

      # ========================================
      # 步骤 8: 合并博客和 Wiki 的输出
      # ========================================
      - name: Combine Blog and Wiki
        run: |
          # 创建最终输出目录
          mkdir -p final-output
          
          # 复制博客内容到根目录
          echo "📦 Copying blog content..."
          cp -r blog/public/* final-output/
          
          # 创建 wiki 子目录并复制 Wiki 内容
          echo "📦 Copying wiki content..."
          mkdir -p final-output/wiki
          cp -r wiki/public/* final-output/wiki/
          
          # 添加 .nojekyll 文件
          touch final-output/.nojekyll
          
          echo "✅ Blog and Wiki combined successfully!"
          echo "📊 Directory structure:"
          ls -la final-output/
          echo "📊 Wiki directory:"
          ls -la final-output/wiki/

      # ========================================
      # 步骤 9: 生成提交信息
      # ========================================
      - name: Generate commit message
        id: commit_message
        run: |
          if [ "${{ github.event_name }}" = "repository_dispatch" ]; then
            echo "message=Deploy Blog+Wiki: repository_dispatch(${{ github.event.action }}) @ ${{ github.sha }}" >> "$GITHUB_OUTPUT"
          elif [ -n "${{ github.event.head_commit.message }}" ]; then
            echo "message=Deploy Blog+Wiki: ${{ github.event.head_commit.message }}" >> "$GITHUB_OUTPUT"
          else
            echo "message=Deploy Blog+Wiki: ${{ github.event_name }} @ ${{ github.sha }}" >> "$GITHUB_OUTPUT"
          fi

      # ========================================
      # 步骤 10: 部署到 GitHub Pages
      # ========================================
      - name: Deploy to GitHub Pages
        uses: peaceiris/actions-gh-pages@v4
        with:
          github_token: ${{ secrets.GITHUB_TOKEN }}
          publish_dir: ./final-output
          publish_branch: gh-pages
          user_name: 'github-actions[bot]'
          user_email: 'github-actions[bot]@users.noreply.github.com'
          commit_message: ${{ steps.commit_message.outputs.message }}
          keep_files: false
          cname: blog.xiangyang.xyz  # ← 你的自定义域名
```

#### 3.2 提交并推送

```bash
cd D:\web\myblog

git add .github/workflows/deploy.yml
git commit -m "feat: Add Wiki integration to deployment workflow"
git push origin main
```

---

### 步骤 4: 在博客导航添加 Wiki 链接

#### 4.1 修改 Freemind 主题配置

编辑 `D:\web\myblog\themes\freemind\_config.yml`，在 `menu` 部分添加 Wiki：

```yaml
menu:
  - title: Archives
    url: archives
    intro: "All the articles."
    icon: "fa fa-archive"
  - title: Categories
    url: categories
    intro: "All the categories."
    icon: "fa fa-folder"
  - title: Tags
    url: tags
    intro: "All the tags."
    icon: "fa fa-tags"
  - title: Wiki              # ← 新增
    url: /wiki               # ← 新增（注意开头的 /）
    intro: "My knowledge base."  # ← 新增
    icon: "fa fa-book"       # ← 新增
  - title: About
    url: about
    intro: "about me"
    icon: "fa fa-user"
```

#### 4.2 提交并推送

```bash
cd D:\web\myblog

git add themes/freemind/_config.yml
git commit -m "feat: Add Wiki link to navigation menu"
git push origin main
```

---

### 步骤 5: 配置自动触发（Wiki 更新自动部署博客）

#### 5.1 创建 Personal Access Token

1. 访问 GitHub Settings → Developer settings → Personal access tokens → Tokens (classic)
2. 点击 "Generate new token (classic)"
3. Note: `Wiki Auto Deploy Token`
4. Expiration: `No expiration`（或根据需求）
5. 勾选权限：
   - ✅ `repo`（全选，包括所有子选项）
6. 点击 "Generate token"
7. **复制 Token**（形如 `ghp_xxxxxxxxxxxx`）

#### 5.2 在 Wiki 仓库添加 Secret

1. 访问 https://github.com/pistachioss/pistachioss-wiki/settings/secrets/actions
2. 点击 "New repository secret"
3. Name: `PERSONAL_ACCESS_TOKEN`
4. Secret: 粘贴刚才的 Token
5. 点击 "Add secret"

#### 5.3 在 Wiki 仓库创建触发器 Workflow

创建文件 `D:\web\myblog-wiki\.github\workflows\trigger-blog.yml`：

```yaml
name: Trigger Blog Deployment

on:
  push:
    branches:
      - main

jobs:
  trigger-blog-rebuild:
    runs-on: ubuntu-latest
    
    steps:
      - name: Trigger Blog Repository Workflow
        run: |
          set -euo pipefail
          response=$(curl -sS -w "\n%{http_code}" -X POST \
            -H "Authorization: token ${{ secrets.PERSONAL_ACCESS_TOKEN }}" \
            -H "Accept: application/vnd.github+json" \
            https://api.github.com/repos/pistachioss/pistachioss.github.io/dispatches \
            -d '{"event_type":"wiki_updated"}')
          
          body=$(echo "$response" | sed '$d')
          code=$(echo "$response" | tail -n1)
          
          echo "HTTP: $code"
          echo "BODY: $body"
          
          if [ "$code" -lt 200 ] || [ "$code" -ge 300 ]; then
            echo "Dispatch failed"
            exit 1
          fi
      
      - name: Workflow Triggered
        run: echo "✅ Blog deployment triggered successfully!"
```

#### 5.4 推送 Workflow

```bash
cd D:\web\myblog-wiki

git add .github/workflows/trigger-blog.yml
git commit -m "feat: Add auto-trigger workflow for blog deployment"
git push origin main
```

---

### 步骤 6: 验证部署

#### 6.1 检查 GitHub Actions

**博客仓库**:
1. 访问 https://github.com/pistachioss/pistachioss.github.io/actions
2. 应该看到一个新的 workflow run
3. 等待构建完成（约 2-3 分钟）

**Wiki 仓库**:
1. 访问 https://github.com/pistachioss/pistachioss-wiki/actions
2. 应该看到 "Trigger Blog Deployment" 成功运行
3. 日志中应该显示 `HTTP: 204` 和 `✅ Blog deployment triggered successfully!`

#### 6.2 访问网站

等待部署完成后，访问：

- **博客首页**: https://blog.xiangyang.xyz
- **Wiki 首页**: https://blog.xiangyang.xyz/wiki

强制刷新浏览器：`Ctrl + Shift + R`

#### 6.3 验证成功的标志

✅ 博客首页有 "Wiki" 菜单  
✅ 点击 Wiki 菜单跳转到 /wiki  
✅ Wiki 首页显示文章列表  
✅ 文章有正确的样式（Wixo 主题）  
✅ Wiki 文章能正常打开和阅读  

---

## 常见问题与解决方案

### 问题 1: Wiki 仓库找不到（Not Found 404）

**错误日志**:
```
Not Found - https://docs.github.com/rest/repos/repos#get-a-repository
```

**原因**: Wiki 仓库还未创建或 GitHub Actions 无权限访问。

**解决方案**:
1. 确保在 GitHub 上创建了 `pistachioss-wiki` 仓库
2. 确保仓库是 **Public**（或配置了正确的 Token）
3. 确保仓库至少有一次提交（main 分支存在）

---

### 问题 2: Wiki 页面显示 "No layout" 警告

**错误日志**:
```
[WARN] No layout: uncategorized/hello-world/index.html
[WARN] No layout: index.html
```

**原因**: Wixo 主题文件未正确添加到 Git 仓库。

**解决方案**:

```bash
cd D:\web\myblog-wiki

# 检查主题文件是否在仓库中
git ls-files themes/wixo

# 如果没有输出，说明主题文件未提交
# 重新添加主题文件
git add themes/wixo -f
git commit -m "Add wixo theme files"
git push origin main
```

**预防措施**:
- 不要使用 `git submodule`，直接克隆主题
- 确保 `.gitignore` 没有忽略 `themes/` 目录
- 推送前检查 `git status` 确认主题文件已添加

---

### 问题 3: Submodule 错误（themes/wixo）

**错误日志**:
```
fatal: No url found for submodule path 'themes/wixo' in .gitmodules
```

**原因**: 主题被误添加为 submodule，但配置不完整。

**解决方案**:

```bash
cd D:\web\myblog-wiki

# 1. 删除 .gitmodules 文件
del .gitmodules  # Windows
# rm .gitmodules  # Linux/Mac

# 2. 移除 Git 中的 submodule 缓存
git rm --cached themes/wixo

# 3. 删除 submodule 配置
rmdir /s .git\modules\themes\wixo  # Windows（如果存在）
# rm -rf .git/modules/themes/wixo  # Linux/Mac

# 4. 确保主题目录存在
dir themes\wixo  # Windows
# ls themes/wixo  # Linux/Mac

# 5. 重新添加主题（作为普通文件）
git add themes/wixo -f

# 6. 提交并推送
git commit -m "Fix: Add wixo theme files directly (not as submodule)"
git push origin main
```

---

### 问题 4: Wiki 更新后博客没有自动部署

**症状**: 推送 Wiki 代码后，博客没有重新部署。

**排查步骤**:

1. **检查 Wiki Actions 是否成功**:
   - 访问 https://github.com/pistachioss/pistachioss-wiki/actions
   - 查看最新的 workflow run
   - 日志应该显示 `HTTP: 204`

2. **检查 Personal Access Token**:
   - 确认在 Wiki 仓库的 Secrets 中添加了 `PERSONAL_ACCESS_TOKEN`
   - Token 应该有 `repo` 权限
   - Token 没有过期

3. **检查博客的 deploy.yml**:
   - 确认添加了 `repository_dispatch` 触发器
   ```yaml
   on:
     push:
       branches:
         - main
     repository_dispatch:
       types: [wiki_updated]  # ← 这一行
   ```

4. **手动触发测试**:
   - 在博客仓库的 Actions 页面
   - 选择 "Build and Deploy Blog with Wiki"
   - 点击 "Run workflow"

---

### 问题 5: 本地能看到 Wiki，线上看不到

**症状**: `hexo server` 本地运行正常，但部署后访问是空白。

**原因**: 通常是主题文件或配置问题。

**排查步骤**:

1. **检查 GitHub Actions 日志**:
   - 查看 "Build Wiki" 步骤
   - 是否有 `[WARN] No layout` 警告
   - 查看生成的文件列表

2. **检查 gh-pages 分支**:
   - 访问 https://github.com/pistachioss/pistachioss.github.io/tree/gh-pages/wiki
   - 确认 wiki 目录存在
   - 确认文章文件存在

3. **检查浏览器缓存**:
   - 强制刷新：`Ctrl + Shift + R`
   - 清除缓存
   - 使用无痕模式访问

4. **直接访问文章 URL**:
   ```
   https://blog.xiangyang.xyz/wiki/uncategorized/hello-world/
   ```
   如果能访问，说明文章已生成，是首页的问题。

---

### 问题 6: Wiki 样式混乱或缺失

**症状**: Wiki 页面能打开，但样式不正确或缺失 CSS/JS。

**原因**: 资源路径配置错误。

**解决方案**:

检查 `myblog-wiki/_config.yml` 的 `root` 配置：

```yaml
# 必须是 /wiki/，不是 /
root: /wiki/
```

如果修改了配置，重新推送：

```bash
cd D:\web\myblog-wiki
git add _config.yml
git commit -m "Fix: Correct root path to /wiki/"
git push origin main
```

---

### 问题 7: 部署后文章没有更新

**症状**: 推送了新文章，但网站上看不到。

**排查步骤**:

1. **确认文章已推送到 GitHub**:
   - 访问 https://github.com/pistachioss/pistachioss-wiki/tree/main/source/_posts
   - 确认新文章文件存在

2. **检查文章格式**:
   ```markdown
   ---
   title: 文章标题
   date: 2026-03-04 12:00:00
   categories:
     - 分类名
   tags:
     - 标签名
   draft: false  # ← 不要设置为 true
   ---
   
   文章内容...
   ```

3. **检查 Actions 日志**:
   - 查看 "Build Wiki" 步骤
   - 确认文章被生成：
   ```
   [INFO] Generated: uncategorized/你的文章/index.html
   ```

4. **清除缓存**:
   - GitHub Pages 需要 1-5 分钟更新
   - 清除浏览器缓存
   - 等待几分钟再访问

---

### 问题 8: 导航栏 Wiki 链接 404

**症状**: 点击博客顶部的 "Wiki" 链接，跳转到 404 页面。

**原因**: Wiki 链接配置错误。

**检查配置**:

`themes/freemind/_config.yml`:

```yaml
menu:
  - title: Wiki
    url: /wiki  # ← 必须有开头的 /，且指向 /wiki 而不是 /wiki/
```

**正确 vs 错误**:
- ✅ `url: /wiki` - 正确，绝对路径
- ❌ `url: wiki` - 错误，相对路径
- ❌ `url: https://blog.xiangyang.xyz/wiki` - 可以但繁琐

---

### 问题 9: 图片或静态资源无法加载

**症状**: Wiki 文章中的图片显示不出来。

**解决方案 1**: 使用图床

推荐使用外部图床服务：
- GitHub 图床
- 腾讯云 COS
- 阿里云 OSS
- Cloudflare Images
- ImgBB

**解决方案 2**: 使用相对路径

```markdown
<!-- 在文章中 -->
![图片描述](/wiki/images/my-image.png)
```

将图片放在 `myblog-wiki/source/images/` 目录。

---

### 问题 10: RSS 或 Sitemap 生成问题

**症状**: Wiki 的 RSS 或 Sitemap 链接失效。

**解决方案**:

安装相关插件：

```bash
cd D:\web\myblog-wiki

# RSS
npm install hexo-generator-feed --save

# Sitemap
npm install hexo-generator-sitemap --save
```

配置 `_config.yml`:

```yaml
# RSS
feed:
  type: atom
  path: atom.xml
  limit: 20

# Sitemap
sitemap:
  path: sitemap.xml
```

---

## 日常使用流程

### 写博客文章

```bash
# 1. 进入博客项目
cd D:\web\myblog

# 2. 创建新文章
hexo new post "我的新文章"

# 3. 编辑文章
# 用你喜欢的编辑器打开 source/_posts/我的新文章.md

# 4. 本地预览（可选）
hexo clean && hexo server

# 5. 提交并推送
git add source/_posts/我的新文章.md
git commit -m "Add: 我的新文章"
git push origin main

# 6. 自动部署
# GitHub Actions 会自动构建博客和 Wiki，然后部署
```

### 写 Wiki 笔记

```bash
# 1. 进入 Wiki 项目
cd D:\web\myblog-wiki

# 2. 创建新笔记
hexo new "Python 学习笔记"

# 3. 编辑笔记
# source/_posts/Python-学习笔记.md

# 4. 本地预览（可选）
hexo clean && hexo server -p 4001

# 5. 提交并推送
git add source/_posts/Python-学习笔记.md
git commit -m "Add: Python 学习笔记"
git push origin main

# 6. 自动触发博客部署
# Wiki 的 Actions 会触发博客的 Actions
# 博客和 Wiki 一起重新部署
```

### 同时运行博客和 Wiki（本地开发）

**Windows PowerShell** - 使用两个终端窗口：

```powershell
# 终端 1 - 博客
cd D:\web\myblog
hexo clean && hexo server

# 终端 2 - Wiki
cd D:\web\myblog-wiki
hexo clean && hexo server -p 4001
```

访问：
- 博客: http://localhost:4000
- Wiki: http://localhost:4001

---

## 高级配置

### 1. 自定义 Wiki 目录结构

推荐的 Wiki 目录组织：

```
myblog-wiki/source/_posts/
├── programming/
│   ├── python/
│   │   ├── python-basics.md
│   │   ├── django-tutorial.md
│   │   └── flask-notes.md
│   ├── javascript/
│   │   ├── es6-features.md
│   │   └── react-hooks.md
│   └── java/
│       └── spring-boot.md
│
├── tools/
│   ├── git-commands.md
│   ├── vscode-shortcuts.md
│   ├── docker-basics.md
│   └── linux-commands.md
│
├── algorithms/
│   ├── sorting.md
│   ├── dynamic-programming.md
│   └── graph-algorithms.md
│
├── database/
│   ├── mysql-optimization.md
│   ├── redis-notes.md
│   └── mongodb-basics.md
│
└── others/
    ├── markdown-syntax.md
    └── interview-questions.md
```

创建文章时指定路径：

```bash
hexo new "programming/python/pandas-tutorial"
```

### 2. 配置评论系统

#### Gitalk（推荐）

**博客项目** (`themes/freemind/_config.yml`):

```yaml
comment_gitalk:
  enable: true
  client_id: "你的 GitHub OAuth App Client ID"
  client_secret: "你的 Client Secret"
  repo: "pistachioss.github.io"
  owner: "pistachioss"
  admin: ["pistachioss"]
  language: zh-CN
```

**Wiki 项目** (`themes/wixo/_config.yml`):

```yaml
comment_gitalk:
  enable: true
  client_id: "你的 GitHub OAuth App Client ID"
  client_secret: "你的 Client Secret"
  repo: "pistachioss-wiki"  # ← 可以用单独的仓库
  owner: "pistachioss"
  admin: ["pistachioss"]
  language: zh-CN
```

创建 GitHub OAuth App:
1. Settings → Developer settings → OAuth Apps → New OAuth App
2. Application name: `Blog Comments`
3. Homepage URL: `https://blog.xiangyang.xyz`
4. Authorization callback URL: `https://blog.xiangyang.xyz`
5. 获取 Client ID 和 Client Secret

### 3. 添加搜索功能

**安装插件**:

```bash
# 博客
cd D:\web\myblog
npm install hexo-generator-search --save

# Wiki
cd D:\web\myblog-wiki
npm install hexo-generator-search --save
```

**配置** (`_config.yml`):

```yaml
search:
  path: search.xml
  field: all
  content: true
```

### 4. 优化构建速度

#### 使用增量构建

```bash
# 只生成修改的文件
hexo generate --incremental
```

#### 并行构建

修改 `_config.yml`:

```yaml
# 使用多核心并行处理
workers: 4
```

#### 缓存优化

GitHub Actions 已经配置了缓存：
- npm 依赖缓存
- Hexo 数据库缓存

### 5. 自定义域名配置

如果 Wiki 想用单独的子域名（如 `wiki.xiangyang.xyz`）：

#### DNS 配置

添加 CNAME 记录：
```
wiki.xiangyang.xyz → pistachioss.github.io
```

#### 修改部署配置

`deploy.yml`:

```yaml
- name: Deploy Wiki to separate subdomain
  uses: peaceiris/actions-gh-pages@v4
  with:
    github_token: ${{ secrets.GITHUB_TOKEN }}
    publish_dir: ./wiki/public
    external_repository: pistachioss/wiki-subdomain
    publish_branch: gh-pages
    cname: wiki.xiangyang.xyz
```

### 6. 添加 Analytics

**Google Analytics**:

```yaml
# _config.yml
google_analytics:
  enable: true
  siteid: 'G-XXXXXXXXXX'
```

**百度统计**:

```yaml
# _config.yml
baidu_tongji:
  enable: true
  siteid: 'your_siteid'
```

### 7. SEO 优化

**安装 SEO 插件**:

```bash
npm install hexo-generator-seo-friendly-sitemap --save
npm install hexo-auto-canonical --save
```

**配置 robots.txt**:

在 `source/` 目录创建 `robots.txt`:

```
User-agent: *
Allow: /
Sitemap: https://blog.xiangyang.xyz/sitemap.xml
Sitemap: https://blog.xiangyang.xyz/wiki/sitemap.xml
```

---

## 总结

### 核心要点

1. **双项目独立**：博客和 Wiki 是两个完全独立的 Hexo 项目
2. **GitHub Actions 合并**：通过 Actions 自动构建并合并到一个网站
3. **路径配置关键**：Wiki 的 `root: /wiki/` 配置决定了最终的访问路径
4. **主题文件必须提交**：不要用 submodule，直接提交主题文件到仓库
5. **自动触发链式部署**：Wiki 更新 → 触发 → 博客重新部署

### 优势

✅ **完全独立**：两个主题互不干扰  
✅ **易于维护**：分别管理博客和 Wiki  
✅ **自动化部署**：推送即部署，无需手动操作  
✅ **灵活扩展**：可以轻松添加更多子项目  
✅ **SEO 友好**：统一域名，有利于搜索引擎收录  

### 注意事项

⚠️ **主题文件必须完整提交**  
⚠️ **不要使用 git submodule**  
⚠️ **URL root 配置要正确**  
⚠️ **Personal Access Token 需要 repo 权限**  
⚠️ **推送前先本地测试**  

---

## 附录

### A. 文件清单

#### 博客项目关键文件

```
myblog/
├── _config.yml                          # Hexo 配置
├── themes/freemind/_config.yml          # Freemind 主题配置
├── .github/workflows/deploy.yml         # 部署 Workflow
├── source/_posts/                       # 博客文章
└── package.json
```

#### Wiki 项目关键文件

```
myblog-wiki/
├── _config.yml                          # Hexo 配置（root: /wiki/）
├── themes/wixo/_config.yml              # Wixo 主题配置
├── .github/workflows/trigger-blog.yml   # 触发器 Workflow
├── source/_posts/                       # Wiki 笔记
└── package.json
```

### B. 常用命令

#### Hexo 命令

```bash
hexo new [layout] <title>      # 创建新文章
hexo new page <title>           # 创建新页面
hexo clean                      # 清理缓存
hexo generate                   # 生成静态文件
hexo server                     # 启动本地服务器
hexo deploy                     # 部署（本方案不使用）
```

#### Git 命令

```bash
git status                      # 查看状态
git add <file>                  # 添加文件
git commit -m "message"         # 提交
git push origin main            # 推送
git log --oneline               # 查看提交历史
git diff                        # 查看差异
```

#### 调试命令

```bash
hexo generate --debug           # 详细日志
hexo server --debug             # 调试模式运行
npm ls                          # 查看依赖
git ls-files themes/wixo        # 查看主题文件是否在仓库中
```

### C. 参考资源

- [Hexo 官方文档](https://hexo.io/zh-cn/docs/)
- [hexo-theme-freemind](https://github.com/wzpan/hexo-theme-freemind)
- [hexo-theme-wixo](https://github.com/wzpan/hexo-theme-wixo)
- [wzpan 的博客](https://www.hahack.com)（参考案例）
- [GitHub Actions 文档](https://docs.github.com/en/actions)
- [GitHub Pages 文档](https://docs.github.com/en/pages)

---

## 结语

恭喜你完成了 Hexo 博客 + Wiki 双主题的部署！🎉

现在你拥有了：
- ✅ 功能完整的博客系统
- ✅ 独立的知识库管理
- ✅ 自动化的部署流程
- ✅ 优雅的双主题展示

**下一步建议**：
1. 创建更多 Wiki 内容，建立你的知识体系
2. 优化主题样式，打造个性化外观
3. 添加评论、搜索等功能
4. 定期备份内容

祝你的博客和知识库越来越丰富！📚✨

---

**文档版本**: v1.0  
**最后更新**: 2026-03-01  
**作者**: Based on pistachioss's implementation  
**License**: MIT
