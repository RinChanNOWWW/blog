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

最近由于项目需要接触了 socket.io 这一套工具，它本身是一个 Node.js 的包，使用 long polling 或者 websocket 的方式提供持续的网络连接服务。由于简单好用（虽然我在知乎上看到有人说这玩意儿就是给小白玩的，2333），很多开发者为其开发了不同语言的 SDK，比如 Java、C++、Go 等，然后我项目的前端用的 Node.js，后端用 Golang。由于 socket.io 本身的设计以及 Go 版本开源库的残缺，踩了很多坑，我决定在这篇博客中好好总结一下。

## 跨域请求的问题



未完待续。。。

## Reference

https://github.com/googollee/go-socket.io/issues/242

https://blog.csdn.net/yonggeit/article/details/102586637