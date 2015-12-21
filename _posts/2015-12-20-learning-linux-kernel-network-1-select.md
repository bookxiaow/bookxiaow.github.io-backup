---
layout: post
title: select的内核实现原理
categories: 网络编程
tags: 
---

# 0 前言

在学习网络编程时,我们总是从最简单的Server程序写起：

socket -> bind -> listen -> accept -> read -> write -> return

再接下来，就是学习如何处理客户端的并发请求。主要思路有：

1. 使用多线程/多进程模型
2. 使用IO多路复用模型
3. 使用多线程 + IO多路复用模型

其中，使用IO多路复用模型，我们总是从**select**系统调用开始学起。但是，我们也总是听到，select效率太低了，大型项目中也从来不使用select/poll，而是使用的**epoll**。

那么，为什么select系统调用效率低下呢？而epoll又是作了哪方面改进，从而要高效很多呢？

本篇文章试图通过跟踪select的内核实现源码，来解答为什么select不适用于高并发量的大型项目中。

# 1 阻塞调用与非阻塞调用

使用多路复用技术时，一般都是要求设置待监听的文件为非阻塞模式。那么我们先来看一下什么是阻塞模式，什么是非阻塞模式。

## 概念描述

当read或write一个文件[注]时，如果read/write不能立即返回，那么调用者就会进入睡眠状态，直到该文件变成可读/可写。这种模式就是阻塞模式。

> 注： 在UNIX文化中，一切皆文件。无论是普通磁盘文件，设备，还是网络连接套接字，在虚拟文件系统中都是以文件的形式存在。用户使用统一的read/write函数去读写文件，而不管文件在底层到底是如何存在的。因此，在Linux中，总是可以看到各种形式的“文件”，比如说[eventfd](http://www.man7.org/linux/man-pages/man2/eventfd.2.html)，可以实现事件通知，等等。

以TCP连接为例，用户空间read/write是与TCP缓冲区交互的，而不是直接同网卡驱动交互。当接收缓冲区内无数据时，read操作就无法立即返回，因此就会阻塞住调用进程，直到对方发送数据过来并由网卡驱动拷贝到接收缓冲区；同理，当网络延时过大或接收方问题导致本端（发送方）发送缓冲区满时，调用write无法将数据写入到发送缓冲区（一个字节都没法写入），调用进程也会阻塞。

这里需要注意的是，阻塞与否，是取决于该文件的类型的，换句话说，是取决于该文件对应的设备驱动程序的。

比方说，对于普通磁盘文件，read/write总是可以立即返回的，除非磁盘出现故障。而对于网络连接，或者管道等类型的文件，read/write是有可能发送阻塞的，而且是一种常态模式。

## 内核中的实现

阻塞调用在内核中的实现包含两方面：

- 阻塞方如何进入睡眠状态
- 驱动程序如何唤醒睡眠进程

先来明确一下，这两个操作在哪里实现，或者说，由谁实现。

进程进入睡眠状态，肯定是在调用read/write时由于某个条件未满足（比如说read时接收缓冲区空），因此首先需要执行条件判断。这个工作肯定是而且是只能由驱动程序来完成的。而在VFS中，read/write系统调用最终会调用驱动程序的内部实现（函数指针保存在`struct file` 的`struct file_operations`中）。因此，进程进入睡眠的工作，在驱动程序的read/write中实现。

同理，也是只有在特定的条件满足了，才会去唤醒进程。当read阻塞时，是因为没有数据可读。那何时有新数据进来呢？对于本地文件，当然是有进程写入数据了（调用write）；而对于网络连接，当然是由于对方发送数据过来。这两个操作也都是在驱动程序的write中完成。

因此，进程的睡眠和唤醒动作都是在文件对应的设备驱动程序的read和write中实现的。

接下来，就描述一下进程如何睡眠，以及如何唤醒。其核心是一个叫做“等待队列”的数据结构。

### 等待队列

等待队列基于`struct list_head`实现，包含一个头结点和若干个队列元素。

```c
struct __wait_queue_head {
	spinlock_t lock;
	struct list_head task_list;
};
```

头结点包含一个指向第一个队列元素的指针和一个保护队列的自旋锁：当插入（add_wait_queue）新元素或删除（remove_wait_queue）旧元素时，均使用自旋锁保证同步。

```c
struct __wait_queue {
	unsigned int flags;
	void *private;
	wait_queue_func_t func;
	struct list_head task_list;
};
```

等待队列中的每个元素包含一个回调函数func，在唤醒该元素时调用。其中，private指针指向的是进程的task_struct结构，这样在唤醒时才知道唤醒哪个进程。元素与元素间使用list_head连成双向链表。

#### 睡眠

调用以下任一函数，将进程睡眠在某个等待队列上。

```c
wait_event(wq, condition);
wait_event_timeout(wq, condition, timeout);

wait_event_interruptible(wq, condition);
wait_event_interruptible(wq, condition, timeout);
```

上面两组函数的区别，在于调用进程睡眠时的状态：前者是TASK_UNINTERRUPTIBLE状态，后者是TASK_INTERRUPTIBLE状态，可以被信号打断睡眠。

睡眠的过程分为四步：

1. 判断条件是否满足，若满足则无需睡眠；
2. 否则，定义一个新的wait_queue_t队列元素，插入到wq表示的等待队列中去；
3. 设置进程状态；
4. 调用schedule()，让出CPU，调度其它进程。

需要注意的是，一个等待队列上可能有多个元素，也就表示有多个进程在同时等待。因此，当条件满足时，所有进程都会被唤醒。而且，第一个被唤醒后得到调度的进程可能会“消费”掉条件，因此所有被唤醒的进程在继续执行后首先需要再次判断条件是否满足。

这也是很多代码将睡眠操作放在while循环中的原因。

```c
	/* wait event */
	while(!condition) {
		/* 如果是非阻塞调用，那么直接返回 */
		if (filp->f_flags & O_NONBLOCK)
			return -EAGAIN;

		if (wait_event_interruptible(wq, condition)) {
			/* 被信号中断了 */
			return -ERESTARTSYS;
		}
	}
	/* handle event */
```

如果设置了文件的O_NONBLOCK标志，也就是非阻塞模式，那么在条件不满足时，直接返回-EAGAIN，不会进入睡眠状态。

#### 唤醒

与睡眠函数相对应，调用以下任一函数，将队列上的进程唤醒

```c
void wake_up(wait_event_head_t *queue);
void wake_up_interruptible(wait_event_head_t *queue);
```

这两个函数会唤醒某个队列上睡眠着的所有进程，分为以下步骤：

1. 对队列中的每一个元素，调用其回调函数func;
2. 如果回调函数返回非0值，那么看队列元素的flag值，如果设置了WQ_FLAG_EXCLUSIVE标志，那么停止遍历，不继续唤醒其它进程；否则继续唤醒下一个进程。

那么这个等待队列的回调函数和flag是什么呢？在wait_event中定义的wait_event_t结构体的值如下所示：

```c
wait_queue_t name = {
	.private	= current,
	.func 		= autoremove_wake_function,
	.task_list	= LIST_HEAD_INIT((name).task_list),
};
```

autoremove_wake_function：

- 调用`default_wake_function`唤醒睡眠的进程，
- 将其从等待队列的list中删除掉，成功会返回1。

而flag参数则在prepare_to_wait函数中被清除掉了WQ_FLAG_EXCLUSIVE标志。函数调用顺序如下：

wait_event_interruptible -> DEFINE_WAIT -> prepare_to_wait -> schedule -> finish_wait

其中，prepare_to_wait完成睡眠的前3个步骤。

因此，wake_up_interruptible和wake_up会唤醒所有进程。

# 2 select系统调用

```c
/*
 * @nfds: 待监听的最大fd值+1
 * @readfds: 待监听的可读文件fd集合
 * @writefds: 待监听的可写文件fd集合
 * @exceptfds: 待监听的异常文件fd集合
 * @timeout: 超时设置，在等待指定时间后返回超时
 * return:返回满足条件的fd数量和，如果出错返回-1，如果是超时返回0
 */
int select(int nfds, fd_set *readfds, fd_set *writefds,
		fd_set *exceptfds, struct timeval *timeout);
```

正常返回之后，readfds/writefds/exceptfds会设置相应满足条件的fd。

先来看一下fd_set到底是什么东西。

```c
#define __NFDBITS   (8 * sizeof(unsigned long))
#define __FD_SETSIZE    1024
#define __FDSET_LONGS   (__FD_SETSIZE/__NFDBITS)

typedef struct {
	unsigned long fds_bits [__FDSET_LONGS];
} __kernel_fd_set;
```

在x86机器上，select最大支持1024个文件（`__FD_SETSIZE`）。在fd_set中每个文件使用一个bit表示，因此共需要`__FDSET_LONGS`个long型整数，放在一个数组里。

以下一组函数专门用来操作fd_set：

```c
void FD_SET(int fd, fd_set *set);
void FD_ZERO(fd_set *set);
void FD_CLR(int fd, fd_set *set);
int  FD_ISSET(int fd, fd_set *set);
```

**超时设置**

如果要求select不是无止境地等待，可以指定在若干时间后直接返回0.超时时间由参数timeout指定，是一个`timeval`结构。

```c
struct timeval {
	long    tv_sec;         /* seconds */
	long    tv_usec;        /* microseconds */
};
```

但是，在内核里，确是用的`timespec`结构。

```c
struct timespec {
	long    tv_sec;         /* seconds */
	long    tv_nsec;        /* nanoseconds */
};
```

# 3 select的实现

接下来我们就可以研究一下select是如何实现的了。

假设让我们来设计select，我们该如何实现呢？

## 3.1 一个简单的select实现

我们先把问题简化一下，假定select只监控文件可读的情况，那么一个简单的算法（伪代码）如下所示：

```c
count=0
FD_ZERO(&res_rset)
for fd in read_set
	if( readable(fd) )
		count++
		FDSET(fd, &res_rset)
		break
	else
		add_to_wait_queue

if count > 0
	return count
else
	wait_any_event

return count
```

上面的伪代码只演示了判断文件可读的条件。算法也很朴素，遍历所有文件，如果有文件可读，那么select无需阻塞，直接返回即可，并标记该文件可读；否则，将调用进程加入到每个文件对应设备驱动的读等待队列中，并进入睡眠状态。

该算法有几个待解决的问题：

1. 如何判断文件当前是否可读（或可写），即如何实现readable(fd)函数？
2. 如何将进程加入到驱动的读等待队列？如果调用wait_event_interruptible，那么进程在遇到第一个不可读的fd后会立即进入睡眠，而无法继续监听其它文件。

仔细想一下，这两个问题的解决方法都需要依赖具体的设备驱动才行。因此，为了方便select/poll函数的实现，在Linux中规定每一个支持select/poll监听的文件所属设备驱动必须实现`struct file_operations`中的poll函数：

## 3.2 poll文件操作

```c
/*
 * @filp: 指向打开文件的指针
 * @p: 传入的poll_table指针
 * return: 一个MASK，标志该文件当前状态，例如可读(POLLIN)，可写（POLLOUT）
 *        或出错(POLLERR)，或到达文件尾（POLLHUP）
 */
unsigned int (*poll) (struct file *filp, struct poll_table_struct *p);
```

poll函数的工作有两部分：

- 判断当前文件状态，并在返回值中标记。
- 对本驱动程序的等待队列调用poll_wait函数。

poll_wait函数有什么用呢？

```c
static inline void poll_wait(struct file * filp, wait_queue_head_t * wait_address, poll_table *p)
{
	if (p && wait_address)
		p->qproc(filp, wait_address, p);
}
```

其实就是调用了poll_table的一个函数指针。这个poll_table是在select调用文件的poll函数时传入的。

所以，回到本小节一开始提的2个问题。第1个问题已经解决了，而第2个问题可以在这个p->qproc中实现。

看一下Linux内核中select是如何实现这个函数的：

```c
/* 1. poll_table存放在一个poll_wqueues结构体中 */
struct poll_wqueues {
	poll_table pt;
	struct poll_table_page *table;
	struct task_struct *polling_task;/* 指向睡眠进程的task_struct */
	int triggered;
	int error;
	int inline_index; 
	struct poll_table_entry inline_entries[N_INLINE_POLL_ENTRIES];
};

/* 2. 在do_select中调用poll_initwait初始化poll_wqueues */
void poll_initwait(struct poll_wqueues *pwq)
{
	init_poll_funcptr(&pwq->pt, __pollwait); /* 初始化poll_table */
	pwq->polling_task = current;
	pwq->triggered = 0;
	pwq->error = 0;
	pwq->table = NULL;
	pwq->inline_index = 0;
};

/* 3. p->qproc指向的就是__pollwait函数 */
static void __pollwait(struct file *filp, wait_queue_head_t *wait_address, poll_table *p)
{
    struct poll_wqueues *pwq = container_of(p, struct poll_wqueues, pt);                                                            
    struct poll_table_entry *entry = poll_get_entry(pwq);                                                                           
    if (!entry)                                                                                                                     
        return;                                                                                                                     
    get_file(filp);                                                                                                                 
    entry->filp = filp;                                                                                                             
    entry->wait_address = wait_address;                                                                                             
    entry->key = p->key;                                                                                                            
    init_waitqueue_func_entry(&entry->wait, pollwake);                                                                              
    entry->wait.private = pwq;                                                                                                      
    add_wait_queue(wait_address, &entry->wait);                                                                                     
}
```

在`__pollwait`中将进程加入到文件的等待队列中。select为管理所有监听的文件，为每个文件分配了一个`poll_table_entry`结构。`poll_table_entry`包含文件信息、等待队列头以及等待队列元素。

### poll_table_entry的内存管理

poll_wqueues如何为新的poll_table_entry对象分配内存呢？

它采取的是静态分配 与 动态分配相结合的方式。

![poll_table_entry_mm](http://img.blog.csdn.net/20151220221834401)

如果只有少量文件，那么将文件的`poll_table_entry`放在poll_wqueues结构体里面的数组里面，避免了额外的内存分配工作；如果有很多文件，那么就分配额外的poll_table_page。

每个poll_table_page占据一个内存页的大小。所有的poll_table_page连成一条链表。

```c
struct poll_table_page {
    struct poll_table_page * next;
    struct poll_table_entry * entry;
    struct poll_table_entry entries[0];
};
```

### 唤醒时的回调函数pollwake

注意到，在__pollwait中重新设置wait_queue_t的回调函数指针，而不是使用wait_event_interruptible中设置的autoremove_wake_function。

pollwake调用了__pollwake，而后者在设置了pwq->triggered为1后，直接调用default_wake_function，以唤醒pwq->polling_task进程。也就是调用select系统调用从而睡眠的进程。

因此，任何一个文件可读/可写/出错时，均会触发pollwake被调用，从而睡眠的进程继续执行。

**最后一个问题**

在进程被唤醒后，如何知道是哪个文件的驱动唤醒的呢？因为在

## 3.3 内核中select的实现


跟踪`sys_select`的内核实现，其大致过程如下：

1. 拷贝用户空间的timeout对象到内核空间end_time，并重新设定时间值（标准化处理）
2. 调用core_sys_select。过程如下：
	1. 根据传入的maxfd值，计算保存所有fd需要多少字节（每fd占1bit），然后判断是在栈上分配内存还是在堆中分配内存。共需要分配6个fdset：用户传入的in, out, exception以及要返回给用户的res_in,res_out和res_exception。
	2. 将3个输入fdset从用户空间拷贝到内核空间，并初始化输出的fdset为0；
	3. 调用do_select，获得返回值ret。do_select的工作就是初始化poll_wqueues对象，并调用驱动程序的poll函数。类似于我们写的简单的select。过程如下所示：
		1. 调用poll_initwait初始化poll_wqueues对象table，包括其成员poll_table；
		2. 如果用户传入的timeout不为NULL，但是设定的时间为0，那么设置poll_table指针wait(即 &table.pt）为NULL；
		3. 将in,out和exception进行或运算，得到all_bits，然后遍历all_bits中bit为1的fd，根据进程的fd_table查找到file指针filp，然后设置wait的key值（POLLEX_SET, POLLIN_SET,POLLIN_SET三者的或运算，取决于用户输入），并调用filp->poll(filp, wait)，获得返回值mask。 再根据mask值检查该文件是否立即满足条件，如果满足，设置res_in/res_out/res_exception的值，执行retval++, 并设置wait为NULL。
		4. 在每遍历32（取决于long型整数的位数）个文件后，调用1次cond_resched()，主动寻求调度，可以等待已经遍历过的文件是否有唤醒的；
		5. 在遍历完所有文件之后，设置wait为NULL，并检查是否有满足条件的文件（retval值是否为0），或者是否超时，或者是否有未决信号，如果有那么直接跳出循环，进入步骤7；
		6. 否则调用poll_schedule_timeout，使进程进入睡眠，直到超时（如果未设置超时，那么是直接调用的schedule()）。如果是超时后进程继续执行，那么设置pwq->triggered为0；如果是被文件对应的驱动程序唤醒的，那么pwq->triggered被设置为1(见第2节).
		7. 最终，函数调用poll_freewait，将本进程从所有文件的等待队列中删掉，并删除分配的poll_table_page对象，回收内存，并返回retval值。
	4. 拷贝res_in, res_out和res_exception到传入的in, out, exception，并返回ret。
3. 调用poll_select_copy_remaining，将剩余的timeout时间拷贝回用户空间。


以上就是select的全部过程了，其实逻辑是很简单的，非常类似于我们自己实现的简单的select。

其中它利用了一个重要的特性，就是以NULL作为poll_table指针去调用驱动程序的poll函数。这会使得驱动不会将进程添加到等待队列上。在上面的过程中，有两次使用了这个特性：

1. 如果传入了非NULL的timeout，但超时时间为0，不会睡眠在任何文件上；
2. 任何一个文件是立即可读/可写的，不会继续睡眠在剩下的文件上。

可以看出，这种实现效率之所以低下，有以下几方面原因：

1. 可以同时监听的文件数量有限，最多1024个。这对要处理几十万并发请求的服务器来说，太少了！
2. 每次调用select，都需要从0bit一直遍历到最大的fd，并且每隔32个fd还有调度一次（2次上下文切换）。试想，如果我要监听的fd是1000，那么该是多么的慢啊！而且在有多个fd的情况下，如果小的fd一直可读，那就会导致大的fd一直不会被监听。
3. 内存复制开销。需要在用户空间和内核空间来回拷贝fd_set，并且要为每个fd分配一个poll_table_entry对象。

# 4 总结

select虽然效率低下，且基本上不会再被用于大型项目中。但是其内核实现相比较epoll则更为简单，便于理解。

在此基础上，我们可以进一步地去研究epoll的实现，两相比较，更好地理解，为什么epoll会更加高效。这也是我下一步要研究的东西。


