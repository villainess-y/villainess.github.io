
---
title: Redis——AOF持久化
date: 2021-06-15 22:46:25
updated: 2021-06-15 22:46:29
tags:
    - 学习
categories:
    - Redis
---

# 概述

RDB持久化通过保存数据库中的键值对来记录数据库状态不同，AOF持久化是通过保存Redis服务器所执行的写命令来记录数据库状态的。

被写的AOF文件的所有命令都是以Redis对的命令请求协议格式保存的，因为Redis的命令请求协议是纯文本，因此可以直接打开一个AOF文件来观察里面的内容。

# AOF持久化的实现

AOF持久化功能的实现可分为命令追加（append）、文件写入、文件同步（sync)三个步骤。

## 命令追加

当AOF持久化功能处于打开状态时，服务器在写完一个命令之后，会以协议格式将被执行的写命令最佳到服务器状态的aof_buf缓冲区的末尾

```c
struct redisServer{
		//...
		//AOF 缓冲区
		sds aof_buf;
		//...
};
```

## AOF文件的写入与同步

Redis的服务器进程就是一个事件循环（loop）,这个循环中的文件事务负责接收客户端的命令请求，以及向客户端发送命令回复，而时间事件则负责执行像***serverCron***函数这样需要定时运行的函数。

因为服务器在处理文件时，可能会执行写命令，使得一些内容被追加到***aof_buf***缓冲区里面，所以服务器每次结束一个时间循环之前，都会调用***flushAppendOnlyField***函数，考虑是否需要将***aof_buf***缓冲区的内容写入和保存到AOF文件里：

```c
def eventLoop():
	while True:
			# 处理文件事件，接受命令请求以及发送命令回复
			# 处理命令请求时可能会有新内容被追加到aof_buf 缓冲区中
			processFileEvents()
      # 处理时间事件
      processTimeEvents()
      # 考虑是否要将aof_bus缓冲区的内容写入到AOF文件中
      flushAppendOnlyField()  
```

***flushAppendonlyFile*** 函数的行为由服务器配置的***appendfsync*** 选项的值来决定：

|      |
| ---- |