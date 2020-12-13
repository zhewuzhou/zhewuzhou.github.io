# SQL：数据世界的通用语


# 目录

-   [摘要](#org00e0306)
-   [SQL 的现在](#orgad865cf)
    -   [Not Only SQL](#org5e4dca3)
    -   [要水平扩展，也要 SQL](#org12c8b94)
-   [总结](#orgcef76ff)
-   [引用](#org2761100)



<a id="org00e0306"></a>

# 摘要

毫不夸张的说，关系数据库是企业软件系统的核心，企业形形色色信息行为的背后，都有
关系数据库的支撑。

SQL 作为关系型数据库最重要的功能之一，有着悠久的历史。 随着数字化大潮的到来，
关系数据库(SQL) 又面临着新的机遇和挑战。对于 IT 行业的从业人员，了解关系数据库
和 SQL 新的发展，对于解决企业 IT 的核心问题十分必要。


<a id="orgad865cf"></a>

# SQL 的现在


<a id="org5e4dca3"></a>

## Not Only SQL

NoSQL 的兴起是对于传统的关系型数据库（SQL) 的最近的一次颠覆尝试。有几个原因导
致了 NoSQL 的兴起：

1.  相对于传统的关系型数据库，NoSQL 更容易为企业提供更好数据库可扩展性，是的企
    业能够应对日益增长的庞大的数据量。
2.  相比于传统的关系型数据库，很多优秀的 NoSQL 以开源的形式存在。
3.  很多操作在关系型数据库中没有支持，比如 JSON 数据格式全文搜索。
4.  没有严格的 Schema 限制，因此在很多情况下比较灵活。

然而很快，NoSQL 便暴露除了很多不足：

1.  没有标准的数据查询语言，不同的 NoSQL 提供了不同且不完备的 SQL 替代品；随着
    应用程序的演进，应用程序所累积的数据会越来越多，数据之间的关系会变得越来越
    复杂，在这种情况下由于 NoSQL 所提供的简单的数据查询语句不成熟且不完备，尤
    其是考虑到 NoSQL 没有严格的 Schema 限制的情况下，导致大量的应用程序和数据
    库之间的脆弱的胶水代码。
2.  NoSQL 中很多数据处理和聚合实际上都是开发人员在应用程序中手写，相比于 SQL
    广泛的标准适用性和成熟的优化方案，NoSQL 在处理数据之间的多对一和多对多关系
    以及数据之间的关联时，性能差距非常明显。

人们很快发现，原来 NoSQL 的真正的意思是 Not Only SQL。


<a id="org12c8b94"></a>

## 要水平扩展，也要 SQL

2017 年 Google 发布论文 Spanner:Becoming a SQL System<sup><a id="fnr.1" class="footref" href="#fn.1">1</a></sup> 在这篇论文里，有如下描
述：

> 尽管这些 NoSQL 系统提供了一些优势，但也确实了很多传统的关系型数据库所拥有的、
> 程序员所依赖的功能。其中最关键的是缺失了健壮的数据库查询语句，其后果是开发
> 人员需要在应用程序中手写复杂的数据处理和聚合的逻辑。因此，Google 决定将
> Spanner 转变为提供全部 SQL 特性的系统。查询的执行跟 Spanner 的其他架构特性 紧密集成。

论文的后续部分还总结了 Spanner 从 NoSQL 到 SQL 的转变原因：

> 尽管 NoSQL 功能使得用户可以很简易的加载 Spanner，在一些简单的应用场景中也显
> 得十分有用； **但 SQL 在复杂数据读取和数据运算方面提供了显著的价值。**

无独有偶，2017年8月，Kafka 发布了流式 SQL 引擎 KSQL ，为 Kafka 在处理数据时，
提供完整的 SQL 支持。 不仅仅 Kafka，RabbitMQ、Spark、Flink 等纷纷开始支持 SQL。

这种趋势正是目前正在进行当中的 NewSQL<sup><a id="fnr.2" class="footref" href="#fn.2">2</a></sup> 大潮。其目标是提供 NoSQL 一样的水
平扩展能力的和同等读写性能的情况下，支持保证原子性、一致性、隔离性和持久性
(ACID) 的事务。也就是说在可扩展性方面匹敌 NoSQL，但同时保留关系型数据库模型。
就目前来看 NewSQL 大体上可以分为三类：

1.  全新的设计的 NewSQL 系统，包括 Google Spanner、CockroachDB 和 ClustrixDB
    等。
2.  基于分片中间件的传统数据库集群，比如 Oracle 就提供了 MySQL 的 proxy。
3.  云化的数据库服务 (DBaaS)，其中最成功的莫过于 AWS Aurora

![img](https://blog-image-1258275666.cos.ap-chengdu.myqcloud.com/SQL_Universe_Lang.png)
尽管新的技术不断涌现，但 SQL 这一古老的技术示出 **强大的生命力** ；作为一个广泛
使用的标准技术，在大数据随处可见的今天， **宛然成为数据世界的通用语** 。这背后
的原因是什么呢？

1.  首先 SQL 一个成熟的标准。SQL 诞生于1974年，并在1986年正式成为国际标准。随
    后尽管数据库系统如过江之鲫，但大体上这些数据库还是会遵守这个标准。
2.  SQL 是一个非常优秀的抽象模型；对于使用者来讲，SQL 简单易用，不用关心 SQL
    背后的诸如存储、同步及先写日志等细节；对于数据库的实现着来讲，SQL 对于如何
    实现查询完全没有约束，使得查询优化成为可能，且查询优化比绝大多数普通程序员
    基于 C 和 C++ 手写的形同逻辑的实现性能更胜一筹。
3.  基于 SQL 的极致性能优化。基于生产力的考量，现代的开发大多基于高阶语言，这
    些高阶语言大多基于通用的抽象模型，比如 SQL 基于关系型代数<sup><a id="fnr.3" class="footref" href="#fn.3">3</a></sup> ，然而站在
    CPU 执行的角度来看，所有的这些通用抽象模型无一例外都是以增加额外的开销，也
    就是牺牲性能为代价；但最近运行时 SQL 编译技术的兴起使得牺牲性能最小化，开
    发人员基于 SQL 快速开发业务，SQL 在运行时由 LLVM 编译成机器码<sup><a id="fnr.4" class="footref" href="#fn.4">4</a></sup> 已获得
    最佳的性能。也就是说使用 SQL 兼顾了生产力和性能。


<a id="orgcef76ff"></a>

# 总结

SQL 这一古老的技术，实际上是一个非常优秀的抽象模型，对使用者来讲简单易用；对数
据库开发者来讲可以灵活的优化；因此展现出十分强大的生命力，随着 NewSQL 的兴起，
在数据日益重要的今天，逐渐成为数据世界的通用语。


<a id="org2761100"></a>

# 引用

<sup><a id="fn.1" href="#fnr.1">1</a></sup> <https://static.googleusercontent.com/media/research.google.com/en//pubs/archive/46103.pdf>

<sup><a id="fn.2" href="#fnr.2">2</a></sup> <https://15721.courses.cs.cmu.edu/spring2018/papers/01-intro/pavlo-newsql-sigmodrec2016.pd://15721.courses.cs.cmu.edu/spring2018/papers/01-intro/pavlo-newsql-sigmodrec2016.pdf>

<sup><a id="fn.3" href="#fnr.3">3</a></sup> <https://en.wikipedia.org/wiki/Relational_algebra>

<sup><a id="fn.4" href="#fnr.4">4</a></sup> <https://15721.courses.cs.cmu.edu/spring2018/papers/03-compilation/p539-neumann.pdf>


## 作者的其他数据库文章链接
1. [SQL：数据世界的通用语][1]
2. [数据库性能之翼：SQL 语句运行时编译][2]
3. [每周一论文：A Survey of B-Tree Locking Techniques][3]
4. [每周一论文：An Empirical Evaluation of In-Memory Multi-Version Concurrency Control][4]
5. [数据库索引数据结构总结][5]

[1]: https://zhewuzhou.github.io/2018/08/07/SQL_as_universe_language_in_data_world/
[2]: https://zhewuzhou.github.io/2018/09/13/SQL_Compilation_Technology_For_Performance/
[3]: https://zhewuzhou.github.io/2018/09/25/Weekly-Paper-A-Survey-of-B-Tree-Locking-Techniques/
[4]: https://zhewuzhou.github.io/2018/09/29/Weekly-Paper-An-Empirical-Evalution-of-In-Memory-MVCC/
[5]: https://zhewuzhou.github.io/2018/10/18/Database-Indexes/

