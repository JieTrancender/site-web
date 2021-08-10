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
该对象结构包含四个元数据和一个指针，type为redisObject的数据类型，包括String、List、Hash等；encoding为redisObject的编码类型，是内部实现各种数据类型所用的数据结构；lru为redisObject的LRU时间；ptr为指向值的指针。
type、encoding、lru三个变量后面的冒号和数值表示该元数据占用的比特数，这种变量后使用冒号和数值的定义方法为C语言中的位域定义方法，可以用来有效地节省内存开销。

##### 嵌入式字符串
一般时候redisObject的ptr指针会指向值的数据结构，但是当创建字符串对象的时候如果字符串长度小于等于`OBJ_ENCODING_EMBSTR_SIZE_LIMIT`时该ptr直接用来表示SDS字符串。
~~~
// object.c
#define OBJ_ENCODING_EMBSTR_SIZE_LIMIT 44
robj *createStringObject(const char *ptr, size_t len) {
    if (len <= OBJ_ENCODING_EMBSTR_SIZE_LIMIT)
        return createEmbeddedStringObject(ptr,len);
    else
        return createRawStringObject(ptr,len);
}
~~~
`createRawStringObject`函数会分配一块连续的内存空间，这块内存空间的大小等于redisObject结构体的大小+SDS结构头sdshdr8的大小+字符串大小+1字节的字符串结尾'\0'。
分配内存后创建SDS结构的指针sh，并把sh指向这块连续空间中SDS结构头所在的位置，接着将redisObject的指针ptr指向SDS结构中的字符数组。

~~~
/* Create a string object with encoding OBJ_ENCODING_EMBSTR, that is
 * an object where the sds string is actually an unmodifiable string
 * allocated in the same chunk as the object itself. */
robj *createEmbeddedStringObject(const char *ptr, size_t len) {
    robj *o = zmalloc(sizeof(robj)+sizeof(struct sdshdr8)+len+1);
    struct sdshdr8 *sh = (void*)(o+1);

    o->type = OBJ_STRING;
    o->encoding = OBJ_ENCODING_EMBSTR;
    o->ptr = sh+1;
    o->refcount = 1;
    if (server.maxmemory_policy & MAXMEMORY_FLAG_LFU) {
        o->lru = (LFUGetTimeInMinutes()<<8) | LFU_INIT_VAL;
    } else {
        o->lru = LRU_CLOCK();
    }

    sh->len = len;
    sh->alloc = len;
    sh->flags = SDS_TYPE_8;
    if (ptr == SDS_NOINIT)
        sh->buf[len] = '\0';
    else if (ptr) {
        memcpy(sh->buf,ptr,len);
        sh->buf[len] = '\0';
    } else {
        memset(sh->buf,0,len+1);
    }
    return o;
}
~~~

### 二、内存数据按一定规则淘汰



### 三、指导
1. 对指针进行加1操作表示将内存地址移动一段距离，而移动的距离等于当前结构的单位大小；`struct sdshdr8 *sh = (void*)(o+1);`表示将指针移动一个redisObject结构体大小的距离；`o->ptr = sh+1;`表示将指针移动一个sdshdr8结构体大小的距离。
