# 前缀查询 prefix seek

## 为什么要有前缀查询

典型场景就是二级索引，myrocks，这种key有公共前缀，用prefix bloom filter来构建一个索引，这样搜索减少IO次数

如果iterator遍历的少，prefix seek效率高，否则和正常seek遍历没区别了。

> （笔者注） redis hash结构 都有公共前缀key ，所以可以构建key的 bloom filter，减少IO过滤
> IO大头，定位的IO和遍历的IO比还是实际遍历更消耗一些

## 定义一个prefix

通过options.prefix_extractor设置

文中的prefix等于 options.prefix_extractor.Transform()

有几个helper 比如NewFixedPrefixTransform NewCappedPrefixTransform，当然也可以定义一个自己的

如果定义了prefix_extractor， comparator也得改，需要判定相同前缀的order

通常推荐NewCappedPrefixTransform，comparator只要bytewise就可以了，性能表现也好大部分特性表现没区别

> 笔者注：默认BytewiseComparator

如果你的DB或者列族的options.prefix_extractor选项有被声明，那么rocksdb就会在一个“前缀搜索”模式，具体会在下面解释。使用的例子如下：


## 配置 prefix bloom filter

```cpp

Options options;

// Set up bloom filter
rocksdb::BlockBasedTableOptions table_options;
table_options.filter_policy.reset(rocksdb::NewBloomFilterPolicy(10, false));
table_options.whole_key_filtering = false;  // 笔者注:这个默认是true的，针对Get()的优化,注意
options.table_factory.reset(
    rocksdb::NewBlockBasedTableFactory(table_options));  // For multiple column family setting, set up specific column family's ColumnFamilyOptions.table_factory instead.

// Define a prefix. In this way, a fixed length prefix extractor. A recommended one to use.
options.prefix_extractor.reset(NewCappedPrefixTransform(3));

DB* db;
Status s = DB::Open(options, "/tmp/rocksdb",  &db);
```

## 怎么读

### 忽略prefix bloom filter

```cpp
ReadOptions read_options;
read_options.total_order_seek = true;
Iterator* iter = db->NewIterator(read_options);
Slice key = "foobar";
iter->Seek(key);  // Seek "foobar" in total order
```

如果不设置total order，这种场景默认是prefix seek

如果搜不到prefix，结果可能是错的，避免这种场景，设置total order seek，相当于忽略bloom filter

> 笔者注：一个弊端 快，但结果是错的，对，但结果很慢

### 自动模式

```cpp

ReadOptions read_options;
read_options.auto_prefix_mode = true;
Iterator* iter = db->NewIterator(read_options);
// ......
```
设置auto_prefix_mode可以保证能用prefix bloom filter就用，不能用就自动total order

6.8版本引入

一个例子

```cpp

options.prefix_extractor.reset(NewCappedPrefixTransform(3));
options.comparator = BytewiseComparator();  // This is the default
// ......
ReadOptions read_options;
read_options.auto_prefix_mode = true;
std::string upper_bound;
Slice upper_bound_slice;

// "foo2" and "foo9" share the same prefix "foo".
upper_bound = "foo9";
upper_bound_slice = Slice(upper_bound);
read_options.iterate_upper_bound = &upper_bound_slice;
Iterator* iter = db->NewIterator(read_options);
iter->Seek("foo2");

// "foobar2" and "foobar9" share longer prefix than "foo".
upper_bound = "foobar9";
upper_bound_slice = Slice(upper_bound);
read_options.iterate_upper_bound = &upper_bound_slice;
Iterator* iter = db->NewIterator(read_options);
iter->Seek("foobar2");

// "foo2" and "fop" doesn't share the same prefix, but "fop" is the successor key of prefix "foo" which is the prefix of "foo2".
upper_bound = "fop";
upper_bound_slice = Slice(upper_bound);
read_options.iterate_upper_bound = &upper_bound_slice;
Iterator* iter = db->NewIterator(read_options);
iter->Seek("foo2");
```

这种用法的局限性
- 比如NewFixedPrefixTransform 或者 比如NewFixedPrefixTransform
- 必须实现IsSameLengthImmediateSuccessor确定order，默认的BytewiseComparator是实现了的
- 目前只有Seek能用这个能力，SeekForPrev遍历不用bloom filter，你问为什么？用不着就没实现，想用可以共建
- 检查bloom filter有CPU开销

### 手动前缀遍历

```cpp
options.prefix_extractor.reset(NewCappedPrefixTransform(3));

ReadOptions read_options;
read_options.total_order_seek = false;
read_options.auto_prefix_mode = false;
Iterator* iter = db->NewIterator(read_options);

iter->Seek("foobar");
// Iterate within prefix "foo"

iter->SeekForPrev("foobar");
// iterate within prefix "foo"
```

可能存在的问题
- 可能删除的key重新被遍历到
- key顺序有问题

这个模式是最早引入到，为了保留兼容性，还留着，但是别用

## 修改 prefix extractor

文件中会有prefix extractor名字信息，如果不同就不使用新的prefix extractor，走total order seek模式

除非之前用的是内置的两个 NewFixedPrefixTransform NewFixedPrefixTransform，会走 auto_prefix_mode模式

## 其他前缀遍历特性

- prefix bloom filter设置memtable上开启 options.memtable_prefix_bloom_size_ratio
- Prefix hash index 看 https://github.com/facebook/rocksdb/wiki/Data-Block-Hash-Index
- PlainTable format 看 https://github.com/facebook/rocksdb/wiki/PlainTable-Format


## 通用API(旧文档)
```cpp
Options options;

// <---- Enable some features supporting prefix extraction
options.prefix_extractor.reset(NewFixedPrefixTransform(3));

DB* db;
Status s = DB::Open(options, "/tmp/rocksdb",  &db);

......

auto iter = db->NewIterator(ReadOptions());
iter->Seek("foobar"); // Seek inside prefix "foo"
iter->Next(); // Find next key-value pair inside prefix "foo"

```
options.prefix_extractor是一个SliceTransform类型的共享指针。通过调用SliceTransform.Transform()，我们可以我们从一个Slice里面提取出一个子串，用来代表该Slice，通常是前缀部分。在这个wiki页面，我们使用“前缀”来指代 options.prefix_extractor.Transform() 对一个 key 处理后的输出。你可以通过调用NewFixedPrefixTransform(prefix_len)，获得一个使用固定长度的前缀转换器，或者你可以根据需要实现自己的转换器，然后传给options.prefix_extractor。

当options.prefix_extractor.不是nullptr，迭代器不能保证所有key都是按顺序迭代的，只能保证相同前缀的key是按顺序迭代的。当调用Iterator.Seek(lookup_key)时，RocksDB会从lookup_key里面提取前缀。与全排序模式不同，如果数据库里面有一个或者多个key满足这个前缀，RocksDB会把迭代器放在前缀相同，或者更大的key上。如果没有前缀等于或者大于lookup_key的前缀，或者在调用几次Next之后，根据ReadOptions.prefix_same_as_start是否为true，我们处理完了相同前缀的所有key，我们可能会反回Valid()=false，或者是任何一个大于前面的key的key。从4.11之后，我们支持前缀模式下使用Prev，但是仅限于迭代器仍旧在该前缀的所有键值范围内的时候。Prev的输出在迭代器已经超出范围的时候不保证正确。

当前缀模式被打开的时候，RocksDB会为了快速定位相同前缀的key或者排除不存在的前缀，自由组织数据，或者构建搜索数据。这里有些为前缀模式进行优化的支持：数据块和memtable的prefix bloom，基于哈希的memtable，还有平表格式。一个示范设定：

```cpp
Options options;

// Enable prefix bloom for mem tables
options.prefix_extractor.reset(NewFixedPrefixTransform(3));
options.memtable_prefix_bloom_bits = 100000000;
options.memtable_prefix_bloom_probes = 6;

// Enable prefix bloom for SST files
BlockBasedTableOptions table_options;
table_options.filter_policy.reset(NewBloomFilterPolicy(10, true));
options.table_factory.reset(NewBlockBasedTableFactory(table_options));

DB* db;
Status s = DB::Open(options, "/tmp/rocksdb",  &db);

......

auto iter = db->NewIterator(ReadOptions());
iter->Seek("foobar"); // Seek inside prefix "foo"

```
从3.5版本开始，我们支持通过一个配置，使得rocksdb即使在前缀模式，也能使用全顺序。调用NewIterator的时候，打开ReadOption.total_order_seek=true就可以打开这个功能。

```cpp
ReadOptions read_options;
read_options.total_order_seek = true;
auto iter = db->NewIterator(read_options);
Slice key = "foobar";
iter->Seek(key);  // Seek "foobar" in total order
```
这个模式下，性能可能会变差。请注意，并不是所有的前缀搜索实现都支持这个功能。例如，平表的实现就不支持，所以如果你尝试这么使用，你会看到一个错误码返回。基于哈希的mentable会在使用这个功能的时候做一次昂贵的在线排序。prefix bloom和使用哈希索引的基于块的表是支持这个模式的。

## 局限

SeekToLast不支持。SeekToFirst只对部分配置有支持。如果你的迭代器需要使用这两个操作，你应该使用全排序模式。

一个常见的使用前缀索引的bug是使用逆序迭代前缀模式。这个还没有支持。如果你需要经常使用逆序迭代，你可以重新对数据进行排序，然后把前缀迭代器的顺序反过来。你可以通过自己实现一个比较器，或者使用其他方法编码你的key

## API变化（2.8->3.0）

这一节，我们会说明从2.8版本到3.0版本的API变化

### 修改前

#### 全排序搜索

这就是你认为的传统的索引行为。搜索会在一个全排序的key空间进行，把迭代器定位到第一个大于或者等于你搜索的key的位置。

```cpp
auto iter = db->NewIterator(ReadOptions());
Slice key = "foo_bar";
iter->Seek(key);

```
并不是所有表组织方式都支持全排序搜索。例如，新引入的平表格式，除非使用全排序模式打开（Options.prefix_extractor == nullptr），否则就只支持基于前缀的seek。

#### 使用ReadOptions.prefix

这是最不灵活的搜索方式。创建迭代器的时候需要支持前缀。

```cpp
Slice prefix = "foo";
ReadOptions ro;
ro.prefix = &prefix;
auto iter = db->NewIterator(ro);
Slice key = "foo_bar"
iter->Seek(key);
```

Options.prefix_extractor需要提前设置。Seek调用会被ReadOptions提供的前缀限制，这意味着，如果你希望搜索一个新的前缀，你需要重新创建迭代器。这种方式的好处是，不相关的文件可以在创建迭代器的时候被过滤掉。所以如果你希望搜索同一个前缀的不同key，他的表现会比较好。然而，我们认为这个使用方式非常少见。

#### 使用ReadOptions.prefix_seek

这个模式比 ReadOption.prefix 更加灵活。创建迭代器的时候没有预过滤。这样，通过一个迭代器就可以用于搜索不同的key和前缀了。

```cpp
ReadOptions ro;
ro.prefix_seek = true;
auto iter = db->NewIterator(ro);
Slice key = "foo_bar";
iter->Seek(key);
```

与ReadOptions.prefix一样，Options.prefix_extractor是前置条件。

### 修改的内容

很显然，三种搜索模式让人感到混乱：

- 一个模式要求另一个选项被设置（例如Options.prefix_extractor）
- 对于我们的用户而言，后面两种模式哪一个在不同的场景下比较合适，不明显。

这个修改希望解决这个问题，并且把事情变得简单明了：如果Options.prefix_extractor被设置，seek默认使用前缀模式，反之亦然。动机很简单：如果Options.prefix_extractor存在，很显然，数据可以被切片，然后前缀搜索就自然匹配这种场景。使用方式就变的统一了：

```cpp
auto iter = db->NewIterator(ReadOptions());
Slice key = "foo_bar";
iter->Seek(key);

```

### 迁移到新的用法

迁移到新的用法很简单。去除Options.prefix或者Options.prefix_seek的赋值，因为他们已经被弃用了。现在，直接搜索你的key或者前缀就好了。因为Next会穿过上限，走到不同的前缀去，你可能需要检查结束状态：

```cpp
    auto iter = DB::NewIterator(ReadOptions());
    for (iter.Seek(prefix); iter.Valid() && iter.key().starts_with(prefix); iter.Next()) {
       // do something
    }
```

