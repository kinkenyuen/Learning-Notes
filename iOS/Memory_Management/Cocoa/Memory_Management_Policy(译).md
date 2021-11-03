# 目录
 * [内存管理策略](#内存管理策略)
 * [基本内存管理规则](#基本内存管理规则)
    * [一个简单的例子](#一个简单的例子)
    * [使用autorelease标记延迟释放](#使用autorelease标记延迟释放)
    * [你不拥有引用返回的对象](#你不拥有引用返回的对象)
 * [实现dealloc以放弃对象的所有权](#实现dealloc以放弃对象的所有权)
 * [Core Foundation使用类似但不同的规则](#core-foundation使用类似但不同的规则)
 * [源文档](#源文档)

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

  你可以调用`release`方法或`autorelease`方法来放弃对象的所有权。因此，在Cocoa术语中，放弃对象的所有权通常被称为`releasing`一个对象

* **You must not relinquish ownership of an object you do not own** （你不能放弃你不拥有的对象的所有权）

  根据前面描述的规则，这一点是必然的

## 一个简单的例子

为了解释上述的策略，请看下面代码段:

```objective-c
{
    Person *aPerson = [[Person alloc] init];
    // ...
    NSString *name = aPerson.fullName;
    // ...
    [aPerson release];
}
```

`Person`对象是使用`alloc`方法创建的，因此当不再需要它时，给它发送一个`release`消息。`person`的`fullName`属性，没有通过所有权方法来获取，所以不需要给它发送`release`消息。但是请注意，这个示例使用的是`release`而不是`autorelease`。

## 使用autorelease标记延迟释放

**当你需要发送一个延迟释放的消息，你可以使用`autorelease`——特别是当你从一个方法中返回一个对象**。例如，你可以像以下那样实现`fullName`方法:

```objc
- (NSString *)fullName {
    NSString *string = [[[NSString alloc] initWithFormat:@"%@ %@",
                                          self.firstName, self.lastName] autorelease];
    return string;
}
```

你拥有通过`alloc`方法创建的字符串，因此为了遵守内存管理规则，你必须在失去对该字符串的引用之前，放弃对该字符串的所有权。但是，如果您使用`release`，则该字符串将在返回之前被释放(这样的话，该方法就返回了一个无效的对象)。使用`autorelease`，你表示你想要放弃所有权，但你允许方法的调用者**在释放返回的字符串之前使用它**。

你也可以使用如下的方式实现`fullName`方法:

```objective-c
- (NSString *)fullName {
    NSString *string = [NSString stringWithFormat:@"%@ %@",
                                 self.firstName, self.lastName];
    return string;
}
```

根据基本规则，`stringWithFormat:`返回的字符串不属于你，所以你可以安全地从方法内返回该字符串。

相比之下，下面的实现是错误的:

```objective-c
- (NSString *)fullName {
    NSString *string = [[NSString alloc] initWithFormat:@"%@ %@",
                                         self.firstName, self.lastName];
    return string;
}
```

根据命名约定，没有什么可以表示`fullName`方法的调用者拥有返回的字符串。因此，调用者没有理由`release`返回的字符串，因此出现了内存泄漏的问题。

## 你不拥有引用返回的对象

`Cocoa`中的一些方法指定**通过引用返回对象**(也就是说，它们接受类型为`ClassName **`或`id *`的参数)。比方说使用NSData的`initWithContentsOfURL:options:error: `方法或`NSString`的`initWithContentsOfFile:encoding:error: `方法时，传入`NSError **`类型。

当你调用这些方法时，你没有创建`NSError`对象，所以你不拥有它。因此，没有必要释放它，如本例所示:

```objc
NSString *fileName = <#Get a file name#>;
NSError *error;
NSString *string = [[NSString alloc] initWithContentsOfFile:fileName
                        encoding:NSUTF8StringEncoding error:&error];
if (string == nil) {
    // Deal with error...
}
// ...
[string release];
```



# 实现dealloc以放弃对象的所有权

`NSObject`类定义了一个方法`dealloc`，当一个对象没有所有者并且它的内存被回收时，这个方法会被自动调用——在Cocoa术语中，它是“freed”或“deallocated”。`dealloc`方法的作用是释放对象自己的内存，并释放它持有的任何资源，包括自身对其他任何对象实例的所有权。

下面的例子说明了如何为`Person`类实现dealloc方法:

```objective-c
@interface Person : NSObject
@property (retain) NSString *firstName;
@property (retain) NSString *lastName;
@property (assign, readonly) NSString *fullName;
@end
 
@implementation Person
// ...
- (void)dealloc
    [_firstName release];
    [_lastName release];
    [super dealloc];
}
@end
```

> 重要提示：永远不要直接调用其他对象的dealloc方法
>
> 你必须要在dealloc的最后调用父类的实现
>
> 你不应该将系统资源的管理与对象生命周期绑定在一起，参见 [Don’t Use dealloc to Manage Scarce Resources](https://developer.apple.com/library/archive/documentation/Cocoa/Conceptual/MemoryMgmt/Articles/mmPractical.html#//apple_ref/doc/uid/TP40004447-SW13)
>
> 当应用程序终止时，对象可能不会被发送dealloc消息，因为进程的内存会在退出时自动清除，所以简单地让操作系统清理资源比调用所有内存管理方法更为有效。

# Core Foundation使用类似但不同的规则

`Core Foundation`对象也有类似的内存管理规则(参见[Memory Management Programming Guide for Core Foundation](https://developer.apple.com/library/archive/documentation/CoreFoundation/Conceptual/CFMemoryMgmt/CFMemoryMgmt.html#//apple_ref/doc/uid/10000127i))。然而，`Cocoa`和`Core Foundation`的命名约定是不同的。特别是，`Core Foundation`的`Create`规则(参见 [The Create Rule](https://developer.apple.com/library/archive/documentation/CoreFoundation/Conceptual/CFMemoryMgmt/Concepts/Ownership.html#//apple_ref/doc/uid/20001148-103029))不适用于返回`Objective-C`对象的方法。例如，在下面的代码片段中，你不需要为放弃`myInstance`的所有权负责:

```objective-c
MyClass *myInstance = [MyClass createInstance];
```

# 源文档

[Memory Management Policy](https://developer.apple.com/library/archive/documentation/Cocoa/Conceptual/MemoryMgmt/Articles/mmRules.html#//apple_ref/doc/uid/20000994-BAJHFBGH)

