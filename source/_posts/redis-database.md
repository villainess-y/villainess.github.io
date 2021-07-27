---
title: Redis——数据库
date: 2021-06-10 17:55:02
tags:
    - 学习
categories:
    - Redis
---


# 概述

本章将说明服务器保存数据库的方法，客户端切换数据库的方法，数据库保存键值对的方法，以及针对数据库的添加、删除、查看、更新操作的实现方法等。除此之外，本章还会说明服务器保存键的过期时间的方法，以及服务器自动删除过期键的方法。

# 服务器中的数据库

Redis服务器将所有的数据库都保存在服务器状态***redis.h/redisServer***结构的db数组中，db数组的每个项都是一个***redis.h/redisDb***结构，每个redisDb结构代表一个数据库：

```c
struct redisServer{
//...
// 一个数组，保存着服务器中的所有数据库
redisDb *db;
//...
};
```

在初始化服务器时，程序会根据服务器状态的dbnum属性来决定应该创建多少个数据库:

```c
struct redisServer{
// ...
// 服务器的数据库数量
int dbnum;
//...
};
```

dbmum属性的值由服务器配置的database选项决定，默认情况下为16.

# 切换数据库

每个Redis客户端都有自己的目标数据库，每当客户端执行数据库写命令或者数据库读命令的时候，目标数据库就为成为这些命令的操作对象。

默认情况下Redis客户端的目标数据库为0号数据库，但客户端可以通过执行***SELECT***命令来切换

```shell
127.0.0.1:6379> SET msg "hello world"
OK
127.0.0.1:6379> GET msg
"hello world"
127.0.0.1:6379> SELECT 2
OK
127.0.0.1:6379[2]> SET msg "another world"
OK
127.0.0.1:6379[2]> GET msg
"another world"
```

在服务器内部，客户端状态***redisClient***结构的db属性记录了客户端当前的目标数据库，这个属性是一个指向***redisDb***结构的指针：

```c
typedef struct redisClient{
//...
//记录客户端当前正在使用的数据库
redisDb *db;
//...
}redisClient;
```

***redisClient.db***指针指向***redisServer.db***数组的其中一个元素，而被指向的元素就是客户端的目标数据库。

# 数据库键空间

***redisDb***结构的dict字典保存了数据库中的所有键值对，我们将这个字典称为键空间：

```c
typedef struct redisDb{
//...
// 键空间
dict *dict;
//..
}redisDb;
```

键空间和用户所见的数据库是直接对应的：

- 键空间的键也就是数据库的键，每个键都是一个字符串对象
- 键空间的值也就是数据库的值，每个值可以是Redis对象中五种对象的任意一种

```shell
redis> SET message "hello world"
OK
redis> RPUSH alphabet "a" "b" "c"
(integer)3
redis> HSET book name "Redis in Action"
(integer)1
redis> HSET book author "Josiah L. Carlson"
(integer)1
redis> HSET book publisher "Manning"
(integer)1
```

{% asset_img image-20210610163951729.png Hash:20210610163951729  %}

```shell
# 添加新键
redis> SET data "2001.01.01"
OK
# 删除键
redis> DEL book
(integer)1
#更新键
redis> SET message "balabala"
OK
#键取值
redis> GET message
"hello world"
redis> LRANGE alphabet 0 -1
1) "a"
2) "b"
3) "c"
```

## 读写键空间是维护操作

当使用Redis命令对数据库进行读写时，服务器不仅会对键空间执行指定的读写操作，还会执行一些额外的维护操作，其中包括：

- 在读取一个键后(读写操作)，服务器会根据键是否存在来更新服务器的键空间命中次数(hit)或者键空间不命中次数(miss)，这两个值可以在***INFO status***命令的 <u>***keyspace_hits***</u>属性和 <u>***keyspace_misses***</u>属性中查看
- 在读取一个键后，服务器会更新LRU
- 如果在读取一个键是发现该键已经过期，那么服务器会先删除这个过期键，然后再执行余下的其他操作
- 如果有客户端使用***WATCH***命令监视了某个键，那么服务器在对被监视的键修改之后，会把这个键标记为脏(dirty)，从而让事务程序注意到这个键已经被修改过
- 服务器每次修改一个键之后，都会对脏键计数器的值+1，这个计数器会触发服务器的持久化以及复制操作
- 如果服务器开启了数据库通知功能，那么在对键进行修改之后，服务器将按配置发送相应的数据库通知

# 设置键的生存时间或者过期时间

通过***EXPIRE***命令或者***PEXPIRE***命令，客户端可以以秒或者毫秒精度为数据库中的某个值设置生存时间（Time To Live TTL），在指定时间过后，服务器就会自动删除生存时间为0的键：

```shell
127.0.0.1:6379[2]> EXPIRE MSG 10
(integer) 1
127.0.0.1:6379[2]> get MSG
"another world"
127.0.0.1:6379[2]> get MSG
(nil)
```

`SETTEX命令可以在设置一个字符串键的同时为键设置过期时间，但只能用于字符串键。`

与***EXPIRE***命令和***PEXPIRE***命令类似，客户端可以通过***EXPIREAT***或者***PEXPIREAT***命令，以秒或者毫秒精度给数据库中的某个键设置过期时间（expire time）。过期时间是个UNIX时间戳，当键的过期时间来临时，服务器就会自动从数据库中删除这个键。

***TTL***命令和***PTTL***命令接受一个带有生存时间或者过期时间的键，返回这个键的剩余生存时间，也就是返回距离这个键被服务器自动删除还有多长时间：

```shell
127.0.0.1:6379> SET key valueOK127.0.0.1:6379> EXPIRE key 1000(integer) 1127.0.0.1:6379> TTL key(integer) 997
```

## 设置过期时间

Redis有四个不同的命令来用于设置键的生存时间或者过期时间：

- **EXPIRE <key> <ttl> **命令用于将键key的生存时间设置为ttl秒
- **PEXPIRE <key> <ttl> **命令用于将键key的生存时间设置为ttl毫秒
- **EXPIREAT <key> <timestamp> **命令用于将键key的过期时间设置为timestamp所指定的秒数时间戳
- **PEXPIREAT <key> <timestamp> **命令用于将键key的过期时间设置为timestamp所指定的毫秒数时间戳

`虽然有多种不同单位和不同形式的设置命令，但是最终都是使用`***PEXPIREAT***`命令来实现的`

```c
# EXPIRE 转换成 PEXPIRE:def EXPIRE(key,ttl_in_sec):	ttl_in_ms = sec_to_ms(ttl_in_sec)  PEXPIRE(key,ttl_in_ms)    # PEXPIRE 转换成 PEXPIREAT:def PEXPIRE(key,ttl_in_ms):	#获取以毫秒计算的当前UNIX时间戳	now_ms = get_current_unix_timestamp_in_ms()	#当前时间加上TTL，得出毫秒格式的键过去时间  PEXPIREAT(key,now_ms+ttl_in_ms)# EXPIREAT 转换成 PEXPIREATdef EXPIREAT(key,expire_time_in_sec):      # 将过期时间从秒转换成毫秒 	expire_time_in_ms = sec_to_ms(expire_time_in_sec)  PEXPIREAT(key,expire_time_in_ms)  
```

{% asset_img image-20210610174145514.png Hash:20210610174145514  %}

## 保存过期时间

redisDb结构的***expires***字典保存了数据库中所有键的过期时间，我们称这个字典为过期字典：

- 过期字典的键是一个指针，这个指针指向键空间中的某个键对象（某个数据库键）
- 过期字典的值是一个long long类型的整数，这个整数保存了键所指向的数据库键的过期时间——一个毫秒精度的UNIX时间戳

```c
typedef struct redisDb{//...// 过期字典dict *expires;//...}redisDb;
```

当客户端执行***PEXPIREAT***命令为一个数据库键设置过期时间时，服务器会在数据库的过期字典中关联给定的数据库键和过期时间。

{% asset_img image-20210610180450860.png Hash:20210610180450860  %}

```shell
redis> PEXPIREAT message 1391234400000(integer)1
```

{% asset_img image-20210610181326281.png Hash:20210610180450860  %}