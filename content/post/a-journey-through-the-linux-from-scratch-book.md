---
title: "第一次体验 Linux From Scratch (LFS)"
date: 2023-11-29T19:26:57+08:00
lastmod: 2023-11-29T19:26:57+08:00
draft: false
keywords: ["Linux"]
description: "A Journey Through the Linux From Scratch Book"
tags: ["Linux"]
categories: ["编程"]
author: "Yike Zhou"

comment: false
toc: true
autoCollapseToc: false
postMetaInFooter: true

flowchartDiagrams:
  enable: false
  options: ""

sequenceDiagrams:
  enable: false
  options: ""

---

Linux From Scratch (LFS) 是一个以**手册**的形式存在的 Linux 发行版，它提供了一组软件包（源码）供用户下载，按照手册中的指令一步步操作，就可以获得一个自己编译出来的 Linux 系统。
本文记录了作者在自己动手的过程中遇到的问题和解决方法。

编译 LFS 的过程可以[分为 3 个部分](https://www.linuxfromscratch.org/lfs/view/stable/prologue/organization.html)：
- 准备工作：创建分区、下载软件包等
- 准备“交叉编译”需要的编译器和必要的辅助工具
- 编译各种软件和 Linux 内核

## 第一部分：在主机 (Host) 上做的准备

在 LFS 准备好之前，编译是在 Host 上进行的，需要 Host 本身具有 GCC 等工具。
（作者一开始是用 Ubuntu Live CD 作 Host 的，后面转移到了 Manjaro 上。）

### 安装必需的工具

对于 Ubuntu Live CD，需要安装 `build-essential bison texinfo gawk` 这几个软件包。同时，

> /bin/sh should be a symbolic or hard link to bash

这个在 Ubuntu 里可以[通过 dpkg 来设置](https://unix.stackexchange.com/a/442517)：

```shell
sudo dpkg-reconfigure dash
```

### 创建新的硬盘分区

为了方便，作者直接使用了 GParted 这个 GUI 工具，它的操作非常简单，并且支持 resize 现有分区。
作者的硬盘上本身安装了 Manjaro Linux 这个系统，它的 /boot/efi 和 Swap 分区都可以和 LFS 共用，所以只需要创建一个新分区用于 LFS 的根目录。

创建好后，将根目录对应的分区挂载到 `$LFS` 路径下。
Swap 分区可能正在被 Host 使用，如果执行 `swapon` 命令，会得到 `swapon failed: Device or resource busy` 的错误信息。实际上，挂载 Swap 分区在这里不是必须的。

作者从 [USTC mirror](https://mirrors.ustc.edu.cn/lfs/lfs-packages/) 上下载了 `lfs-packages-12.0.tar` 这个压缩包，里面包含了所有的软件包，还有手册中提到的 [Patches](https://www.linuxfromscratch.org/lfs/view/stable/chapter03/patches.html)，这些 Patches 是成功编译所必需的。
最新的 [Security Advisories](https://www.linuxfromscratch.org/lfs/advisories/12.0.html) 对应的 Patches 则需要额外下载。

### 在根目录中创建文件夹

创建好后，`$LFS` 目录中将会出现这些文件夹：

```
[bin]  etc  [lib]  lib64  [sbin]  sources  tools  usr  var
```
`[]` 表示符号链接，`/bin` 实际上指向 `/usr/bin`，另外两目录 `/lib` 和 `/sbin` 也是这样。

### 添加临时用户

这里添加的是 **Host 系统**中的用户，并不是 LFS 系统上的，添加这个非 root 用户是为了编译下面几个工具的时候避免用 root 用户导致潜在的风险。
同时，手册上还为这个临时用户定义了一些有用的变量：

```
LFS=/mnt/lfs
LC_ALL=POSIX
LFS_TGT=$(uname -m)-lfs-linux-gnu
```

## 第二部分：准备 LFS 系统中的编译工具

这一部分编译的软件包是下一步——构造正式系统中的软件——所必须的，主要就是 GCC 和 Glibc。

首先，我们是在 Host 系统上编译 LFS 中的 GCC，虽然 Host 和 LFS 实际上可以共用同一套编译工具（都是 x86_64 架构和 Linux 系统），但是手册故意把它们视为两个不同的平台，并且给 LFS 设置了自己的 Triplet。
交叉编译过程是这样的：

| **Stage** | **Build** | **Host** | **Target** | **Action**                                   |
|:---------:|:---------:|:--------:|:----------:|----------------------------------------------|
|     1     |     pc    |    pc    |     lfs    | Build cross-compiler cc1 using cc-pc on pc.  |
|     2     |     pc    |    lfs   |     lfs    | Build compiler cc-lfs using cc1 on pc.       |
|     3     |    lfs    |    lfs   |     lfs    | Rebuild and test cc-lfs using cc-lfs on lfs. |

其次，GCC 和 Glibc 存在相互依赖的关系：Glibc 本身需要被编译，从而需要 GCC；GCC 需要的 libgcc 和 libstdc++ 又需要链接 Glibc。

为了解决这个先有鸡还是先有蛋的问题，需要进行多次编译：
- 首先，用 cc1 构建一个功能不完全的 libgcc，它缺少线程和异常处理等功能。
- 然后，使用这个功能不完善的 libgcc 构建 Glibc（Glibc 本身是完整的），并且构建 libstdc++（缺少一些 libgcc 的功能）。
- 最后，重新构建具有完整功能的 libgcc 和 libstdc++。

## 第三部分：编译 LFS 系统中的所有软件

这一部分非常枯燥，就是按照手册中给的指令一一编译安装几十个软件。由于作者的 Manjaro Linux (Host System) 有 GRUB，可以不必在 LFS 里面再次安装 GRUB 了。
编译完实在太累了，直接跳过了 [Stripping](https://www.linuxfromscratch.org/lfs/view/stable/chapter08/stripping.html) 这个步骤。

为了系统能够正常工作，还需要进行一些设置。

### 设备管理

这里有一些关于网络和键盘的设置，由于作者压根没准备上网，所以根本没仔细看，只生成了 `/etc/hosts` 和 `/etc/resolv.conf` 这两个文件，文件的具体内容参考了 Host 上的这两个文件。

Linux Console 的设置手册没有直接给出（可能是因为对于英文键盘不需要什么额外的配置）。
由于作者用的是 U.S. Keyboard，所以没有设置 FONT 和 KEYMAP 这些，只是添加了 UNICODE 选项。

```
# Begin /etc/sysconfig/console
UNICODE="1"
# End /etc/sysconfig/console
```

### 创建 /etc/fstab 文件

这个文件是用来获取默认挂载的分区的，手册已经给出了一个例子。
除了根目录和 Swap 分区外，作者还添加了自己的 /boot/efi 到这个文件中（因为看到 Host 的这个文件中出现了 /boot/efi）。
设置好后是下面这样：

```
# Begin /etc/fstab

# file system  mount-point    type     options             dump  fsck
#                                                                order

/dev/nvme0n1p1 /boot/efi      vfat     umask=0077          0     2
/dev/nvme0n1p3 /              ext4     defaults            1     1
/dev/nvme0n1p4 swap           swap     pri=1               0     0
proc           /proc          proc     nosuid,noexec,nodev 0     0
sysfs          /sys           sysfs    nosuid,noexec,nodev 0     0
devpts         /dev/pts       devpts   gid=5,mode=620      0     0
tmpfs          /run           tmpfs    defaults            0     0
devtmpfs       /dev           devtmpfs mode=0755,nosuid    0     0
tmpfs          /dev/shm       tmpfs    nosuid,nodev        0     0
cgroup2        /sys/fs/cgroup cgroup2  nosuid,noexec,nodev 0     0

# End /etc/fstab
```

### 编译 Linux 内核

这里需要根据手册的指示，在 menuconfig 中修改编译选项，最后生成一个 `.config` 文件。
由于我们希望通过 **UEFI** 模式启动，所以除了 LFS 中给出的配置外，还需要按照 [BLFS](https://www.linuxfromscratch.org/blfs/view/12.0/postlfs/grub-setup.html) 中给出的配置进行修改。

### 使用 GRUB 配置启动引导

首先，LFS 中的 [GRUB](https://www.linuxfromscratch.org/lfs/view/stable/chapter08/grub.html) 还有后面的[启动设置](https://www.linuxfromscratch.org/lfs/view/stable/chapter10/grub.html)都是针对 BIOS 模式的，如果要使用 UEFI 模式，需要按照 BLFS (Beyond Linux From Scratch) 手册的[对应内容](https://www.linuxfromscratch.org/blfs/view/12.0/postlfs/grub-setup.html)编译和配置 GRUB。

其次，可以直接用 Host System 上的 GRUB 进行配置，将 LFS 添加到启动菜单中。有了 `os-prober` 工具的帮助，只需要执行一条指令即可完成引导配置。

```shell
grub-mkconfig -o /boot/grub/grub.cfg
```

{{% admonition type="note" title="2023-11-27 21:30" %}}
最后一步：重启电脑。

——成功进入系统了！
{{% /admonition %}}
