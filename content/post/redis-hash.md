---
title: "Redis Hash"
date: 2021-08-02T10:58:22+08:00
tags: ["redis"]
categories: ["redis"]
toc: false
draft: false
---

1. Redis中的dict数据结构采用**链式哈希**方式存储。负载因子大于等于1时触发哈希扩容检测，当存在RDB子进程或者AOF子进程时关闭哈希扩容(避免父进程大量写时复制)，不过即使没开启hash扩容在负载因子大于**dict_force_resize_ratio**时也会执行哈希扩容，扩容大小为当前容量的2倍。
2. 当往Redis中写入新的键值对或者是修改键值对时，Redis都会判断是否需要rehash，函数调用链为：dictAdd/dictReplace/dictAddOrFind -> dictAddRaw -> _dictKeyIndex -> _dictExpandIfNeeded。
3. Redis哈希扩容时使用渐进式rehash，即每次只进行n个bucket的rehash处理，避免大量的键需要从原来的位置拷贝到新位置时主线程无法执行其他请求而阻塞；rehash时当出现10*n个都为空bucket时会结束本次处理，防止阻塞太长时间。
4. Redis中有五处函数调用_dictRehashStep进而调用dictRehash函数来执行rehash，调用链为：ictAddRow/dictGenericDelete/dictFind/dictGetRandomKey/dictGetSomeKeys -> _dictRehashStep -> dictRehash。
5. 全局哈希表除了主动函数调用(增删改查)执行rehash外还有定时rehash，主线程会默认每间隔100ms执行一次迁移操作。单次以100bucket作为单位迁移数据，如果超过1ms就结束当前迁移。
6. 在rehash期间，查询hash表找不到结果时还会在新哈希表查询一次。
