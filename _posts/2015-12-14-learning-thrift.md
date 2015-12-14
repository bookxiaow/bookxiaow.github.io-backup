---
layout: post
title: Thrift框架学习笔记
categories: Thrift
tags: thrift
---

Thrift是Facebook公司开发的一种开源RPC框架（[点我去官网](https://thrift.apache.org/)）。

它的最大特色就是跨语言平台。使用者只需要定义好接口文件，然后就可以自动生成目标语言的源代码。

也就是说，只要Client和Server使用相同的接口文件就可以互相通信，即使Client是由Python编写的，而Server是由C++或Java开发的。

而且，接口文件的格式很简单，也很容易理解。比如说，下面就是一个正确的thrift文件。

```
// simple_service.thrift

namespace cpp SimpleService

struct ReturnType {
		1: i32 rescode,
			   	2: i32 result
}

service SimpleService {
		ReturnType ping(1: i32 id, 2: i32 id2, 3: i32 id3),
}
```

Thrift说到底是一种RPC框架，因此其主体仍然是**服务**。Server提供服务，Client访问服务接口。

在上面的thrift文件中，我们定义了一种SimpleService服务，并且只包含一个接口ping。Client只要调用ping，
那么就能够触发Server端执行ping，并且获取Server端的执行结果。

Thrift文件使用了一种称为**Thrift IDL**的接口定义语言来定义接口。可以看到，用户可以像主流编程语言一样
使用include/typedef等语句，也支持自定义的struct/enum/exception/union等结构，还可以使用namespace来指定
编译后的源码的namespace。

有关Thrift IDL的语法，可以参考[官方网站](https://thrift.apache.org/docs/idl)，也可以参照官方给的一个[示例thrift文件](https://git-wip-us.apache.org/repos/asf/thrift/?p=thrift.git;a=tree;f=tutorial)。

#1 使用Thrift

在定义好Thritf文件后，就可以生成源码了。我们以C++为例

```sh
$ thrift --gen cpp ./simple_service.thrift
```

生成的源码位于gen-cpp目录下，有以下文件：

- simple_service_constants.cpp[.h] 常量(`simple_serviceConstants`)
- simple_service_types.cpp[.h] 自定义类型(`ReturnType`, `_ReturnType__isset`)
- SimpleService.cpp[.h] 服务接口
- SimpleService_server.skeleton.cpp 示例Server

上面()中表示该文件定义的类。在Thrift文件中定义的常量和结构均包含在这两组文件中。

SimpleService.cpp就比较复杂了，它包含了很多的类。其中最重要的就是`SimpleServiceProcessor`了。

## SimpleServiceProcessor

SimpleServiceProcessor继承自Thrfit框架中的TProcessor类，其关系如下图所示

![Thrift Processor](/image/thrift-processor.png)

TProcessor作为抽象类，定义了Processor的接口。Processor，顾名思义，是消息的处理单元：
它负责从输入读取消息，然后判断RPC函数的名称并调用该函数，再将函数返回结果写到输出中去。

TProcessor的process函数是纯虚函数，因此具体的Processor必须实现自己的process函数。好在这些都有thrift框架自动帮你生成。

以SimpleService为例，我们在SimpleService.cpp中看到了SimpleServiceProcessor的实现。

`SimpleServiceProcessor::process`首先实现了一组函数，对应各个服务函数。我们的例子中SimpleServiceProcessor就只有一个
`process_ping`，对应于thrift文件中的ping接口。

所有的这些接口函数全部放在map里面。以接口名称为索引，在process函数中解析出接口名称后，就可以从map里查找到对应的处理函数。

SimpleServiceProcessor并不会也没有必要帮助使用者实现具体的ping，因为这是使用者的工作。因此，SimpleServiceProcessor的
`process_ping`会直接去调用`iface_->ping`。

`iface_`是一个指向SimpleServiceIf对象的指针，在SimpleServiceProcessor的构造函数中传递该该指针。SimpleServiceIf同样是一个
抽象类，定义了具体消息处理器的接口。使用者应该继承该类，实现一个具体的消息处理器。

好了，我们已经知道消息在业务层面的处理流程了。那么接下来，我们需要知道消息从哪里来，以及到哪里去的问题。

## 消息的输入和输出

我们看一下`SimpleServiceProcessor::process`的实现，看它从哪儿读消息，以及如何将结果发送回给客户端

```cpp
bool SimpleServiceProcessor::process(boost::shared_ptr< ::apache::thrift::protocol::TProtocol> piprot, boost::shared_ptr< ::apache::thrift::protocol::TProtocol> poprot) {

  ::apache::thrift::protocol::TProtocol* iprot = piprot.get();                                                                                                           
  ::apache::thrift::protocol::TProtocol* oprot = poprot.get();                                                                                                           
  std::string fname;
  ::apache::thrift::protocol::TMessageType mtype;                                                                                                                        
  int32_t seqid;
  
  iprot->readMessageBegin(fname, mtype, seqid);                                                                                                                          
  
  if (mtype != ::apache::thrift::protocol::T_CALL && mtype != ::apache::thrift::protocol::T_ONEWAY) {                                                                    
    iprot->skip(::apache::thrift::protocol::T_STRUCT);
    iprot->readMessageEnd();
    iprot->getTransport()->readEnd();                                                                                                                                    
    ::apache::thrift::TApplicationException x(::apache::thrift::TApplicationException::INVALID_MESSAGE_TYPE);                                                            
    oprot->writeMessageBegin(fname, ::apache::thrift::protocol::T_EXCEPTION, seqid);
    x.write(oprot);
    oprot->writeMessageEnd();                                                                                                                                            
    oprot->getTransport()->flush();                                                                                                                                      
    oprot->getTransport()->writeEnd();                                                                                                                                   
    return true;
  } 
  
  return process_fn(iprot, oprot, fname, seqid);                                                                                                                         
} 
```

函数需要传入两个指向TProtocol的指针，并且调用`pioprot->readMessageBegin`获取到消息的fname（也就是函数调用名，例如"ping"）、mtype（message type）以及表示消息序号的seqid（在异步框架中用来作上下文id）。

可以看到，消息的输入和输出都需要借助于TProtocol对象。我们接下来就看一下TProtocol是如何实现消息的输入和输出的。

#2 Thrift框架

Thrift框架如下所示：

```
+-------------------------------------------+
| Server							|
| (single-threaded, event-driven etc)		|
+-------------------------------------------+
| Processor									|
| (compiler generated)						|
+-------------------------------------------+
| Protocol									|
| (JSON, compact etc)						|
+-------------------------------------------+
| Transport									|
| (raw TCP, HTTP etc)						|
+-------------------------------------------+
```

#3 总结
