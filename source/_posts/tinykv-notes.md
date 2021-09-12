---
title: TinyKV 实现笔记
date: 2021-09-12 16:19:51
tags:
  - 个人项目
categories:
  - 编程开发
---

原项目：https://github.com/tidb-incubator/tinykv

我的实现：https://github.com/RinChanNOWWW/tinykv-impl

PingCAP incubator 的项目，实现一个微型 TiKV。在此文章中记录一下开发中的值得注意的点。

<!-- more -->

## Project 1

基于 badger 实现一个单机存储引擎，没什么好说的。

## Project 2

实现 Raft 以及基于 Raft 的 KV server。之前在 MIT 6.824 课程学习的时候，也使用 Go 作为编程语言完成了其实现 Raft 算法的课程实验，不过当时因为是第一次接触，所以代码写的很杂乱，鲁棒性也很低。这次重新实现，借助于非常详实的 test case，写起来也更加顺畅，代码架构也比之前写的清晰了许多，虽然到后面也开始变得杂乱无章了起来。

### Part A

在本部分主要是实现了 Raft 算法本身以及暴露给外部的接口层 `rawnode`。

#### 关于 Raft 日志的保存

在实现过程中，值得注意的是，TinyKV 的 Raft 层中 log 存在于两个部分，一个是内存中的 `RaftLog.entries`，一个是实际落盘的 `RaftLog.storage`。一开始我被这两个东西搞得晕头转向，其实一开始就该明确，前者是算法运行过程中实际活跃的 log，而后者是被applied 的 stable 数据。所以 Raft 算法运行过程中所有的日志都应该从 `RaftLog.entries` 中存取。只不过由于 `RaftLog.entries` 可能会被清理（被压缩，或者重启 server 等），所以它的第一条日志所对应的 index 可能不是实际的第一个 index，所以需要结合 `RaftLog.storage.FirstIndex` 来换算。

#### 关于日志同步

日志同步是通过 Leader 调用 `sendAppend` 来向 Follower 批量发送未同步的日志。这里需要注意的是更新 commit 值的时机，除了收到 `AppendResponse` 外，在成为 Leader 之后也需要立即 check 一遍，因为集群中可能只有 Leader 一个 Node 存在。

#### 关于状态信息的恢复

还有一个值得注意的地方就是，在调用 `newRaft` 的时候需要先从 `config.storage` 中恢复一些落盘的状态信息：Term, Vote, Commit 以及整个集群的节点配置信息。如果忽略这一点会导致一些 test case 失败。

### Part B

这一部分的实现一开始完全没有头绪，所以很大程度上参考了 https://github.com/platoneko/tinykv 的实现。这一部分主要实现的是如何将 Raft 协议同步的日志 apply 到 upper application 中，也就是如何将 Raft 日志中记录的 KV 操作应用到实际的存储引擎中。

主要的步骤就是：

1. 集群接收到 KV 操作，并交给 Leader 节点。
2. Leader 节点将 KV 操作转换为 Raft 日志，通过 propose 操作开启 Raft 同步算法。
3. 每个节点都通过 `rawnode` 中定义的 `Ready` 接口拿到此时自己节点的状态与日志信息，并将其应用到实际的存储中（`SaveReady`)。最后更新 stable 与 applied 信息（`Advance`）。
4. 应用完日志之后还需向客户端返回响应，实现中体现为给 proposal 的回调字段添加返回信息，并通知 channel。

这里面一开始困扰我的就是不知道整个集群是如何应用 Raft 日志的，其实十分简单，就是每个节点都会持续运行一个 worker 来检查 Raft 层的状态，如果能够进行处理那就进行处理，不行就等待下一次循环。在这之中需要注意的是对 snapshot 的特殊处理。

### Part C

这部分实现了 Raft 算法中对 snapshot 处理。对于进行压缩处理的相关逻辑与代码已经被实现不用我自己实现，需要我来实现的就只有添加发送 snapshot 与接受 snapshot 的逻辑，以及完善之前代码中需要 check snapshot

的部分。

这里需要解决的疑惑是，**snapshot 都是无法访问的**。TinyKV 会启动协程定期对日志进行压缩处理，**一旦日志被压缩了，那它便不会被 Raft 算法涉及**。snapshot 和日志一样，在没有被 upper application 处理之前，都一直处于 pending 状态。所以当 Leader 读取日志时发现读不到的时候，它就知道有日志被压缩成 snapshot 了，它就需要向 Follower 同步这个 snapshot。想明白了这些实现起来就容易了。

## [WIP] Project 3

这部分会实现 TiKV 的多 Region 多 Raft Group 机制。

### Part A

此部分依靠 Raft extend 论文的 Section 6 以及 https://web.stanford.edu/~ouster/cgi-bin/papers/OngaroPhD.pdf 的 Section 3.10 实现了 Raft 层面的 Leader Transfer 以及节点配置的变化（也就是 Raft 集群的成员变更）。其中对于成员变更的细节主要参考了原论文以及 https://zhuanlan.zhihu.com/p/375059508 这篇文章的解读。后者是前者的翻译 + 更加平实的语言的分析。

本部分实现起来较为简单，主要是复习了一遍 Raft 的成员变更策略，之前面试问到都没想起来……之前读论文的时候这后面的部分读的太水了……