这份文档展示 为了在批量写请求还在写入的时候，把写memtable这个操作从准备阶段移动到未准备阶段 进行的最初设计。

```
WriteUnprepared很快就能成为可以在生产使用的功能了
```

# 目标

以下是这个项目的目标：

1. 减少内存消耗，这使得处理非常大的事务变为可能。
2. 避免由于一次性写入巨大的事务造成的写失速。

# 梗概

目前，事务会被缓存在内存，知道2PC的准备阶段。当事务非常巨大的时候，缓存的数据也会很大，并且一次性写入如此巨大的事务会对并行的事务造成负面影响。更进一步，缓存一个非常巨大的事务的所有数据会导致机器内存耗尽。对内存消耗最大的贡献者是 i)缓存的键值对，比如说，批量写  ii)每个key的锁。在这个设计，我们把工程拆分为三步，以减少内存。i)值 ii)键 iii)他们对应的锁

为了排除在巨大事务中用于缓存值使用的内存，未准备的数据会被逐步写入未预备批处理，并且在写批处理还在构建的时候就被写入数据库。然而，为了简化回滚算法，key还是被缓存起来了。当一个事务提交，每个未预备的批处理都会 更新 提交缓存中的一个项。这里主要的挑战是处理回滚，以及读取自己的写入。

事务需要可以读取他们自己的未提交事务。之前这是通过在搜索DB前，先查找缓存的数据来实现的。我们通过保持记录已经写入硬盘的未预备批量写的序列号，然后增加ReadCallback，让其在独到的数据的序列号能匹配的时候返回true，来解决这个问题。这只对巨大的事务生效，这些事务不需要让批量写请求缓存到内存。

目前，WritePrepared的回滚算法只能在没有存活快照的恢复流程之后工作。在WriteUnPrepared中，即使现在有快照，事务也可以回滚。我们这样设计回滚算法 i) 追加被事务修改key的值，在事务修改前的值，这样肯定可以取消写入 ii)提交回滚的事务。把一个回滚事务看做提交事务 极大简化了实现，因为现有的提交时处理存活快照的机制能与之无缝结合。WAL仍旧包括一个回滚标记，以保证在DB崩溃后，恢复过程能重新进行回滚。为了找到事务修改前的值，修改了的key的值必须被缓存在Transaction对象中。在没有缓存的批量写的情况下，在工程的第一阶段，我们仍需要缓存被修改的key的集合。在第二步，如果事务很大，我们从WAL中获取key集合。

RocksDB使用TransactionLockMgr追踪一个事务被锁定的key。对于巨大事务，被锁定的key的列表可能无法填入内存。自动锁定会上升至一个范围锁，用于近似一个集合的点锁。当RocksDB检测到一个事务正在锁定一个非常大的数量的在一个特定范围的key的时候，他会自动升级到区间锁。我们会在这个工程的第三阶段讨论这个。更多细节参考下面的“key锁”

# 阶段

计划是我们分三个阶段执行这个项目：

- 实现未预备批量写，但是key仍旧缓存在内存中，用于回滚和key上锁
- 回滚的时候，使用WAL来获取key集合，而不是从内存缓冲区获得。
- 对巨大的事务，实现区间锁机制。

# 实现

## WritePrepared概览

在已经有的WritePrepared策略，数据结构包括：

- PrepareHeap：一个正在处理中的准备序列号的堆
- CommitCache：一个从准备序列号到提交序列号的映射
	- max_evicted_seq_：从CommitCache淘汰的最大淘汰序列号
- OldCommitMap：从CommitCache淘汰的项，如果他们在某些线上快照还能看到的话，进入这里。
- DelayedPrepared：从PrepareHeap出来的小于max_evicted_seq_的准备序列号。

## Put

写入数据库需要使用批处理以避免因为写队列带来的额外消耗。为了避免与“批量写”冲突，他们会被成为“批量未预备(unprepared batches)”。通过批处理，我们还会节省未预备序列号的号码，这个号码我们需要生成，并且在我们的数据结构中追踪。一旦一个批处理到达一个可配置的阈值，他会被标记，以确定是不是有一个预备操作在WritePreparedTxn中，只不过一个新的WAL类型，名为BeginUnprepareXID，会被用于BeginPersistedPrepareXID（在WritePreparedTxn策略中被使用）的反面。所有的在同一个未预备批处理中的key会获得同样的序列号（除非有一个冲突的key，会把批处理拆分为多个子批处理）

提出的WAL格式： [BeginUnprepareXID] .... [EndPrepare(XID)]

这意味着，在最后准备之前，我们需要知道事务的XID（XID是一个事务生命周期中唯一的编码）。（这在Myrocks里面是这样的，因为我们在事务开始的时候生成XID）

Transaction对象会需要追踪已经写入到db的未预备事务的列表。为此，Transaction对象会包含一系列unprep_seq数字，当一个未预备批处理被写入，unprep_seq会被加入到这个集合。

在未预备批量写中，unprep_seq数字同样会被加入到未预备堆（类似WritePreparedTxn的预备堆）

## prepare

在prepare的时候，我们会把当前批量写中剩余的项写出到WAL，但是使用BeginPersistedPrepareXID来表示这个事务已经准备好。这样，在崩溃的时候，我们可以给应用返回已经准备好的事务，这样应用可以进行正确的操作。未预备事务会在恢复的时候隐式地回滚。

提出的WAL格式： [BeginUnprepareXID]...[EndPrepare(XID)] ... [BeginUnprepareXID]...[EndPrepare(XID)] ... [BeginPersistedPrepareXID]...[EndPrepare(XID)] ... ...[Commit(XID)]

在这种情况下，最后的准备获得BeginPersistedPrepareXID，而不是BeginUnprepareXID，以表示事务真的准备好了。

注意，尽管DB（sst文件）在WritePreparedTxn和WriteUnpreparedTxn是向前向后兼容的，WriteUnpreparedTxn的WAL对WritePreparedTxn不是想前兼容的：WritePreparedTxn肯定会在回复一个通过WriteUnpreparedTxn生成的WAL的时候失败，因为有新的标记类型。然而，WriteUnpreparedTxn仍旧向后兼容WritePreparedTxn，并且可以读取WritePreparedTxn的WAL，因为他们看起来是一样的。

## Commit

提交的时候，提交映射和未预备堆需要被更新。对于WriteUnprepared，一个提交会潜在地拥有多个预备序列号。所有(unprep_seq, commit_seq)对都需要被加入到提交映射，并且所有unprep_seq都必须从unprepare_heap删除。

如果提交在没有准备的时候被执行，并且事务没有提前写入未预备批处理，那么当前的未预备批量写会直接写到类似WritePreparedTxn的CommitWithoutPrepareInternal的情况中。如果事务已经写入为预备批处理，那么我们认为预备阶段也被加入。

## Rollback

在WritePreparedTxn中，回滚实现被限制在只在恢复之后进行回滚。他大概是这样实现的：

1. 对于已经写入的以准备数据，prep_seq = seq
2. 对于每个修改的key，通过prep_seq-1读取原始的数值
3. 回写原有的值，但是使用一个新的序列号，rollback_seq
4. rollback_seq被加入到提交映射里
5. prep_seq从PrepareHeap中移除

这个实现在 存在线上快照能看到prep_seq的时候，是不能工作的。因为如果max_evicted_seq增加到prep_seq之上了，我们会有 `prep_seq < max_evicted_seq < snaphot_seq < rollback_seq`。这时候，正在序列号snapshot_seq上读取的快照会假设在prep_seq的数据已经被提交了，因为`prep_seq < max_evicted_seq`且在old_commit_map里面没有记录

这个缺点在WritePreparedTxn中可以容忍，因为Mysql只会在恢复的时候回滚准备好的事务，此时不会有存活快照，因此不会有这个不一致问题。然而，在WriteUnpreparedTxn，这个场景不止发生在恢复阶段，同时会发生在用户发起的未预备事务的回滚上。

我们通过写入一个回滚标记解决这个问题，在放弃的事务后追加回滚数据，然后在提交映射里提交事务。因为事务后面追加了回滚数据，尽管被提交了，但是他不会修改数据库的状态，因此他被有效地回滚了。如果max_evicted_seq增加到超过prep_seq了，由于<prep_seq, commit_seq>被加入到CommitCache，已有路径，比如，增加淘汰项到old_commit_map，会处理存活的，满足`prep_seq < snapshot_seq < commit_seq`的快照。如果他在回滚过程中崩溃，在恢复的时候，他读取回滚标记，然后完成回滚操作。

如果DB在回滚中间泵快，恢复者会看到一些部分写入到WAL的回滚数据。因此恢复过程会最终重试完成回滚，这种部分数据会简单地 使用之前的数值，覆盖为新的回滚批处理。

回滚批处理会被 一次性，或者 如果事务很大，分成多个子事务 写入。我们未来会探讨其他的可能实现。

其他关于WriteUnpreparedTxn的回滚问题就是如何知道应该回滚哪些key了。之前，由于整个准备好的批处理缓存在了内存，因此可以值迭代写批处理来找到修改了的，需要回滚的key的集合。在这个工程，第一个迭代，我们仍旧保留把key集合写入内存的做法。在下一个迭代，如果key集合的大小增长到一个阈值，我们会从内存中清理这个key的集合，然后如果事务被丢弃了，就从WAL中读取key。每个事务已经在追踪一个列表的未预备序列号，这可以被用于查找WAL中正确的位置。

## Get

读取的路径跟WritePreparedTxn基本一致。惟一的区别在于，对于事务可以读自己的写入。目前，GetFromBatchAndDB会处理这个问题，具体做法是，在从DB拉取数据前，在ReadCallback被调用已决定哪些可读前，先检查写批处理。在没有写批处理的时候，我们需要其他机制来处理这个。

记得每个事务都维护一个unprep_seq的列表。在进入主要的可视性检查前，如WritePreparedTxn中描述的，检查一个key是否有一个存在unprep_seq的序列号，如果是，则这个key可见。这个逻辑在ReadCallback中调用，目前不能支持一个序列号的集合，但是这个可以被拓展，这样unprep_seq的集合就能往下传了。

目前，从DB读数据的时候，Get和Seek会直接查找 快照声明的 序列号，这样同一个事务写入的未提交的数据会潜在地，在检查可视性的逻辑前，被跳过。为了解决这个问题，如果当前事务足够大，使得它的写批处理缓冲被删除然后以未预备批处理写入DB，这个优化会被删除。

## Recovery

恢复过程会跟WritePreparedTxn一样工作，除了一些修改用于决定事务状态（未预备，预备，放弃，提交）。

在恢复过程中，带相同的XID的未预备批处理必须被追踪，直到看到EndPrepare标记。如果恢复结束，但是没看到EndPrepare，那么事务就是未预备的，并且等价的，暗示着一个应用回滚。

如果恢复以EndPrepare结束，但是没有提交记录，那么事务已经准备好，然后会出现在应用中。

如果一个回滚标记在EndPrepare之后被发现，但是没有提交标记，那么事务就是被丢弃了，并且恢复过程必须使用它们之前的数据覆盖被修改的key。

如果一个提交标记被发现，那么这个事务就是被提交了。

## 延迟预备

巨大的事务也可能会花费很长时间。如果一个事物在一段时间之后没有提交（对于一个1k TPS的工作压力，是1分钟），他的序列号会被移入DelayedPrepared，目前是一个简单的，被锁保护的集合。如果最后当前的实现变成了一个瓶颈，我们会改变DelayedPrepared，把他从一个集合（set）修改为一个分片的哈希表，类似于事务锁key的方式。如果对于key锁（发生的更加频繁）已经足够好用，那么用于追踪预备好的事务应该也足够了。

## key锁

目前，rocksdb支持通过一个TransactionLockMgr中的分片哈希表来实现点锁。每一次申请一个锁，这个key会被哈希然后会检查同一个key是否有锁。如果有，线程挂起，否则，获得锁，并且插入哈希表。这意味着所有锁上的key会存在内存中，对于巨大的事务，可能造成问题。

为了解决这个问题，当一个事务被探测到在一个区间内申请了非常多锁的时候，区间锁可以用于近似替代这个巨大集合的点锁。我们这里展现一个初步用于解决这个问题的实现，以展现这是可行的。当到达这个项目的这个阶段的时候，我们会重新考虑其他方案，以及/或者是否并行区间锁是否已经解决了这个问题

为了支持去检索，key空间需要被分割成N个逻辑分片，每个分片代表一个key空间内的连续的区间。一个分片key会代表每个分片，并且可以从key本身通过一个应用提供的回调计算得到。如果在一个分片的被上锁的key达到一定阈值，一个分片key会自动写上锁，此时每个独立的key锁会释放。

一个集合会被用于持有所有的分片。一个分片会有以下结构：
```
struct Partition { 
	map<TransactionID, int> txn_locks; 
	enum { UNLOCKED, LOCKED, LOCK_REQUEST } 
	status = UNLOCKED; 
	std::shared_ptr part_mutex; 
	std::shared_ptr part_cv; 
	int waiters = 0;
	 }
```

当我们申请一个key的锁的时候：

- 检测对应的分片结构。如果它不存在，那么就创建，并插入他。如果状态位UNLOCKED，那么增加txn_locks[id]++，否则，增加waiters，然后在part_cv挂起。当线程被唤醒，减少waiters然后重复这个步骤。
- 在点锁哈希表上请求一个点锁
	- 如果点锁获取成功，那么检查txn_locks[id]以确认阈值是否达到。
	- 如果点锁超时，减小txn_locks[id]--

为了升级一个锁：

- 把分片状态设置为LOCK_REQUEST，增加waiters，然后在part_cv挂起，知道txn_locks只包含当前当前事务。被唤醒的时候，减少waiters然后重新检查txn_locks。
- 设置状态为LOCKED
- 通过点锁哈希表，删除本分片所有的点锁。需要移除的锁可以通过tracked_keys_来得到。
- 更新tracked_keys_以移除该分片所有点锁，然后增加分片锁到tracked_keys_

为了解锁一个点锁：

- 通过点锁哈希表，删除点锁
- 减少对应分片的txn_locks。
- 如果waiters为非零，给part_cv发信号。否则，删除分片。

为了解锁一个分批锁（没有用户API触发这个，他发生在事务结束的时候）：

- 如果waiters为非零，给part_cv发信号。否则，删除分片。

注意，任何时候，只要从分片读数据，他的互斥锁part_mutex就必须被持有。

