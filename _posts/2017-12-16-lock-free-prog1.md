---
layout:     post
title:      Lock-free 编程：A case study（上）
subtitle:   "基于 CAS 的无锁数据结构"
date:       2017-12-14 15:56:00
author:     "YuanBao"
header-img: "img/post-bg-os-metro.jpg"
header-mask: 0.25
catalog: true
tags:
 - protocol
 - Distributed
 - lock-free
---

在并行和分布式编程领域，Lock-free 是一个非常复杂也非常具有挑战性的话题。前面的文章 [Memory reordering 浅析](/2017/09/22/memory-barrier/) 中，我们提到了一个与 Lock-free 息息相关的概念--内存屏障。今天我们通过一个由浅入深的编程范例来具体介绍 lock-free 数据结构和算法思想，包括 [Hazard Pointer](https://en.wikipedia.org/wiki/Hazard_pointer) 和 [Read-Copy Update](https://en.wikipedia.org/wiki/Read-copy-update) 的基本概念。这篇文章的初衷都来源于对 Andrei Alexandrescu 的文章 [Lock-Free Data Structures](https://erdani.com/publications/cuj-2004-10.pdf) 的扩展和理解。

## Lock-free vs. Wait-free

从直观上理解，lock-free 数据结构即不使用任何锁的数据结构，Lock-free 编程则指利用一组特定的原子操作来控制多个线程对于同一数据的并发访问。相比于基于锁的算法而言，Lock-free 算法具有明显的特征：**某个线程在执行数据访问时挂起不会阻碍其他的线程继续执行**。这意味着在任意时刻，多个 lock-free 线程可以同时访问同一数据而不产生数据竞争或者损坏。

<!--more-->

上面的定义保证了 lock-free 程序中的一组线程中至少有一个可以顺利的执行而不产生阻塞，从而确保整个程序的顺利执行。尽管 lock-free 算法不会产生死锁的问题，但在程序运行的过程中，某个线程仍然会被其他线程的行为所影响，甚至被长时间任意地阻塞。这种特性可能会导致 lock-free 算法的性能下降。Wait-free 算法可以用来避免这种问题。

相比于 Lock-free，Wait-free 的要求更为严格：任何一组满足 Wait-free 线程都必须在有限的执行步骤中完成，而不必受到其他线程的影响。因此从这个定义来看，任何可能导致某个线程无限重试的算法都不能算作 Wait-free。显而易见，由于 Wait-free 算法的更高要求，其比 Lock-free 实现起来更为复杂，同时也会为单个操作例如 read，write 引入额外的负载。但在实时性要求较高的场景下，引入 Wait-free 数据结构能够达到比 Lock-free 更好的并发性能。

## 实现一个 Lock-free Map

在基本了解了 Lock-free 概念之后，我们以一个具体的例子来介绍 Lock-free 算法会涉及到的编程思想。这个例子来源于 [Lock-Free Data Structures](https://erdani.com/publications/cuj-2004-10.pdf)，我们将其通过 C++11 展现并扩展自己的理解。在开始之前，先定义一个概念 *Write-Rarely-Read-Many*，也就是*读多写少*。这代表了一个典型的优化并发性能的场景（例如 Linux 内核中的路由表），针对这个场景产生了很多优化技术，例如 Read-Writer Lock，Sequence Lock 以及以后要介绍的 Hazard Pointer 和 RCU。下文中我们即来实现一个 Write-Rarely-Read-Many 场景下的 map。

#### Lock-based Map

如果要实现一个支持并发访问的 Map 数据结构，最简单的方法就是使用互斥锁锁住临界区：

```c++
#include <map>
#include <mutex>

template <typename K, typename V>
class LockMap {
private:
    std::map<K, V> map_;
    std::mutex mutex_;

public:
    V LookUp(K& key) {
        std::lock_guard<std::mutex> lock(mutex_);
        return map_[key];
    }

    void Update(K& key, V& val) {
        std::lock_guard<std::mutex> lock(mutex_);
        map_[key] = val;
    }
};
```

使用锁的代码非常清晰，也能够保证并发访问的正确性。然而从性能角度上来考虑，多个并发调用上述 `LookUp` 的线程将会产生锁竞争，从而降低访问性能。尤其是在并发读线程数量很大时，这种性能降低尤为明显，一个直观的展示可以参考[这里](http://preshing.com/20111118/locks-arent-slow-lock-contention-is/)。

为了将上述代码从 Lock-based 转为 Lock-free，我们需要使用到基本的原子操作原语：CAS（compare-and-swap）。CAS 的语义可以通过如下的代码来描述：

```c++
template <typename T>
bool CompareAndSwap(T *p, T old_val, T new_val) {
    if(*p == old_val) {
        *p = new_val;
        return true;
    }
    return false;
}
```

即当指针指向的值等于 `old_val` 时，将其值改为 `new_val` 并返回 `true`，否则返回 `false`。需要说明的是，我们所使用的 CAS 原语相比于上述的 C++ 代码有两点不同：1）CAS 原语是一个原子操作，整个过程的原子性由编译器和操作系统保证；2）CAS 操作是针对位长而言，其并不需要模板化；例如，Intel IA32 体系结构可以提供 64-bit 的 CAS 操作。事实上，在进行 Lock-free 编程时，我们并不需要去直接调用底层的汇编代码。通常语言库，操作系统或者编译器都提供了封装好的 CAS 操作供我们调用：

```c++
// for gcc
bool __sync_bool_compare_and_swap(type *ptr, type oldval, type newval, ...)
type __sync_val_compare_and_swap(type *ptr, type oldval, type newval, ...)

// for Microsoft
long InterlockedCompareExchange(long* pointer, long desired, long expected);

// for C++11
template <typename T>
bool atomic_compare_exchange(T* pointer, T* expected, T desired);
```

#### Lock-free Map: First Glance

我们可以基于如下思想来初步实现一个 Lock-free 的并发 Map：

1. read 是完全无锁的，因此可以并发操作；
2. 对于 update 操作，首先将存储数据的 Map 完全复制一份，更新复制后 Map，并通过 CAS 操作将原来旧 Map 替换成为新的 Map；如果 CAS 操作失败，则循环执行 copy/update/CAS 操作直到成功；

这里最大的疑问在于如何通过 CAS 操作替换新 Map。我们可以将上述代码中对于 `std::map<K, V> map_` 的引用改成指针，即使用指针 `std::map<K, V>* pmap_` 来指向存储数据的区域，这样可以通过改变 `pmap_` 指向来实现 Map 的替换。需要说明的是，大多数的现代体系结构均提供了对于对齐整数以及指针访问的原子性保证。

经过改造的 Lock-free Map 的代码如下：

```c++
#define CAS(val, oldval, newval) \
    __sync_bool_compare_and_swap((val), (oldval), (newval))
    
template <typename K, typename V>
class LockFreeMap {
private:
    std::map<K, V> *pMap_;

public:
    V LookUp(K &key) {
        return pMap_[key];
    }

    void Update(K &key, V& val) {
        std::map<K, V> pNewMap = nullptr;
        std::map<K, V> pOldMap = nullptr;
        do {
            pOldMap = pMap_;
            if (pNewMap != nullptr) {
                delete pNewMap;
            }
            pNewMap = new std::map<K, V>(*pOldMap);
            pNewMap[key] = val;
        } while( CAS(&pMap_, pOldMap, pNewMap) );
    }
};
```

简单的分析一下可知，上面的 `Update` 实现并不满足 wait-free 的要求。如果多个线程同时执行 `Update` 操作，那么某些线程可能会一直处于循环中；但在整个过程中，部分线程一定能够顺利地完成操作，从而保证程序整体能够继续执行。

上面的实现对于 go 或者 java 这样天生具有自动垃圾回收机制的语言来说可能已经足够了。然而，对于使用 c++ 实现的代码，你会发现 `pOldMap` 指向的内存块并没有被释放。那么，**什么时候、采用什么机制来释放旧的数据结构成为了横亘在我们面前最大的问题**。

#### Lock-free Map：垃圾内存释放

在需要应用程序自行管理内存的场合，引用计数一般都能排上用场。引用计数的思想非常直观：我们将一个计数 count 与指针 `pMap_` 绑定在一起；当需要访问该 Map 时，将 count 值加一，然后再读取 `pMap_` 的值，访问完成之后将 count 值减一；对于 `Update` 而言，当检测到 `pOldMap` 的 count 值为 0 时，即可『安全』将其删除。

这种方案看起来似乎天衣无缝，完美解决了我们的问题。然而考虑以下这种执行顺序：

1. Writer A 调用 `Update`，检测到 `pOldMap` 对应的 count 为 0；
2. Reader B 将 `pOldMap` 的 count 值加一，并尝试读取 `pOldMap`；
3. Writer A 此时执行 `delete` 操作删除 `pOldMap`；
4. Reader B 读取数据将会出现未定义的异常行为；

仔细思考一下就会发现，不论我们使用什么样的方式，都会碰到一个矛盾的问题：如果需要读取 `pMap_`，我们需要先读取它的 count 值，但是 count 值又必须与 `pMap_` 原子性地同时读取，那么我们将陷入**读取 A 必须读取 B，而读取 B 又必须读取 A **的境地。那么我们有没有什么办法来处理这种进退两难的问题呢？答案就是 *CAS2* 原语。

CAS2 也就是双倍字长的 CAS 原语，在 32-bit 的机器上 CAS2 能够实现 64-bit 的原子操作，同样在 64-bit 的机器上可以实现 128-bit 长度的 CAS 操作。

<p class="caution"><strong>注意</strong>：CAS2 与 DCAS 是不同的原语，例如在 32-bit 的机器上，DCAS 可以同时对两个 32-bit 的指针指向的区域进行原子 compare-and-swap，而 CAS2 是对 64-bit 长的连续空间做 CAS。DCAS 由于开销较大，很少被现代的体系结构所支持。</p>

在我的 64-bit 机器上，为了能够使用 CAS2，需要将指针 `pMap_` 与 `count` 变量存储在连续的 128-bit 空间内：

```c++
#define MEM_ALIGNED __attribute__(( __aligned__(128) ))

template <typename K, typename V>
class LockFreeMap2 {
private:
    struct MapPointer {
        std::map<K, V> *p_;
        long count_;

        MapPointer() {
            p_ = nullptr;
            count_ = 0;
        };
    } MEM_ALIGNED;

    typedef struct MapPointer MapPointer;
    MapPointer pMap_;
    
    ...
};
```

通过结构 `MapPointer`，我们可以同时操作 128-bit 的空间，因此 `LookUp` 的实现如下：

```c++
#define CAS2(val, oldval, newval) \
    __sync_bool_compare_and_swap((long long*)(val), \
        (*(long long*)(oldval)), (*(long long*)(newval)))
    
V LookUp(K &key) {
    MapPointer pOld, pNew;
    do {
        pOld = pMap_;
        pNew = pOld;
        pNew.count_++;
    } while (!CAS2(&pMap_, pOld, pNew));
    V temp = pNew.p_[key];
    do {
        pOld = pMap_;
        pNew = pOld;
        pNew.count_--;
    } while (!CAS2(&pMap_, pOld, pNew));
    return temp;
}
```

逻辑应该很容易理解，在每次读取之前，我们需要先通过 CAS2 来增加 `pMap_` 对应的 `count_` 值，在获得了所需值之后，我们再将其对应的 `count_` 值减一。因此 `count_` 值为 0 时表示当前可以安全地释放 `pMap_` 指向的内存。所以 `Update` 操作即需要在 `count_` 值为 0 的窗口期间将内存安全地释放掉：

```c++
void Update(K &key, V& val) {
    MapPointer pOld, pNew;
    do {
        pOld.p_ = pMap_.p_;
        if(pNew.p_ != nullptr) {
            delete pNew.p_;
        }
        pNew.p_ = new std::map<K, V>(*pOld.p_);
        pNew.p_[key] = val;
    } while(!CAS2(&pMap_, pOld, pNew));
    delete pOld.p_;
}
```

注意 `pOld` 和 `pNew` 的 `count_` 初始值为 0。上述的代码仍然是按照 copy/update/CAS 的逻辑执行的：line 4 每次均先将当前的 `pMap_` 记录下来，然后将记录下来的数据复制到 `pNew` 中，并且更新 `pNew`。如果 line 10 执行成功，表明当前 `pMap_` 的 `count_` 值一定为 0，因此将被替换成为 `pNew`；替换完成之后，可以安全的删除 `pOld` 中的垃圾内存。

观察一下现在的实现与前面的实现有何不同：`Update` 操作必须等待所有的 `LookUp` 操作读取完数据之后才能删除 `pOld`；这确保了读取者不会出现前面所说的异常的行为。

## 更进一步

上面通过一步步的实现表明了使用 CAS2 可以达到我们想要的实现一个 lock-free map 的目的，但是如同 perfbook 中所说的那样：

> concurrency has most deﬁnitely reduced the usefulness of reference counting!

在并发的环境下，相比于引用计数，存在更好地替代地方案来完成同样的工作，这也就是这篇文章的下部即将要介绍的 Hazard Pointer 以及 RCU 的思想。





