---
layout: post
title: Node.js之WebSocket
comments: true
category: 设计
tags: Node.js WebSocket 
---

### 安装
* 正常安装流程看[Github](https://github.com/websockets/ws)
* 会遇到的问题
    - Error: Cannot find module 'ws'  没有添加环境变量xxx\node_modules\ws
    - use of const in strict mode 把node升级到最新版本

<!--more-->

### 服务器端

下面代码存为 server.js 运行 node service.js

```
// 监听8181
var WebSocketServer = require('ws').Server,
wss = new WebSocketServer({ port: 8181 });

// connection为新的链接，ws为新的链接的句柄，后续操作可以通过ws
wss.on('connection', function (ws) {
    console.log('client connected');

    // 消息处理 message，可以通过ws进行消息回复
    ws.on('message', function (message) {
        console.log(message);
        ws.send("server received message");
    });

    // 处理断链消息
    ws.on('close', function close() {
  console.log('disconnected');
});
});
```

### 客户端

client.html，用支持html5的浏览器打开

```
<html xmlns="http://neland.github.io">
<head>
    <script>
    var ws = new WebSocket("ws://localhost:8181");
    
    ws.onopen = function (e) {
        var element=document.getElementById("header");
        element.innerHTML="websocket connected";
    }
    ws.onmessage = function(e)
    {
        var element=document.getElementById("header");
        element.innerHTML = e.data;
    }
    ws.onclose = function(e)
    {
        var element=document.getElementById("header");
        element.innerHTML= "websocket closed";
    }
    ws.onerror = function(e)
    {
        var element=document.getElementById("header");
        element.innerHTML= "websocket error";
    }
    function sendMessage() {
      var element=document.getElementById("message");
        ws.send( element.value);
    }
    </script>
</head>

<body >
    <h1 id="header"></h1>
    <div class="vertical-center">
        <div class="container">
            <p>&nbsp;</p>
            <form role="form" id="chat_form" onsubmit="sendMessage(); return false;">
                <div class="form-group">
                    <input class="form-control" type="text" name="message" id="message"
                           placeholder="Type text to echo in here" value="" />
                </div>
                <button type="button" id="send" class="btn btn-primary"
                        onclick="sendMessage();">
                    Send!
                </button>
            </form>
        </div>
    </div>
</body>
</html>
```

### 一些后续
* websocket可以理解为一个双工的通信载体
* 配合protobuff、xml、json等，可以解决大部分点对点的通信场景
* 断链重连需要框架支持
* node处理websocket是单线程，多客户端为串行处理
