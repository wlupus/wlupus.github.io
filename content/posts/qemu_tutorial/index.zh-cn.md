---
title: "使用QEMU创建Linux虚拟机"
subtitle: ""
date: 2023-03-02T11:58:45+08:00
draft: false
author: ""
authorLink: ""
description: "在macos下搭建linux环境的一点记录"
keywords: ""
license: ""
comment: false
weight: 0

tags:
- "qemu"
- "linux"
categories:
- "笔记"

hiddenFromHomePage: false
hiddenFromSearch: false

summary: ""
resources:
- name: featured-image
  src: \@heripiro-FiE2lirUYAAStgo.jpg
- name: featured-image-preview
  src: preview.png

toc:
  enable: true
math:
  enable: true
lightgallery: true

# See details front matter: /theme-documentation-content/#front-matter
---

<!--more-->

## 1 安装qemu
使用包管理器安装即可
```
brew install qemu
```
qemu有两种工作模式：system emulation & user mode emulation, 本文中使用system emulation的模式

## 2 创建硬盘镜像 
硬盘镜像是一个文件，用来存储虚拟机硬盘上的内容。qemu支持的镜像有两种格式，分别是`raw`和`qcow2`。raw格式I/O开销小，但是占用全量空间(分配大小即为实际大小)；qcow2格式仅当系统实际写入内容时，才会分配空间，另外还支持快照。这里采用qcow2格式
```
qemu-img create -f qcow2 server_file 60G
```

## 3 准备安装介质
下载ubuntu18.04 server版本的iso镜像，放到image file相同路径下。之所以选择server版本，是因为在m1 mac下运行x86的虚拟机效率很低，带图形化界面会很卡，基本难以使用
```
ls
server_file  ubuntu-18.04.6-live-server-amd64.iso
```

## 4 安装操作系统
```
qemu-system-x86_64 -smp 4\
    -m 8G\
    -cdrom ubuntu-18.04.6-live-server-amd64.iso\
    -boot order=d\
    -drive file=server_file,format=qcow2
```
以`qemu-system-*`格式命名的程序有很多，其中`*`处代表的是客户机(Guest)对应的架构。本文中是在arm平台下起x86的虚拟机，所以选择`qemu-system-x86_64`

命令中涉及的部分参数含义：
- `-smp` 指定cpu数量
- `-m` 指定内存数量
- `-boot` 其中有二级参数，order参数指定启动介质，可选值有a, b(floppy 1 and 2), c(first hard disk), d(first CD-ROM)
- `-drive` 定义一个新的drive(或者blockdev)，二级参数file指定该设备使用的硬盘镜像

更多参数相关请参考[qemu文档](https://www.qemu.org/docs/master/system/invocation.html)

## 5 运行系统
安装完成后需要重启，此时将cdrom的挂载去掉，并将`-boot`参数去掉即可(默认从硬盘启动)
```
qemu-system-x86_64 -smp 4\
    -m 8G\
    -drive file=server_file,format=qcow2
```
如果不想看到qemu弹出的虚拟机窗口，可以添加`-display none`参数

## 6 端口映射
为了方便ssh连接虚拟机，采用将guest的22端口映射到host的端口的方法
```
qemu-system-x86_64 -smp 4\
    -m 8G\
    -drive file=server_file,format=qcow2\
    -nic user,hostfwd=tcp::10022-:22\
    -display none
```
添加参数`-nic`，user指config user mode host network backend, `hostfwd=[tcp|udp]:[hostaddr]:hostport-[guestaddr]:guestport`

如果guest需要使用host的代理服务，可以配置如下环境变量
```
export http_proxy=http://ip_addr:port
export https_proxy=http://ip_addr:port
# ip_addr 为host的ip地址，port为host代理服务的端口号
```
{{<admonition tip>}}
qemu中默认映射的host ip地址为10.0.2.2
{{</admonition>}}