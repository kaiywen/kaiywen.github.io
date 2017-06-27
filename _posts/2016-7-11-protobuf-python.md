---
layout:     post
title:      "Protocol Buffer 基础：Python"
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

## 前言

这篇文章为 Python 编程者提供了一个使用 protobuf 的基本教程。下文通过创建一个简单的应用，向你展示了三点主要内容：

* 怎样在一个 `.proto` 文件中定义消息（Message）格式
* 怎样使用 protobuf 编译器
* 怎样利用 protobuf 的 API 来读写消息

这并不是一个在 Python 中使用 protobuf 的完整的教程。如果你需要更加详细的信息，可以查看  [Protocol Buffer Language Guide](https://developers.google.com/protocol-buffers/docs/proto)，[Python API Reference](https://developers.google.com/protocol-buffers/docs/reference/python/index.html)，[Python Generated Code Guide](https://developers.google.com/protocol-buffers/docs/reference/python-generated) 以及 [Encoding Reference](https://developers.google.com/protocol-buffers/docs/encoding)。

## 为什么要使用 Protobuf ？

我们接下来将使用一个非常简单的『地址簿』应用，该应用将能够从一个文件中读取某人的联系方式或将联系方式写回到文件中。在地址簿中的每个人都有一个姓名，ID，email 以及联系电话。

<!--more-->

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

每一个 `.proto` 文件都由一个包声明开头，同其他语言一样，这种包声明能够有效地避免不同工程之间的命名冲突。在 Python 中，包自然而然地由目录结构所定义，因此你在 `.proto` 所定义的 `package` 名对生成的代码并没有效果。然而，在 Python 中你仍然需要像其他语言一样定义一个包声明以避免命名冲突。

接下来是消息（Message）定义，每一个消息都包含了一系列具有类型的字段。Protobuf 提供了需要简单的数据类型来作为字段类型的定义，包括 `bool`, `int32`, `float`, `double` 以及 `string`。你也可以使用其他的消息类型类作为某一个字段的类型，例如在上面的例子中，`Person` 消息中 包含有一个 `PhoneNumber` 消息，同时 `AddressBook` 消息中包含有 `Person` 消息。如同你所看到的，你可以在消息体内定义其他的消息类型，例如 `Person` 消息体内的 `PhoneNumber` 消息。如果你需要你的字段拥有一些提前定义的值，你也可以使用 `enum` 来定义枚举类型。这里我们希望电话类型 `PhontType` 可以是 `MOBILE`, `HOME` 以及 `WORK` 中的一种。

上述代码中的 `=1`，`=2` 代表不同的字段在二进制编码过程中的唯一标签。相比于较大的数据，标签 1-15 需要更少的字节来进行编码。因此处于优化方面的考虑，你可以将标签 1-15 用于哪些经常使用或者重复使用的字段，而将 16 以上的标签分配给较少使用的可选的字段。每一个 repeated 字段都需要重编码其标签，因此重复的字段非常适合分配较小的标签数字来优化性能。

消息中的每一个字段都必须通过如下的修饰符来标识：

* `required`：表明该字段必须提供值，否则该消息将会被认为是『未初始化』。尝试序列化一个未初始化的消息将会抛出一个异常，而尝试解码一个未初始化的消息将会直接失败。除了这点以外，一个 `required` 的字段和一个 `optional` 字段的表现完全一样。
* `optional`：该字段可以设置或者不设置值。如果没有对 `optional` 的字段设置值，那么它将自动拥有一个默认的值。对于简单的类型，你可以自行指定默认值，例如上述代码中 `PhoneNumber` 中的 `PhoneType`。如果为未自行指定默认值，系统将会为其设置默认值：数值类型将被设置成为 0，字符串类型被设置成空字符串，布尔类型被设置成为 `false`。对于嵌入的消息类型，其默认值总是它默认的实例或者原型。默认的实例中所有的字段都未设置值。当获取一个没有被显示设置值的 `optinal` 字段时总是会返回这个字段的默认值。
* `repeated`：这种字段可以将值重复设置并连接起来，并且这些值在 protobuf 中的顺序将会得以保存。你可以将 `repeated` 字段看成是一种具有动态容量的数组。

<p class="caution"><strong>Required 字段注意事项</strong>：你应该非常谨慎的将字段定义为 <code>required</code>. 如果在后续的某个时刻，你不想再使用一个 <code>required</code> 字段，那么直接将其改为 <code>optional</code> 将会引起问题 -- 旧的协议读取代码将会认为不带有该字段的消息是不完全的并且将会拒绝或者无意地丢弃它们。事实上，你应该为你的 buffers 编写一些应用相关的定制的验证规则。Google 的一些工程师甚至认为使用 <code>required</code> 字段弊大于利，他们倾向于仅仅使用 <code>optional</code> 和 <code>repeated</code> 字段。然而，这种说法并没有达成普遍共识。</p>

在教程 [Protocol Buffer Language Guide](https://developers.google.com/protocol-buffers/docs/proto)里，你将可以学到所有有关 `.proto` 文件的写法。不要尝试着去寻找类继承之类的语法，protobuf 不支持那一套。

## 编译 Protocol Buffers

现在已经有了一个 `.proto` 文件，接下来需要做的是生成一个你可以读写 `AddressBook` 的类。为了完成这点，你需要运行 protobuf 的编译器 `protoc` 来编译你的 `.proto` 文件：

1. 如果你还没有安装 `protoc` 编译器，你可以在[这里](https://developers.google.com/protocol-buffers/docs/downloads.html)下载并且按照指导安装它。
2. 接下来你可以运行编译器，你需要指定你的源文件目录（即你的 `.proto` 文件所在的目录，如果未指定，将会使用当前的目录），生成类的目标文件目录以及你的 `.proto` 的文件路径，例如在本例中，你可以使用输出选项 `--python_out` 生成 Python 中的类：

```shell
    protoc -I=$SRC_DIR --python_out=$DST_DIR $SRC_DIR/addressbook.proto
```

## Protobuf API

不像你使用 Java 或者 C++ 的 protobuf 代码一样，Python 的 protobuf 编译器并不会直接为你生成存取数据的代码。相反的，它为你的消息，枚举类型以及一些莫名其妙的空类产生一种特殊的描述符，每一种消息都有其自己的描述符：

```python
class Person(message.Message):
    __metaclass__ = reflection.GeneratedProtocolMessageType

    class PhoneNumber(message.Message):
        __metaclass__ = reflection.GeneratedProtocolMessageType
        DESCRIPTOR = _PERSON_PHONENUMBER
    DESCRIPTOR = _PERSON

class AddressBook(message.Message):
    __metaclass__ = reflection.GeneratedProtocolMessageType
    DESCRIPTOR = _ADDRESSBOOK
```

每一行中的 `__metaclass__ = reflection.GeneratedProtocolMessageType` 非常重要，它们控制了 Python 中的元类（Metaclass）构建 Python 类 （Class）的方式。在代码装载时，元类 `GeneratedProtocolMessageType` 将会使用指定的描述符来创建那些与你需要使用的消息相关的 Python 方法，并且将他们放到 Python 类中，之后你就可以在你的代码中使用这些生成的 Python 类代码。

最终的效果就是，在 Python 代码中，你可以向 Message 中定义字段的结构一样来使用这些类，例如：

```python
import addressbook_pb2
person = addressbook_pb2.Person()
person.id = 1234
person.name = "John Doe"
person.email = "jdoe@example.com"
phone = person.phone.add()
phone.number = "555-4321"
phone.type = addressbook_pb2.Person.HOME
``` 

注意上述这些代码并不仅仅只是将一些新的字段加入到 Python 对象中。如果你尝试添加一个未在 `.proto` 文件中定义的字段，程序将会抛出 `AttributeError`，如果你尝试为一个字段指定错误的类型，代码也将抛出 `TypeError` 异常。同时，获取一个未被设置的字段将会得到其默认值。

```python
person.no_such_field = 1  # raises AttributeError
person.id = "1234"        # raises TypeError
```

如果你想要了解 protobuf 编译器针对某种特殊的字段定义将会产生什么样的类成员，可以查看 [Python generated code reference](https://developers.google.com/protocol-buffers/docs/reference/python-generated).

#### 枚举类型

枚举类型将会被元类扩展成为一组有符号的整型常数。因此例子中的常数 `addressbook_pb2.Person.Work` 将被赋予常数值 2.

#### 标准的消息方法

每一个生成的类也包含了其他一些方法来供你检查或者操纵整个消息，这些方法包括：

* `IsInitialized()`：检查是否所有的 `required` 字段都被设置相应值。
* `__str__()`：返回一个易于人类阅读的消息字符串。该函数非常适合 Bug 调试。（例如使用 `str(message)` 或者 `print message`.）
* `CopyFrom(other_msg)`：将消息内容改写为 `other_msg` 的内容。
* `Clear()`：将消息中的所有元素清空到空状态。

对于更多的函数，请查看 [complete API documentation for Message](https://developers.google.com/protocol-buffers/docs/reference/python/google.protobuf.message.Message-class)

#### 读取消息和序列化消息

每一个 protobuf 类都提供读取或者保存消息的函数，这些函数可以操纵使用 protobuf 的二进制格式定义好的类型：

* `SerializeToString()`：将消息序列化成为一个字符串 `string`。注意序列化最终的结果其实是二进制的字节串，我们仅仅使用 `str` 作为一个合适的包装而已。
* `ParseFromString(data)`：将一个给定的字符串转化成为消息。

针对解码和序列化这两个函数有一系列的参数，你可以查看 [Message API Reference](https://developers.google.com/protocol-buffers/docs/reference/python/google.protobuf.message.Message-class) 获取详细信息。

<p class="caution"> <strong> Protobuf 和 面向对象设计：</strong> 需要注意 protobuf 生成的消息类仅仅作为一种数据的包装（类似于 C++ 语言中的 struct）；他们在对象模型中并不算做一等公民。如果你想要让产生的消息类具有更多的行为，最好的方式就是将他们包装在一个应用相关的类中。当你不能控制<code> .proto </code>文件的构建时（例如你复用了别人的工程代码），自行包装一个生成的消息类是一个非常不错的办法。这允许你隐藏一些数据和方法并暴露出一些更加易用的函数接口。<strong>你绝对不能通过继承生成的消息类来为他们扩展更多的行为。</strong>这种做法将会破坏内部机制并且不是一个好的面向对象的做法。</p>

## 编写一个消息

现在你可以尝试使用已经生成的消息类。首先最想要完成的第一个动作就是陷入个人信息到你的地址簿中。为了达到目的，你可以创建并且持有你的 protobuf 消息类的实例，并且将他们写入到一个输出管道中（可以是文件或者网络）。

下面是一个从文件中读取 `AddressBook` 的例子，它基于用户的输入创建一个新的 `Person` 对象，并且将新的 `AddressBook` 实例写回到文件中。

```python
#! /usr/bin/python

import addressbook_pb2
import sys

# This function fills in a Person message based on user input.
def PromptForAddress(person):
  person.id = int(raw_input("Enter person ID number: "))
  person.name = raw_input("Enter name: ")

  email = raw_input("Enter email address (blank for none): ")
  if email != "":
    person.email = email

  while True:
    number = raw_input("Enter a phone number (or leave blank to finish): ")
    if number == "":
      break

    phone_number = person.phone.add()
    phone_number.number = number

    type = raw_input("Is this a mobile, home, or work phone? ")
    if type == "mobile":
      phone_number.type = addressbook_pb2.Person.MOBILE
    elif type == "home":
      phone_number.type = addressbook_pb2.Person.HOME
    elif type == "work":
      phone_number.type = addressbook_pb2.Person.WORK
    else:
      print "Unknown phone type; leaving as default value."

# Main procedure:  Reads the entire address book from a file,
#   adds one person based on user input, then writes it back out to the same
#   file.
if len(sys.argv) != 2:
  print "Usage:", sys.argv[0], "ADDRESS_BOOK_FILE"
  sys.exit(-1)

address_book = addressbook_pb2.AddressBook()

# Read the existing address book.
try:
  f = open(sys.argv[1], "rb")
  address_book.ParseFromString(f.read())
  f.close()
except IOError:
  print sys.argv[1] + ": Could not open file.  Creating a new one."

# Add an address.
PromptForAddress(address_book.person.add())

# Write the new address book back to disk.
f = open(sys.argv[1], "wb")
f.write(address_book.SerializeToString())
f.close()
```

## 读取一个消息

当然，如果你不能从地址簿中读取消息，那么你的地址簿也并没有什么X用。下面的例子展示了如何从上述代码创建的文件中读取信息并且打印：

```python
#! /usr/bin/python

import addressbook_pb2
import sys

# Iterates though all people in the AddressBook and prints info about them.
def ListPeople(address_book):
  for person in address_book.person:
    print "Person ID:", person.id
    print "  Name:", person.name
    if person.HasField('email'):
      print "  E-mail address:", person.email

    for phone_number in person.phone:
      if phone_number.type == addressbook_pb2.Person.MOBILE:
        print "  Mobile phone #: ",
      elif phone_number.type == addressbook_pb2.Person.HOME:
        print "  Home phone #: ",
      elif phone_number.type == addressbook_pb2.Person.WORK:
        print "  Work phone #: ",
      print phone_number.number

# Main procedure:  Reads the entire address book from a file and prints all
#   the information inside.
if len(sys.argv) != 2:
  print "Usage:", sys.argv[0], "ADDRESS_BOOK_FILE"
  sys.exit(-1)

address_book = addressbook_pb2.AddressBook()

# Read the existing address book.
f = open(sys.argv[1], "rb")
address_book.ParseFromString(f.read())
f.close()

ListPeople(address_book)
```

## 扩展 protobuf

在你以后的代码中，你肯定想要扩展你的 protobuf 的定义。如果你想要让你新定义的 protobuf 能够后向兼容，并且让你的旧定义前向兼容，你需要注意一些事项。在你新版本的 protobuf 定义中：

* 你绝对不能改变任何一个已经存在的字段的标签号。
* 你绝对不能删除或者添加任何 `required` 字段。
* 你可以删除 `optional` 或者 `repeated` 字段。
* 你可以添加新的 `optional` 或 `repeated` 字段，但是必须分配给他们新的标签号。（『新标签号』指的是从来没有用过的标签号，即使是已经删除的字段的标签号也不能重复使用）。

（尽管上述的规则有一些[例外](https://developers.google.com/protocol-buffers/docs/proto.html#updating)，但是这些例外基本可以忽略）

如果你遵循这些规则，你的旧代码将可以读取你的新消息并忽略掉新定义的字段。对于旧的代码来说，那些已经删除的 `optional` 字段将简单地被设为他们的默认值，那些删除的 `repeated` 字段将会成为空值。新的代码也能够无缝的读取旧的消息。然而，需要记住的是新的 `optional` 字段将不会出现在旧的消息当值，因此你要么使用 `has_` 方法显示的检查是否有该字段，要么在你的 `.proto` 文件中为其设置默认值。如果新的 `optional` 字段并没有被指定默认值，那么默认值将被自动设定。注意如果你添加了一个新的 `repeated` 字段，你的代码可能不能分辨出该字段是被设置为空（新代码）还是重来都没有设置过（旧代码）。





