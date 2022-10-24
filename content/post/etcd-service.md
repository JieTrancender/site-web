---
title: "etcd lualib for skynet"
date: 2022-10-24T18:30:00+08:00
tags: ["etcd", "skynet"]
categories: ["etcd", "skynet"]
toc: false
draft: false
---

[skynet-etcd](https://github.com/JieTrancender/skynet-etcd)是一个lua库用于skynet处理与etcd的数据交互，深度参考了**lua-resty-etcd**(https://github.com/api7/lua-resty-etcd)项目。目前采用http与etcd交互，所以v3版本etcd需要通过配置`enable-grpc-gateway: true`开启grpc网关。
