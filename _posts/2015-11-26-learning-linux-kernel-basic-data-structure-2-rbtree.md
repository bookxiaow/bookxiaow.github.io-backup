---
layout: post
title: Linux内核学习之基础数据结构(二) 红黑树
categories: linux_kernel
tags: linux_kernel
---

#  什么是红黑树

红黑树是一种自平衡的搜索二叉树（[self-balancing binary search tree](https://en.wikipedia.org/wiki/Self-balancing_binary_search_tree)）。

相比于普通搜索二叉树，红黑树赋予每个节点一个color的属性（取值红或黑），并通过一系列的约束条件，保证了在插入或删除操作后红黑树的
高度仍然在在logN级，不会退化为链表，从而保证查找操作的复杂度一直保持在logN。

##约束条件

除了具有普通二叉搜索树的特性之外，红黑树拥有以下约束条件：

1. 节点要么是红色，要么是黑色；
2. 根节点是黑色；
3. 所有的叶子节点均不含有数据，为NULL节点，且均为黑色；
4. 红色节点的孩子节点必须是黑色；
5. 对于任意一个非叶子节点，从它到它下面的任意一个叶子节点（NULL节点）的路径上的黑色节点的数量必须一致。根节点到NULL节点路径上的黑色节点的数量
称为红黑树的*black-height*

![red black tree](/image/rbtree.png)

条件4和条件5保证了：从根节点到最深的叶子节点的路径，不会超过它到最浅叶子节点的路径的2倍。这是因为：

* 1）最短的路径上是全部为黑色节点的路径，假定为B个；
* 2）最长路径相当于在最短路径上插入若干个红色节点（因为黑色节点数量不可更改），但红色节点又不能连续（条件4），所以红色节点数量最多不超过B-1个，
总过节点数量就是2B-1。

##插入操作

先来看一下如何构建一颗红黑树。

```c
struct rbtree_node {
	int key;
	int color;
	struct rbtree_node *left;
	struct rbtree_node *right;
	void *data;
};	

#define RED 0
#define BLACK 1
```

先静态创建根节点和NULL节点（叶子节点）

```c
struct rbtree_node nil_node = {
	.color = BLACK,
	.left = NULL,
	.right = NULL,
	.data = NULL
};

struct rbtree_node root = {
	.color = BLACK,
	.left = &nil_node,
	.right = &nil_node,
	.data = NULL
};
```

假设要插入的节点是N，其color初始为RED，其插入过程如下：

1. 同普通二叉搜索树一样，根据key值将节点N插入到树中合适的位置
2. N的初始颜色为Red，然后调整整个树，使满足红黑树约束条件。

第1步需要注意红黑树的约束条件3，节点N会替换掉某个内部节点的NULL子节点，并且自己也包含两个NULL子节点。

重点就是第2步了，需要对树进行翻转和重新着色，其过程可以用伪代码所示如下：

```
RB_INSERT_FIXUP(T, z)
	while z.p.color == RED
		if z.p == z.p.p.left // 父节点是祖父节点的左子节点
			y = z.p.p.right // 叔节点
			if y.color == RED // 叔节点和父节点都是RED（case 1）
				z.p.color = BLACK
				y.color = BLACK
				z.p.p.color = RED
				z = z.p.p
			else if z == z.p.right // z是父节点的右子节点（case 2）
					z = z.p
					LEFT_ROTATE(T, z)
				z.p.color = BLACK
				z.p.p.color = RED
				RIGHT_ROTATE(T,z.p.p)
		else // 父节点是祖父节点的右子节点，与上面的情况对称处理即可
	
	T.root.color = BLACK
```

循环不变量：

- z是红色的；
- 如果z.p是root，那么z.p是黑色的；
- 如果违反了红黑树特性，那么最多只会违反其中1条，要么是条件2（根节点不为black），要么是条件4（红色节点子节点不为黑色）。

如果违反了条件2，那么只需要将根节点重新设置为黑色即可，这发生在N节点就是根节点时。
先来看一下不违反约束条件的插入

![normal_insert](/image/rbtree_normal_insert.png)

在将新节点插入后，红黑树特性仍能满足。但在下图(a)和图(b)中，就违反了条件4.

![error_insert](/image/rbtree_error.png)

(a)和(b)的区别在于，N->p的邻居节点是红色还是黑色。这两种情况的处理方式有稍许不同。

在(a)中，我们只需要将N的父节点和叔节点全部标为红色，再将N的祖父节点标为红色，然后继续将祖父节点作为新的N节点，继续循环即可。

在(b)中，我们需要通过旋转将N的父节点提升一层，并重新上色，如下图所示。

![rb_tree_update](/image/rbtree_update.png)

如果是上图中(a)所示的情况，需要先对N的父节点进行左旋转（左边的节点下降一层，右边的节点上升一层），变成(b)所示情况。

## 删除操作

# Linux内核中的实现

```c
/* 节点定义 */
struct rb_node
{
    unsigned long  rb_parent_color; //不是parent's color,而是变量复用，既是parent，又是color
#define RB_RED      0
#define RB_BLACK    1
    struct rb_node *rb_right;
    struct rb_node *rb_left;
}

/* root */
struct rb_root
{
    struct rb_node *rb_node;
};
```

红黑树的节点应该包含4个属性：指向父节点、左右子节点的指针，以及color。在内核的实现中，指向parent的指针和color复用一个变量`rb_parent_color`。
其中，整形值的最低1bit作为color（红色为0，黑色为1），高30bit为指向parent的指针（因为32位机器上指针是4字节对齐的）。

```c
/* 获取指向parent的指针 */
#define rb_parent(r)   ((struct rb_node *)((r)->rb_parent_color & ~3))

/* 获取color值 */
#define rb_color(r)   ((r)->rb_parent_color & 1)

/* 判断和设置color */
#define rb_is_red(r)   (!rb_color(r))
#define rb_is_black(r) rb_color(r)
#define rb_set_red(r)  do { (r)->rb_parent_color &= ~1; } while (0)
#define rb_set_black(r)  do { (r)->rb_parent_color |= 1; } while (0)
```

## 使用方法

使用内核中的rbtree，同list_head一样，需要将rb_node嵌入到自定义数据结构中去，因为rb_node并没有包含key值和data域，不能单独使用。

现在用一个例子，来说明如何使用rb_node。假定我们定义的数据结构如下所示：

由于维持二叉搜索树的有序性的关键key值保存在自定义的数据结构里面，因此需要实现自己的搜索和插入操作。

下面是一个完整的事例

```c
#include <linux/init.h>
#include <linux/module.h>
#include <linux/rbtree.h>
#include <linux/slab.h>

MODULE_LICENSE("Dual BSD/GPL");
MODULE_AUTHOR("Wang Shuxiao");


struct myroot {
	struct rb_root rbroot;
};

static struct myroot groot;

struct mynode {
	char *data;
	int key;
	struct rb_node rbnode;
};

/* 自定义的查找操作 */
static inline struct mynode * rb_search_mynode(struct myroot *root, int key)
{
	struct rb_node *n = root->rbroot.rb_node;

	struct mynode *p;

	while (n) {
		p = rb_entry(n, struct mynode, rbnode);

		if (key < p->key)
			n = n->rb_left;
		else if (key > p->key)
			n = n ->rb_right;
		else
			return p;
	}
	return NULL;
}

/* 自定义插入操作第1步，按照key值找到新节点合适的位置并插入*/
static inline struct mynode * __rb_insert_mynode(struct myroot *root, int key, struct rb_node * node)
{
	struct rb_node **p = &root->rbroot.rb_node;
	struct rb_node * parent = NULL;
	struct mynode *tmp_node;

	while (*p) {
		parent = *p;
		tmp_node = rb_entry(parent, struct mynode, rbnode);

		if (key < tmp_node->key)
			p = &(*p)->rb_left;
		else if (key > tmp_node->key)
			p = &(*p)->rb_right;
		else
			return tmp_node;
	}

	rb_link_node(node, parent, p);
}

/* 完整的插入操作 */
static inline struct mynode * rb_insert_mynode(struct myroot *root, int key, struct rb_node * node)
{
	struct mynode * ret;
	if ((ret = __rb_insert_mynode(root, key, node)))
		goto out;
	/* 自定义插入操作第2步，重新调整rbtree，以满足约束条件*/
	rb_insert_color(node, &root->rbroot);

out:
	return ret;
}

static int hello_init(void)
{
	RB_EMPTY_ROOT(&groot.rbroot);
	struct mynode * n = kmalloc(sizeof(struct mynode), GFP_KERNEL);
	n->key = 3;
	n-> data = "hello, my dear\n";
	rb_insert_mynode(&groot, n->key, &(n->rbnode));

	return 0;
}

static void hello_exit(void)
{
	struct mynode * n = rb_search_mynode(&groot, 3);

	printk(KERN_INFO "%s", n->data);

	kfree(n);
}

module_init(hello_init);
module_exit(hello_exit);
```

再编写一个简单的Makefile

```makefile
ifneq ($(KERNELRELEASE),)
obj-m := rb_test.o

else
KERNELDIR ?= /lib/modules/$(shell uname -r)/build
PWD := $(shell pwd)

default:
	$(MAKE) -C $(KERNELDIR) M=$(PWD) modules
clean:
	$(MAKE) -C $(KERNELDIR) M=$(PWD) clean

endif
```

然后编译模块，执行加载和卸载操作，观察syslog输出

```sh
$ make
$ sudo insmod rb_test.ko
$ sudo rmmod rb_test
$ tail -f /var/log/syslog
Nov 24 18:02:02 ubuntu-kernel kernel: [90583.886326] hello, my dear
```

在我们的事例中，在加载模块时将key值为3的一个mynode节点插入到rbtree中，然后在卸载模块时再执行搜索操作获得mynode，然后打印出data字符串，结果如上所示。
