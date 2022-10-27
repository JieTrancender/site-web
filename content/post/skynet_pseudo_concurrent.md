---
title: "skynet伪并发"
date: 2022-10-27T11:08:40+08:00
tags: ["skynet", "游戏后端", "lua"]
categories: ["skynet", "游戏后端", "lua"]
toc: false
draft: false
---

#### 一、背景
**和 erlang 不同，一个 skynet 服务在某个业务流程被挂起后，即使回应消息尚未收到，它还是可以处理其他的消息的。所以同一个 skynet 服务可以同时拥有多条业务执行线。所以，你尽可以让同一个 skynet 服务处理很多消息，它们会看起来并行，和真正分拆到不同的服务中处理的区别是，这些处理流程永远不会真正的并行，它们只是在轮流工作。一段业务会一直运行到下一个 IO 阻塞点，然后切换到下一段逻辑。你可以利用这一点，让多条业务线在处理时共享同一组数据，这些数据在同一个 lua 虚拟机下时，读写起来都比通过消息交换要廉价的多。**

**互有利弊的是，一旦你当前业务处理线挂起，等回应到来继续运行时，内部状态很可能被同期其它业务处理逻辑所改变，请务必小心。在 skynet api 文档中，已经注明了哪些 API 可能导致阻塞。两次阻塞 API 调用之间，运行过程是原子的，利用这个特性，会比传统多线程程序更容易编写。**

#### 二、配置加载
当前项目我们配置是使用**skynet sharetable**模块来管理的，skynet现在也推荐使用sharetable来管理。进程启动时通过`sharetbale.loadfile`或者`sharetable.loadtable`加载所有配置到内存，其他各个服务根据配置文件名按需调用`sharetable.query`获取配置。代码示例：
~~~
local function test_sharetable()
    local skynet     = require "skynet"
    local sharetable = require "skynet.sharetable"

    local configList = {}
    local function queryConfig(t, configName)
        local config = sharetable.query(configName)
        t[configName] = config
        return config
    end
    
    setmetatable(configList, {__index = queryConfig})
    
    local totalCount = 0
    local CMD = {}
    function CMD.doSomething()
        local count = totalCount
        local testConfig = configList["test"]
        totalCount = count + 1
    end

    function CMD.getCount()
        return totalCount
    end
    
    skynet.start(function()
        skynet.dispatch("lua", function(session, source, command, ...)
            local f = assert(CMD[command], string.format("source:%s, command:%s", skynet.address(source), command))
            if session > 0 then
                skynet.retpack(f(...))
            else
                f(...)
            end
        end)
    end)
end

skynet.start(function()
    sharetable.loadtable("test", {content = "test content"})
    local testservice = service.new("test_sharetable", test_sharetable)
    for i = 1, 100 do
        skynet.send(testservice, "lua", "doSomething")
    end
    skynet.sleep(100)
    print("test_sharetable count is ", skynet.call(testservice, "lua", "getCount"))
end
~~~

最终的输出结果可能不是预期中的**1**而是**1**，这就是因为`sharetable.query`函数中调用了skynet.call挂起协程让出执行导致的伪并发。
~~~
/game-server # ./skynet/skynet examples/config.testpseudoconcurrent.lua 
[:00000002] LAUNCH snlua bootstrap
[:00000003] LAUNCH snlua launcher
[:00000004] LAUNCH snlua cdummy
[:00000005] LAUNCH harbor 0 4
[:00000006] LAUNCH snlua datacenterd
[:00000007] LAUNCH snlua service_mgr
[:00000008] LAUNCH snlua testpseudoconcurrent
[:00000009] LAUNCH snlua service_provider
[:0000000a] LAUNCH snlua service_cell sharetable
[:0000000b] LAUNCH snlua service_cell test_sharetable
test_sharetable count is        1
~~~

#### 三、避免伪并发
##### 阻塞API前置
获取配置这一步阻塞导致了伪并发发生，所以在所有接口中都前置配置加载就可以直接避免。即将示例中的`local testConfig = configList["test"]`获取配置这一步放到函数最前面。大多时候类似的阻塞API都可以通过前置解决，但有时候上下文强关联这就解决不了了。

##### 配置预加载
在每个接口中都前置配置处理会增加开发人员的心智负担，尤其是不了解这一原因的更容易漏处理。如果能在所有接口调用前即服务启动后第一时间就进行所有配置预加载至少配置就不会导致伪并发了，这是我们想要的。

提交[feat: add sharetable queryall[#1536]](https://github.com/cloudwu/skynet/pull/1536)和[perf: add sharetable queryall default behavior(all share data)[#1537]](https://github.com/cloudwu/skynet/pull/1537)两个pr后就可以做这件事了，服务启动后首先同步调用**init**接口获取所有配置指针并缓存到配置管理器，业务任何时候获取配置都不会挂起协程了。
~~~
function CMD.init()
    for filename, ptr in pairs(sharetable.queryall()) do
        configList[filename] = ptr
    end
end

local testservice2 = service.new("test_sharetable2", test_sharetable2)
-- 提前加载配置
skynet.call(testservice2, "lua", "init")
for i = 1, 100 do
    skynet.send(testservice2, "lua", "doSomething")
end
skynet.sleep(100)
print("test_sharetable2 count is ", skynet.call(testservice2, "lua", "getCount"))
~~~

##### skynet.queue模块
有时候调用阻塞API被挂起后，服务响应其他消息可能造成时序问题，这就不是前置和预加载能够解决问题的了。skynet提供的`skynet.queue`模块能简单解决这类伪并发问题。queue函数每次调用都可以得到一个新的临界区。临界区可以保护一段代码不被同时运行。

在本篇示例中需要临界区保护的就是doSomething函数，提前获取临界区，将函数体都放入临界区就会获得预期结果。
~~~
local doSomethingQueue = queue()
function CMD.doSomething()
    doSomethingQueue(function()
        local count = totalCount
        local testConfig = configList["test"]
        totalCount = count + 1
    end)
end
~~~

#### 备注
1. 本篇涉及的伪并发测试参见[game-server](https://github.com/JieTrancender/game-server)项目的[feat: add skynet pseudo concurrent[#9]](https://github.com/JieTrancender/game-server/pull/9)提交。
