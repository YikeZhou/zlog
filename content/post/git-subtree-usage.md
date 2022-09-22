---
title: "Git Subtree 简单使用"
date: 2022-09-07T12:00:00+08:00
lastmod: 2022-09-07T12:00:00+08:00
draft: false
keywords: ["Git"]
description: "git subtree 命令的使用方法"
tags: ["Git"]
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
git subtree 与 submodule 相比，不会创建新的 metadata 文件（.gitmodule 等），而且其它的 git repo 使用者不会发现使用 subtree 的痕迹。
git clone 的时候，所有的依赖都会一起被 clone 下来。所以用它来管理一个项目的依赖非常地方便。

{{% figure class="center" src="https://www.repstatic.it/content/localirep/img/rep-torino/2016/10/07/112008349-c2daced1-20ee-487c-98a2-ffa26fc2b400.jpg" title="double tree of Casorzo" alt="granadoubletree.jpg" %}}

## tl;dr

- 添加：下面的命令把 remote repo 添加到指定目录下，并自动 commit
```shell
$ git subtree add --prefix {local directory} {remote repo URL} {remote branch} --squash
```

- 拉取更新：将 `add` 换成 `pull` 即可
```shell
$ git subtree pull --prefix {local directory} {remote repo URL} {remote branch} --squash
```

- 发布更新：使用 `push`
```shell
$ git subtree push --prefix {local directory} {remote repo URL} {remote branch}
```

## 参考资料

- [Git Subtree Basics](https://gist.github.com/SKempin/b7857a6ff6bddb05717cc17a44091202)