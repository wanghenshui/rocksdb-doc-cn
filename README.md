# rocksdb-doc-cn

rocksdb wiki以及代码分析/每周更新笔记

基线版本是 https://github.com/johnzeng/rocksdb-doc-cn

在此表示感谢

[rocksdb wiki](https://github.com/facebook/rocksdb/wiki) 中文补充版本，结合自己的理解，注意，不是一比一翻译

会有笔者自己的私货，用 引用加上(笔者注)来标记出来，觉得不对可以多多批评

# 目录

- [概述](doc/OverView.md)
- [FAQ](doc/RocksDB-FAQ.md)
- [术语](doc/Terminology.md)
- [TODO 开发者指南](doc/Contributor-Guide.md)
- [基本操作](doc/Basic-Operations.md)
  - [迭代器](doc/Iterator.md)
  - [前缀搜索](doc/Prefix-seek.md)
  - [向前搜索](doc/SeekForPrev.md)
  - [尾部迭代器](doc/Tailing-Iterator.md)
  - [Compaction Filter](doc/Compaction-Filter.md)
  - [读-修改-写操作符](doc/Merge-Operator.md)
  - [列族](doc/Column-Families.md)
  - [创建以及导入SST文件](doc/Creating-and-Ingesting-SST-files.md)
  - [单删除](doc/Single-Delete.md)
  - [低优先级写入](doc/Low-Priority-Write.md)
  - [生存时间(TTL)支持](doc/Time-to-Live.md)
  - [事务](doc/Transactions.md)
  - [快照](doc/Snapshot.md)
  - [范围删除](doc/DeleteRange.md)
  - [原子落盘](doc/Atomic-flush.md)
  - Read-only and Secondary instances
  - Approximate Size
  - User-defined Timestamp
  - Wide Columns
  - BlobDB
  - Online Verification
- 配置选项
  - [基础选项以及调优](doc/Setup-Options-and-Basic-Tuning.md)
  - [选项字符串以及OptionMap](doc/Option-String-and-Option-Map.md)
  - [配置文件](doc/RocksDB-Options-File.md)
  - [Memtable](doc/MemTable.md)
- Journal
  - [WAL日志](doc/Write-Ahead-Log.md)
    - [WAL日志格式](doc/Write-Ahead-Log-File-Format.md)
    - [WAL恢复模式](doc/WAL-Recovery-Modes.md)
    - WAL Performance
    - WAL Compression
  - [MANIFEST](doc/MANIFEST.md)
  - Track WAL in MANIFEST
- Cache
  - [块缓存](doc/Block-Cache.md)
  - SecondaryCache (Experimental)
- [写缓冲管理器](doc/Write-Buffer-Manager.md
- [压缩/compaction](doc/Compaction.md)
  - [leveled-compaction](doc/Leveled-Compaction.md)
  - [universal-compaction](doc/Universal-Compaction.md)
  - [FIFO-compaction](doc/FIFO-compaction-style.md)
  - [手动压缩](doc/Manual-Compaction.md)
  - [子压缩](doc/Sub-Compaction.md)
  - [选择Level压缩的文件](doc/Choose-Level-Compaction-Files.md)
  - [管理磁盘空间](doc/Managing-Disk-Space-Utilization.md)
  - Trivial Move Compaction
  - Remote Compaction (Experimental)
- SST文件格式
  - [基于块的表格式](doc/Rocksdb-BlockBasedTable-Format.md)
  - [平表](doc/PlainTable-Format.md)
  - CuckooTable Format
  - Index Block Format
  - [bloom过滤器](doc/RocksDB-Bloom-Filter.md)
  - [数据块哈希索引](doc/Data-Block-Hash-Index.md)
- [IO](doc/IO.md)
  - [限流器](doc/Rate-Limiter.md)
  - [直接IO](doc/Direct-IO.md)
    -Rate Limiter
    -SST File Manager
    -Direct I/O
- [压缩/compression](doc/Compression.md)
  - [字典压缩](doc/Dictionary-Compression.md)
- Full File Checksum and Checksum Handoff
  - Background Error Handling
  - Huge Page TLB Support
- Tiered Storage (Experimental)
- Logging and Monitoring
  - [日志](doc/Logger.md)
  - [统计](doc/Statistics.md)
  - [压缩统计和数据库状态](doc/Compaction-Stats-and-DB-Status.md)
  - [性能与IO上下文](doc/Perf-Context-and-IO-Stats-Context.md)
  - [事件监听器](doc/EventListener.md)
- Known Issues
- Tests
  - Stress Test
  - Fuzzing
  - Benchmarking
- Tools / Utilities
  - [数据管理和访问工具](doc/Administration-and-Data-Access-Tool.md)
  - [checkpoint](doc/Checkpoints.md)
  - [如何备份RocksDB](doc/How-to-backup-RocksDB.md)
    Administration and Data Access Tool
    How to Backup RocksDB?
    Replication Helpers
    Checkpoints
    How to persist in-memory RocksDB database
    Third-party language bindings
    RocksDB Trace, Replay, Analyzer, and Workload Generation
    Block cache analysis and simulation tools
    - IO Tracer and Parser
- Implementation Details
  - [删除过期文件](doc/Delete-Stale-Files.md)
  - [分片索引-过滤器](doc/Partitioned-Index-Filters.md)
  - [写预备事务](doc/WritePrepared-Transactions.md)
  - [写未预备事务](doc/WriteUnprepared-Transactions.md)
  - [我们是如何维护存活SST文件的](doc/How-we-keep-track-of-live-SST-files.md)
  - [优化SST文件索引以获得更好的搜索性能](doc/Indexing-SST-Files-for-Better-Lookup-Performance.md)
  - [合并运算实现](doc/Merge-Operator-Implementation.md)
  - [RocksDB修复器](doc/RocksDB-Repairer.md)
  - [两步提交实现](doc/Two-Phase-Commit-Implementation.md)
  - [迭代器的实现](doc/Iterator-Implementation.md)
  - [模拟缓存](doc/Simulation-Cache.md)
  - [废弃 持久化读缓存](doc/Persistent-Read-Cache.md)
    - Write Batch With Index
    - DeleteRange Implementation
    - unordered_write
- Extending RocksDB
  - RocksDB Configurable Objects
  - The Customizable Class
  - Object Registry
- RocksJava
  - [RocksJava基础](doc/RocksJava-Basics.md)
  - [RocksJava性能测试](doc/RocksJava-Performance-on-Flash-Storage.md)
    Logging in RocksJava
    JNI Debugging
    RocksJava API TODO
    Tuning RocksDB from Java
    Lua
    Lua CompactionFilter
- Performance
  - [RocksDB内存使用](doc/Memory-usage-in-RocksDB.md)
  - [调优指南](doc/RocksDB-Tuning-Guide.md)
  - [写失速](doc/Write-Stalls.md)
  - [使用RocksDB实现队列服务](doc/Implement-Queue-Service-Using-RocksDB.md)
    Performance Benchmarks
    In Memory Workload Performance
    Read-Modify-Write (Merge) Performance
    Delete A Range Of Keys
    Pipelined Write
    MultiGet Performance
    Speed-Up DB Open
    Asynchronous IO
    Projects Being Developed
    Misc
    Building on Windows
    Developing with an IDE
    Open Projects
    Talks
    Publication
    Features Not in LevelDB
    How to ask a performance-related question?
    Articles about Rocks
