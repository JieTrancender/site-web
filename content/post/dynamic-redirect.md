---
title: "后端服务动态转发"
date: 2022-10-25T16:01:34+08:00
draft: true
tags: ["openresty", "redis", "skynet"]
categories: ["openresty", "redis", "skynet"]
toc: false
draft: false
---

#### 一、背景
当前我们游戏架构通过水平扩展wgame节点(和区服无关只处理玩家连接和玩家服务数据)来承载在线玩家的。通过一直扩展减轻单个wgame节点负载和承载更多在线玩家。

在wgame节点前面增加一层nginx以实现wss(客户端和服务端通过websocket连接，但是微信等小程序要求wss)。以前单个渠道不用太多wgame节点就能满足线上需求，随着业务增长和游戏类型决定同时在线玩家数较以前出现较大增长，每次增加线上新机器或者机器上面新增wgame节点(wgame监听客户端连接端口)就需要重新生成nginx配置并reload一次，导致nginx配置越来越难维护。后期统一入口即在nginx前面加入SLB之后每次还需要在SLB增加新的端口监听和转发。


#### 二、动态转发
引入openresty，通过读取存在于redis中的后端地址数据来实现动态转发。SLB只需要监听443端口然后配置转发到每台线上机器的openresty监听端口。新增wgame节点后只需要更新redis中的数据，仅添加线上新机器的时候需要在SLB配置一下转发即可。大大的减少了nginx配置的复杂程度，更方便管理地址信息。

增加独立动态转发nginx子配置`dynamic_redirect.conf`，增加后端节点变量`set $backend_node ''`，在**access**阶段执行转发函数赋值后端节点变量，然后通过proxy_pass就可以代理转发到具体的后端服务了，配置示例：
~~~
server {
    listen 8080;

    location = /ws {
        set $backend_node '';
        access_by_lua_block {
            lifesix.dynamic_redirect()
        }
        proxy_redirect off;
        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_set_header X-Real_IP $remote_addr;
        proxy_set_header X-Forwarded-For $remote_addr:$remote_port;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_pass http://$backend_node;
        access_log logs/${backend_node}_access.log main;
    }
}
~~~

连接wgame节点的时候增加请求参数节点名称**nodeName**、节点序列**threadId**和渠道信息**platform**；拼接成redis哈希key`projectName_$platform_node_$nodeName`，然后根据节点序列threadId查找该节点的地址信息，赋值该节点信息到`$backend_node`变量，示例代码：

~~~
local function is_empty(s)
    return s == nil or s == '' or s == 'null' or s == 'NULL' or s == ngx.null
end

function _M.dynamic_redirect()
    local nodeName = ngx.var.arg_nodeName
    local threadId = ngx.var.arg_threadId
    local platform = ngx.var.arg_platform

    if not nodeName or not threadId or not platform then
        ngx.log(ngx.ERR, "arguments are wrong, nodeName or threadId or platform is not exist")
        return ngx.exit(500)
    end
    
    local ngx_ctx = ngx.ctx
    local redis_cli = redis:new()
    redis_cli:set_timeouts(1000, 1000, 1000)

    local ok, err = redis_cli:connect("127.0.0.1", 6379)
    if not ok then
        ngx.log(ngx.ERR, "failed to connect redis, err: ", err)
        return ngx.exit(500)
    end

    -- ignore this if not auth
    -- local res, err = redis_cli:auth("123456")
    -- if not res then
    --     ngx.log(ngx.ERR, "failed to auth redis, err: ", err)
    --     return ngx.exit(500)
    -- end
    
    local backend_node = redis_cli:hget(table.concat({"game-server", platform, "node", nodeName}, "_"), threadId)
    if is_empty(backend_node) then
        ngx.log(ngx.ERR, "failed to get node info, key is ", table.concat({"game-server", platform, "node", nodeName}, "_"))
        return ngx.exit(500)
    end

    ngx.var.backend_node = backend_node
end
~~~

#### 三、测试
增加新的openresty server作为后端转发服务。
~~~
server {
    listen 80;
    location / {
        default_type text/html;
        content_by_lua_block {
            ngx.say("<p>hello, world!</p>")
        }
    }
}
~~~

将改地址信息放入redis中，当前项目名称为game-server，渠道名称为dev。
~~~
127.0.0.1:6379> hset game-server_dev_node_wgamed 1 127.0.0.1:80
(integer) 1
~~~

请求转发服务，成功转发到后端服务。
~~~
[root@005912f18ad1 lifesix-openresty]# curl -ig 'localhost:8080/ws?nodeName=wgamed&threadId=1&platform=dev'
HTTP/1.1 200 OK
Server: openresty/1.21.4.1
Date: Tue, 25 Oct 2022 07:54:38 GMT
Content-Type: text/html; charset=utf-8
Transfer-Encoding: chunked
Connection: keep-alive

<p>hello, world!</p>
~~~

#### 四、备注
1. [game-server](https://github.com/JieTrancender/game-server)项目是工作几年有关线上skynet项目的实战总结。
2. 本篇涉及的代码参见[#1](https://github.com/JieTrancender/lifesix-openresty/pull/1)
