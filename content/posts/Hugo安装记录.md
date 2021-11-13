---
title: "Hugo安装记录"
date: 2021-11-13T13:15:02+08:00
draft: true
tags: ["Hugo"]
categories: [""]
description: "记录了安装Hugo过程和踩坑"
summary: ""
draft: false
---

# 安装Hugo

本机的环境是Mac，并且已经安装了`Homebrew`，直接使用命令安装：

```shell
brew install hugo
```

在硬盘中选取合适的存储路径，然后命令行中使用如下指令生成网页本地文档：

```shell
hugo new site personal-site
```

> Note: 使用 -f 切换到使用yml格式的配置文件，PaperMod主题推荐使用yml
>
> ```shell
> hugo new site <name of site> -f yml
> ```

由此可得到如下文件目录：

```text
personal-site
├── archetypes
├── config.toml
├── content
├── data
├── layouts
├── static
└── themes
```

常用目录用处如下：

| 子目录名称 | 功能 |
| ------------ | ---------------------------------------------------------------------- |
| archetypes | 新文章默认模板 |
| config.toml | `Hugo` 配置文档 |
| content | 存放所有 `Markdown` 格式的文章 |
| layouts | 存放自定义的 `view`，可为空 |
| static | 存放图像、CNAME、css、js 等资源，发布后该目录下所有资源将处于网页根目录 |
| themes | 存放下载的主题 |

使用下面的命令生成新的文章草稿：

```shell
hugo new posts/first-post.md
```

在 content 目录中会自动以 `archetypes/default.md` 为模板在 `content/posts` 目录下生成一篇名为 `first-post.md` 的文章草稿：

```text
---
title: "First Post"
date: 2017-12-27T23:15:53-05:00
draft: true
---
```

我们可以加一个标题在下面并去掉标记为草稿的这一行：`draft: true`

```text
---
title: "First Post"
date: 2017-12-27T23:15:53-05:00
---

## Hello world
```

下载PaperMod主题并加载到`config.toml` 文件中：

```shell
git init
git submodule add https://github.com/adityatelange/hugo-PaperMod.git themes/PaperMod --depth=1
git submodule update --init --recursive # needed when you reclone your repo (submodules may not get cloned automatically)

# Edit your config.toml configuration file
# and add the Ananke theme.
echo 'theme = "PaperMod"' >> config.tomlecho 'theme = "ananke"' >> config.yml
```

将`.md`文件编译成静态页面，使用命令便生成在默认的 public 子目录中了：

```shell
hugo
```

使用如下命令建立本地服务器：

```shell
hugo server
```

并在浏览器中输入网址 `http://localhost:1313/` 就可以在浏览器中查看网页效果了。

![KG4VRaIs6F8Bpik](https://i.loli.net/2021/11/13/KG4VRaIs6F8Bpik.png)

# 配合 Github Actions 自动构建

## 初始化 GitHub 仓库

新建一个repo，用来保存我们的 Hugo 的源码。

然后起一个 `gh-pages` 分支，推送到远端，用来当做我们的 GitHub Pages 展示的分支。

GitHub Pages 其实就是一个静态页面展示的一个地方，他利用生成静态页面，直接通过域名来给用户展示。

```shell
git checkout -b gh-pages
git push origin gh-pages
git checkout -b master
git push origin master
```

## Actions 自动构建

这里，我们只需要去监听 develop 是否推送就可以了。

构建我们需要做一下流程：

1. 检出代码
2. 安装 Hugo 环境
3. 编译 Hugo
4. 将 public 下的文件夹推送到 gh-pages 分支

我们再 `.github/workflows` 里面新建一个 `gh_pages.yml`:

```yml
name: GitHub Page Deploy

on:
  push:
    branches:
      - master
jobs:
  build-deploy:
    runs-on: ubuntu-18.04
    steps:
      - name: Checkout master
        uses: actions/checkout@v1
      - name: Setup Hugo
        uses: peaceiris/actions-hugo@v2
        with:
          hugo-version: '0.61.0'
          # extended: true

      - name: Build Hugo
        run: |
          hugo

      - name: Deploy Hugo to gh-pages
        uses: peaceiris/actions-gh-pages@v2
        env:
          ACTIONS_DEPLOY_KEY: ${{ secrets.ACTIONS_DEPLOY_KEY }}
          PUBLISH_BRANCH: gh-pages
          PUBLISH_DIR: ./public
```

上面代码中，只要配几个参数就可以用。参数之中， 需要我们的秘钥去推送到 gh-pages 分支，使用的是加密变量，需要在项目的 settings/secrets 菜单里面设置。

具体 我们可以看 [peaceiris/actions-gh-pages@v2](https://github.com/peaceiris/actions-gh-pages) 的文档，里面告诉了我们如何加入到 secrets 里面。

## 推送到 Github

最后编写推送到Github的脚本：

```sh
#!/bin/bash

echo -e "\033[0;32mDeploying updates to GitHub...\033[0m"

# Build the project.
hugo -t even

# Add changes to git.
git add .

# Commit changes.
msg="rebuilding site `date`"
if [ $# -eq 1 ]
  then msg="$1"
fi
git commit -m "$msg"

# Push source and build repos.
git push origin master
```

