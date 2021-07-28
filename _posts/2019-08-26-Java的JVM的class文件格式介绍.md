---
layout:     post
title:      Java的JVM的class文件格式介绍
subtitle:   介绍Java的虚拟机的基本知识
date:       2019-08-26
author:     qin4zhang
header-img: img/post-bg-universe.jpg 
catalog: true 						# 是否归档
tags:								#标签
    - Java
    - JVM
---
# 注意
> 想法及时记录，实现可以待做。

要说明什么是JVM，无疑看官方文档是最好的方式，看看JVM规范。<a href="https://docs.oracle.com/javase/specs/jvms/se8/html/index.html" target="_blank">官方文档</a>

## JVM介绍

Java虚拟机是Java平台的基石。它是负责其硬件和操作系统独立性、编译代码的小尺寸以及保护用户免受恶意程序的能力的技术组成部分。

Java虚拟机对Java编程语言一无所知，只知道一种特殊的二进制格式，即类文件格式。类文件包含Java虚拟机指令（或字节码）和符号表以及其他辅助信息。

为了安全起见，Java虚拟机对类文件中的代码施加了很强的语法和结构约束。但是，任何具有可以以有效类文件表达的功能的语言都可以由Java虚拟机托管。

对于JDK、JRE、JVM的关系，也说明下。JRE包括了JVM和Java所需的一些类库等。JDK除了包含JRE之外，还包括了一些开发、诊断工具。

一份Java源码，编译成字节码文件（class文件）后，通过JVM即可在各个操作系统中运行，由JVM负责与操作系统交互，而不是由具体的语言，这也是"Write Once, Run Anywhere"。

其实不仅对于Java这一种语言，只要能通过编译器编译成符合规范的字节码文件，任何语言都是可以运行在JVM上，常见的语言还有Scala、Groovy等等。

## class文件格式

本章描述了Java虚拟机的类文件格式。每个类文件包含单个类或接口的定义。尽管类或接口不需要包含在文件中的外部表示形式(例如，因为类是由类装入器生成的)，但我们通常将类或接口的任何有效表示形式称为类文件格式。

一个类文件由一个8位字节的流组成。所有16位、32位和64位的数量都是通过分别读取两个、四个和八个连续的8位字节来构造的。

多字节数据项总是以大端顺序存储，其中高字节放在前面。在Java SE平台中，这种格式由接口`java.io.DataInput`和`java.io.DataOutput`以及类如`java.io.DataInputStream`和`java.io.DataOutputStream`支持。

本章定义了它自己的一组表示类文件数据的数据类型:类型u1、u2和u4分别表示无符号的1字节、2字节或4字节数量。

在Java SE平台中，这些类型可以通过接口`java.io.DataInput`的`readUnsignedByte`、`readUnsignedShort`和`readInt`等方法读取。

本章介绍了用类c结构表示法编写的伪结构的类文件格式。为了避免与类的字段和类实例等混淆，描述类文件格式的结构的内容被称为项。连续项按顺序存储在类文件中，没有填充或对齐。

`Tables`由零个或多个可变大小的项组成，在几个类文件结构中使用。尽管我们使用类似c的数组语法来引用表项，但表是不同大小结构的流这一事实意味着不可能将表索引直接转换为表的字节偏移量。

在我们将数据结构称为数组的地方，它由零个或多个连续的固定大小的项组成，并且可以像数组一样建立索引。

本章中对ASCII字符的引用应该解释为对应于该ASCII字符的Unicode码位。

1. ClassFile结构

一个类文件由一个单独的ClassFile结构组成:

```
ClassFile {
    u4             magic;
    u2             minor_version;
    u2             major_version;
    u2             constant_pool_count;
    cp_info        constant_pool[constant_pool_count-1];
    u2             access_flags;
    u2             this_class;
    u2             super_class;
    u2             interfaces_count;
    u2             interfaces[interfaces_count];
    u2             fields_count;
    field_info     fields[fields_count];
    u2             methods_count;
    method_info    methods[methods_count];
    u2             attributes_count;
    attribute_info attributes[attributes_count];
}
```

ClassFile结构中的主要项如下:

- magic

magic提供识别类文件格式的魔术数字;它的值是0xCAFEBABE。

- access_flags

access_flags项的值是用于表示该类或接口的访问权限和属性的标志的掩码。

|Flag Name|Value|含义|
|---|---|---|
|ACC_PUBLIC|	0x0001|	Declared public; may be accessed from outside its package.|
|ACC_FINAL|	0x0010|	Declared final; no subclasses allowed.|
|ACC_SUPER|	0x0020|	Treat superclass methods specially when invoked by the  `invokespecial` instruction.|
|ACC_INTERFACE|	0x0200|	Is an interface, not a class.|
|ACC_ABSTRACT|	0x0400|	Declared abstract; must not be instantiated.|
|ACC_SYNTHETIC|	0x1000|	Declared synthetic; not present in the source code.|
|ACC_ANNOTATION|	0x2000|	Declared as an annotation type.|
|ACC_ENUM|	0x4000|	Declared as an enum type.|

接口通过设置`ACC_INTERFACE`标志来区分。如果没有设置`ACC_INTERFACE`标志，则这个类文件定义了一个类，而不是一个接口。

如果设置了`ACC_INTERFACE`标志位，则必须同时设置`ACC_ABSTRACT`标志位，不能设置`ACC_FINAL`、`ACC_SUPER`和`ACC_ENUM`标志位。

如果没有设置`ACC_INTERFACE`标志，上述表中除了`ACC_ANNOTATION`外的其他标志都可以设置。但是，这样的类文件不能同时设置`ACC_FINAL`和`ACC_ABSTRACT`标志。

`ACC_SUPER`标志指出，如果`invokespecial`指令出现在这个类或接口中，那么这两种语义中哪一种会被`invokespecial`指令表示。

Java虚拟机指令集的编译器应该设置`ACC_SUPER`标志。在Java SE 8及以上版本中，Java虚拟机认为`ACC_SUPER`标志要在每个类文件中设置，而不考虑类文件中标志的实际值和类文件的版本。

`ACC_SYNTHETIC`标志表示这个类或接口是由编译器生成的，没有出现在源代码中。

注释类型必须有`ACC_ANNOTATION`标记集。如果设置了`ACC_ANNOTATION`标志，那么`ACC_INTERFACE`标志也必须设置。

`ACC_ENUM`标志表示这个类或它的父类被声明为枚举类型。

上述表中未分配的`access_flags`项的所有位都保留以供将来使用。它们应该在生成的类文件中设置为零，并且应该被Java虚拟机实现忽略。





2. 名称的内部形式


3. 描述符


4. 常量池


5. 字段


6. 方法


7. 属性


8. 格式检查


9. Java虚拟机代码约束



10. 类文件的验证




11. Java虚拟机的限制


## 加载、链接、初始化



## JVM指令集



## 参考

1. <a href="https://docs.oracle.com/javase/specs/jvms/se8/html/index.html" target="_blank">官方文档</a>
