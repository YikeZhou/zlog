---
title: "Git Subtree 简单使用"
date: 2022-09-07T12:00:00+08:00
lastmod: 2022-12-09T12:00:00+08:00
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

---

{{% admonition info "不那么好用的 Git 子模块 (submodule)" %}}
在 *Pro Git* 这本书中，子模块的功能被概括为：

“子模块允许你将一个 Git 仓库作为另一个 Git 仓库的子目录。它能让你将另一个仓库克隆到自己的项目中，同时还保持提交的独立。”

当使用了一段时间子模块之后，你可能会发现它有这样一些不便之处：
1. 在克隆主项目到本地后，子模块并不会直接出现在它的目录下，需要手动将其克隆到本地。
2. 如果主项目中不同的分支 (branch) 包含不同版本的子模块，切换分支后，需要手动同步子模块的内容。
3. 必须时常留意其他人是否修改了子模块，避免你的 commit 中子模块的版本不对。
4. 如果想在子模块上做一些修改，然后 push 到它的远程仓库，也需要万分小心，确保远程的子模块和主项目都得到了更新。

其中，一部分问题可以通过配置 `git config submodule.recurse true` 来解决。但是，子模块的操作还是比较繁琐。
{{% /admonition %}}

总之，对于作者这样的 Git 新手，子模块的使用实在是太令人头大了。所以，我发现了 subtree 之后，立刻抛弃了 submodule。

subtree 与 submodule 相比，不会创建新的 metadata 文件（.gitmodule 等），而且其它的 git repo 使用者不会发现使用 subtree 的痕迹。
git clone 的时候，所有的依赖都会一起被 clone 下来。所以用它来管理一个项目的依赖非常地方便。

{{% figure class="center" src="https://www.repstatic.it/content/localirep/img/rep-torino/2016/10/07/112008349-c2daced1-20ee-487c-98a2-ffa26fc2b400.jpg" title="double tree of Casorzo" alt="granadoubletree.jpg" %}}

# 初次使用

一开始，我只学了下面3个简单粗暴的命令：

- 添加：把远程仓库 `{remote repo URL}` 的某个分支 `{remote branch}` 添加到指定目录 `{local directory}` 下，并自动创建一个提交。
```shell
$ git subtree add --prefix {local directory} {remote repo URL} {remote branch} --squash
```

- 拉取更新：将 `add` 换成 `pull` 即可。
```shell
$ git subtree pull --prefix {local directory} {remote repo URL} {remote branch} --squash
```

- 发布更新：使用 `push` 命令即可。
```shell
$ git subtree push --prefix {local directory} {remote repo URL} {remote branch}
```

这样用了一段时间之后，突然发现 `pull` 命令不好用了，竟然提示有冲突！好不容易合并之后，发现提交历史变得非常诡异。一番查找资料后，我发现，subtree 原来只是 Git 提供的一个脚本，它其实是调用了一组 Git 的命令完成的。我们也可以手动执行这些命令（主要是 `merge`、`cherry-pick` 和 `read-tree`）。

{{% admonition quote "底层命令与上层命令" %}}
由于 Git 最初是一套面向版本控制系统的工具集，而不是一个完整的、用户友好的版本控制系统，所以它还包含了一部分用于完成底层工作的子命令。这些命令被设计成能以 UNIX 命令行的风格连接在一起，抑或藉由脚本调用，来完成工作。这部分命令一般被称作“底层（plumbing）”命令，而那些更友好的命令则被称作“上层（porcelain）”命令。
{{% /admonition %}}



# 更多命令

现在，我有了一个更复杂的需求：
1. 自己的仓库 my-repo 依赖 GitHub 上一个开源的仓库 that-repo；
2. 自己在本地对开源工具进行修改；
3. 同时需要拉取远程仓库的更新到本地。

{{% admonition tip "提示" %}}
在测试时，可以在本地创建2个目录，模拟本地仓库和远程仓库。

```
.
├── my-repo
└── remotes
    └── that-repo
```
{{% /admonition %}}

## 添加一个 subtree

上一节中用到的 `git subtree` 是 Git 提供的一个捷径。这里我们使用手动添加的方法，看看需要进行哪些操作。

首先，可以将子项目的 URL 记录为一个 remote repo，避免每次都重复输入长长的地址。在 `my-repo` 目录下执行：

```shell
$ git remote add -f that-repo ../remotes/that-repo
```

其中 `-f` 表示同时对它执行 `fetch` 命令，所以也可以这样写：

```shell
$ git remote add that-repo ../remotes/that-repo
$ git fetch that-repo
```

下一步，需要将 that-repo 中的内容放入 my-repo 的一个子目录 `that-repo/` 中，作为一个 subtree。这一步实际上得进行2个操作：
1. 向 `./my-repo/that-repo` 中添加来自 that-repo 的文件。
2. 修改 my-repo 仓库的 index（保存暂存区信息的文件）。

首先使用 `git merge`，其中 `-s ours` 指定了合并的策略。
接着使用 [`git read-tree`](https://git-scm.com/docs/git-read-tree) 命令，它会读取一个分支的根目录树到当前的暂存区和工作目录里：

```shell
$ git merge -s ours --no-commit --allow-unrelated-histories that-repo/main
$ git read-tree --prefix=that-repo -u that-repo/main
```

完成后，执行 `git status` 命令可以观察到 my-repo 仓库中出现了来自 that-repo 的新文件。我们可以提交这一更改到 my-repo。

```shell
$ git commit -m 'add that-repo as subtree'
```

这样做有一个缺点，就是会将 that-repo 的所有分支提交历史都添加到 my-repo 的历史中。

## 拉取远程仓库的更新

现在 that-repo 远程仓库更新了，my-repo 也新增了一些不相关的提交（没有修改 that-repo），我们需要更新本地仓库 my-repo 中的版本。

```shell
$ git fetch that-repo
$ git merge -s subtree that-repo/main
```

`-s subtree` 将合并策略设置为“子树合并”，这样做将把 that-repo 中新增的修改添加到 my-repo 中。

也可以用一行命令完成：

```shell
$ git pull -s subtree that-repo main
```

如果在 my-repo 中已经对 that-repo 进行了修改，前一种方法可以正常工作，后一种可能会报错 `fatal: Need to specify how to reconcile divergent branches.`——这个问题 Google 一下可以找到很多解释，这里不再赘述。

## 在 my-repo 中修改子模块

当你想对 that-repo 进行的修改和 my-repo 有关，并且这个修改不需要 push 到远程的 that-repo 上时，subtree 就比 submodule 好用多了。

将 my-repo 中的提交分为下面4种：
1. 仅对 that-repo 进行了修改，并且需要推送到远程 that-repo 仓库（例如 bug-fix）。
2. 在 my-repo 上进行的修改，与 that-repo 无关。
3. 对 my-repo 和 that-repo 都进行了修改，也需要推送到 that-repo 仓库。
4. 对 that-repo 的修改，不需要同步到远程 that-repo 仓库。

如果只使用 `git subtree` 操作，则不能保留一部分不需要推送的修改。这个需求要用到 `git cherry-pick` 命令，并且最好用另一个分支维护 `that-repo`。
首先创建一个新的分支：

```shell
$ git checkout -b backport-that-repo that-repo/main
```

下面的两条指令分别将第1种和第3种提交应用到 `backport-that-repo` 分支上，修改完成后，可以直接在这一分支上调用 `git push`。

```shell
$ git cherry-pick -x <commit>
$ git cherry-pick -x --strategy=subtree <commit>
```

## 精简提交历史

有时候，我们不需要在 my-repo 中保留 that-repo 全部的分支提交历史，这时可以使用 `--squash` 选项。

# 参考资料

- [Git Subtree Basics](https://gist.github.com/SKempin/b7857a6ff6bddb05717cc17a44091202)
- [About Git subtree merges](https://docs.github.com/en/get-started/using-git/about-git-subtree-merges?platform=linux)
- [Git subtree: the alternative to Git submodule](https://www.atlassian.com/git/tutorials/git-subtree)
- [Mastering Git subtrees](https://medium.com/@porteneuve/mastering-git-subtrees-943d29a798ec)
- [Pro Git book](https://git-scm.com/book/zh/v2)
- [git-read-tree - Reads tree information into the index](https://git-scm.com/docs/git-read-tree)
- [Git subtree用法与常见问题分析](https://juejin.cn/post/6881580754854215687)