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
