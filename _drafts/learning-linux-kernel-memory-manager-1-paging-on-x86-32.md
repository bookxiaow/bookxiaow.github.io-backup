---
layout: post
title: Linux内核学习之内存管理(一) i386上的分页机制
categories: linux_kernel
tags: linux_kernel
---

分页机制主要关注点：

- 整体架构
- 页访问保护机制
- 地址转换高速缓存
- 换页机制
- 物理地址扩展PAE

# 整体架构

现代计算机系统均采用分页机制来管理物理内存。

> 分页：首先将物理内存划分为一个一个的页（框），CPU寻址时先将线性地址送到MMU（内存管理单元）转换为物理地址，再将物理地址送到地址总线上。

线性地址如何转换为物理地址呢？

在x86-32机器上，假定不开启PAE，那么线性地址的范围是0~4GB，每个页的大小是4KB。

![address_translation](/image/addr_translation.png)

将线性地址分成3部分：

- 31~22bit作为索引去查找页全局目录（起始地址在CR3寄存器内），获得页表的起始地址；
- 再将21~12bit作为索引去页表中查找，获得页框的物理地址；
- 最后将11~0bit作为页内偏移，加上页框的物理地址，即是该线性地址的物理地址。

在32位机器上，一张页表包含1024个页框的物理地址，因此一张页表的大小是32KB；
页全局目录包含1024个页表的物理地址，其大小也是32KB。

如果所有页表都放在内存中的话，需要占用1024 * 32KB = 32 MB。这还只是一个进程的页表需要占用的内存，如果同时运行几百几千个进程，那就要几十几百G。

计算机可没有这么大的内存，而且大部分进程也不会使用全部的4GB空间，因此页全局目录中没有被进程使用的页表设置为NULL，这样就可以大大减少物理内存使用量。

那要想开机i386的分页机制，软件上需要哪些工作呢？

- 1. 准备好一张页全局目录表（一段4KB的连续内存页，保存有各个页表的地址）和若干页表；
- 2. 将页全局目录表的起始地址写入到寄存器CR3；
- 3. 设置CR0寄存器的最高位（PG）为1，打开分页机制。

内核在切换进程时，需要更新CR3寄存器，指向新进程的页全局目录。那么进程的页全局目录地址这些信息保存在哪里呢？x86上，是依靠一个特殊的段TSS来保存的。

在使用mov指令改变CR3寄存器值或者Load新的TSS且其中的CR3的值与当前值不一致的时候，处理器会flush TLB。
## 页目录项和页表项

下面具体看一下上面2级目录表的表项。

![PGD entry](/image/paging-entries.png)

在地址转换过程中，如果某个entry的Present标志位为0，那么表示该页表或页框不在内存中，就会触发缺页异常（paging execption）。内核相关线程会安装相应的异常处理函数，进行换页操作。

**使用分页机制的好处**

1. 隔离进程。每个进程通过线性地址只能访问自己的代码段数据段和内核的代码段数据段，不能直接访问其它进程的代码段和数据段。
2. 物理地址复用。每个进程可以认为自己**独立地**拥有4GB地址空间，通过换页机制，总是可以令当前运行的进程能够访问4GB线性地址空间内的任何地址。

# 页保护机制

通过页目录项和页表项中的User/Supervisor标志位和Read/Write标志位，可以检查代码是否有权限对相关页进行读写操作。

页表项或页目录项的User/Supervisor标志，将页或页表区分成两种模式：

- 超级用户模式（U/S为0）。只有CPL为0,1或2的代码才能寻址。
- 普通用户模式（U/S为1）。只有CPL为3的代码才能寻址。

> 在Linux上，CPL为3表示用户空间，CPL为0表示内核空间。

接下来，再看Read/Write标志：

- 只读模式（Read/Write为0）。
- 读写模式（Read/Write为1）。

内核态代码可以读写任意的页。不管R/W值；用户态代码只对R/W为1的页才拥有写权限。

页目录项和页表项均有U/S和R/W标志，那么两者组合的逻辑如下：

- 只要有一个U/S是Supervisor，那么最终的U/S就是Supervisor；
- 如果最终U/S为User，那么只要有一个R/W是Read-Only，那么最终的R/W就是Read-Only。

其实说白了，就是对两个U/S值作**逻辑与**操作，R/W也一样。
# 换页机制

当要访问的页不存在时，会触发paging exception。这时候操作系统就需要把需要的页换入到内存，这需要先在内存中空出一个页框出来。

这就涉及到

1. 如何标记一个线性地址所在的页不在内存中？
2. 如何决定换出哪块页？

针对问题1，可以查看页表项的Present标志位。

针对问题2，处理器在页表项中提供了Dirty标志位和Access标志位，并保证了以下约束条件：

- 在对任何页执行读或写操作之前，先标记其所关联的页目录项和页表项的Access标志；
- 在对任何页执行写操作之前，先标记其所关联的页表项的Dirty标志。

并且，处理器只负责Set，并不负责Clear这些标志。

操作系统可以利用这些标记位来实现自己的换页机制，比如FIFO页面置换算法，LRU页面置换算法，基于计数的页面置换算法等等。

# TLB

如果进程每一次的寻址操作都需要进行地址转换的话，那得多浪费CPU啊！好在有TLB（Translation Look-aside Buffers）。

TLB是CPU内部的一组寄存器或者一小块访问速度远超内存的存储器。TLB存放的是一组key-value键值对。

在进行地址转换之前，先利用key值去查找TLB，如果命中，那就立即返回页框地址；否则执行地址转换，并将转换后的结果写入TLB。

在i386机器上，每一条TLB记录都是独立的，而且每一条记录包含以下信息：

- 物理页框地址
- 页访问权限
- 页Dirty标志

# PAE

Todo

x86为了扩展32位地址总线，设计了PAE方案。

#题外话——缓存

我们知道计算机系统上的存储系统是分层次的。从上到下，依次是

CPU内部寄存器 -> CPU内部高速缓存 -> 内部存储器（内存） -> 外部存储器（磁盘等外存）

越上面的，造价越高，访问速度越快，而且存储空间越小。

从一方面讲，每一级的存储都可以看作是对下一级存储的缓存（Cache）。比较好理解的就是内部高速缓存了，CPU在访问内存前先访问高速缓存，若命中的话，就无需访问内存了。

谈到缓存，就需要考虑2个因素：一是缓存的命中率，二就是若未命中需要多大的额外开销。任何缓存的设计都力求提高缓存命中率，以及降低未命中时的开销。

系统数据可以永久保存在磁盘上，在系统启动后将需要的数据加载到内存中，包括可执行文件镜像以及各种文件。从这个角度来讲，内存其实就是对外存的一种缓存，降低了CPU访问磁盘的次数。

访问磁盘的时间是访问内存的100000倍。对一个进程而言，它能访问的地址空间称为虚拟地址空间。但是进程的虚拟地址空间不可能全部存放在内存中，部分是存放在磁盘上的交换空间。而内存就是用来缓存进程虚拟地址空间的。而页表项则类似于高速缓存项，当该页表在内存中时，称为命中缓存；当该页表不在内存中时，称为未命中，需要从磁盘中调入页。

访问磁盘的时间是访问内存的100000倍。这导致页面置换算法具备以下特点：

- 为降低未命中开销，将缓存单位设置为比高速缓存行大得多的页（4KB~2MB）
- 全相连缓存，任何虚拟页都可以存放在任何的物理内存页框
- 采用写会策略，而非直写策略，否则每次写内存都会造成写磁盘。