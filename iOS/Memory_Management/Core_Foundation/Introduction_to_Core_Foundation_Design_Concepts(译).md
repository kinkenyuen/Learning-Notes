#  Introduction to Core Foundation Design Concepts

`Core Foundation`是C语言实现的编程接口库，这些接口从概念上源自基于`Objective-C`的`Foundation`框架。`Core Foundation`在C语言中实现了一个有限的对象模型。`Core Foundation`定义了封装数据和函数的`opaque types` ，以下称为"对象"。

`Core Foundation`对象的编程接口被设计为易于使用和重用。通常，`Core Foundation`:

* 支持在各种框架和库之间共享代码和数据
* 使某种程度上的操作系统独立性成为可能
* 使用Unicode字符串支持国际化
* 提供通用API和其他有用的功能，包括plug-in architecture、XML property lists和preferences

`Core Foundation`可以让OS X上的不同框架和库共享代码和数据。应用程序、库和框架可以定义C例程，将`Core Foundation`类型合并到外部接口中;因此，它们可以通过这些接口相互通信数据(作为`Core Foundation`对象)。

`Core Foundation`还在某些服务和`Cocoa`的`Foundation`框架之间提供了"`toll-free bridging`"。`toll-free bridging`使你能够在参数中用`Cocoa`对象替换`Core Foundataion`对象，反之亦然。

一些`Core Foundataion`类型和函数是在不同操作系统上具有特定实现的事物的抽象。因此，使用这些api的代码更容易移植到不同的平台上。

> Date and number types abstract time utilities and offers facilities for converting between absolute and Gregorian measures of time. It also abstracts numeric values and provides facilities for converting between different internal representations of those values.

`Core Foundation`给应用程序开发带来的主要好处之一是国际化支持。通过它的字符串对象，`Core Foundation`促进了跨所有OS X和`Cocoa`编程接口和实现简单、健壮和一致的国际化。这种支持的主要部分是一个`CFString`类型，它的实例表示一个16位`Unicode`字符数组。`CFString`对象足够灵活，可以容纳数兆字节的字符，而且足够简单和底层，可以在通信字符数据的所有编程接口中使用。它的性能与标准C字符串的性能相差无几。

你应该阅读这篇文档来了解`Core Foundation`的基本设计原则，以及`Core Foundation`对象如何与`Cocoa (Touch)`对象交互。

## 文档内容

这些概念和任务讨论了`Core Foundation`中使用的对象模型:

- [Opaque Types](https://developer.apple.com/library/archive/documentation/CoreFoundation/Conceptual/CFDesignConcepts/Articles/OpaqueTypes.html#//apple_ref/doc/uid/20001106-CJBEJBHH)
- [Object References](https://developer.apple.com/library/archive/documentation/CoreFoundation/Conceptual/CFDesignConcepts/Articles/ObjectReferences.html#//apple_ref/doc/uid/20001107-CJBEJBHH)
- [Polymorphic Functions](https://developer.apple.com/library/archive/documentation/CoreFoundation/Conceptual/CFDesignConcepts/Articles/PolymorphicFunctions.html#//apple_ref/doc/uid/20001108-CJBEJBHH)
- [Varieties of Objects](https://developer.apple.com/library/archive/documentation/CoreFoundation/Conceptual/CFDesignConcepts/Articles/VarietyOfObjects.html#//apple_ref/doc/uid/20001109-CJBEJBHH)
- [Comparing Objects](https://developer.apple.com/library/archive/documentation/CoreFoundation/Conceptual/CFDesignConcepts/Articles/Comparing.html#//apple_ref/doc/uid/20001112-CJBEHAAG)
- [Inspecting Objects](https://developer.apple.com/library/archive/documentation/CoreFoundation/Conceptual/CFDesignConcepts/Articles/Inspecting.html#//apple_ref/doc/uid/20001113-CJBEHAAG)

此外，在使用`Core Foundation`之前，你应该熟悉其他非对象类型和API约定:

- [Naming Conventions](https://developer.apple.com/library/archive/documentation/CoreFoundation/Conceptual/CFDesignConcepts/Articles/NamingConventions.html#//apple_ref/doc/uid/20001110-CJBEJBHH)
- [Other Types](https://developer.apple.com/library/archive/documentation/CoreFoundation/Conceptual/CFDesignConcepts/Articles/OtherTypes.html#//apple_ref/doc/uid/20001111-CJBEJBHH)
- [Toll-Free Bridged Types](https://developer.apple.com/library/archive/documentation/CoreFoundation/Conceptual/CFDesignConcepts/Articles/tollFreeBridgedTypes.html#//apple_ref/doc/uid/TP40010677-SW1)

# Opaque Types(不透明类型) 

`Opaque Types` 意思为不透明类型，意思是我们可以接触到该类型，但是不知道它内部是如何运作的。

`Core Foundation`对象模型支持封装和多态函数，是基于不透明类型的。

基于不透明类型的对象的各个字段对我们来说是隐藏的，但是该类型的函数提供了对这些字段的大多数值的访问。图1描述了它"隐藏"的数据和它呈现给我们的接口中的不透明类型。

> 注意："Class"并不是指不透明类型，尽管Class和不透明类型在概念上相似，许多人可能会发现这个术语令人困惑。是因为Core Foundation文档经常将这些类型的特定的、承载数据的实例称为"对象"。

`Core Foundation`有许多不透明的类型，这些类型的名称反映了它们的预期用途。例如，`CFString`是一种不透明类型，它代表并操作`Unicode`字符数组。(`CF`当然是`Core Foundation`的前缀。)`CFArray`是用于基于索引的集合功能的不透明类型。支持不透明类型的函数、常量和其他辅助数据类型通常在具有该类型名称的头文件中定义;例如，`CFArray.h`包含`CFArray`类型的符号定义。

图1 不透明类型

![1](./imgs/opaquetypes_2x.png)

## Advantages of Opaque Types(不透明类型的优点)

**在某些情况下，不透明类型可能会阻止直接访问结构内容，从而似乎施加了不必要的限制**。似乎还会有与可能影响程序性能的不透明类型相关的开销。但不透明类型的好处超过了这些表面上的限制。

不透明类型为底层功能的实现提供了更好的抽象和更大的灵活性。通过隐藏细节(如结构的字段)，`Core Foundation`减少了在这些细节更改时客户端代码中可能发生错误的机会。而且，不透明类型允许进行优化，如果公开这些优化可能会造成混淆。例如，`CFString`表示`UniChar`类型的16位字符数组。但是，`CFString`可以选择将`ASCII`范围内的一段字符存储为8位值。复制一个不可变对象可能会(通常也会)导致对该对象的共享引用，而不是内存中的两个独立对象(参见[Memory Management Programming Guide for Core Foundation](https://developer.apple.com/library/archive/documentation/CoreFoundation/Conceptual/CFMemoryMgmt/CFMemoryMgmt.html#//apple_ref/doc/uid/10000127i))。

继续以`CFString`为例，使用不透明类型来存储字符似乎很重要。然而，事实证明，这种存储的CPU成本并不比使用简单的C字符数组高多少，内存成本通常更低。此外，不透明并不一定意味着不透明类型永远不能提供直接访问内容的机制。例如，`CFString`提供了`CFStringGetCStringPtr`函数来实现此目的。

最后，你可以在某种程度上自定义一些不透明类型。例如，集合类型允许你为集合中的元素添加回调。

# Object References

# Polymorphic Functions

# Varieties of Objects

# Naming Conventions

# Other Types

# Comparing Objects

# Inspecting Objects

# Toll-Free Bridged Types

## Casting and Object Lifetime Semantics

## Toll-Free Bridged Types

