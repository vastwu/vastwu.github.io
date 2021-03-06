---
layout: post
title: node学习笔记（创建持久连接和通信）
category: work
---


开年第一篇，上班用了一段时间mac，回家就给window装上了powershell，sublime打开了vim快捷键，一下感觉高达上了有没有。

最近工作中有了些机会和需要，搞了下node，之前的node总局限于本地文件操作，开发build脚本什么之类的，
这次尝试了下网络相关的玩意，普通的Server很简单，记录下怎么弄个最简单的web-socket

<!--more-->

### Server部分(server.js)

首先建立个最基本的Server对象

~~~javascript
  var server = http.createServer().listen('80');
~~~

随后是监听 upgrade 事件，在回调函数中会获取到socket，设置下保持链接，超时时间什么的，随后监听data事件，就完事啦，非常easy
往回发的那个socket是告诉客户端通信成功，如果要最简单的话，其实这个也可以无所谓啦

~~~javascript
  server.on('upgrade', function(request, socket, head){
    console.log('remotePort:' + socket.remotePort + ' connecting');
    socket.setTimeout(0);
    socket.setNoDelay(true);
    socket.setKeepAlive(true, 0);
    socket.on('data', function(data){
      console.log('receive a socket data：' + data);
      socket.write('fine thank you');
    });
    socket.write('HTTP/1.1 101 Web Socket Protocol Handshake\r\n' +
               'Upgrade: WebSocket\r\n' +
               'Connection: Upgrade\r\n' +
               '\r\n');
  };
~~~

<!--more-->

### Client部分（client.js）

使用http模块创建一个请求对象

~~~javascript
  var client = http.request({
      'host':'127.0.0.1',
      'port':'80',
      'headers': {
          'Connection': 'Upgrade',
          'Upgrade': 'websocket'
      }
  }, function(res){
      console.log('request start');
  });
  client.end();
~~~  

随后监听 upgrade 事件，捕获连接成功

~~~javascript
  client.on('upgrade', function(res, socket, upgradeHead) {
      console.log('got upgraded!');

      setInterval(function(){
          client.write('how are you?');
      }, 2000);
  
      socket.on('data', function(data){
          console.log('request data:' + data);
      })
  });
~~~

在连接成功后开启一个setInterval，每隔2秒像服务器发送一个 how are you 的字符串，并监听服务器socket的data事件，在控制台输出

然后就可以启动了，先运行server.js， 启动完成后，再运行client.js，
看两个控制台，就会一来一往的发送how are you 和 find thank you啦

这一小段可以说是个最最基础的长连接通信模式，还算不上websocket，真正根浏览器结合的websocket是需要配合一些标准中的head头信息之类的验证下
过滤掉一些杂乱请求

学习过程参考 [node-websocket-server](https://github.com/miksago/node-websocket-server);

这个再往后详细做的话，就需要server端根据`socket.remotePort`来保存多个socket连接，来识别不同的来源，保存下socket引用，就可以开发出类似广播的机制了。
基本的链接建立打通后，就可以弄出很多可玩的东西啦。




