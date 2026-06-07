---
title: "Hugo + PaperMod 个人博客搭建实践回顾"
date: 2026-05-10
draft: false
description: "记录一次从 GitHub 用户名确定、Hugo 建站、PaperMod 主题配置，到归档、搜索、缓存和端口问题排查的完整过程。"
tags: ["Hugo", "PaperMod", "GitHub Pages", "WSL", "博客"]
categories: ["博客搭建", "编程环境"]
ShowToc: true
TocOpen: false
---

这篇文章记录一次使用 **Hugo + PaperMod + GitHub Pages** 搭建个人博客的完整过程。

本次博客搭建的目标是：在 WSL 环境中创建一个可长期维护的个人技术博客，用于记录学习过程。

<!--more-->

## 一、最终方案

本次最终选择的技术路线是：

```text
WSL Ubuntu
  ↓
Hugo 静态博客框架
  ↓
PaperMod 主题
  ↓
Markdown 写文章
  ↓
Git 管理源码
  ↓
GitHub Pages 部署
```

最终博客信息：

```text
GitHub 用户名：wisexplore
博客名称：WiseXplore
博客地址：https://wisexplore.github.io
博客框架：Hugo
博客主题：PaperMod
部署方式：GitHub Pages
```

选择 Hugo 的主要原因：

- Hugo 属于 Go 生态，和当前学习 Go、Linux、终端开发比较契合；
- Hugo 构建速度快，适合长期维护；
- Hugo 的项目结构清晰，适合用 Git 管理；
- PaperMod 主题简洁，适合技术博客。

---

## 二、创建 Hugo 项目

进入项目目录：

```bash
cd ~/project
```

创建 Hugo 站点：

```bash
hugo new site wisexplore.github.io
cd wisexplore.github.io
```

初始化 Git 仓库：

```bash
git init
git branch -M master
```

这里使用 `master` 作为默认分支，和 GitHub 当前默认分支习惯保持一致。

---

## 三、安装 PaperMod 主题

使用 Git submodule 安装主题：

```bash
git submodule add --depth=1 https://github.com/adityatelange/hugo-PaperMod.git themes/PaperMod
```

检查主题是否安装成功：

```bash
ls themes
ls themes/PaperMod
```

正常情况下可以看到：

```text
PaperMod
```

以及：

```text
LICENSE  README.md  assets  go.mod  i18n  images  layouts  theme.toml
```

其中比较关键的是：

```text
assets
layouts
theme.toml
```

如果 `layouts` 不存在，Hugo 就无法使用主题模板渲染页面。

---

## 四、配置 hugo.yaml

Hugo 初始化项目后，默认可能会生成：

```text
hugo.toml
```

但本次使用的是：

```text
hugo.yaml
```

创建或编辑配置文件：

```bash
vim hugo.yaml
```

配置内容如下：

```yaml
baseURL: "https://wisexplore.github.io/"
title: "WiseXplore"
theme: "PaperMod"
languageCode: "zh-cn"
defaultContentLanguage: "zh"

enableRobotsTXT: true
enableEmoji: true
hasCJKLanguage: true

pagination:
  pagerSize: 10

params:
  env: production
  title: "WiseXplore"
  description: "我的小天地。"
  author: "wisexplore"

  defaultTheme: auto
  ShowReadingTime: true
  ShowPostNavLinks: true
  ShowBreadCrumbs: true
  ShowCodeCopyButtons: true
  ShowToc: true
  TocOpen: false

  homeInfoParams:
    Title: "WiseXplore"
    Content: "我的小天地。"

  socialIcons:
    - name: github
      url: "https://github.com/wisexplore"

menu:
  main:
    - identifier: posts
      name: 文章
      url: /posts/
      weight: 10
    - identifier: archives
      name: 归档
      url: /archives/
      weight: 20
    - identifier: tags
      name: 标签
      url: /tags/
      weight: 30
    - identifier: search
      name: 搜索
      url: /search/
      weight: 40

outputs:
  home:
    - HTML
    - RSS
    - JSON
```

几个关键配置说明：

| 配置 | 作用 |
|---|---|
| `baseURL` | 博客最终访问地址 |
| `theme: "PaperMod"` | 指定使用 PaperMod 主题 |
| `hasCJKLanguage: true` | 更好支持中文统计、摘要和阅读时间 |
| `ShowToc: true` | 文章页显示目录 |
| `ShowCodeCopyButtons: true` | 代码块显示复制按钮 |
| `outputs.home.JSON` | 为搜索功能生成 `index.json` |

---

## 五、删除默认 hugo.toml

一开始遇到过这样的警告：

```text
found no layout file for "html" for kind "home"
found no layout file for "html" for kind "section"
found no layout file for "html" for kind "page"
```

这个问题的原因是项目根目录下同时存在：

```text
hugo.toml
hugo.yaml
```

其中 `hugo.toml` 是 Hugo 默认生成的配置文件，内容类似：

```toml
baseURL = 'https://example.org/'
languageCode = 'en-us'
title = 'My New Hugo Site'
```

它没有配置：

```yaml
theme: "PaperMod"
```

所以 Hugo 没有正确加载 PaperMod 主题，导致找不到页面模板。

解决方法是删除默认配置：

```bash
rm hugo.toml
```

保留：

```text
hugo.yaml
```

重新启动后，PaperMod 主题即可正常加载。

---

## 六、创建第一篇文章

创建文章：

```bash
hugo new posts/wsl-golang-docker-k8s-practice.md
```

或者直接编辑：

```bash
vim content/posts/wsl-golang-docker-k8s-practice.md
```

文章头部需要写 Front Matter，例如：

```yaml
---
title: "WSL + Git + Go + Docker + K8s 实战回顾"
date: 2026-05-10
draft: false
description: "从 WSL 环境配置到 Go Web 服务、Docker 容器化、kind 本地 Kubernetes 部署的一次完整实践复盘。"
tags: ["WSL", "Git", "Go", "Docker", "Kubernetes"]
categories: ["编程环境", "云原生"]
ShowToc: true
TocOpen: false
---
```

其中：

```yaml
draft: false
```

表示这篇文章正式发布。

如果是：

```yaml
draft: true
```

表示草稿。正式构建时，草稿文章不会显示。

---

## 七、处理文章脱敏

发布技术实践文章前，需要检查是否包含敏感信息。

本次主要处理了以下内容：

| 类型 | 处理方式 |
|---|---|
| 旧 GitHub 用户名 | 替换成当前用户名或占位符 |
| 本机用户目录 | 使用 `~` 或 `/home/<user>` |
| 局域网代理地址 | 替换成 `<proxy-host>:<proxy-port>` |
| 终端主机名 | 替换成通用示例 |
| 私钥、Token、密码 | 不应出现在文章中 |

---

## 八、创建归档页

PaperMod 的归档页面需要手动创建。

创建目录：

```bash
mkdir -p content/archives
```

创建文件：

```bash
vim content/archives/index.md
```

写入：

```markdown
---
title: "归档"
layout: "archives"
url: "/archives/"
summary: archives
---
```

之后访问：

```text
http://localhost:1313/archives/
```

归档页会根据文章的 `date` 自动按时间整理文章。

---

## 九、创建搜索页

PaperMod 的搜索页面也需要手动创建。

创建目录：

```bash
mkdir -p content/search
```

创建文件：

```bash
vim content/search/index.md
```

写入：

```markdown
---
title: "搜索"
layout: "search"
url: "/search/"
summary: search
---
```

搜索功能依赖 `hugo.yaml` 中的：

```yaml
outputs:
  home:
    - HTML
    - RSS
    - JSON
```

其中 `JSON` 会生成搜索索引：

```text
public/index.json
```

如果没有这个配置，搜索功能可能无法正常工作。

---

## 十、启动本地预览

本地开发时使用：

```bash
hugo server --disableFastRender
```

浏览器访问：

```text
http://localhost:1313/
```

这个命令的含义：

| 部分 | 作用 |
|---|---|
| `hugo server` | 启动本地预览服务器 |
| `--disableFastRender` | 禁用快速渲染，改配置和搜索时更稳定 |

如果需要显示草稿文章，可以使用：

```bash
hugo server -D
```

其中 `-D` 表示显示 `draft: true` 的文章。

---

## 十一、正式构建网站

发布前可以执行：

```bash
hugo --gc --minify
```

这个命令会生成最终静态网站到：

```text
public/
```

参数解释：

| 参数 | 作用 |
|---|---|
| `hugo` | 构建静态网站 |
| `--gc` | 清理无用缓存资源 |
| `--minify` | 压缩 HTML、CSS、JS 等文件 |

这个命令适合：

- 发布前检查构建是否正常；
- GitHub Actions 中自动构建；
- 清理缓存后重新生成正式网站。

---

## 十二、解决搜索重复问题

后来搜索页面出现了重复结果：明明只有一篇文章，却搜索出了多条相同结果。

先检查文章标题：

```bash
grep -R "title:" content
```

结果显示只有：

```text
content/archives/index.md:title: "归档"
content/search/index.md:title: "搜索"
content/posts/wsl-golang-docker-k8s-practice.md:title: "WSL + Git + Go + Docker + K8s 实战回顾"
```

说明 `content/posts/` 中并没有多篇重复文章。

因此判断问题来自旧构建产物或搜索索引缓存残留。PaperMod 搜索依赖：

```text
public/index.json
```

如果旧索引没有清理干净，可能会出现重复结果。

解决方法：

```bash
rm -rf public resources .hugo_build.lock
hugo --gc --minify
hugo server --disableFastRender
```

这些目录和文件的作用：

| 目录/文件 | 作用 |
|---|---|
| `public/` | Hugo 构建生成的最终网站文件 |
| `resources/` | Hugo 处理资源时产生的缓存 |
| `.hugo_build.lock` | Hugo 构建锁文件 |

清理后重新生成搜索索引，重复结果消失。

---

## 十三、解决 1313 端口占用问题

启动 Hugo 时出现：

```text
port 1313 already in use, attempting to use an available port
Web Server is available at http://localhost:39941/
```

这表示默认端口 `1313` 被旧的 Hugo 服务占用了。

查看 Hugo 进程：

```bash
ps aux | grep hugo
```

可能会看到：

```text
hugo server -D
```

如果状态中有 `T`，例如：

```text
Tl
```

说明进程处于暂停状态，通常是之前按过：

```text
Ctrl + Z
```

`Ctrl + Z` 是暂停进程，不是停止进程。停止 Hugo 服务应该使用：

```text
Ctrl + C
```

如果已经出现旧进程占端口，可以杀掉：

```bash
pkill -9 hugo
```

再次确认：

```bash
ps aux | grep hugo
```

如果只剩：

```text
grep --color=auto hugo
```

说明真正的 Hugo 服务已经停止。

然后重新启动：

```bash
hugo server --disableFastRender
```

即可重新使用：

```text
http://localhost:1313/
```

---

## 十四、理解 ps aux | grep hugo

命令：

```bash
ps aux | grep hugo
```

用于查找 Hugo 相关进程。

拆开看：

```bash
ps aux
```

表示列出系统中正在运行的进程。

```bash
grep hugo
```

表示筛选包含 `hugo` 的行。

中间的：

```bash
|
```

叫管道，表示把前一个命令的输出交给后一个命令处理。

如果看到：

```text
hugo server -D
```

说明 Hugo 服务还在运行。

如果只看到：

```text
grep --color=auto hugo
```

说明只是搜索命令本身，不代表 Hugo 服务还在。

---

## 十五、RSS 是什么

RSS 可以理解成博客的订阅源。

Hugo 会生成类似：

```text
https://wisexplore.github.io/index.xml
```

别人可以用 RSS 阅读器订阅博客。以后博客发布新文章，对方可以在阅读器里看到更新。

配置中的：

```yaml
outputs:
  home:
    - HTML
    - RSS
    - JSON
```

含义：

| 输出 | 作用 |
|---|---|
| `HTML` | 正常网页 |
| `RSS` | 订阅源 |
| `JSON` | 搜索索引 |

---

## 十六、Markdown 图片放在哪里

推荐把图片放到：

```text
static/images/
```

例如：

```text
static/images/hugo-preview.png
```

Markdown 中引用：

```markdown
![Hugo 本地预览效果](/images/hugo-preview.png)
```

注意不是：

```markdown
![错误写法](/static/images/hugo-preview.png)
```

因为 Hugo 构建时会把 `static/` 目录中的内容直接放到网站根路径。

也可以按文章分类管理：

```text
static/images/
├── hugo-blog/
│   ├── preview.png
│   └── search.png
└── wsl-k8s/
    ├── docker-build.png
    └── kubectl-pods.png
```

文章中引用：

```markdown
![搜索页面效果](/images/hugo-blog/search.png)
```

---

## 十七、当前博客结构

目前项目结构大致如下：

```text
wisexplore.github.io/
├── archetypes/
├── assets/
├── content/
│   ├── archives/
│   │   └── index.md
│   ├── posts/
│   │   └── wsl-golang-docker-k8s-practice.md
│   └── search/
│       └── index.md
├── static/
├── themes/
│   └── PaperMod/
├── hugo.yaml
└── public/
```

其中：

| 目录/文件 | 作用 |
|---|---|
| `content/posts/` | 存放博客文章 |
| `content/archives/` | 归档页 |
| `content/search/` | 搜索页 |
| `static/` | 存放图片等静态资源 |
| `themes/PaperMod/` | PaperMod 主题 |
| `hugo.yaml` | Hugo 主配置文件 |
| `public/` | 构建生成的网站文件 |

---

## 十八、提交到 GitHub：使用 master 分支

本地确认没有问题后，可以把 Hugo 博客源码提交到 GitHub。  
本次坚持使用 `master` 分支，因此后续所有 GitHub Actions 配置也统一监听 `master`。

### 18.1 建议先配置 .gitignore

Hugo 本地构建会生成一些临时文件和产物，例如：

```text
public/
resources/
.hugo_build.lock
```

如果后续使用 GitHub Actions 自动部署，一般不需要把 `public/` 提交到源码仓库。

创建 `.gitignore`：

```bash
vim .gitignore
```

写入：

```gitignore
# Hugo build output
public/
resources/
.hugo_build.lock

# OS files
.DS_Store
Thumbs.db

# Editor
.vscode/
.idea/

# Logs
*.log
```

### 18.2 处理 Git 换行符提示

执行 `git add .` 时出现过类似提示：

```text
LF will be replaced by CRLF the next time Git touches it
```

含义是：当前文件使用 Linux 换行符 `LF`，但 Git 配置可能会在后续把它转成 Windows 换行符 `CRLF`。

因为博客是在 WSL/Linux 环境中维护，建议保持 `LF`。可以执行：

```bash
git config core.autocrlf input
git config --global core.autocrlf input
```

含义：提交时可以把 `CRLF` 转成 `LF`，但检出文件时不强制转成 `CRLF`。

如果前面已经 `git add .`，可以重新整理暂存区：

```bash
git reset
git add .
```

### 18.3 使用 master 分支

初始化后如果当前分支不是 `master`，可以执行：

```bash
git branch -M master
```

查看状态：

```bash
git status
```

期望看到：

```text
On branch master
```

### 18.4 第一次提交

查看状态：

```bash
git status
```

添加文件：

```bash
git add .
```

提交：

```bash
git commit -m "init hugo papermod blog"
```

绑定远程仓库：

```bash
git remote add origin https://github.com/wisexplore/wisexplore.github.io.git
```

如果远程仓库已经存在，则使用：

```bash
git remote set-url origin https://github.com/wisexplore/wisexplore.github.io.git
```

查看远程仓库：

```bash
git remote -v
```

推送到 `master`：

```bash
git push -u origin master
```

推送成功后，GitHub 仓库中应能看到：

```text
archetypes/
assets/css/extended/
content/
themes/
.gitignore
.gitmodules
hugo.yaml
```

---

## 十九、修复 PaperMod submodule 提示

执行 `git add .` 时曾出现过这样的提示：

```text
warning: adding embedded git repository: themes/PaperMod
You've added another git repository inside your current repository.
```

这个提示说明：`themes/PaperMod` 本身也是一个 Git 仓库，而当前博客项目也是一个 Git 仓库。Git 担心你只是把一个“嵌套 Git 仓库”硬加进来，而不是用 submodule 管理。

本次选择用 **Git submodule** 管理 PaperMod 主题。修复步骤如下：

```bash
cd ~/project/wisexplore.github.io

git reset

git rm --cached .gitmodules 2>/dev/null || true
git rm --cached themes/PaperMod 2>/dev/null || true

rm -f .gitmodules
rm -rf .git/modules/themes/PaperMod
rm -rf themes/PaperMod
```

重新添加 PaperMod submodule：

```bash
git submodule add --depth=1 https://github.com/adityatelange/hugo-PaperMod.git themes/PaperMod
```

检查：

```bash
ls themes
ls themes/PaperMod
cat .gitmodules
```

正常输出类似：

```text
PaperMod
LICENSE  README.md  assets  go.mod  i18n  images  layouts  theme.toml
```

`.gitmodules` 应类似：

```ini
[submodule "themes/PaperMod"]
        path = themes/PaperMod
        url = https://github.com/adityatelange/hugo-PaperMod.git
```

然后重新添加并提交：

```bash
git add .
git status
git commit -m "init hugo papermod blog"
git push -u origin master
```

如果后续别人 clone 这个仓库，需要带上 submodule：

```bash
git clone --recurse-submodules https://github.com/wisexplore/wisexplore.github.io.git
```

如果已经普通 clone 了，也可以进入项目后执行：

```bash
git submodule update --init --recursive
```

---

## 二十、配置 GitHub Actions 自动部署

源码 push 到 GitHub 后，下一步是让 GitHub Actions 自动构建 Hugo，并发布到 GitHub Pages。

创建 workflow 文件：

```bash
cd ~/project/wisexplore.github.io
mkdir -p .github/workflows
vim .github/workflows/hugo.yml
```

写入：

```yaml
name: Deploy Hugo site to GitHub Pages

on:
  push:
    branches:
      - master
  workflow_dispatch:

permissions:
  contents: read
  pages: write
  id-token: write

concurrency:
  group: pages
  cancel-in-progress: false

defaults:
  run:
    shell: bash

jobs:
  build:
    runs-on: ubuntu-latest
    env:
      HUGO_VERSION: 0.154.5
    steps:
      - name: Checkout
        uses: actions/checkout@v4
        with:
          submodules: recursive
          fetch-depth: 0

      - name: Install Hugo
        run: |
          wget -O ${{ runner.temp }}/hugo.deb https://github.com/gohugoio/hugo/releases/download/v${HUGO_VERSION}/hugo_extended_${HUGO_VERSION}_linux-amd64.deb
          sudo dpkg -i ${{ runner.temp }}/hugo.deb

      - name: Configure Pages
        uses: actions/configure-pages@v5

      - name: Build
        run: hugo --gc --minify

      - name: Upload artifact
        uses: actions/upload-pages-artifact@v3
        with:
          path: ./public

  deploy:
    environment:
      name: github-pages
    runs-on: ubuntu-latest
    needs: build
    steps:
      - name: Deploy to GitHub Pages
        uses: actions/deploy-pages@v4
```

这里最关键的是：

```yaml
branches:
  - master
```

因为本次仓库使用的是 `master` 分支，所以 workflow 必须监听 `master`。

另一个关键点是：

```yaml
submodules: recursive
```

因为 PaperMod 是 submodule，GitHub Actions 构建时必须把主题也拉下来，否则 Hugo 找不到主题模板。

提交 workflow：

```bash
git add .github/workflows/hugo.yml
git commit -m "ci: deploy hugo site to github pages"
git push
```

推送后，在 GitHub 仓库顶部进入：

```text
Actions
```

可以看到：

```text
Deploy Hugo site to GitHub Pages
```

---

## 二十一、设置 GitHub Pages 来源

部署第一次出现 404 时，原因不是 Hugo 本地构建失败，而是 GitHub Pages 可能还在使用：

```text
Deploy from a branch
```

这种模式会直接把仓库分支作为静态网站发布。由于 Hugo 源码仓库根目录没有 `index.html`，所以访问根路径会出现：

```text
404
File not found
For root URLs, you must provide an index.html file.
```

Hugo 的 `index.html` 是构建后生成在：

```text
public/index.html
```

因此正确做法是让 GitHub Pages 使用 GitHub Actions 构建产物。

进入仓库：

```text
Settings
  ↓
Pages
  ↓
Build and deployment
  ↓
Source 选择 GitHub Actions
```

如果一开始找不到 Pages，可以直接访问：

```text
https://github.com/wisexplore/wisexplore.github.io/settings/pages
```

注意不要进错到：

```text
Settings → Deploy keys
```

`Deploy keys` 是配置 SSH 部署密钥的，不是 GitHub Pages。

---

## 二十二、重新触发部署并解决 404

切换 Pages Source 为 `GitHub Actions` 后，可以重新触发部署。

方式一：在 GitHub Actions 页面手动运行：

```text
Actions
  ↓
Deploy Hugo site to GitHub Pages
  ↓
Run workflow
  ↓
选择 master
```

方式二：本地推送一个空提交：

```bash
git commit --allow-empty -m "chore: trigger pages deploy"
git push
```

触发后，在 Actions 中会看到类似：

```text
Deploy Hugo site to GitHub Pages #2
Commit fd467dd pushed by wisexplore
```

点进去查看是否全部绿色通过，尤其关注：

```text
Checkout
Install Hugo
Configure Pages
Build
Upload artifact
Deploy to GitHub Pages
```

如果 workflow 通过，稍等几十秒到几分钟，访问：

```text
https://wisexplore.github.io
```

如果仍然 404，可以强制刷新：

```text
Ctrl + F5
```

最终网站成功上线。

---

## 二十三、日常写博客流程

以后写博客的基本流程：

```bash
cd ~/project/wisexplore.github.io
```

新建文章：

```bash
hugo new posts/article-name.md
```

编辑文章：

```bash
vim content/posts/article-name.md
```

本地预览：

```bash
hugo server --disableFastRender
```

构建检查：

```bash
hugo --gc --minify
```

提交并推送到 `master`：

```bash
git status
git add .
git commit -m "add new post"
git push
```

推送后，GitHub Actions 会自动构建 Hugo，并发布到 GitHub Pages。

---

## 二十四、本次踩坑总结

| 问题 | 原因 | 解决方法 |
|---|---|---|
| 首页 Page Not Found | 主题没有正确加载 | 删除默认 `hugo.toml`，保留 `hugo.yaml` |
| found no layout file | Hugo 没读到 PaperMod 配置 | 确认 `theme: "PaperMod"` 生效 |
| 归档页 404 | 没创建归档页面 | 创建 `content/archives/index.md` |
| 搜索页 404 | 没创建搜索页面 | 创建 `content/search/index.md` |
| 搜索结果重复 | 旧搜索索引缓存残留 | 删除 `public resources .hugo_build.lock` 后重建 |
| 1313 端口占用 | 旧 Hugo 进程没停止 | `ps aux | grep hugo` 查看，`pkill -9 hugo` 清理 |
| Ctrl + Z 后服务没停 | `Ctrl + Z` 是暂停，不是停止 | 停服务用 `Ctrl + C` |
| 文章不显示 | `draft: true` | 改成 `draft: false` |
| 搜索不可用 | 未生成 JSON 索引 | 配置 `outputs.home.JSON` |
| Git 提示 CRLF | WSL 项目不建议自动转 Windows 换行 | `git config --global core.autocrlf input` |
| PaperMod 被当成嵌套仓库 | 主题目录本身含 `.git` | 清理后重新用 submodule 添加 |
| GitHub Pages 404 | Pages 直接发布分支根目录，没有 `index.html` | Source 改为 `GitHub Actions` |
| Actions 找不到主题 | 没拉取 submodule | checkout 配置 `submodules: recursive` |
| workflow 不触发 | 分支名不一致 | workflow 监听 `master` |

---

## 二十五、最终理解

这次搭建的核心链路是：

```text
Markdown 文章
  ↓
Hugo 读取内容和配置
  ↓
PaperMod 负责页面渲染
  ↓
Hugo 生成 public 静态文件
  ↓
本地通过 hugo server 预览
  ↓
Git 管理博客源码
  ↓
Push 到 GitHub master 分支
  ↓
GitHub Actions 拉取源码和 PaperMod submodule
  ↓
GitHub Actions 执行 hugo --gc --minify
  ↓
GitHub Pages 发布构建产物
  ↓
https://wisexplore.github.io 成功访问
```

最重要的经验是：

```text
Hugo 项目要保证配置文件清晰
主题要正确加载
归档和搜索需要单独创建页面
搜索依赖 index.json
旧构建缓存可能导致异常
PaperMod 建议用 submodule 管理
GitHub Actions 要拉取 submodule
Pages Source 要选择 GitHub Actions
本次全流程统一使用 master 分支
本地服务要用 Ctrl + C 正确停止
```

博客搭建只是第一步。后面更重要的是把每一次实践写成可复盘、可检索、可长期积累的文章。
