---
layout: post
title:  "Linux CGroup 备忘"
categories: Linux
---
# Linux CGroup

## 简介

CGroup是Control Group的缩写，是Linux内核的提供的特性，用于限制、说明和隔离一组进程对CPU、硬盘I/O、网络带宽资源的使用。这些资源在内核中都有各自的子系统对其进行管理，CGroup是通过与内核各个子系统配合来完成对资源的控制的。

## 原理

Linux内核进程结构体task_struct中存在相关成员变量（cgroup相关结构体）来与cgroup建立关联。内核初始化时通过参数（`CONFIG_*_CGROUP`等）决定task_struct结构体中实际具体包含的cgroup结构体。

cgroup结构体可以组织成一棵树的形式，每一颗cgroup结构体组成的树称之为一个cgroups层级结构。cgroups层级结构可以attach到N个cgroup资源子系统中，当前层级结构可以对其attach的cgroup资源子系统进行资源的限制。

基于上述描述作为基础，Linux通过将进程Id添加到cgroup层级结构中某一个节点的进程控制列表中，来实现对该进程对相应资源使用的控制。

Linux基于VFS，实现了cgroup文件系统。通过用户对cgroup文件系统的操作，在用户态项用户开放了cgroup的具体功能，内核再通过读取相关数据结构的更新，与具体的某资源子系统进行交互，实现对资源的控制的目的。

## 应用

- 查看启用Control Group的资源子系统。`cat /proc/cgroups` `ls -l /sys/fs/cgroup/`
- 在某个具体类型（比如memory）资源Cgroup层级结构下创建一个文件夹，cgroup文件系统会自动在其下面生成一系列相关文件，向memory.limits文件写入限制大小，memory.proc中写入受控制的进程ID，实现对进程的相应类型的资源控制。

## 启示

Linux对CPU、Memory、Disk I/O等资源的限制，需要在用户态设置相关限制值，然而资源的使用限制又需要在内核中实现，那么就需要提供一种机制将用户态的数据送入内核，当然也可以通过增加系统调用的形式，与内核进行交互。但是如果通过文件系统来实现，不仅可以解决与内核交互的问题，同时文件系统作为层级结构的数据结构，可以比较便捷的实现对资源使用限度的层级控制，使得与资源的分配更加便于控制也更直观。从CGroup的实现，我们也可以学习到层级结构对于解决精细化控制，统筹，统计等这类场景时的优势。