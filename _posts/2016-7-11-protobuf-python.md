---
layout:     post
title:      "『译』Protocol Buffer 基础：Python"
subtitle:   ""
date:       2016-7-10
author:     "YuanBao"
header-img: "img/post-python2.jpg"
header-mask: 0.25
catalog: true
tags:
 - python
 - protobuf
---

这篇文章为 Python 编程者提供了一个使用 protobuf 的基本教程。下文通过创建一个简单的应用，向你展示了三点主要内容：

* 怎样在一个 `.proto` 文件中定义消息格式
* 怎样使用 protobuf 的的编译器
* 怎样利用 protobuf 的 API 来读写消息

这并不是一个在 Python 中使用 protobuf 的完整的教程。如果你需要更加详细的信息，可以查看  [Protocol Buffer Language Guide](https://developers.google.com/protocol-buffers/docs/proto)，[Python API Reference](https://developers.google.com/protocol-buffers/docs/reference/python/index.html)，[Python Generated Code Guide](https://developers.google.com/protocol-buffers/docs/reference/python-generated) 以及 [Encoding Reference](https://developers.google.com/protocol-buffers/docs/encoding)。

## 为什么要使用 Protobuf ？

我们接下来将使用一个非常简单的『地址簿』应用，该应用将能够从一个文件中读取某人的联系方式或将联系方式写回到文件中。在地址簿中的每个人都有一个姓名，ID，email 以及联系电话。

关于怎样序列化以及读取这种结构化的数据，你可能有以下几种方式：

* 使用 Python 中的 `pickling`。 这是一种集成在 Python 语言内的默认的方式，但是其不能够应对模式演化。同时，当你想要在两种使用不同语言例如 C++ 或者 Java 编写的程序之间共享数据时，`pickling` 将不能很好的工作。
* 你可以自定义一种方式来将数据编码成一个特殊的字符串，例如你可以将 4 个整数编码成 "12:3:-23:67"。这是一种简单并且灵活的方式，尽管你可能需要编写一次性的编码和解码器，并且在编解码时损失一些效率。对于一些非常简单的数据来说，这是一种很好地选择。
* 将数据序列化成为 XML 格式。由于 XML 非常易于人类阅读，其对于很多语言都具有已经绑定好的库，其成为了一种非常具有吸引力的方案。然而，XML 格式非常浪费空间，并且在编解码的过程中会对应用产生较大的性能影响。同时，在 XML DOM 树中进行字段的查询比在普通的类中进行字段查询复杂的多。

相比之下，Protocol Buffer 提供了一种非常灵活自动的方案来准确的解决这个问题。通过 Protobuf，你可以编写一个 `.proto` 的文件来描述你想要存储的数据类型。Protobuf 的编译器将能够自动的创建一个类来实现对于 protobuf 数据的高效的二进制编解码。这个自动产生的类为使用者提供了 `getter` 和 `setter` 函数对协议中的域进行访问，其负责处理所有的底层的读写细节。重要的是，protobuf 能够轻松地支持协议格式扩展。随着时间的推移，尽管定义的协议格式发生了变化，protobuf 仍然能够读取使用旧协议格式编码的数据。

## 定义你的协议格式

为了创建你的地址簿应用，你需要首先定义一个 `.proto` 文件。`proto` 文件的定义方式非常简单：为每一个需要序列化的数据结构定义一个消息，之后再为消息中的每个字段定义一个名称和类型。下面是一个定义了地址簿数据结构的 `.proto` 文件 `addressbook.proto`：

``` java
package tutorial;

message Person {
    required string name = 1;
    required int32 id = 2;
    optional string email = 3;
    
    enum PhoneType {
        MOBILE = 0;
        HOME = 1;
        WORK = 2;
    }

    message PhoneNumber {
        required string number = 1;
        optional PhoneType type = 2 [default = HOME];
    }

    repeated PhoneNumber phone = 4;
}

message AddressBook {
    repeated Person person = 1;
}
```

你可以看到，protobuf 的定义语法非常类似于 C++ 或者 Java. 我们接下来细细地看一下该文件中每个部分的具体含义。

每一个 `.proto` 文件都由一个包声明开头，同其他的语言一样，这种包声明能够有效地避免不同工程之间的命名冲突。在 Python 中，包自然而然地由目录结构所定义，因此你在 `.proto` 所定义的 `package` 名对生成的代码并没有效果。然而，在 Python 中你仍然需要像在其他语言中一样定义一个包声明开避免命名冲突。

接下来是消息（Message）定义，每一个消息都包含了一系列具有类型的字段。Protobuf 提供了需要简单的数据类型来作为字段类型的定义，包括 `bool`, `int32`, `float`, `double` 以及 `string`。你也可以使用其他的消息类型类作为某一个字段的类型，例如在上面的例子中，`Person` 消息中 包含有一个 `PhoneNumber` 消息，同时 `AddressBook` 消息中包含有 `Person` 消息。如同你所看到的，你可以在消息体内定义其他的消息类型，例如 `Person` 消息体内的 `PhoneNumber` 消息。如果你需要你的字段拥有一些提前定义的值，你也可以使用 `enum` 来定义枚举类型。这里我们希望电话类型 `PhontType` 可以是 `MOBILE`, `HOME` 以及 `WORK` 中的一种。

上述代码中的 `=1`，`=2` 代表不同的字段在二进制编码过程中的唯一标签。相比于较大的数据，标签 1-15 需要更少的字节来进行编码。因此处于优化方面的考虑，你可以将标签 1-15 用于哪些经常使用或者重复使用的字段，而将 16 以上的标签分配给较少使用的可选的字段。每一个 repeated 字段都需要重编码其标签，因此重复的字段非常适合分配较小的标签数字来优化性能。

消息中的每一个字段都必须通过如下的修饰符来标识：

* `required`：表明该字段必须提供值，否则该消息将会被认为是『未初始化』。尝试序列化一个未初始化的消息将会抛出一个异常，而尝试解码一个未初始化的消息将会直接失败。除了这点以外，一个 `required` 的字段和一个 `optional` 字段的表现完全一样。
* `optional`：该字段可以设置或者不设置值。如果没有对 `optional` 的字段设置值，那么它将自动拥有一个默认的值。对于简单的类型，你可以自行指定默认值，例如上述代码中 `PhoneNumber` 中的 `PhoneType`。如果为自行指定默认值，系统将会为其设置默认值：数值类型将被设置成为0，字符串类型被设置成空字符串，布尔类型被设置成为 `false`。对于嵌入的消息，其默认值总是它默认的实例或者原型。


