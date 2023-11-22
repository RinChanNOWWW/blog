---
title: 校招生在找工作时如何装成一个经验丰富的开发者
date: 2023-11-22 18:24:57
tags:
    - 校招
    - 数据库
categories:
    - 生活
---


应届生在找正式工作时，除了熟练掌握算法题与八股之外，实习、项目经历也是一大考察点。如果对口的经历比较丰富，也能给面试官留下好的印象甚至免掉不少算法与八股环节。本文笔者从自身经历出发，侧重于实习与项目经历部分，总结了自己如何装成一个经验比较丰富的开发者。

<!-- more -->

## 0. 先找一个方向

人的精力总归是有限的，想在所有领域都装成经验丰富的样子是不太可能的，所以最好先找一个相对较细的方向开始，后续有精力也许可以举一反三进行拓展。拿笔者较为熟悉的数据库领域举例，按照产品形态，业界存在事分布式数据库、云原生数据库、在线数仓、离线数仓、数据湖、查询引擎、流数据库、时序数据库等。可以选择一个特定方向开始刷经验，下文将以笔者较为熟悉的 OLAP 领域为例来展开说明。

## 1. 实习

刷经验的最快方式自然是实习，不过没有实习没关系，后面还有其他方法。

实习能让你迅速了解一个工业级产品的整体结构，也能帮助你了解现在的产品想要解决当前领域的哪些问题，这能让你在了解这个领域时更加有系统性与方向性。因此，对于面试而言，除了自己的工作内容，这一块必需准备的问题自然是，系统架构是怎么样的，在这个领域中对于竞品而言有什么优势。

对于实习，大厂和小厂都有自身的优势。大厂自然是背书更好，你在介绍你的产品时会更加有说服力，但缺点便是短时间内可能无法接触到太核心的区域（但是可以通过文档学习，纸上谈兵）。小公司的优势便是可能可以快速上手核心内容，笔者便有幸主导了多个模块的优化与开发工作，其中不乏完整独立功能的设计与开发。其中有几项工作每一次面试都会介绍，可谓是个人招牌了。

实习也能辅助接下来要说的其他方法，具体来说就是执行下述方法时更有目的性与驱动力。

## 2. 参与或学习明星项目

每一个领域一般都会有几个行业标杆与明星项目，他们有的开源有的闭源，但他们一定都会有相关介绍的文章或视频。在实际工作中，大家其实都是互相借鉴，所以学习或参与一些有名的项目，也能增强自己的吹逼能力。当然这个过程也不可能涵盖整个项目，一般是针对某一个当前关心的问题来进行学习，比如我想看看这个数据库的执行是 pull based 还是 push based；他们排序用的是堆还是直接归并；他们聚合所用的哈希表是怎么设计的等等。

对于开源项目，除了直接看代码与注释，还有几个容易被不常关注开源社区的同学忽略的地方，那就是 Issue/PR/Discussion 区（以GitHub为例），每一个 PR 中开发者大多都会在 summary 中写上当前 PR 的思路和概要设计等等，在 PR/Issue/Discussion 的留言串中也可以看到开发者们的讨论过程，了解他们关心的问题和思考过程，这些内容也可以吸收为吹逼的内容。另外就是定期翻阅项目的 commit log 也能大致了解项目目前的发展方向。除了代码托管平台上的讨论，很多社区也提供了邮件列表/Discord/Slack/微信群等交流方式。

对于许多项目（开源与闭源），很多产品都有介绍其产品细节的博客文章，公开演讲等。这些信息大多都能从其官网，社交账号（X、Youtube、公众号等）发布。

在介绍自己的项目或实习经历时，可以通过“引经据典”来展示自己对此类场景较有经验。

实战经历：

- “对于排序算子的实现，ClickHouse 采用的的是基于堆的多路归并，DuckDB 采用了并行的归并排序算法。ClickHouse 曾考虑过引入败者树来优化排序流程，但是……通过综合自身架构，我们的方案是……”
- “对于这个问题，我在 apache/arrow-rs 社区提出过一个方案，但是社区其他开发者认为……”
- “这个功能我们参考了 Snowflake 的…与 firebolt 的…，他的原理是……”

对于 OLAP 领域，在此推荐几个可以深入了解的项目（报个菜名）：

- 数据库，数仓：Databend, Snowflake, ClickHouse, Apache Doris, DuckDB, StarRocks, Hyper, BigQuery。
- 查询执行引擎相关：Apache Arrow-Datafusion, Presto, Velox。
- 开放列存格式：Apache Arrow/Parquet, Apache ORC。
- 数据湖相关：Apache Iceberg, Apache Hudi, Delta Lake。

## 3. 论文

工业界与学术界每年都会发表大量论文，向大众展示开发者们在当前领域总结提炼出的一些成果。很多明星项目会发表有关于他们产品的论文，介绍整体架构与一些精妙的设计，有些成果经过沉淀甚至成为了各家产品都会考虑实现的设计，相关从业人员几乎都有过了解（比如Hyper的那篇大名鼎鼎的 Morsel-Driven Parallelism<sup>[1]</sup>）。

与上一节项目学习类似，这些内容都可以在面试时作为吹逼的材料有意无意地提及，增加自己的专业程度。

实战经历：

- “DuckDB参考了一篇叫 Merge Path 的论文<sup>[2]</sup>来实现并行化的归并排序，他的原理是……”
- “Parquet对于嵌套类型的组织方式参考了 Google 的 Dremel，实现方式在其论文<sup>[3]</sup>中有所描述，他的主要结构有……”
- “我们的执行流水线也是基于 Morsel-Driven Parallelism<sup>[1]</sup> 的思想……”

在实际工作中，这些内容也许不会被用到，但可以纳入自己的知识储备，以后遇到相似问题时，也许能有一些方案可供参考。

秉着拾人牙慧的态度，笔者的论文收集有以下几个来源：同事推荐，项目中提及的引用，网络上大佬的推荐，社群中小伙伴们的讨论等。

与上节所述学习某一个项目类似，学习论文时也一般是针对某一些想解决的问题来查找和阅读，充分提高效率。

在此也为想要入坑 AP 领域的同学列举一些笔者在实际开发中接触到的文章：

- 执行框架<sup>[1][4]</sup>
- 列存存储<sup>[3][5][6]</sup>
- 基于元信息裁剪数据<sup>[7]</sup>
- 存算分离<sup>[8]</sup>
- 数据湖对比<sup>[9]</sup>
- 聚合优化<sup>[10]</sup>
- 哈希表优化<sup>[1][11][12]</sup>

## 4. 其他

在今年的秋招中，笔者注意到多家公司的多位面试官都有问到平时有用过哪些内存检测工具与性能调优工具，甚至叫面试者介绍其原理。因此笔者认为了解这些周边及生态工具也有助于增长自己的专业程度。

笔者常用的内存检测工具为 valgrind，性能分析一般使用 perf 与 flamegraph 脚本来生成火焰图或差分火焰图。不过笔者对这方面的工具了解不够多，在此就不多加赘述了。

## 5. 总结

笔者认为，校招生通过实习、学习业界明星项目、学习论文以及掌握相关工具有助于将自己包装成一个在相关领域颇有经验的开发者，提升面试官的面评。这些方法也许在未来正式工作中也会成为常态（毕竟大家都在互相借鉴）。笔者也认为，这些积累其实也是一个滚雪球的过程，因为某一个领域在同一个时代的产品，大家实现都大同小异，在数据库界甚至有一句名言叫做“数据库没有任何秘密可言”，了解的产品越多，在接触新的内容时也能了解的更快，甚至能直接猜出对应的设计与解决方案。当展现出自己的专业性之后，面试官也许也愿意与你深入交流一些产品的细节与亟待解决的问题等等。

对于方向不对口的领域，本文所述内容可能没啥卵用，还是需要扎实的八股与算法功底。但在方向对口的领域，笔者认为还是有点作用的。笔者也有幸在今年秋招拿到了略微几个薪资福利还不错的方向对口的offer。

本文主要围绕 OLAP 数据库领域展开，其他领域也可以举一反三加以扩展，部分大佬也许可以同时掌握多个领域。举个例子，你还可以把自己包装成一个有丰富经验的 C++ 开发者，深谙实际开发中的各种奇技淫巧，对流行库的实现信手拈来，把标准倒背如流。

以上就是本文的全部内容了，欢迎大家批评指正，也欢迎未来愿意从事数据库行业（虽然又卷又累又穷）的同学一起深入交流。

## 参考文献

[1] Leis, Viktor, et al. "Morsel-driven parallelism: a NUMA-aware query evaluation framework for the many-core age." Proceedings of the 2014 ACM SIGMOD international conference on Management of data. 2014.
[2] Green, Oded, Saher Odeh, and Yitzhak Birk. "Merge path-A visually intuitive approach to parallel merging." arXiv preprint arXiv:1406.2628 (2014).
[3] Melnik, Sergey, et al. "Dremel: interactive analysis of web-scale datasets." Proceedings of the VLDB Endowment 3.1-2 (2010): 330-339.
[4] Pedreira, Pedro, et al. "Velox: meta's unified execution engine." Proceedings of the VLDB Endowment 15.12 (2022): 3372-3384.
[5] Raman, Vijayshankar, et al. "DB2 with BLU acceleration: So much more than just a column store." Proceedings of the VLDB Endowment 6.11 (2013): 1080-1091.
[6] Kuschewski, Maximilian, et al. "BtrBlocks: Efficient Columnar Compression for Data Lakes." Proceedings of the ACM on Management of Data 1.2 (2023): 1-26.
[7] Edara, Pavan, and Mosha Pasumansky. "Big metadata: when metadata is big data." Proceedings of the VLDB Endowment 14.12 (2021): 3083-3095.
[8] Durner, Dominik, Viktor Leis, and Thomas Neumann. "Exploiting Cloud Object Storage for High-Performance Analytics." Proceedings of the VLDB Endowment 16.11 (2023): 2769-2782.
[9] Jain, Paras, et al. "Analyzing and Comparing Lakehouse Storage Systems." CIDR, 2023.
[10] Kohn, André, Viktor Leis, and Thomas Neumann. "Building advanced SQL analytics from low-level plan operators." Proceedings of the 2021 International Conference on Management of Data. 2021.
[11] Zheng, Tianqi, Zhibin Zhang, and Xueqi Cheng. "Saha: a string adaptive hash table for analytical databases." Applied Sciences 10.6 (2020): 1915.
[12] Gubner, Tim, Viktor Leis, and Peter Boncz. "Optimistically Compressed Hash Tables & Strings in theUSSR." ACM SIGMOD Record 50.1 (2021): 60-67.