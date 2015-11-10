---
layout: post
title: 在多线程环境下使用libmysqlclient
categories: mysql
tags: mysql 多线程
---

几个函数

```c
void my_init();
my_bool STDCALL mysql_thread_init();
void STDCALL mysql_thread_end();
uint STDCALL mysql_thread_safe(void);
```

```c

```
