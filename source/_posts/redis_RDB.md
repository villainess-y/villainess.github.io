
---
title: Redis——RDB持久化
date: 2021-06-15 22:46:25
updated: 2021-06-15 22:46:29
tags:
    - 学习
categories:
    - Redis
---

# 概述

Redis是内存数据库，所以需要将数据持久化到磁盘里。Redis提供了RDB持久化功能，这个功能可以将Redis在内存中的数据库状态保存到磁盘里，避免数据丢失。

# RDB文件的创建与载入

有两个Redis命令可以用于生成RDB文件，一个是 ***SAVE*** 一个是***BGSAVE*** 。

***SAVE*** 命令会阻塞Redis服务器进程，直到RDB文件创建完成为止，在服务器进程阻塞期间，服务器不能处理任何命令请求

***BGSAVE*** 命令会派生出一个子进程，然后由子进程负责创建RDB文件，服务器进程（父进程）继续处理命令请求

创建RDB文件的实际工作由 <u>***rdb.c/rdbSave***</u> 函数完成，***SAVE*** 命令和 ***BGSAVE*** 命令会以不同的方式调用这个函数，以下是伪代码：

```c
def SAVE():
	#创建RDB文件
	rdbSave();

def BGSAVE():
	#创建子进程
	pid = fork()
  if pid == 0:
		# 子进程负责创建RDB文件
		rdbSave()
    # 完成之后向父进程发送信号
    signal_parent()
  elif pid > 0
    # 父进程继续处理命令请求，并通过轮询等待子进程的信号
      handle_request_and_wait_signal()
  else:
		# 处理出错情况
		handle_fork_error()
```

RDB文件的载入工作是在服务器启动时自动执行的，所有Redis并没有专门用于载入RDB文件的命令，只要Redis服务器在启动时检测到RDB文件存在，它就会自动载入RDB文件。

AOF文件的更新频率通常比RDB文件的更新频率高，所以：

- 如果服务器开了AOF持久化功能，那么服务器会优先使用AOF文件来还原数据库状态
- 只有在AOF持久化功能处于关闭状态时，服务器才会使用RDB文件来还原数据库状态

`载入RDB文件的实际工作由rdb.c/rdbLoad函数完成` 

## SAVE命令执行时服务器状态

当***SAVE*** 命令时，Redis服务器会被阻塞，客户端发送的所有命令请求都会被拒绝。只有当服务器执行完***SAVE*** 命令，重新开始接受命令请求之后，客户端发送的命令才会被处理。

## BGSAVE命令执行时服务器状态

在***BGSAVE*** 命令执行期间，客户端发送的 ***SAVE*** 命令会被服务器拒绝，服务器禁止 ***SAVE***命令和 ***BGSAVE*** 命令同时执行是为了避免父进程（服务器进程）和子进程同时执行两个***RDBSAVE*** 调用，防止产生竞争条件。
其次，***BGSAVE***也会被服务器拒绝，因为同时执行两个 ***BGSAVE***命令也会产生竞争条件。
最后，***BGREWRITEAOF*** 和***BGSAVE*** 两个命令不能同时执行：

- 如果***BGSAVE*** 命令正在执行，那么客户端发送的 ***BGREWRITEAOF*** 命令会被延迟到***BGSAVE***命令执行完毕之后执行。
- 如果***BGREWRITEAOF***命令正在执行，那么客户端发送的***BGSAVE***命令会被绝

***BGREWRITEAOF*** 和 ***BGSAVE*** 两个命令的实际工作都是由子进程执行，同时执行会并发出两个子进程，并且两个子进程都会同时执行大量的磁盘写入操作，影响性能。

`服务器在载入RDB文件期间会一直阻塞，直到载入工作完成为止。`

# 自动间隔性保存

因为***BGSAVE***命令可以在不阻塞服务器进程的情况下执行，所以Redis允许用户通过设置服务器配置的save选项，让服务器每隔一段时间自动执行一次***BGSAVE***命令。

用户可以通过 <u>***save***</u> 选项设置多个保存条件，只要其中任意一个条件满足，服务器就会执行***BGSAVE*** 命令：

```shell
save 900 1
save 300 10
save 60 10000
```

900秒之内对数据库进行1次修改、300秒内对数据库进行10次修改、60秒内10000次修改

`这是服务器save选项的默认条件`

## 设置保存条件

服务器程序会根据 <u>***save***</u> 选项所设置的保存条件，设置服务器状态***redisServer*** 结构的***saveparams*** 属性。

***saveparams*** 属性是一个数组，数组中的每个元素都是一个 ***saveparam*** 结构，每个***saveparam***结构都保存了一个***save***选项设置的保存条件：

```c
struct redisServer{
	// ...
	struct saveParam *saveparams;
  long long dirty;
  time_t lastsave;
	//...
};

struct saveparam{
  // 秒
  time_t seconds;
  // 修改数
  int changes;
};
```

## dirty计数器和 lastsave属性

除了 ***saveparams*** 数组外，还有一个 ***dirty***计数器与 ***lastsave*** 属性：

- ***dirty*** 计数器记录距离上次成功执行 ***SAVE*** 命令或者 ***BGSAVE*** 命令之后，服务器对数据状态（所有数据库）进行了多少次修改（增删改）
- ***lastsave*** 属性是一个时间戳，记录了距离上次成功执行***SAVE*** 命令或者 ***BGSAVE*** 命令的时间

## 检查保存条件是否满足

Redis的服务器周期性操作函数***serverCron*** 默认每隔100毫秒就会执行一次，该函数用于对正在运行的服务器进行维护，它的其中一项工作就是检查***save*** 选项所设置的保存条件是否已经满足，如果满足的话，就执行***BGSAVE***命令。

***serverCron*** 函数检查保存条件的伪代码：

```c
def serverCron():
	# ...
  # 遍历所有保存条件
	for saveparam in server.saveparams:
			# 计算距离上次执行保存操作有多少秒
			save_interval = unixtime_now() - server.lastsave
      # 如果数据库状态的修改次数超过
      # 并且距离上次保存的时间超过
      # 执行保存
      if(save.dirty >= saveparam.changes and 
         save_interval > saveparam.seconds) :
			 BGSAVE()
 # ...        
```

# RDB文件结构

{% asset_img image-20210615144856868.png Hash : RDB file %}

RDB文件最开头是***REDIS*** 部分，这个部分长度为5字节，保存着"REDIS"无个字符，通过这个字符，可以快速检查载入的文件是否是RDB文件。

***db_version***长度为4字节，是一个字符串表示的整数，记录了RDB文件的版本号，例如："0006"

***databases***部分包含着零或任意多个数据库，以及各个数据库中的键值对数据：

- 如果服务器的数据库状态为空（所有数据库都是空的），那么这部分也是空，长度为0字节
- 如果服务器的数据库状态不为空（至少有一个数据库非空），那么这个部分也是非空

***EOF***常量的长度为1字节，表示RDB文件正文内容的结束，当程序读到这个值的时候，它就知道数据库所有的键值对都已经载入完成。

***check_sum*** 是一个8字节长的无符号整数，保存着一个校验和，是程序通过前面四个部分的内容进行计算所得到的数值，服务器在载入RDB文件时，将载入数据所计算出的校验和，与***check_sum***记录的校验和进行比对，以此来检查RDB文件是否出错或损坏。

## databases部分

每个非空数据库在RDB文件中都可以保存为 ***SELECTDB、db_bumber、key_value_pairs***  三个部分。

***SELECTDB*** 常量的长度为1字节，当读入程序遇到这个值的时候，它知道接下来要读入的是一个数据库号码。

***db_number***保存的是一个数据库号码，长度为1、2、5字节。当程序读入***db_number***部分之后，服务器会调用***SELECT*命令，根据读入的数据库号码进行数据库切换，使得之后读入的键值对可以载入到正确的饿数据库中。

***key_value_pairs*** 部分保存了数据库中的所有键值对数据，如果键值对带有过期时间，也会保存在一起。

## key_value_pairs部分

不带过期时间的键值对在RDB文件中由***TYPE、kye、value*** 三个部分组成。***TYPE***记录了***value***的类型，长度为1字节，值是以下常量之一：

- REDIS_RDB_TYPE_STRING
- REDIS_RDB_TYPE_LIST
- REDIS_RDB_TYPE_SET
- REDIS_RDB_TYPE_ZSET
- REDIS_RDB_TYPE_HASH
- REDIS_RDB_TYPE_LIST_ZIPLIST
- REDIS_RDB_TYPE_SET_INTSET
- REDIS_RDB_TYPE_ZSET_ZIPLIST
- REDIS_RDB_TYPE_HASH_ZIPLIST

程序会根据***TYPE***的值来决定如何读入和解释***value***的数据。
***key***总是一个字符串对象，它的编码方式和***REDIS_RDB_TYPE_STRING***类型的value一样。

带有过期时间的键在RDB文件中由***EXPIRETIME_MS、ms、TYPE、key、value***五个部分构成。

***EXPIRETIME_MS*** 常量的长度为1字节，它告知程序，接下来要读入的是一个单位为毫秒的过期时间

***ms*** 是一个8字节长的毫秒单位的UNIX时间戳，保存着键值对的过期时间。

## value的编码

value编码就是之前的对象编码。

### 1.字符串对象

如果字符串编码是***REDIS_ENCODING_RAW***，那么说明对象保存的是一个字符串值，根据字符串长度的不同，也有压缩和不压缩的方法来保存字符串：

- 如果字符串长度<= 20字节， 那么会被原样保存
- 如果字符串长度 > 20字节，那么会被压缩之后再保存

`以上两种条件，服务器需要打开RDB文件压缩功能才行，具体在redis.conf文件中rdbcompression选项说明，否则总是以无压缩方式保存`

{% asset_img image-20210615152931262.png Hash : zip list %}

未压缩的字符串仅有***len、string*** 两个部分。

压缩后的字符串包括***REDIS_RDB_ENC_LZF、compressed_len、origin_len、compressed_string*** 四个部分。

***REDIS_RDB_ENC_LZF*** 常量标志着字符串已经被 <u>***LZF***</u> 算法压缩过了，读入程序在碰到这个常量时，会根据之后的***compressed_len、origin_len、compressed_string*** 三个部分对字符串进行解压缩。

***compressed_len***记录的是字符串被压缩后的长度，***origin_len***记录的是原字符串的长度，***compressed_string***记录的是被压缩之后的字符串。

### 2.列表对象

***REDIS_RDB_TYPE_LIST***

### 3.集合对象

***REDIS_RDB_TYPE_SET***

### 4.哈希表对象

***REDIS_RDB_TYPE_HASH***

### 5.有序集合对象

***REDIS_RDB_TYPE_ZSET***

### 6.INTSET编码的集合

如果***TYPE***的值为 ***REDIS_RDB_TYPE_SET_INTSET*** ，那么value保存的就是一个整数集合对象，RDB文件保存这种对象的方法是先将整数集合转换成字符串对象，然后再保存到RDB文件中。

读入RDB文件的过程中，会把字符串对象转成整数集合。

### 7.ZIPLIST编码的列表、哈希表或有序集合

***TYPE***的值为 ***REDIS_RDB_TYPE_LIST_ZIPLIST、REDIS_RDB_TYPE_HASH_ZIPLIST、REDIS_RDB__TYPE_ZSET_ZIPLIST***，RDB文件保存这种对象，会先把压缩列表转成一个字符串对象，然后保存到RDB文件中。
读入RDB的过程中，程序会根据TYPE来转成原来的列表对象。

# 分析RDB文件

使用 <u>***od***</u>命令来分析Redis服务器产生的RDB文件,-c可以以ASCII编码方式打开，-x以16进制打开

# 重点回顾

- RDB文件用于保存和还原Redis服务器所有数据库中的所有键值对数据
- ***SAVE*** 命令由服务器进程直接执行保存操作，所以该命令会阻塞服务器
- ***BGSAVE*** 命令由子进程执行保存操作，所以该命令不会阻塞服务器
- 服务器状态中会保存所有用***save***命令选项设置的保存条件，当任意一个保存条件被满足时，服务器会自动执行***BGSAVE***命令
- RDB文件是一个经过压缩的二进制文件，由多个部分组成
- 针对于不同类型的键值对，RDB文件会有不同的方式来保存

