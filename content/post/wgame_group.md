---
title: "腾讯云mongo服务采坑"
date: 2022-10-25T16:54:27+08:00
tags: ["skynet", "游戏后端"]
categories: ["skynet", "游戏后端"]
toc: false
draft: false
---

#### 一、背景
我们项目存储游戏数据采用的mongo服务，流水等记录数据存储采用的mysql服务。之前是部署到每台业务机器上，开在线上机器的游戏服一般直接连接本机mongo服务。最新的项目由于业务增长运营方要求将我们使用的mongo服务和mysql服务迁移到腾讯的云服务上面去，保障”**数据库的安全可用**“。

众所周知国内游戏主要靠活动来获得流水的提升，我们的水晶活动(类似抽卡)可以使玩家获得明显属性增长，所以每次水晶活动开始时会吸引大量玩家持续抽卡。某次水晶活动凌晨10分左右由于抽卡的大量持续请求导致负载升高服务不可用，持续十五分钟左右服务才恢复正常。国庆期间又会开启该活动，为此我们做了很多努力来力保不再出现问题，现在要说的就是其中的一部分。

#### 二、负载不够加机器
前面[动态转发](../dynamic-redirect/)提到过我们是通过水平扩展wgame节点来降低单个wgame节点负载和提升玩家在线数量的，所以最简单直接解决负载不足的方式就是增加足够的机器。

国庆活动很重要，且不能在一个地方摔倒两次，在做了很多优化(后面再聊)之后我们还是决定增加机器，同时在国庆前还提前增加了mongo服务实例和提升了mysql服务示例配置来扩展数据库能力。由于我们wgame节点是均匀承载单个服玩家的，每个wgame节点都有某个服的玩家，再加上游戏服服务(wcenter节点centerd服务)也会与数据库连接，所以与mongo服务连接数等于`(wgame节点数 + 1) x 游戏服数`。腾讯云显示实例最大连接数为16000(见下图)。

![腾讯云mongo最大连接数](https://storage.keyboard-man.com/site-web/images/tencent_mongo_maxconnection.png)

当时我们有两百多个服，最终起到50个wgame节点，根据计算差不多连接使用率是80%左右，计划国庆后再处理使用率过多问题。一切准备就绪，不出意外的话果然出了意外。

#### 三、异常报警
国庆前日下午，静待放假的我们收到钉钉报警写入mongo报错提示`not master`。

查看服务器状态负载正常，mongo的cpu和内存也在比较良好的状态。暂时无玩家反馈异常问题。反馈给腾讯云经过一段时间的排查后给出的结果是连接数超了，**mongo的读写最大连接数是12000**，没错前面写的16000是不对的，是包含从节点的读连接的，巨坑。他们内核限制了不能也不敢在这个特殊时期调整。

#### 四、临时解决
腾讯云没法解决，最终只能到我们这里。我们打算减少15个wgame节点，平均线上机器wgame节点数调整wgame节点权重为0，踢断这些节点上的在线玩家(客户端自动重连会重新分配到其他wgame节点上)，关闭这些进程后连接数稳定到了预期的7000左右。

玩家服务退出的时候我们有检测重要数据是否一致(比如背包英雄数据是否和数据库英雄数据一致)，如果不一致则用内存覆盖，如果退出的时候数据库状态还未正常则不退出服务。所以最终没有产生比较严重的后果，实际还是有一定影响的，不过竟然没有一个玩家反馈。

#### 五、最终解决
实际上优化和加机器前不是因为承载不了单个服的玩家数据，但是加wgame节点导致连接数大量增加了(流失后单个服的玩家可能很少了，可能很多服一个wgame节点就足以应对)。最终解决方案就是对wgame节点分组，15个wgame节点为一个组，单个服分配一个wgame节点组即可，现在连接使用率稳定20%。就算后期持续新服也不会导致暴增，而且不久的将来肯定会合服，连接数会进一步减少。

虽然mysql的连接数限制没有这么小，但是分组一样的减少了mysql连接，让mysql连接所占的内存减少了。

#### 六、备注
1. [game-server](https://github.com/JieTrancender/game-server)项目是工作几年有关线上skynet项目的实战总结。
