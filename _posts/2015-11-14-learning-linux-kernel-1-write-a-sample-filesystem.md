---
layout: post
title: Linux内核学习笔记（1）文件系统
categories: Linux_kernel
tags: Linux kernel filesystem
---

在Linux下编程跟内核打交道的最频繁的方式估计就是操作文件了。谁让Linux遵循“一切皆文件”的思想呢。

因此，咱们学习Linux内核的第一步就是自己动手来写一个简单的文件系统samplefs，希望藉此能够跨入Linux内核的大门。

# 什么是文件系统

有关文件系统的概念，可以在很多操作系统的教材上看到。一般是这样介绍的：

**首先**从计算机存储的层次结构方面讲起，存储介质按访问速度（受限于介质和寻址方式）可以依次分为：

CPU内部寄存器 --> CPU内部缓存（多级） --> 主内存 --> 辅助外部存储（磁盘、flash、软盘等）

内存是按字节存储数据的，CPU直接将要访问的内存地址送到地址总线上即可，无需额外的管理。

但外部存储就不一样了，外存一般用来存放文件（也可用作其它用途，例如Linux swap分区）。不同的操作系统支持的文件类型不一样，
而且不同的文件大小也不一样，每个保存在外存上的文件必须要标记该文件的类型、大小、存放位置等等文件信息（即描述数据属性的数据，元数据）。这些元数据同样要保存在外存上。

因此，如何将文件整齐划一的保存到外存设备上，是操作系统必须要解决的问题。而这个问题的答案就是——文件系统。

## 文件系统的功能

当用户打开计算机，进入系统后，在登陆shell中会自动进入到个人目录（/home/xxx/）下。用户可以看到从根目录（/）开始的文件目录树，可以进入到任何一个目录，打开、修改或删除文件。

因此文件系统的第一个功能就是，负责文件系统中文件的访问操作。

此外，呈现给用户的文件目录信息，都是内核启动时从外部存储设备中读取出来的。而这需要首先将文件系统“安装”到这个外存设备上，在Linux中使用mkfs.xxx工具；在windows中叫做“格式化”。

在安装Linux系统时，需要对磁盘进行分区，而且必须要为“/”根目录建立分区。建立起根分区之后，就可以将其它的已安装有文件系统的设备mount到根目录结构中的某个路径（称作“mount point”）。

比如说，假定我为我的虚拟机（ubuntu 10.04）新添加了一块硬盘，第一步就是为硬盘分区（建立分区表）：

```sh
bookxiaow@ubuntu1004:~$ sudo fdisk -l /dev/sdb

Disk /dev/sdb: 2147 MB, 2147483648 bytes
255 heads, 63 sectors/track, 261 cylinders
Units = cylinders of 16065 * 512 = 8225280 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disk identifier: 0x00000000

Disk /dev/sdb doesn't contain a valid partition table

bookxiaow@ubuntu1004:~$ sudo fdisk /dev/sdb
Device contains neither a valid DOS partition table, nor Sun, SGI or OSF disklabel
Building a new DOS disklabel with disk identifier 0xc5c04c60.
Changes will remain in memory only, until you decide to write them.
After that, of course, the previous content won't be recoverable.

Warning: invalid flag 0x0000 of partition table 4 will be corrected by w(rite)

WARNING: DOS-compatible mode is deprecated. It's strongly recommended to
         switch off the mode (command 'c') and change display units to
         sectors (command 'u').
		 
Command (m for help): m
Command action
   a   toggle a bootable flag
   b   edit bsd disklabel
   c   toggle the dos compatibility flag
   d   delete a partition
   l   list known partition types
   m   print this menu
   n   add a new partition
   o   create a new empty DOS partition table
   p   print the partition table
   q   quit without saving changes
   s   create a new empty Sun disklabel
   t   change a partition's system id
   u   change display/entry units
   v   verify the partition table
   w   write table to disk and exit
   x   extra functionality (experts only)
   
Command (m for help): n
Command action
   e   extended
   p   primary partition (1-4)
p
Partition number (1-4): 1
First cylinder (1-261, default 1): 
Using default value 1
Last cylinder, +cylinders or +size{K,M,G} (1-261, default 261): 
Using default value 261

Command (m for help): p

Disk /dev/sdb: 2147 MB, 2147483648 bytes
255 heads, 63 sectors/track, 261 cylinders
Units = cylinders of 16065 * 512 = 8225280 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disk identifier: 0xc5c04c60

   Device Boot      Start         End      Blocks   Id  System
/dev/sdb1               1         261     2095458+  83  Linux

Command (m for help): w
The partition table has been altered!
```

现在我们已经建立好分区表了，并且只分了一个区/dev/sdb1，接下来就是安装ext4文件系统了

```sh
bookxiaow@ubuntu1004:~$ sudo mkfs.ext4 -c /dev/sdb1
mke2fs 1.41.11 (14-Mar-2010)
Filesystem label=
OS type: Linux
Block size=4096 (log=2)
Fragment size=4096 (log=2)
Stride=0 blocks, Stripe width=0 blocks
131072 inodes, 523864 blocks
26193 blocks (5.00%) reserved for the super user
First data block=0
Maximum filesystem blocks=536870912
16 block groups
32768 blocks per group, 32768 fragments per group
8192 inodes per group
Superblock backups stored on blocks: 
	32768, 98304, 163840, 229376, 294912

Checking for bad blocks (read-only test): done                                
Writing inode tables: done                            
Creating journal (8192 blocks): done
Writing superblocks and filesystem accounting information: done

This filesystem will be automatically checked every 30 mounts or
180 days, whichever comes first.  Use tune2fs -c or -i to override.
```

最后，就可以挂载到根目录下了

```sh
bookxiaow@ubuntu1004:~$ sudo mkdir /mnt/test
bookxiaow@ubuntu1004:~$ sudo mount -t ext4 /dev/sdb1 /mnt/test
bookxiaow@ubuntu1004:~$ cd /mnt/test/
bookxiaow@ubuntu1004:/mnt/test$ ls
lost+found
```

如果目录/mnt/test已经存在了，并且里面已经了有了文件，那么挂载之后原来的文件内容就被“隐藏”起来了，在unmount之后会再暴露出来！这个小功能是不是可以用来藏东西呢，A_A

# Linux文件系统的结构

Linux中，跟用户直接交互的是VFS（Virtual Filesystem）。它向用户提供了统一的文件操作接口（open/create/close/write/read/...），尽管某些操作在某些文件系统中根本不支持。

[VFS整体结构]（/image/VFS.jpg）



