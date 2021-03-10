# 介绍

`Core Foundation`内存管理使用`allocators`,引用计数机制和基于函数名约定的对象所有权策略。本文涵盖了对象创建、复制、`retaining`、`releasing`的相关技术。

内存管理是高效地使用`Core Foundation`的基础。本文档是所有使用`Core Foundation`的开发人员的必要阅读材料。

# 本文内容

下面的概念讨论了系统提供的`Core Foundation`对象的内存分配和回收：

* [Ownership Policy](https://developer.apple.com/library/archive/documentation/CoreFoundation/Conceptual/CFMemoryMgmt/Concepts/Ownership.html#//apple_ref/doc/uid/20001148-CJBEJBHH) (所有权策略)
* [Core Foundation Object Lifecycle Management](https://developer.apple.com/library/archive/documentation/CoreFoundation/Conceptual/CFMemoryMgmt/Articles/lifecycle.html#//apple_ref/doc/uid/TP40002439-SW1)生命周期管理）
* [Copy Functions](https://developer.apple.com/library/archive/documentation/CoreFoundation/Conceptual/CFMemoryMgmt/Concepts/CopyFunctions.html#//apple_ref/doc/uid/20001149-CJBEJBHH) (关于复制)
* [Allocators](https://developer.apple.com/library/archive/documentation/CoreFoundation/Conceptual/CFMemoryMgmt/Concepts/Allocators.html#//apple_ref/doc/uid/20001146-CJBEJBHH)

如果你需要自定义你的allocators，需要阅读:

* [Using Allocators in Creation Functions](https://developer.apple.com/library/archive/documentation/CoreFoundation/Conceptual/CFMemoryMgmt/Tasks/UsingAllocators.html#//apple_ref/doc/uid/20001152-CJBEHAAG)
* [Using the Allocator Context](https://developer.apple.com/library/archive/documentation/CoreFoundation/Conceptual/CFMemoryMgmt/Tasks/UsingAllocatorContext.html#//apple_ref/doc/uid/20001153-CJBEHAAG)
* [Creating Custom Allocators](https://developer.apple.com/library/archive/documentation/CoreFoundation/Conceptual/CFMemoryMgmt/Tasks/CustomAllocators.html#//apple_ref/doc/uid/20001154-CJBEHAAG)

要了解更多关于字节序和交换的信息，请参见:

- [Byte Ordering](https://developer.apple.com/library/archive/documentation/CoreFoundation/Conceptual/CFMemoryMgmt/Concepts/ByteOrdering.html#//apple_ref/doc/uid/20001150-CJBEJBHH)
- [Byte Swapping](https://developer.apple.com/library/archive/documentation/CoreFoundation/Conceptual/CFMemoryMgmt/Tasks/ByteSwapping.html#//apple_ref/doc/uid/20001155-CJBEHAAG)

相关内容:

* [Core Foundation Design Concepts](https://developer.apple.com/library/archive/documentation/CoreFoundation/Conceptual/CFDesignConcepts/CFDesignConcepts.html#//apple_ref/doc/uid/10000122i)

* [Advanced Memory Management Programming Guide](https://developer.apple.com/library/archive/documentation/Cocoa/Conceptual/MemoryMgmt/Articles/MemoryMgmt.html#//apple_ref/doc/uid/10000011i)

# Ownership Policy

使用`Core Foundation`的应用程序不断地访问、创建和释放对象。为了确保不会泄漏内存，`Core Foundation`定义了获取和创建对象的规则。

## 基础内容

当试图理解`Core Foundation` 应用程序中的内存管理时，不要从内存管理本身的角度考虑，而是从对象所有权的角度考虑，这是很有帮助的。一个对象可以有一个或多个所有者;它使用`retain count`记录所有者的数量。如果一个对象没有所有者(如果它的引用计数下降到零)，它将被释放。

`Core Foundation`为对象所有权和对象销毁定义了以下规则：

* 如果你创建了一个对象（无论是直接创建或从另一个对象中`copy`副本——参见 [The Create Rule](#TCR)），你就拥有（持有）了它
* 如果你从其他地方得到一个对象，你就不拥有它。如果你想防止它被销毁，你必须将自身成为对象的所有者(使用`CFRetain`)
* 如果你是一个对象的所有者，你必须在使用完它之后放弃它的所有权（使用`CFRelease`）

## Naming Conventions



## <span id = "TCR">The Create Rule</span>

 

## The Get Rule



## Instance Variables and Passing Parameters



## Ownership Examples



