---
title: 深入浅出 LevelDB (4) - Varint 变长编码
author: "Atwood"
tags: ["LevelDB", "Databases", "C++", "数据引擎"]
categories: ["Release"]
date: 2020-03-22
---

## Varint 编码
在深入到memtable之前, 我们还需要了解一个在LevelDB中广泛使用的一个编码格式: Varint (variable integer). 它是一种使用一个或多个字节序列化整数的方法, 会把整数编码为变长字节. 通常一个整数需要占用32个bit, 即4个字节. 如果我们只需要存储很小的数的时候, 用4个字节存储是很浪费的. 通过Varint编码, 小的数字只需要1个byte就可以保存, 大的数字使用5个bytes. 虽然极端条件下overhead有点大, 需要5个字节, 但是实际场景下小数字的使用率远远多于大数字, 所以用Varint在大部分的情况下都可以起到很好的压缩效果.

## Varint 原理
 
Varint编码中, 除了最后一个字节外, 每个字节都设置了最高有效位（most significant bit - msb). msb为1则表明后面的字节还是属于当前数据的, 如果是0那么这是当前数据的最后一个字节数据. 每个字节的低7位用于以7位为一组存储数字的二进制补码表示, 最低有效组在前, 或者叫最低有效字节在前, 因为varint编码后数据的字节是按照小端序(Little Endian)排列的.

我们来通过解码一个varint的数据来熟悉varint编码的原理.

比如整数1, 在varint编码后只需要一个字节, 为:

`0000 0001`

第一位为0, 表示这个字节已经是当前数据的最后一个字节了, 后面7位二进制位000 0001, 代表1. 所以这个数为1.

再举一个例子, 300在varint编码后需要2个字节, 为:

`1010 1100 0000 0010`

先看第一个字节, 最高位为1, 代表后面一个字节也是当前数据的一部分. 然后看第二个字节, 最高位为0, 代表这个字节是当前数据的最后一个字节. 所以我们知道, 这个数据一共占用了两个字节. 然后把每个字节的最高位去掉, 分别为`010 1100`和`000 0010`. 因为存储是用小端存储的, 所以在解码时, 最低有效组应该在前面, 所以两个字节拼在一起是`000 0010 010 1100`, 即 `100101100`. 这就是300的二进制表示.

另外在LevelDB中, 所有的数字都是以字符数组的形式存储的.

## Varint 相关源码
在LevelDB中, 有一个文件里保存着所有处理编码和解码的代码. 里面具体编码和解码varint的源码这里不再详述, 因为原理上面解释的比较清楚了, 这里只看几个在后面用的特别多的和解码相关的方法.

#### GetVarint32Ptr
这个方法主要用于解析后面会讲到的memtable_key或者memtable_value的长度, 因为这些字段格式都是长度+具体数据, 其中长度是个varint, 要想取出来具体数据, 得解析出来长度是多少. 这个方法, 会从p指针开始解析长度, 解析到limit. limit一般等于p+5, 因为变长varint最多占用5个字节. 然后把解析出来的长度赋值在value里, 把p指针移动到真正数据开始的地方, 并且返回p.
```c++
inline const char* GetVarint32Ptr(const char* p, const char* limit,
                                  uint32_t* value) {
  if (p < limit) {
    uint32_t result = *(reinterpret_cast<const uint8_t*>(p));
    if ((result & 128) == 0) {
      *value = result;
      return p + 1;
    }
  }
  return GetVarint32PtrFallback(p, limit, value); // 具体解析的逻辑, 这里不详述
}
```

#### VarintLength
求出来v在Varint格式下需要占用的字节数, 比如0-127是1
```c++
int VarintLength(uint64_t v) {
  int len = 1;
  while (v >= 128) {
    v >>= 7;
    len++;
  }
  return len;
}
```

本篇内容的详细源码可看[leveldb/util/coding.h](https://github.com/google/leveldb/blob/master/util/coding.h) 和 [leveldb/util/coding.cc](https://github.com/google/leveldb/blob/master/util/coding.cc)
