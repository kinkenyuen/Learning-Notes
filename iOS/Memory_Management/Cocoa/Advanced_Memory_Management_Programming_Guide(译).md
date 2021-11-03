# 关于内存管理

应用程序内存管理是在程序运行时分配内存、使用内存以及在使用完内存后释放内存的过程。写得好的程序使用的内存越少越好。在Objective-C中，它也可以被看作是一种将有限内存资源的所有权分配数据和代码的方式。当你阅读完这篇指南，你就会掌握内存管理的知识。你所需要做的是**显式地管理对象的生命周期，并在不再需要它们时释放它们**。

尽管内存管理通常是针对单个对象而言，但是实际上你需要管理的是一个**对象图**。简而言之，对象图是一组对象通过它们彼此之间的引用关系形成一个网络。通常而言，你期望确保您的进程内存中不会有您不需要的对象。

![1](./imgs/memory_management_2x.png)

# 概览

Objective-C提供了两种应用程序内存管理方法。

1. `"manual retain-release" or MRR`，你需要显式管理你拥有的对象的寿命周期。这是使用一个模型来实现的，这个模型被称为**引用计数**，它是基础类`NSObject`与运行时环境一起提供的。
2. 在自动引用计数(`ARC`)中，系统使用与`MRR`相同的引用计数系统，但它在编译时为您插入适当的内存管理方法调用。强烈建议您在新项目中使用`ARC`。如果您使用`ARC`，通常不需要理解本文档中描述的底层实现，尽管它在某些情况下可能会有帮助。有关更多`ARC`信息，参见[Transitioning to ARC Release Notes](https://developer.apple.com/library/archive/releasenotes/ObjectiveC/RN-TransitioningToARC/Introduction/Introduction.html#//apple_ref/doc/uid/TP40011226)

## 良好的习惯可避免内存问题

错误的内存管理主要有两种：

1. 提前释放或覆盖正在使用的内存数据，这会导致内存损坏，甚至程序崩溃。
2. 不再使用的内存数据没有释放，从而导致内存泄漏。内存泄漏导致应用程序使用越来越多的内存，从而可能导致系统性能低下或应用程序被终止。

然而，从引用计数的角度考虑内存管理常常适得其反，因为您倾向于内存管理的实现细节，而不是根据您自身的实际目标来考虑。**相反，您应该从对象所有权和对象图的角度来考虑内存管理**。

**Cocoa使用简单的命名约定来表示您何时拥有由方法返回的对象**。参见[Memory Management Policy](https://developer.apple.com/library/archive/documentation/Cocoa/Conceptual/MemoryMgmt/Articles/mmRules.html#//apple_ref/doc/uid/20000994-BAJHFBGH).

尽管内存管理策略很简单，但您可以采取一些实际措施，使内存管理更容易，并帮助确保您的程序保持可靠和健壮，同时最小化其资源需求。参见[Practical Memory Management](https://developer.apple.com/library/archive/documentation/Cocoa/Conceptual/MemoryMgmt/Articles/mmPractical.html#//apple_ref/doc/uid/TP40004447-SW1).

自动释放池块提供了一种机制，您可以通过这种机制向对象发送"延迟的"释放消息。**当您想要放弃对象的有权，但又想避免立即释放它时(例如当您从一个方法返回一个对象时)，这是非常有用的**。在某些情况下，您可能会使用自己的自动释放池。参见[Using Autorelease Pool Blocks](https://developer.apple.com/library/archive/documentation/Cocoa/Conceptual/MemoryMgmt/Articles/mmAutoreleasePools.html#//apple_ref/doc/uid/20000047-CJBFBEDI).

## 使用分析工具调试内存问题

要在编译时识别代码的问题，你可以使用Xcode内置的Clang静态分析器。

如果出现内存管理问题，您可以使用其他工具和技术来识别和诊断问题。

* 在Technical Note TN2239, [iOS Debugging Magic](https://developer.apple.com/library/archive/technotes/tn2239/_index.html#//apple_ref/doc/uid/DTS40010638)中提到了很多工具和技术，特别是使用`NSZombie`来帮助定位`over-released`对象
* 您可以使用`Instruments`来跟踪引用计数相关信息以及查看内存泄漏，参见[Collecting Data on Your App](https://developer.apple.com/library/archive/documentation/DeveloperTools/Conceptual/InstrumentsUserGuide/TheInstrumentsWorkflow.html#//apple_ref/doc/uid/TP40004652-CH5)

# 源文档

[About Memory Management](https://developer.apple.com/library/archive/documentation/Cocoa/Conceptual/MemoryMgmt/Articles/MemoryMgmt.html#//apple_ref/doc/uid/10000011-SW1)
