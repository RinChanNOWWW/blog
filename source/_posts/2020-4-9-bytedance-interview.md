---
title: 4月9日字节跳动视频架构后台开发面经
date: 2020-04-10 08:06:33
tags:
  - 字节跳动面试
categories:
  - 面经
---

4月9日字节跳动视频架构后台开发面经

<!-- more -->

## 一面

- 项目经历

  简单问了一下我大一下的的一个大作业车间调度算法的实现。

- 基础知识

  - 进程和线程的区别。

  - CPU 调度的最小单位是什么。

  - HTTP 与 HTTPS 的区别。

  - 知道哪些数据库引擎。

    ​	我就只知道 InnoDB。。。

  - 对称加密与非对称加密的区别。

  - 数据库中 inner join 和 left join 的区别。

  - 使用 select 语句分页查询的效率如何，比如查询前 100 个记录和后 100 个记录效率比较。

  - 数据库事务的特点并解释。

    ​	即 ACID。（当时有点慌没有答好。。）

  - 平衡二叉树和红黑树的定义。谁的查找效率更好？谁的插入效率更高？

    ​	平衡二叉树更好查找，因为它更矮更平均。红黑树的维护没有平衡二叉树那么复杂，所以插入效率更高。

  - ARP 协议的过程。

  - 不同架构的机器（如 x86 和 arm）编译的可执行文件可以在对方机器上运行吗？为什么？要如何才能在不同架构的机器上执行？可以在一种架构的机器上编译另一种架构机器的可执行文件吗？如何做到？

  - Linux 会用吗？

  - Linux 如何查看文件权限。

  - 操作系统进程调度的几种算法。

- 算法

  - 单链表反转。（手写）

  - Top K 问题。

    具体问法是 100w 个无序数据中选 100 个最大的数据。一开始我还在往位图算法上想。。。结果经面试官一提醒，直接建立一个容量为 100 的大根堆就行了。。

## 二面

- 基础知识

  - C++ `static` 关键字的作用。

  - C++ 中重载和重写分别是什么？

  - C++ 的动态绑定是怎样的？

    ​	一开始没理解面试官的意思，然后突然想起他应该指的是虚函数的动态绑定。

  - 对 Python 多线程的理解。

  - 对协程的看法。

  - 知道哪些内存泄漏的例子。

  - TCP 与 UDP 的区别。

  - socket 编程中使用 TCP 连接的过程。

  - 知道 RPC 吗？

    ​	这个只知道叫 远程过程调用，具体不清楚。。

  - 小端模式与大端模式。

- 算法

  - 最大子段和。（手写）

  - 统计一个数二进制中 1 的个数。（手写）

    有很多种方法，可参考 https://www.cnblogs.com/graphics/archive/2010/06/21/1752421.html

- 提问环节

  问了下他们的业务。

## 三面

​		三面只有十分钟，没有问什么技术问题。问了我对实习的看法，以后的规划。然后问我本科期间最后成就感的事情，然后给他看了下我的博客。基本上就是确认我会过去进行一段较长时间的实习。

## 总结

​		首先要感谢[耿总](https://ipotato.me)的内推，不过耿总五月份就要去 aws 走上他的架构师之路了，这里要由衷地说一句**<font color=red>太强了</font>**。

​		头条的效率是真的高，一天下午就搞完三面。自己来说感觉还是比较充分的，不过还是有点小小的紧张，有很多问题都是张口就来，没有好好的思考，不过面试官都挺好的，我好几次都是经过面试官提醒才突然改答案，这么一想还有点尴尬。。。后来也渐渐进入状态。

​		我面试的小组目前做的是云游戏的内容（没想到头条也在搞云游戏），不过目前还只是起步阶段，后台开发主要是做云游戏的调度等工作，然和后其他视频平台比如抖音做对接。只能说是很巧了，我的的确确对云游戏很感兴趣，这里要再次感谢耿总的内推。二面的时候我还问了下面试官腾讯的云游戏（指堡垒之夜）没有办法进行语音，不知道这是为什么，是有技术上的问题吗，然后面试官：你来就知道了（笑）。

