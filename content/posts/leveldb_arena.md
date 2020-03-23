---
title: 深入浅出 LevelDB (2) - Arena 内存池
author: "Atwood"
tags: ["LevelDB", "Databases", "C++", "数据引擎"]
categories: ["Release"]
date: 2020-03-21
---

## Arena 内存池
内存池的存在主要就是减少`malloc`或者`new`调用的次数, 较少内存分配所带来的系统开销, 提升性能. 

LevelDB里自己实现了一个内存池, 它的做法是, 先向系统申请一块大的内存, 程序里需要申请内存时, 先把已有的内存块分配给用户, 如果不够用则再申请一块大的内存. 当内存池对象析构时, 分配的内存均被释放, 保证了内存不会泄漏. 

Arena以block为单位来管理内存, 每个block的大小为4096KB. 


先来看Arena类的结构

![leveldb_arena](/images/leveldb_arena/arena_uml.png)

再来看具体的内部实现

#### Allocate
```c++
inline char* Arena::Allocate(size_t bytes) {
  // The semantics of what to return are a bit messy if we allow
  // 0-byte allocations, so we disallow them here (we don't need
  // them for our internal use). 最少也得申请一个byte
  assert(bytes > 0);
  // 如果需要的内存小于剩余内存, 直接分配, 并且返回指针
  if (bytes <= alloc_bytes_remaining_) {
    // 先保存当前指针偏移量, 用来返回
    char* result = alloc_ptr_;
    // 增加指针偏移量, 为下次分配的起始点
    alloc_ptr_ += bytes;
    // 剩余内存减少
    alloc_bytes_remaining_ -= bytes;
    return result;
  }
  return AllocateFallback(bytes);
}
```

可以看到, 如果当前block的剩余内存大于需要分配的内存, 直接分配即可, 如果当前block大小不够, 则进入`AllocateFallback`方法

#### AllocateFallback 
```c++
char* Arena::AllocateFallback(size_t bytes) {
  // 如果需要的内存大于1/4个block大小, 即1024kb, 直接单独分配这一块的内存给它, 当前block现有的剩余内存不变. 比如之前Arena内存中还剩500kb, 现在请求2000kb, 则重新用new申请2000kb的内存, 保存内存的指针, 然后返回. 然后Arena剩余还是500kb, 下一次有请求申请内存, 还是会首先尝试在500kb中分配
  if (bytes > kBlockSize / 4) {
    // Object is more than a quarter of our block size.  Allocate it separately
    // to avoid wasting too much space in leftover bytes.
    char* result = AllocateNewBlock(bytes);
    return result;
  }

  // We waste the remaining space in the current block.
  // 如果需要的内存小于1024kb, 意味着现在Arena中剩余的内存已经不足1024kb, 否则根本不会进入这个方法. 那么直接丢弃这1024kb的内存,
  // 重新new一块block, 即4096kb大小的内存, 然后直接使用4096kb那块内存分配. 虽然浪费了之前的那一块内存, 但是我理解目的是为了减少new的次数.
  // 既然剩余内存已经比较小了, 所以干脆重新申请一块大一点的, 这样大概率接下来的请求都可以直接返回, 不用再进行new操作
  alloc_ptr_ = AllocateNewBlock(kBlockSize);
  alloc_bytes_remaining_ = kBlockSize;

  char* result = alloc_ptr_;
  alloc_ptr_ += bytes;
  alloc_bytes_remaining_ -= bytes;
  return result;
}
```

#### AllocateNewBlock
上面的方法中用到了`AllocateNewBlock`方法, 这个其实就是实际去new一块内存出来的代码
```c++
// 用new去申请新内存
char* Arena::AllocateNewBlock(size_t block_bytes) {
  char* result = new char[block_bytes];
  // 记录这个指针, 用于销毁的时候用
  blocks_.push_back(result);
  // 增加一共使用的内存大小, 包括指针的大小, 用于记录
  memory_usage_.fetch_add(block_bytes + sizeof(char*),
                          std::memory_order_relaxed);
  return result;
}
```

#### AllocateAligned
这个方法和`Allocate`方法很像, 只是为了字节对齐, 可能申请n大小的内存, 实际需要大于n的内存. 主要的区别在于求出具体需要的内存大小, 实际内存分配的逻辑和`Allocate`一样
```c++

char* Arena::AllocateAligned(size_t bytes) {
  const int align = (sizeof(void*) > 8) ? sizeof(void*) : 8;
  static_assert((align & (align - 1)) == 0,
                "Pointer size should be a power of 2");
  size_t current_mod = reinterpret_cast<uintptr_t>(alloc_ptr_) & (align - 1);
  size_t slop = (current_mod == 0 ? 0 : align - current_mod);
  // needed算出实际需要分配的内存大小, 其余的和普通Allocate方法一样
  size_t needed = bytes + slop;
  char* result;
  if (needed <= alloc_bytes_remaining_) {
    result = alloc_ptr_ + slop;
    alloc_ptr_ += needed;
    alloc_bytes_remaining_ -= needed;
  } else {
    // AllocateFallback always returned aligned memory
    result = AllocateFallback(bytes);
  }
  assert((reinterpret_cast<uintptr_t>(result) & (align - 1)) == 0);
  return result;
}
```

本篇内容的详细源码可看[leveldb/util/arena.h](https://github.com/google/leveldb/blob/master/util/arena.h) 和 [leveldb/util/arena.cc](https://github.com/google/leveldb/blob/master/util/arena.cc)