
---
title: Redis——对象
date: 2021-06-09 22:46:25
updated: 2021-06-15 22:46:29
tags: 
    - 学习
    - 数据结构
categories: 
    - Redis
---

# 概述

​		Redis并没有直接使用前面介绍的SDS,跳表，字典，压缩链表，整数集合等数据结构来实现键值对数据库，而是基于这些数据结构创建了一个对象系统，这个系统包含***字符串对象***、***列表对象***、***哈希对象***、***集合对象***、***有序集合***对象这五种类型的对象，每个对象都用到了至少一种前面介绍的数据结构。

​		通过这五种不同类型的对象，Redis可以在执行命令之前，根据对象的类型来判断一个对象是否可以执行给定的命令，可以针对不同的使用场景，为对象设置多种不同的数据结构实现，从而优化对象在不同场景下的使用效率

# 对象的类型与编码

​		Redis使用对象来表示数据库中的键和值，每次在Redis的数据库中新建一个键值对时，至少会创建两个对象，一个对象用作键(键对象)，另一个用于值(值对象)。

```c
typedef struct redisObject{
  //类型
  unsigned type:4;
  //编码
  unsigned encoding:4;
  //指向底层实现数据结构的指针
  // void *ptr;
  ....
}robj;
```

## 类型

|   类型常量   |  对象的名称  | TYPE命令的输出 |
| :----------: | :----------: | -------------- |
| REDIS_STRING |  字符串对象  | "string"       |
|  REDIS_LIST  |   列表对象   | "list"         |
|  REDIS_HASH  |   哈希对象   | "hash"         |
|  REDIS_SET   |   集合对象   | "set"          |
|  REDIS_ZSET  | 有序集合对象 | "zset"         |

```shell
#键，值均为字符串对象
redis > SET msg "hello world"
OK
redis > TYPE msg
string
#键为字符串，值为列表
redis > RPUSH numbers 1 3 5
(integer)6
redis > TYPE numbers
list
......
```

## 编码和底层实现

​		对象的prt指针指向对象的底层实现数据结构，而这些数据结构由对象的encoding属性决定。

​		表列出了每种类型的对象可以使用的编码

|     类型     |           编码            |              对象              |
| :----------: | :-----------------------: | :----------------------------: |
| REDIS_STRING |    REDIS_ENCODING_INT     |   使用整数值实现的字符串对象   |
| REDIS_STRING |   REDIS_ENCODING_EMBSTR   | embstr编码的简单动态字符串实现 |
| REDIS_STRING |    REDIS_ENCODING_RAW     |        简单动态字段实现        |
|  REDIS_LIST  |  REDIS_ENCODING_ZIPLIST   |          压缩列表实现          |
|  REDIS_LIST  | REDIS_ENCODING_LINKEDLIST |           双链表实现           |
|  REDIS_HASH  |  REDIS_ENCODING_ZIPLIST   |          压缩列表实现          |
|  REDIS_HASH  |     REDIS_ENCODING_HT     |            字典实现            |
|  REDIS_SET   |   REDIS_ENCODING_INTSET   |          整数集合实现          |
|  REDIS_SET   |     REDIS_ENCODING_HT     |            字典实现            |
|  REDIS_ZSET  |  REDIS_ENCODING_ZIPLIST   |          压缩列表实现          |
|  REDIS_ZSET  |  REDIS_ENCODING_SKIPLIST  |         跳表和字典实现         |

使用***OBJECT ENCODING***命令可以查看一个数据库键的值对象的编码

```shell
redis > SET msg "hello world"
OK
redis > OBJECT ENCODING msg
"embstr"
redis > SET story "long long long long long ago ...."
OK
redis > OBJECT ENCODING story
"raw"
```

通过encoding属性来设定对象所使用的编码，而不是为特定类型的对象关联一种固定的编码，极大地提升了Redis的灵活性和效率，因为Redis可以根据不同的使用场景来为一个对象设置不同的编码，从而优化对象在某一场景下的效率。

- 因为压缩列表比双向链表节约内存，所以元素数量少时，在内存中以连续内存块保存可以更快载入到缓存中
- 随着列表对象包含的元素越来越多，使用压缩列表的优势逐渐消失，对象就会将底层实现从压缩列表转向功能性更强、也适合保存大量元素的双向链表上

# 字符串对象

字符串对象的编码可以是***int***、***raw***或者***embstr***。

如果一个字符串对象保存的是整数值，并且这个整数值可以用long类型来表示，那么字符串对象将会把整数值保存在字符串对象结构的ptr属性里(把void* 转换成long)，并将字符串对象的编码设置为***int***。

```shell
redis > SET number 10086
OK
redis > OBJECT ENCODING number
"int"
```

如果一个字符串对象保存的是一个字符串值，并且这个字符串值的长度大于44字节，那么字符串对象将使用一个SDS来保存这个字符串值，并将对象的编码设置为***raw***。

如果这个字符串的长度小于44([最新版本](https://blog.csdn.net/qq_33996921/article/details/105226259))字节，那么字符串对象将使用***embstr***编码的方式来保存这个字符串。

```
redis> set story "Long, long long long long long long ago ..."
OK
redis> STRLEN story
(integer) 43
redis> OBJECT ENCODING story
"embstr"
redis> set story "Long, long long long long long long ago ....."
OK
redis> STRLEN story
(integer) 45
redis> OBJECT ENCODING story
"raw"
```

embstr编码是专门用于保存短字符串的一种优化编码方式，这种编码方式和raw编码方式一样，都使用redisObject结构和sdshdr结构来表示字符串对象，然是raw编码会调用两次内存分配，来分别创建redisObject结构和sdshdr结构，而embstr编码只会调用一次内存分配函数来分配一块连续的空间，空间中依次包含redisObject和sdshdr两个结构。

可以用long double类型表示的浮点数在Redis中也是作为字符串值来保存的，如果我们要保存一个浮点数到字符串对象里，那么程序会先把这个浮点数转成字符串值然后再保存该字符串值

```shell
redis> set pi 3.14
OK
redis> OBJECT ENCODING pi
"embstr"
redis> 
```

如果需要对浮点型数据进行操作，程序会把字符串对象里的字符串转换回浮点数，执行完操作之后，再转换回字符串，保存在字符串对象里。

## 编码的转换

int编码的字符串对象和embstr编码的字符串对象在条件满足的情况下，会被转换为raw编码的字符串对象。

对于int编码的字符串对象，如果执行完操作之后不再是整数值而是字符串值，则对应的字符串对象的编码会从int变成raw。

```shell
redis> SET number 11
OK
redis> OBJECT ENCODING number
"int"
redis> APPEND number "is a good number!"
(integer) 19
redis> GET number
"11is a good number!"
redis> OBJECT ENCODING number
"raw"
```

Redis并没有为embstr编码的字符串对象编写任何的修改程序，所以embstr编码的字符串对象实际上是只读的。让我们使用命令对embstr编码对象执行操作时，程序会先将对象的编码从embstr转换成raw，然后再执行相应的命令。(实际这点并未复现)

# 列表对象

列表对象的编码可以是***ziplist*** 或者***linkedlist***。

ziplist使用压缩列表作为底层实现，每个压缩列表节点保存了一个列表元素。

linkedlist使用双向链表作为底层实现，每个双向链表节点都保存了一个字符串对象，而每个字符串对象都保存了一个列表元素。

## 编码转换

当列表对象同时满足一下两个条件时，列表对象使用ziplist编码：

- 列表对象保存的所有字符串元素的长度都小于64字节；
- 列表对象保存的元素数量小于512个；

不满足这两个条件的列表对象需要使用linkedlist编码

这两个条件的上限值可以修改，具体看配置文件<u>***list-max-ziplist-value***</u>和<u>***list-max-ziplist-entries***</u>选项的说明



## 快速链表

链表的附加空间相对太高，prev和next指针就要占16字节，另外每个节点的内存都是单独分配，会加剧内存的碎片化，影响内存管理效率。因此Redis3.2版本开始对列表数据结构进行了改造，使用<u>***quicklist***</u>替代了ziplist和linkedlist。

[快速链表](https://www.cnblogs.com/hunternet/p/12624691.html)



# 哈希对象

哈希对象的编码可以是<u>***ziplist***</u>或者<u>***hashtable***</u>

ziplist编码的哈希对象使用压缩列表作为底层实现，每当有新的键值对要加入到哈希对象时，程序会先将保存了键的压缩列表节点推入到压缩列表表尾，然后再讲保存了值的压缩列表节点推入到压缩列表表尾，因此：

- 保存了同一键值对的两个节点总是紧凑在一起，键的节点在前，值的节点在后
- 先添加到哈希对象中的键值对会被放在压缩列表的表头方向，后添加的会在表尾方向

{% asset_img image-20210609162339053.png Hash : zip list %}

HashTable编码的哈希对象使用字典作为底层实现，哈希对象中的每个键值对都使用一个字典键值对来保存：

- 字典的每个键都是一个字符串对象，对象中保存了键值对的键
- 字典的每个值都是一个字符串对象，对象值保存了键值对的值

{% asset_img image-20210609162841624.png Hash:hash table  %}

## 编码转换

当哈希对象可以同时满足依稀两个条件时，哈希对象使用ziplist编码：

- 哈希对象保存的所有键值对的键和值的字符串长度都小于64字节；
- 哈希对象保存的键值对数量小于512个

不满足这两个条件的哈希对象需要用HashTable编码。

这两个条件的上限值可以修改，具体看配置文件<u>***hash-max-ziplist-value***</u>和<u>***hash-max-ziplist-entries***</u>选项的说明

```shell
redis> HSET book name "mastering C++ in 21 days"
(integer) 1
redis> OBJECT ENCODING book
"ziplist"
# 添加一个键长度为66字节
redis> HSET book long_long_long_long_long_long_long_long_long_long_long_description "content"
(integer) 1
redis> OBJECT ENCODING book
"hashtable"
```

# 集合对象

集合对象的编码可以是<u>***intset***</u>或者<u>***hashtable***</u>

intset编码的集合对象使用整数集合作为底层实现，集合对象包含的所有元素都被保存在整数集合里面。

```shell
redis> SADD numbers 1 3 5
(integer) 3
redis> OBJECT ENCODING numbers
"intset"
```

{% asset_img image-20210609163741336.png Set:int set  %}

hashtable编码的集合对象使用字典作为底层实现，字典的每个键都是一个字符串对象，每个字符串对象包含了一个集合元素，而字典的值全部被设置为NULL。

```shell
redis> SADD Dfruits "apple" "banana" "cherry"
(integer) 3
redis> OBJECT ENCODING Dfruits
"hashtable"
```

{% asset_img image-20210609163851700.png Set:hash table  %}

## 编码转换

当集合对象可以同时满足一下两个条件时，对象使用intset编码：

- 集合对象保存的所有元素都是整数值；
- 集合对象保存的元素数量不超过512个

不满足这两个条件的集合对象需要使用hashtable编码。

第二个条件的上限值是可以修改的，具体看配置文件中<u>***set-max_intset-entries***</u>选项的说明。

```shell
redis> SADD numbers 1 3 5
(integer) 0
redis> OBJECT ENCODING numbers
"intset"
redis> SADD numbers "seven"
(integer) 1
redis> OBJECT ENCODING numbers
"hashtable"
```

# 有序集合对象

有序集合对象的编码可以是<u>***ziplist***</u>或者<u>***skiplist***</u>。

ziplist编码使用压缩列表作为底层实现，每个集合元素使用两个紧挨在一起的压缩列表节点来保存，第一个节点保存元素的成员，而第二个节点则保存元素的分值。

压缩列表内的集合元素按分值从小到大进行排序，分值小的元素放置在靠近表头的方向，而分值大的元素靠近表尾方向。

```shell
redis> ZADD price 8.5 apple 5.0 banana 6.0 cherry
(integer) 3
redis> OBJECT ENCODING price
"ziplist"
```

{% asset_img image-20210609165540223.png Zset:skip list  %}

skiplist编码的有序集合对象使用zset结构作为底层实现，一个zset结构同时包含一个字典和跳表：

```c
typedef struct zset{
	zskiplist *zsl;
	dict *dict;
}zset;
```

zset结构中的zsl跳表按照分值从小到大保存了所有集合元素，每个跳表节点都保存了一个集合元素：跳表节点的object属性保存了元素的成员，而跳表节点的score属性则保存了元素的分值。通过这个跳表，程序可以对有序集合进行范围型操作。

除此之外，zset结构中的dict字典为有序集合创建了一个从成员到分值的映射，字典中每个键值对都保存了一个集合元素：字典的键是元素的值，而字典的值则是元素的分值。通过这个字典，程序可以O(1)查找给定成员的分值。

有序集合每个元素的成员都是一个字符串对象，而每个元素的分值都是一个double类型的浮点数。虽然zset结构同时使用跳表和字典来保存有序集合元素，但是这两种数据结构都会通过指针来共享相同元素的成员和分值，所以同时使用跳表和字典来保存元素不会产生任何重复成员或者分值，也不会因此而浪费额外的内存。

## 为什么有序集合需要同时使用跳表和字典来实现？

只使用字典实现有序集合，可以O(1)查找成员的分值，但字典无序，范围性操作会产生额外的时空间复杂度。

只使用跳表，范围性操作优点可以保留，但是查找会从O(1)上升为O(logN)。

综合使用则保留了两种方法的优点。

## 编码转换

当有序集合对象可以同时满足一下两个条件时，对象使用ziplist编码：

- 有集合保存的元素数量小于128个；

- 有序集合保存的所有元素成员的长度都小于64字节

不满足以上两个条件的有序集合对象将使用skiplist编码。

这两个条件的上限值可以修改，具体看配置文件<u>***zset-max-ziplist-entries***</u>和<u>***zset-max-ziplist-value***</u>选项的说明

# 类型检查与命令多态

Redis中用于操作键的命令可以分为两种。

一种可以对任何类型的键执行，比如DEL，EXPIRE，RENAME，TYPE，OBJECT.

另一种命令只能对特定类型的键执行：

- SET	GET	APPEND	STRLEN等只能对字符串键执行；
- HDEL、HSET、HGET、HLEN等只能对哈希键执行；
- RPUSH、LPOP、LINSERT、LLEN等只能对列表键执行；
- SADD、SPOP、SINTER、SCARD等只能对集合键执行；
- ZADD、ZCARD、ZRANK、ZSCORE等只能对有序集合键执行；

## 类型检查的实现

在执行一个类型特定的命令之前，Redis会先检查输入键的类型是否正确，然后再决定是否执行给定的命令，通过检查redisObject结构的type属性来实现。

## 多态命令的实现

除了根据值对象的类型来判断键是否能够执行指定命令之外，还会根据值对象的编码方式，选择正确的命令实现代码来执行命令。

如果对一个键执行LLEN命令，那么服务器除了要确保执行命令的键是列表键之外，还要根据键的值对象所使用的编码来选择正确的LLEN命令：

- 如果列表对象的编码是ziplist，那么说明列表对象的实现为压缩列表，程序将使用ziplistLen函数来返回列表的长度；
- 如果列表对象的编码是linkedList，那么程序将使用listLength函数来返回双向链表的长度；

# 内存回收

C语言并不具备自动内存回收功能，所以Redis在自己的对象系统中构建了一个引用计数技术实现的内存回收机制，通过这一机制，程序可以通过跟踪对象的引用计数信息，在适当的时候自动释放对象并进行内存回收。

```c
typedef struct redisObject{
  // ...
  // 引用计数
  int refcount;
  //...
}robj;
```

对象的引用计数信息会随着对象的使用状态而不断变化：

- 在创建一个新对象时，引用计数的值会被初始化为1；
- 当对象被一个新程序使用时，它的引用计数的值会+1；
- 当对象不再被一个程序使用时，它的引用计数的值会-1；
- 当对象的引用计数值为0时，对象所占用的内存会被释放。

# 对象共享

除了用于实现引用计数内存回收机制之外，对象的引用计数属性还带有对象共享的作用。

如果键A创建一个包含整数值100的字符串对象作为值对象，这时键B也要创建一个同样保存了整数值100的字符串对象作为值对象，那么服务器有两种做法：

1. 为键B创建一个100的字符串对象；
2. 让键A和键B共享一个字符串对象。

为了节约内存Redis显然是第二中方法：

1. 将数据库键指针指向一个现有的值对象；
2. 将被共享的值对象的引用计数+1。

目前而言Redis会在初始化服务器的时候创建一万个字符串对象，包含了0~9999所有的整数值，当服务器需要用到这些字符串对象时，服务器就会使用这些共享对象，而不是创建新对象。

这些共享字符串对象的数量可以通过修改<u>***redis.h/REDIS_SHARED_INTEGERS***</u>常量来修改。

[最新版如果是共享变量，那么引用计数为设置为INT_MAX](https://blog.csdn.net/qq_34839150/article/details/109844848)

# 对象的空转时长

除了<u>***type、encoding、ptr、refcount***</u>这四个属性之外，还有最有一个为<u>***lru***</u>属性，该属性记录了对象最后一次被命令程序访问的时间：

```c
typedef struct redisObject{
//...
unsigned lru:22;
//...
}robj;
```

OBJECT IDLETIME命令可以打印出给定键的空转时长，这一空转时长就是通过将当前时间减去键的值对象的lru时间计算得出：

```shell
redis> SET msg "hello world"
OK
#等待几秒
redis> OBJECT IDLETIME msg
(integer) 12
# 再等待几秒
redis> OBJECT IDLETIME msg
(integer) 23
redis> GET msg
"hello world"
# 键处于活跃状态，空转时长会重置
redis> OBJECT IDLETIME msg
(integer) 1
```

除了可以被OBJECT IDLETIME命令打印出来之外，键的空转时长还有另一个作用：如果服务器打开了<u>***maxmemory***</u>选项，并且服务器用于回收内存的算法为<u>***volatile-lru***</u>或者是<u>***allkeys-lru***</u>那么当服务器占用内存数超过了<u>***maxmemory***</u>选项所设置的上限值时，那么Redis就回回收掉空转时长较高的键值。

# 重点回顾

- Redis数据库中每个键值对的键和值都是一个对象
- Redis共有字符串、列表、哈希、集合、有序集合五种类型的对象，每种类型的对象都至少有两种以上的编码，不同编码在不同场景上可以优化对象的使用效率
- 在执行某些命令之前，服务器会先检查键类型是否能执行指定的命令
- Redis对象系统带有引用计数实现内存回收机制
- Redis会共享值为0~9999的字符串对象
- 对象会记录自己最后一次被访问的时间，可以用于计算对象的空转时间

