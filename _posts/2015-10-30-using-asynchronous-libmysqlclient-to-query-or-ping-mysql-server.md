---
layout: post
title: 使用异步libmysqlclient连接MySQL Server  
categories: mysql 
tags: mysql
---

#0 前言

在[上篇文章](http://bookxiaow.github.io/connect-to-mysql-server-by-using-libmysqlclient/)中提到的API都是**同步**操作：Client要一直等待Server返回请求结果。

考虑以下场景：

Client ----------------- DB\_Proxy ----------------- MySQL\_Server

- Client与DB\_Proxy按特定的协议传输请求和响应，例如Thrift/HTTP等等;

- DB\_Proxy将请求转换成SQL语句，调用libmysqlclient访问MYSQL\_Server。

DB\_Proxy的一种设计方案是：主线程使用epoll或libevent接收Client请求，将其压入请求队列；再起N个线程从队列中获取请求，同步执行SQL请求后再将结果返回给Client。

如果Server迟迟不返回呢？例如Server负载过大导致延时过大，Server进程Down掉，网络断开等等。这就会导致所有任务线程阻塞，同时主线程仍在持续不断的接收请求，队列满后即拒绝服务。

一般地，会有多个Server提供服务，如果当前Server出现问题的时候，能否立即发现从而切换到其它Server呢？

定时检测Server状态的操作不能影响DB\_Proxy执行正常的请求，有两种方案：

- 新起一个线程，定时执行`mysql_ping`，检查Server状态。如果`mysql_ping`阻塞了，那么需要在定时器里监听该线程状态：在执行`mysql_ping`之前启动定时器，Timeout之后仍未返回则切换到其它Server，并唤醒线程重新监测新的Server状态；

- 在主线程的epoll中异步执行`mysql_ping`，若超时后仍未返回，则切换Server。

第二种方案更简单，效率更高，但这需要一个异步的`mysql_ping`。接下来，本文主要介绍如何实现异步的`mysql_ping`。

#1 异步mysql\_ping

下载一份libmysqlclient的源码，找到`mysql_ping`的源码实现，如下：

```c
int STDCALL
mysql_ping(MYSQL *mysql)
{
  int res; 
  DBUG_ENTER("mysql_ping");
  res= simple_command(mysql,COM_PING,0,0,0);
  if (res == CR_SERVER_LOST && mysql->reconnect)
    res= simple_command(mysql,COM_PING,0,0,0);
  DBUG_RETURN(res);
}
```

#2 异步mysql_query

#3 示例
