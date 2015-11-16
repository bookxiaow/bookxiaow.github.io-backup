---
layout: post
title: Linux内核学习之文件系统（2）文件路径解析
categories: Linux_kernel
tags: Linux kernel filesystem
---

本文继续研究一下VFS中的一个知识点：文件路径解析。

从userspace调用open(), mkdir(), rename() 或者 stat() 等系统调用时都需要传入文件路径作为参数。
内核必须能够从文件路径名寻找到对应的inode，才能执行文件操作。这个过程就是文件路径解析。

## 1 问题描述

### 绝对路径 vs 相对路径

如果路径名以"/"开始，那么就是绝对路径，从根目录（/）开始，每个文件名（除了最后一个）都代表一个目录。

如果路径名不是以"/"开始，那么就是相对路径，从当前目录开始解析。

上篇文章提到，进程可以拥有自己的namespace。因此要想修改进程的根目录，可以修改namespace的根目录。但是对namespace的修改会影响所有共享该namespace的其它进程。

作为选择，进程可以只修改自己的根目录和当前目录信息（chroot() & chdir()）。

```c
struct task_struct {
/* filesystem information */
    struct fs_struct *fs;
};

struct fs_struct {
    atomic_t count;
    rwlock_t lock;
    int umask;
    struct dentry * root, * pwd, * altroot; // 根目录和当前目录
    struct vfsmount * rootmnt, * pwdmnt, * altrootmnt; //根目录和当前目录挂载的文件系统
};
```

### 最简单的算法

文件路径解析的问题可以这样描述：

`struct inode * path_lookup(const char *pathname);`

**输入**
字符串pathname

**输出**
指向该文件inode的指针

算法描述：
1. 判断是绝对路径还是相对路径，确定匹配的起始dentry
2. 从当前dentry包含的所有子目录找到文件路径的第一个分量，进入下一级目录，若无匹配项则返回错误
3. 重复上一步操作，直到最后一级目录，若匹配到，则可以找到文件的dentry，
4. 通过dentry，获得指向inode的指针，算法结束。

乍看起来算法很简单，但是还有很多因素没有被考虑到：

1. 进程在进入每一级目录时，必须检查对该目录是否有相应的权限。目录的权限意义如下：
		
		* 可读：可以查看目录下文件列表（ls命令）,但不能查看文件属性，不能穿过该层目录查看下一级目录文件列表

    	* 可写：可以在目录下新建或删除文件

    	* 可执行：可以进入该目录（cd），可以查看目录下文件属性

2. 对于软连接文件，需要跟踪到实际的文件路径，还需要考虑是否存在软连接是否成环。
3. 如果路径中的某个目录挂载了其它的文件系统，需要进入新的文件系统目录树。
4. 需要考虑当前进程位于哪个namespace内，不同的namespace下同一个路径对应不同的文件。

## 2 Linux的实现

Linux中的实现是

```c
/**
 * path_lookup - 根据文件路径名查找文件inode信息
 * @name: 文件路径名，例如"/home/bookxiao/file.txt"或"file.txt"
 * @flags: 文件访问方式，见下表1
 * @nd: 返回查找的文件信息，格式见下文
 * return: 成功返回0，nd包含有效信息；失败返回其它值。
 */
int fastcall path_lookup(const char *name, unsigned int flags, struct nameidata *nd);

/**
 * 文件查找结果信息
 */
struct nameidata {
    struct dentry   *dentry; // 目录项
    struct vfsmount *mnt; // 挂载的文件系统信息
    struct qstr last; // 文件路径最后一个名字，即不包含路径的文件名
    unsigned int    flags;
    int     last_type; // 文件路径最后一个component的类型
    unsigned    depth;//如果是符号链接，depth表示跟踪的深度
    char *saved_names[MAX_NESTED_LINKS + 1];//如果是符号链接，这里保存所有跟踪的文件名称

    /* Intent data */
    union {
        struct open_intent open; //文件将被如何访问
    } intent;
};
```

调用`path_lookup`返回的是指向dentry对象和vfsmount对象的指针，假如调用之后这两个对象被释放回收了，再访问这两个指针就会segment fault。
因此`path_lookup()`会增加这两个对象的引用计数，以保证指针有效。在使用完后，调用`path_release()`释放资源（减少引用计数）。

表1 path_lookup() flag参数

| Macro | 功能 |
| ---- | ---- |
| LOOKUP_FOLLOW | 如果最后一个component是符号链接，跟踪它|
| LOOKUP_DIRECTORY | 最后一个component必须是目录 |
| LOOPUP_PARENT | 返回最后一个component所在目录的信息 |
| LOOPUP_CONTINUE | 暂不清楚 |
| LOOKUP_OPEN | 查询目的是open |
| LOOKUP_CREATE | 查询目的是create |
| LOOKUP_ACCESS | 查询目的是检查用户权限 |


