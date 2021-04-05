---
title: hexo config
top: false
cover: false
password:
toc: true
mathjax: true
summary:
tags: hexo
categories: 环境搭建
---

[Hexo](https://hexo.io/zh-cn/) 是一款快速、简洁并且高效博客框架

## Prepare the environment

- Node.js >= 12.0
- npm
- git
- hexo >= 5.0

以上环境搭建若不会建议问 Google

### Init hexo

```bash
npm install hexo-cli -g
# 新建空文件夹
mkdir blog
cd blog
# 初始化基础文件
hexo init
# 安装依赖
npm install
```

## Quick Start

### Create a new post

```bash
hexo new "My New Post"
```

More info: [Writing](https://hexo.io/docs/writing.html)

### Run server

``` bash
hexo server
# 简化
hexo s
```

More info: [Server](https://hexo.io/docs/server.html)

### Generate static files

```bash
hexo generate
# 简化
hexo g
```

More info: [Generating](https://hexo.io/docs/generating.html)

### Deploy to remote sites

``` bash
hexo deploy
# 简化
hexo d
```

> **注意**：若 Github 设置自定义域名，部署之后，`CNAME` 文件会被覆盖，导致网站不可用,可以将 `CNAME` 文件放在`themes/[theme-name]/source`目录下，这样编译静态文件会自动带过去，之后会一同推送到远端

More info: [Deployment](https://hexo.io/docs/one-command-deployment.html)

### Clean

```bash
hexo clean
```

清除缓存文件 (db.json) 和已生成的静态文件 (public)。

## Change theme

hexo 有许多优秀主题可供切换，本博客使用的主题是[matery](https://github.com/blinkfox/hexo-theme-matery/blob/develop/README_CN.md)，具体修改配置参见README
