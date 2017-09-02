#### 1. 介绍

> A pure stream http push technology for your Nginx setup.

> Comet made easy and really scalable.

> Supports EventSource, WebSocket, Long Polling, and Forever Iframe. 

[nginx-push-stream-module](https://github.com/wandenberg/nginx-push-stream-module)是nginx的一个模块，用它可以轻易实现websocket服务器。

以前我们实现websocket的服务器，不外乎两种方式，第一种是嵌入到web进程中，成为web进程的一部分，只是以路由的方式提供服务，还有一种是单独的进程，比如用puma来启动包含actioncable的rails项目。

这两种方式多多少少跟web进程都有些关系，嵌入型的就不用多说，就算是单独的进程这种方式，也是用了另一种服务器去启动。

现在我们来考虑另外一种方式，就是用完全独立于web进程的服务器，比如，之前我们的web是用ruby写的，可能会用puma或unicorn来启动，现在我们可以用c++启动websocket服务器，而web进程是通过http请求等方式来连接websocket服务器。

当然，这一篇文章，我们是用nginx来启动一个websocket的服务器，nginx本身没有这样的功能，需要其中的一个模块，就是本章介绍的`nginx-push-stream-module`。

现在我们先来跑一下流程，再来讲述一下它的原理以及为什么能够这样做。

#### 2. 使用

首先得先安装一下这个模块。

##### 2.1 安装

安装很简单，跟之前的模块安装一模一样的步骤，具体可以查看这篇文章[nginx之编译第三方模块(六)](http://www.rails365.net/articles/nginx-zhi-bian-yi-di-san-fang-mo-kuai-liu)。

现在来列出一下大概的流程。

``` bash
$ git clone https://github.com/wandenberg/nginx-push-stream-module.git
# 进入到nginx源码目录，--add-module后面接nginx-push-stream-module的源码目录
$ ./configure --add-module=../nginx-push-stream-module
# 编译
$ make
# 安装
$ sudo make install
# 结束老的nginx进程
$ sudo nginx -s quit
# 开启新的nginx进程
$ sudo nginx
```

接着我们来使用这个模块。

##### 2.2 配置

在配置文件`nginx.conf`中添加下面这样的内容：

``` conf
http {
  
    push_stream_shared_memory_size 32M;

    server {
        location /channels-stats {
            # activate channels statistics mode for this location
            push_stream_channels_statistics;

            # query string based channel id
            push_stream_channels_path               $arg_id;
        }

        location /pub {
            # activate publisher (admin) mode for this location
            push_stream_publisher admin;

            # query string based channel id
            push_stream_channels_path               $arg_id;
        }

        location ~ /ws/(.*) {
            # activate websocket mode for this location
            push_stream_subscriber websocket;

            # positional channel path
            push_stream_channels_path                   $1;
            # message template
            push_stream_message_template                "{\"id\":~id~,\"channel\":\"~channel~\",\"text\":\"~text~\"}";

            push_stream_websocket_allow_publish         on;

            # ping frequency
            push_stream_ping_message_interval           10s;
        }
    }
}
```

##### 2.3 原理

其中`push_stream_shared_memory_size`是添加在`http`下，其他的都在`server`下。

`push_stream_shared_memory_size`我就不详说了，应该设置的是一个内存的容量，我对此细节并不了解，按照默认的就好了。

我们来讲述一下`server`之下的三个`location`。

* /channels-stats
* /pub
* ~ /ws/(.*)

第一个是关于websocket统计相关的东西，这个稍后再讲。

另外两个是关于发布订阅的。

其中客户端连接到服务器使用的是`~ /ws/(.*)`这个`location`，而服务器端向客户端推送消息是使用`/pub`这个`location`。

至于客户端与服务器端是如何交互的，我们可以回顾一下。

客户端，比如浏览器，发送`new WebSocket`给websocket服务器，表示要建立websocket请求，这个过程表示的是订阅，一直在等待服务器发送消息过来，一旦有消息过来，就会更改一些状态，比如DOM更新等。这个过程，不止只有一个客户端连接上服务器，可能有好多客户端同时连接。假如现在有了业务变化，服务器需要向所有的客户端推送消息，这个过程就是发布，广播消息。通过什么广播消息呢，这个机制可以自己实现，也可以用redis的pub/sub功能，比如，一旦客户端连接上服务器，就会订阅redis的一个channel，而发布的时候，就是往这个channel里推送消息，这样，所有的客户端都能接收到消息。

`nginx-push-stream-module`不需要redis的pub/sub，它是自己实现的。

##### 2.4 测试

现在我们开始来测试一下这个应用。

还记得之前提到的`/channels-stats`这个`location`吗？它是统计信息用的。

我们先来看下它的结果。

``` bash
$ curl -s -v 'http://localhost/channels-stats'
```

输出的内容主要是下面的json信息：

```
{"hostname": "macintoshdemacbook-air.local", "time": "2016-05-07T12:02:34", "channels": 0, "wildcard_channels": 0, "published_messages": 0, "stored_messages": 0, "messages_in_trash": 0, "channels_in_trash": 0, "subscribers": 0, "uptime": 19755, "by_worker": [
{"pid": "21117", "subscribers": 0, "uptime": 19755}
]}
```

上面的信息包括主机名，时间，通道的个数，消息的个数等，我们先不管。

现在我们用浏览器建立一个连接到websocket服务器，也就是要请求`~ /ws/(.*)`这个`location`。

``` javascript
ws = new WebSocket("ws://localhost/ws/ch1");
ws.onmessage = function(evt){console.log(evt.data);};
```

很简单，使用`new WebSocket`建立一个websocket请求，地址为`ws://localhost/ws/ch1`。

`ch1`是通道的名称，` push_stream_channels_path                   $1;`这一行配置指的就是它。

`onmessage`表示接收服务器端的消息，一旦有消息过来，就用`console.log`输出来。

我们一直在关注着浏览器的输出。

现在我们给客户端推送一条消息，自然是使用`/pub`这个`location`。

``` bash
$ curl http://localhost/pub\?id\=ch1 -d "Some Text"  
{"channel": "ch1", "published_messages": 1, "stored_messages": 0, "subscribers": 1}
```

使用的是curl这个命令，`ch1`表示的是通道名，它可以以参数的形式来指定，这样就会灵活很多，不同类型的连接可以用不同的通道名。

果然浏览器输出了信息了：

```
{"id":1,"channel":"ch1","text":"Some Text"}
```

`id`是消息的编号，默认从1开始，这个数字会自增，`channel`表示通道名，`text`是服务器端发送的信息。

输出的内容，跟`push_stream_message_template                "{\"id\":~id~,\"channel\":\"~channel~\",\"text\":\"~text~\"}";`这里定义的模版有关。

果然是推送了什么内容就是输出了什么内容。

现在我们来看看统计内容的输出：

``` bash
$ curl -s -v 'http://localhost/channels-stats'
{"hostname": "macintoshdemacbook-air.local", "time": "2016-05-07T12:24:13", "channels": 1, "wildcard_channels": 0, "published_messages": 1, "stored_messages": 0, "messages_in_trash": 0, "channels_in_trash": 0, "subscribers": 1, "uptime": 21054, "by_worker": [
{"pid": "21117", "subscribers": 1, "uptime": 21054}
]}
```

可以看到`"channels": 1`表示有一个通道，之前是没有的，`"published_messages": 1`表示发布的消息也多了一条了。

我们可以发起多个`new WebSocket`或开多个浏览器进行测试，那样可以观看到更多的效果。

之前通过curl工具，向`/pub`这个`location`发送了http请求，这个就间接向客户端发送数据，只是表现方式跟之前的不太一样。

##### 2.5 ruby

在实际的编程中，我们可以会用ruby应用结合nginx的`nginx-push-stream-module`这个模块来做应用，总不至用curl这个工具，这个工具主要用于测试，我们现在试一下用ruby来代替curl。

开启一个ruby的命令终端`irb`。

``` ruby
require 'net/http'

uri = URI("http://localhost/pub\?id\=ch1")
http = Net::HTTP.new(uri.host, uri.port)

req = Net::HTTP::Post.new(uri.to_s)
req.body = 'Some Text'

http.request(req)
```

你会发现，效果是一样的。

`nginx-push-stream-module`是个不错的工具，如果灵活运用它，肯定有意想不到的好处。

完结。
