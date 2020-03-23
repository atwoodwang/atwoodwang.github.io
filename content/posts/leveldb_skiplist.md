---
title: 深入浅出 LevelDB (3) - SkipList 跳跃链表
author: "Atwood"
tags: ["LevelDB", "Databases", "C++", "数据引擎"]
categories: ["Release"]
date: 2020-03-22
---

## Skiplist 介绍

上一篇我们提到, LevelDB中的数据是按照key的值进行有序存储的. LevelDB在内存中存储数据的区域称为memtable, 这个memtable底层是用跳跃链表skiplist来实现的. redis内部也采用跳跃链表来实现有序链表. 

Skiplist的效率可以和AVL树, 红黑树媲美, 平均O(logN), 最坏O(N)的查询效率, 但是用Skiplist实现比平衡树实现简单许多, 所以很多程序用跳跃链表来代替平衡树.

Skiplist简单来说就是一个多层的单向链表, 是对一般我们所说的单向链表的补充. 

![skiplist](/images/leveldb_skiplist/linkedlist.png)

这是一个单向链表, 如果我们想查找一个元素的话, 只能从头找到尾, 需要整个遍历一次, 时间为O(N)

如果是说链表是排序的, 并且节点中还存储了指向前面第二个节点的指针的话, 那么在查找一个节点时, 仅仅需要遍历N/2个节点即可. 这就是跳跃链表的核心, 利用不同节点间跳跃的指针, 来优化搜索的时间. 

![skiplist](/images/leveldb_skiplist/skiplist_sample.png)

比如这个例子, 如果我们想搜索7, 比起单链表的从3到4到7, 在跳跃链表中, 我们从最高(最稀疏)层开始向下寻找, 直接可以通过4这个节点找到7. 如果我们想找寻14, 则通过12节点, 两步就可以搜索到了.

![skiplist](/images/leveldb_skiplist/skiplist_search.png)

对于一个跳跃链表, 最下层就相当于一个普通的单向链表, 所有节点依次相连, 随着层数的增加, 每层里面的节点数逐渐减少. 搜索的时候, 从最高层开始寻找. 因为节点是有序的, 所以在搜索中一次就可以跨越多个无效节点, 大大节省时间. 

## 源码分析

下面我们进入万众期待的源码分析环节!! 撒花~~

### Node
Skiplist数据结构在源码的skiplist.h文件里, 本身是一个类. 在看skiplist类本身之前, 我们先看看skiplist里面节点的定义.

Node结构里面主要有两个成员:
1. key: 这个是skiplist节点的Key, 这个名字可能会比较让人迷惑, 因为LevelDB是一个键值数据库, 用户的插入操作会包含key和value, 第一次看这个部分的时候, 我一直在想node里面如果只存了key, 那用户传入的value存在哪里呢? 后来看了后面的代码才发现, Node的里面这个key, 和我们理解的用户插入的key, 是不一样的. 他实际上是包含了用户插入的键和值一起. 这里面具体的编码格式我们后面再说.
2. std::atomic<Node*> next_[1]: 这是一个数组, 里面保存着当前节点下一点的指针. 之所以是一个数组, 是因为skiplist是多层的, 所以一个节点可以有多个下一节点. 比如上面图中的4节点, 它就是2层的, 所以它就有7和12两个下一节点. 所以对于4这个node, next_[0] = 7, next_[1] = 12. 这里把数组的长度设成1, 其实是一个很巧妙的设计, 因为每个node的层数不固定, 所以为了节省内存, 先假定只有一层, 然后在实际创建node的时候根据实际node的层数再分配实际所需的内存

下面是详细代码, 里面包含了注释.

```c++
// Implementation details follow
template <typename Key, class Comparator>
struct SkipList<Key, Comparator>::Node {
  explicit Node(const Key& k) : key(k) {}

  Key const key;

  // Accessors/mutators for links.  Wrapped in methods so we can
  // add the appropriate barriers as necessary.
  // 获取当前节点第n层的下一个节点
  Node* Next(int n) {
    assert(n >= 0);
    // Use an 'acquire load' so that we observe a fully initialized
    // version of the returned Node.
    // 这个会确保获取到的node是已经完全初始化好的
    return next_[n].load(std::memory_order_acquire);
  }

  // 设置当前节点的下一个节点
  void SetNext(int n, Node* x) {
    assert(n >= 0);
    // Use a 'release store' so that anybody who reads through this
    // pointer observes a fully initialized version of the inserted node.

    // 确保任何读这个指针的都能看到完全初始化好的node
    next_[n].store(x, std::memory_order_release);
  }

  // No-barrier variants that can be safely used in a few locations.
  // 不加屏障的查找下一个node
  Node* NoBarrier_Next(int n) {
    assert(n >= 0);
    return next_[n].load(std::memory_order_relaxed);
  }

  // 不加屏障的设置下一个node
  void NoBarrier_SetNext(int n, Node* x) {
    assert(n >= 0);
    next_[n].store(x, std::memory_order_relaxed);
  }

 private:
  // Array of length equal to the node height.  next_[0] is lowest level link.
  // 一个数组, 里面是这个节点的下一个节点, 如果这个节点的高度是5, 则它最多可能有5个下一节点, 每层都可能有一个
  // 这里很巧妙, 因为每个node的层数不固定, 所以为了节省内存,先假定只有一层, 然后在NewNode方法里根据实际node的层数再分配实际所需的内存
  std::atomic<Node*> next_[1]; //定义下一个节点
};
```

### Skiplist
再来看Skiplist类, 首先skiplist的结构是这样的:

![skiplist](/images/leveldb_skiplist/skiplist_uml.png)

来看几个最重要的方法的实现

#### Constructor
```c++
template <typename Key, class Comparator>
SkipList<Key, Comparator>::SkipList(Comparator cmp, Arena* arena)
    : compare_(cmp),
      arena_(arena),
      head_(NewNode(0 /* any key will do */, kMaxHeight)),
      max_height_(1),
      rnd_(0xdeadbeef) {
  // 初始化, 把head设成最大高度, 然后把每一层的下一个都先设为null
  for (int i = 0; i < kMaxHeight; i++) {
    head_->SetNext(i, nullptr);
  }
}
```

#### NewNode
之前说过Node类里默认只假定了层数为一层, 所以在实际生成node的时候, 根据高度分配实际的内存. 
```c++
template <typename Key, class Comparator>
typename SkipList<Key, Comparator>::Node* SkipList<Key, Comparator>::NewNode(
    const Key& key, int height) {
  // Node类里面默认假定的是只有一层, 所以需要根据实际的层数height分配更多的内存, 大小为Node类本来的大小, 加上高度-1个node指针的大小.
  char* const node_memory = arena_->AllocateAligned(
      sizeof(Node) + sizeof(std::atomic<Node*>) * (height - 1));
  return new (node_memory) Node(key);
}
```

#### Insert
```c++
template <typename Key, class Comparator>
void SkipList<Key, Comparator>::Insert(const Key& key) {
  // TODO(opt): We can use a barrier-free variant of FindGreaterOrEqual()
  // here since Insert() is externally synchronized.
  Node* prev[kMaxHeight]; //新插入的key, 在每一层的前一个节点，因为这个新节点的高度还未知，所以先长度设为最大值12
  Node* x = FindGreaterOrEqual(key, prev); //查找现在跳表中比key大的最小的node, 并且填充prev

  // Our data structure does not allow duplicate insertion
  assert(x == nullptr || !Equal(key, x->key));

  // 生成随机的高度
  int height = RandomHeight();
  // 如果新节点的高度已经超过现在当前的最高的高度了, 那么需要把新增加的高度的prev设为head
  if (height > GetMaxHeight()) {
    for (int i = GetMaxHeight(); i < height; i++) {
      prev[i] = head_;
    }
    // It is ok to mutate max_height_ without any synchronization
    // with concurrent readers.  A concurrent reader that observes
    // the new value of max_height_ will see either the old value of
    // new level pointers from head_ (nullptr), or a new value set in
    // the loop below.  In the former case the reader will
    // immediately drop to the next level since nullptr sorts after all
    // keys.  In the latter case the reader will use the new node.
    // 记录最新的层数
    max_height_.store(height, std::memory_order_relaxed);
  }

    // 根据插入的key生成新的节点
      x = NewNode(key, height);
  for (int i = 0; i < height; i++) {
    // NoBarrier_SetNext() suffices since we will add a barrier when
    // we publish a pointer to "x" in prev[i].
    //
    // 就相当于给linkedlist里面插入节点, 比如a->b, 要在a和b中插入c, 则:
    // 1. c.next = a.next
    // 2. a.next = c
    x->NoBarrier_SetNext(i, prev[i]->NoBarrier_Next(i));
    prev[i]->SetNext(i, x);
  }
}
```

#### FindGreaterOrEqual
```c++
template <typename Key, class Comparator>
typename SkipList<Key, Comparator>::Node*
SkipList<Key, Comparator>::FindGreaterOrEqual(const Key& key,
                                              Node** prev) const {
  Node* x = head_;
  // 总共可能要寻找的层数, 如果当前最大的height为n, 则需要从0遍历到n-1
  int level = GetMaxHeight() - 1;
  while (true) {
    // 获取x在第level层的下一个节点.
    Node* next = x->Next(level);
    if (KeyIsAfterNode(key, next)) {
      // Keep searching in this list
      // 如果key比next要大, 则应该放在next后面, 则把x设成next, 继续遍历
      // 相当于双指针在链表里移动.
      // 比如 x(key=3), next(key=6), key = 9. 要插入的key比next里面的key更大, 所以指针向右移
      x = next;
    } else {
      // 在插入操作调用这个函数的时候, prev永远不会为空, 但是在查找操作时, 因为不需要prev的信息, 会传null进来
      // 既然上面的if是false, 说明在这一层, key已经比x里面的key要大, 但是比next里面的key要小了, 所以要插入的key, 前一个节点就应该
      // 是x.
      if (prev != nullptr) prev[level] = x;
      if (level == 0) {
        // 如果已经到第0层了, 则直接返回next的值, 而且next里的key一定是比参数里的key要大的最小的key.
        return next;
      } else {
        // Switch to next list
        // 继续遍历下一层
        level--;
      }
    }
  }
}
```

#### Contains
```c++
// 判断跳表是否含有某个key. 因为FindGreaterOrEqual会返回key值大于等于参数里key的node, 所以如果返回空, 肯定就不存在.
// 如果返回不为空, 则可能相等可能返回值里的key更大, 所以再比较一下即可.
template <typename Key, class Comparator>
bool SkipList<Key, Comparator>::Contains(const Key& key) const {
  Node* x = FindGreaterOrEqual(key, nullptr);
  if (x != nullptr && Equal(key, x->key)) {
    return true;
  } else {
    return false;
  }
}
```

#### RandomHeight
在需要新创建一个节点的时候, 我们需要确定这个节点的高度. 这个随机的高度并不是在1-12间以相同的概率选择一个, 而是越高的层数越少出现, 类似于金字塔的形状. 

```c++
template <typename Key, class Comparator>
int SkipList<Key, Comparator>::RandomHeight() {
  // Increase height with probability 1 in kBranching
  static const unsigned int kBranching = 4;
  int height = 1;

  // 4分之一的几率, 高度会加一, 最高是12
  while (height < kMaxHeight && ((rnd_.Next() % kBranching) == 0)) {
    height++;
  }

  // 确保最后选定的层数肯定大于0, 又小于等于kMaxHeight, kMaxHeight默认为12.
  assert(height > 0);
  assert(height <= kMaxHeight);
  return height;
}
```

### skiplist的iterator
Skiplist类中还定义了一个iterator, 用来遍历或者查找某一个key. 因为上面介绍的`FindGreaterOrEqual`之类的方法都是private的, 外部无法直接访问. 在读取一个key的值的时候, 就是通过iterator来和skiplist做交互的.

首先来看Iterator类的结构

![skiplist_iterator](/images/leveldb_skiplist/skiplist_iterator_uml.png)
 
再来看看主要的方法实现, 其实都很简单, 一目了然, 就是把skiplist类里面的一些私有方法做了封装.

#### key
```c++
template <typename Key, class Comparator>
inline const Key& SkipList<Key, Comparator>::Iterator::key() const {
  assert(Valid());
  return node_->key;
}
```

#### Next
这里可以看出, next返回的是离当前遍历器里存储的node最近的节点, 也就是返回的第0层的下一节点. 因为第0层实际上就是一个单向链表, skiplist里所有的元素都存在于第0层.
```c++
template <typename Key, class Comparator>
inline void SkipList<Key, Comparator>::Iterator::Next() {
  assert(Valid());
  node_ = node_->Next(0);
}
```

#### Seek
实际上就是找到比target大的最近的节点.
```c++
template <typename Key, class Comparator>
inline void SkipList<Key, Comparator>::Iterator::Seek(const Key& target) {
  node_ = list_->FindGreaterOrEqual(target, nullptr);
}
```

#### Prev
就是找到比当前node小的最近的节点, 并把指针移动到那个节点
```c++
template <typename Key, class Comparator>
inline void SkipList<Key, Comparator>::Iterator::Prev() {
  // Instead of using explicit "prev" links, we just search for the
  // last node that falls before key.
  assert(Valid());
  node_ = list_->FindLessThan(node_->key);
  if (node_ == list_->head_) {
    node_ = nullptr;
  }
}
```

相信大家已经对Skiplist的实现有了一个比较清晰的认识, 下一节我们会开始介绍Memtable, 其实现实际上就是对Skiplist的进一步封装.