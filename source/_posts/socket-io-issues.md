---
title: Socket.io 踩坑记录
date: 2020-12-31 18:40:20
tags:
	- Node.js
	- 学习
	- Golang
categories:
	- 编程学习
---

最近由于项目需要接触了 socket.io 这一套工具，它本身是一个 Node.js 的包，使用 long polling 或者 websocket 的方式（在我的项目中我是用的是 websocket）提供持续的网络连接服务。由于简单好用（虽然我在知乎上看到有人说这玩意儿就是给小白玩的，2333），很多开发者为其开发了不同语言的 SDK，比如 Java、C++、Go 等，然后我项目的前端用的 Node.js，后端用 Golang。由于 socket.io 本身的设计以及 Go 版本开源库的残缺，踩了很多坑，我决定在这篇博客中好好总结一下。

## 跨域请求的问题

由于 go-socket.io 是基于 http 服务器的，在建立 websocket 连接的时候会先发送一个 get 请求，由于请求是跨域的，所以理所应当会被服务器给拦下来，然后返回一个 403，所以首先得处理跨域的问题。

处理这个问题有很多种方式。

第一种方式是直接拿到请求后把 Header 里的 Origin 字段干掉，然后再交给接下来的 http 处理逻辑处理。第二种方式是添加一个中间件来设置 Header 中的允许跨域字段，这两种方式在这个 issue 中都有实现方法： https://github.com/googollee/go-socket.io/issues/242

第三种方式是社区开发者们新提供的方式，就是在启动 socket.io server 的 option 种指定允许的域名，这算是最合适的一种方法了吧。

```go
allowOrigin := func(r *http.Request) bool {
    return true
}
server, err := socketio.NewServer(&engineio.Options{
    Transports: []transport.Transport{
        &polling.Transport{
	    Client: &http.Client{
	        Timeout: time.Minute,
	    },
	    CheckOrigin: allowOrigin,
	    },
	&websocket.Transport{
	    CheckOrigin: allowOrigin,
	},
    },
})
```

更多信息可以查看这个 issue：https://github.com/googollee/go-socket.io/issues/372

## go-socket.io 与 socket.io 的兼容性问题

在前端，我一开始使用的是 3.0 版本的 socket.io。结果发现建立连接的握手请求前后端对应不上，后端触发两次 connect 事件，之后也无法进行正常的双工通信，后来发现原来 go 这个库还不支持 2.0 版本及以上的 socket.io。这其中应该是通信协议上不一样。

这个项目其实很早就已经搁置了，owner 在几年前就撒手不管了，这几年一直是靠开源社区的开发者们来维护，所以这个项目的进度迟迟得不到更新。目前这个库只支持 socket.io@2.0 之前的版本，不支持 2.0。

如果实在要与 socket.io 2.0 进行对接，可以选用其他的语言进行开发，或者使用其他的几个 go 开源库，虽然这几个 star 不多，我也没用过。

- https://github.com/pschlump/socketio

- https://github.com/ambelovsky/gosf-socketio

详情可见这个 issue：https://github.com/googollee/go-socket.io/issues/188

之后我也打算看看这个项目，看看能不能贡献点什么。

## 分布式场景下的负载均衡问题

## 一些其他问题
