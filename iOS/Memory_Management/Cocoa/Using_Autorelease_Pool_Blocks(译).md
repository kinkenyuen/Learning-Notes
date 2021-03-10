# 使用自动释放池代码块

自动释放池提供了一种机制，你可以放弃对象的所有权，并且避免对象立即被释放(**例如当你从一个方法返回一个对象时**)。通常，你不需要创建自己的自动释放池，但是在某些情况下，你必须这样做，或者这样做是更合理的。

# 关于自动释放池

`autorelease`池块使用`@autoreleasepool`标记，示例如下:

```objective-c
@autoreleasepool {
    // Code that creates autoreleased objects.
}
```

在自动释放池末尾时（**pool本身需要释放、销毁或pop的时候**），那些被自动释放池块包含的并且曾经被发送过`autorelease`消息的对象，会接收到一个`release`消息。

像其他代码块一样，自动释放池块可以嵌套:

```objective-c
@autoreleasepool {
    // . . .
    @autoreleasepool {
        // . . .
    }
    . . .
}
```

(你通常不会看到完全如上所述的代码;通常，一个源文件中的自动释放池块中的代码会调用包含在另一个自动释放池块中的另一个源文件中的代码。)

> For a given `autorelease` message, the corresponding `release` message is sent at the end of the autorelease pool block in which the `autorelease` message was sent.

`Cocoa`总是希望代码在自动释放池块中执行，否则被`autoreleased`的对象不会被释放，从而导致应用程序内存泄漏。(如果你在自动释放池块之外发送一个`autorelease`消息，`Cocoa`会抛出一个错误消息。) `AppKit`和`UIKit`框架在自动释放池块中处理每个事件循环迭代(如鼠标点击事件或屏幕触摸)。因此，你通常不必自己创建一个自动释放池块，甚至不必查看用于创建一个自动释放池块的代码。然而，有三种情况你可能需要创建自己的自动释放池块:

* 编写不基于UI框架的程序，如命令行工具

* 在一个会创建许多临时对象的循环代码中

  您可以在循环中使用一个自动释放池块，在下一次迭代之前释放这些对象。在循环中使用自动释放池块有助于减少应用程序的最大内存占用。

* `spawn`一个辅助线程

  当线程开始执行时，你必须创建你自己的自动释放池块;否则，应用程序将出现内存泄漏。(参见[Autorelease Pool Blocks and Threads](https://developer.apple.com/library/archive/documentation/Cocoa/Conceptual/MemoryMgmt/Articles/mmAutoreleasePools.html#//apple_ref/doc/uid/20000047-1041876) )

# 使用自己的自动释放池块来减少内存占用峰值

很多程序创建了临时对象，并且它们是`autoreleased`的，这些对象会占用程序的内存，直到自动释放池结束。**在许多情况下允许创建一些临时对象并累积到当前事件循环迭代中，并且不会产生很大的内存开销**；然而，在某些情况下，你可能**创建大量的临时对象**，这些临时对象大大增加了内存占用，因此你应该对这些大量的临时对象做处理。在这种情况下，你就需要创建自己的自动释放池，如此一来，自定义的自动释放池结束后，这些大量的临时对象也会被立即销毁，从而减少程序的内存占用。

下面的示例展示了如何在`for`循环中使用自定义自动释放池块:

```objective-c
NSArray *urls = <# An array of file URLs #>;
for (NSURL *url in urls) {
 
    @autoreleasepool {
        NSError *error;
        NSString *fileContents = [NSString stringWithContentsOfURL:url
                                         encoding:NSUTF8StringEncoding error:&error];
        /* Process the string, creating and autoreleasing more objects. */
    }
}
```

`for`循环一次处理一个文件。在自动释放池块中被发送`autorelease`消息的任何对象(例如`fileContents`)都会在块的末尾释放。

在自动释放池块之后，您应该将块内`autoreleased`的任何对象视为“已处理”。不要向该对象发送消息或将其返回给方法的调用者。如果你必须在自动释放池块之外使用一个临时对象，你可以发送一个`retain`消息给块内的对象，然后在块之后发送`autorelease`，如下例所示:

```objective-c
– (id)findMatchingObject:(id)anObject {
 
    id match;
    while (match == nil) {
        @autoreleasepool {
 
            /* Do a search that creates a lot of temporary objects. */
            match = [self expensiveSearchForObject:anObject];
 
            if (match != nil) {
                [match retain]; /* Keep match around. */
            }
        }
    }
 
    return [match autorelease];   /* Let match go and return it. */
}
```

在自动释放池块内给`match`对象发送`retain`消息，让`match`对象不在块结束时立即释放，接着在块外部返回这个临时对象时，发送`autorelease`消息，可以让外部接着使用这个对象。（这个返回的对象通常会`push`到当前线程`runloop`迭代中使用`autorelease pool`）

# 自动释放池块和线程

`Cocoa`应用程序中的每个线程维护自己的自动释放池块栈。如果你正在编写一个基于只`Foundation`的程序，或者如果你`detach`了一个线程，你需要创建你自己的自动释放池块。

如果你的应用程序或者线程是长期存在的并且可能会产生很多`autoreleased`对象，你应该使用自动释放池块(就像`AppKit`和`UIKit`在主线程上做的那样);否则，`autoreleased`对象会累积，内存占用会增加。如果你的`detached`线程不进行`Cocoa`调用，你就不需要使用自动释放池块。

> 注意:如果你使用POSIX线程api而不是NSThread创建辅助线程，你不能使用Cocoa，除非Cocoa处于多线程模式。只有在detach了第一个NSThread对象之后，Cocoa才会进入多线程模式。要在辅助的POSIX线程上使用Cocoa，应用程序必须首先detach至少一个NSThread对象，这样然后可以立即退出。你可以通过NSThread类方法ismultithreading来测试Cocoa是否处于多线程模式。

# 源文档

[Using Autorelease Pool Blocks](https://developer.apple.com/library/archive/documentation/Cocoa/Conceptual/MemoryMgmt/Articles/mmAutoreleasePools.html#//apple_ref/doc/uid/20000047-CJBFBEDI)