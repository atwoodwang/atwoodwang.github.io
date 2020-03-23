---
title: 深入浅出 LevelDB (1) - 初识LevelDB  
author: "Atwood"
tags: ["LevelDB", "Databases", "C++", "数据引擎"]
categories: ["Release"]
date: 2020-03-20
---
![level_db_icon](/images/leveldb_intro/leveldb_icon.png)

## Intro
根据[LevelDB](https://github.com/google/leveldb)官方介绍:
>LevelDB is a fast key-value storage library written at Google that provides an ordered mapping from string keys to string values.

LevelDB是一个可持久化的KV数据库引擎，是大名鼎鼎的BigTable论文中描述的键值存储系统的单机版的实现, 它提供了一个极其高速的键值存储系统，并且由 Bigtable 的作者, Google传奇工程师Jeff Dean和Sanjay Ghemawat开发并开源。

主要特点如下:
1. Keys and values are arbitrary byte arrays. 键和值都是任意长度的字节数组
2. Data is stored sorted by key. 内部存储是键值对的形式, 默认是按照键值的字典序进行排序.
3. Callers can provide a custom comparison function to override the sort order. 用户可以提供自定义的排序方式
4. The basic operations are Put(key,value), Get(key), Delete(key). 提供基本操作接口, put(), get(), delete()
5. Multiple changes can be made in one atomic batch. 支持批量的操作以原子的方式进行
6. Users can create a transient snapshot to get a consistent view of data. 可以创建全量数据的快照
7. Forward and backward iteration is supported over the data. 可以支持对数据的正向和反向的遍历.
8. Data is automatically compressed using the Snappy compression library. 自动使用snappy压缩数据
9. External activity (file system operations etc.) is relayed through a virtual interface so users can customize the operating system interactions. 可移植性

当然, LevelDB也有一些限制:
1. This is not a SQL database. It does not have a relational data model, it does not support SQL queries, and it has no support for indexes. 非关系型数据模型（NoSQL），不支持sql语句，也不支持自己创建索引；
2. Only a single process (possibly multi-threaded) can access a particular database at a time. 一次只允许一个进程访问一个特定的数据库；
3. There is no client-server support builtin to the library. An application that needs such support will have to wrap their own server around the library. 没有内置的C/S架构，但开发者可以使用LevelDB库自己封装一个server；

## LevelDB基础架构
![avatar](/images/leveldb_intro/structure.jpg)

上图简单展示了 LevelDB 的整体架构。LevelDB主要有几个组成部分:

1. MemTable：这是用来在内存中存储用户插入的key-value键值对的数据结构，具体的内部实现是 SkipList, 跳跃链表。里面的数据是根据key进行排序的. 用户对数据库的最新修改都会先保存在这里.
2. Immutable MemTable：当 MemTable 的大小达到设定的阈值时, 会变成 Immutable MemTable, 只接受读操作，不再接受写操作, 后续由后台线程 Flush 到磁盘上.
3. SST Files：这是LevelDB数据持久化的文件格式. 全称Sorted String Table File, 分为 Level0 到 LevelN 多层, 每一层包含多个 SST 文件, 文件内数据有序. Level0 直接由 Immutable Memtable Flush 得到, 其它每一层的数据由上一层进行 Compaction 得到.
4. Manifest Files：Manifest 文件中记录 SST 文件在不同 Level 的分布, 单个 SST 文件的最大、最小 key, 以及其他一些 LevelDB 需要的元信息. 由于 LevelDB 支持 snapshot, 需要维护多版本, 因此可能同时存在多个 Manifest 文件.
5. Current File：由于 Manifest 文件可能存在多个, Current 记录的是当前的 Manifest 文件名.
6. Log Files (WAL)：和其他数据库类似的WAL日志, 在插入操作时, 会先写入到log中, 以免在内存中数据还没来得及存储到磁盘上时出现意外情况导致数据丢失.

关于LevelDB的介绍和系统架构, 网络上已经有非常详细的介绍, 这里就不再赘述, 咱们从下一节开始, 直接深入LevelDB的具体源码, 看看这个大名鼎鼎的数据库是怎么实现的.