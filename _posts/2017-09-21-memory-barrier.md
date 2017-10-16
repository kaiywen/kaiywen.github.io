---
layout:     post
title:      "Memory reordering 浅析"
subtitle:   ""
date:       2017-9-22 22:49:00
author:     "YuanBao"
header-img: "img/post-bg-os-metro.jpg"
header-mask: 0.35
catalog: true
tags:
 - programming
 - memory order
 - lock-free
---

最近越来越感觉到坚持写 Blog 不是一件容易的事情，先前立下的每周至少更新一篇的目标也没有完成；leveldb 的系列也一直拖着没有继续，这个要自我反省一下。

这段时间工作之余在慢慢地看 [perfbook](https://www.kernel.org/pub/linux/kernel/people/paulmck/perfbook/perfbook.html) ，个人感觉收获颇多。尽管书中很多内容涉及到了底层体系结构，操作系统和编译系统，在平时的工作中极少会使用到；但是书中所提出的各种设计思想和思考问题的方式，给我带来了很多启发，也为后续学习很多新东西打下了基础。举个例子，最近百度开源了其内部使用的 rpc 框架 [brpc](https://github.com/brpc/brpc)，今天扫描了一下其中 bvar 的实现，发现完全就是 perfbook 中 counting 所使用的 data ownership 设计思想，如果有相应的模式在脑海里，理解起来非常容易。

有关 perfbook 中更多细节，以后再慢慢道来。今天这篇文章所关注的主题 -- **Memory reordering** 以及 **Memory barrier** 也由 perfbook 和 leveldb 中引出，与 lock-free algorithm 息息相关。尽管这个主题涉及到 Cache coherence 协议，CPU 体系结构，Sequential consistency 概念等问题，看起来非常底层，但是一旦深入理解，对于后续编程和学习必定大有帮助。

<!--more-->

## Memory reordering

对于很多人来说，Memory reordering 是一个非常陌生的词。在我们日常的编程过程中，它可能没有发生过；或者它发生了，但是我们从未意识到它的存在。Memory reordering 翻译成中文就是**内存乱序**，我们可以将这种现象描述如下：

> Reads and writes do not always happen in the order that you have written them in your code.
> 读内存操和写内存操作不一定是按照他们在程序中所定义的顺序来执行

你可能会觉得这个表现非常反直觉，从而产生疑惑--如果读写内存操作不按照定义的顺序执行，程序的效果不会受到影响吗？事实上，这是一个非常重要的问题，其表明了 Memory reordering 并不能随意地无条件地产生。因此，在介绍 Memory reordering 的详细内容之前，我们需要指出其产生的前提：**从单个线程的角度来看，任何一段程序在 Reordering 前与 Reordering 后，拥有相同的执行效果**。这段话不是太好理解，我们会在后续内容中结合实例来解释其含义。

#### Compiler Reordering

任何一个程序都需要经过编译才能运行到 CPU 上，而在编译和运行阶段都"可能"产生 Memory reordering。我们把编译阶段产生的乱序称为 Compiler Reordering（编译乱序），也即 Software memory reordering；把产生于运行阶段的乱序被称为 CPU memory reordering，也叫做 Hardware memory reordering。这一小节我们先来介绍编译乱序。

<p class="caution"><strong>注意</strong>：很多不甚了解 Memory Reordering 的人可能会混淆这两个概念，或者认为它们等价。事实上，对于不同体系架构（eg. 单核，多核，Intel-arch 或 PowerPC），Compiler Reordering 与 CPU reordering 有相应的联系和区别。 </p>

编译乱序很好理解，即在编译阶段，编译器为了优化程序的执行效率，自行地将内存操作指令重排，从而使得读写内存的指令与程序定义的操作顺序不一致。我们通过一个例子来证明 Compiler Reordering 的存在。对于如下简单的 c 程序：

```c++
int a, b;
int foo()
{
    a = b + 1;
    b = 0; 
    return 1;
}
```

在 CentOS 6 with gcc 4.4.7 环境下通过命令编译 `gcc -S -masm=intel foo.c` 得到汇编程序，打开 foo.s 查看汇编代码如下：

```c++
mov eax, DWORD PTR b[rip]
add eax, 1
mov DWORD PTR a[rip], eax    // --> store to a
mov DWORD PTR b[rip], 0      // --> store to b
```

通过以上注释可以看到，汇编代码严格按照程序中定义顺序执行 load 和 store 指令，即先保存变量 `a` 的值，后保存变量 `b` 的值。因此，这段代码并没有产生任何内存乱序。

OK，接下来我们使用编译指令 `gcc -S -O2 -masm=intel foo.c` 重新编译 foo.c（注意在 intel 体系结构下，gcc 通过 `-O2` 的编译选项会生成乱序代码，而 clang 经过测试则不会），仔细观察这次生成的汇编代码：

```c++
mov eax, DWORD PTR b[rip]
mov DWORD PTR b[rip], 0      // --> store to b
add eax, 1
mov DWORD PTR a[rip], eax    // --> store to a
```

可以看到，汇编指令先执行变量 `b = 0` 的存储，之后才执行 `a = b + 1` 的操作，表明变量 `a` 和 `b` 的 store 操作没有按照他们在程序中定义的顺序来执行，即产生了 Compiler Reordering。我们验证一下编译乱序的产生是否符合前述的前提条件。很容易看到，在单线程的场景下，最终都会得到 `a = 1` 以及 `b = 0`，即乱序没有影响单线程的执行结果.

Compiler Reordering 是编译器为了提高程序的执行效率而故意产生的。但是在很多时候，我们希望避免这种乱序的发生，即告诉编译器不应产生乱序。一般而言有两种方式来阻止编译器产生乱序，即*显式的内存屏障（memory barrier）*和*隐式的内存屏障*。

对于 GCC 及 clang 编译器而言，我们可以使用如下指令来显示地阻止编译器产生乱序：

```c++
#define barrier() __asm__ __volatile__("":::"memory")  // linux
#define barrier() _ReadWriteBarrier()                  // win
``` 

**`barrier()` 指令指示编译器不要将该指令之前的 Load-store 操作移动到该指令之后执行**。我们修改上面的程序如下，并再次使用 `-O2` 选项来生成汇编代码。此时你会发现 gcc 将不会把变量 `b` 的 store 操作提前到变量 `a` 的 store 操作之前

```c++
int foo()
{
    a = b + 1;
    barrier();
    b = 0;
    return 1;
}
```

需要指明的是，上述的 `barrier()` 并不是一条需要 CPU 执行的指令，其只是让应用程序告知编译器不要产生跨过该指令的编译乱序。在生成的汇编代码中不包括任何与 barrier 相关的指令，这一点与后续的 CPU barrier 不同。

除了上述显示的 barrier() 以外，程序中的其他元素，例如同步原语（e.g. Mutex，RWLock，信号量，CPU 屏障等），非 inline 函数以及 C++11 提供的 non-relaxed 原子操作等均可以扮演 compiler barrier 的作用。关于 C++11 提供的内存模型和原子操作，后续有时间再专门介绍。

#### CPU memory reordering

介绍完编译乱序，接下来我们再来看一下运行时产生的乱序，即 CPU memory reordering，也被称为 Hardware memory reordering。相比于编译乱序而言，CPU 乱序的产生原因，产生背景以及对程序的影响都要更为复杂。其至少涉及到如下的相关内容：

1. acquire-release semantics;
2. Sequential consistency;
3. Synchronize-with relation;
4. Strongly- or Weakly-ordered memory;
5. Cache coherence protocol

当然，要完全把所有的东西都说明白可能需要一篇很长的文章，我们这里尽量先用简单的例子来证明 CPU memory reordering 的存在，然后介绍一下几种不同的 CPU reordering 的类型以及相应的 barrier。

对于如下的两个程序 P1 和 P2，假设 X 和 Y 的值初始均为 1， 且 P1 和 P2 分别由两个线程运行在一个双核 CPU 上，我们的问题是：两个线程并行执行得到的 `r1` 以及 `r2` 的值会不会同时为 0 ？

```c++
  |---------------|          |---------------|
  |1  mov [X], 1  |          |1  mov [Y], 1  |
  |2  mov r1, [Y] |          |2  mov r2, [X] |
  |---------------|          |---------------|
         P1                         P2
```

大多数人可能都会认为 `r1` 和 `r2` 是不可能同时为 0 的。然而根据 [Intel Arch Specification](https://software.intel.com/en-us/articles/intel-sdm)，Intel CPU 的每个核是可以自由地重排其指令的执行顺序，从而导致两个核都可能先执行对于 `r1` 和 `r2` 的复制操作，从而导致最终得到  `r1` 以及 `r2` 均为 0。

如果你是一个主动思考的人，看到这里一定会有一个疑问：这种 CPU 产生的乱序难道没有违背先前所制定的产生 memory reordering 的前提吗？为了清楚地说明这个问题，我们再重申一下任何体系架构产生 memory reordering 的前提要求:

> 从单个线程的角度来看，任何一段程序在 Reordering 前与 Reordering 后，拥有相同的执行效果

我们分别看执行 P1 和 P2 的两个线程，如果我们从 P1 的线程角度角度来看，先执行指令 1 还是 指令 2 对 P1 本身来讲没有任何影响；同样的，如果单从 P2 的角度来看，先执行指令 1 还是 指令 2 对其本身也没有任何影响，因此这种乱序完全满足上述的前提。（这个例子也充分解释了什么叫做"从单线程的角度"）。

接下来我借用 [Preshing](http://preshing.com) 的代码来证明 CPU memory reordering 的存在。为了支持在 MacOS 下的运行，用 `dispatch_semaphore_t` 替换了 Linux 下的 `sem_t`，完整修改过的程序见 [github](https://github.com/kaiywen/code-for-blog/blob/master/ordering.cpp)。这里要向 Preshing 表示感谢，同样作为一个游戏制造者，我从他的 Blog 学习到了很多，也只能自愧不如，然后见贤思齐了。

程序的主体通过两个线程分别执行如下程序，也就是完整地实现了上述关于 X 和 Y 的伪代码：

```c++
void *thread1Func(void *param) {
    MersenneTwister random(1);
    for (;;) {
        sem_wait(&beginSema1);
        while (random.integer() % 8 != 0) {}  // 产生一个随机地延迟

        X = 1;
#if USE_CPU_FENCE
        asm volatile("mfence" ::: "memory");  // 阻止 CPU 乱序
#else
        asm volatile("" ::: "memory");  // 阻止编译乱序
#endif
        r1 = Y;
        sem_post(&endSema);
    }
    return NULL;  // Never returns
};
```

然后主程序进行如下同步，其执行多次地循环来输出 `r1` 和 `r2` 的值，当检测到 `r1` 和 `r2` 均为 0 时，表明上述的 CPU reordering 产生了：

```c++
int main()
{
    sem_init(&beginSema1, 0, 0);
    sem_init(&beginSema2, 0, 0);
    sem_init(&endSema, 0, 0);

    pthread_t thread1, thread2;
    pthread_create(&thread1, NULL, thread1Func, NULL);
    pthread_create(&thread2, NULL, thread2Func, NULL);

    int detected = 0;
    for (int iterations = 1; i < MAX_ITER; iterations++)
    {
        X = 0; Y = 0;
        // Signal both threads
        sem_post(&beginSema1);
        sem_post(&beginSema2);
        // Wait for both threads
        sem_wait(&endSema);
        sem_wait(&endSema);
        // Check if there was a simultaneous reorder
        if (r1 == 0 && r2 == 0)
        {
            detected++;
            printf("%d reorders after %d iters\n", detected, iterations);
        }
    }
    return 0;
}
```

可以看到 main 函数通过 `beginSema1`，`beginSema2` 以及 `endSema` 进行同步。`beginSema` 扮演了 CPU memory barrier 的作用，确保在两个线程开始运行时，其看到的 `X` 和 `Y` 的值已经被设为 0，同时 `endSema` 也扮演了 CPU barrier 的作用，确保经过两个线程修改得到的 `r1` 和 `r2` 已经传播到 main 线程中。这个解释你可能暂时不太理解，等以后充分介绍了 acquire-release 语义之后应该就会明白了。

修改 `USE_CPU_FENCE` 为 0 ，并且执行 main 函数中的主循环 300 万次，输出结果如下：

```
22308 reorders detected after 2999891 iterations
```

即出现了 22308 次 CPU 乱序，如果设置 `USE_CPU_FENCE` 为 1， 并重新编译运行该程序，结果将检测不到任何的 CPU 乱序。

#### CPU memory barrier

CPU 对内存的操作共分两种，分别是 load 和 store，因此在理论上存在四种 CPU 内存乱序：Load-Load，Load-Store，Store-Store，Store-Load 乱序。这里用『理论上』三个字的原因是：在不同的硬件内存模型上，可能产生的内存乱序的种类并不相同。在 Strongly-ordered memory 环境例如 X86/X64 下，唯一允许产生的乱序是 Store-Load 乱序。（不妨回头去检查一下以上的程序实例是否是 Store-Load 乱序）。

针对四种不同的 CPU 内存乱序，也应该存在对应的四种 barrier。然而针对 Intel CPU 而言，我们主要讨论三种内存 Barrier：

1. Store Barrier，确保 Barrier 前后的 store 操作不会发生乱序;
2. Load Barrier，确保 Barrier 前后的 load 操作不会发生乱序;
3. Full Barrier，确保 Barrier 前后的内存操作不会发生乱序;

我们可以通过 CPU 提供的如下的指令来显示的达到 Barrier 的目的：

```c++
#define LOAD_BARRIER() __asm__ __volatile__("lfence")
#define STORE_BARRIER() __asm__ __volatile__("sfence")
#define FULL_BARRIER() __asm__ __volatile__("mfence")
```

同时，任何带有 `lock` 操作的指令以及某些原子操作指令均可以当做隐式的 Barrier，例如：

```c++
__asm__ __volatile__("lock; addl $0,0(%%esp)");

__asm__ __volatile__("xchgl (%0),%0");
```

我们也可以将 Compiler Barrier 和 CPU Barrier 通过一条指令来实现：

```c++
#define ONE_BARRIER() __asm__ __volatile__("mfence":::"memory")
```

最后需要说明的是，**不同于编译屏障，CPU 内存屏障是在 CPU 上执行的指令，任何在程序中定义的 CPU 屏障最后都会编译成为汇编代码中的指令**，例如，使用上面的 `LOAD_BARRIER()` 会在汇编代码中相应的位置插入 `lfence` 指令。

## 更多内容

有关 memory reordering 的简介到这里就先结束了。这一块的内容想要用一篇文章说明白显然不太现实，关于前面提到的：

* acquire-release semantics
* Sequential consistency
* Synchronize-with relation
* Strongly- or Weakly-ordered memory
* Cache coherence protocol 

等内容，留待后续介绍。

回想一下，突然发现欠了好多东西没写。。。











