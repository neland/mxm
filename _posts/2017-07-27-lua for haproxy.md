---
layout: post
title: Lua for Haproxy
comments: true
category: 开源
tags: Haproxy lua
---

### 安装
* 先装Lua5.3+ 
```
wget http://www.lua.org/ftp/lua-5.3.3.tar.gz
tar -xzf lua-5.3.3.tar.gz
cd lua-5.3.3 
make linux
make install
```
* 再装Haproxy
```
// 获取1.6以后的版本
wget http://www.haproxy.org/download/1.6/src/haproxy-1.6.0.tar.gz
tar -xzf haproxy-1.6.0.tar.gz
cd haproxy-1.6.0
// 主要是添加USER_LUA=1 linux2628可以通过 uname -a看到，linux2628=2.6.28
make TARGET=linux2632 USE_PCRE=1 USE_OPENSSL=1 USE_ZLIB=1 USE_CRYPT_H=1 USE_LIBCRYPT=1 USE_LUA=1
make install
```

### 使用

#### 流程
    Haproxy使用Lua，分为两个步骤
    1、全局加载Lua文件，注册Lua函数
    2、配置文件中使用注册的函数

#### 加载
* 在配置文件中加载Lua文件
* haproxy启动的时候，会执行main.lua这个文件
```
# haproxy.conf
global
   lua-load main.lua
```    

#### 注册函数
* 函数注册支持匿名函数
* 这些函数在Haproxy使用的时候，会根据场景，将上下文传递给注册的函数
   - register_service会传递 tcp或http的对象 applet
   - register_action会传递本次请求的对象txn
   - 在函数中，可以使用这些对象，具体对象属性和功能，可以参看官方[API](http://www.arpalert.org/src/haproxy-lua-api/1.8dev/index.html#http-class)
   - register_init和register_task，是会在Haproxy加载完成后就执行

```
-- main.lua
core.register_action("myaction", { "tcp-req", "http-req" }, function(txn)
   txn:Info("Hello world by my action")
end)

core.register_fetches("choose",function(txn)
    local path = txn.f:path()
    if path == "/9999" then
        return "service9999"
    end
    return "service8888"
end)

function myservice(applet)
   local response = "Hello World by myservice!"
   applet:set_status(200)
   applet:add_header("content-length", string.len(response))
   applet:add_header("content-type", "text/plain")
   applet:start_response()
   applet:send(response)
end

core.register_service("myservice", "http", myservice )
```

#### 函数调用
* 在配置文件中，以lua为前缀，后面跟注册的函数名，如lua.myservice
   - 替换Service
       + http-request content use-service lua.myservice
       + tcp-request  content use-service lua.myservice
   - 替换行为
       + http-request lua.myaction
       + tcp-request  lua.myaction
   - 替换字符
       + %[lua.myfetch] 
   - 写了一些[例子](https://github.com/neland/haproxy-for-lua)

```
global
   lua-load main.lua

defaults
  mode http
  timeout client 10000
  timeout server 10000
  timeout connect 1000

frontend example
   bind 127.0.0.1:10000
   # 在每次有http请求的时候，执行myaction
   http-request  lua.myaction
   # 在每次要选择目标服务器的时候，用choose函数返回的服务器名
   use_backend %[lua.choose]

backend service8888
   server srr8888 127.0.0.1:8888
backend service9999
   server srv9999 127.0.0.1:9999
```

### 其它
* Lua文件只会加载一次，不会动态加载，不过可以考虑在Lua函数中动态加载
* 注册函数有优先级，比如service会覆盖backend相关的
