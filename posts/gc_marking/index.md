# GC 标记算法：从分阶段标记到无停顿标记

- [摘要<a id="sec-"></a>](#摘要)
- [为什么需要关注垃圾回收器？<a id="sec-"></a>](#为什么需要关注垃圾回收器)
- [垃圾回收关键 - 标记<a id="sec-"></a>](#垃圾回收关键---标记)
- [根集合<a id="sec-"></a>](#根集合)
- [CMS 回收器标记<a id="sec-"></a>](#cms-回收器标记)
- [G1 回收器标记<a id="sec-"></a>](#g1-回收器标记)
  - [SATB 算法<a id="sec-"></a>](#satb-算法)
- [C4/Z 回收器标记<a id="sec-"></a>](#c4z-回收器标记)
- [结论<a id="sec-"></a>](#结论)
- [引用](#引用)


# 摘要<a id="sec-"></a>

垃圾回收作为 Java 语言的重要特性，把开发人员从繁重的内存管理中解放出来，极大的 提高了生产效率。尽管市面上有形形色色不同的垃圾回收器，Hotspot 自带且成熟的有：

1.  Serial GC
2.  Parallel GC
3.  Parallel Old GC (Parallel Compacting GC)
4.  Concurrent Mark & Sweep GC (or “CMS”)
5.  Garbage First (G1) GC(Java 9 默认)

Oracle 正在开发或者处于试验阶段的垃圾回收有：

1.  ZGC，高吞吐量，低延时的 GC，承诺最大垃圾回收造成的应用暂停时间不大于 10 ms， 无论堆和存活对象大小多少。<sup><a id="fnr.1" class="footref" href="#fn.1">1</a></sup>
2.  Epsilon GC<sup><a id="fnr.2" class="footref" href="#fn.2">2</a></sup> ，测试目的 GC

此外，还有不少非 Oracle 主导的 GC ，比如 Redhat 的 Shenandoah GC<sup><a id="fnr.3" class="footref" href="#fn.3">3</a></sup> ，Azul 的 C4 GC 等。

形形色色的垃圾回收器让人眼花缭乱，遑论繁多的配置参数；本文试图通过提供一组通用 的视角，使得开发人员可以透过现象看本质，更好的理解和使用垃圾回收器，本篇文章主 要关注垃圾回收器的标记过程。

# 为什么需要关注垃圾回收器？<a id="sec-"></a>

在回答以上问题之前，有一个根本的问题需要回答，那就是作为一个应用程序开发人员， 为什么需要关注垃圾回收器？

实际上，尽管垃圾回收器在大多数情况下都是高效的，以至于大多数开发人员根本没有察 觉到它的存在；然而在另外一些场景下：

1.  垃圾回收器并不意味着没有内存泄露。<sup><a id="fnr.4" class="footref" href="#fn.4">4</a></sup>
2.  垃圾回收器可能导致超过 30s 的应用暂停。<sup><a id="fnr.5" class="footref" href="#fn.5">5</a></sup>

因此，即使作为应用程序开发人员，了解和学习垃圾回收器也是必要的，至少在遇到上面 两种情况的时候，可以是的开发人员更快的解决问题。

# 垃圾回收关键 - 标记<a id="sec-"></a>

程序运行过程中产生的无用对象即已死对象。对于一个程序员来讲，利用经验或者的知识 可以轻易判断一个对象已死；垃圾回收器作为一个程序，并没有可利用的经验或者知识来 做出同样的判断，因之，通过程序判断对象已死的方式如下：

1.  编译时分析
2.  引用计数
3.  可达性分析

本文主要讨论的主题是 JVM 上的垃圾回收，而在 JVM 上目前已知的垃圾回收器都基于可 达性分析；因此本文的所有描述都基于可达性分析算法。

如果一个对象可以通过当前活跃的线程“可达”，那么就判定该对象为存活对象。例如对象 对象的引用直接在活跃线程的栈上面，此种情况可称为直接可达；如果该对象持有其对象 的引用，那么这些其他对象亦可称之为可达。因此，可达性分析的关键就在于找出直接可 达的对象，这些直接可达的对象也被称为“根集合”。

可达性分析的关键在于找出那些可达也即存活的对象，其他的对象就可以视为垃圾进行回 收。那么那些对象可以算作“根集合”？

# 根集合<a id="sec-"></a>

在 ZGC 中跟集合如下：

```c++
  class ZRootsIterator {
    private:
      ZOopStorageIterator _vm_weak_handles_iter;
      ZOopStorageIterator _jni_handles_iter;
      ZOopStorageIterator _jni_weak_handles_iter;
      ZOopStorageIterator _string_table_iter;

      void do_universe(OopClosure* cl);
      void do_vm_weak_handles(OopClosure* cl);
      void do_jni_handles(OopClosure* cl);
      void do_jni_weak_handles(OopClosure* cl);
      void do_object_synchronizer(OopClosure* cl);
      void do_management(OopClosure* cl);
      void do_jvmti_export(OopClosure* cl);
      void do_jvmti_weak_export(OopClosure* cl);
      void do_jfr_weak(OopClosure* cl);
      void do_system_dictionary(OopClosure* cl);
      void do_class_loader_data_graph(OopClosure* cl);
      void do_threads(OopClosure* cl);
      void do_code_cache(OopClosure* cl);
      void do_string_table(OopClosure* cl);

      ZSerialOopsDo<ZRootsIterator, &ZRootsIterator::do_universe>                  _universe;
      ZSerialOopsDo<ZRootsIterator, &ZRootsIterator::do_object_synchronizer>       _object_synchronizer;
      ZSerialOopsDo<ZRootsIterator, &ZRootsIterator::do_management>                _management;
      ZSerialOopsDo<ZRootsIterator, &ZRootsIterator::do_jvmti_export>              _jvmti_export;
      ZSerialOopsDo<ZRootsIterator, &ZRootsIterator::do_jvmti_weak_export>         _jvmti_weak_export;
      ZSerialOopsDo<ZRootsIterator, &ZRootsIterator::do_jfr_weak>                  _jfr_weak;
      ZSerialOopsDo<ZRootsIterator, &ZRootsIterator::do_system_dictionary>         _system_dictionary;
      ZParallelOopsDo<ZRootsIterator, &ZRootsIterator::do_vm_weak_handles>         _vm_weak_handles;
      ZParallelOopsDo<ZRootsIterator, &ZRootsIterator::do_jni_handles>             _jni_handles;
      ZParallelOopsDo<ZRootsIterator, &ZRootsIterator::do_jni_weak_handles>        _jni_weak_handles;
      ZParallelOopsDo<ZRootsIterator, &ZRootsIterator::do_class_loader_data_graph> _class_loader_data_graph;
      ZParallelOopsDo<ZRootsIterator, &ZRootsIterator::do_threads>                 _threads;
      ZParallelOopsDo<ZRootsIterator, &ZRootsIterator::do_code_cache>              _code_cache;
      ZParallelOopsDo<ZRootsIterator, &ZRootsIterator::do_string_table>            _string_table;

    public:
      ZRootsIterator();
      ~ZRootsIterator();

      void oops_do(OopClosure* cl, bool visit_jvmti_weak_export = false);
};
```

尽管源码中用于枚举根集合的类型有超过十种，但大体上，可以归纳为三类：

1.  包含指向堆栈引用的全局变量。
2.  包含指向堆栈引用的寄存器变量。
3.  包含指向堆栈引用的栈内变量。

现在垃圾回收器可以基于以上的根集合来枚举存活对象；为了降低应用程序响应时间，现 代的垃圾回收器基本都是并行增量标记；然而标记过程中仍然会遇到很多挑战：

1.  应用程序线程将一个未标记的对象引用读入 CPU 缓存，然后从内存中删除该对象。
2.  未标记的对象被存储到已经标记的区域。

为了解决类似上述的问题，垃圾回收器的设计者们需要考虑一下问题：

1.  Stop The World 是简单粗暴的问题解决办法，可以使得 GC 标记尽可能精准；但这是 以应用程序响应时间为代价的，如何在标记过程中尽量避免 Stop The World?
2.  整个标记过程需要全局扫描和跟踪堆上的对象，时间相对较长，为了使得必要的 Stop The World 尽量短，有必要对标记过程进行切分，使得标记过程分成不同的阶段，如何 划分不同的阶段？

# CMS 回收器标记<a id="sec-"></a>

CMS 算法标记过程一般描述：

-   初始标记(initial-mark)：从GC Root开始，仅扫描与根节点直接关联的对象并标记， 这个过程需要 Stop The World，但是GC Root数量有限，因此时间较短
-   并发标记(concurrent-marking)：这个阶段在初始标记的基础上继续向下进行遍历标记。 这个阶段与用户线程并发执行，因此不停顿
-   重新标记(remark)：重新标记阶段会对堆上的对象进行扫描，以对并发标记阶段遭到破 坏的对象引用关系进行修复，以保证执行清理之前对象引用关系是正确的。这一阶段需 要STW，时间也比较短暂

需要说明的是在 CMS 一个垃圾回收的周期中，在对象或者一组对象重新分配的时候，使 用 card marking write barrier 来追踪对象的变化<sup><a id="fnr.6" class="footref" href="#fn.6">6</a></sup> ，且在初始标记和重新标记 的时候引入 Stop The World

# G1 回收器标记<a id="sec-"></a>

G1 的标记过程一般描述：

-   初始标记(initial-marking)：需要 Stop The World， 扫描根集合，标记所有从根集 合可直接到达的对象并将它们的字段压入扫描栈 （marking stack）中等到后续扫描。 G1使。在分代式G1模式中，初始标记阶段借用 young generation GC 的暂停， 因而没 有额外的单独的暂停阶段。
-   并发标记(concurrent-marking)：跟应用程序并行， 不断从扫描栈取出引用递归扫描 整个堆里的对象图。每扫描到一个对象就会对其标记，并将其字段压入扫描栈。重复扫 描过程直到扫描栈清空。
-   最终标记(remark)： 需要 Stop The World，在完成并发标记后，每个Java线程还会有 一些剩下的由 write barrier 记录的引用尚未处理。这个阶段就负责把剩下的引用处 理完。

## SATB 算法<a id="sec-"></a>

G1 使用的是 SATB 标记算法，主要应用于垃圾收集的并发标记阶段，解决了CMS 垃圾收 集器重新标记阶段长时间 Stop The World 的潜在风险。其算法全称是 Snapshot At The Beginning，由字面理解，是垃圾回收器开始时活着的对象的一个快照。它是通过 “根集合”穷举可达对象得到的，穷举过程中采用了三色标记法：

-   白：对象没有被标记到，标记阶段结束后，会被当做垃圾回收掉。
-   灰：对象被标记了，但是它的field还没有被标记或标记完。
-   黑：对象被标记了，且它的所有field也被标记完了。

在并发标记的过程中，应用程序可能会修改对象，使得一个白色对象被漏标

-   应用程序插入了一个从黑色对象到该白色对象的新引用
-   应用程序删除了所有从灰色对象到该白色对象的直接或者间接引用。

SATB 利用 write barrier 将所有即将被删除的引用关系的旧引用记录下来，最后以这 些旧引用为根 Stop The World 地重新扫描一遍即可避免漏标问题。 因此 G1 Remark阶 段 Stop The World 与 CMS 了的remark有一个本质上的区别，那就是这个暂停只需要扫 描有 write barrier 所追中对象为根的对象， 而 CMS 的 remark 需要重新扫描整个根 集合，因而CMS remark有可能会非常慢。

# C4/Z 回收器标记<a id="sec-"></a>

C4/Z 整个垃圾回收期间都不需要 Stop The World，是无停顿（Pauseless)垃圾回收器， 因此其标记过程也是并行的，因此 C4/Z 无视堆的大小和存活对象的多少，可以提供至多 10ms 的应用程序暂停，实际上这是非常保守的说法。

C4/Z 也是采用了增量的并行标记，跟 G1/CMS 相比，不同点在于：

1.  采用 Checkpoint 的方式，应用程序线程无需停顿，尤其是在初始标记，穷举根集合 的阶段，Checkpoint 是应用程序线程到达一个 Safepoint ，完成少量垃圾回收的工 作，然后继续业务逻辑，而垃圾回收器要等到所有的应用程序经过 Checkpoint 后， 才能开始一个垃圾回收周期；这一点跟 G1/CMS 完全不同，在G1/CMS initial mark 阶段所有的应用程序必须要处于 Safepoint 或者 Saferegion，因此存在所有的应用 程序线程需要等最慢到达 Safepoint 线程的情况。
2.  在标记过程中，新对象分配在新的内存页上，新的内存页在标记过程中被忽略。
3.  CMS/G1 使用 write barrier 由垃圾回收器线程完成追踪对象的变化，而 C4/Z 则使 用 read barrier 且由应用程序完成增量标记的任务；这种应用程序通过硬件中断自 行修复标记的过程也被成为 "Self healing"。<sup><a id="fnr.7" class="footref" href="#fn.7">7</a></sup>

通过这些不同的技术，C4/Z 在整个标记过程中都不需要 Stop The World。

# 结论<a id="sec-"></a>

随着硬件的不断提升和用户体验要求不断被拔高，人们对应用程序的响应时间要求越来越 苛刻。在这个大背景下，垃圾回收器的标记技术也不短的在改变。

本文通过分析 CMS/G1/Z/C4 垃圾回收器的标记过程，揭示出 Java 垃圾回收器的一大技 术趋势，即在大内存堆的前提下尽 GC 可能的降低对应用程序的影响；从 CMS 的分阶段 增量标记，到 G1 通过 SATB 算法改正 remark 阶段的 Stop The World 的影响，再到 Z/C4甚至在标记阶段无需 Stop The World，莫不如此。
# 引用

<a id="fn.1" href="#fnr.1">1. </a><https://wiki.openjdk.java.net/display/zgc/Main>

<a id="fn.2" href="#fnr.2">2. </a><https://wiki.openjdk.java.net/display/shenandoah/Main>

<a id="fn.3" href="#fnr.3">3. </a><http://openjdk.java.net/jeps/318>

<a id="fn.4" href="#fnr.4">4. </a><https://medium.com/@plumbr/memory-leaks-fallacies-and-misconce-8c79594a3986>

<a id="fn.5" href="#fnr.5">5. </a><https://medium.com/ai-build-techblog/jvm-garbage-collector-murder-mystery-1b6a4aec5117>

<a id="fn.6" href="#fnr.6">6. </a><http://www.cs.ucsb.edu/~ckrintz/racelab/gc/papers/ossia-concurrent.pdf>

<a id="fn.7" href="#fnr.7">7. </a><https://www.usenix.net/events/vee05/full_papers/p46-click.pdf>



