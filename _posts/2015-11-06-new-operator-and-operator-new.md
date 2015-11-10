---
layout: post
title: operator new 和 new operate
categories: c++
tags: c++
---

本文参考 [cplusplus手册](http://www.cplusplus.com/reference/new/operator%20new/)

以下是对 C++ `operator new()` 和 new operator的小结。

首先区分一点，`new operator`是操作符，遵从C++语言定义的语义。例如：

```cpp
int * pInt = new int(3);
```

而`operator new()`是可以被类重载的操作符（static 成员函数），如果不重载，则默认是全局命令空间的`void * ::operator new(size_t sz)`。

在执行`new operator`时，会首先调用`operator new()`分配所需内存，接着再初始化内存（自定义类调用构造函数）。

C++标准库中定义有3种形式的`operator new`：

```cpp
// throwing，出错抛出std::bad_alloc异常
void* operator new (std::size_t size) throw (std::bad_alloc);

// nothrow
void* operator new (std::size_t size, const std::nothrow_t& nothrow_value) throw();

// placement, 不重新分配内存，在ptr指向的内存上调用构造函数
void* operator new (std::size_t size, void* ptr) throw();
```

示例如下：

```cpp
// operator new example
#include <iostream>     // std::cout
#include <new>          // ::operator new,标准库里的operator new

struct MyClass {
  int data[100];
  MyClass() {std::cout << "constructed [" << this << "]\n";}
};

int main () {

  std::cout << "1: ";
  MyClass * p1 = new MyClass;
      // allocates memory by calling: operator new (sizeof(MyClass))
      // and then constructs an object at the newly allocated space

  std::cout << "2: ";
  MyClass * p2 = new (std::nothrow) MyClass;
      // allocates memory by calling: operator new (sizeof(MyClass),std::nothrow)
      // and then constructs an object at the newly allocated space

  std::cout << "3: ";
  new (p2) MyClass;
      // does not allocate memory -- calls: operator new (sizeof(MyClass),p2)
      // but constructs an object at p2

  // Notice though that calling this function directly does not construct an object:
  std::cout << "4: ";
  MyClass * p3 = (MyClass*) ::operator new (sizeof(MyClass));
      // allocates memory by calling: operator new (sizeof(MyClass))
      // but does not call MyClass's constructor

  delete p1;
  delete p2;
  delete p3;

  return 0;
}
```

运行结果如下：

```sh
$ ./test 
1: constructed [0x1091010]
2: constructed [0x10911b0]
3: constructed [0x10911b0]
4: 
```

在3种形式的operator new中，前两种位于全局namespace中（不在std namespace）中，而且应用程序可以直接使用，无需include 头文件。

### 重载 `operator new()`

在重载`operator new()`之前，需要明确一点：

如果在自定义类中重载了任何形式的`operator new()`，那么任何针对该类的`new operator`都只会去类中寻找对应的`operator new()`。

比方说，在类中自定义了`void * operator new(size_t sz, void *ptr)`，那么如果调用`T *p = new T()`,就会出错：

```sh
error: no matching function for call to 'T::operator new(long unsigned int)'
note: candidate is:
note: static void* T::operator new(size_t, void*) [with size_t = long unsigned int]
```

在自定义类中重载`operator new()`时，可以添加任何参数，但需要保证：函数返回`void *`；函数的第一个参数是`size_t sz`，表示要分配的内存大小。

可以直接通过类限定符(::)调用重载的`operator new()`，也可以在调用`new operator`时指定对应的参数即可：

```cpp
// void * Test::operator new(size_t sz, void *arg1, int arg2);
Test *p1 = reinterpret_cast<Test*>(Test::operator new(sz, arg1, arg2); // just allocate memory, no construction occurs.
Test *p2 = new(arg1, arg2) Test();
```

### 重载 `operator new[]()`

与 `operator new()`类似。

### 重载 `operator delete()`

> There are global and class-scoped operator delete functions. Only one operator delete function can be defined for a given class; if defined, it hides the global operator delete function. The global operator delete function is always called for arrays of any type.

> There are global and class-scoped operator delete functions. Only one operator delete function can be defined for a given class; if defined, it hides the global operator delete function. The global operator delete function is always called for arrays of any type.

```cpp
void operator delete( void * );
void operator delete( void *, size_t );
```

任何类只能重载上面两种形式中的一种，如果重载了，那么在调用`delete pT`时就会去调用类自定义的函数。

### 使用心得

一般情况下，无需重载`operator new()`和`operator delete()`，但是在一些特殊情况下可以通过重载这两个操作符来实现自己的缓冲区管理。
