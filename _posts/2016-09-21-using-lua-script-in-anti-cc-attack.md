---
layout: post
title: 用来防止机器行为刷接口的控制脚本
description: "风控防刷好帮手"
category: config
tags: [config]
imagefeature: http://7xwdx7.com1.z0.glb.clouddn.com/5551e1cabd3bb.jpg
comments: true
share: true
---

### 防刷脚本

近来我厂的APP接口经常被机器暴力破解，由于前期风控做的不够处理起来很是费力。万般无奈之际，只好在nginx上部署了一个防刷脚本，对于单一IP地址的高频请求进行拦截。

### 测试demo

本地环境中，为了方便直接使用了openresty进行测试。首先需要在nginx配置文件http属性中增加lua支持：

`lua_package_path "/usr/local/openresty/nginx/lua/?.lua;/usr/local/openresty/lualib/?.lua;;";`

`lua_package_cpath "/usr/local/openresty/lualib/?.so;;";`

同时，配置access_by_lua_file字段、开启lua代码缓存

`lua_code_cache on;`

`access_by_lua_file "/usr/local/openresty/nginx/lua/access_limit.lua";`

最后，将进行访问频率控制的lua脚本放到配置文件中规定的指定路径

~~~lua

-- access_by_lua_file '/opt/ops/lua/access_limit.lua'
local function close_redis(red)
    if not red then
        return
    end
    --释放连接(连接池实现)
    local pool_max_idle_time = 10000 --毫秒
    local pool_size = 100 --连接池大小
    local ok, err = red:set_keepalive(pool_max_idle_time, pool_size)

    if not ok then
        ngx_log(ngx_ERR, "set redis keepalive error : ", err)
    end
end

local redis = require "resty.redis"
local red = redis:new()
red:set_timeout(1000)
local ip = "redis-ip"
local port = redis-port
local ok, err = red:connect(ip,port)
if not ok then
    return close_redis(red)
end

local clientIP = ngx.req.get_headers()["X-Real-IP"]
if clientIP == nil then
   clientIP = ngx.req.get_headers()["x_forwarded_for"]
end
if clientIP == nil then
   clientIP = ngx.var.remote_addr
end

local incrKey = "user:"..clientIP..":freq"
local blockKey = "user:"..clientIP..":block"

local is_block,err = red:get(blockKey) -- check if ip is blocked
if tonumber(is_block) == 1 then
   ngx.exit(ngx.HTTP_FORBIDDEN)
   return close_redis(red)
end

res, err = red:incr(incrKey)

if res == 1 then
   res, err = red:expire(incrKey,1)
end

if res > 200 then
    res, err = red:set(blockKey,1)
    res, err = red:expire(blockKey,600)
end

close_redis(red)

~~~

### 运行效果

下面两张图分别是正常http请求和访问超频被封禁两种状态的结果。可以看到对于机器高频访问，会直接返回403进行处理。

![正常HTTP请求](http://7xwdx7.com1.z0.glb.clouddn.com/normal_access.png)

![被封禁HTTP请求](http://7xwdx7.com1.z0.glb.clouddn.com/limit_access.png)
