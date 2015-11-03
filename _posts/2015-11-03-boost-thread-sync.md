---
layout: post
title: boost thread的同步编程
categories: boost
tags: boost
---

在C++代码里，使用boost::thread编写多线程程序非常方便。多线程不得不提的就是线程同步了。本文主要记录boost thread各种同步类的使用方法，以作备忘。

在Linux系统上使用boost::thread，不得不提POSIX标准的pthread库了，[详见另外一篇笔记]()。

在POSIX PTHREAD标准中，线程同步需要借助3种类型：

- `pthread_mutex_t`
- `pthread_cond_t`
- `pthread_


