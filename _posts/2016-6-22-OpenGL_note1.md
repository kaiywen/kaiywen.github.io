---
layout:     post
title:      "OpenGL 学习笔记（一）"
subtitle:   "OpenGL 简介及环境配置"
date:       2016-7-3 15:07:00
author:     "YuanBao"
header-img: "img/post-book.jpeg"
header-mask: 0.25
catalog: true
tags:
 - programming
 - OpenGL
---

>作为一个游戏开发者，如果不懂图形学渲染，那么你会很悲剧

## OPENGL 简介

在开始整个学习之前，我们需要知道 OpenGL 是什么。根据 wikipedia 的定义：

> OpenGL 是用于渲染 2D、3D 矢量图形的跨语言、跨平台的应用程序编程接口（API）。这个接口由近 350 个不同的函数调用组成，用来从简单的图形比特绘制复杂的三维景象。

这个定义告诉我们 **OpenGL 本身即不是编程语言，也不是程序库，而是一个 API 标准**。事实上，OpenGL 的 API 仅仅定义了若干可被客户端程序调用的函数，以及一些具名整型常数（例如，常数 GL_TEXTURE_2D 对应的十进制整数为 3553 ）。虽然这些函数的定义表面上类似于 C 语言，但它们是语言独立的。因此，OpenGL 有许多语言绑定，值得一提的包括：JavaScript 绑定的 WebGL（基于 OpenGL ES 2.0 在 Web 浏览器中的进行 3D 渲染的 API）；C 绑定的 WGL、GLX 和 CGL；iOS 提供的 C 绑定；Android 提供的 Java 和 C 绑定等。

更近一步讲，OpenGL 不仅语言无关，而且平台无关。其规范只字未提获得和管理 OpenGL 上下文相关的内容，而是将这些作为细节交给底层的窗口系统。出于同样的原因，OpenGL 纯粹专注于渲染，而不提供输入、音频以及窗口相关的 API。

在本文中以及后文中，我们关注的 OpenGL 实现方案均限定于 C++ 语言。

<!--more-->

#### Core-profile 和快速模式 （Immediate mode）

如果对图形渲染有了解都会知道，从编程的角度我们可以使用两种方式完成某一项渲染功能：通过固定管线渲染或者通过着色器渲染。OpenGL 早期的快速模式（Immediate mode）即对应着固定管线渲染。虽然使用固定管线的方式非常简单易学，开发者无需深入到渲染的背后原理，但是固定管线的渲染方式非常僵化而不灵活，不能满足开发者随心所欲控制渲染效果的要求。因此，从 OpenGL 3.2 版本开始，快速模式就不被推荐使用了，取而代之的是 core-profile 模式，即通过着色器来灵活地实现丰富多彩的视觉效果。当使用 core-profile 模式时，以前版本的废旧的函数将不再被支持，OpenGL 也推荐在代码中使用更现代的函数，在本文以及后文中，我们使用 OpenGL 3.3 版本。

#### 基本原理

首先需要明白的是，**OpenGL 本身是一个大的状态机，这个状态机的所有信息通过 OpenGL context 来维护，例如变量，渲染内存等等**。这里我们不需要担心 OpenGL context 究竟是什么，因为已有的窗口管理系统已经帮助我们维护了这个 context。我们只需要对某些变量进行相应的操作就能够控制 OpenGL 完成我们想要达到的效果。

这里举个简单的例子说明可能更好理解。加入我们需要绘制一个三角形，我们可以通过以下几个步骤完成：

1. 生成一个 buffer 来存储三角形的定点信息 （不用关心 buffer 放在哪里，窗口管理器会负责维护），我们会得到一个 buffer ID。
2. 将定点数据拷贝到 buffer 中 （通过 buffer ID 告诉 OpenGL 往哪里拷贝）。
3. 设置相应的参数和变量，做好渲染的准备。
4. 调用渲染函数渲染。

整个过程我们只需要按照相应的步骤调用相应的函数即可轻松实现。

## OpenGL 在 Mac 中的配置

#### 几个必须的库

在我们可以编写 OpenGL 程序之前，我们需要安装好所需要的环境。在本文以及后文中，我们共需要以下几种库文件：

* **[GLFW](http://www.glfw.org)**：一个 C 语言编写的专门用于 OpenGL 开发的库，它只提供把物体渲染到屏幕所需的必要功能。它可以给我们创建一个 OpenGL 环境，定义窗口参数，以及相应用户输入。前面已经说了，OpenGL 并不规定窗口创建和管理的部分，这一部分完全交由 GLFW 来实现。（当然，你也可以选择使用其他的窗口和环境管理库，例如 [GLUT](https://www.opengl.org/resources/libraries/glut/) 和 [SDL](https://www.libsdl.org/release/SDL-1.2.15/docs/html/guidevideoopengl.html) 等）。
* **[GLEW](http://glew.sourceforge.net)**：由于 OpenGL 是一种标准/规范，它需要由驱动制造商在驱动中来实现这份特定的规范。因为有许多不同版本的OpenGL 驱动，OpenGL 的大多数函数在编译时（compile-time）是未知状态的，需要在运行时（run-time）来请求。这就是开发者的任务了，开发者获取所需的函数的地址，把它们储存在函数指针中以备后用。为了避免这种复杂的过程，我们需要 GLEW 来帮助我们简化函数的获取过程。
* **[SOIL](http://www.lonesock.net/soil.html)**：显然大多数图形学处理的任务都需要读取图片到内存当中。当前存在着各种各样的图片格式，例如 jpeg, bmp, png 等等。我们不可能自行编写读取不同图片格式的代码，因此我们需要一种通用的读取图片的库，它就是 SOIL。
* **[GLM](http://glm.g-truc.net)**：我们在编写图形渲染代码中需要大量的使用到矩阵变换，向量变换以及各种针对矩阵和向量的计算。这一整套都由 glm 库为我们提供。

当我们下载完了所需要的四种库的源代码，我们需要将其在相应的平台上进行编译，并在我们的工程中链接这些库文件。

#### 工程配置

首先需要去上述官网下载相应的源代码文件，这里以 GLFW 在 Xcode 下的配置为例说明。注意最好不要直接下载编译好的二进制文件安装，下载源代码自行编译能够获得在特定平台下最好的性能。由于每个人使用的平台以及 IDE 不同，为了保证下载的源码在每个人的机器上都能成功地编译，这里需要使用到 CMake.

CMake 是一个可以生成项目或者解决方案的工具，它以使用CMake的预定义脚本让用户选择目标IDE（比如：Visual Studio、Code::Blocks、Eclipse）以及平台。这样我们就可以从 GLFW 的源码包生成 Mac 下的编译工程，然后就能编译出这个库。安装 Mac 下的 CMake 相当简单。

当 CMake 安装好了，你可以选择是从命令行还是 GUI 来运行 CMake。这里我们并不想把问题复杂化，因此我们可以直接打开 CMake 的 GUI。CMake 需要一个源码文件夹和一个用于生成二进制的目标文件夹。我们把 source code 目录设置为下载的 GLFW 根目录。然后创建一个叫做build 的目录作为二进制输出的目录。点击左下角 configure 之后，我们可以选择为 Xcode 生成工程，这里我们选择简单点的方式，使用 Unix Makefiles。两次点击 Configure (点击一次之后出现红色警告，继续点击 configure 即可)， 然后点击 generate 之后就可以在刚刚设置的 build 目录下生成相应的 Makefile了。

我们进入 build 目录中就可以使用 make 命令进行编译安装库文件了，一般都会安装到 `/usr/local/include` 和 `/usr/local/lib` 中。

要在 Xcode 中调用相应的 GLFW 函数，我们还需要在 Xcode 工程中添加相应的头文件和库文件，这里我们有两种方式可以让 OpenGL 工程找到相应的库文件：

1. 将 GLFW 的 include 和 lib 文件夹拷贝到 Xcode IDE 本身的 include 和 lib 文件夹中（不推荐）。
2. 在新建的工程中链接 GLFW 的 include 和 lib 文件。这样做更加清晰也更加易于管理，但是每次新建工程都要重新链接相应的库文件。

我们采用第二种方式来配置我们的工程：

1. 通过 Xcode 新建一个空工程或者 Command Line Tool
2. 在工程的 Build Settings 中找到 Header Search Paths，将 `/usr/local/include` 添加到头文件搜索路径中。
3. 在工程的 Build Settings 中找到 Library Search Paths，将  `/usr/local/lib` 添加进来。
4. 在工程的 Build Phases 中的 Link Binary With Libraries 中，添加以下几个库文件，注意 libglfw3.a 可能需要手动去 `/usr/local/lib` 中寻找添加。

![需要的库文件](/img/gllib.png){: width="350px" height="250px" }

#### 环境测试

使用同样的方法，我们可以配置编译安装其他的几个库，并且按照相同的方式将库文件添加到工程 Build Phases中的 Link Binary With Libraries，之后我们可以通过下面的代码简单判断一下我们的环境配置是否成功：

```c
// GLEW
#define GLEW_STATIC
#include <GL/glew.h>
 
// GLFW
#include <GLFW/glfw3.h>

int main()
{
    glfwInit();
    glfwWindowHint(GLFW_CONTEXT_VERSION_MAJOR, 3);
    glfwWindowHint(GLFW_CONTEXT_VERSION_MINOR, 3);
    glfwWindowHint(GLFW_OPENGL_PROFILE, GLFW_OPENGL_CORE_PROFILE);
    glfwWindowHint(GLFW_RESIZABLE, GL_FALSE);
    return 0;
}
```

如果编译成功，那么你的 OpenGL 的环境基本就配置好了，接下来就可以开始我们的 OpenGL 旅程了。




