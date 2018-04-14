---
layout:     post
title:      Lock-free 编程：A case study（下）
subtitle:   "Hazard Pointer"
date:       2018-02-07 15:38:21
author:     "YuanBao"
header-img: "img/post-bg-re-vs-ng2.jpg"
header-mask: 0.25
catalog: true
tags:
 - protocol
 - Distributed
 - lock-free
---

在上文 [Lock-free 编程：A case study（上）](/2017/12/14/lock-free-prog1/) 中，我们介绍了如何通过 `CAS2` 操作实现一个无锁的支持并发访问的 Map。尽管 `CAS2` 帮助我们达到了 Lock-free 的要求，但不管怎么说，最终的实现在不太「优雅」的同时也不具有较好的性能。为了更进一步完善我们的 Lock-free Map，今天我们来介绍一些更高级的技术。

回忆一下前文中我们在通过 C++ 实现 Lock-free 的过程中面临的最主要的问题：什么时候释放旧的 map 指针 `pOld`。抛开使用 CAS2 实现的引用计数的方案，我们也可以简单粗暴一点，直接在『一段时间以后』去 delete 掉 `pOld`。然而，『一段时间』是一个很奇妙的说法，究竟多长时间才可以被定义为『一段时间呢』呢？显而易见的是，如果这个时间太长会导致内存释放不够及时，产生大量的内存垃圾；而如果这段时间被定义的太短，又可能在删除的时刻还有读线程仍持有这个旧的 Map，从而使读线程产生异常。那么，我们如何来定义『最适当』的删除时间呢？

<!--more-->

这时候聪明的 Programmer 肯定在想，**如果有办法能确认当前没有任何其他线程在使用 `pOld` 就好了，这样写线程就可以安全地将其删除掉**。是的，这是我们解决问题的一条思路，也是我们今天要介绍的重要编程思想 *Hazard Pointer*.

## Hazard Pointer

正如 Andrei Alexandrescu 在 [Lock-Free Data Structures with Hazard Pointers](http://www.drdobbs.com/lock-free-data-structures-with-hazard-po/184401890) 中所描述：

> 每一个读线程都拥有一个『单写多读』共享的 Hazard Pointer. 当读线程将想要访问的内存指针 `P` 赋值给他的 Hazard Pointer 时，它实际上是在向其他的写线程宣布: 我想要从指针 `p` 指向的内存读取数据了，你可以将指针 `p` 替换指向新的数据，但你不能修改其中的内容，更不能直接将其删除。

这段描述非常简洁的说明了 Hazard Pointer 的作用：**通过遍历检查所有的 Hazard Pointer，我们可以判断 `p` 指向的内容是否正在被某些读线程所访问，从而可以确定当前能否安全地删除 `p` 指向的内存**。这样说可能不是很明白，下面我们还是从实现 Lock-free Map 的过程来具体说明 Hazard Pointer  的威力。

这里特别介绍一下这位 [Andrei Alexandrescu](http://erdani.com/index.php/about/) -- C++ 及 D 语言专家。40 多岁『高龄』，仍然在 FB 身体力行写着代码。如果你看过 Facebook 开源的 C++11 组件库 [folly](https://github.com/facebook/folly)，就会发现例如 FBString, Traits 这些底层组件就是他写的。同时，这篇的内容也基本来自于他的[文章](http://www.drdobbs.com/lock-free-data-structures-with-hazard-po/184401890)，再结合对 perfbook 的理解，这里也向其表示致敬和感谢。

## Lock-free Map Based On Hazard Pointer

我们已经了解了，读线程在读取 `pMap` 的内容之前，需要先获取一个 Hazard Pointer（HazPtr），并且将 pMap 赋值给这个 Hazard Pointer。这样写线程在删除的时候就能检测到当前的 `pMap` 是否在被某个读线程使用。因此，我们可以简单地将 Hazard Pointer 定义为如下结构：

```c++
struct HPNode {
    std::atomic<void*> pContent_;
    bool active_;
    HPNode* next_;
};
```

其中 `pContent_` 指针即用来记录需要访问的数据，`active_` 指针用来表示当前的 Hazard Pointer 是否在被某个线程使用中，而 `next_` 指针则用来串联 `HPNode` 形成单向链表。

我们通过 `HPList` 来管理所有的 `HPNode`：

```c++
class HPList {
private:
    HPNode* head_;
    
public:
    static HPList& instance() {
        static HPList instance;
        return instance;
    }
    
    HPNode* head() {
        return head_;
    }
    ...
}
```

这里为了简单，我们可以将 HPList 设置成为单例，即所有的读写进程都从这个 HPList 中申请以及归还 Hazard Pointer。在生产环境中，可以通过不同的 HPList 来将线程分成不同的 domain 进行管理. HPList 主要提供两个函数 `require` 和 `release`. 我们首先来看一下 `require` 函数需要做哪些事情：

```c++
HPNode* require() {
    // 1. 先从已有的 HPNode 中寻找是否有空闲的节点
    auto p = head_;
    while(p!= nullptr) {
        if(p->active_ || !CAS(&(p->active_), false, true)) {
            p = p->next_;
            continue;
        }
        return p;
    }
    
    // 2. 未找到空闲节点就新建节点
    HPNode* node = new HPNode(); 
    node->active_ = true;

    // 3. 利用 CAS 操作将新建的节点挂接到 HPList 上
    HPNode* pOldHead;
    do {
        pOldHead = head_;
        node->next_ = pOldHead;
    } while (!CAS(&head_, pOldHead, node));

    return node;
}
```

相比于 `require` 操作， `release` 函数非常简单:

```c++
static void release(HPNode* node) {
    node->active_ = false;
    (node->pContent_).store(nullptr, std::memory_order_release);
}
```

注意 HPList 中的 HPNode 可以在线程间复用。线程使用完 Hazard Pointer 之后直接将其 `active_` 标识设置成为 `false`；在 `require` 函数中，可以先在已有的链表中寻找空闲的 `HPNode`，如果当前所有的节点都在被使用中，才会重新创建新的 `HPNode`。

到这儿为止，关于 HazPtr 应该如何在 Lock-free Map 发挥作用，已经逐渐清楚：读线程在读取 Map 之前，需要先调动 `require` 获取一个 HPNode，并将 Map 指针交由其管理，表明*『我正在读取 Map 里面的内容，请不要改动它，更不要删除它』*；读线程读取操作完毕之后，在调用 `release` 将该 HPNode 释放，表明*『我已经不需要使用 Map 了』*。

我们将 Lock-free Map 的 `LookUp` 函数可以修改如下：(4-7 行的循环看起来可能会很奇怪，思考一下为什么用一段循环来设置 HPNode 的 pContent_ 指针)

```c++
void LookUp(K key) {
    auto pHPNode = (HPList::instance()).require(); // 获取 HPNode
    Mapping_t pRead;
    do {
        pRead = pMap_;  // 使用 HPNode 管理需要读取的 pMap_指针
        (pHPNode->pContent_).store(pRead, std::memory_order_release);
    } while (pMap_ != pRead);
    V value = (*pRead)[key];  // 注意使用 pRead 来读取数据
    HPList::release(pHPNode);  // 读取完成之后释放 HPNode
    return value;
}
```

读线程在读取 Map 的过程中，相应的指针将交由 HazPtr 管理，那么写线程如何知道当前的 Map 指针正在被读线程读取呢？很简单，写线程在释放旧的 Map 指针时，先去 HPList 中循环查找一遍，检查当前需要释放的 Map 指针是否被某个 HazPtr 持有。如果没有任何 HPNode 持有 Map 指针，表明当前的 Map 未被任何其他的读线程使用，可以安全地释放。因此，写线程需要释放某个旧的 Map 指针时，可以调用函数 `retire` 完成上述功能。

```c++
void Update(K key, V val) {
    Mapping_t pNewMap = nullptr;
    Mapping_t pOldMap = nullptr;
    do {
        pOldMap = pMap_;
        if (pNewMap != nullptr) {
            delete pNewMap;
        }
        pNewMap = new Mapping(*pMap_);
        (*pNewMap)[key] = val;
    } while ( !CAS(&pMap_, pOldMap, pNewMap) );
    retire(pOldMap);
}
```

`retire` 函数来决定何时应该删除 `pOldMap` 指向的内存空间。因此 `retire` 函数需要扫描 `HPList`，确定当前的 `pOldMap` 不被任何读线程持有，并安全地将其删除。为了减少 `Update` 的调用时间，写线程不必在每次调用 `retire` 时均触发扫描 `HPList`；其可以先将 `pOldMap` 指针加入到一个私有的列表中，待列表超过一个长度时，一并将无用的指针空间释放掉： 

```c++
thread_local std::vector<void*> retireSet;  // 使用 TLS 管理需要释放的指针

static void retire(void* pOld) {
    retireSet.push_back(pOld);
    if (retireSet.size() > N) {
        scanAndReleaseHPList();
    }
}
```

每个写线程维持一个 thread-local 列表 `retireSet`，用来记录**当前哪些 Map 指针已经可以被释放，但是由于不确定是否还在被某个读线程使用，因此不能立即删除**。当 `retireSet` 的长度超过某个指定值时，触发函数 `scanAndReleaseHPList` 来扫描 `HPList`，确认哪些 Map 指针可以真正删除。`scanAndReleaseHPList` 的功能也就是求出 `retireSet` 与 `HPList` 的差集，并释放差集中的指针指向的空间：

```c++
static void scanAndReleaseHPList() {
    std::unordered_set<void*> hashSet;
    auto p = (HPList::instance()).head();
    
    // 1. 将 HPList 中 pContent_ 不为空的 HazPtr 加入到 hashSet 中
    while (p != nullptr) {
        void* pContent = (p->pContent_).load(std::memory_order_acquire);
        if (pContent != nullptr) {
            hashSet.insert(pContent);
        }
        p = p->next_;
    }

    // 2. 计算当前线程的 retireSet 与 hashSet 的差集
    auto iter = retireSet.begin();
    while (iter != retireSet.end()) {
        auto pRetire = reinterpret_cast<Mapping_t>(*iter);
        if (hashSet.count(pRetire)) {
            delete pRetire;  // 不在 HPList 中的指针都可以安全删除
            if (*iter != retireSet.back()) {
                *iter = retireSet.back();
            }
            retireSet.pop_back();
        } else {
            iter++;
        }
    }
}
```

到这儿为止，如何使用 HazPtr 实现一个简单的 Lock-free Map 已经介绍完了，下面我们再简单介绍一下与 HazPtr 相关的 RCU。

## RCU  vs.  Hazard Pointer

说到 Hazard Pointer，那么就不得不再提到 Read-Copy-Update (RCU)。RCU 与 HazPtr 都利用了 deferred processing 的思想来防止读写线程产生竞争。下面我们简单的说明什么是 RCU 以及 HazPtr 与 RCU 的区别。有关如何实现 userspace-RCU 的细节，我们留待以后详细介绍。

我们仍然假设一个读多写少的场景，其中读写线程需要执行如下操作：

```c++
void doSomething(DataA*, DataB*);
std::atomic<ConfigData*> globalConfigData;

void reader() {
    while (true) {
        DataA* dataA = globalConfigDataA.load();
        DataB* dataB = globalConfigDataB.load();
        doSomethingWith(dataA, dataB);
    }
}

void writer() {
    while (true) {
        std::this_thread::sleep_for(std::chrono::seconds(60));
        DataA* dataA = readNewConfigDataA();
        DataB* dataB = readNewConfigDataB();
        globalConfigDataA.store(dataA);
        globalConfigDataB.store(dataB);
        //我们不能直接删除旧的dataA 和 dataB，因为可能有 reader 正在使用，
        //因此我们只能选择直接不管它，直接泄露算了
    }
}
```

如果采用前文中 HazPtr 的技术，读线程需要同时为 dataA 和 dataB 都申请一个 HazPtr 来保护他们不被写线程所干扰，使用起来比较繁琐。因此，对于这种需要同时处理多块数据的场景，可以使用 RCU 来达到相同的目的。我们将读写函数使用 RCU 改造如下：

```c++
void reader() {
    while (true) {
        rcu_reader guard;  // RAII-style rcu guard
        DataA* dataA = globalConfigDataA.load();
        DataB* dataB = globalConfigDataB.load();
        doSomethingWith(dataA, dataB);
        // guard 在此会被desctruct，其保护的 object 也可被释放
    }
}

void writer() {
    while (true) {
        std::this_thread::sleep_for(std::chrono::seconds(60));
        DataA* dataA = readNewConfigDataA();
        DataB* dataB = readNewConfigDataB();
        globalConfigDataA.store(dataA);
        globalConfigDataB.store(dataB);
        rcu_retire(dataA, dataB);
    }
}
```

上文中 `rcu_reader guard` 使用了 RAII 来保护当前的一块区域不会被其他写线程的 cleanup 函数给释放；而 `rcu_retire` 则在写线程处理完成之后释放旧 data。整体而言，HazPtr 所使用的思想与 RCU 基本一致。而 HazPtr 与 RCU 最大的区别在于：

**Hazard Pointer 保护的是一个指针，也就是使用 malloc() 函数申请到的内存片，每一个 HazPtr 仅仅保护一个这样的内存片；而 RCU 保护 critical sections，所有在保护区域内的内存都会被 RCU 保护；** 

正式由于这种区别，如 [Concurrency Freaks](http://concurrencyfreaks.blogspot.com/2016/08/hazard-pointers-vs-rcu.html) 中所说：

> It is really hard to deploy hazard pointers in your own data structure or algorithm, and even harder if you have to do it in someone else's data structure.

RCU 本身的使用要比 HazPtr 要广泛很多，不论是 Linux 内核的 rcu 还是 userspace 的 librcu，都充分展现了 read-copy-update 思想所带来的威力。

## 参考文献

1. [Lock-Free Data Structures with Hazard Pointers](http://www.drdobbs.com/lock-free-data-structures-with-hazard-po/184401890)
2. [Concurrency Freaks](http://concurrencyfreaks.blogspot.com/2016/08/hazard-pointers-vs-rcu.html)
3. [P0461R1: Proposed RCU C++ API](http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2017/p0461r1.pdf)
4. [perfbook](https://cdn.kernel.org/pub/linux/kernel/people/paulmck/perfbook/perfbook.html)

 
















