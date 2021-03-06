# 关于内存管理

应用程序内存管理是在程序运行时分配内存、使用内存以及在使用完内存后释放内存的过程。写得好的程序使用的内存越少越好。在Objective-C中，它也可以被看作是一种将有限内存资源的所有权分配数据和代码的方式。当你阅读完这篇指南，你就会掌握内存管理的知识。你所需要做的是**显式地管理对象的生命周期，并在不再需要它们时释放它们**。

尽管内存管理通常是针对单个对象而言，但是实际上你需要管理的是一个**对象图**。简而言之，对象图是一组对象通过它们彼此之间的引用关系形成一个网络。通常而言，你期望确保内存中不会有你不需要的对象。

![1](./imgs/memory_management_2x.png)

# 概览

Objective-C提供了两种应用程序内存管理方法。

1. `“manual retain-release” or MRR`，你需要显式管理你拥有的对象的寿命周期。这是使用一个模型来实现的，这个模型被称为引用计数，它是基础类`NSObject`与运行时环境一起提供的。
2. 在自动引用计数(`ARC`)中，系统使用与`MRR`相同的引用计数系统，但它在编译时为您插入适当的内存管理方法调用。强烈建议您在新项目中使用`ARC`。如果您使用`ARC`，通常不需要理解本文档中描述的底层实现，尽管它在某些情况下可能会有帮助。有关更多`ARC`信息，参见[Transitioning to ARC Release Notes](https://developer.apple.com/library/archive/releasenotes/ObjectiveC/RN-TransitioningToARC/Introduction/Introduction.html#//apple_ref/doc/uid/TP40011226)

