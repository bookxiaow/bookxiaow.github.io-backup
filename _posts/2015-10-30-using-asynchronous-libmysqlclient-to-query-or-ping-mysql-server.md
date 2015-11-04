---
layout: post
title: 异步调用mysql_ping来检查MySQL Server状态
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

下载一份[libmysqlclient的源码](http://downloads.mysql.com/archives/get/file/mysql-connector-c-6.1.5-src.tar.gz)，找到`mysql_ping`的源码实现，如下：

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

继续跟踪`simple_command`:

```c
#define simple_command(mysql, command, arg, length, skip_check) \
  ((mysql)->methods \
    ? (*(mysql)->methods->advanced_command)(mysql, command, 0, \
                                            0, arg, length, skip_check, NULL) \
    : (set_mysql_error(mysql, CR_COMMANDS_OUT_OF_SYNC, unknown_sqlstate), 1))
```

这个宏调用了存放在`mysql->methods`里面的函数指针，不能继续跟踪下去了，除非知道这个指针被设置成了什么值。

全文搜索一下找到这么一个静态变量`static MYSQL_METHODS client_methods`。应该就是这个了。

```c
my_bool
cli_advanced_command(MYSQL *mysql, enum enum_server_command command,
		     const uchar *header, size_t header_length,
		     const uchar *arg, size_t arg_length, my_bool skip_check,
                     MYSQL_STMT *stmt)
{
  NET *net= &mysql->net;
  my_bool result= 1;
  my_bool stmt_skip= stmt ? stmt->state != MYSQL_STMT_INIT_DONE : FALSE;
  DBUG_ENTER("cli_advanced_command");

  if (mysql->net.vio == 0)
  {						/* Do reconnect if possible */
    if (mysql_reconnect(mysql) || stmt_skip)
      DBUG_RETURN(1);
  }
  if (mysql->status != MYSQL_STATUS_READY ||
      mysql->server_status & SERVER_MORE_RESULTS_EXISTS)
  {
    DBUG_PRINT("error",("state: %d", mysql->status));
    set_mysql_error(mysql, CR_COMMANDS_OUT_OF_SYNC, unknown_sqlstate);
    DBUG_RETURN(1);
  }

  net_clear_error(net);
  mysql->info=0;
  mysql->affected_rows= ~(my_ulonglong) 0;
  /*
    Do not check the socket/protocol buffer on COM_QUIT as the
    result of a previous command might not have been read. This
    can happen if a client sends a query but does not reap the
    result before attempting to close the connection.
  */
  net_clear(&mysql->net, (command != COM_QUIT));

  MYSQL_TRACE_STAGE(mysql, READY_FOR_COMMAND);
  MYSQL_TRACE(SEND_COMMAND, mysql, (command, header_length, arg_length, header, arg));

  if (net_write_command(net,(uchar) command, header, header_length,
			arg, arg_length))
  {
    DBUG_PRINT("error",("Can't send command to server. Error: %d",
			socket_errno));
    if (net->last_errno == ER_NET_PACKET_TOO_LARGE)
    {
      set_mysql_error(mysql, CR_NET_PACKET_TOO_LARGE, unknown_sqlstate);
      goto end;
    }
    end_server(mysql);
    if (mysql_reconnect(mysql) || stmt_skip)
      goto end;
    
    MYSQL_TRACE(SEND_COMMAND, mysql, (command, header_length, arg_length, header, arg));
    if (net_write_command(net,(uchar) command, header, header_length,
			  arg, arg_length))
    {
      set_mysql_error(mysql, CR_SERVER_GONE_ERROR, unknown_sqlstate);
      goto end;
    }
  }

  MYSQL_TRACE(PACKET_SENT, mysql, (header_length + arg_length)); 

  result=0;
  if (!skip_check)
  {
    result= ((mysql->packet_length= cli_safe_read_with_ok(mysql, 1, NULL)) ==
             packet_error ? 1 : 0);
  }

end:
  DBUG_PRINT("exit",("result: %d", result));
  DBUG_RETURN(result);
}
```

MySQL Client和 MySQL Server之间依据MySQL协议进行数据传输。而`cli_advanced_command`函数就是用来实现数据传输功能的。

先看一下函数的参数：

- `MYSQL *mysql` mysql句柄，包含有关连接的所有信息，例如连接socket，连接状态等等
- `enum enum_server_command command` 命令码，例如连接/查询/建表/删表等等
- `const uchar *header, size_t header_length` 头部信息
- `const uchar *arg, size_t arg_length` 附加参数
- `my_bool skip_check` 标志位，是否等待返回
- `MYSQL_STMT *stmt` 略过

可以看出来，MySQL协议消息是由3部分组成：

| 命令码 | 头部 | 参数 |
| --- | --- | --- |

再看函数体，

- 首先，检查连接状态 
- 接着，调用`net_write_command`发送消息
- 最后，如果`skip_check`为false，那么调用`cli_safe_read_with_ok`读取返回结果

所以，要实现异步`mysql_ping_async`的第一步就是，设置`skip_check`参数为true来调用`simple_command`,这样就只发送数据：

```c
int STDCALL
mysql_ping_async(MYSQL *mysql)
{
    int res; 
    DBUG_ENTER("mysql_ping_async");
    res = simple_command(mysql, COM_PING, 0, 0, 1);
    DBUG_RETURN(res);
}
```

接着，提供接口来读取ping的结果：

```c
int STDCALL
mysql_read_ping(MYSQL *mysql)
{
	int res; 
	DBUG_ENTER("mysql_read_ping");
	res = cli_safe_read_with_ok(mysql, 1, NULL);
	return (res == packet_error) ? 1 : 0; 
}
```

最后，要在epoll/libevent中监听ping的结果，我们需要知道到Server的TCP连接的fd。

```c
int get_mysql_fd(MYSQL *mysql)
{
	return mysql->net.fd;
}
```

最后，就可以编译出新的libmysqlclient.so文件。

#2 示例

有了异步的`mysql_ping_async`我们就可以在epoll/libevent中定期监测MySQL Server了。

Todo
