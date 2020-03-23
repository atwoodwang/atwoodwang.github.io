---
title: 深入浅出 LevelDB (5) - Memtable 内存表
author: "Atwood"
tags: ["LevelDB", "Databases", "C++", "数据引擎"]
categories: ["Release"]
date: 2020-03-23
---

## Memtable
Memtable是LevelDB的一个重要组成部分, 我们再回顾一篇LevelDB的架构. 

![avatar](/images/leveldb_intro/structure.jpg)

在用户写入数据时, 首先会写入WAL文件(这个后面再讲), 然后就会写入到内存中的memtable里, 当memtable数据达到一定大小时, 会变为immutable的memtable, immutable的memtable只可读不可写, 同时LevelDB会开启一个新的memtable接受新的写入. Immutable的memtable会被后台线程异步的刷新到磁盘上, 变成SST文件, 这就是LevelDB数据持久化的步骤.

Memtable其实是对skiplist的一层封装, 因为用户的数据在内存中是保存在skiplist里的, 在我们对memtable进行读写时, 底层操作的实际上就是skiplist.

## Memtable结构
现在我们来看看Memtable的结构:

![memtable](/images/leveldb_memtable/memtable_uml.png)

不难看出Memtable里最核心的方法就是`Add`和`Get`, 分别是往内存中写数据和读数据. 但是读者可能会感觉一脸懵逼, 为什么`Add`方法里的参数看起来这么奇怪? `SequenceNumber`, `ValueType`, `Slice`都是什么? 别急, 我会一个一个解释.

- SequenceNumber: 在LevelDB中, 每个键值对都是有一个版本的. 在我们更新某一个key的值时, 并不会直接覆盖掉之前的值, 而是为这个key创建一个新的版本. 这样设计的理由是因为LevelDB支持snapshot, 所以需要保留旧的键值信息. 所以在插入和读取数据时, 都需要提供一个版本号, 这个版本号的生成会在以后更上层的逻辑里面讲
- ValueType: 这是一个enum, 对应着一个字节. 如果为0, 代表这一次记录是删除操作, 如果为1, 代表这一条记录为插入或者更新.
- Slice: 这个其实就是LevelDB中自己实现了一个string的类, 提供了一些有用的方法. 在阅读源码时可以把它就当做一般的string来看待. 比如key和value都是string类型.

现在我们知道了插入需要提供的参数有版本号, 类型, 键还有值, 那么我们就可以理解具体在memtable里, 用户传入的键值对是怎么存储的了. 为了防止混淆, 以后我们把用户在读写操作时传入的key叫做`user key`, 传入的值叫做`user value`. 之前在看skiplist源码的时候我就有一个疑问, 为什么skiplist里插入的时候只需要传入key呢, 数据库存的因该是键值对, 如果skiplist只存了`user key`, 那么`user value`存哪里呢? 后来看了memtable才知道, 原来将数据存入skiplist时, 传入的key并不是用户在进行写操作时传入的user key, 而是已经把user key, user value, sequence number 等等信息打包好编码好以后生成的一个复合key. 下面就是这个key的结构:

![memtable](/images/leveldb_memtable/key_structure.png)

这里面可以看到, 当用户进行写操作时, user key, sequence number和type会组成为一个internal key, 再加上这个internal key进行varint32编码后长度, 会组成为一个memtable key. 同样, user value和它的长度, 会组成为一个memtable value. memtable key和value一起, 才组成了真正被插入到skiplist中的key的结构. 

有了上面的铺垫, 我们就可以更容易理解接下来的源码了

## Memtable 源码

#### Add
```c++
// Memtable插入操作
void MemTable::Add(SequenceNumber s, ValueType type, const Slice& key,
                   const Slice& value) {
  // Format of an entry is concatenation of:
  //  key_size     : varint32 of internal_key.size()
  //  key bytes    : char[internal_key.size()]
  //  value_size   : varint32 of value.size()
  //  value bytes  : char[value.size()]
  size_t key_size = key.size();
  size_t val_size = value.size();
  
  // internal key = userKey +  sequenceNumber(7 Bytes) + type (insert or delete, 1 Byte)
  size_t internal_key_size = key_size + 8;

  // 所以下面求出来的是整个entry的大小, 也就是上面图中的skiplist key的大小. 
  const size_t encoded_len = VarintLength(internal_key_size) +
                             internal_key_size + VarintLength(val_size) +
                             val_size;
  // 申请内存
  char* buf = arena_.Allocate(encoded_len);
  // 把internal_key_size编码后存到buf中
  char* p = EncodeVarint32(buf, internal_key_size);
  // 接着internal_key_size后面保存user key的内容
  memcpy(p, key.data(), key_size);
  p += key_size;
  // 保存sequence number 和 type的值
  EncodeFixed64(p, (s << 8) | type);
  p += 8;
  // 把user value的长度编码后保存
  p = EncodeVarint32(p, val_size);
  // 保存user value的值
  memcpy(p, value.data(), val_size);
  assert(p + val_size == buf + encoded_len);
  // 把整个entry(key+value)插入到skiplist中, 这里调用的就是之间介绍过的skiplist的insert方法.
  table_.Insert(buf);
}
```

我看到这里的时候, 又有一个疑问, 既然skiplist的insert方法里传入的字符串是memtable_key + memtable_value, 那skiplist里面具体是怎么样来排序的呢? 为什么能保证是按user key的顺序来排序, 而且对于相同的key, 大的sequence number总在后面呢? 

这里就是比较器起到的作用了, 仔细看上面Memtable的结构图里, 我们会发现它有一个私有成员: `comparator_`. 它会被用来生成skiplist, 并在skiplist对不同key比大小的时候使用到. 用户可以传入自定义的comparator, 这里我们看看默认的实现:
 
在skiplist里面排序, 是需要按照internalkey的大小排序的, 所以在comparator里面, 需要先从skiplist key里面解析出internal_key再比较.

```c++
int MemTable::KeyComparator::operator()(const char* aptr,
                                        const char* bptr) const {
  // Internal keys are encoded as length-prefixed strings.
  Slice a = GetLengthPrefixedSlice(aptr);
  Slice b = GetLengthPrefixedSlice(bptr);
  return comparator.Compare(a, b);
}
```

`GetLengthPrefixedSlice`方法的目的就是从skiplist key里面获取到internal_key的值. 因为skiplist的头几个字节就是internal_key的长度, 通过GetVarint32Ptr, 可以解析出来internal_key的长度为len, 而且p指针已经移动到internal_key开始的地方, 那么直接用p和len获取internal_key就好了.

```c++
static Slice GetLengthPrefixedSlice(const char* data) {
  uint32_t len;
  const char* p = data;
  p = GetVarint32Ptr(p, p + 5, &len);  // +5: we assume "p" is not corrupted
  return Slice(p, len);
}
```

通过上面的讲解, 我们已经了解了Add方法的实现, 在看Get方法的实现之前, 我们还需要看一下Get方法的参数. 

`bool Get(const LookupKey& key, std::string* value, Status* s);`

这里面有一个LookupKey, 其实就是对user key和sequence number做了一层封装. 里面有几个实用的方法.

![memtable](/images/leveldb_memtable/lookupkey_uml.png)

里面的几个方法其实看名字就知道是什么意思了, 这里我们只看看最重要的constructor

```c++
// LookupKey = | Size (int32变长) | User key (string) | sequence number (7 bytes) | value type (1 byte) |
LookupKey::LookupKey(const Slice& user_key, SequenceNumber s) {
  size_t usize = user_key.size();
  // 为什么加13？因为8是SequenceNumber和 value type的长度，5是Size (int32变长)的最大长度
  size_t needed = usize + 13;  // A conservative estimate
  char* dst;
  // 默认最小使用大小为200的char数组来存LookupKey
  if (needed <= sizeof(space_)) {
    dst = space_;
  } else {
    dst = new char[needed];
  }
  // LookupKey的起始位置
  start_ = dst;
  // 把internalkey的长度转换成varint32并填充在size的位置
  dst = EncodeVarint32(dst, usize + 8);
  // size后面就是userkey的起始位置了, kstart_是user_key的起始位置
  kstart_ = dst;
  // 拷贝user_key
  memcpy(dst, user_key.data(), usize);
  dst += usize;
  // 转换sequence number和 type
  EncodeFixed64(dst, PackSequenceAndType(s, kValueTypeForSeek));
  dst += 8;
  end_ = dst;
}
``` 

#### Get
了解了LookupKey, 接下来我们来看看Memtable的Get方法

```c++
// Memtable读取操作
bool MemTable::Get(const LookupKey& key, std::string* value, Status* s) {
  Slice memkey = key.memtable_key();
  Table::Iterator iter(&table_);
  // 之前讲解skiplist的iterator的时候有讲过, seek会把遍历器的指针移动到移动到比提供的key大的最近的节点上.
  iter.Seek(memkey.data());
  if (iter.Valid()) {
    // entry format is:
    //    klength  varint32
    //    userkey  char[klength]
    //    tag      uint64
    //    vlength  varint32
    //    value    char[vlength]
    // Check that it belongs to same user key.  We do not check the
    // sequence number since the Seek() call above should have skipped
    // all entries with overly large sequence numbers.
    // entry是memtable_key加memtable_value, 因为skiplist里面排序是按照internal_key来排序的, 所以这里已经确保了, 获取到的entry里面,
    // 即使user_key一样, sequence也是更高的, 意味着更新的修改.
    const char* entry = iter.key();
    // internal_key_length
    uint32_t key_length;
    const char* key_ptr = GetVarint32Ptr(entry, entry + 5, &key_length);
    // 确保获取到的internal_key里的user_key和传入的lookup_key里的user_key相等
    if (comparator_.comparator.user_comparator()->Compare(
            Slice(key_ptr, key_length - 8), key.user_key()) == 0) {
      // Correct user key
      // tag = sequence number + type
      const uint64_t tag = DecodeFixed64(key_ptr + key_length - 8);
      switch (static_cast<ValueType>(tag & 0xff)) {
        case kTypeValue: {
          // 说明这个entry记录的是插入操作, 可以读取里面的value. key_ptr指向的是internal_key的起始点, 所以key_ptr + key_length
          // 就是memtable_value的起始点了, 然后用GetLengthPrefixedSlice就可以获取到value的值
          Slice v = GetLengthPrefixedSlice(key_ptr + key_length);
          value->assign(v.data(), v.size());
          return true;
        }
        case kTypeDeletion:
          // 如果是删除, 说明key已经被删除了, 就返回NotFound
          *s = Status::NotFound(Slice());
          return true;
      }
    }
  }
  // false表示遍历器不合法, 我理解是skiplist为空或者其他未知错误的时候
  return false;
}
```

