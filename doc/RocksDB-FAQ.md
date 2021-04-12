## 编译 RocksDB
**问: 编译所需的gcc最低版本**

答: 4.8.

**问: rocksdb的稳定版本?**

答: 看这里 https://github.com/facebook/rocksdb/releases . 如果是 RocksJava,看这里 https://oss.sonatype.org/#nexus-search;quick~rocksdb.

## 基本的读写
**问: 所有的操作 `Put()`, `Write()`, `Get()`  `NewIterator()` 线程安全吗?**

答: 安全.

**问: 支持多进程写?**

答:不可以，可以多进程只读模式打开，使用Secondary DB

**问: 支持多进程读?**

答: 用 `DB::OpenAsSecondary()`或者 `DB::OpenForReadOnly()`

**问: 其他线程有读写/手动compact动作时关db是线程安全的吗?**

答:不安全，使用者需要保证没有其他的读写/compact动作，可以主动调用 `CancelAllBackgroundWork()`.

**问: key value最大限制?**

答:不建议用大key/value，key 8MB value 3GB.

**问:最快导入数据到rocksdb的方法?**

答: 最快insert:

1. 单线程写入有序的kv
2. 合并key到一个 write batch再提交
3. 使用 vector memtable
4. 设置`options.max_background_flushes` 最少为 4
5. 暂停compaction，可以设置 `options.level0_file_num_compaction_trigger`, `options.level0_slowdown_writes_trigger`  `options.level0_stop_writes_trigger`设置成非常大的数， 插入完再触发 manual compaction.

3-5 可以通过调用 `Options::PrepareForBulkLoad()`自动配置

如果是离线倒入，最好通过sst文件方式导入，使用SstFileWriter，它可以让你直接创建RocksDB的SST文件，然后直接把他们加入到数据库。然而，如果你把SST文件加入到已有的RocksDB数据库，那么这些键不能跟已有的键值有交集。参看 [创建以及导入SST文件](Creating-and-Ingesting-SST-files.md)

**问: 正确删除DB的方法?**

答:先close在 `DestroyDB()` ，不close直接destroydb是为定义行为

**问: `DestroyDB()`和直接删目录有什么区别**

答: DB文件不一定都在一个目录，如果设置了`DBOptions::db_paths`, `DBOptions::db_log_dir`,  `DBOptions::wal_dir` 对应的文件回分开放

**问: map-reduce生成的key怎么导入 RocksDB?**

答: 使用SstFileWriter，它可以让你直接创建RocksDB的SST文件，然后直接把他们加入到数据库。然而，如果你把SST文件加入到已有的RocksDB数据库，那么这些键不能跟已有的键值有交集。参看 [创建以及导入SST文件](Creating-and-Ingesting-SST-files.md)

**问: 在compaction filter 回调里再读/写是安全的吗?**

答: 读可以，写可能回触发死锁(在write-stop 场景)

**问:生成snapshot 的时候，RocksDB会保留SST文件以及memtable吗？**

答：不会，参考 [snapshot工作原理](OverView.md#getiterators以及snapshots)

**问:如果设置了DBWithTTL，过期的key被删除的时间有保证吗？**

答：没有，DBWithTTL不保证一个时间上限。过期的key只在他们被压缩的时候被删除。然而，没人保证这个压缩一定会发生。例如，你有一部分key永远都不会更新，那么压缩基本不会影响这部分key所以他们也不会因过期而被删除。

**问:如果我删除了一个列族，但是不删除他的handle，我还能访问这个数据库吗？**

答：可以的，DropColumnFamily()只是把特定的列族标记为丢弃，在他的引用减少到0之前都不会被删除。

**问: 为什么我只有写请求，还会有磁盘读？**

答:  compactions导致的磁盘读

**问: Is block_size before compression , or after?**

答: block_size is for size before compression.

**问: After using `options.prefix_extractor`, I sometimes see wrong results. What's wrong?**

答: There are limitations in `options.prefix_extractor`. If prefix iterating is used, doesn't support `Prev()` or `SeekToLast()`, and many operations don't support `SeekToFirst()` either. A common mistake is to seek the last key of a prefix by calling `Seek()`, followed by `Prev()`. This is, however, not supported. Currently there is no way to find the last key of prefix with prefix iterating. Also, you can't continue iterating keys after finishing the prefix you seek to. In places where those operations are needed, you can try to set `ReadOptions.total_order_seek = true` to disable prefix iterating.

**问: If `Put()` or `Write()` is called with `WriteOptions.sync=true`, does it mean all previous writes are persistent too?**

答: Yes, but only for all previous writes with `WriteOptions.disableWAL=false`.

**问: I disabled write-ahead-log and rely on `DB::Flush()` to persist the data. It works well for single family. Can I do the same if I have multiple column families?**

答: Yes. Set `option.atomic_flush=true` to enable atomic flush across multiple column families.

**问: What's the best way to delete a range of keys?**

答: See https://github.com/facebook/rocksdb/wiki/DeleteRange .

**问: What are column families used for?**

答: The most common reasons of using column families:

1. Use different compaction setting, comparators, compression types, merge operators, or compaction filters in different parts of data
2. Drop a column family to delete its data
3. One column family to store metadata and another one to store the data.

**问: What's the difference between storing data in multiple column family and in multiple rocksdb database?**

答: The main differences will be backup, atomic writes and performance of writes. The advantage of using multiple databases: database is the unit of backup or checkpoint. It's easier to copy a database to another host than a column family. Advantages of using multiple column families:

1. write batches are atomic across multiple column families on one database. You can't achieve this using multiple RocksDB databases
2. If you issue sync writes to WAL, too many databases may hurt the performance.

**问: Is RocksDB really “lockless” in reads?**

答: Reads might hold mutex in the following situations:
1. access the sharded block cache 
2. access table cache if `options.max_open_files != -1`
3. if a read happens just after flush or compaction finishes, it may briefly hold the global mutex to fetch the latest metadata of the LSM tree.
4. the memory allocators RocksDB relies on (e.g. jemalloc), may sometimes hold locks. These locks are only held rarely, or in fine granularity.

**问: If I update multiple keys, should I issue multiple `Put()`, or put them in one write batch and issue `Write()`?**

答: Using `WriteBatch()` to batch more keys usually performs better than single `Put()`.

**问: What's the best practice to iterate all the keys?**

答: If it's a small or read-only database, just create an iterator and iterate all the keys. Otherwise consider to recreate iterators once a while, because an iterator will hold all the resources from being released. If you need to read from consistent view, create a snapshot and iterate using it.

**问: I have different key spaces. Should I separate them using prefixes, or use different column families?**

答: If each key space is reasonably large, it's a good idea to put them in different column families. If it can be small, then you should consider to pack multiple key spaces into one column family, to avoid the trouble of maintaining too many column families.

**问: Is the performance of iterator `Next()` the same as `Prev()`?**

答: The performance of reversed iteration is usually much worse than forward iteration. There are various reasons for that:
1. delta encoding in data blocks is more friendly to `Next()`
2. the skip list used in the memtable is single-direction, so `Prev()` is another binary search
3. the internal key order is optimized for `Next()`.

**问: If I want to retrieve 10 keys from RocksDB, is it better to batch them and use `MultiGet()` versus issuing 10 individual `Get()` calls?**

答: There are potential performance benefits in using `MultiGet()`. See https://github.com/facebook/rocksdb/wiki/MultiGet-Performance .

**问: If I have multiple column families and call the DB functions without a column family handle, what the result will be?**

答: It will operate only the default column family.

**问: Can I reuse `ReadOptions`, `WriteOptions`, etc, across multiple threads?**

答: As long as they are const, you are free to reuse them.

## Feature Support
**问: Can I cancel a specific compaction?**

答: No, you can't cancel one specific compaction.

**问: Can I close the DB when a manual compaction is in progress?**

答: No, it's not safe to do that. However, you call `CancelAllBackgroundWork(db, true)` in another thread to abort the running compactions, so that you can close the DB sooner. Since 6.5, you can also speed it up using `DB::DisableManualCompaction()`.

**问: Is it safe to directly copy an open RocksDB instance?**

答: No, unless the RocksDB instance is opened in read-only mode.


**问: Does RocksDB support replication?**

答: No, RocksDB does not directly support replication.  However, it offers some APIs that can be used as building blocks to support replication.  For instance, `GetUpdatesSince()` allows developers to iterate though all updates since a specific point in time. 
See https://github.com/facebook/rocksdb/wiki/Replication-Helpers

**问: Does RocksDB support group commit?**

答: Yes. Multiple write requests issued by multiple threads may be grouped together. One of the threads writes WAL log for those write requests in one single write request and fsync once if configured.

**问: Is it possible to scan/iterate over keys only? If so, is that more efficient than loading keys and values?**

答: No it is usually not more efficient. RocksDB's values are normally stored inline with keys. When a user iterates over the keys, the values are already loaded in memory, so skipping the value won't save much. In BlobDB, keys and large values are stored separately so it maybe beneficial to only iterate keys, but it is not supported yet. We may add the support in the future.

**问: Is the transaction object thread-safe?**

答: No it's not. You can't issue multiple operations to the same transaction concurrently. (Of course, you can execute multiple transactions in parallel, which is the point of the feature.)

**问: After iterator moves away from a key/value, is the memory pointed by those key/value still kept?**

答: No, they can be freed, unless you set `ReadOptions.pin_data = true` and your setting supports this feature.

**问: Can I programmatically read data from an SST file?**

答: We don't support it right now. But you can dump the data using `sst_dump`. Since version 6.5, you'll be able to do it using SstFileReader.

**问: RocksDB repair: when can I use it? Best-practices?**

答: Check https://github.com/facebook/rocksdb/wiki/RocksDB-Repairer

## Configuration and Tuning
**问: What's the default value of the block cache?**

答: 8MB. That's too low for most use cases, so it's likely that you need to set your own value.

**问: Are bloom filter blocks of SST files always loaded to memory, or can they be loaded from disk?**

答: The behavior is configurable.  When `BlockBaseTableOptions::cache_index_and_filter_blocks` is set to true, then bloom filters and index block will be loaded into a LRU cache only when related `Get()` requests are issued.  In the other case where `cache_index_and_filter_blocks` is set to false, then RocksDB will try to keep the index block and bloom filter in memory up to `DBOptions::max_open_files` number of SST files.

**问: Is it safe to configure different prefix extractor for different column family?**

答: Yes.

**问: Can I change the prefix extractor?**

答: No.  Once you've specified a prefix extractor, you cannot change it.  However, you can disable it by specifying a null value.

**问: How to configure RocksDB to use multiple disks?**

答:  You can create a single filesystem (ext3, xfs, etc.) on multiple disks. Then, you can run RocksDB on that single file system.
Some tips when using disks:

* If RAID is used, use larger RAID stripe size (64kb is too small, 1MB would be excellent). 
* Consider enabling compaction read-ahead by specifying `ColumnFamilyOptions::compaction_readahead_size` to at least 2MB.
* If workload is write-heavy, have enough compaction threads to keep the disks busy
* Consider enabling async write behind for compaction

**问: Can I open RocksDB with a different compression type and still read old data?**

答:  Yes, since RocksDB stored the compression information in each SST file and performs decompression accordingly, you can change the compression and the db will still be able to read existing files. In addition, you can also specify a different compression for the last level by specifying `ColumnFamilyOptions::bottommost_compression`.

**问: Can I put log files and sst files in different directories? How about information logs?**

答: Yes.  WAL files can be placed in a separate directory by specifying `DBOptions::wal_dir`, information logs can as well be written in a separate directory by using `DBOptions::db_log_dir`.

**问: If I use non-default comparators or merge operators, can I still use `ldb` tool?**

答: You cannot use the regular `ldb` tool in this case.  However, you can build your custom `ldb` tool by passing your own options using this function `rocksdb::LDBTool::Run(argc, argv, options)` and compile it.

**问: What will happen if I open RocksDB with a different compaction style?**

答: When opening a RocksDB database with a different compaction style or compaction settings, one of the following scenarios will happen:

1. The database will refuse to open if the new configuration is incompatible with the current LSM layout.
2. If the new configuration is compatible with the current LSM layout, then RocksDB will continue and open the database.  However, in order to make the new options take full effect, it might require a full compaction.

Consider to use the migration helper function `OptionChangeMigration()`, which will compact the files to satisfy the new compaction style if needed.

**问: Does RocksDB have columns? If it doesn't have column, why there are column families?**

答: No, RocksDB doesn't have columns. See https://github.com/facebook/rocksdb/wiki/Column-Families for what is column family.

**问: How to estimate space can be reclaimed If I issue a full manual compaction?**

答: There is no easy way to predict it accurately, especially when there is a compaction filter. If the database size is steady, DB property `rocksdb.estimate-live-data-size` is the best estimation.

**问: What's the difference between a snapshot, a checkpoint and a backup?**

答: Snapshot is a logical concept. Users can query data using program interface, but underlying compactions still rewrite existing files.

A checkpoint will create a physical mirror of all the database files using the same `Env`. This operation is very cheap if the file system hard-link can be used to create mirrored files.

A backup can move the physical database files to another `Env` (like HDFS). The backup engine also supports incremental copy between different backups.

**问: Which compression type should I use?**

答: Start with LZ4 (or Snappy, if LZ4 is not available) for all levels for good performance. If you want to further reduce data size, try to use ZStandard (or Zlib, if ZStandard is not available) in the bottommost level. See https://github.com/facebook/rocksdb/wiki/Setup-Options-and-Basic-Tuning#compression

**问: Is compaction needed if no key is deleted or overwritten?**

答: Even if there is no need to clear out-of-date data, compaction is needed to ensure read performance.

**问: After a write following `option.disableWAL=true`, I write another record with `options.sync=true`, will it persist the previous write too?**

答: No. After the program crashes, writes with `option.disableWAL=true` will be lost, if they are not flushed to SST files.

**问: What is `options.target_file_size_multiplier` useful for?**

答: It's a rarely used feature. For example, you can use it to reduce the number of the SST files.

**问: I observed burst write I/Os. How can I eliminate that?**

答: Try to use the rate limiter: See https://github.com/facebook/rocksdb/wiki/Rate-Limiter

**问: Can I change the compaction filter without reopening the DB?**

答: It's not supported. However, you can achieve it by implementing your `CompactionFilterFactory` which returns different compaction filters.

**问: How many column families can  a single db support?**

答: Users should be able to run at least thousands of column families without seeing any error. However, too many column families don't usually perform well. We don't recommend users to use more than a few hundreds of column families.

**问: Can I reuse `DBOptions` or `ColumnFamilyOptions` to open multiple DBs or column families?**

答: Yes. Internally, RocksDB always makes a copy to those options, so you can freely change them and reuse these objects.



## Portability
**问: Can I run RocksDB and store the data on HDFS?**

答: Yes, by using the Env returned by `NewHdfsEnv()`, RocksDB will store data on HDFS.  However, the file lock is currently not supported in HDFS Env.

**问: Does RocksJava support all the features?**

答: We are working toward making RocksJava feature compatible.  However, you're more than welcome to submit pull request if you find something is missing



## Backup
**问: Can I preserve a “snapshot” of RocksDB and later roll back the DB state to it?**

答: Yes, via the [BackupEngine](https://github.com/facebook/rocksdb/wiki/How-to-backup-RocksDB%3F) or [[Checkpoints]].

**问: Does `BackupableDB` create a point-in-time snapshot of the database?**

答: Yes when `BackupOptions::backup_log_files = true` or `flush_before_backup = true` when calling `CreateNewBackup()`.

**问: Does the backup process affect accesses to the database in the mean while?**

答: No, you can keep reading and writing to the database at the same time.

**问: How can I configure RocksDB to backup to HDFS?**

答: Use `BackupableDB` and set backup_env to the return value of `NewHdfsEnv()`.

## Failure Handling

**问: Does RocksDB throw exceptions?**

答: No,  RocksDB returns `rocksdb::Status` to indicate any error. However, RocksDB does not catch exceptions thrown by STL or other dependencies.  For instance, so it's possible that you will see `std::bad_malloc` when memory allocation fails, or similar exceptions in other situations.  

**问: How RocksDB handles read or write I/O errors?**

答:  If the I/O errors happen in the foreground operations such as `Get()` and `Write()`, then RocksDB will return `rocksdb::IOError` status.  If the error happens in background threads and `options.paranoid_checks=true`, we will switch to the read-only mode. All the writes will be rejected with the status code representing the background error.

**问: How to distinguish type of exceptions thrown by RocksJava?**

答: Yes, RocksJava throws `RocksDBException` for every RocksDB related exceptions.


## Failure Recovery

**问:如果我的进程crash了，我的数据库数据会受影响吗？**

答：不会，但是如果你没有开启WAL没有刷入到存储介质的memtable数据可能会丢失。

**问:如果我的机器crash了，RocksDB能保证数据的完整吗？**

答：数据在你调用一个带sync的写请求的时候会被写入磁盘（使用WriteOptions.sync=true的写请求），或者你可以等待memtable被刷入存储的时候。


**问: How to know the number of keys stored in a RocksDB database?**

答: Use `GetIntProperty(cf_handle, "rocksdb.estimate-num-keys")` to obtain an estimated number of keys stored in a column family, or use `GetAggregatedIntProperty(“rocksdb.estimate-num-keys", &num_keys)` to obtain an estimated number of keys stored in the whole RocksDB database.


**问: Why GetIntProperty can only return an estimated number of keys in a RocksDB database?**

答: Obtaining an accurate number of keys in any LSM databases like RocksDB is a challenging problem as they have duplicate keys and deletion entries (i.e., tombstones) that will require a full compaction in order to get an accurate number of keys.  In addition, if the RocksDB database contains merge operators, it will also make the estimated number of keys less accurate.

## Resource Management
**问: How much resource does an iterator hold and when will these resource be released?**

答:  Iterators hold both data blocks and memtables in memory.  The resource each iterator holds are:

1. The data blocks that the iterator is currently pointing to. See https://github.com/facebook/rocksdb/wiki/Memory-usage-in-RocksDB#blocks-pinned-by-iterators
2. The memtables that existed when the iterator was created, even after the memtables have been flushed.
3. All the SST files on disk that existed when the iterator was created, even if they are compacted.

These resources will be released when the iterator is deleted.

**问: How to estimate total size of index and filter blocks in a DB?**

答: For an offline DB, `"sst_dump --show_properties --command=none"` will show you the index and filter size for a specific sst file. You can sum them up for all DB. For a running DB, you can fetch from DB property `kAggregatedTableProperties`. Or calling `DB::GetPropertiesOfAllTables()` and sum up the index and filter block size of individual files.

**问: Can RocksDB tell us the total number of keys in the database? Or the total number of keys within a range?**

答: RocksDB can estimate number of keys through DB property `“rocksdb.estimate-num-keys”`. Note this estimation can be far off when there are merge operators, existing keys overwritten, or deleting non-existing keys.

The best way to estimate total number of keys within a range is to first estimate size of a range by calling `DB::GetApproximateSizes()`, and then estimate number of keys from that.

## Others

**问: Who is using RocksDB?**

答: https://github.com/facebook/rocksdb/blob/master/USERS.md

**问: How should I implement multiple data shards/partitions.**

答: You can use one RocksDB database per shard/partition. Multiple RocksDB instances could be run as separate processes or within a single process. When multiple instances of RocksDB are used within the single process, some resources (like thread pool, block cache, rate limiter etc..) could be shared between those RocksDB instances (See https://github.com/facebook/rocksdb/wiki/RocksDB-Basics#support-for-multiple-embedded-databases-in-the-same-process)  

**问: DB operations fail because of out-of-space. How can I unblock myself?**

答: First clear up some free space. The DB will automatically start accepting operations once enough free space is available. The only exception is if 2PC is enabled and the WAL sync fails (in this case, the DB needs to be reopened). See [[Background Error Handling]] for more details.



**问:RocksDB会抛异常嘛？**

答：不，出错的时候RocksDB会返回一个rocksdb::Status，用于指明错误。然后RocksDB本身也不捕捉来自STL库和其他依赖的异常，所以如果内存不够，你可能会遇到std::bad_malloc异常，或者类似的场景下的类似的异常。

**问:如何获取RocksDB里面存储的键值的数量？**

答：使用GetIntProperty(cf_handle, “rocksdb.estimate-num-keys") 可以获得存储在一个列族里的键的大概数量。GetAggregatedIntProperty(“rocksdb.estimate-num-keys", &num_keys) 可以获得整个RocksDB数据库的键数量。

**问:为什么GetIntProperty只能返回RocksDB总键数的估算值？**

答：在像RocksDB这种LSM数据库里获得key的总数量是一个非常有挑战性的问题，因为他们总是存储有一些重复的key和一些被删除的项目，所以如果你需要精确数量，你需要进行一次全压缩。另外，如果RocksDB的数据库使用了合并操作符，这个估算的值将更加不准确。

**问:哪些基本操作，Put(),Write(),NewIterator()，是线程安全的吗**

答：是的

**问:我可以从多个进程同时写RocksDB吗？**

答：不可以。然而，你可以在多个进程中用只读模式打开RocksDB。

**问:RocksDB支持多进程读吗？**

答：RocksDB支持多个进程以只读模式打开rocksDB，可以通过使用DB::OpenForReadOnly()打开数据库。

**问:最大支持的key和value的大小是多少**

答：RocksDB不是针对大key设计的。推荐的最大key value大小分别为8MB和3GB

**问:我可以在HDFS上跑RocksDB吗？**

答：可以，使用NewHdfsEnv()接口返回的Env，RocksDB就可以把数据存储到HDFS上了。然而HDFS上目前无法支持文件锁。

**问:我可以保存一个snapshot，然后回滚rocksDB到这个状态吗？**

答：可以，请使用[BackupEngine](How-to-backup-RocksDB%3F.md)或者[Checkpoints](Checkpoints.md)接口

**问:如何快速将数据载入rocksdb**

答：其中一种方法是直接往RocksDB里面插入数据：

- 使用单一写线程，然后按顺序插入
- 将数百个键值在一个写请求插入
- 使用vector memtable
- 确保options.max_background_flushes至少为4
- 开始插入数据前，关闭自动压缩，将options.level0_file_num_compaction_trigger, options.level0_slowdown_writes_trigger and options.level0_stop_writes_trigger 设置到非常大的值。插入完成后，开始一次手动压缩。
	
3-5条可以通过调用Options::PrepareForBulkLoad()一次完成。

如果你可以离线处理数据，有一个更加快的方法：你可以对数据进行排序，并行生成没有交集的SST文件，然后批量加载这些SST文件接口。参考 [创建以及导入SST文件](Creating-and-Ingesting-SST-files.md)

**问: RocksJava已经支持全部功能了嘛？**

答：我们还在开发RocksJava。当然，如果你发现某些功能缺失，你也可以主动提交PR。

**问:谁在用RocksDB**

答：参考 [Users](https://github.com/facebook/rocksdb/blob/master/USERS.md)

**问:如何正确删除一个DB？我可以直接对活动的DB调用DestoryDB吗？**

答：先调用close然后调用destory才是正确的方法。对一个活动的DB调用DestoryDB会导致未定义行为。

**问:调用DestoryDB和直接删除DB所在的文件夹有什么差别？**

答：如果你的数据存放在多个文件夹，DestoryDB会处理好这些文件夹。一个DB可以通过配置DBOptions::db_paths, DBOptions::db_log_dir, 和 DBOptions::wal_dir以将数据存储在不同的目录。

**问:BackupableDB会创建一个指定时间的snapshot吗？**

答：调用CreateNewBackup的时候，如果BackupOptions::backup_log_files = true或者flush_before_backup为true，就会创建。

**问:备份进程会影响到其他数据库访问吗？**

答：不会，你可以在备份的同时读写数据库。


**问:对不同的列族配置不同的前缀提取器安全吗？**

答：安全。


**问:如何配置RocksDB以使用多个磁盘？**

答：你可以在多个磁盘上创建一个文件系统（ext3，xfs，等）。然后你就可以在这个单一文件系统上跑RocksDB了。使用多个磁盘的一些tips：
- 如果使用RAID，请使用大的stripe size（64kb太小，推荐使用1MB）
- 考虑把ColumnFamilyOptions::compaction_readahead_size设置到大于2MB以打开压缩预读
- 如果主要的工作压力是写，可以增加压缩线程以保持磁盘满负载工作。
- 考虑为压缩打开一步写

**问:直接拷贝一个打开的RocksDB实例安全吗？**

答：不安全，除非这个实例是以只读模式打开。

**问:如果我用一种新的压缩方式打开RocksDB，我还能读到旧的（用其他压缩方式保存的）数据吗？**

答：可以，RocksDB会在SST文件里面保存压缩方式，所以就算换了压缩方式，仍旧能读到现有的文件。你甚至可以通过ColumnFamilyOptions::bottommost_compression给最底层换一个压缩算法。

**问:我想在HDFS上备份RocksDB，怎么配置呢？**

答：使用BackupableDB然后把backup_env设置为NewHdfsEnv()的返回值即可。






**问:如果我删除了一个列族，但是不删除他的handle，我还能访问这个数据库吗？**

答：可以的，DropColumnFamily()只是把特定的列族标记为丢弃，在他的引用减少到0之前都不会被删除。

**问:为什么RocksDB在我只做写请求的时候还会发起读请求？**

答：这些读请求来自压缩操作。RocksDB的压缩操作会读取不定数量的SST文件，然后进行合并排序之类的操作，生成新的SST文件，然后删除旧的读取的SST文件。

**问:RocksDB支持拷贝吗？**

答：不支持，RocksDB不直接支持拷贝。然而，他提供一些API可以用来帮忙支持拷贝。例如，GetUpdateSince（）允许开发者遍历从某个时间点之后的所有更新。[参考](https://github.com/facebook/rocksdb/blob/4.4.fb/include/rocksdb/db.h#L676-L687)。


**问:block_size是压缩前的大小，还是压缩后的？**

答：压缩前的。

**问:使用了options.prefix_extrator之后，我有时会看到错误的结果。哪里出错了呢？**

答：options.extrator有一些限制。如果使用前缀迭代器，那么他将不支持Prev()或者SeekToLast()，很多操作还不支持SeekToFirst()。一个常见的错误是通过Seek和Prev来找一个前缀的最后一个键。这是不支持的。目前没法通过前缀迭代器来拿到某个前缀的最后一个键。同事，如果所有prefix都查完了，你不能继续迭代这些键。如果你确实需要这些操作，你可以试着将ReadOptions.total_order_seek设置为true来关闭前缀迭代。

**问:一个迭代器需要持有多少资源，这些资源会在什么时候被释放？**

答：迭代器需要持有数据块以及内存的memtable。每个迭代器需要持有以下资源：
- 当前的迭代器所指向的所有数据块。参考 [迭代器固定的数据块](Memory-usage-in-RocksDB.md#迭代器固定的块)
- 迭代器创建的时候存在的memtable，即使这些memtable被刷到磁盘，也需要持有
- 所有在迭代器创建的时候，硬盘上的SST文件。即使这些SST文件被压缩了，也需要持有
这些资源会在迭代器删除的时候被释放

**问:我可以把日志文件和sst文件放在不同目录吗？info日志呢？**

答：可以，WAL文件可以通过DBOptions::wal_dir指定存放目录，info日志可以通过DBOptions::db_log_dir指定存放目录

**问:bloom filter的SST文件总是被加载到内存吗？还是他们会从硬盘加载**

答：这是可配置的。 BlockBaseTableOptions::cache_index_and_filter_blocks为true的时候，bloom filter以及索引块会在Get调用的时候被载入一个LRU缓存。否则，RocksDB会尝试加载DBOptions::max_open_files个SST文件的bloom filter文件及索引进入内存。

**问:发起一个Put和Write请求的时候，如果设置WriteOptions.sync=true，是不是之前写的数据也会落盘呢？**

答：如果之前写的所有请求都是WriteOptions.disableWAL=false的话，是的。

**问:我关闭了WAL，然后依赖DB::Flush来落盘数据。在单一列族的时候这很完美。如果我有多个列族，我也能这么做吗？**

答：不可以，目前 DB::Flush不是跨列族的原子操作。我们有计划支持这个功能。

**问:如果使用非默认的压缩器和合并操作符，我还能用ldb工具吗？**

答：这种情况下，常规的ldb工具是不能使用的。然而，你可以把这些东西编译进你的定制ldb工具，然后通过rocksdb::LDBTool::Run(argc, argv, options) 把参数传过去。

**问:RocksDB如何处理读写IO错误？**

答：如果IO错误发生在前端请求，例如Get和Write的时候，RocksDB会返回一个rocksdb::IOError状态。如果错误发生在后台线程并且options.paranoid_checks为true，我们会切换到只读模式。所有写操作都会被拒绝，并且返回具体的后台的错误。

**问:我可以取消某次特定的压缩吗？**

答：不可以。你没法指定。


**问:如果我用一个不同的压缩方式打开RocksDB，会发生什么？**

答：如果用不同的压缩方式打开rocksdb的数据库，一下场景可能会发生：
- 如果当前LSM布局的压缩方式与配置不兼容，数据库会拒绝打开。
- 如果新的配置跟当前的LSM布局兼容，rocksdb会继续大开数据库。然而，为了保证新的配置完全生效，rocksdb会做一次全量压缩 

**问:如何正确删除一个范围的key？**

答：参考 [这里](https://github.com/facebook/rocksdb/wiki/Delete-A-Range-Of-Keys)

**问:列族是用来干嘛的？**

答：用列族的原因包括：
- 对不同的数据使用不同的压缩方式，压缩算法， 压缩风格，合并操作符，压缩过滤器。
- 通过删除一个列族来删除其携带的所有数据。
- 一个列族用于携带另一个列族的元数据。

**问:把数据存在不同的列族和存在不同的rocksdb数据库有什么不同？**

答：主要差异在于备份，原子写以及写性能。
使用多个数据库的优点：数据库是备份(backup)和检查点(checkpoint)的单位。拷贝到另一个主机的时候，数据库比列族更方便。是用列族的优点：在一个数据库里，批量写是跨列族原子的。用多个数据库的时候没法做到这个。如果你对WAL调用sync，大量的数据库会损耗性能。

**问:RocsDB有列吗？如果没有，那为什么有列族呢？**

答：没有，RocksDB没有列。参考 [Column Families](Column-Families.md)了解什么是列族

**问:RocksDB真的在读的时候无锁吗？**

答：下列情况读请求会需要锁：
- 读区共享的缓存块。
- 读取options.max_open_files != -1的表缓存
- 如果一个读请求发生在一次落盘或者压缩之后，他可能会短暂地持有一俄国全局锁，用于加载最新的LSM树的元数据。
- RocksDB使用的内存分配器（如jmelloc），有时候会加锁。这些锁通常都是很少出现的，并且都有性能保证。

**问:如果我需要更新多个key，我应该用多个Put操作，还是把他们放到一个批量写操作，然后使用Write调用？**

答：通常使用一次Write会比多次调用Put获得更好的性能。

**问:迭代所有的key的最好方式是什么？**

答：如果是一个小的或者只读的数据库，创建一个迭代器然后遍历所有key就行了。否则，考虑隔一段时间就重新创建一个迭代器，毕竟迭代器会持有所有资源，不让他们释放。如果你需要一个一致的视图，创建一个snapshot，然后遍历这个snapshot即可。

**问:如果我有不同的键值空间，我应该用不同的前缀区分它们，还是放在不同的列族？**

答：如果每个键值空间都很大，放在不同的列族是好的。如果它们可能很小，你应该考虑把几个不同的键值空间打包到一个列族，这样就不用维护好几个不同的列族了。

**问:如何估算一次强制全压缩会归还的空间？**

答：目前没有一个简单的方法估算到精确值，特别是如果你用了压缩过滤器的时候。如果数据库的大小比较稳定，读取DB属性rocksdb.estimate-live-data-size应该是最好的估算。

**问:snapshot，checkpoint和backup有什么区别**

答：snapshot是一个逻辑概念，用户可以通过API查询数据，但是底层的压缩还是会覆盖存在的文件。
checkpoint会用同一个Env为所有数据库文件创建一个物理镜像。如果操作系统允许使用硬链接创建镜像文件，那么这个操作就比较轻量。
backup可以把物理的数据库文件移动到其他环境（如HDFS）。backup引擎还支持不同备份之间的增量复制。

**问:我应该用哪种压缩风格呢？**

答：从把所有层设置为LZ4（或者Snappy）开始，已得到一个好性能。如果你希望减小数据大小，尝试在最后一层实用Zlib。

**问:如果没有覆盖或者删除，压缩还有必要吗？**

答：即使没有过时数据，压缩仍旧是有必要的，这可以提高读性能。

**问:如果我用option.disableWAL=true发起了一个写请求，然后用options.sync=true发起另一个写请求，前面那个请求会被落盘吗？**

答：不会。程序崩溃的话，如果option.disableWAL=true的数据没有被刷入sst文件，这些数据就丢了。

**问:options.target_file_size_multiplier是用来干嘛的？**

答：这是一个非常少用的功能。你可以用这个来减小SST文件的数量。

**问:如何区分RocksJava抛出的异常？**

答：RocksJava会用RocksDBException封装所有的异常。

**问:我观察到写IO有尖峰。我应该如何消除它们？**

答：尝试使用限速器：[https://github.com/facebook/rocksdb/blob/v4.9/include/rocksdb/options.h#L875-L879](https://github.com/facebook/rocksdb/blob/v4.9/include/rocksdb/options.h#L875-L879)

**问:不重新打开rocksdb，我能修改压缩过滤器吗？**

答：不支持这种操作。然而，你可以通过编写自己的，返回不同压缩过滤器的CompactionFilterFactory来实现这个操作。

**问:对于迭代器，Next和Prev的性能是一样的吗？**

答：反向迭代器的性能通常比正向迭代差。有几个原因：1，数据块的编码方式对Next更加友好。memtable的跳表是单向的，所以每次Prev调用都是一个新的二分搜索。3，内部的key排序是为Next优化过的。

**问:一个数据库最多可以有多少个列族？**

答：用户在至少上千个列族的时候，都是不应该看到错误的。然而，列族太多会影响性能。我们不推荐用户使用超过数百个列族。

**问:RocksDB可以提供数据库里的key总数吗？或者某个范围内的key总数？**

答：rocksDB可以通过DB属性"rocksdb.estimate-num-keys"估算总key数量。注意，如果有合并操作符，写覆盖，删除不存在的键值，这个估算会偏差很大。

估算一个区间的key数量的最好办法是，先调用DB::GetApproximateSizes，然后通过该返回估算一个值。

**问:我什么时候应该使用RocksDB repair工具？有什么最佳实践吗？**

答：参考 [RocksDB Repairer](RocksDB-Repairer.md)

**问:如果我想取10个key出来，是调用MultiGet好还是调用10次Get好？**

答：性能很接近。MultiGet从同一个一致视图读取，但是不会更快。

**问:我应该怎么处理数据分片？**

答：你可以从对每个数据分片用一个rocksdb数据库开始。

**问:块缓存的默认大小是多大？**

答：8MB。

**问:如果我有好几个列族然后在调用DB函数的时候不传列族指针，会发生什么？**

答：只会操作默认列族。

**问:由于磁盘空间不足，DB操作失败了，我要如何把自己解锁呢？**

答：先清理出一些空间。然后你需要重启DB，使之恢复正常。目前没有不重启DB就恢复的方法。

**问:ReadOptions，WriteOptions可以被跨线程使用吗？**

答：只要他们是不变的，你就可以复用它们。

**问:我可以复用DBOptions或者ColumnFamilyOptions来打开多个DB或者列族吗？**

答：可以。内部实现上，RocksDB总是会拷贝一次这些选项，所以你可以修改，复用它们。

**问:RocksDB支持群提交吗？**

答：是的。多个线程提交的多个写请求会被聚集起来。你可以配置其中一个线程为这些请求通过一个写调用写WAL日志并且落盘。

**问:有办法只对key做遍历吗？如果可以，这样会比加载key和value更高效吗？**

答：不，这通常不会更高效。RocksDB的key和value一般存在一起。当用户遍历key的时候，value已经被加载到内存了，所以不管value不会有太大的提升。在BlobDB，key和value被分开存储，所以他可以在只遍历key的时候有更好的性能，不过这个还没有支持。我们以后可能会支持这个特性。

**问:事务对象是线程安全的吗？**

答：不是的。你不可以并发地对同一个事务对象进行操作。（当然，你可以平行地对多个事务对象进行操作，毕竟这个是该功能的意义）

**问:当迭代器从一个key/value上移开之后，该key/value占用的内存还会被保留吗？**

答：不，它们会被释放，除非你设置了ReadOptions.pin_data = true，并且你的设置支持这个功能。

**问:如何估算DB索引和过滤块的大小**

答：对一个离线的DB，"sst_dump --show_properties --command=none"会给出一个sst文件的索引和过滤器的大小。你可以把它们加起来，得到数据库的总大小。对于运行中的DB，你可以读取DB属性kAggregatedTableProperties。或者对每个独立的文件调用DB::GetPropertiesOfAllTables()，然后把它们加起来。

**问:我可以从程序里读取SST文件的数据吗？**

答：我们目前不支持这么做。你可以通过sst_dump来导出数据。


