---
layout:     post
title:      "动态链接之 Hook 系统函数"
subtitle:   ""
date:       2017-5-3 16:10:00
author:     "YuanBao"
header-img: "img/many-books.jpg"
header-mask: 0.35
catalog: true
tags:
 - C
 - programming
---

最近花了两天时间看了下腾讯开源的 C++ 协程库 [*libco*](https://github.com/Tencent/libco)。libco 是微信后台大规模使用的 c/c++ 协程库，2013年至今稳定运行在微信后台的数万台机器上。其通过提供 socket 族函数的 hook，使得后台逻辑服务几乎不用修改逻辑代码就可以完成异步化改造。根据腾讯号称，libco 现在可以轻松达到[**单机千万并发**](http://www.infoq.com/cn/articles/CplusStyleCorourtine-At-Wechat)。

有关 libco 的详细架构和实现我会在后续的博客中详细介绍，今天我们主要来关心一下 libco 介绍中所说的这个 高（chui）大（niu）上（bi）的实现细节，那就是通过 hook 系统的 socket 函数族来实现无需修改代码的异步话改造。简单来说，**就是利用动态链接的原理来修改符号指向，从而达到『偷梁换柱』的编程效果**。在介绍具体怎么做之前，我们需要回顾一下链接的知识。

<!--more-->

## 静态链接库

我们都知道，编译器可以将我们编写的代码编译成为**目标代码**，而链接器则负责**将多个目标代码收集起来并组合成为一个单一的文件**。链接过程可以执行于*编译时(compile time)*，也可以执行于*加载时(load time)*，甚至可以执行于*运行时(run time)*。执行于编译时的链接被称为静态链接，而执行于*加载时*和*运行时*被称为动态链接。

像 Unix ld 一样的静态链接程序以一系列的可重定位的目标文件和参数作为输入，生成一个可以被加载和运行的可执行目标文件。然而，编译系统都提供另外一种机制，也就是将所有相关的目标模块打包成为一个单独的文件，称为静态库。静态库同样可以作为连接器的输入。当输出可执行目标文件时，链接器将拷贝静态库中被程序引用的目标模块并进行链接。

<p class="caution"><strong>注：</strong>在 Unix 下你可以使用 <code>-static</code> 选项生成一个完全链接的可执行文件，该程序将可以加载到存储器运行并无需更进一步的链接。然而对于 MacOS 下的 Clang 而言， <code>-static</code> 选项并无多大意义，从而不被支持。</p>

当然静态库具有许多明显的缺点，比如，静态库需要像软件一样进行定期的维护和更新，一旦程序员需要使用一个库的最新版本，他们必须显示地将程序与最新库链接。再比如，静态链接库中的模块总是被多个进程复制到自己的文本段内，造成存储资源的极大浪费。

## 加载时的动态链接

动态链接库是现代编译系统为了解决静态链接的缺陷而提出的。提到动态链接库，很多人都知道，windows 下的 `.dll` 文件，Linux 系统下的 `.so` 文件，以及 Mac 系统下的 `.dylib` 文件。在任何的操作系统中，一个共享库只能存在一个对应的文件，所有引用该库的目标文件都需要共享该库中的代码和数据。在共享库加载到内存中之后，其 `.text` 段可以被不同的进程所共享。

上文中提到，动态链接可以发生在加载时，也可以发生于运行时。我们用一个图来简要说明 Linux 下加载时动态链接的过程（详细的连接过程涉及到符号解析，重定位一系列复杂的过程，后面有时间再说）：

![](/img/dy_ld.png){: width="300px" height="100px" }

1. 编译器将原始代码文件编译成为可重定位的目标文件。
2. 链接器 ld 将可重定位目标文件与动态链接库进行部分链接生成部分链接的文件 m。注意这里 ld 并没有将 libc.so 中使用到的模块的代码段和数据段拷贝到 m 中，而是拷贝了一下符号表和可重定位信息。
3. 在生成的部分链接文件 m 中，包含了一个 `.interp` 节 [(Dynamic Linker)](https://en.wikipedia.org/wiki/Dynamic_linker#cite_note-8)，其保存了动态链接器的路径名。加载器在加载 m 时首先加载和运行这个动态链接器（在 Linux 下典型是 [ld-linux.so](http://man7.org/linux/man-pages/man8/ld-linux.so.8.html))。

   <p class="caution"><strong>注：</strong>在 MacOS 下面，动态连接器的加载嵌入到了 Mach-   O 程序的 LOAD 指令中，可以通过 <code>otool -l</code> 指令观察到 <code>name /usr/lib/dyld (offset 12)</code>相关的内容。</p>

4. 动态链接器重定位 libc.so 的文本和数据段到某个存储器段上，然后重定位 m 中所有对 libc.so 定义的符号的引用。
5. 之后跳到应用程序起始地址开始执行，这时共享库的位置完全固定，在整个程序运行期间都不会发生变化。

## 运行时的动态链接

尽管加载时的动态链接是大部分编译加载系统都在使用的方式，操作系统同样提供了**<font color="#ff3385">运行时加载和链接共享库的方式</font>**。其可以通过共享库的方式来让引用程序在下次运行时执行不同的代码，这也为应用程序 **Hook** 系统函数提供了基础。

Unix-like 系统提供了 `dlopen`，`dlsym` 系列函数来供程序在运行时操作外部的动态链接库，从而获取动态链接库中的函数或者功能调用。我们用一个简单的例子来说明如何通过 `dlsym` 来包装系统函数，从而实现 Hook 的功能。

假如我们想要统计某一个程序中 `malloc` 函数的调用次数，但是不能对代码进行侵入式的修改，那我们应该怎么做？最简单的思路就是，我们让应用程序对 `malloc` 的解析转为调用我们定义的某个函数，然后在自己的函数中计数 `malloc` 的调用次数，就能达到目的。例如对于如下简单的程序 main.c（无任何意义，仅作为示例使用）:

```c
#include <stdio.h>
#include <stdlib.h>

int main(int argc, char** argv) {
    int* p = (int *)malloc(sizeof(int));
    free(p);
    return 0;
}
```

为了让上述程序中对 `malloc` 的调用重定位到我们自己定义的函数，我们可以利用 `dlsym` 编写如下文件 dlsym_test_preload.c：

```c
#include <stdio.h>
#include <dlfcn.h>

static unsigned int invoke_times = 0;

void* malloc(size_t sz) { 
    void* (*my_malloc)(size_t) = dlsym(RTLD_NEXT, "malloc");
    invoke_times += 1;
    printf("my malloc invoked\n");
    return my_malloc(sz);
}
``` 

上面 line 7 中出现的宏 `RTLD_NEXT` 的含义是告诉链接器，将 `malloc` 这个符号的定义解析到后面的可执行目标文件或者共享库中，而不要解析为本模块中的定义。接下来我们可以将 dlsym_test_preload.c 编译成为动态链接库，由于我在 Mac 下操作，因此我的编译结果如下：

```shell
> clang -o dlsym_test.dylib -shared -fPIC -o2 dlsym_test_preload.c
```

为了能够操纵动态链接器的行为，Linux 下的 ld-linux.so 定义了一系列加载共享库的次序规则 [ld-linux.so](http://man7.org/linux/man-pages/man8/ld-linux.so.8.html)。其中 `LD_PRELOAD` 参数允许用户自定义提前加载的共享库的路径。这也就是说，通过参数 `LD_PRELOAD` 指定的共享库将最先被加载进内存，甚至先于 `/user/lib` 以及 `/user/local/lib`，这为我们实现覆盖系统的函数解析提供了方便。（当然 `LD_PRELOAD` 存在安全隐患，我们这里暂且不说 ）

说到这里，可能用过 TC_MALLOC 的人都会想起来，google 官方提供了一种无需编译的嵌入 TC_MALLOC 的方式，那就是通过 `export LD_PRELOAD = "/usr/lib/libtcmalloc.so"`，然而 google 不建议采用这种方式，具体的原因有很多，例如 `LD_PRELOAD` 可能会影响到所有的应用程序的行为，再例如出于`LD_PRELOAD` 的安全性考虑。

当然，尽管 MacOS 的编译链接系统与 Linux 不同，但我们仍然可以通过相似的原理来实现操纵其动态链接库的目的。对于 MacOS 下的程序我们可以通过修改参数 `DYLD_INSERT_LIBRARIES` 来达到提前载入指定共享库的目的。然而，通过 `man dyld` 可以知道，`DYLD_INSERT_LIBRARIES` 对于 [two-level namespace images](https://developer.apple.com/library/content/documentation/DeveloperTools/Conceptual/MachOTopics/1-Articles/executing_files.html) 没有影响，因此在 Mac 下我们需要通过如下的参数编译程序才能达到目的：

```shell
> clang -o main -force_flat_namespace main.c
> DYLD_INSERT_LIBRARIES=dlsym_test.dylib ./main
> my malloc invoked
```

说到这里，我们简单地来看一下 libco 是如何使用动态链接 Hook 系统函数的。事实上，libco 最大的特点就是将系统中的关于网络操作的阻塞函数全部进行相应的非侵入式改造，例如对于 `read`，`write` 函数，libco 均定义了自己的版本，然后通过 `LD_PRELOAD` 进行运行时地解析，从而来达到阻塞时自动让出协程，并在 IO 事件发生时唤醒协程的目的。关于 libco 的整个架构和流程，后续会有详细介绍。

## 参考

1. [Dynamic Linker](https://en.wikipedia.org/wiki/Dynamic_linker#cite_note-8)
2. [Talk:Dynamic linker](https://en.wikipedia.org/wiki/Talk%3ADynamic_linker)
3. [ld-linux.so](http://man7.org/linux/man-pages/man8/ld-linux.so.8.html)
4. [DYLD_INSERT_LIVRARIES (stackoverflow)](https://stackoverflow.com/questions/34114587/dyld-library-path-dyld-insert-libraries-not-working)
5. *Computer systems : a programmer's perspective*, Randal E·Bryant, David R·O'Hallaron.












