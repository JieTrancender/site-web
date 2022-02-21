---
title: "一条SQL更新语句是如何执行的 MySQL学习笔记二"
date: 2022-02-18T12:38:59+08:00
tags: ["MySQL"]
categories: ["MySQL"]
toc: false
draft: false
---

更新语句同样会走一遍查询语句的那一套流程，即客户端、连接器、分析器、优化器、执行器、存储引擎等。表更新的时候会将该表有关的查询缓存情况，所以建议一般不使用查询缓存。

与查询你流程不一样的是更新流程还设计两个重要的日志模块，redo log（重做日志）和binlog（归档日志）。

#### redo log

在MySQL里，如果每一次的更新操作都需要写进磁盘，磁盘找到对应的那条记录更新，整个过程IO成本很高、查询成本都很高。为了解决这个问题设计了redo log。

MySQL里经常说到的WAL技术（Write-Ahread Logging），**关键点就是先写日志，再写磁盘**。具体来说当有一条记录需要更新的时候，InnoDB引擎会先把记录写到redo log里面并更新内存。InnoDB引擎在适当的时候再将这个操作记录更新到磁盘里面，往往在系统比较空闲的时候做。

InnoDB的redo log是固定大小循环写的，当无空余位置的时候就不能执行新的更新，需要先将一部分操作记录更新到磁盘。

InnoDb 的redo log保证即使数据库发生异常重启，之前提交的记录都不会丢失。

#### binlog

redo log和binlog不同点有：

1. redo log是InnoDB引擎特有的；binlog是MySQL的Server层实现的，所有引擎都可以使用。
2. redo log是物理日志，记录的是“在某个数据页上做了什么修改”；binlog是逻辑日志，记录的是这个语句的原始逻辑，比如“给ID=2这一行的c字段加1”。
3. redo log是循环写的，空间固定会用完；binlog是可以追加写入的。“追加写”是指binlog文件写到一定大小后会切换下一个，并不会覆盖以前的日志。

~~~mysql
create table T(ID int primary key, c int);
update T set c = c + 1 where ID = 2;
~~~



执行器和InnoDB引擎在执行简单update语句时的流程为：

{{< mermaid loadMermaidJS="true" showSequenceNumbers="true" actorMargin=130 >}}

graph TD;

{{< /mermaid >}}

1. 执行器先找引擎取ID为2这一行。ID是主键，引擎直接用树搜索找到这一行。如果这一行所在的数据页本来就在内存中，直接返回给执行器；否则需要先从磁盘读入内存再返回。
2. 执行器将引擎给的数据中这个值加1，再调用引擎接口写入这行新数据。
3. 引擎将这行新数据更新到内存，同时将这个更新操作记录到redo log里面，此时redo log处于prepare状态。然后告知执行器执行完成了，随时可以提交事务。
4. 执行器生成这个操作的binlog，并把binlog写入磁盘。
5. 执行器调用引擎的提交事务接口引擎把刚刚写入的redo log改为提交（commit）状态，更新完成。


