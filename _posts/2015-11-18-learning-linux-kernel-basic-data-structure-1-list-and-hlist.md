---
layout: post
title: Linux内核学习之基础数据结构(一) list & hash table
categories: linux_kernel
tags: linux kernel 
---

内核中使用最多的数据结构估计就是链表了，这是因为操作系统的本质工作就是管理系统中各种各样的资源。
资源管理当然离不开链表了，比如系统中所有的进程，文件目录树，所有挂载的文件系统等等。

内核自己实现了两种链表：双向循环链表list_head 和 专用于hash表的hlist_head。

# 1 list_head

list_head的设计如下图所示：

![list_head结构图](/image/list_head.jpg)

使用链表的第一步是定义head节点。head节点不包含有效数据，是一个哑节点。

链表节点定义如下：

```c
struct list_head {
	struct list_head *next, *prev; 
};
```

**初始化链表**

```c
struct list_head head;
INIT_LIST_HEAD(&head);
```

**插入新节点**

因为是循环链表，所以任何的插入操作都是在两个节点中间插入：

```c
static inline void __list_add(struct list_head *new,
                  struct list_head *prev,
                  struct list_head *next)
{
    next->prev = new;
    new->next = next;
    new->prev = prev;
    prev->next = new;
}
```

由于head节点不包含有效数据，因此在链表头部插入节点，相当于是在head节点和head->next节点中间插入：

```c
static inline void list_add(struct list_head *new, struct list_head *head)
{
    __list_add(new, head, head->next);
}
```

同理，在链表尾部插入节点，相当于在head->prev和head之间插入：

```c
static inline void list_add_tail(struct list_head *new, struct list_head *head)
{
    __list_add(new, head->prev, head);
}
```

**删除节点**

对于双向链表，删除操作不仅要更新prev节点的next值，还要更新next节点的prev值。
因此删除节点操作需要先更新next节点和prev节点：

```c
static inline void __list_del(struct list_head * prev, struct list_head * next)
{
    next->prev = prev;
    prev->next = next;
}
```

提供给外部的删除接口list_del在删除操作之外，还重置了删除节点的指针值（指向某个特殊的地址）。
如果在删除节点之后，仍然访问该节点的next/prev指针，就会触发page fault。

```
static inline void list_del(struct list_head *entry)
{
    __list_del(entry->prev, entry->next);
    entry->next = LIST_POISON1; 
    entry->prev = LIST_POISON2;
}

/*
 * These are non-NULL pointers that will result in page faults
 * under normal circumstances, used to verify that nobody uses
 * non-initialized list entries.
 */
#define LIST_POISON1  ((void *) 0x00100100)
#define LIST_POISON2  ((void *) 0x00200200)
```

同时，提供了另外一个`list_del_init()`，在删除之后重新初始化删除的节点。

```c
static inline void list_del_init(struct list_head *entry)
{
    __list_del(entry->prev, entry->next);
    INIT_LIST_HEAD(entry);
}
```

**使用方法**

list_head结构体并不包含任何有效数据，那么在实际使用时，需要将list_head嵌入到具体的结构体中去。

```c
/* 自定义链表节点，包含list_head */
struct my_list_node {
	int data;
	struct list_head list;
};

/* 获取下一个链表节点 */
struct my_list_node * next(struct my_list_node *node)
{
	// return list_entry(node->list.next, struct my_list_node, list);
	return container_of(node->list.next, struct my_list_node, list);
}
```

使用container_of宏，可以通过指向结构体中某个变量的指针，获取到指向整个结构体的指针。

它的原理很简单，假定我在地址0处定义了一个结构体，那么结构体中某个变量的地址就是它在结构体中的偏移量。
再把该变量的地址减去这个偏移量，就是整个结构体的地址了。

```c
#define container_of(ptr, type, member) ({          \
        const typeof( ((type *)0)->member ) *__mptr = (ptr);    \
        (type *)( (char *)__mptr - offsetof(type,member) );})
```

**遍历链表**

遍历链表，只要从head开始，一直follow next或prev即可。

```c
/* 宏定义 */
#define list_for_each(pos, head) \
	for (pos = (head)->next; prefetch(pos->next), pos != (head); \
			pos = pos->next)

/* 使用示例 */
struct list_head *node;
list_for_each(node, head) {
	/* now, node points to the current node, starting from head->next */
	// do something
}
```

但如果遍历的是嵌入了list_head的自定义结构体呢？

```c
#define list_for_each_entry(pos, head, member)              \
    for (pos = list_entry((head)->next, typeof(*pos), member);  \
         prefetch(pos->member.next), &pos->member != (head);    \
         pos = list_entry(pos->member.next, typeof(*pos), member))

/* 使用示例 */
struct my_list_node head;
struct my_list_node * node;
list_for_each_entry(node, &head, list) {
	/* now node points to the current node */
} 
```

# 2 hlist

内核里面经常需要执行字符串查找操作，例如文件名查找，或者设备名查找等等。使用hash结构可以大大减少查找时间。

一个冲突很少的hash表，其查找时间为O(1)。但hash表存储有限，而带存储的数据量大的话，冲突难以避免。解决hash表冲突的
方法有很多种，例如开放定址法和拉链法。

如果使用拉链法，那么就需要一条链表来维护hash到相同key值的所有记录。hash表中只保存一条条链表的头节点。
例如，如果使用list_head来作hash链表的话:

```c
struct list_head hash_table[HASH_TABLE_SIZE];
```

但是，list_head保存2个指针，有点浪费内存空间了，因此内核开发人员设计了另外一种链表hlist，比list_head更适用于设计hash表。

```c
/* 表头，只包含一个指针 */
struct hlist_head {
	struct hlist_node *first;
};

/* 链表节点，双向链表 */
struct hlist_node {
	struct hlist_node *next, **pprev;
};
```

在一般链表中，指向相邻节点的next或prev指针都是直接指向next或prev节点，例如list_nead。但是hlist的pprev却不是指向上个节点的指针，而是指向上个节点的next指针的指针。如下图所示：

![hlist结构](/image/hlist.jpg)

head不保存有效数据，其first指针指向第一个hlist node。很明显，hlist_node第二个指针设计为指向指针的指针，就是为第一个hlist node设计的，因为它的前一个节点并不是
hlist_node类型的。

hlist同样有一组类似于list_head的操作：

```c
/* 初始化头 */
#define HLIST_HEAD(name) struct hlist_head name = {  .first = NULL }
#define HLIST_HEAD_INIT { .first = NULL }
#define INIT_HLIST_HEAD(ptr) ((ptr)->first = NULL)

/*初始化节点*/
static inline void INIT_HLIST_NODE(struct hlist_node *h); // h->next, h->pprev设置为NULL

/*插入节点*/
/* 在头部插入 */
static inline void hlist_add_head(struct hlist_node *n, struct hlist_head *h);
/* 在指定节点的前面插入 */
static inline void hlist_add_after(struct hlist_node *n, struct hlist_node *next);
/* 在指定节点的后面插入 */
static inline void hlist_add_before(struct hlist_node *n, struct hlist_node *next);

/* 删除节点 */
static inline void hlist_del(struct hlist_node *n);
/* 删除后初始化 */
static inline void hlist_del_init(struct hlist_node *n);

/* 遍历hlist */
#define hlist_entry(ptr, type, member) container_of(ptr, type, member)

#define hlist_for_each(pos, head) \
    for (pos = (head)->first; pos && ({ prefetch(pos->next); 1; }); \
         pos = pos->next)

#define hlist_for_each_entry(tpos, pos, head, member)            \
    for (pos = (head)->first;                    \
         pos && ({ prefetch(pos->next); 1;}) &&          \
        ({ tpos = hlist_entry(pos, typeof(*tpos), member); 1;}); \
         pos = pos->next)

/* safe版，在处理node之前，先保存其next值，以免其它代码删除了本node */
#define hlist_for_each_safe(pos, n, head) \
    for (pos = (head)->first; pos && ({ n = pos->next; 1; }); \
         pos = n)

#define hlist_for_each_entry_safe(tpos, pos, n, head, member)        \
    for (pos = (head)->first;                    \
         pos && ({ n = pos->next; 1; }) &&               \
        ({ tpos = hlist_entry(pos, typeof(*tpos), member); 1;}); \
         pos = n)
```

使用hlist同样也是需要将hlist_head嵌入到链表头部结构，将hlist_node嵌入到链表节点结构中去。
