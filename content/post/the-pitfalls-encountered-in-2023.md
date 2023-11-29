---
title: "2023 年踩过的那些坑（未完待续）"
date: 2023-06-17T13:43:00+08:00
lastmod: 2023-10-13T18:00:00+08:00
draft: false
keywords: ["Troubleshooting", "Linux", "Python", "Docker", "WSL"]
description: "各种各样的问题及其解决方案记录"
tags: ["Linux"]
categories: ["编程"]
author: "Yike Zhou"

comment: false
toc: true
autoCollapseToc: false
postMetaInFooter: true

---

## WSL

{{% admonition warning "磁盘访问十分缓慢" %}}
把项目放在 WSL 自己的文件系统里。
{{% /admonition %}}

{{% admonition warning "进行大规模编译时无响应" %}}
结合 GitHub 上的讨论，初步判断是内存不足导致的问题。
根据[官方文档](https://learn.microsoft.com/en-us/windows/wsl/wsl-config)上给出的方法调低了分配给 WSL 的处理器核数量，间接降低了并行编译时占用的内存，从而避免了因为内存占用过高导致的 WSL 无响应问题。
{{% /admonition %}}

{{% admonition quote "参考资料" %}}
- [Advanced settings configuration in WSL](https://learn.microsoft.com/en-us/windows/wsl/wsl-config)
- [Docker Desktop WSL 2 backend on Windows](https://docs.docker.com/desktop/wsl/)
- [Best practices](https://docs.docker.com/desktop/wsl/best-practices/)
- [Discussions on "Hacker News"](https://news.ycombinator.com/item?id=28487374)
- [How to solve the problems caused by WSL 2's filesystem changes?](https://superuser.com/questions/1594279/how-to-solve-the-problems-caused-by-wsl-2s-filesystem-changes)
- **Issues found on GitHub**:
  - [WSL 2 freezing with 100% disk usage after windows store update](https://github.com/microsoft/WSL/issues/9383)
  - [WSL Becomes Non Responsive, Uses High Amounts Of CPU & Memory](https://github.com/microsoft/WSL/issues/9429)
  - [WSL2 consume huge RAM and don't free them, also consume hug DISK I/O](https://github.com/microsoft/WSL/issues/9906)
{{% /admonition %}}


## Docker

{{% admonition question "Smaller Python images?" %}}
Python Image 太大了，希望能尽量小一些，方便上传到服务器上，就看到了 [Smaller python docker containers](https://samroeca.com/docker-python-install-wheels.html) 这篇文章。
根据里面介绍的方法，在前一个 stage 里面编译好 wheels，然后从 wheels 安装第三方库，大部分依赖都可以成功安装，然而 z3 就是不行。后来发现是 Alpine Linux 使用的 libc 比较特殊的原因。
[Don’t use Alpine Linux for Python images](https://pythonspeed.com/articles/alpine-docker-python/) 介绍了 Alpine Linux 的缺点。
{{% /admonition %}}


## Python

{{% admonition bug "Debug with gdb" %}}
主要参考这几个手册：
- [Python Developer’s Guide - GDB support](https://devguide.python.org/development-tools/gdb/)
- [DebuggingWithGdb](https://wiki.python.org/moin/DebuggingWithGdb)
- [Features/EasierPythonDebugging](https://fedoraproject.org/wiki/Features/EasierPythonDebugging)
- [Debugging Programs with Multiple Threads](https://sourceware.org/gdb/download/onlinedocs/gdb/Threads.html)

不过它们没提到 Python in Docker 要怎么配置，经我尝试发现大致需要这几步：
1. 安装这几个包：`gdb python3-all python3-dbg python3-pip procps`；
2. 运行 Docker container 的时候加上 `--cap-add=SYS_PTRACE` 选项；
3. 使用 `sudo gdb -p {PID}` 来 debug 正在运行的进程。

还有其它一些可能的问题：
1. 系统本身可能有自带的 Python，与手动安装的版本不同。这时 pip 可能也有多个版本，默认在不同的地方安装第三方库，需要格外注意。
2. 运行 gdb 时提示 `Operation not permitted` 可能是因为没有打开 `--cap-add=SYS_PTRACE` 或者使用 `sudo`。
{{% /admonition %}}

{{% admonition question "search() vs. match()" %}}
其实[手册里](https://docs.python.org/3/library/re.html#search-vs-match)讲得很清楚了，但是用了这么久竟然一直都不知道。简而言之：
- `match()` 是从字符串**开头**匹配；
- `search()` 可以从任意位置开始；
- `fullmatch()` 是检查整个字符串是否匹配。

因此，`match()` 成功可能是匹配上了字符串的前半部分，使用时要注意。
{{% /admonition %}}

## Linux

{{% admonition info "cgroup change of group failed" %}}
因为电脑性能有限，所以希望用 cgroup 来限制某进程及其子进程占用的资源。看了各种教程，每到用 cgexec 运行新进程这一步就会报错。Google 了各种别人的回答，下面是两个最有用的：
- [Andy Pan's Answer to *How to run cgexec without sudo as current user on Ubuntu 22.04 with cgroups v2, failing with "cgroup change of group failed"?*](https://askubuntu.com/a/1450845)
- [Mike N.'s Answer to *Using cgroups v2 without root*](https://unix.stackexchange.com/a/741631)

按照 [Linux Kernel Docs](https://docs.kernel.org/admin-guide/cgroup-v2.html#delegation-containment) 里面所写：

> The writer must have write access to the "cgroup.procs" file of the common ancestor of the source and destination cgroups.

我们需要找到这个公共祖先，再看它对应的目录下 cgroup.procs 文件是否有写权限。

1. 通过 `cat /proc/self/cgroup` 命令找到当前 Shell 进程的 cgroup：user.slice/**user-1000.slice**/session-1.scope；
2. 已知创建的新 cgroup 放在 user.slice/**user-1000.slice**/user\@1000.service/app.slice/ 中。

二者的公共祖先是 **user-1000.slice**，使用 `ls -l /sys/fs/cgroup/user.slice/user-1000.slice/cgroup.procs` 确认权限。
```
-rw-r--r-- 1 root root ... (omitted)
```

确实是普通用户没有写权限造成的问题，所以可以用上面回答中给出的方法，手动修改权限。
```
# chmod o+w /sys/fs/cgroup/user.slice/user-1000.slice/cgroup.procs
```

虽然问题解决了，但是让人奇怪的是，为什么 user-1000.slice/ 里面的 cgroup.procs 文件所有者不是 uid=1000 的普通用户，而它的子目录 user\@1000.service/ 与 app.slice/ 中的此文件所有者又是 uid=1000 的普通用户？

**其它有用的页面**：
- [cgroups - ArchWiki](https://wiki.archlinux.org/title/cgroups)
- [Ubuntu Manpage: cgconfig.conf - libcgroup configuration file](https://manpages.ubuntu.com/manpages/jammy/en/man5/cgconfig.conf.5.html)
- [Resource Management Guide Red Hat Enterprise Linux 6 | Red Hat Customer Portal](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/6/html/resource_management_guide/index)
- [Control Group v2 — The Linux Kernel documentation](https://docs.kernel.org/admin-guide/cgroup-v2.html)
{{% /admonition %}}
