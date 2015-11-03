---
layout: post
title: 使用gdb和qemu调试操作系统 
categories: 操作系统 
tags: xv6
---

按照MIT 6.828课程网站搭建好xv6编译环境后，我们就可以调试xv6了。

**编译**

```sh
$ make 
```

编译完成后，会生成二进制的内核镜像文件 kernel。首先，查找到代码的起点

```sh
$ nm kernel | grep _start
8010b50c D _binary_entryother_start
8010b4e0 D _binary_initcode_start
0010000c T _start
```

代码入口点就是 0x0010000c

**在qemu中运行**

```sh
$ make qemu-nox-gdb
```

**使用gdb调试**

在另外一个shell中启动gdb

```sh
$ cd /path/to/xv6
$ gdb
```

首先，设置断点

```sh
(gdb) br * 0010000c
Breakpoint 1 at 0x10000c
(gdb) c
Continuing.
The target architecture is assumed to be i386
=> 0x10000c:	mov    %cr4,%eax

Breakpoint 1, 0x0010000c in ?? ()
```

查看寄存器信息

```sh
(gdb) info reg
eax            0x0	0
ecx            0x0	0
edx            0x1f0	496
ebx            0x10054	65620
esp            0x7bbc	0x7bbc
ebp            0x7bf8	0x7bf8
esi            0x100000	1048576
edi            0x11515c	1134940
eip            0x10000c	0x10000c
eflags         0x46	[ PF ZF ]
cs             0x8	8
ss             0x10	16
ds             0x10	16
es             0x10	16
fs             0x0	0
gs             0x0	0
```

还可以查看当前栈中的信息：

```sh
(gdb) x/24x %esp
0x7bbc:	0x00007db8	0x00100000	0x0000b596	0x00001000
0x7bcc:	0x00000000	0x00000000	0x00000000	0x00000000
0x7bdc:	0x00010054	0x00000000	0x00000000	0x00000000
0x7bec:	0x00000000	0x00000000	0x00000000	0x00000000
0x7bfc:	0x00007c4d	0x8ec031fa	0x8ec08ed8	0xa864e4d0
0x7c0c:	0xb0fa7502	0xe464e6d1	0x7502a864	0xe6dfb0fa
```

当前堆栈栈顶位于0x7bbc处，该处存放的指令为0x00007db8。其汇编语句为

```sh
(gdb) x/i 0x00007db8
   0x7db8:	add    $0x2c,%esp
(gdb) x/i 0x0000b596
   0xb596:	add    %al,(%eax)
```
