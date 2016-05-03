---
layout:     post
title:      "Python 中 struct 模块的用法"
subtitle:   ""
date:       2015-12-25 19:45:00
author:     "YuanBao"
header-img: "img/post-python.jpg"
header-mask: 0.25
catalog: true
tags:
 - programming
 - python
---


Python 为了保持语言的简洁，仅仅为用户提供了几种简单的数据结构：`int`, `float`, `str`, `list`, `dict` 和 `tuple`。不同于编译型语言 C/C++，在 Python 中，我们往往不需要关心不同类型的变量在解释器内部的实现方式。例如，对于一个长整形数据，我们在 Python 2 中可以直接写成 `a=123456789012345L`，而不用去考虑变量 `a` 占了几个字节。这种抽象的方式为程序的编写提供了足够的支持，但是在某些情况下（比如读写二进制文件，进行网络 Raw Socket 编程）的时候，我们需要一些其他模块来实现我们关于变量长度控制的需求。


## struct 模块
当我们在 Python 中跟二进制数据打交道的时候，就要用到 `struct` 这个模块了。`struct` 模块为 Python 与 C 的混合编程，处理二进制文件以及进行网络协议交互提供了便利。理解这个模块主要需要理解三个函数：

```python
struct.pack(fmt, v1, v2, ...)
struct.unpack(fmt, string)
struct.calcsize(fmt)
```
第一个函数 `pack` 负责将不同的变量打包在一起，成为一个字节字符串，即类似于 C 语言中的字节流。第二个函数 `unpack` 将字节字符串解包成为变量。第三个函数 `calsize` 计算按照格式 fmt 打包的结果有多少个字节。这里打包格式 fmt 确定了将变量按照什么方式打包成字节流，其包含了一系列的格式字符串。这里就不再给出不同格式字符串的含义了，详细细节可以参照 [Python Doc (struct)][1].

## 使用 struct 打包定长结构
一般而言，在使用 struct 的时候，要打包的数据都是定长的。定长的数据代表你需要明确给出要打包的或者解包的数据长度，否则打包解包函数将会出错。下面用例子说明什么是定长打包：

```python
import struct

a = struct.pack("2I3sI", 12, 34, "abc", 56)
b = struct.unpack("2I3sI", a)

print b

## 输出 (12, 34, 'abc', 56)
```

上面的代码将两个整数 12 和 34，一个字符串 "abc" 和一个整数 56 一起打包成为一个字节字符流，然后再解包。其中打包格式中明确指出了打包的长度：`"2I"` 表明起始是两个`unsigned int`，`"3s"` 表明长度为4的字符串，最后一个 `"I"` 表示最后紧跟一个 `unsigned int`。所以上面的打印 b 输出结果是：(12, 34, 'abc', 56)。

我们可以调用 `calcsize()` 来计算 `"2I3sI"` 这个模式占用的字节数：

```python
print struct.calcsize("2I3sI")
## 输出 16
```

可以看到上面的三个整型加一个 3 字符的字符串一共占用了 16 个字节。为什么会是 16 个字节呢？不应该是 15 个字节吗？其实，在 `struct` 的打包过程中，根据特定类型的要求，必须进行字节对齐。由于默认 `unsigned int` 型占用四个字节，因此要在字符串的位置进行4字节对齐，因此即使是 3 个字符的字符串也要占用 4 个字节。

再看一下不需要字节对齐的模式：

```python
print struct.calcsize("2Is")
## 输出 9
```

由于单字符出现在两个整型之后，不需要进行字节对齐，所以输出结果是 9.

需要指出的是，对于 `unpack` 而言，只要 `fmt` 对应的字节数和字节字符串 `string` 的字节数一致，就可以成功的进行解析，否则 `unpack` 函数将抛出异常。例如我们也可以使用如下的 `fmt` 解析出 `a`：

```python
c = struct.unpack("2I2sI", a)
print struct.calcsize("2I2sI")
print c

## 输出 16 (12, 34, 'ab', 56)
```
可以看到这里 `unpack` 解析出了字符串的前两个字符，没有产生任何问题。

## struct 处理不定长数据
在上一节的介绍中，我们看到了在使用 `pack` 和 `unpack` 的过程中，我们需要明确的指出打包模式中每个位置的长度。比如格式 `"2I3sI"` 就明确指出了整型的个数和字符串的个数。有时候，我们还可能会需要处理变长的打包数据。

#### 变长字符串的打包
例如我们在程序中可能会得到一个字符串 s，这个 s 没有一个固定的长度，所以我们每次打包的时候都需要将 s 的长度也打包到一起，这样我们才能进行正确的解包。其实，这种情况在处理网络数据包中非常常见。在使用网络编程的时候，我们可能利用报文的第一个字段记录报文的长度。每次读取报文的时候，我们先读取报文的第一个字段，获取其长度之后在处理报文内容。

我们可以采用两种方式处理这种情况：

```python
s = bytes(s, 'utf-8')    # Or other appropriate encoding
struct.pack("I%ds" % (len(s),), len(s), s)
```
或者

```python
struct.pack("I", len(s)) + s
```
第一种方式先将报文转变成为字节码，然后获取字节码的长度，将长度嵌入到打包之后的报文中去。可以看到格式字符串中的 `"I"` 就用来记录报文的长度。第二种方式是直接将字符串的长度打包成字节字符串，再跟原始字符串做一个连接操作。

#### 变长字符串的解包
根据上面的打包方式，我们可以轻松的解开打包串：

```python
int_size = struct.calcsize("I")
(i,), data = struct.unpack("I", data[:int_size]), data[int_size:]
data_content = data[i:]
```
由于报文的长度 `len(s)` 我们使用定长的整型 `"I"` 进行了打包，所以解包的时候我们可以先将报文长度获取出来，之后再根据报文长度读取报文内容。


[1]:	https://docs.python.org/2/library/struct.html