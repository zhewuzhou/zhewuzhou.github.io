# 数据库性能之翼：SQL 语句运行时编译


# 数据库性能之翼：SQL 语句运行时编译

## 摘要
现代服务器的一大特点是内存越来越大，对于运行在这些服务器上的数据库，性能的瓶颈是 CPU 而非内存；然而传统 SQL 执行模型即“火山模型” 诞生于内存是瓶颈的年代，其以行为单位的迭代执行过程虽然灵活，但对 CPU 非常不友好。

在这个大背景下，SQL 语句运行时编译技术应运而生，为传统的关系型数据库的 SQL  执行性能插上翅膀。

一些 SQL 编译的实现如下：
1. Apache Spark Tungsten 引擎运行时将 SQL 语句的 Where 部分转换成抽象语法树，然后再讲抽象语法树运行时编译成 Java 字节码。
2. Oracle 数据库将 SQL 语句运行时转换为 C\C++ 代码，然后将其编译为机器码。
3. Postgres 11 中提供了基于 LLVM 的即时编译(JIT) 的 [SQL 语句执行引擎][1]。有实验证明 Postgres 上 SQL 语句编译技术能够将 Postgres 的事务处理能力[提升20%之500%][2]。

随着 CPU 和多核瓶颈日益凸显，SQL 语句编译技术会成为重要的数据库性能提升技术。希望读者通过本篇博客能够了对 SQL 编译技术有个大概的认识。

## 为什么要进行 SQL 编译
### 通用（抽象） vs 定向优化
SQL 是一个非常优秀的抽象模型；对于使用者来讲，SQL 简单易用，不用关心 SQL
背后的诸如存储、同步及先写日志等细节；从而使得：
1. SQL 可以运行在任何一台计算机上
2. 开发人员不需要关心 SQL 语句的执行过程，通过陈述的方式描述业务逻辑

另外一方面，现代的计算机硬件在不断进步，SQL 语句运行速度的关键是针对这些硬件进行定向优化。

SQL 运行时编译正好可以弥补通用抽象和定向优化之间的性能差距。

### 火山模型
火山模型是数据库成熟的 SQL 语句解释执行方案，该模型是一个数据库执行器实现中非常流行的“设计模式”。该设计模式将关系型代数中的每一种操作抽象成一个 Operator，整个 SQL 语句在这种情况下形成一个 Operator 树；通过自顶向下的调用 next 接口，火山模型能够以数据库行为单位处理数据。
![SQL](https://blog-image-1258275666.cos.ap-chengdu.myqcloud.com/SQL.png)
![Volcano Model](https://blog-image-1258275666.cos.ap-chengdu.myqcloud.com/Vocano-Model.png)

火山模型有如下特点：
1. 首先该模型以数据行为单位处理数据，每一行数据的处理，都会调用 next 接口多次；当 SQL 语句涉及的数据行数特别多的情况下，next 的调用次数会相当大。
2. next 接口的调用，是通过虚函数机制；相比于直接调用函数，虚函数机制需要的 CPU 指令更多，因此也更加昂贵。
3. 以行为单位的数据处理，会导致 CPU 缓存使用效率低下和一些不必要的复杂性；首先数据库必须记住处理到哪一行，以便处理跳到下一行；其次处理完一行后需要将下一行加载到 CPU 缓存中，而实际上 CPU 缓存所能存储的数据行数远不止一行。
4. [火山模型最大的好处是工程上非常干净][4]，每个Operator有良好的抽象，只需要关心自己负责的逻辑就可以，比如Filter只需要关心如何根据谓词过滤数据，Aggregates只需要关心如何聚合数据。

一言以蔽之，火山模型首先会导致更多的 CPU 指令，更为严重的是会导致 CPU 缓存效率低下，严重影响数据库的性能；而 SQL 语句编译技术就是针对这些问题进行优化。

## SQL 编译技术细节

## 减少不必要的指令
从根本上讲，提高数据库性能的方法是减少 CPU 指令数量；有实验之处数据库处理事务过程中，正真有用的指令，即用于事务逻辑的指令[不到5%][3]
![DB-Useful-Work](https://blog-image-1258275666.cos.ap-chengdu.myqcloud.com/DB-Useful-Work.png)
SQL 语句编译成机器码的过程，就是通过优化有效的降低 CPU 指令数的过程。

## 增加 CPU 缓存效率
火山模型以 Operator 为中心进行事务处理，其弊端非常明显；SQL 编译技术为了最大可能的将数据保留在 CPU 寄存器中，采取如下措施：
1. 以数据为中心，同事处理多行数据，而不是一行；自底向上，而不是自顶向下。
![VS](https://blog-image-1258275666.cos.ap-chengdu.myqcloud.com/Volcano-VS-Push.png)
2. SQL 通过 LLVM 编译成机器码，编译优化使得数据能够尽可能久的停留在CPU寄存器中。实际上编译成机器码使得 SQL 语句能够得到硬件和编译器优化技术进步带来的替身，这是火山模型无法匹敌的。
3. 同一批数据尽可能久的停留在 CPU 寄存器中，即数据不变而 Operator 流转，相比于火山模型，这种方式更像是数据不断地被 Push 给 Operator，而不是 Operator Pull 数据。

这种以多行数据为单位，基于 Push 的技术应用于数据库 SQL 执行引擎，可带来[5倍][5]的性能提升。

## 代码生成
### 代码到代码
这种方式基本上是将 SQL 语句转换成 C\C++ 代码，有如下特点：
1.  对于给定的 SQL 语句，通过模板将其替换为同等逻辑的 C\C++ 代码，然后编译成机器码
2. 一般情况下会另起一个进程或者线程，运行诸如 gcc 等成熟的编译器，对生成的 C\C++ 代码进行一步编译
3. 编译好的代码通过运行时链接，可调用数据库其他方法
4. 这种方式能够带来 SQL 语句执行效率的提升，然而如果 SQL 语句过大，可能导致编译时间长，总体时间反而不如解释执行的情况。

### JIT
这种方式并非将 SQL 语句直接编译成机器码，而是首先将其编译成中间代码，比如 LLVM IR，有如下特点：
1. 由于 JIT 的方式使得数据库编译器拥有 SQL 运行时的数据，通过数据生存周期分析等技术优化，有针对性的进行优化和编译，让数据能够尽可能久的停留在 CPU 缓存中；上述 Push 数据的 SQL 执行引擎成基于 JIT 实现。
2. SQL 语句编译并非没有代价，实际上编译时间和 SQL 语句的复杂程度是线性关系；由于编译器拥有运行时数据，且数据库引擎以一组数据单位，因此编译器可以实时评估收益，在执行下一组数据的时候在如下方式间灵活无缝的切换：
- 简单 SQL 语句解释执行
- 中等规模的SQL 语句不经优化编译成机器码执行
- 复杂的的 SQL 语句深度优化编译成机器码执行

3. 简单的[算法][6]如下
``` cpp
//Dispatch
dispatch(handleB, state):
  nextMorsel = grabMorsel()
  if (handleB.isCompiled()):
    handleB.fn(state, nextMorsel)
  else:
    VM.execute(handleB.byteCode, state, nextMorsel)
  choice = choice = extrapolatePipelineDurations(...)
  if (choice != DoNothing):
    unAsync(λ -> handleB.fn = handleB.compile(choice))

//Evaluate
// f: worker function
// n: remaining tuples
// w: active worker threads
extrapolatePipelineDurations(f, n, w):
  r0 = avg(rateinthreadRates)
  r1 = r0*speedup1(f); c1 = ctime1(f)
  r2 = r0*speedup2(f); c2 = ctime2(f)
  t0 = n / r0 / w
  t1 = c1 + max(n - (w-1)*r0*c1, 0) / r1 / w
  t2 = c2 + max(n - (w-1)*r0*c2, 0) / r2 / w
  switchmin(t0, t1, t2):
    case t0: return DoNothing
    case t1: return Unoptimized
    case t2: return Optimized
```
## 作者的其他数据库文章链接
1. [SQL：数据世界的通用语][7]
2. [数据库性能之翼：SQL 语句运行时编译][8]
3. [每周一论文：A Survey of B-Tree Locking Techniques][9]
4. [每周一论文：An Empirical Evaluation of In-Memory Multi-Version Concurrency Control][10]
5. [数据库索引数据结构总结][11]

[1]: https://www.postgresql.org/docs/11/static/jit-decision.html
[2]: https://www.pgcon.org/2017/schedule/events/1092.en.html
[3]: https://15721.courses.cs.cmu.edu/spring2018/papers/02-inmemory/hstore-lookingglass.pdf
[4]: https://www.zhihu.com/question/52220920/answer/340220500
[5]: https://www.pgcon.org/2017/schedule/events/1092.en.html
[6]: https://15721.courses.cs.cmu.edu/spring2018/papers/03-compilation/kohn-icde2018.pdf
[7]: https://zhewuzhou.github.io/posts/sql_as_universe_language_in_data_world/
[8]: https://zhewuzhou.github.io/posts/sql_compilation_technology_for_performance/
[9]: https://zhewuzhou.github.io/posts/weekly-paper-a-survey-of-b-tree-locking-techniques/
[10]: https://zhewuzhou.github.io/posts/weekly-paper-an-empirical-evalution-of-in-memory-mvcc/
[11]: https://zhewuzhou.github.io/posts/database-indexes/

