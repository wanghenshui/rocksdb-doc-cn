对于一个Get()请求，RocksDB遍历可变memtable，不可变memtable列表，以及SST文件，以查找目标key。SST文件被组织成许多层。

在Level 0，文件基于他们落盘的时间进行排序。他们key范围（通过FileMetaData.smallest和FileMetaData.largest来定义）通常会相互交叉覆盖。所以他需要查找所有L0文件。

压缩会被周期性地调度起来，从上层捡取一些文件然后跟下层的一些文件合并他们。就结果而言，键值对会被从L0逐渐移动到在LSM树的下层。压缩会对键值对排序，然后把它们切分到文件中。从level 1开始往下，SST文件根据key进行排序。他们的key范围互相无交集。为了检查一个key是否落在一个SST文件中，Rocksdb不需要检查每个SST文件，只需要进行针对FileMetaData.largest进行一次二分搜索，就能定位到一个备选文件，该文件**可能**包含目标key。这把复杂度从O(N)降低到O(log(N))。然而，log(N)对于最底层仍旧是非常大的。对于一个扇出比例为10，层数为3的，可以有1000个文件。这需要10个比较来确定一个候选文件。对于一个需要[每秒进行好几百万操作](https://wanghenshui.github.io/rocksdb-doc-cn/doc/RocksDB-In-Memory-Workload-Performance-Benchmarks.html)的内存数据库，这个开销是非常大的。

针对这个问题，一个可以观察到的事实是：在LSM树构建后，一个SST文件在某一层的位置是固定的。更进一步，他相对于下一层的位置，也是固定的。基于这个观点，我们可以实施[分级层叠](http://en.wikipedia.org/wiki/Fractional_cascading)类的优化来降低二分搜索范围。这里是一个例子：

```
                                         file 1                                          file 2
                                      +----------+                                    +----------+
level 1:                              | 100, 200 |                                    | 300, 400 |
                                      +----------+                                    +----------+
           file 1     file 2      file 3      file 4       file 5       file 6       file 7       file 8
         +--------+ +--------+ +---------+ +----------+ +----------+ +----------+ +----------+ +----------+
level 2: | 40, 50 | | 60, 70 | | 95, 110 | | 150, 160 | | 210, 230 | | 290, 300 | | 310, 320 | | 410, 450 |
         +--------+ +--------+ +---------+ +----------+ +----------+ +----------+ +----------+ 
```

Level 1有2个文件，level 2 有8个文件。现在，我们希望检索key 80。一个基于FileMetaData.largest 的二分搜索告诉你，文件1是候选者。然后key 80会与FileMetaData.smallest和 FileMetaData.largest 进行比较以确定他是不是在范围内。比较显示，80小于文件的FileMetaData.smallest（100），所以文件1不可能包含key 80。我们继续检查level 2。通常，我们需要在level 2的8个文件中做二分搜索。但是因为我们已经知道目标key 80小于100，且只有一个文件1到文件3可以包含小于100的key，我们可以在搜索的时候很安全地移除其他文件。就结果而言，我们把搜索空间从8个文件降低到了3个。

我们看看另一个例子。我们希望获得key 230。一个level 1的二分搜索定位到文件2（这也暗示着key 230大于文件1的FileMetaData.largest 200）。一个与文件2的范围比较显示，目标key小于文件2的FileMetaData.smallest 300。尽管，我们不能再level 1中找这个key，但是我们得到了启示，key在范围200和300之间。在level 2任何与[200,300]没有交集的文件都可以安全地被移除。就结果而言，我们只需要检索level 2的文件5和6.

受这个概念启发，我们在压缩的时候预建立指针，从level 1的文件指向一个范围到level 2.例如，level 1的文件1指向文件3（在level 2）的左边，以及文件4的右边。文件2会指向level 2的文件6和文件7.查询的时候，这些指针会被用于根据比对结果，决定实际的二分搜索范围。

我们的压力测试显示，这个优化对[这里](https://wanghenshui.github.io/rocksdb-doc-cn/doc/RocksDB-In-Memory-Workload-Performance-Benchmarks.html)提到的设置，增加了大概5%左右的查询QPS


