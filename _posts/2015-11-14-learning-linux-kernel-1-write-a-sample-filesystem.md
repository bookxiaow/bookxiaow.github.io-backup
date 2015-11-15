---
layout: post
title: Linux内核之文件系统学习笔记（1）认识文件系统
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

在上面的图中，一个磁盘文件必定位于某个安装有特定文件系统（超级块）的分区中，而且对应一个 inode 结构；但是一个inode可能会对应多个目录项dentry，比如说硬链接（注意不是cp）。
对一个目录项，不同的进程可能会同时打开它，因此也就产生了多个`struct file`结构，这样不同的进程就可以设置自己的打开方式和读写偏移量。在某些情况下，也会有同个进程或不同进程内
的多个fd指向同一个`struct file`结构，比如说`dup()/dup2()`或`fork()`之后父子进程。

##进程中的文件表示

对进程而言，一个打开的文件对应于一个进程内唯一的整数fd。这个fd必然是某个数组的索引值。

看一下`struct task_struct`的内容：

```c
struct task_struct {
	/* open file information */
	struct files_struct *files;
};
```

继续看`struct files_struct`的定义：

```c
struct files_struct {
	atomic_t count;
	struct fdtable *fdt;
	struct fdtable fdtab;
	fd_set close_on_exec_init;
	fd_set open_fds_init;
	struct file * fd_array[NR_OPEN_DEFAULT]; /*on i386, NR_OPEN_DEFAULT is 32*/
	spinlock_t file_lock; 
};
```

这里有个 fd_array数组，其元素正是指向`struct file`的指针。因此，可以猜测进程会用fd作索引去这个数组里查找`struct file`。但数组大小只有32，不可能进程只能打开32个文件吧？

那么真实情况是什么样的呢？让我们跟一下write系统调用的实现看看它是怎么从fd找到file指针的吧。

```c
asmlinkage ssize_t sys_read(unsigned int fd, char __user * buf, size_t count)
{
	int fput_needed;
	file = fget_light(fd, &fput_needed);
}
```

要看懂`fget_light`的代码，要先理解[RCU原理](http://blog.chinaunix.net/uid-12260983-id-2952617.html)，因为内核使用RCU来保护`files_struct`的更新。

跟踪代码的结果很诧异，`struct file*`数组并不是`fd_array`，而是`fdt->fd`。至于`fd_array`有啥用，暂时还不清楚。

## 打开的文件——`struct file`

一个打开的文件，在内核中对应一个`struct file`结构体，采用[slab分配器](http://www.ibm.com/developerworks/cn/linux/l-linux-slab-allocator/)来管理创建与回收工作。

文件的属性可分为静态属性和动态属性。静态属性是文件的固有属性，比如文件路径、文件所有者、文件类型等等；而动态属性则在打开文件时才有意义，比如文件打开模式（只读/只写/读写）和文件当前偏移。

文件路径保存在目录项dentry里，而dentry又保存有指向文件元数据的inode指针，因此file结构只需要保存指向dentry结构的指针即可（`struct dentry *f_dentry）。

同时，file结构又维护文件的动态属性：

```c
struct file {
	struct file_operation *f_op; /*一组文件操作函数指针*/
	atomic_t f_count; /*本结构引用计数*/
	unsigned_int f_flags; /* 文件打开方式 */
	mode_t f_mode; /* 文件是否可读或可写*/
	loff_t f_pos; /*当前偏移量*/
};
```

用户空间对文件的read/write操作最终都是调用`f_op`里的函数。从fd到file结构只需要经过一次转换，看起来好像跟底层文件系统没什么关系，其实关键就是这个`f_op`了。

不同的底层文件系统，在创建文件（file结构）时，会设置`f_op`为指向特定的`struct file_operation`的指针。

## 从文件名到inode的桥梁——目录项dentry

上面提到，`struct file`中只保存有指向`struct dentry`的指针，因此对于打开的文件，`struct dentry`需要保存指向`struct inode`的指针。

另一方面，在open时，内核需要根据文件名pathname来找到该文件对应的inode才能获取到文件数据。如果是新建文件，也需要找到该文件所在目录的inode，才能创建新文件。

因此，如何从pathname定位到inode，也需要借助`struct dentry`。

有关文件路径名解析的具体实现，可以参靠相关资料，这里只是描述一下大致过程：

1. 首先看文件路径名是绝对路径（/path/to/file）还是相对路径（filename），这决定了是从根目录还是从当前目录开始搜索
2. 当前进程的根目录和当前目录信息都保存在`struct tast_struct`的`struct fs_struct *fs`里面

```c
struct fs_struct {
	atomic_t count; // 引用计数
	rwlock_t lock; //锁保护
	int umask;
	struct dentry *root, *pwd, *altroot;
	struct vfsmount *rootmnt, *pwdmnt, *altrootmnt;
};
```

3. dentry结构里面一方面保存了自己的文件（或目录）名称（`d_iname`），如果是目录的话又保存了指向本目录下所有文件（子目录）的dentry的指针（`d_child`）。这样从根目录（或当前目录）开始逐层比对，
即可找到文件的dentry结构了。

## 描述文件数据的数据——inode

