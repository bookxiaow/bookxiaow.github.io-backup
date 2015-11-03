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

继续跟踪`simple_command`:

```c
#define simple_command(mysql, command, arg, length, skip_check) \
  ((mysql)->methods \
    ? (*(mysql)->methods->advanced_command)(mysql, command, 0, \
                                            0, arg, length, skip_check, NULL) \
    : (set_mysql_error(mysql, CR_COMMANDS_OUT_OF_SYNC, unknown_sqlstate), 1))
```

`(mysql)->methods`是一组函数指针，它的`advanced_command`结构体变量应该被初始化为`cli_advanced_command`(`static MYSQL_METHODS client_methods`):

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

#if !defined(EMBEDDED_LIBRARY)
  /*
    If auto-reconnect mode is enabled check if connection is still alive before
    sending new command. Otherwise, send() might not notice that connection was
    closed by the server (for example, due to KILL statement), and the fact that
    connection is gone will be noticed only on attempt to read command's result,
    when it is too late to reconnect. Note that such scenario can still occur if
    connection gets killed after this check but before command is sent to
    server. But this should be rare.
  */
  if ((command != COM_QUIT) && mysql->reconnect && !vio_is_connected(net->vio))
    net->error= 2;
#endif

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

#if defined(CLIENT_PROTOCOL_TRACING)
  switch (command)
  {
  case COM_STMT_PREPARE:
    MYSQL_TRACE_STAGE(mysql, WAIT_FOR_PS_DESCRIPTION);
    break;

  case COM_STMT_FETCH:
    MYSQL_TRACE_STAGE(mysql, WAIT_FOR_ROW);
    break;

  /* 
    No server reply is expected after these commands so we reamin ready
    for the next command.
 */
  case COM_STMT_SEND_LONG_DATA: 
  case COM_STMT_CLOSE:
  case COM_REGISTER_SLAVE:
  case COM_QUIT:
    break;

  /*
    These replication commands are not supported and we bail out
    by pretending that connection has been closed.
  */
  case COM_BINLOG_DUMP:
  case COM_BINLOG_DUMP_GTID:
  case COM_TABLE_DUMP:
    MYSQL_TRACE(DISCONNECTED, mysql, ());
    break;

  /*
    After COM_CHANGE_USER a regular authentication exchange
    is performed.
  */
  case COM_CHANGE_USER:
    MYSQL_TRACE_STAGE(mysql, AUTHENTICATE);
    break;

  /*
    Server replies to COM_STATISTICS with a single packet 
    containing a string with statistics information.
  */
  case COM_STATISTICS:
    MYSQL_TRACE_STAGE(mysql, WAIT_FOR_PACKET);
    break;

  /*
    For all other commands we expect server to send regular reply which
    is either OK, ERR or a result-set header.
  */
  default: MYSQL_TRACE_STAGE(mysql, WAIT_FOR_RESULT); break;
  }
#endif

  result=0;
  if (!skip_check)
  {
    result= ((mysql->packet_length= cli_safe_read_with_ok(mysql, 1, NULL)) ==
             packet_error ? 1 : 0);

#if defined(CLIENT_PROTOCOL_TRACING)
    /*
      Return to READY_FOR_COMMAND protocol stage in case server reports error 
      or sends OK packet.
    */
    if (!result || mysql->net.read_pos[0] == 0x00)
      MYSQL_TRACE_STAGE(mysql, READY_FOR_COMMAND);
#endif
  }

end:
  DBUG_PRINT("exit",("result: %d", result));
  DBUG_RETURN(result);
}
```

#2 异步mysql_query

#3 示例
