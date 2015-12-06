---
layout: post
title: Linux内核学习之使用gcc内联汇编
categories: linux_kernel
tags: linux_kernel
---

Linux内核源码中大量使用内联汇编，一方面是有时候需要直接访问相关寄存器，一方面是有时候使用内联汇编效率会高点。

先来看一个例子：

```c
#include <stdio.h>

static __inline__ void atomic_add(int i, int* m)
{
	__asm__ __volatile__ (
		"addl %1, %0"
		:"=m"(*m)
		:"ir"(i), "m"(*m));
}

int main()
{
	int a = 3;
	atomic_add(1, &a);

	printf("a=%d\n", a);

	return 0;
}
```

Linux平台的内联汇编使用的是由gcc提供的asm语句，因此也可以在应用程序代码中使用。在上面的例子中，我们仿照内核的atomic_t实现的加法操作（少了LOCK原语）。


