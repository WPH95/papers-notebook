---

Execution Overview

1. The MapReduce library in the user program first splits the input files into M pieces of typically **16 Mb to 64 MB per piece** (con- trollable by the user via an optional parameter). It then starts up many copies of the program on a clus- ter of machines. 
2. **One of the copies of the program is special – the master.** The rest are workers that are assigned work by the master. There are M map tasks and R reduce tasks to assign. The master picks idle workers and assigns each one a map task or a reduce task. 
3. A worker who is assigned a map task reads the contents of the corresponding input split. It parses key/value pairs out of the input data and passes each pair to the user-defined Map function. The intermediate key/value pairs produced by the Map function are buffered in memory. 
4. Periodically, the buffered pairs are written to local disk, partitioned into R regions by the partitioning function. The locations of these buffered pairs on the **local disk** are passed back to the master, who is responsible for forwarding these locations to the reduce workers. 
5. When a reduce worker is notified by the master about these locations, it uses remote procedure calls to read the buffered data from the local disks of the map workers. **When a reduce worker has read all in- termediate data, it sorts it by the intermediate keys so that all occurrences of the same key are grouped together. The sorting is needed because typically many different keys map to the same reduce task. If the amount of intermediate data is too large to fit in memory, an external sort is used.** 
6. The reduce worker iterates over the sorted interme- diate data and for each unique intermediate key encountered, it passes the key and the corresponding set of intermediate values to the user’s Reduce function. The output of the Reduce function is appended to a final output file for this reduce partition. 

7. When all map tasks and reduce tasks have been completed, the master wakes up the user program. At this point, the MapReduce call in the user pro- gram returns back to the user code. 

---





### Fault Tolerance

#### Worker Failure

The master pings every worker periodically. Similarly, any map task or reduce task in progress on a failed worker is also reset to idle and becomes eligible for rescheduling.	

#### Master Failure

It is easy to make the master write periodic checkpoints of the master data structures described above.

#### Sementics in Presence of Failures

We rely on atomic commits of map and reduce task outputs to achieve this property.

In the presence of non-deterministic operators, the output of a particular reduce task R1 is equivalent to the output for R1 produced by a sequential execution of the non-deterministic program. However, the output for a different reduce task R2 may correspond to the output for R2 produced by a different sequential execution of the non-deterministic program.
Consider map task M and reduce tasks R1 and R2. Let e(Ri) be the execution of Ri that committed (there is exactly one such execution). The weaker semantics arise because e(R1) may have read the output produced by one execution of M and e(R2) may have read the output produced by a different execution of M 



#### Locality

The MapReduce master takes the location information of the input files into account and attempts to schedule a map task on a machine that contains a replica of the corre- sponding input data.

When running large MapReduce operations on a significant fraction of the workers in a cluster, most input data is read locally and consumes no network bandwidth.

#### Task Granularity

In practice, we tend to choose M so that each individual task is roughly 16 MB to 64 MB of input data (so that the locality optimization described above is most effective), and we make R a small multiple of the num- ber of worker machines we expect to use. We often per- form MapReduce computations with M = 200, 000 and R = 5, 000, using 2,000 worker machines.



#### Backup Tasks

One of the common causes that lengthens the total time taken for a MapReduce operation is a “**straggler**”: a machine that takes an unusually long time to complete one of the last few map or reduce tasks in the computation. Stragglers can arise for a whole host of reasons. For example, a machine with a bad disk may experience frequent correctable errors that slow its read performance from 30 MB/s to 1 MB/s. The cluster scheduling system may have scheduled other tasks on the machine, causing it to execute the MapReduce code more slowly due to competition for CPU, memory, local disk, or network bandwidth. A recent problem we experienced was a bug in machine initialization code that caused processor caches to be disabled: computations on affected machines slowed down by over a factor of one hundred.



### Refinements

#### Partitioning Function

#### Ordering Guarantees

#### Combiner Function

in some cases, there is significant repetition in the intermediate keys produced by each map task, and the user specified Reduce function is commutative and associative.

The Combiner function is executed on each machine that performs a map task. Typically the same code is used to implement both the combiner and the reduce functions. 

#### Side-effects

We rely on the application writer to make such side-effects atomic and idempotent.

#### Skipping Bad Records

When the master has seen more than one failure on a particular record, it indicates that the record should be skipped when it issues the next re-execution of the corre- sponding Map or Reduce task.