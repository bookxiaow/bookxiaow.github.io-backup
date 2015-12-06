---
layout: post
title: UNIX网络编程学习笔记(1)基础知识
categories: 网络编程
tags: 网络编程
---

在生产环境下，或者说在正式项目的框架中，网络编程变得更轻松:你只需要调用框架中既定的接口即可。时间久了，有时候想写一个原始的（raw）的C/S都需要翻翻书。本文就试图对原始套接口编程中比较容易忘的一些细节做个整理，以作备忘。

#头文件的问题

套接口编程的第一个问题恐怕就是头文件了。要想写一个最简单的网络程序，你至少要包含以下头文件：

```c
#include <sys/socket.h>
#include <netinet/in.h>
#include <arpa/inet.h>
#include <string.h>
```
## socket地址问题

第一个文件是基本头文件，包含所有的socket相关API。

第二个文件则跟socket地址相关。系统上有多种socket，比如有IPv4/IPv6/Unix域/Datalink/Storage等等。各种socket地址格式并不统一，因此就有了不同的结构体定义。

```c
/* IPv4 地址 */
struct sockaddr_in {
	uint16_t		sin_len;
	sa_family_t		sin_family;
	in_port_t		sin_port; /* 16-bit, 网络字节序 */
	struct in_addr	sin_addr;
	char			sin_zero[8];
};

struct in_addr {
	in_addr_t		s_addr; /* 32-bit, 网络字节序 */
};

/* IPv6 地址 */
struct in6_addr;
struct sockaddr_in6;

/* UNIX 域套接口地址 */
struct sockaddr_un;
```

但是socket API不可能为每一种协议类型定义特定的API（就像C++中的函数重载）。因此，在所有socket API中，socket 地址全部用统一的结构体`struct sockaddr`来表示。

```c
struct sockaddr {
	uint8_t		sa_len;
	sa_family_t sa_family;
	char		sa_data[14];
};
```

所以，在传递地址指针到API的时候，全部结构体都要转化为`struct sockaddr *`。

注意，在x86上`sockaddr`结构体和`sockaddr_in`均占用16字节，但是IPv6地址占用28字节。而像UNIX域套接口和Datalink套接口这种地址长度不固定的，必须要知道确切的地址长度才行。所以，socket API中在传递地址指针的同时，还需要传递地址长度。就是这个原因。

在基础头文件中"sys/socket.h"中，只包含了`struct sockaddr`的定义；ipv4和ipv6的地址定义在"netinet/in.h"中;Unix域套接口地址定义在"linux/un.h"中。

## 2 地址转换

有关地址的第二个问题就是主机字节序和网络字节序的转换以及字符串形式的IP地址和整数形式之间的转换了。

### 字节序的转换

```c
#include <netinet/in.h>

uint16_t htons(uint16_t host16bitvalue);
uint32_t htonl(uint32_t host32bitvalue);
uint16_t ntohs(uint16_t net16bitvalue);
uint32_t ntohl(uint32_t net32bitvalue);
```

传递到socket API的地址参数中保存的均是网络字节序，所以在写入或读出socket地址时，都需要调用上述函数进行字节序转换。

### 字符串和二进制IP地址之间的转换

用户输入的参数一般都是字符串形式的IP地址，比如"192.168.1.1"，便于阅读。

```c
#include <arpa/inet.h>
/* inet_pton - presentation（字符串）to numeric（二进制） 
 * @family - AF_IET(IPV4)或AF_INET6(IPV6)
 * @strptr - 字符串
 * @addrptr - 指向struct in_addr或struct in6_addr的指针
 * return - 1-OK, 0-invalid input, -1 - error
 */
int inet_pton(int family, const char *strptr, void *addrptr);

/* 多了len参数，保证不会数组越界 */
const char *inet_ntop(int family, const void *addrptr, char *strptr, size_t len);
```

也可以使用下面一组函数，但只适用于IPv4地址。

```c
#include <arpa/inet.h>
/* 字符串转为二进制 */
int inet_aton(const char *strptr, struct in_addr *addrptr);
/* 二进制转为字符串 */
char * inet_ntoa(struct in_addr inaddr);
```

# 处理并发请求

Server不可能只服务于一个client，因此总是要设计一种架构，用来应付多个client的并发请求。

## 多进程

多进程方案应用的比较少。毕竟多进程之间共享数据总没有多线程来的方便。

```c
while(1) {
	int clifd = accept(listenfd, (struct sockaddr *)&cliaddr, &clilen);

	if ((pid = fork()) == 0) { /* child */
		close(listenfd);
		doRequest();
		exit(0);
	}
	close(clifd);
}
```

主进程负责监听请求，与client建立连接之后，由子进程负责处理请求。

## 多线程

单纯的多线程架构类似于多进程，只是使用多线程，共享地址空间比较方便和自然。

## I/O复用

使用epoll/select可以同时监听多个IO。

```c

```
