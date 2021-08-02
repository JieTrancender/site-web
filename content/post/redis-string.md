---
title: "Redis String"
date: 2021-08-02T22:42:59+08:00
tags: ["redis"]
categories: ["redis"]
toc: false
draft: false
---

一、数据结构基础
1. sds在C字符数组的基础上增加了长度和空间元数据，使基于字符串长度的追加、复制、比较等操作可以直接读取元数据，提升效率。。
2. SDS总共设计了五种类型，分别为sdshdr5、sdshdr8、sdshdr16、sdshdr32和sdshdr64。他们的主要区别是数据结构中字符数组现有len和分配空间长度alloc的不同，sdshdr5没有被使用，后四种len数据类型分别为uint8_t、uint16_t、uint32_t、uint64_t。
~~~
struct __attribute__ ((__packed__)) sdshdr8 {
    uint8_t len; /* used */
    uint8_t alloc; /* excluding the header and null terminator */
    unsigned char flags; /* 3 lsb of type, 5 unused bits */
    char buf[];
};

struct __attribute__ ((__packed__)) sdshdr64 {
    uint64_t len; /* used */
    uint64_t alloc; /* excluding the header and null terminator */
    unsigned char flags; /* 3 lsb of type, 5 unused bits */
    char buf[];
};
~~~
3. SDS设计不同的结构头是为了灵活保存不同大小的字符串，从而有效节省空间。结构头占用内存空间不一样，保存小字符串时结构头占用小可以节约内存。
4. 结构体定义中`__attribute__ ((__packed__))`的作用是告诉编译器不要使用字节对齐方式，而是采用紧凑型额方式分配内存（默认情况编译器会按照8字节对齐的方式分配内存）。
5. 为了避免频繁内存重新分配，当SDS扩容时，如果所需内存小于1M时扩容翻倍，当大于等于1M时扩大1M。
6. SDS保存数据的末尾设置为空字符，并且总会为buf数组分配空间时多分配一个字节来容纳这个空字符，使保存文本数据的SDS可以重用一部分<string.h>库定义的函数。
7. SDS采用惰性空间的释放策略。

二、C字符串与SDS比较
1. C字符串获取长度复杂度为O(N)，SDS直接读取元数据O(1)。
2. C字符串API是不安全的，可能造成缓冲区溢出，SDSAPI是安全的，不会造成缓冲区溢出。
3. C字符串修改字符串长度N次必然需要执行N次内存分配，SDS修改N次最多需要执行N次内存重分配。
4. C字符串以'\0'结尾只能保存文本数据，SDS可以保存文本数据和二进制数据。

三、使用举例
1. 包含字符串值的键值对在底层都是SDS实现的。
2. 客户端状态中的输入缓冲区。
3. 写操作追加到AOF时，会先写入AOF缓冲区。
