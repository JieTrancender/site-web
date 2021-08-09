---
title: "Redis Memory Optimization"
date: 2021-08-09T12:33:11+08:00
tags: ["redis"]
categories: ["redis"]
toc: false
draft: true
---

Redis主要是通过**数据结构的优化设计与使用**和**内存数据按一定规则淘汰**这两大方面的技术来提升内存使用效率的。其中前者主要包括**内存友好的数据结构设计**和**内存友好的数据使用方式**，这也是本文分析的重点。

### 一、内存友好的数据结构设计
Redis中**简单动态字符串(SDS)**、**压缩列表(ziplist)**和**整数集合(intset)**这三种数据结构针对内存使用效率做了设计优化。

#### 1. SDS的内存友好设计
SDS设计了不同类型的结构头，包括sdshdr8、sdshdr8、sdshdr16、sdshdr32、sdshdr64，不同大小的字符串使用特定的结构头能有效的节约内存使用，详情请看[redis string](./redis-string.md)。
Redis还是用了**嵌入式字符串**的设计方法，即将字符串直接保存在Redis的基本数据对象结构中，避免给字符串分配额外的空间。

这里先分析一下Redis用来保存键值对中值的基本数据对象**redisObject**。
~~~
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


### 二、内存数据按一定规则淘汰
