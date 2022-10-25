---
title: "etcd lualib for skynet"
date: 2022-10-24T18:30:00+08:00
tags: ["etcd", "skynet"]
categories: ["etcd", "skynet"]
toc: false
draft: false
---

#### 一、背景
以前项目配置都是放在skynet的启动配置里面，当线上项目进程节点变多后需要加新配置内容的时候需要全部重新生成启动配置，不统一的配置内容(mongo数据库地址、mysql数据库地址、进程节点ip和进程监听端口等)会导致配置管理变得非常麻烦，而且不能比较方便实时更改配置。为了解决这个问题我们引入[etcd](https://github.com/etcd-io/etcd)来管理所有业务配置，现在skynet启动配置只需要传入etcd所需要必要参数即可。现在的配置内容大体示例为：
~~~
root = "./"

--DEBUG = true
project_name = "wgame"
thread = 16
logservice = "mylogger"
harbor = 0
start = "main"
bootstrap = "snlua bootstrap"
luaservice = root.."service/?.lua;"..root.."skynet/service/?.lua;"..root..project_name.."/?.lua"
lualoader = root.."skynet/lualib/loader.lua"
cpath = root.."skynet/cservice/?.so;"..root.."cservice/?.so"
lua_cpath = root.."luaclib/?.so;"..root.."skynet/luaclib/?.so"
lua_path = root.."lualib/?.lua;"..root..project_name.."/?.lua;"..root.."skynet/lualib/?.lua;"..root.."skynet/lualib/skynet/?.lua"
snax = root.."service/?.lua;"..root.."skynet/service/?.lua"

logpath = "."

--参数
thread_id=1
logger="./log/"..project_name.."_"..thread_id
etcd_base_path="/config/game-server/dev/"
etcd_hosts="http://127.0.0.1:2379,http://127.0.0.1:2381,http://127.0.0.1:2383"
etcd_user="root"
etcd_pass="123456"
etcd_protocol="v3"
~~~

#### 二、基础etcd库

[skynet-etcd](https://github.com/JieTrancender/skynet-etcd)是一个lua库用于skynet处理与etcd的数据交互，深度参考了**lua-resty-etcd**(https://github.com/api7/lua-resty-etcd)项目。目前采用http与etcd交互，所以v3版本etcd需要通过配置`enable-grpc-gateway: true`开启grpc网关。

#### 三、 构建etcdd服务
skynet中为与业务分离，与etcd连接最好做成单独服务，下面是一个简易只实现**get**、**set**、**readdir**接口的独立etcdd服务。在init方法中通过传入的**etcd_hosts**、**etcd_user**、**etcd_passwd**、**etcd_protocol**参数初始化连接。
~~~
local util   = require "util"
local skynet = require "skynet"
local etcd   = require "etcd.etcd"

local etcd_cli
local _M = {}

local function genEtcdHosts(etcd_hosts)
    return util.split(etcd_hosts, ",")
end

function _M.init(etcd_hosts, etcd_user, etcd_passwd, etcd_protocol)
    local opt = {
        http_host = genEtcdHosts(etcd_hosts),
        user = etcd_user,
        password = etcd_passwd,
        protocol = etcd_protocol,
        serializer = "json",  -- 默认使用json格式配置
    }
    
    local err
    etcd_cli, err = etcd.new(opt)
    if not etcd_cli then
        return err
    end

    return nil
end

function _M.get(...)
    return etcd_cli:get(...)
end

function _M.set(key, value, ttl)
    return etcd_cli:set(key, value, ttl)
end

function _M.version()
    return etcd_cli:version()
end

function _M.readdir(key, recursive)
    return etcd_cli:readdir(key, recursive)
end

skynet.start(function()
    skynet.dispatch("lua", function(session, source, command, ...)
        local f = assert(_M[command], string.format("source:%s, command:%s", skynet.address(source), command))
        if session > 0 then
            skynet.retpack(f(...))
        else
            f(...)
        end
    end)
end)
~~~

#### 四、测试独立服务
通过`skynet.uniqueservice("etcdd")`创建进程唯一etcdd服务，初始化后设置两个缓存节点的配置，再通过节点配置路径获取节点配置测试是否设置成功。
~~~
local util           = require "util"
local skynet         = require "skynet"
local etcd_hosts     = skynet.getenv("etcd_hosts")
local etcd_user      = skynet.getenv("etcd_user")
local etcd_passwd    = skynet.getenv("etcd_passwd")
local etcd_protocol  = skynet.getenv("etcd_protocol")
local etcd_base_path = skynet.getenv("etcd_base_path")

skynet.start(function()
    local etcdd = skynet.uniqueservice("etcdd")
    local err = skynet.call(etcdd, "lua", "init", etcd_hosts, etcd_user, etcd_passwd, etcd_protocol)
    if err ~= nil then
        skynet.error("failed to init etcdd:", err)
        return
    end

    local wcachedConf1 = {ips = "127.0.0.1:11001,127.0.0.1:11002,127.0.0.1:11003", nodeName = "wcached", threadId = 1, serviceName = "wcached"}
    local wcachedConf2 = {ips = "127.0.0.1:11101,127.0.0.1:11102,127.0.0.1:11103", nodeName = "wcached", threadId = 2, serviceName = "wcached"}
    local res, err = skynet.call(etcdd, "lua", "set", etcd_base_path.."node/wcached/1", wcachedConf1)
    local res, err = skynet.call(etcdd, "lua", "set", etcd_base_path.."node/wcached/2", wcachedConf2)
    local res, err = skynet.call(etcdd, "lua", "get", etcd_base_path.."node/wcached/2")
    print("res = ", util.table_dump_line(res), err)
    
    skynet.error("test etcd successd.")
end)
~~~

#### 五、测试结果
~~~
[:00000002] LAUNCH snlua bootstrap
[:00000003] LAUNCH snlua launcher
[:00000004] LAUNCH snlua cdummy
[:00000005] LAUNCH harbor 0 4
[:00000006] LAUNCH snlua datacenterd
[:00000007] LAUNCH snlua service_mgr
[:00000008] LAUNCH snlua testetcd
[:00000009] LAUNCH snlua etcdd
config is       {["threadId"] = 2.0,["nodeName"] = "wcached",["ips"] = "127.0.0.1:11101,127.0.0.1:11102,127.0.0.1:11103",["serviceName"] = "wcached",}
[:00000008] test etcd successd.
[:00000002] KILL self
~~~

#### 六、备注
1. [game-server](https://github.com/JieTrancender/game-server)项目是工作几年有关线上skynet项目的实战总结。
2. 本篇涉及的代码参见[#3](https://github.com/JieTrancender/game-server/pull/3)。
