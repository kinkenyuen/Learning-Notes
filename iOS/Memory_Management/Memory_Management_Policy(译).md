# 内存管理策略

在引用计数环境中用于内存管理的基本模型，是由`NSObject`协议和**标准方法命名约定**提供定义的。`NSObject`类也定义了一个方法`dealloc`，它在一个对象被释放时被自动调用。本文描述了在Cocoa程序中正确管理内存所需知道的所有基本规则，并提供了一些正确使用的示例。

# 基本内存管理规则

内存管理模型是**基于对象所有权**的。任何对象都可以有一个或多个所有者。**只要对象至少有一个所有者，它就继续存在。如果对象没有所有者，运行时系统会自动销毁它**。为了确保明确你什么时候拥有一个对象，什么时候不拥有，Cocoa定义了以下策略:

* **You own any object you create** （你拥有你创建的任何对象）

  例如，你使用名称以"`alloc`"、"`new`"、"`copy`"或"`mutableCopy`"开头的方法来创建对象

* **You can take ownership of an object using retain** （你可以使用`retain`获得对象的所有权）

  > A received object is normally guaranteed to remain valid within the method it was received in, and that method may also safely return the object to its invoker. (这句翻译过来有点拗口，就引用原文)

  你可以在以下两种情况下使用`retain`:

  * 在实现访问器方法或init方法时，要获取要存储的对象的所有权作为属性值
  * 防止对象在某些操作下被释放 (参见解释[Avoid Causing Deallocation of Objects You’re Using](https://developer.apple.com/library/archive/documentation/Cocoa/Conceptual/MemoryMgmt/Articles/mmPractical.html#//apple_ref/doc/uid/20000043-1000922))

* **When you no longer need it, you must relinquish ownership of an object you own** （当你不再需要某对象时，你必须放弃你所拥有的对象的所有权）

  你可以调用`release`方法或`autorelease`方法来放弃对象的所有权。因此，在Cocoa术语中，放弃对象的所有权通常被称为“`releasing`”一个对象

* **You must not relinquish ownership of an object you do not own** （你不能放弃你不拥有的对象的所有权）

  根据前面描述的规则，这一点是必然的

