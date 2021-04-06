---
title: hexo config
top: false
cover: false
password:
toc: true
mathjax: true
summary: 简单介绍 hexo 的环境搭建及基础命令
tags: hexo
categories: 环境搭建
---

[Hexo](https://hexo.io/zh-cn/) 是一款快速、简洁并且高效博客框架，几乎不需要了解前端知识就能搭建，有很多主题可供选择，容易上手，非常适合自建博客网站。

## Prepare the environment

- Node.js >= 12.0
- npm
- git
- hexo >= 5.0

前三个环境搭建若不会建议问 Google，这里着重介绍 hexo 的安装及部署

### Init hexo

```bash
# 全局安装 hexo 命令行客户端
npm install hexo-cli -g
# 新建空文件夹
mkdir blog
cd blog
# 初始化基础文件
hexo init
# 安装 Node.js 相关依赖库
npm install
```

上述命令完成后，应该会有如下目录

```text
.
├── _config.yml     // 基本配置信息
├── node_modules    // 依赖库
├── package.json    // 项目信息
├── scaffolds       // 文章模板文件夹
├── source          // 文章资源
|   ├── _drafts     // 草稿目录
|   └── _posts      // 文章目录
└── themes          // 可配置主题，用于生成静态页面
```

## Quick Start

### Create a new post

```bash
# 默认使用 _config.yml 中的 default_layout 参数代替。默认参数为 post, 即要发布的文章
hexo new "My New Post"
# 可以在 new 后面添加 `post` `draft` `page` 等选项创建不同的文件
# 新建草稿
hexo new draft "new draft"
# 发布草稿, 将草稿移动至文章目录下
hexo publish draft "new draft"
```

More info: [Writing](https://hexo.io/zh-cn/docs/writing)

### Run server

``` bash
hexo server
# 简化
hexo s
```

More info: [Server](https://hexo.io/zh-cn/docs/server.html)

### Generate static files

```bash
hexo generate
# 简化
hexo g
```

More info: [Generating](https://hexo.io/zh-cn/docs/generating.html)

### Deploy to remote sites

``` bash
hexo deploy
# 简化
hexo d
```

> **注意**：若 Github 设置自定义域名，部署之后，`CNAME` 文件会被覆盖，导致网站不可用,可以将 `CNAME` 文件放在`themes/[theme-name]/source`目录下，这样编译静态文件会自动带过去，之后会一同推送到远端

More info: [Deployment](https://hexo.io/zh-cn/docs/one-command-deployment.html)

### Clean

```bash
hexo clean
```

清除缓存文件 (db.json) 和已生成的静态文件 (public)。建议有删除文章后先 `clean`，再编译

## Change theme

hexo 有许多优秀主题可供切换，本博客使用的主题是[matery](https://github.com/blinkfox/hexo-theme-matery/blob/develop/README_CN.md)，具体修改配置参见README
