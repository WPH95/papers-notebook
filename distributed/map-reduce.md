###  Map-Reduce

>  Google 老三驾马车的最简单一匹，论文已经翻了第 N 次，~~Google 论文，每次都有新感觉~~



流程

1. 输入文件会被拆成 *M* 个分片，每个大小在 16 - 64 MB
2. 会选择一个 Master，他讲会分配 *M* 个 map 任务和 *R* 个任务到空闲的机器上
3. Mapper 任务会将输入信息解析成 KV-Pair，并将处理后的 KV-Pair（中间结果）存在内存中
4. Mapper 运行同时，会根据内存中 KV-Pair 对数据大小 [1]进行持久化本地落盘，Map 会将数据进行分 Partition，例如 `Hash(Key) mod R `，准备好后向 master 汇报相应情况。同时可以自定义 Combiner Func， 在 mapper 端做一些 Reducer 的逻辑，减少中间结果传递量。
5. Master 根据 Mapper 汇报的情况，发配任务给 Reducer。Reducer 拿着这些元信息，去相应的 Mapper 所在机器通过 RPC 获取存在其本地磁盘上的对应的中间结果
6. Reducer 运行 Reduce 函数，合并相应结果

Hadoop 实现补充

1. Hadoop 实现中，引用了一个 suffle 的概念，论文中并没有这块内容。mapper 将数据复制到 reducer 这个过程被称作为 shuffle。【4的后半部分】hadoop 对 暂存在内存 KV-pair 设了一个阈值[1],达到阈值后即开始将数据落盘，并启动 shuffle 机制，这样 reducer 可以不用等待 mapper 100% 完成后开始。
2. 每个 reducer 用户定义任务执行前，会进行一个 sort 操作，加速 reducer // 论文 4.2 Ordering Guarantees



---

感受

首先 论文来自 04 年，当时作者认为 “network cross-section bandwidth” 是制约瓶颈 

MapReduce 的模型精简，可以随着机器规模的增加线性增长，并隐藏了很多分布式处理起来痛苦的地方

1. 没有复杂的 schedule，每个节点都是 worker （除了 master)
2. 追踪 task 的状态
3. 数据迁移
4. 灾后恢复

论文很大篇幅在将优化，主要是从如何降低网络流量和提升性能一块

降低网络流量

1. mapper 所需的 input 就近读取。
2. mapper 产生的中间结果存本地磁盘，不存 GFS
3. 中间数据根据 `Hash(key)` 来进行分区(partition)
4. 在 mapper 端进行 Combiner

提高性能

1. Sort，因为整个 MR 过程中，KV-pairs 都按照 升/降序 有序排列。这样利于 merge/count 等等操作 (归并排序)
2. 慢速 task 进行 backup。通过增长 10% 的计算量，提升 40 %的性能，降低长尾效应
3. 重试 错误/出现异常 的 task，如果多次异常就 skip

MR 模型不是最高效的也不是最灵活的，存在这一坨又一坨的问题

1. 维护成本贵

2. 复杂度其实不低 反而非常复杂 // 为了复杂的分析 pipline

3. 性能不行，调优调参玄学

4. 只是一个 Batch model


但还是非常的经典，作为一种 "编程范式" 看 Spark/Kafka/~~我司代码~~ 分布式领域的都能看到 MR 的影子



> 不过毕竟 04 年的东西，当今的情况有很大不同。
>
> 随着网络带宽和块存储发展，混布的优势不是那么大。
>
> 现在对流式计算的需求也高，batch streaming 合一也是个方向。



---

6.824 的提问和 一些自己想的问题



1. > What's a good data structure for implementing this?

   DAG 别问 工作流问就 DAG 就完事了

2. > Why not stream the records to the reducer (via TCP) as they are being produced by the mappers?

    实现困难，好处不大。在整个 MR 任务开始前就得指定 Mapper 发送给哪些 Reducer，如果 Reducer 挂了，还得重新 TCP // 想想就头大，就不再更细想了

3. How does detailed design reduce effect of slow network?

   上面说过，不在重复

4. How do they get good load balance

   参考论文 3.5 Task Granularity，将 mapper 和 reducer 的任务数量适当增多 (e.g. M = 100*machine)

5. Why not re-start the whole job from the beginning?

   讲道理没太懂问题意思。就当是说如果发生故障为什么可以不用整个重启...

   首先 MapReduce 的概念来自函数式编程 ，mapper 和 reducer 是纯函数(Pure function)

   // 但论文 3.3 Sementics in Presence of Failures 有说到一些 weaker semantics。当 reducer 接收到不同时期(多次启动，数据结果可能存在不同) Mapper 产生的数据。

6. > Splits the input file into M pieces, 16-64 MB

   论文里没说为什么不分成 128mb 256mb 更大的块，估计别的论文里可能有对照图吧,也有可能 GFS 为毛是大块的原因？不止谁是因谁是果，等之后再看一遍 GFS 可能就知道了，更小的块会给 GFS 产生压力，更大的块给 Fault Tolerance 带去副作用（需要重算的数据更

7. > One of the copies of the program is special -- the master Master picks idle worker and assign each one a map task or reduce task.

   意思是其实任何 Worker 都可以转换成 master [假如能拿到 master 的 “binlog”]？ master 和 worker 是相同的代码包？ 其实这点蛮有伏笔的感觉



---

REF

<https://pdos.csail.mit.edu/6.824/notes/l01.txt>

<http://matt33.com/2016/03/02/hadoop-shuffle/>

<https://zhuanlan.zhihu.com/p/34849261>

<https://stackoverflow.com/questions/22141631/what-is-the-purpose-of-shuffling-and-sorting-phase-in-the-reducer-in-map-reduce>





[1] 记得 hadoop 默认是内存的 80% 待确认