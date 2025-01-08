---
title: "skynet火焰图"
date: 2022-10-26T18:05:37+08:00
tags: ["skynet", "游戏后端", "lua"]
categories: ["skynet", "游戏后端", "lua"]
toc: false
draft: false
---

#### 一、背景
前面[腾讯云mongo服务采坑](../wgame_group/)和[优化Mongo降CPU](../mongo_cpu/)两篇文章从两个角度记录了我们解决水晶活动异常所用到的方式。但这些都是外部的，真正是否机器不够用、数据库拖慢服务还是其它问题还是得从代码层面入手。从数据库入库记录和请求日志已经很清晰就是水晶抽卡导致负载上升服务异常的，所以直接压测接口找到代码层面问题是最首先应该做的事，而火焰图就是最好的工具。
<!--more-->

#### 二、skynet火焰图
开源项目[swt](https://github.com/lsg2020/swt)提供了火焰图基础库和ui界面。

开源项目[luaprofile](https://github.com/lsg2020/luaprofile) profile c and lua function and support coroutine yield。

启动**swt master**节点处理ui请求，启动**swt agent**节点用来收集服务信息。再启动持续运行消耗cpu的cost_cpu服务供收集资源消耗生成火焰图。代码如下：
~~~
local util    = require "util"
local skynet  = require "skynet"
local service = require "skynet.service"
local swt     = require "swt"

local function cost_cpu()
    local skynet = require "skynet"

    local function execRandom()
        for i = 1, 100000000 do
            randomV = math.random(1, i)
        end
    end
    
    local function execTableConcat()
        local arr = {}
        for i = 1, 10000000 do
            table.insert(arr, i)
        end
        table.concat(arr, " ")
    end
    
    skynet.start(function()
        skynet.dispatch("lua", function()
            local randomV
            for j = 1, 100 do
                execRandom()
                execTableConcat()
                skynet.yield()
                print("exec random", j)
            end
        end)
    end)
end

skynet.start(function()
    swt.start_master("0.0.0.0:11001")
    swt.start_agent("app", "node1", "127.0.0.1:11002")

    local costservice = service.new("cost_cpu", cost_cpu)
    skynet.send(costservice, "lua", "test")
end)
~~~

浏览器访问`http://127.0.0.1:11001/admin#/dashboard`输入节点地址`127.0.0.1:11002`提交。切换左侧profiler栏目，点击开启开始测试，完毕后查看`snlua service_cell cost_cpu`服务的CPU和MEM火焰图。
![火焰图](https://storage.keyboard-man.com/site-web/images/skynet-profile-dashboard.png)

根据CPU火焰图能够看到消耗CPU主要是`execRandom`函数中的`math.random`函数调用和`execTableConcat`函数中的`table.concat`函数调用。
![CPU火焰图](https://storage.keyboard-man.com/site-web/images/skynet-profile-cpu.png)

根据MEM火焰图能够看到消耗MEM主要是`execTableConat`函数中的`table.concat`函数调用，占比达到了71%。
![MEM火焰图](https://storage.keyboard-man.com/site-web/images/skynet-profile-mem.png)

#### 三、总结
从示例的测试火焰图代码中不能看出明显的需要优化的点(这里测试只是简单的循环让服务一直跑)，但直接找到消耗资源的具体位置，在实际中这就很重要了，有了目标点就可以定制具体的优化办法。

具体到我们的这次代码优化就是优化接口涉及、批量操作、配置预处理减少不必要的循环等。

#### 四、备注
1. 本篇涉及的火焰图测试参见[game-server](https://github.com/JieTrancender/game-server)项目的[#8](https://github.com/JieTrancender/game-server/pull/8)提交。