---
layout: post
title: python中使用thrift协议
categories: python
tags: python thrift
---

最近因为工作需要，要访问服务器的thrift接口来批量干点事情，所以写了个python脚本来访问thrift接口，这里做一下笔记，以作备忘。

## 话不多说，先贴code

```py
#!/usr/bin/env python

import sys
sys.path.append('gen-py')
sys.path.append('/usr/lib/python2.6/site-packages')

from webdb import webdb_service
from webdb.ttypes import service_exception

from thrift import Thrift
from thrift.transport import TSocket
from thrift.transport import TTransport
from thrift.protocol import TBinaryProtocol

def main():
	
	# Make socket
	transport = TSocket.TSocket('127.0.0.1', 8990)
	
	# Buffering is critical. Raw sockets are very slow
	transport = TTransport.TFramedTransport(transport)
	
	# Wrap in a protocol
	protocol = TBinaryProtocol.TBinaryProtocol(transport)
	
	# Create a client to use the protocol encoder
	client = webdb_service.Client(protocol)
	
	try:
		# Connect!
		transport.open()
		
		uidFile = open('uids.txt')
		outputFile = open('uids_nick.txt', 'w')
		props = ['nick']
		for line in uidFile:
			uid = line.strip('\n')
			result = client.get_user_info(uid,props)
			
			nick_idx = result.key_index['nick']
			
			if len(result.dataset) == 0:
				continue
			nick = result.dataset[0][nick_idx].decode('utf-8')
			print "uid:%s, nick:%s" % (uid, nick)
			outputFile.write("%s %s\n" % (uid, nick.encode('utf-8')))
			
		outputFile.write("\n")
		# Close!
		transport.close()
		
	except Thrift.TException, tx:
		print '%s' % (tx.exception)

if __name__ == '__main__':
	main()

```

这个例子访问服务器的`get_user_info`接口根据uid查用户昵称信息，然后将结果保存到文件中。

## 需要注意的几点

1. 字符串编解码问题
2. thrift协议需要与server端保持一致

数据以'utf-8'格式保存在数据库里，然后经过网络传输到达本地后，首先需要 调用 `decode`方法将其解码成Unicode字符串。

在写入到文件或其它地方（保存时总要遵循一种编码方式）前，需要再调用`encode`方法将其重新按指定格式编码成字节流。
