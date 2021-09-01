---
title: 使用 gihub-action 发布 go release
top: false
cover: false
toc: true
mathjax: false
date: 2021-08-31 18:03:53
password:
summary:
categories: tools
tags:
- github
- Golang
- workflow
- ci
---

当需要自动编译并发布release程序时，可以使用 github action 的工作流来实现

## 创建工作流脚本

新建yml脚本 `.github/workflows/release.yml`,

```yml
# 工作流名称
name: Go Release

# 触发条件
on:
  push:
    # 创建 tag 时
    tags:
    - v*
    # 推送到 master 分支时
    branchs:
    - master

jobs:
  release:
    # 运行环境
    runs-on: ubuntu-latest
    steps:
    # 切换到对应 tag 源码
    - name: Checkout
      uses: actions/checkout@v2
      with:
        fetch-depth: 0
    # 安装 Go
    - name: Set up Go
      uses: actions/setup-go@v2
      with:
        go-version: 1.17
    # 使用 goreleaser 编译 release
    - name: Create release on GitHub
      uses: goreleaser/goreleaser-action@v2
      with:
        # GoReleaser 版本
        version: latest
        # 传递给 GoReleaser 的参数
        args: release --rm-dist
      env:
        # 提供访问仓库token
        GITHUB_TOKEN: ${{secrets.GITHUB_TOKEN}}
```

这里使用 [goreleaser](https://goreleaser.com/) 提供的多平台二进制文件构建 github action，可以更方便的编译多平台版本, 详细信息请阅读 [GoReleaser Action Usage](https://github.com/goreleaser/goreleaser-action#usage)

### 创建配置文件

使用该 action 还需要提供配置文件 `.goreleaser.yaml`，放置在仓库根目录下，若不提供配置文件，则会根据[默认的配置文件](https://github.com/goreleaser/goreleaser/blob/master/.goreleaser.yml)，编译所有平台的版本，以及 docker 等编译流程，有可能会出错。因此，根据自己需要，参考默认配置文件，选择适合自己的配置，确保工作流正常执行
