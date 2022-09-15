---
title: "Pacman 常用命令"
date: 2022-01-27T12:00:00+08:00
lastmod: 2022-01-27T12:00:00+08:00
draft: false
keywords: ["Linux", "Pacman"]
description: "Pacman 是 Arch Linux / Manjaro 默认的包管理器"
tags: ["Linux"]
categories: ["编程"]
author: "Yike Zhou"

# You can also close(false) or open(true) something for this content.
# P.S. comment can only be closed
comment: false
toc: true
autoCollapseToc: true
postMetaInFooter: false
hiddenFromHomePage: false
# You can also define another contentCopyright. e.g. contentCopyright: "This is another copyright."
contentCopyright: false
reward: false
mathjax: false
mathjaxEnableSingleDollar: false
mathjaxEnableAutoNumber: false

# You unlisted posts you might want not want the header or footer to show
hideHeaderAndFooter: false

# You can enable or disable out-of-date content warning for individual post.
# Comment this out to use the global config.
#enableOutdatedInfoWarning: false

flowchartDiagrams:
  enable: false
  options: ""

sequenceDiagrams:
  enable: false
  options: ""

---

<!--more-->
## 搜索

普通搜索，显示包名 (package name) 和描述信息。将 `-s` 换成 `-i` 可以获得更详细的信息。

```shell
$ pacman -Ss keyword
```

搜索已安装在本地的包：

```shell
$ pacman -Qs keyword
```

列出已安装的所有包：

```shell
$ pacman -Ql
```

## 安装

常用的安装命令：

```shell
$ sudo pacman -Syu keyword
```

从本地的软件包安装（通常为 `.pkg.tar.xz` 格式）：

```shell
$ sudo pacman -U packagelocation
```

## 卸载

卸载指定的软件包（不影响它的依赖）：

```shell
$ sudo pacman -R packagename
```

同时卸载不需要的依赖：

```shell
$ sudo pacman -Rsu packagename
```

## 参考资料

- Manjaro Wiki: [Pacman Overview](https://wiki.manjaro.org/index.php/Pacman_Overview)