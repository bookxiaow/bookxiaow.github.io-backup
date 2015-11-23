---
layout: post
title: Linux内核学习之进程管理(一)概述
categories: linux_kernel
tags: linux_kernel
---

在操作系统课程里，我们学习了进程的概念，以及进程调度的算法。所谓进程，是一个可执行二进制程序运行的一个实例。

对于每个进程而言，它认为它独占CPU和整个内存地址空间。这是操作系统的进程调度子系统和虚拟存储子系统的功劳。

在本文中，我主要想研究一下Linux内核中进程的实现，包括

* 如何描述进程？
* 如何创建进程？第一个进程是如何创建的？
* 进程调度的基本框架（并不涉及具体算法）

# 如何描述进程

在用户空间，当我们使用fork()来产生一个子进程时，我们会得到一个新的pid。并且，当发送信号（例如Ctrl+C)给进程时，我们总是需要
指定进程的pid。

在Linux 内核中，一个进程就是一个`struct task_struct`，包含了一个进程运行所需要的全部资源。一个`tast_struct`大概2KB。

我们先来看看内核是怎么把用户空间的pid和内核空间的task_struct关联起来的吧。

## 为新进程分配pid

在内核里，分配新的pid，需要调用`alloc_pid()`。

```c
struct pid 
{
    atomic_t count;
    /* Try to keep pid_chain in the same cacheline as nr for find_pid */
    int nr; 
    struct hlist_node pid_chain;
    /* lists of tasks that use this pid */
    struct hlist_head tasks[PIDTYPE_MAX];
    struct rcu_head rcu;
};

struct pid *alloc_pid(void);
```



## 进程间关系树

每个进程均有一个指向父进程的指针，同时它的所有的子进程也连接成一张list。

```c
struct task_struct {
	struct task_struct *parent; /* parent process */
	struct list_head children;  /* list of my children */
	struct list_head sibling;   /* linkage in my parent's children list */
};
```

有关list_head的用法，可以参考我的另一篇笔记。

从*children*表头开始，可以遍历该进程的所有子进程；根据*sibling*，则可以遍历（因为是循环链表）所有的兄弟进程（即具有相同的父进程）。

系统中的第一个进程是init进程，进程pid为1。因此，从init进程的task_struct开始遍历，即可查找到系统所有的进程信息。

## 当前进程

当前进程的`task_struct`可以通过全局变量`current`来访问：

```c
/* include/asm-generic/current.h */
#define current get_current()
#define get_current() (current_thread_info()->task)
```

thread_info保存了当前CPU上运行的进程的一些信息，其中就有指向当前进程的task_struct结构的指针。

```c
struct thread_info { 
    struct task_struct  *task;      /* main task structure */
    struct exec_domain  *exec_domain;   /* execution domain */
    __u32           flags;      /* low level flags */
    __u32           status;     /* thread synchronous flags */
    __u32           cpu;        /* current CPU */
    int         preempt_count;  /* 0 => preemptable, <0 => BUG */
    mm_segment_t        addr_limit;
    struct restart_block    restart_block;
    void __user     *sysenter_return;
#ifdef CONFIG_X86_32
    unsigned long           previous_esp;   /* ESP of the previous stack in case of nested (IRQ) stacks */
    __u8            supervisor_stack[0];
#endif
    int         uaccess_err;
};
```

而CPU会把当前进程的thread_info结构存放在内核栈的最底部。在x86架构上，不同的特权级会使用不同的栈。Linux内核使用了
0和3两个特权级，因此会存在两种不同的栈：用户态栈和内核态栈。

一般，内核栈的大小是页大小的两倍，因此在32位机器上，一般为8kb（宏`THREAD_SIZE`）。

所以，根据内核栈顶地址（%esp）的值，屏蔽掉最后13bit的值（8kb的栈），即可得到thread_info结构的起始地址。

```c
/* include/asm/thread_info.h */
/* how to get the current stack pointer from C */
register unsigned long current_stack_pointer asm("esp") __used;

/* how to get the thread information struct from C */
static inline struct thread_info *current_thread_info(void)
{
	    return (struct thread_info *)
			        (current_stack_pointer & ~(THREAD_SIZE - 1));
}
```

## 进程的状态

进程的状态总共有

* TASK_RUNNING - 正在运行中
* TASK_INTERRUPTIBLE - 正在睡眠，以等待某个条件（例如读文件/网络数据阻塞），当条件满足或者收到信号，会转为TASK_RUNNING状态
* TASK_UNINTERRUPTIBLE - 与上个相比，不能被信号打断睡眠状态。
* \__TASK_STOPPED - 进程终止，例如收到SIGSTOP, SIGTSTP, SIGTTIN, SIGTTOU等信号。
* \__TASK_TRACED - 被其它进程trace，例如ptrace工具。

在`task_struct`中，第一个变量就是进程当前的状态

```c
	volatile long state;    /* -1 unrunnable, 0 runnable, >0 stopped */
```

# 进程的创建

内核使用[slab](http://bookxiaow.github.io/learning-linux-kernel-basic-data-structure-2-slab)来管理task_struct结构体的创建和删除。

## init进程的创建

init进程的`task_struct`是静态创建的，而进程本身则是内核在系统初始化的最后阶段通过创建内核线程来创建init进程的。

init进程负载系统用户空间的初始化。它会为用户启动登录shell，并执行/etc/rc目录下的文件。有关内核启动流程，可以参考相关资料。

## 用户进程的创建

所有用户进程都是由父进程调用`fork()`来创建的。


