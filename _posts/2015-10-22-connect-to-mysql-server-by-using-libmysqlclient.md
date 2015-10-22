---
layout: post
title: 使用libmysqlclient连接到MySQL Server
categories: mysql 
tags: mysql 
---

C++程序访问MySQL服务器的重点在于两个：

- 如何连接
- 如何获得返回结果

应用程序要想访问数据库，必须使用数据库提供的编程接口。目前业界广泛被使用的API标准有ODBC和JDBC。

- [ODBC](https://msdn.microsoft.com/en-us/library/ms710252%28v=vs.85%29.aspx)是由微软提出的访问关系型数据库的C程序接口。
- [JDBC](http://www.oracle.com/technetwork/java/javase/jdbc/index.html)（Java Data Base Connectivity,java数据库连接）是一种用于执行SQL语句的Java API，可以为多种关系数据库提供统一访问，它由一组用Java语言编写的类和接口组成。

MySQL实现了三种Connector用于C/C++ 客户端程序来访问MySQL服务器：

 - [Connector/ODBC](https://dev.mysql.com/downloads/connector/odbc/)
 - [Connector/C++](https://dev.mysql.com/downloads/connector/cpp/)(JDBC)
 - [Connector/C](https://dev.mysql.com/downloads/connector/c/)(libmysqlclient)

关于如何安装和配置Connector，请参考相关官方文档。

在安装好Connector之后，我们就可以在程序中使用这些API来连接到MySQL数据库了。本文主要介绍的是使用libmysqlclient库来访问MYSQL数据库。

###1 主要数据结构

-  MYSQL

mysql数据库连接句柄。在执行任何数据库操作之前首先就需要创建一个MYSQL结构。

- MYSQL_RES

执行查询语句（SELECT, SHOW, DESCRIBE, EXPLAIN）返回的结果。

- MYSQL_ROW

用来表示返回结果中的一行数据。由于每行数据格式不一致，因此使用此结构来统一表示。调用mysql_fetch_row()可从MYSQL_RES中返回一个MYSQL_ROW结构

- MYSQL_FIELD

用来表示一个field信息的元数据（元数据，即描述数据的数据），包括field name，field type以及field size等。MYSQL_FIELD不包含field的值（MYSQL_ROW真正保存各field的值）

- MYSQL_FIELD_OFFSET

field在row中的索引值，从0开始。

###2 主要API

- mysql_init()

	```
	MYSQL *mysql_init(MYSQL *mysql)`
	```

	创建一个MYSQL对象。
	
- mysql_real_connect()

	```
	MYSQL *mysql_real_connect(MYSQL *mysql, const char *host, const char *user, const char *passwd, const char *db, unsigned int port, const char *unix_socket, unsigned long client_flag);
	```

	连接到数据库服务器。
	 
- mysql_real_query()

	```
	int mysql_real_query(MYSQL *mysql, const char *stmt_str, unsigned long length);
	```

	执行MySQL语句stmt_str，成功返回0

- mysql_store_result()

	```
	MYSQL_RES *mysql_store_result(MYSQL *mysql);
	```

	在执行完查询语句（mysql_store_result() or mysql_use_result()）后，调用此函数获得执行结果（result set）。如果执行正确且有结果返回，那么此函数返回非NULL的指针。
		
- mysql_affected_rows() 

	```
	my_ulonglong mysql_affected_rows(MYSQL *mysql);
	```

	如果执行的是UPDATE、INSERT和DELETE操作，那么MySQL会告诉你此操作影响了多少行（Rows）。调用此函数即能返回该值。有关在不同操作下此函数返回值的解释，详见[官方文档](https://dev.mysql.com/doc/refman/5.6/en/mysql-affected-rows.html)。在以下几种情况下函数会返回0：1）带有WHERE的UPDATE操作没有匹配任何行；2）调用之前没有执行任何query操作；3）对于SELECT操作，在调用mysql_store_reuslt()之前调用此函数。

在三种情况下会返回NULL：1）执行的语句不是查询语句，例如INSERT/UPDATE等；2）有result set但读取出错（到server的连接出问题）；3）调用malloc为result set分配空间出错（result set太大）。第一种情况可通过**mysql_field_count**()是否返回0来判断；后两种情况可通过 mysql_error()返回非空字符串或者 mysql_errno() 返回大于0来判断。

因此，一般情况下执行SQL语句的流程如下所示：

```
MYSQL_RES *result;
unsigned int num_fields;
unsigned int num_rows;

if (mysql_real_query(&mysql,query_string, srlen(query_string)))
{
    // error
}
else // query succeeded, process any data returned by it
{
    result = mysql_store_result(&mysql);
    if (result)  // there are rows
    {
        num_fields = mysql_num_fields(result);
        // retrieve rows, then call mysql_free_result(result)
    }
    else  // mysql_store_result() returned nothing; should it have?
    {
        if(mysql_field_count(&mysql) == 0)
        {
            // query does not return data
            // (it was not a SELECT)
            num_rows = mysql_affected_rows(&mysql);
        }
        else // mysql_store_result() should have returned data
        {
            fprintf(stderr, "Error: %s\n", mysql_error(&mysql));
        }
    }
}
```

### 3 解析返回结果

在上一节，假定我们成功地执行了语句，并获得了结果（MYSQL_RES结构）。那么怎么从MYSQL_RES中解析出我们想要的东西呢？

先来看一个示例：

```
#include <mysql.h>
#include <string>
#include <stdio.h>

using namespace std;

int main()
{
	if (argc != 2)
	{
		printf("Usage: ./test \"SQL Statement\"\n");
		return -1;
	}
	MYSQL mysql;
	mysql_init(&mysql);
	if (!mysql_real_connect(&mysql,"localhost","mysql","mysql","mytest",3306,NULL,0))//连接到本地数据库
	{
		fprintf(stderr, "Failed to connect to database: Error: %s\n",
		mysql_error(&mysql));
	}

	printf("connect to %s:%d success...\n", "localhost", 3306);

	string sql(argv[1]);
	if (mysql_real_query(&mysql, sql.c_str(), sql.length()))
	{
		//error
	}
	else 
	{ // 成功执行SQL语句
		MYSQL_RES* res = mysql_store_result(&mysql);//获得结果
		if (res) // sucess
		{
			printf("\"%s\" success\n", sql.c_str());
			int num_fields = mysql_num_fields(res);
			int num_rows = mysql_num_rows(res);
			printf("result: %d rows  %d fields\n", num_rows, num_fields);
			printf("----------------------------\n");

			//1. 获得列属性（名称）
			MYSQL_FIELD* fields;//数组，包含所有field的元数据
			fields = mysql_fetch_fields(res);
			for (int i = 0; i < num_fields; ++i)
			{
				printf("%s\t", fields[i].name);
			}
			printf("\n");

			//2. 获得每行的数据
			MYSQL_ROW row;
			while ((row = mysql_fetch_row(res)))
			{
				unsigned long *lengths;
				lengths = mysql_fetch_lengths(res);
				for (int i = 0; i < num_fields; ++i)
				{
					printf("%s\t",  row[i] ? row[i] : "NULL");
				}
				printf("\n");
			}
			printf("----------------------------\n");
			mysql_free_result(res);
		}
		else
		{
			//接下来就需要判断为什么res为NULL了
			
			int ret = mysql_field_count(&mysql);
			//printf("mysql_field_count %d\n", ret);
			if (ret == 0)
			{
				// 说明执行的是UPDATE/INSERT等这类操作
				int ret = mysql_affected_rows(&mysql);
				printf("Query OK, %d rows affected\n", ret);
			}
			else
			{
				fprintf(stderr, "Error: %s\n", mysql_error(&mysql));
			}
		}
	}
	mysql_close(&mysql);
	return 0;
}
```

可以看到，从MYSQL_RES中获取结果依靠的是两个函数：

- mysql_fetch_fields() 获取field元数据
- mysql_fetch_rows() 获取每一行的数据。每调用一次，该MYSQL_RES的row offset自动增1，因此可以在while循环中调用此函数，直到返回NULL。

以下是编译运行结果：

首先写个简单的makefile：

```
#Makefile 
test:test.cpp
	g++ -c `mysql_config --cflags` test.cpp
	g++ -o test test.o `mysql_config --libs`

.PHONY: clean

clean:
	rm -f *.o test
```

其中，可以使用mysql_config工具来获得安装Connector时候的头文件位置和库文件位置（[详见官方文档](http://dev.mysql.com/doc/refman/5.6/en/c-api-building-clients.html)）。

编译运行结果：

```
$ make
g++ -c `mysql_config --cflags` test.cpp
g++ -o test test.o `mysql_config --libs`

$ ./test "SELECT * FROM table1"
connect to localhost:3306 success...
"SELECT * from table1" success
result: 1 rows  3 fields
----------------------------
tid	sid		name	
123	12345	haha
----------------------------

$ ./test "UPDATE table1 set sid=12345 where tid=123"
connect to localhost:3306 success...
Query OK, 0 rows affected

$ ./test "UPDATE table1 set sid=54321 where tid=123"
connect to localhost:3306 success...
Query OK, 1 rows affected

$ ./test "SELECT * from table1"
connect to localhost:3306 success...
"SELECT * from table1" success
result: 1 rows  3 fields
----------------------------
tid	sid		name	
123	54321	haha
----------------------------
```

可以看到，对于UPDATE操作，如果设置的值与原来的值一样，那么`mysql_affected_rows`返回的是0。

可以对比以下我们的测试程序的输出结果和mysql命令行工具的输出结果：

```
mysql> select * from table1;
+-----+-------+------+
| tid | sid   | name |
+-----+-------+------+
| 123 | 12345 | haha |
+-----+-------+------+
1 row in set (0.00 sec)

mysql> UPDATE table1 set sid=54312 where tid=123;
Query OK, 1 row affected (0.04 sec)
Rows matched: 1  Changed: 1  Warnings: 0

mysql> UPDATE table1 set sid=54312 where tid=123;
Query OK, 0 rows affected (0.00 sec)
Rows matched: 1  Changed: 0  Warnings: 0

mysql> select * from table1;
+-----+-------+------+
| tid | sid   | name |
+-----+-------+------+
| 123 | 54312 | haha |
+-----+-------+------+
1 row in set (0.00 sec)
```

在运行可执行文件时提示

"./test: /usr/lib/x86_64-linux-gnu/libmysqlclient.so.18: no version information available (required by ./test)"

猜测可能是因为Connector库版本和mysql server版本不一致导致，但貌似不影响程序的执行，暂不管他了。
