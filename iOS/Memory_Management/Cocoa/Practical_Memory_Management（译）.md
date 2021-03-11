#目录
* [实用的内存管理技巧](#实用的内存管理技巧)
* [使用访问器方法让内存管理更容易](#使用访问器方法让内存管理更容易)
  * [使用访问器方法设置属性值](#使用访问器方法设置属性值)
  * [不要在初始化方法和dealloc中使用访问器方法](#不要在初始化方法和dealloc中使用访问器方法)
* [使用弱引用避免循环引用](#使用弱引用避免循环引用)
* [避免对正在使用的对象做释放操作](#避免对正在使用的对象做释放操作)
* [不要使用dealloc来管理稀缺资源](#不要使用dealloc来管理稀缺资源)
* [容器类拥有它们所包含的对象](#容器类拥有它们所包含的对象)
* [所有权策略是使用引用计数来实现的](#所有权策略是使用引用计数来实现的)
* [源文档](#源文档)

# 实用的内存管理技巧

尽管[Memory Management Policy](https://developer.apple.com/library/archive/documentation/Cocoa/Conceptual/MemoryMgmt/Articles/mmRules.html#//apple_ref/doc/uid/20000994-BAJHFBGH)中描述的基本概念很简单，但是你可以采取一些实际步骤来简化内存管理，并帮助确保你的程序保持可靠和健壮，同时最小化其资源需求。

# 使用访问器方法让内存管理更容易

如果你的类有属性是对象类型，那么必须确保任何被赋值的对象在使用时都不会是已经释放的了。因此，当进行对象类型赋值时，你必须声明对它的所有权。你还必须确保你随后放弃所有当前持有的对象的所有权。

这有可能看起来非常乏味，但如果始终使用访问器方法，出现内存管理问题的可能性就会大大降低。如果在整个代码中对实例变量使用`retain`和`release`，则几乎可以肯定这是错误的。

考虑以下`Counter`对象，假设你要给`count`赋值

```objective-c
@interface Counter : NSObject
@property (nonatomic, retain) NSNumber *count;
@end;
```

该属性会由编译器自动合成两个访问器方法(`set/get`),看一下它们是如何实现的是非常有意义的。

在`get`方法中，你只需要返回自动生成的实例变量,所以不需要`retain`或`release`

```objc
- (NSNumber *)count {
    return _count;
}
```

在`set`方法中，如果其他所有人都遵循相同的规则，对象的所有权可能随时被其他操作放弃，因此你必须通过向它发送`retain`消息来获得对象的所有权，以确保它不会被丢弃。你还必须通过向旧的`count`对象发送一个`release`消息来放弃它的所有权。(在`Objective-C`中，发送消息给`nil`是允许的，所以即使`_count`还没有赋值，这样操作也是没问题的)**你必须将它发送到[newCount retain]之后，以防这两个是同一个对象——您不想无意中导致它被释放。**

**你必须在`[newCount retain]`之后调用对旧值的`release`，因为假如新旧值是同一个对象，先`release`会导致该对象被你无意地释放了**。

```objective-c
- (void)setCount:(NSNumber *)newCount {
    [newCount retain];
    [_count release];
    // Make the new assignment.
    _count = newCount;
}
```

## 使用访问器方法设置属性值

假设您想实现一个方法来重置`count`。你有几个选择,第一种实现使用`alloc`创建`NSNumber`实例，因此需要用一个`release`来平衡它。

```objective-c
- (void)reset {
    NSNumber *zero = [[NSNumber alloc] initWithInteger:0];
    [self setCount:zero];
    [zero release];
}
```

第二种实现使用一个便利构造函数来创建一个新的`NSNumber`对象。因此，不需要`retain`或`release`

```objective-c
- (void)reset {
    NSNumber *zero = [NSNumber numberWithInteger:0];
    [self setCount:zero];
}
```

注意，两者都使用`set`访问器方法。

以下这种做法可能在某些情况下工作正常，这里没有使用访问器方法，但这样做会导致在某些阶段导致错误（例如，你忘记`retain`或`release`,或者实例变量的内存管理语义发生了改变）

> The following will almost certainly work correctly for simple cases, but as tempting as it may be to eschew accessor methods, doing so will almost certainly lead to a mistake at some stage (for example, when you forget to retain or release, or if the memory management semantics for the instance variable change).

```objective-c
- (void)reset {
    NSNumber *zero = [[NSNumber alloc] initWithInteger:0];
    [_count release];//释放旧值
    _count = zero;	 //赋值新值，但是需要在合适的时机对_count进行release
}
```

还要注意，如果您使用`KVO`，那么以这种方式更改变量是不会触发`KVO`的。

## 不要在初始化方法和dealloc中使用访问器方法

唯一不应该使用访问器方法来设置实例变量的地方是**初始化方法和dealloc**。要用`zero`初始化`count`，你可以这样实现`Counter`的`init`方法：

```objective-c
- init {
    self = [super init];
    if (self) {
        _count = [[NSNumber alloc] initWithInteger:0];
    }
    return self;
}
```

为了允许用非0的计数初始化`count`，你可以实现一个`initWithCount:`方法，如下所示:

```objective-c
- initWithCount:(NSNumber *)startingCount {
    self = [super init];
    if (self) {
        _count = [startingCount copy];
    }
    return self;
}
```

因为`Counter`类有一个对象实例变量，所以您还必须实现一个`dealloc`方法。它应该通过发送一个`release`消息来放弃任何实例变量的所有权，并且最终调用父类`super`的实现:

```objective-c
- (void)dealloc {
    [_count release];
    [super dealloc];
}
```

# 使用弱引用避免循环引用

`retain`一个对象会创建对该对象的强引用。一个对象的所有强引用被解除之前，这个对象不会被释放。如果两个对象彼此之间有强引用（要么直接引用，要么通过其他对象链），就会出现所谓的循环引用问题。

图1中表示的对象关系说明了一个循环引用问题。`Document`对象有一个`Page`对象，`Page`对象有`parent`属性表示它所属的Document。如果`Document`对象对`Page`对象有强引用，而`Page`对象又对`Document`对象有强引用，那么这两个对象都不能被释放，简而言之，两个对象都在等待对方的引用计数变为0。

图1 循环引用示例

<img src="./imgs/retaincycles_2x.png" alt="2" style="zoom:50%;" />

解决循环引用的问题是使用弱引用。**弱引用是一种非归属关系，其中源对象不拥有被引用对象的所有权**。

然而，为了保持对象图的完整性，必须在某个地方存在强引用（如果只有弱引用，那么Page和Paragraph可能没有任何所有者，因此这两个对象就会马上被回收）。因此，Cocoa建立了一个约定，**即"parent"角色的对象应该保持对其"children"角色的对象的强引用，而"children"角色的对象则保持对其"parent"角色的对象的弱引用**。

因此，在图1中，`Document`对`Page`强引用(`retain`)，而Page对Document弱引用(`not retain`)

Cocoa中使用弱引用的例子包括但不限于:**tableview的datasource，outline view items，通知observers，各种各样的targets和代理**。

在向弱引用对象发送消息时，需要小心。如果在对象被释放后向其发送消息，则应用程序可能会崩溃。对于对象何时有效，你要心中有数。在大多数情况下，弱引用对象知道其他对象对它的弱引用，就像循环引用一样，并负责在释放时通知其他对象。例如，当您向通知中心注册对象时，**通知中心将存储对该对象的弱引用**，并在发出适当的通知时向它发送消息。当该对象释放时，你必须将它从通知中心注销（移除），以防止通知中心向一个不存在的对象发送消息。同样地，当一个大力对象被释放时，你需要通过发送一个带有`nil`参数的`setDelegate:` 消息给另一个对象来移除代理绑定关系。这些消息通常从对象的`dealloc`方法发送。

# 避免对正在使用的对象做释放操作

`Cocoa`的所有权策略指定`received objects`通常应该在调用方法的整个作用域内保持有效。同样地，在一个方法内返回一个`received object`，也不必担心它被释放。对象的`getter`方法返回缓存的实例变量或计算的值对应用程序来说并不重要。**重要的是对象在你需要它的时候，保持有效**。

这条规则偶尔也有例外，主要分为两类。

1. 当一个对象从基本集合类型中移除时

```objective-c
heisenObject = [array objectAtIndex:n];
[array removeObjectAtIndex:n];
// heisenObject could now be invalid.
```

当一个对象从一个基本集合类中移除时，它会被发送一个`release`(而不是`autorelease`)消息。如果集合是被删除对象的唯一所有者，那么被删除的对象(示例中的heisenObject)将立即被释放。

2. 当"parent"角色的对象被释放了

```
id parent = <#create a parent object#>;
// ...
heisenObject = [parent child] ;
[parent release]; // Or, for example: self.parent = nil;
// heisenObject could now be invalid.
```

在某些情况下，你从另一个对象中持有了`parent`对象，然后在某处直接或间接地释放了`parent`对象,并且`parent`对象是`children`对象的唯一所有者，那么`children`(本例中的`heisenObject`)也会同时被释放（假设在`parent`对象的`dealloc`方法中发送的是`release`消息，而不是`autorelease`消息）。

为了防止这些情况发生，你在接收到`heisenObject`时`retain`它，当你使用完它时`release`它。例如:

```objective-c
heisenObject = [[array objectAtIndex:n] retain];
[array removeObjectAtIndex:n];
// Use heisenObject...
[heisenObject release];
```

# 不要使用dealloc来管理稀缺资源

您通常不应该在`dealloc`方法中管理稀缺资源，如文件描述符、网络连接和缓冲区或缓存。特别是你不应该设计这样的类，理想当然地认为`dealloc`会被调用。`dealloc`的调用可能会被延迟或忽略，因为出现了内存泄漏，对象`dealloc`方法没有被调起。

相反，如果你有一个类，它的实例管理稀缺资源，那么你应该设计你的应用程序，以便你知道何时不再需要资源，然后可以告诉实例在某一时刻进行清理工作。在这之后，你通常会释放实例，然后`dealloc`就会调用。即使你不释放实例，也不会出现上面提到的问题。

如果你试图在`dealloc`之上附加资源管理，可能会出现问题。例如:

1. 对对象图分解的依赖关系进行排序

   对象图分解机制本质上是非有序的。虽然你期望对象以一个特定的顺序释放，但是，这会让你的程序变得不健壮。例如，如果一个对象意外地自动释放而不是被释放，那么销毁顺序可能会改变，这可能会导致意外的结果。

2. 不回收稀缺资源。

   内存泄漏是应该修复的错误，但它们通常不会立即导致错误。但是，如果在你希望释放稀缺资源的时候没有释放它们，那么你可能会遇到更严重的问题。例如，如果应用程序用完文件描述符，用户可能无法保存数据。

3. 清除在错误线程上执行的逻辑。

   如果一个对象在一个意外的时间被自动释放，它将在它所在的任何线程的自动释放池块上被释放。这对于只能从一个线程访问的资源来说是致命的。

# 容器类拥有它们所包含的对象

当您将对象添加到集合(例如数组、字典或集合)时，集合将获得该对象的所有权。当对象从集合中移除或集合本身被释放时，集合将放弃所有权。因此，例如，如果你想创建一个数字数组，你可以这样做:

```objective-c
NSMutableArray *array = <#Get a mutable array#>;
NSUInteger i;
// ...
for (i = 0; i < 10; i++) {
    NSNumber *convenienceNumber = [NSNumber numberWithInteger:i];
    [array addObject:convenienceNumber];
}
```

在本例中，您没有调用`alloc`，因此不需要调用`release`。不需要`retain`新数字(`convenienceNumber`)，因为数组会`retain`这些数字。

```objective-c
NSMutableArray *array = <#Get a mutable array#>;
NSUInteger i;
// ...
for (i = 0; i < 10; i++) {
    NSNumber *allocedNumber = [[NSNumber alloc] initWithInteger:i];
    [array addObject:allocedNumber];
    [allocedNumber release];
}
```

在这种情况下，您确实需要在`for`循环的范围内发送一个`release`消息给`allocedNumber`，以平衡`alloc`。因为数组在`addObject:`添加时`retain`了这个数字，所以它在数组中不会被释放。

>To understand this, put yourself in the position of the person who implemented the collection class.You want to make sure that no objects you’re given to look after disappear out from under you, so you send them a retain message as they’re passed in. If they’re removed, you have to send a balancing release message, and any remaining objects should be sent a release message during your own dealloc method.

# 所有权策略是使用引用计数来实现的

所有权策略是通过引用计数来实现的——通常在`retain`方法之后称为`retain count`。每个对象都有一个`retain count`。

* 当你创建一个对象，它的引用计数为1

* 当你向对象发送`retain`消息，它的引用计数加1

* 当你想对象发送`release`消息，它的引用计数减1

  当你向对象发送`autorelease`消息，它的引用计数在当前`autorelease pool` `pop`之后减1

* 如果对象的引用计数减少到0，它就会被释放。

> 重要提示:主动去查询对象的引用计数是不合理的，参见[retainCount](https://developer.apple.com/library/archive/documentation/LegacyTechnologies/WebObjects/WebObjects_3.5/Reference/Frameworks/ObjC/Foundation/Protocols/NSObject/Description.html#//apple_ref/occ/intfm/NSObject/retainCount)。通常返回的结果是误导的，因为你可能不知道哪些框架对象`retain`了你感兴趣的对象。**在调试内存管理问题时，您应该只关心确保您的代码遵循所有权规则**。



# 源文档

[Practical Memory Management](https://developer.apple.com/library/archive/documentation/Cocoa/Conceptual/MemoryMgmt/Articles/mmPractical.html#//apple_ref/doc/uid/TP40004447-SW1)

