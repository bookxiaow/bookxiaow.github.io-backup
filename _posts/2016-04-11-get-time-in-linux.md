---
layout: post
title: Linux C如何获取时间
categories: Linux_C
tags: time
---

程序中，经常涉及到两种时间：

- 日历时间。需要知道现在的时间是多少，比如2016/04/11 11:00:00
- 相对时间。比如一个操作前后经过了多少时间

## 日历时间

### 获取`time_t`

在Linux C程序中需要获取到当前日历时间，首先需要调用以下两个函数获取到从Epoch, 1970-01-01 00:00:00 +0000 (UTC)时间点到现在经历的时间：

```c
#include <time.h>
time_t time(time_t* t);

#include <sys/time.h>
int gettimeofday(struct timeval *tv, struct timezone *tm);
```

其中，time只返回秒数，而gettimeofday更精确一点，返回秒数和微秒数：

```c
struct timeval {
	time_t		tv_sec;
	suseconds_t	tv_usec;
};
```

### 转换成可读字符串或"broken-down time"

而将一个time_t时间转换成我们认识的年月日时分秒，需要知道时区，比如常用的[UTC](https://zh.wikipedia.org/wiki/%E5%8D%8F%E8%B0%83%E4%B8%96%E7%95%8C%E6%97%B6)，中国处于UTC+8。

可以使用以下函数将日历时间转换成可读的时间：

```c
char *asctime(const struct tm *tm);
char *ctime(const time_t *timep);

struct tm *gmtime(const time_t *timep);
struct tm *localtime(const time_t *timep);
```

这里有一种新的数据接口`struct tm`，用来存储转换后的年月日时分秒等信息：

```c
struct tm {
	int tm_sec;         /* seconds */
    int tm_min;         /* minutes */
    int tm_hour;        /* hours */
    int tm_mday;        /* day of the month */
    int tm_mon;         /* month */
    int tm_year;        /* year */
    int tm_wday;        /* day of the week */
    int tm_yday;        /* day in the year */
    int tm_isdst;       /* daylight saving time */
};
```

当然，相比于这种"broken-down time"，程序更多的是需要将时间转换成可读的ASCII文本：

"Mon Apr 11 11:27:12 2016"

所以，

- 如果需要转换成`struct tm`结构，调用gmtime或者localtime；

> 注：gmtime默认转换成0时区的标准时间；localtime则转换成系统设置时区的标准时间。
 
- 如果需要转换成可读字符串，调用ctime

当然，可以也可以调用`asctime`将`struct tm`转换成ASCII字符串。

上述4个函数，均提供了线程安全的版本：

```c
char *asctime_r(const struct tm *tm, char *buf);
char *ctime_r(const time_t *timep, char *buf);
struct tm *gmtime_r(const time_t *timep, struct tm *result);
struct tm *localtime_r(const time_t *timep, struct tm *result);
```

相比于原来的函数，线程安全版将转换后的结果存储在用户自定义的缓冲区内。