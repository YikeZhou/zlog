---
title: "使用 WSL 过程中遇到的坑"
date: 2023-06-17T13:43:00+08:00
lastmod: 2023-06-17T14:00:00+08:00
draft: false
keywords: ["WSL", "Troubleshooting"]
description: "WSL 使用上的问题及其解决方案记录"
tags: ["Windows", "WSL"]
categories: ["编程"]
author: "Yike Zhou"

comment: false
toc: true
autoCollapseToc: false
postMetaInFooter: true

---

## 问题记录

### (1) 磁盘访问十分缓慢

把项目放在 WSL 自己的文件系统里。

### (2) 进行大规模编译时无响应

结合 GitHub 上的讨论，初步判断是内存不足导致的问题。
根据[官方文档](https://learn.microsoft.com/en-us/windows/wsl/wsl-config)上给出的方法调低了分配给 WSL 的处理器核数量，间接降低了并行编译时占用的内存，从而避免了因为内存占用过高导致的 WSL 无响应问题。

## 参考资料

- [Advanced settings configuration in WSL](https://learn.microsoft.com/en-us/windows/wsl/wsl-config)
- [Discussions on "Hacker News"](https://news.ycombinator.com/item?id=28487374)
- [How to solve the problems caused by WSL 2's filesystem changes?](https://superuser.com/questions/1594279/how-to-solve-the-problems-caused-by-wsl-2s-filesystem-changes)
- **Issues found on GitHub**:
  - [WSL 2 freezing with 100% disk usage after windows store update](https://github.com/microsoft/WSL/issues/9383)
  - [WSL Becomes Non Responsive, Uses High Amounts Of CPU & Memory](https://github.com/microsoft/WSL/issues/9429)
  - [WSL2 consume huge RAM and don't free them, also consume hug DISK I/O](https://github.com/microsoft/WSL/issues/9906)