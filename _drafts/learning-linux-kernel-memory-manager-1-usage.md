---
layout: post
title: Linux内核学习之内存管理(一) 动态申请内存
categories: linux_kernel
tags: linux_kernel
---

在C语言中，我们习惯使用*malloc*和*free*来动态分配和释放内存；在C++中，也有类似的*new*和*delete*。

那么，在内核编程中，该怎么样申请动态内存呢?

# 虚拟地址和物理地址

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

内核在切换进程时，需要更新CR3寄存器，指向新进程的页全局目录。那么进程的页全局目录地址这些信息保存在哪里呢？x86上，是依靠一个特殊的段TSS来保存的。

**使用分页机制的好处**

1. 隔离进程。每个进程通过线性地址只能访问自己的代码段数据段和内核的代码段数据段，不能直接访问其它进程的代码段和数据段。
2. 物理地址复用。每个进程可以认为自己**独立地**拥有4GB地址空间，通过换页机制，总是可以令当前运行的进程能够访问4GB线性地址空间内的任何地址。

# 物理地址划分

线性地址的大小取决于计算机指令系统。对于特定的计算机系统和操作系统，线性地址空间大小也就固定了。

但是物理地址可就取决于你给计算机安装了多大的内存了。比如32位机PC的内存就有可能是1G，2G或者4G。

物理地址和线性地址的大小可分为2种：

- 物理地址大小小鱼线性地址
```c
/* #include <linux/gfp.h> */
/* alloc_page(s) - 从物理内存中分配页 */
struct page * alloc_pages(gfp_t gfp_mask, unsigned int order);
struct page * alloc_page(gfp_t gfp_mask);
/* __free_pages - 释放分配的页 */
void __free_pages(struct page *page, unsigned int order);

/* #include <linux/mm.h> */
/* page_address - 返回物理页的虚拟地址 */
unsigned long page_address(struct page *page);

/* __get_free_pages - 分配页并直接返回虚拟地址 */
unsigned long __get_free_pages(gfp_t gfp_mask, unsigned int order);
unsigned long __get_free_page(gfp_t gfp_mask);
/* free_page(s) - 释放虚拟地址对应的物理页 */
void free_pages(gfp_t gfp_mask, unsigned int order);
void free_page(gfp_t gfp_mask);

/* get_zeroed_page - 分配页并直接返回虚拟地址，页内所有字节全重置为0. 当需要返回给用户空间时，可以防止敏感数据被发现*/
unsigned long get_zeroed_page(gfp_t gfp_mask, unsigned int order);
