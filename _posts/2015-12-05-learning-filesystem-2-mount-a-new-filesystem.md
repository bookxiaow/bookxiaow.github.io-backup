---
layout: post
title: Linux内核学习之文件系统(三) 挂载新的文件系统
categories: Linux内核之文件系统
tags: kernel filesystem
---

使用mount工具可以将安装有某种文件系统的设备挂载到系统根目录下的某个挂载点上。

> 准确地说，并不一定是系统根目录，而是当前登录shell所在的namespace的根目录

mount工具对应的系统调用也是mount( man 2 mount)，其对应的内核实现是`sys_mount`。

```c
/*
 * mount - 挂载设备到某个目录下
 * @dev_name - 设备名称，例如"/dev/sdb1"
 * @dir_name - 挂载点，例如"/mnt/testfs"
 * @type - 文件系统类型，例如"ext4"，必须是已经注册到内核的（cat /proc/filesystems）
 * @flags - 挂载标志
 * @data - private data, can be passed to super
 */
SYSCALL_DEFINE5(mount, char __user *, dev_name, char __user *, dir_name,
		char __user *, type, unsigned long, flags, void __user *, data)
{
	//
}
```

`sys_mount`首先将参数从用户空间拷贝到临时内核空间，然后调用`do_mount`；待`do_mount`返回后释放内核临时缓冲区并返回。

`do_mount`实现了具体的挂载操作。这里先不深究函数的具体实现。我们先来搞清楚挂载操作发生的几种可能情况：

1. 挂载点不存在怎么办？
2. 挂载点并不是空目录，甚至已经是其它文件系统的挂载点了怎么办？




