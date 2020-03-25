---
title: 深入浅出 LevelDB (6) - SST文件
author: "Atwood"
tags: ["LevelDB", "Databases", "C++", "数据引擎"]
categories: ["Release"]
date: 2020-03-23
---

## SST 文件
上一篇我们提到, 当memtable的大小达到一定的阈值, 会变为immutable memtable, 然后由后台线程异步刷新到磁盘上. LevelDB里的数据在磁盘上就是以SST文件的格式存储的. 最新版本的SSTable文件名后缀为ldb, 同时保留了sst后缀的兼容. 这一节我们就来看看SST文件相关的实现. 

在LevelDB源码的doc文件夹下有关于sst文件的文档[table_format.txt](https://github.com/google/leveldb/blob/master/doc/table_format.md), 里面介绍了sst文件的设计和结构.

这里, 我先放一张自己制作的图, 看看SSTable文件的整体结构:

![sst](/images/leveldb_sst/sst.png)

第一眼看上去会觉得十分复杂, 但是没关系, 我会一步一步的把这个图拆解开来逐个解释. 

## Record 记录
首先, 让我们从基础开始讲起. 在上图的右上角, 我们可以看到一个一个的record. 其实代表的就是用户插入的一个一个键值对, 它的结构放大来看是这样:

![record](/images/leveldb_sst/record.png)

我们上一节讲到, skiplist里面保存的每一个节点的格式是memtable_key+memtable_value, 所以每个节点里都保存有user key的全部信息. 但是实际上, 对于相邻的节点, user key有较大几率是有一定重合的. 所以在把数据保存到磁盘上的sst文件的时候, 为了节省空间, LevelDB对user key利用相同的前缀做了一定的压缩. 

每一条记录有 `key共享长度` ＋ `key非共享长度`＋ `value长度` ＋ `key非共享内容` + `value内容组成`. 两条记录如果user key有相同的部分, 那么第二条记录可以和第一条共享相同的前缀. 

比如我们有两条记录需要保存. 第一条记录为hello_world:9，第二条记录为hello_you:18.

那么第一条记录存储格式为:

`0` + `11` + `1` + `hello_world` + `9`

`0`是因为默认第一条记录的共享长度为0. `11`是"hello_world"的长度, 因为第一条记录没有共享前缀, 所以非共享的长度就是key的整个长度. `1`是第一条记录的value在varint编码下会占用的字节长度. 因为没有共享, 所以最后两个部分就是完整的user key和user value.

第二条记录存储格式为：

`6` + `3` + `1` + `you` + `18`

`6`是因为第二条记录和第一条有共享的`hello_`前缀, 长度为6. `3`是因为非共享的部分`you`的长度为3. `1`是第二条记录的value在varint编码下会占用的字节长度. 然后最后两个部分是key的非共享内容: `you` 和 user value: `18`.
 
虽然共享前缀能够减少存储空间, 但是假如我们有1000条记录, 我们不可能全部和第一条记录的前缀去计算共享的部分, 因为随着key的距离间隔变大, key也会变的越来越不一样, 相同的部分会减少. 所以我们需要隔一段距离设置一个`restart point`, 把当前的key设置为比较的基准点, 之后的key会和新的基准点进行前缀的比较. LevelDB里默认是每隔16条记录就会设置一个`restart point`.

## Block 块
在存储数据到sst文件中时, LevelDB把sst文件划分为一个一个block, 根据不同的类型又分为data block, meta block, meta block index, data block index. 其中meta block和meta block index只有在打开数据库时传入filter policy参数才会有, 所以这里为了不一次性讲述太多细节, 这里就不详细解释了. 对于block我们只关注data block和data block index.

这里我们先关注data block, 因为文件的这个部分保存的是用户的键值对信息, 也就是上面我们介绍的一条一条的记录. 每个block的大小默认为4096KB. sst文件的索引, 也就是data block index部分, 保存的其实就是每个data block的起始地址和那个block相应的key值信息. 所以在查询的时候, 我们能够迅速定位到自己想要查询的user key值在哪个block中

我们来看看data block这一部分的放大图:

![block](/images/leveldb_sst/block.png)

我们从左往右看, 每个block包含三个部分:
- block content: 里面是具体的数据部分
- compression type: block content里面的数据可能是被压缩过了, 比如LevelDB支持用snappy进行压缩, 所以这里需要指明block content里的数据是如何压缩的. 长度为1字节.
- CRC number: 这里存放block content加上compression type的CRC32校验码, 用来确保block数据的完整性和正确性, 长度为4字节. 

Block Content中, 存放的是一条一条的记录, 还有我们之前提到的`restart point`信息, 因为一个block中会有多个重启点(每隔16条记录就会有一个), 所以`restart point`的信息保存在array里, 我们需要记录每个重启点在文件里的偏移量还有这个array的长度. 其中, 每个重启点的偏移量所占用的文件大小为4字节, 假设一共有100个重启点, 则这部分的信息一共会占用100*4 + 4(记录长度本身) = 404字节.

## index block 索引块
上面我们提到了数据块的结构, SST文件中还有一个重要的部分, 那就是索引. 没有索引, 在查找一个key时, 我们只能遍历整个文件, 一条一条记录的查找, 效率不用我多说肯定会很差. 

SST文件的索引块里, 保存的就是当前文件中每个block的在文件中的起始偏移量和每个block对应的key的信息, 可以方便LevelDB在查找一个user key的时候快速的定位到user key在哪一个block中. 索引块里保存的也是一条一条的记录, 每一条记录对应着一个block. 因为data block里的数据是根据key有序的, 意味着对于相邻的两个block, 前一个block里最大的key, 一定小于后一个block中最小的key. 

所以索引块里每一条数据分为两个部分:
- 第一部分 就是一个key值, 这个值一定大于等于他所指向的block里最大的key值, 也必须小于它所指向的block后面一个block中最小的key值. 
- 第二部分, 就是一个BlockHandle对象. BlockHandle的作用就是方便我们快速的在文件中定位某一个位置. BlockHandle对象里有两个字段:
   - offset: block的起始位置在当前文件中的偏移量. 
   - size: block的长度.

索引块的结构和数据块基本一样, 只是在数据块里, 存储的是一个个user key和user value. 而在索引块里, 存储的是一个个key和偏移量.

## Footer 文件尾
在每个SST文件的末尾, 还有一个Footer, 它的结构是这样的.

![footer](/images/leveldb_sst/footer.png)

它存储了data block index和meta block index的block handle, 方便我们在打开文件的时候能快速定位到索引在哪. Footer长度为固定2个BlockHandle的最大长度(20)和1个64bit魔数长度, 这样方便解析.
BlockHandle实际长度和最大长度之间的空隙填充为0. metaindex_handle指出了meta index block的起始位置和大小. index_handle指出了index block的起始地址和大小. 这两个字段都是BlockHandle对象，可以理解为索引的索引，通过Footer可以直接定位 到metaindex和index block. 最后一个部分是一个固定的常量, 被称为魔数(0xdb4775248b80fb57)。


## SST文件整体
了解了SST文件的每个部分, 我们可以来看看文章开头那个大图最左边的部分, 也就是SST文件的宏观组成结构:

![overview](/images/leveldb_sst/overview.png)

现在看起来应该就比较清楚了, 一个sst文件, 由多个data block, 多个meta block(这里可以暂时忽略, 因为只有在打开数据库时提供了filter policy才会有), 一个meta index block(可以暂时忽略, 是前面meta block的索引), 一个data block index和一个footer组成.

前面铺垫了这么多, 我们现在可以来看看源码了, 这一篇我们主要focus在sst文件的写入, 读取以后再讲.

## 源码

### BlockBuilder
首先我们来看看每个block是怎么生成的, LevelDB里定义了一个BlockBuilder类, 专门生成每一个block. 这一部分源码在[leveldb/table/block_builder.h](https://github.com/google/leveldb/blob/master/table/block_builder.h)和[leveldb/table/block_builder.cc](https://github.com/google/leveldb/blob/master/table/block_builder.cc)中.

![blockbuilder](/images/leveldb_sst/blockbuilder.svg)

其中最重要的就是`Add`和`Finish`方法, 我们一个一个看.

#### Add
```c++
void BlockBuilder::Add(const Slice& key, const Slice& value) {
  Slice last_key_piece(last_key_);
  assert(!finished_);
  // 确保两个重启点之间的条数没有大于option里面设置的值
  assert(counter_ <= options_->block_restart_interval);
  // 确保buffer里要不就是空的, 要不新的key一定大于上一条插入的key. 因为数据是存在跳表里, 理应是有序的
  assert(buffer_.empty()  // No values yet?
         || options_->comparator->Compare(key, last_key_piece) > 0);
  size_t shared = 0;
  if (counter_ < options_->block_restart_interval) {
    // See how much sharing to do with previous string
    // counter小于options_->block_restart_interval, 说明这个key还可以压缩
    const size_t min_length = std::min(last_key_piece.size(), key.size());
    // 找出来当前key和上一条key共享的长度
    while ((shared < min_length) && (last_key_piece[shared] == key[shared])) {
      shared++;
    }
  } else {
    // Restart compression
    // 说明需要开启一个新的重启点, 并且这个key不压缩
    restarts_.push_back(buffer_.size());
    counter_ = 0;
  }
  const size_t non_shared = key.size() - shared;

  // Add "<shared><non_shared><value_size>" to buffer_
  // 按照前面讲过的record的格式写入到buffer里.
  PutVarint32(&buffer_, shared);
  PutVarint32(&buffer_, non_shared);
  PutVarint32(&buffer_, value.size());

  // Add string delta to buffer_ followed by value
  // 添加共享前缀之后的字符串到buffer里, 如果key是abcdef, 共享前缀是abc, 这key.data() + shared是d字符开始的指针, 然后长度为def的长度,
  // 就是3
  buffer_.append(key.data() + shared, non_shared);
  buffer_.append(value.data(), value.size());

  // Update state
  // 把last_key设成这次新加入的key
  last_key_.resize(shared);
  last_key_.append(key.data() + shared, non_shared);
  assert(Slice(last_key_) == key);
  counter_++;
}
```

#### CurrentSizeEstimate
```c++
// 计算block当前的大致大小. 因为block是由一个一个的key/value pair和重启点的内容和重启点array的size组成的, 所以加起来就好
size_t BlockBuilder::CurrentSizeEstimate() const {
  return (buffer_.size() +                       // Raw data buffer
          restarts_.size() * sizeof(uint32_t) +  // Restart array
          sizeof(uint32_t));                     // Restart array length
}
```

#### Finish
在当前block大小达到option里配置的block最大大小时, 就会触发Finish方法, 把标志位finished_设置为true, 表示当前block已经结束, 然后在buffer后面加上restart point的信息, 并返回这整个block的内容.

```c++
// 每个Block的结尾, 都会加上每个重启点的位置, 还有重启点的数量
Slice BlockBuilder::Finish() {
  // Append restart array
  for (size_t i = 0; i < restarts_.size(); i++) {
    PutFixed32(&buffer_, restarts_[i]);
  }
  PutFixed32(&buffer_, restarts_.size());
  finished_ = true;
  return Slice(buffer_);
}
```

### TableBuilder
另一个重要的类是TableBuilder, 上面的BlockBuilder只是负责生成block的内容, 具体监控block的文件大小, 并把一个个block的内容还有其他元数据整合成SST文件的格式, 都是由TableBuilder负责的. 这部分的源码在[leveldb/table/table_builder.cc](https://github.com/google/leveldb/blob/master/table/table_builder.cc)中.

先看看TableBuilder的结构:

![tablebuilder](/images/leveldb_sst/tablebuilder.svg)

Rep其实是个struct, 专门用来存储TableBuilder的配置和一些状态

![rep](/images/leveldb_sst/rep.svg)

TableBuilder里其实最主要的就是Add, Flush和Finish方法:
- Add: 往当前block中写入一条数据
- Flush: 当前block大小达到阈值, 结束当前block写入.
- Finish: 当前sst文件大小达到阈值, 结束当前文件的写入.

#### Add
```c++
void TableBuilder::Add(const Slice& key, const Slice& value) {
  Rep* r = rep_;
  assert(!r->closed);
  if (!ok()) return;
  if (r->num_entries > 0) {
      // 当前插入的key需要大于之前的key
    assert(r->options.comparator->Compare(key, Slice(r->last_key)) > 0);
  }

  // 当pending_index_entry为空时, 证明上一个block已经结束了, 然后传入的key是新的block的第一个key, 这个时候我们可以为上一个block生成index
  // 之所以等插入新的block的第一个key的时候再生成index, 是因为这样可以节省index空间. 比如上一个block的最后的key是
  // "the quick brown fox", 新block的第一个key是 "the who", 这样我们可以把上一个block的index key设成"the r", 因为它大于上一个block
  // 的最后一个key, 也小于新block的第一个key, 但是这个只有当我们看到"the who"的时候才知道
  if (r->pending_index_entry) {
    assert(r->data_block.empty());
    // 找到last_key和当前key最短的separator, 比如按照上面comment里的例子就是the r, 并赋值到last_key里面
    r->options.comparator->FindShortestSeparator(&r->last_key, key);
    std::string handle_encoding;
    // pending_handle里存的是上一个block的地址信息, 开始地址和长度, 这里
    r->pending_handle.EncodeTo(&handle_encoding);
    // 往index_block里面写上一个block的索引信息, 对应的是key和地址
    r->index_block.Add(r->last_key, Slice(handle_encoding));
    //赋值为false，开始新一个数据块写
    r->pending_index_entry = false;
  }

  if (r->filter_block != nullptr) {
    r->filter_block->AddKey(key);
  }

  // 将last_key再重写为这一次插入的key
  r->last_key.assign(key.data(), key.size());
  r->num_entries++;
  r->data_block.Add(key, value);

  const size_t estimated_block_size = r->data_block.CurrentSizeEstimate();
  // 如果table里当前的block_size大于了option里设定的值, 则触发一次flush
  if (estimated_block_size >= r->options.block_size) {
    Flush();
  }
}
```

最后我们可以看到, 当block大小到达一定阈值时, 会触发一次flush, 将block写到磁盘中. 这个flush并不是BlockBuilder里的flush, 我们来看看它的实现

### Flush
```c++
// 将当前data block写到磁盘中
void TableBuilder::Flush() {
  Rep* r = rep_;
  assert(!r->closed);
  if (!ok()) return;
  if (r->data_block.empty()) return;
  assert(!r->pending_index_entry);
  // 将block写到内存缓冲区, 等待flush到磁盘
  WriteBlock(&r->data_block, &r->pending_handle);
  if (ok()) {
    r->pending_index_entry = true;
    // 将文件flush到磁盘上
    r->status = r->file->Flush();
  }
  if (r->filter_block != nullptr) {
    r->filter_block->StartBlock(r->offset);
  }
}
```

### WriteBlock
上面的Flush方法调用了WriteBlock方法. 这个方法会获取到block里的内容, 对它做压缩, 并且传给更加底层的WriteRawBlock方法去做实际的写入.

```c++
void TableBuilder::WriteBlock(BlockBuilder* block, BlockHandle* handle) {
  // File format contains a sequence of blocks where each block has:
  //    block_data: uint8[n]
  //    type: uint8
  //    crc: uint32
  assert(ok());
  Rep* r = rep_;
  // 调用blockbuilder里的finish方法, 获取了data block里的内容  
  Slice raw = block->Finish();

  Slice block_contents;
  // 根据options里的compression配置对block_content做压缩
  CompressionType type = r->options.compression;
  // TODO(postrelease): Support more compression options: zlib?
  switch (type) {
    case kNoCompression:
      block_contents = raw;
      break;

    case kSnappyCompression: {
      std::string* compressed = &r->compressed_output;
      if (port::Snappy_Compress(raw.data(), raw.size(), compressed) &&
          compressed->size() < raw.size() - (raw.size() / 8u)) {
        block_contents = *compressed;
      } else {
        // Snappy not supported, or compressed less than 12.5%, so just
        // store uncompressed form
        block_contents = raw;
        type = kNoCompression;
      }
      break;
    }
  }
  // 把压缩后的内容写到用户态缓冲区
  WriteRawBlock(block_contents, type, handle);
  // 重置block的相关数据, 准备重新开始写
  r->compressed_output.clear();
  block->Reset();
}
```

#### WriteRawBlock
上面的方法又调用了WriteRawBlock, 这个方法把压缩后的block内容, 连同压缩的方法加上CRC校验码一起写到用户态缓冲区

```c++
void TableBuilder::WriteRawBlock(const Slice& block_contents,
                                 CompressionType type, BlockHandle* handle) {
  Rep* r = rep_;
  // 更新pending_handle里的offset, 为接下来生成索引做准备
  handle->set_offset(r->offset);
  // 设置block的长度
  handle->set_size(block_contents.size());
  // 将block内容写进用户态缓冲区
  r->status = r->file->Append(block_contents);
  if (r->status.ok()) {
    // 在每个block最后加上5个byte, 1个为compression type, 4个为crc校验码
    char trailer[kBlockTrailerSize];
    trailer[0] = type;
    uint32_t crc = crc32c::Value(block_contents.data(), block_contents.size());
    crc = crc32c::Extend(crc, trailer, 1);  // Extend crc to cover block type
    EncodeFixed32(trailer + 1, crc32c::Mask(crc));
    r->status = r->file->Append(Slice(trailer, kBlockTrailerSize));
    if (r->status.ok()) {
      // 将文件偏移量移到下一个block开始的地方
      r->offset += block_contents.size() + kBlockTrailerSize;
    }
  }
}
```

#### Finish
上面的Add和Flush, 是以block为粒度的, 对于每一个block, 在写完以后会调用flush刷到磁盘上, 但是这写的都是data block. 在sst文件整个的大小达到一定的容量时, 会触发Finish方法, 在文件后加上索引块和footer, 变成最终的sst文件. 

```c++
// 最后为sst文件写入index block和footer
Status TableBuilder::Finish() {
  Rep* r = rep_;
  Flush();
  assert(!r->closed);
  r->closed = true;

  BlockHandle filter_block_handle, metaindex_block_handle, index_block_handle;

  // Write filter block
  if (ok() && r->filter_block != nullptr) {
    WriteRawBlock(r->filter_block->Finish(), kNoCompression,
                  &filter_block_handle);
  }

  // Write metaindex block
  if (ok()) {
    BlockBuilder meta_index_block(&r->options);
    if (r->filter_block != nullptr) {
      // Add mapping from "filter.Name" to location of filter data
      std::string key = "filter.";
      key.append(r->options.filter_policy->Name());
      std::string handle_encoding;
      filter_block_handle.EncodeTo(&handle_encoding);
      meta_index_block.Add(key, handle_encoding);
    }

    // TODO(postrelease): Add stats and other meta blocks
    WriteBlock(&meta_index_block, &metaindex_block_handle);
  }

  // Write index block
  if (ok()) {
      // 写入最后一个block的index
    if (r->pending_index_entry) {
      r->options.comparator->FindShortSuccessor(&r->last_key);
      std::string handle_encoding;
      r->pending_handle.EncodeTo(&handle_encoding);
      r->index_block.Add(r->last_key, Slice(handle_encoding));
      r->pending_index_entry = false;
    }
    // 把index数据写到文件里
    WriteBlock(&r->index_block, &index_block_handle);
  }

  // Write footer
  if (ok()) {
    Footer footer;
    footer.set_metaindex_handle(metaindex_block_handle);
    footer.set_index_handle(index_block_handle);
    std::string footer_encoding;
    // 生成footer
    footer.EncodeTo(&footer_encoding);
    r->status = r->file->Append(footer_encoding);
    if (r->status.ok()) {
      r->offset += footer_encoding.size();
    }
  }
  return r->status;
}
```

通过这一篇的学习, 我们了解了sst文件的结构, 还有最主要的三个方法的详细实现. 至于这三个方法在哪里被调用, 我们在后面的章节再讲, 因为这是更上层的逻辑. 