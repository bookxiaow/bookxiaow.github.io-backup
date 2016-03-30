---
layout: post
title: C++ forbids taking the address of an unqualified or parenthesized non-static member function to form a pointer to member function
categories: cpp 
tags: cpp
---


最近在用boost_thread库写多线程程序时，会碰到这样的编译错误：

```
error: ISO C++ forbids taking the address of an unqualified or parenthesized non-static member function to form a pointer to member function.
```

代码如下：

```cpp
#include <iostream>
#include <boost/thread.hpp>

using namespace std;

class Test {
public:
	Test() {
		m_pThread = new boost::thread(boost::bind(&thread_func, this));
		m_pThread->detach();
	}

	void thread_func(){
		while(1) {
			cout << "hello, world" << endl;
			sleep(3);
		}
	}

private:
	boost::thread * m_pThread;
};

int main()
{
	Test t;

	while(1){}

	return 0;
}
```

参考了stackoverflow上的回答，只需要将第9行代码改成下面这样就行：

```cpp
	m_pThread = new boost::thread(boost::bind(&Test::thread_func, this));
```

很奇怪，明明是在类的构造函数里面，为什么还需要加"Test::"前缀限制。看来是在获取成员函数指针时，无论是在哪里，总是需要加类名限定值的。