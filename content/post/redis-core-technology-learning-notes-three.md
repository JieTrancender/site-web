---
title: "Redis核心技术与实战 学习笔记 三"
date: 2021-01-26T19:55:37+08:00
tags: ["redis"]
categories: ["redis"]
toc: false
draft: true
---

String类型不是什么场合都适用的“万金油”，在保存键值对本身占用的内存空间不大时，String类型的元数据(RedisObject结构、SDS结构、DictEntry结构的内存开销)开销占据主导，会造成大量资源浪费。

<!--more-->

在redis6.0版本中五大数据类型都会封装成`RedisObject`结构，不同数据类型的主要区别就是**type**和**encoding**字段，同一种数据类型有不同的编码(比如下文的String)。见代码文件**src/server.h**。

~~~c

/* The actual Redis Object */
#define OBJ_STRING 0    /* String object. */
#define OBJ_LIST 1      /* List object. */
#define OBJ_SET 2       /* Set object. */
#define OBJ_ZSET 3      /* Sorted set object. */
#define OBJ_HASH 4      /* Hash object. */

/* Objects encoding. Some kind of objects like Strings and Hashes can be
 * internally represented in multiple ways. The 'encoding' field of the object
 * is set to one of this fields for this object. */
#define OBJ_ENCODING_RAW 0     /* Raw representation */
#define OBJ_ENCODING_INT 1     /* Encoded as integer */
#define OBJ_ENCODING_HT 2      /* Encoded as hash table */
#define OBJ_ENCODING_ZIPMAP 3  /* Encoded as zipmap */
#define OBJ_ENCODING_LINKEDLIST 4 /* No longer used: old list encoding. */
#define OBJ_ENCODING_ZIPLIST 5 /* Encoded as ziplist */
#define OBJ_ENCODING_INTSET 6  /* Encoded as intset */
#define OBJ_ENCODING_SKIPLIST 7  /* Encoded as skiplist */
#define OBJ_ENCODING_EMBSTR 8  /* Embedded sds string encoding */
#define OBJ_ENCODING_QUICKLIST 9 /* Encoded as linked list of ziplists */
#define OBJ_ENCODING_STREAM 10 /* Encoded as a radix tree of listpacks */

typedef struct redisObject {
    unsigned type:4;
    unsigned encoding:4;
    unsigned lru:LRU_BITS; /* LRU time (relative to global lru_clock) or
                            * LFU data (least significant 8 bits frequency
                            * and most significant 16 bits access time). */
    int refcount;
    void *ptr;
} robj;
~~~

Redis中字符串都使用的是自定义简单动态字符串（Simple Dynamic Strings，SDS），一共有五种类型（**sdshdr5**、**sdshdr8**、**sdshdr16**、**sdshdr32**、**sdshdr64**），不同类型的sdshdr头内存占用不同，根据字符串的长度选择不同的结构以节约内存。



- sdshdr5

字符串对象有三种编码（可使用`OBJECT ENCODING keyName`语句查看具体的编码类型）：

- int：负责保存整数。当所存整数能够被sdshdr8
- embstr：负责保存短字符串，即长度小于44字节的字符串。
- raw：负责保存长字符串。

当对int编码的key执行append(因为只有string支持该操作)时，编码将会转成raw并且不能再转回。

~~~redis
127.0.0.1:6380[15]> set intKey 1
OK
127.0.0.1:6380[15]> OBJECT ENCODING intKey
"int"
127.0.0.1:6380[15]> append intKey 100
(integer) 4
127.0.0.1:6380[15]> OBJECT ENCODING intKey
"raw"
~~~

当对embstr编码的key执行修改操作时，编码将会转成raw并且不能再转回。

~~~redis
127.0.0.1:6380[15]> set embstrKey embstrValue
OK
127.0.0.1:6380[15]> OBJECT ENCODING embstrKey
"embstr"
127.0.0.1:6380[15]> append embstrKey appendValue
(integer) 22
127.0.0.1:6380[15]> OBJECT ENCODING embstrKey
"raw"
~~~



当以10位数表示key值也是10位数的时候每个键值对消耗56字节。为什么是56字节呢

因为该10位数能够用long表示，则字符串使用**int编码**;4字节type，4字节encoding，24字节LRU，10字节数字，1字节'\n'

~~~c
typedef struct redisObject {
    unsigned type:4;
    unsigned encoding:4;
    unsigned lru:LRU_BITS; /* LRU time (relative to global lru_clock) or
                            * LFU data (least significant 8 bits frequency
                            * and most significant 16 bits access time). */
    int refcount;
    void *ptr;
} robj;
~~~

