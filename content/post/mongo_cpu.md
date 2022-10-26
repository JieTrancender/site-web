---
title: "优化Mongo降CPU"
date: 2022-10-26T10:02:38+08:00
tags: ["skynet", "游戏后端", "数据库", "mongo"]
categories: ["skynet", "游戏后端", "数据库", "mongo"]
toc: false
draft: false
---

#### 一、背景
前面[腾讯云mongo服务采坑](../wgame_group/)中提到每次水晶活动才开始的时候会吸引大量玩家来抽卡，会导致cpu居高。当然就算没有水晶活动每日凌晨跨天游戏玩法重置等入库外加新的一天较多玩家同时进入也会导致cpu比平时多挺多，如图所示。
![tencent-mongo-cpu-0916](https://storage.keyboard-man.com/site-web/images/tencent-mongo-cpu-0916.png)

#### 二、降CPU
##### 加机器

避免单实例cpu太高的方法也可以通过加机器(增加mongo服务实例)来解决，但是已经在当前实例的数据库迁移到新实例的话需要关服以及处理一些配置，这不是我们想要的。

##### 加索引

如果更新mongo的集合不是以`_id`为筛选条件的话，大多时候都需要增加对应的索引，避免更新时遍历全表。事实上我们基本每个表在上线前都已经加上了索引，查看慢查询补上遗漏的几张表的索引后有少许好点，但问题依然很严重。

##### 减少写请求

我们和数据库交互是通过独立skynet服务来的，在该服务上面临时对每个接口加上分析统计，获取跨天时的100s所有该类型服务的请求就能知道跨天的时候到底做了什么，简单统计代码：
~~~
local isAnalyse, analyseList = false
local CommandEnum = {
	Update      = 1,  -- 更新
	MultiUpdate = 2,  -- 批量更新
	Find        = 3,  -- 查询
	FindOne     = 4,  -- 查询一个
	MultiFind   = 5,  -- 多次查询
	Delete      = 6,  -- 删除
	MultiInsert = 7,  -- 批量插入
	Insert      = 8,  -- 插入
	Drop        = 9,  -- 删除表
	GC          = 10,  -- GC
}

local function analyse(commandType, tableName)
	if not isAnalyse or not commandType or not tableName then
		return
	end

	analyseList[commandType] = analyseList[commandType] or {}
	analyseList[commandType][tableName] = (analyseList[commandType][tableName] or 0) + 1
end

function accept.update(tableName, selector, tbl, upsert, multi)
	local ok, err, ret = db[db_name][tableName]:safe_update(selector, tbl, upsert, multi)
	if not ok then
		logAlarm("mongod accept.update fail", err, util.table_dump_line(ret))
	end

	analyse(CommandEnum.Update, tableName)
end
~~~

经过统计我们看到了水晶活动开启当天最开始100s的玩家主要操作数据(只截取超过1w次的单类型操作，单类型操作不一定是单挑记录操作)：
![0916数据库操作记录](https://storage.keyboard-man.com/site-web/images/mongo-operation-data-0916.png)
总计达到了160w，主要是道具、货币和水晶抽卡。现在目标明确了，就是请求太多每次请求会有多次入库导致的。

当前我们的入库操作都是在具体业务处理完毕后调用入库，首先解决方式就是增加缓存不要每次操作都立即入库。例如单个模块n秒内最多入库一次，这样入库次数就不和玩家操作有关了，只和在线数有关。

这里只统计玩家个人操作，我们还有一些常驻服务(worldd/nonamed/centerd等)跨天也会有业务入库，前面也是跨天实时入库的，优化方案就是跨天当时不立即入库，延迟半小时~一小时后分批定量入库，达到削峰效果。当然避免跨天有关服需求，在关闭服务前如果有要入库的数据需要先入库再退出。

这些优化推出后现在跨天cpu相对稳定了，再也没有之前夸张的跑满cpu导致服务异常。有一定尖刺是因为确实跨天很多重置玩家操作导致cpu占用上升。
![最近7日mongo cpu](https://storage.keyboard-man.com/site-web/images/tencent-mongo-cpu-latest-seven.png)

#### 三、反思
**目前我们延迟入库是在业务层做的这个事，是不是放到底层服务来解决这个事更好呢？**

如果没有删除操作，这个是完全可行而且是非常好的方案，不用在每个业务里面各自处理延迟和批量。当前我们删除的时候可能涉及批量，不太好处理，所以暂时没有做这一步优化。删除改为也是一种更新(增加是否已删除字段)就可以避免这个问题，不过数据库记录数可能就会增加很多很多了。

**除了入库次数变多外忘记入库在这几年中我们组也出现了好几次😓，这个有好的办法解决吗？**

模块操作数据变化后增加标志，退出服务时检测所有模块的所有标志，如果有脏数据标志则报警提示可以一定意义上减少这个问题出现的概率。具体实现后文再说。

或者就是周期性的遍历模块，将所有模块数据在退出服务的时候都入库一遍。这样可以完全杜绝漏入库，但是用户量上来后会导致mongo cpu高很多，因为我们线上部分入库离线都入库一次跑过。

#### 四、备注
1. [game-server](https://github.com/JieTrancender/game-server)项目是工作几年有关线上skynet项目的实战总结，更多内容完善中，欢迎加星。
