---
title: TinySQL 实现笔记
date: 2021-09-29 14:17:39
tags:
  - 个人项目
categories:
  - 编程开发
---

原项目：https://github.com/tidb-incubator/tinysql

我的实现：https://github.com/RinChanNOWWW/tinysql-impl

TInyKV 之外的另一个 PingCAP incubator 的项目，实现一个微型 TiDB，即 TinySQL。正好十月中旬要参加 OB 的比赛，做这个刚好可以了解了解数据库 SQL 层的一些东西，这些在 CMU 15-445 的实验中没怎么深入。虽然 TiDB 的存储层是 KV 的形式，但是总体上的思想应该都大同小异。与之前 TinyKV 一样，这个文章用来做个笔记。

<!-- more -->

## Project 1

将行记录与索引（实际上是 KV）中的 Key 解码，得到行以及索引的相关信息：table id，row id，index id，index col value 等。没什么好说的。就是代码骨架中有很多现成的函数，比如自己从切片中一个一个的取。

## Project 2

WIP
