# 每周一论文：Bridging the Archipelago betweenRow-Stores and Column-Stores for Hybrid Workloads


## 论文摘要

传统的数据库分成两种不同类型：
- **偏重事务处理（online transactional processin, OLTP）**：此类数据库将不同属性连续存储，也即按行存储。按行存储可以使得插入/更新/删除更快，毕竟一条数据的所有属性是连续存储的。这种存储模型也叫做 N-Ary Storage Model (NSM)。
- **偏重数据分析（online analytical processing, OLAP）**：此类数据库将不同数据的同一属性连续存储，也即列存储。这种存储可以使得查询操作只读关心的数据属性，而不是一整条数据，减少浪费；按列储存可以更好地支持复杂查询。这种存储模型也叫做 Decomposition Storage Model (DSM)。

很多企业架构中，这两种不同类型的任务分别有不同的技术栈不同的团队完成，且 OLAP（很多情况下也叫商业智能，Business Intelligence） 类的任务一般都离线进行；然而随着时间的推移，数据分析的价值越来越小。且技术上面临如下问题：
1. OLAP 和 OLTP 系统间通常会有几分钟甚至几小时的时延，OLAP 数据库和 OLTP 数据库之间的一致性无法保证，很难满足对分析的实时性要求很高的业务场景。
2. 企业需要维护不同的数据库以便支持两类不同的任务，管理和维护成本高。
3. 企业软件开发团队需要为不同的数据库编写查询语句，且有可能需要将不同系统的数据进行聚合，开发成本高。

[本篇论文][1]描述了单一数据库如何支持分析(Hybrid transactional/analytical processing, HTAP) 。


## HTAP 数据库列表
为了应对上述挑战，HTAP 数据库即单一数据库同时支持 OLAP/OLTP业务场景应运而生，比如：
1. 腾讯云 [TiDB][2]
2. 阿里云 [HybridDB for MySQL][3]
3. 百度 [BaikalDB 数据库][4]
3. [MemSQL][5]

## HTAP 为什么合理？

- 数据刚进入数据库的时，可称之为“热数据”；热数据在 OLTP 场景下会被频繁修改。此时数据宜行存储。
- 随着数据慢慢变久，数据越来越“冷”；这个时候数据不太可能被频繁的修改，对数据的查询和分析越来越多。此时数据宜列存储。

## 如何实现 HTAP？

本篇论文提供了一种统一的架构，同时支持 OLTP/OLAP：
- 数据不再纯粹的行存储、或者列存储；而是按照块(Tile)连续存储，即若干属性组成一个块。
![Tile](https://blog-image-1258275666.cos.ap-chengdu.myqcloud.com/HTAP-Tile.png)

- 数据库执行引擎会实时收集执行的指标，这些指标通过聚类算法(K Means) 实时调整那些一个块由那些属性组成。“热数据” 块涵盖其所有属性，随着数据变“冷”，数据块仅涵盖相关性强的属性；即在行存储和列存储是块存储的两种不同的形式，在块存储中，数据随着时间的推移，慢慢由行存储变成列存储。
![Adaptive](https://blog-image-1258275666.cos.ap-chengdu.myqcloud.com/HTAP-Adaptive.png)

- 物理的数据块之上，提供了一次能够逻辑数据块的抽象，其主要的目的在于使得数据库的执行引擎无需关系数据的存储；操作开始执行的时候创建逻辑块，在最终返回结果的时候，将逻辑快转换成物理块，整个执行过程中，执行引擎无需知道块存储的细节。
![Logical Tile](https://blog-image-1258275666.cos.ap-chengdu.myqcloud.com/Logical-Tile.png)

- HTAP 由于要支持 OLAP 场景，因此并发控制要求读不阻塞写，因此选择 MVCC 作为并发控制。
![MVCC](https://blog-image-1258275666.cos.ap-chengdu.myqcloud.com/HTAP-MVCC.png)

## 总结

在大数据推动行业发展的年代，企业往往选择多种数据库产品，分别支持在线交易、报表生成、日志存储、离线分析等，用以驱动业务的高速发展，但这种组合式解决方案，需要精细的控制不同产品间的数据流转和一致性问题，使用难度颇高，每个数据库产品间的数据同步和冗余，也带来了很高的成本开销，进一步限制了企业级应用的发展。HTAP 数据库在高度可扩展的情况下，同时支持 OLTP/OLAP，是解决这些问题的有效手段。

## 作者的其他数据库文章链接
1. [SQL：数据世界的通用语][6]
2. [数据库性能之翼：SQL 语句运行时编译][7]
3. [每周一论文：A Survey of B-Tree Locking Techniques][8]
4. [每周一论文：An Empirical Evaluation of In-Memory Multi-Version Concurrency Control][9]
5. [数据库索引数据结构总结][10]

[1]: https://15721.courses.cs.cmu.edu/spring2018/papers/10-storage/arulraj-sigmod2016.pdf
[2]: https://cloud.tencent.com/product/tidb
[3]: https://yq.aliyun.com/articles/193401
[4]: https://github.com/baidu/BaikalDB
[5]: https://www.memsql.com/
[6]: https://zhewuzhou.github.io/posts/sql_as_universe_language_in_data_world/
[7]: https://zhewuzhou.github.io/posts/sql_compilation_technology_for_performance/
[8]: https://zhewuzhou.github.io/posts/weekly-paper-a-survey-of-b-tree-locking-techniques/
[9]: https://zhewuzhou.github.io/posts/weekly-paper-an-empirical-evalution-of-in-memory-mvcc/
[10]: https://zhewuzhou.github.io/posts/database-indexes/

