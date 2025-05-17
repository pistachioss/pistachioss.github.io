---
title: Hexo Github Action部署
date: 2020-02-11
categories:  Hexo
tags: wiki
toc: true
description: "Hexo 迁移自动化部署"

---

根据目前的状况和迁移需求，整理了一份迁移方案。

**核心思路：**

1.  **备份**：确保现有内容安全。
2.  **本地准备**：将你的 Hexo 博客源代码目录初始化为 Git 仓库。
3.  **远程仓库准备**：清理或重命名 `pistachioss.github.io` 仓库的 `main` 分支，因为它目前存放的是静态文件，未来要存放源代码。
4.  **推送源代码**：将本地 Hexo 源代码推送到 `pistachioss.github.io` 仓库的 `main` 分支。
5.  **配置 GitHub Actions**：创建 workflow 文件，实现自动拉取 `main` 分支代码、构建、然后推送到 `gh-pages` 分支。
6.  **GitHub Pages 设置**：更改 GitHub Pages 的部署源为 `gh-pages` 分支。
7.  **CNAME 处理**：确保自定义域名配置正确。

---

**详细迁移步骤：**

**阶段一：准备工作和备份**

1.  **备份现有已部署的网站（可选但推荐）**：
    虽然你的目标是 `gh-pages` 分支，但以防万一，可以先将 `pistachioss.github.io` 仓库 `main` 分支（当前存放静态文件的分支）的内容克隆到本地另一文件夹作为备份。
    ```bash
    git clone https://github.com/pistachioss/pistachioss.github.io.git backup_deployed_site
    ```

2.  **备份你的 Hexo 博客根目录**：
    直接复制整个博客根目录文件夹到另一个安全位置。这是最重要的备份，包含了你的所有源文件。

**阶段二：本地 Hexo 源代码 Git 初始化与配置**

1.  **进入你的 Hexo 博客根目录**：
    ```bash
    cd /path/to/your/hexo-blog-root
    ```

2.  **删除 `.deploy_git` 文件夹（可选但推荐清理）**：
    既然不再使用 `hexo deploy` 的默认方式，可以删除这个文件夹。
    ```bash
    rm -rf .deploy_git
    ```

3.  **初始化 Git 仓库**：
    ```bash
    git init
    ```

4.  **创建 `.gitignore` 文件**：
    在博客根目录下创建 `.gitignore` 文件，告诉 Git 哪些文件不需要版本控制。内容至少应包括：
    ```gitignore
    .DS_Store
    Thumbs.db
    db.json
    *.log
    node_modules/
    public/
    .deploy*/
    # 如果你的主题是通过 git clone 下载的，并且你不希望管理主题的 .git 历史，
    # 可以加上 themes/<your-theme-name>/.git/
    # 或者直接在添加主题时，删除主题文件夹内的 .git 文件夹
    ```
    *注意：如果你的主题是通过 `git submodule` 添加的，则不应忽略主题文件夹。但从你的描述看，似乎不是这种情况。*

5.  **添加 CNAME 文件到 `source` 目录**：
    为了让 Hexo 在生成静态文件时包含你的自定义域名配置，在博客根目录下的 `source` 文件夹中创建一个名为 `CNAME` (无后缀) 的文件。
    文件内容就是你的自定义域名，例如：
    `your.custom.domain.com` (替换成你自己的域名)

6.  **修改 Hexo 配置文件 `_config.yml` (可选，但建议检查)**：
    确保 `url` 和 `root` 配置正确，特别是 `url` 应该指向你的自定义域名。
    ```yaml
    url: https://your.custom.domain.com  # 替换成你的域名
    root: /
    ```
    `deploy` 部分的配置对于 GitHub Actions 来说不是必需的了，你可以注释掉或删除它。

7.  **添加所有文件到 Git 并提交**：
    ```bash
    git add .
    git commit -m "Initial commit of Hexo source code"
    ```

**阶段三：远程 GitHub 仓库 (`pistachioss.github.io`) 配置**

1.  **处理 `pistachioss.github.io` 仓库的 `main` 分支**：
    目前这个分支存放的是已部署的静态文件。我们需要让它存放源代码。你有几个选择：
    *   **推荐做法：重命名远程 `main` 分支**
        1.  打开 GitHub 仓库 `pistachioss/pistachioss.github.io`。
        2.  进入 "Settings" -> "Branches"。
        3.  在 "Default branch" 部分，你可能会看到一个切换默认分支的选项，或者直接找到 `main` 分支，看是否有重命名选项。如果操作不直观，可以先创建一个临时的新分支 (例如 `temp-main`)，将其设为默认分支，然后删除旧的 `main` 分支。
        4.  或者，更简单的是，在本地推送前，先在 GitHub 上将 `main` 分支重命名为 `legacy-deployed-site` 或类似名称。这样可以保留旧的已部署内容以作参考。
    *   **不推荐但可行：强制推送覆盖** (如果你确定不再需要 `main` 分支上的旧静态文件历史)
        这种方法会清除 `main` 分支的现有历史。

2.  **关联本地仓库到远程**：
    ```bash
    git remote add origin https://github.com/pistachioss/pistachioss.github.io.git
    ```
    如果提示 `remote origin already exists.`，说明之前 `.deploy_git` 可能已经设置过。你可以先移除旧的：`git remote rm origin`，然后再执行上面的 `add` 命令。

3.  **推送源代码到 `main` 分支**：
    ```bash
    git push -u origin main
    ```
    如果远程 `main` 分支不存在（因为你可能在上一步删除了或重命名了），这个命令会自动创建它。如果远程 `main` 存在且历史不同（比如你没有清空它），你可能需要强制推送 `git push -u -f origin main`，但请确保你明白强制推送的后果。

**阶段四：设置 GitHub Actions 自动部署**

1.  **在本地博客根目录创建 GitHub Actions workflow 文件**：
    创建文件夹和文件：`.github/workflows/deploy.yml`

2.  **编辑 `deploy.yml` 文件**，填入以下内容：
    ```yaml
    name: Build and Deploy Hexo Blog
    
    on:
      push:
        branches:
          - main # 当 main 分支有 push 操作时触发
    
    jobs:
      build-and-deploy:
        runs-on: ubuntu-latest # 使用最新的 ubuntu 系统作为运行环境
        steps:
          - name: Checkout source code # 步骤1: 拉取 main 分支的源代码
            uses: actions/checkout@v3
    
          - name: Setup Node.js # 步骤2: 设置 Node.js 环境
            uses: actions/setup-node@v3
            with:
              node-version: '18' # 根据你的 Hexo 版本和依赖选择合适的 Node.js 版本 (e.g., 16, 18, 20)
              cache: 'npm' # 缓存 npm 依赖，加快后续构建速度
    
          - name: Install Dependencies # 步骤3: 安装项目依赖
            run: npm install
    
          - name: Build Hexo # 步骤4: 生成静态文件
            run: npx hexo clean && npx hexo generate # 或者 npm run build (如果你在 package.json 中定义了)
    
          - name: Deploy to gh-pages # 步骤5: 部署到 gh-pages 分支
            uses: peaceiris/actions-gh-pages@v3
            with:
              github_token: ${{ secrets.GITHUB_TOKEN }} # GitHub Action 自动提供的 token
              publish_dir: ./public # Hexo 生成的静态文件目录
              publish_branch: gh-pages # 推送到的目标分支
              # cname: your.custom.domain.com # 如果你没有在 source/CNAME 添加文件，可以在这里指定
                                              # 但推荐使用 source/CNAME 文件的方式
              user_name: 'github-actions[bot]' # 提交者名称
              user_email: 'github-actions[bot]@users.noreply.github.com' # 提交者邮箱
              commit_message: 'Deploy: ${{ github.event.head_commit.message }}' # 自定义提交信息
    ```

3.  **提交并推送 workflow 文件**：
    ```bash
    git add .github/workflows/deploy.yml
    git commit -m "Add GitHub Actions workflow for deployment"
    git push origin main
    ```

**阶段五：GitHub Pages 设置和验证**

1.  **更改 GitHub Pages 部署源**：
    *   进入你的 GitHub 仓库 `pistachioss/pistachioss.github.io`。
    *   点击 "Settings" (设置)。
    *   在左侧导航栏选择 "Pages"。
    *   在 "Build and deployment" -> "Source" 部分，选择 "Deploy from a branch"。
    *   在 "Branch" 部分，选择 `gh-pages` 分支和 `/(root)` 目录。
    *   点击 "Save"。

2.  **检查自定义域名 (Custom domain)**：
    *   在 "Pages" 设置页面，向下滚动到 "Custom domain" 部分。
    *   你的域名 `your.custom.domain.com` 应该仍然在此处列出。
    *   如果它丢失了，重新输入并保存。GitHub 可能会重新验证 DNS 设置。
    *   确保 "Enforce HTTPS" 是勾选的。

3.  **触发 Action 并观察**：
    *   你刚才推送 `deploy.yml` 到 `main` 分支的操作应该已经自动触发了 GitHub Action。
    *   进入仓库的 "Actions" 标签页，你应该能看到一个正在运行或已完成的 workflow。
    *   点击进去查看详细日志，确保每一步都成功执行，特别是最后部署到 `gh-pages` 分支的步骤。

4.  **验证部署**：
    *   等待 Action 完成后（可能需要几分钟），访问你的自定义域名。
    *   如果一切顺利，你应该能看到你的博客。首次从 `gh-pages` 部署可能需要一点时间让 GitHub Pages 更新。

**迁移完成！**

**后续维护：**
以后，你只需要在本地修改你的 Hexo 博客内容（写新文章、修改主题等），然后将更改 `commit` 并 `push` 到 `main` 分支。GitHub Actions 会自动完成构建和部署到 `gh-pages` 分支的后续所有工作。

---

**总结一下关键点：**
*   **博客源代码** (Markdown 文件, 主题配置, Hexo `_config.yml` 等) 现在位于 `pistachioss.github.io` 仓库的 `main` 分支。
*   **生成的静态 HTML/CSS/JS 文件** 会由 GitHub Actions 自动构建并推送到 `pistachioss.github.io` 仓库的 `gh-pages` 分支。
*   **GitHub Pages** 设置为从 `gh-pages` 分支提供服务。
*   **自定义域名** 通过 `source/CNAME` 文件（或 Action 配置）和 GitHub Pages 设置来保持。

这个方案应该能满足你的需求。如果在操作过程中遇到任何问题，欢迎追问。