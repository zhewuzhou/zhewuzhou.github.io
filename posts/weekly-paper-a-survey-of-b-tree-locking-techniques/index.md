# 每周一论文：A Survey of B-Tree Locking Techniques


## 论文概要

B-Tree 及各种变种数据类型作为数据库索引已经有几十年的历史了，虽然此类数据结构功能简单，无非查询、插入和删除节点；然而并发控制却异常复杂，尤其是涉及到数据库事务的情况下。本篇[论文][1]系统的总结了基于 B-Tree 的数据库索引的并发控制，提纲挈领。

### 问题定义

数据库中的并发问题分为两类：
1. 多线程并发访问内存数据的同步问题。数据库中有很多线程共享数据，比如数据库的锁定表(Locking Table)。此类问题即编程中常见的并发控制问题，通常大家通过锁(Locking)决此类同步问题，但在数据库同样的技术却被命名为闩(Latchs)，以便却别事务并发。
2. 多个事务并发访问数据库内容的同步问题。比如两个事务并发读写问同一个索引节点，又或者两个事务同时读写同一个内存页。一般解决事务并发会用到锁(Locks)。

![Locks VS Latchs](https://blog-image-1258275666.cos.ap-chengdu.myqcloud.com/LockVsLatch.png)

### 闩的实现
#### 一切闩的基础
**CAS(Compare and Swap)**: 即 CPU 原子指令，对于给定的内存地址M，比较其值A和给定值B
> CAS 操作包含三个操作数 —— 内存位置（V）、预期原值（A）和新值(B)。如果内存位置的值与预期原值相匹配，那么处理器会自动将该位置值更新为新值。否则，处理器不做任何操作。无论V值是否等于A值，都将返回V的原值。CAS 有效地说明了：我认为位置 V 应该包含值 A；如果包含该值，则将 B 放到这个位置；否则，不要更改该位置，只告诉我这个位置现在的值即可
#### 数据库闩的实现
1. **操作系统提供的互斥锁(Mutex)**: 其特点是简单易用；加锁或者释放锁操作需要系统调用，因此非常慢，大概需要几十 ns，大概相当于 100个 CPU 指令，50次 CPU L1 缓存访问。
2. **读写锁(Reader and Writer Lock)** 可以基于自旋锁实现；通过维护一个读和一个写的原子计数，允许多线程并发读；读写锁的设计需要考虑很多问题，读和写的优先级如何保证？锁的公平性如何保证？
3. **检查并设置自旋锁(Test-and-Set Spin Lock)**: Test-and-Set 是 CPU 的原子指令，给制定内存设置给定的值，并返回旧值。Test-and-Set SpinLock 是操作系统常用的锁技术，比较高效，但并不保证公平性。
4. **排号自旋锁(Queue Based Spin Lock) MCS**: 基于链表的自旋锁
> [MCS Spinlock][2] 是一种基于链表的可扩展、高性能、公平的自旋锁，申请线程只在本地变量上自旋，直接前驱负责通知其结束自旋，从而极大地减少了不必要的处理器缓存同步的次数，降低了总线和内存的开销。笔者使用 Linux 内核开发者 Nick Piggin 的自旋锁压力测试程序对内核现有的排队自旋锁和 MCS Spinlock 进行性能评估，在 16 核 AMD 系统中，MCS Spinlock 的性能大约是排队自旋锁的 8.7 倍

排号自旋锁有以下特点：
1. 释放自旋锁时，锁的拥有者 A 必须十分小心。如果有直接后继 B，即 A 的 mcs_lock_node 结构的 next 域不为 NULL，那么只须将 B 的 waiting 域置为 0 即可。
2. 如果 A 此时没有直接后继，那么说明 A “可能”是最后一个申请者（因为判断是否有直接后继和是否是最后一个申请者的这两个子操作无法原子完成，因此有可能在操作中间来了新的申请者），这可以通过使用原子比较-交换操作来完成，该操作原子地判断 mcs_lock 是否指向 A 的 mcs_lock_node 结构，如果指向的话表明 A 是最后一个申请者，则将mcs_lock 置为 NULL；否则不改变 mcs_lock 的值。无论哪种情况，原子比较-交换操作都返回 mcs_lock 的原值。
3. 如果A 不是最后一个申请者，说明中途来了新的申请者 B，那么 A必须一直等待 B 将链表构建完整，即 A 的 mcs_lock_node 结构的 next 域不再为 NULL。最后 A 通过 next 域将 B 的 waiting 域置为 0。


无论数据库的读写，都可能需要遍历索引数；其操作过程中闩的使用如下：
- 搜索：从根节点开始，获取子节点的读闩，然后释放父节点的读闩；重复这个过程，直到找到目标节点位置。
- 插入/删除：从根节点开始，获取子节点的写闩；重复这个过程，直到找到目标节点位置；如果子节点是安全的，插入/删除不会引起树结构的变化即父节点不需要调整，可释放所有祖先写闩；乐观的插入/删除是先走搜索获得目标节点的读闩，如果目标节点并不安全，则回归上述从根节点获得写闩的过程。

虽然闩能够保证多线程修改临界内存的操作线程安全，但由于释放闩是在索引节点级别，而不是事务级别，因此仅仅用闩，还是可能导致幻读的问题。

## 锁的实现

不同于闩，锁是用来保证和解决数据库事务之间的并发问题，而不是数据库内存临界关键数据结构的并发问题；因此闩和锁的主要实现区别有：
1. 一旦某个事务获取了锁，则锁在该事务的整个生命周期发挥作用。因此事务级别的并发问题如幻读可以使用锁来解决。**数据库的隔离等级也通过锁完成。**
2. 仅就数据库索引而言，事务从索引树叶子节点获得锁。
3. 锁不跟索引树存放在一起，而是存放在锁表中；因为锁表也属于多线程共享资源，因此事务访问锁表需要用到闩。
4. 即便使用最慢的操作系统互斥锁，闩的获取和释放可以再100个 CPU 指令以内完成；而数据库锁的获取和释放往往在1000个 CPU 指令以上。
![Locking Table](https://blog-image-1258275666.cos.ap-chengdu.myqcloud.com/Locking-Table.png)

当然锁仅仅是事务并发同步的一种方式，数据库也往往通过诸如 MVCC 等技术实现事务之间的同步，尽量避免用锁（访问锁表）。

### 锁类型
- 键值锁(Key Value Locking)：顾名思义，这种锁的作用范围限于一个索引节点。
![Key Value Locking](https://blog-image-1258275666.cos.ap-chengdu.myqcloud.com/Key-Value.png)
- 间隙锁(Gap Locking)：Gap就是索引树叶子节点中插入新记录的间隙。相应的间隙锁就是加在间隙上的锁。一般情况下，事务一般首先获取键值锁，然后获取键值锁相邻的间隙锁。索引树插入新值得情况下，需要获取间隙锁。
![Gap Locking](https://blog-image-1258275666.cos.ap-chengdu.myqcloud.com/Gap-Locking.png)
- 键范围锁(Key Range Locking)：事务锁定一个范围内的键，一般情况下，事务一般首先获取键值锁，**并且**获取键值锁相邻的间隙锁，因此范围键锁可以考虑为一个键值锁加一个间隙锁。
![Key Range Locking](https://blog-image-1258275666.cos.ap-chengdu.myqcloud.com/Key-Range-Locking.png)
- 层级锁(Hierarchical Locking)：相对于前面的三中锁，这种锁的实现相对复杂；这种锁允许数据库在较大索引范围内锁定索引表，然后在此大范围锁中定义其他粒度较小的锁，如上述的三种锁。
![Hierarchical Locking](https://blog-image-1258275666.cos.ap-chengdu.myqcloud.com/Hierarchical-Locking.png)

## 数据库无锁实现的含义

通常情况下，数据库无锁实现是下面两种情况之一：
- 事务在进行过程中不获取锁，但通过闩来保证写入线程安全。比如 MVCC。
- 数据库通过原子指令完成更新写入，即不需要闩来保障线程安全，但事务在提交的时候需要通过锁来进行事务验证。

## 实际案列
### MySQL 中的锁

MySQL 的 InnoDB 引擎支持行锁和表锁，其中行锁相对于表锁粒度小而并发性好，其行锁包括：
- 键值锁(Key Value Locking) 
- 间隙锁(Gap Locking)
- 键范围锁(MySQL 中也叫做 Next Key Locking)

### PostgreSQL 中的锁

默认情况下：
> [PostgreSQL利用多版本并发控制(MVCC)来维护数据的一致性][3]。这就意味着当检索数据时，每个事务看到的都只是一小段时间之前的数据快照(一个数据库版本)，而不是数据的当前状态。这样，如果对每个数据库会话进行事务隔离，就可以避免一个事务看到其它并发事务的更新而导致不一致的数据。MVCC通过避开传统数据库系统锁定的方法，最大限度地减少锁竞争以允许合理的多用户环境中的性能。

PostgreSQL 也支持显示的行级锁
>- **FOR UPDATE**：令那些被SELECT检索出来的行被锁住，就像在更新一样。这样就避免它们在当前事务结束前被其它事务修改或者删除；也就是说， 其它企图UPDATE, DELETE, SELECT FOR UPDATE, SELECT FOR NO KEY UPDATE,SELECT FOR SHARE 或 SELECT FOR KEY SHARE 这些行的事务将被阻塞，直到当前事务结束。同样， 如果一个来自其它事务的UPDATE,DELETE,SELECT FOR UPDATE已经锁住了某个或某些选定的行，SELECT FOR UPDATE将等到那些事务结束，并且将随后锁住并返回更新的行(或者不返回行，如果行已经被删除)。但是，在REPEATABLE READ或SERIALIZABLE事务内部，如果在事务开始时要被锁定的行已经改变了，那么将抛出一个错误。
- **FOR NO KEY UPDATE**：的行为类似于FOR UPDATE，只是获得的锁比较弱：该锁不阻塞尝试在相同的行上获得锁的SELECT FOR KEY SHARE命令。该锁模式也可以通过任何不争取FOR UPDATE锁的UPDATE获得。
- **FOR SHARE**：的行为类似于FOR NO KEY UPDATE，只是它在每个检索出来的行上获得一个共享锁，而不是一个排它锁。一个共享锁阻塞其它事务在这些行上执行UPDATE,DELETE,SELECT FOR UPDATE或SELECT FOR NO KEY UPDATE，却不阻止他们执行SELECT FOR SHARE或SELECT FOR KEY SHARE。
- **FOR KEY SHARE**：的行为类似于FOR SHARE，只是获得的锁比较弱： 阻塞SELECT FOR UPDATE但不阻塞SELECT FOR NO KEY UPDATE。一个共享锁阻塞其他事务执行DELETE或任意改变键值的UPDATE， 但是不阻塞其他UPDATE，也不阻止SELECT FOR NO KEY UPDATE, SELECT FOR SHARE 或SELECT FOR KEY SHARE。

## 作者的其他数据库文章链接
1. [SQL：数据世界的通用语][4]
2. [数据库性能之翼：SQL 语句运行时编译][5]
3. [每周一论文：A Survey of B-Tree Locking Techniques][6]
4. [每周一论文：An Empirical Evaluation of In-Memory Multi-Version Concurrency Control][7]
5. [数据库索引数据结构总结][8]


[1]: https://15721.courses.cs.cmu.edu/spring2018/papers/07-latching/a16-graefe.pdf
[2]: https://www.ibm.com/developerworks/cn/linux/l-cn-mcsspinlock/index.html
[3]: https://my.oschina.net/liuyuanyuangogo/blog/497929
[4]: https://zhewuzhou.github.io/2018/08/07/SQL_as_universe_language_in_data_world/
[5]: https://zhewuzhou.github.io/2018/09/13/SQL_Compilation_Technology_For_Performance/
[6]: https://zhewuzhou.github.io/2018/09/25/Weekly-Paper-A-Survey-of-B-Tree-Locking-Techniques/
[7]: https://zhewuzhou.github.io/2018/09/29/Weekly-Paper-An-Empirical-Evalution-of-In-Memory-MVCC/
[8]: https://zhewuzhou.github.io/2018/10/18/Database-Indexes/


