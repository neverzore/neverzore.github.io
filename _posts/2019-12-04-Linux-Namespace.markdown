---
layout: post
title:  "Linux Namespace"
date:   2019-12-04 19:09:35 +0800
categories: Linux
---
# Linux Namespace

## 简介

Linux Namespace是**Linux内核**层面的一个特性，在进程层面对其可见的内核资源进行不同Namespace的进程间**隔离**。Linux Namespace主要目的是实现**轻量级进程虚拟化**服务。Linux Namespace的类型包括**Mount、PID、User、UTS、Network、IPC、Cgroup**，其余还有Security、Security Keys、Device等未实现的类型。Linux Namespace控制隔离的资源是操作系统层面的**虚拟资源**，还有另一种Cgroup机制控制CPU、Disk物理类型资源。

## 原理

内核进程结构体**task_struct**包含结构体**nsproxy成员变量**，对于共享各个Namespace的进程间nsproxy结构体也是共享的。

nsproxy结构体包含指向每个进程的各种类型的namespace的指针，一旦某个类型的namespace被clone或者unshare系统调用，nsproxy会被内核**复制传递**。nsproxy结构体内部还包含一个原子操作类型的count字段，标识其持有的进程引用的数量。nsporxy结构体中的各个类型的namespace结构体，各自内部也有引用计数变量，该计数变量受打开的进程、打开的namespace文件（ /proc/<pid>/ns/<ns-kind> ）、挂载绑定的namespace文件影响，该变量用于标识该namespace是否可以被内核回收。

上述数据结构是内核对某一虚拟资源不同Namespace下的进程间的隔离控制实现的基础。

不同namespace下的进程间由于其该类型资源间的相对独立隔绝，使得进程相应资源的操作对namespace外的进程**影响受限**，之所以说受限，是因为当资源本身在不同namespace间共享时，不同namespace中进程对该共享资源的操作仍会影响其他namespace下的进程。

## 应用

Linux 对外提供了3个与Namespace相关的系统调用，简要描述如下：

- clone，通过clone系统调用传入不同的标识参数来明确新进程需要处理的具体类型namespace。
- unshare，允许某个进程在其它进程共享的执行上下文中与某个具体类型的namepsace隔离。
- setns，进入某个namespace通过传入具体的文件描述符。

通过上述系统调用，Linux内核开放构建**虚拟化的能力**给到用户态在用户空间进行调用。

## 启示

Linux Namespace 针对内核资源调度单位（进程），通过提供执行环境副本的形式，隔离了不同Namespace空间下进程间的相互操作、影响，不仅提供容器化的基础，在应用安全方面也存在价值。

