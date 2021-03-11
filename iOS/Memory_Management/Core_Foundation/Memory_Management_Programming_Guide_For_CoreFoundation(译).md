# 目录

   * [介绍](#介绍)
   * [本文内容](#本文内容)
   * [<span id="user-content-OP">Ownership Policy </span>](#ownership-policy-)
      * [<span id="user-content-basic">基础内容 </span>](#基础内容-)
      * [Naming Conventions (命名约定)](#naming-conventions-命名约定)
      * [<span id="user-content-TCR">The Create Rule</span> (对象创建规则)](#the-create-rule-对象创建规则)
      * [<span id="user-content-TGR">The Get Rule</span> (对象获取引用规则)](#the-get-rule-对象获取引用规则)
      * [Instance Variables and Passing Parameters (实例变量和参数传递)](#instance-variables-and-passing-parameters-实例变量和参数传递)
      * [Ownership Examples](#ownership-examples)
   * [Core Foundation Object Lifecycle Management](#core-foundation-object-lifecycle-management)
      * [Retaining Object References](#retaining-object-references)
      * [Releasing Object References](#releasing-object-references)
      * [Copying Object References](#copying-object-references)
      * [Determining an Object's Retain Count](#determining-an-objects-retain-count)
   * [<span id="user-content-CF">Copy Functions</span>](#copy-functions)
      * [Shallow Copy (浅拷贝)](#shallow-copy-浅拷贝)
      * [Deep Copy (深拷贝)](#deep-copy-深拷贝)
   * [Allocators](#allocators)
   * [备注](#备注)
   * [源文档](#源文档)

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

## Naming Conventions(命名约定)

有很多方法可以使用`Core Foundation`获得对对象的引用。根据`Core Foundation`所有权策略，你需要知道你否拥有函数返回的对象，以便你知道在内存管理方面应该采取什么操作。`Core Foundation`为其函数建立了一个命名约定，让你确定是否拥有函数返回的对象。简而言之，**如果函数名中包含单词"`Create`"或"`Copy`"，那么你就拥有该对象**。如果函数名包含单词"`Get`"，则不拥有该对象。这些规则在[The Create Rule](#TCR)和[<span id="user-content-TGR">The Get Rule</span> (对象获取引用规则)](#the-get-rule-对象获取引用规则)中有更详细的解释。

> 重要提示：Cocoa为内存管理定义了一套类似的命名约定(参见[Advanced Memory Management Programming Guide](https://developer.apple.com/library/archive/documentation/Cocoa/Conceptual/MemoryMgmt/Articles/MemoryMgmt.html#//apple_ref/doc/uid/10000011i))。Core Foundation命名约定，特别是"create"一词的使用，只适用于返回Core Foundation对象的C函数。Objective-C方法的命名约定受Cocoa的约定控制，不管这个方法返回的是Core Foundation还是Cocoa对象。

## The Create Rule(对象创建规则)

`Core Foundation` 的函数名称指示你什么时候拥有（持有）返回的对象:

* 在函数名称中带有"Create"单词的对象创建函数
* 在函数名称中带有"Copy"单词的对象复制函数

如果你拥有一个对象，当你使用完它时，你需要放弃它的所有权(使用`CFRelease`)。

请看下面的例子，第一个示例使用了两个与`CFTimeZone`关联的`create`函数，另一个使用与`CFBundle`关联的`create`函数。

```objective-c
CFTimeZoneRef   CFTimeZoneCreateWithTimeIntervalFromGMT (CFAllocatorRef allocator, CFTimeInterval ti);
CFDictionaryRef CFTimeZoneCopyAbbreviationDictionary (void);
CFBundleRef     CFBundleCreate (CFAllocatorRef allocator, CFURLRef bundleURL);
```

第一个函数的名称中包含单词"`Create`"，它创建了一个新的`CFTimeZone`对象。你拥有这个对象，你有责任放弃它的所有权。第二个函数在其名称中包含单词"`Copy`"，并创建一个`time zone` 对象属性的副本。(请注意，这与获取属性本身不同——参见[The Get Rule](#TGR)。) 同样，你拥有这个对象，你有责任放弃所有权。第三个函数`CFBundleCreate`在其名称中包含单词"`Create`"，但文档声明它可能返回一个现有的`CFBundle`。不过，不管是否实际创建了一个新对象，你都拥有这个对象。如果返回的是一个已存在的对象，那么它的引用计数将增加，因此你有责任放弃所有权。

下一个示例可能看起来更复杂，但它仍然遵循相同的简单规则。

```objective-c
/* from CFBag.h */
CF_EXPORT CFBagRef  CFBagCreate(CFAllocatorRef allocator, const void **values, CFIndex numValues, const CFBagCallBacks *callBacks);
CF_EXPORT CFMutableBagRef   CFBagCreateMutableCopy(CFAllocatorRef allocator, CFIndex capacity, CFBagRef bag);
```

`CFBag`函数`CFBagCreateMutableCopy`在其名称中包含了"`Create`"和"`Copy`"。它是一个创建函数，因为函数名包含单词"`Create`"。还要注意，第一个参数的类型是`CFAllocatorRef`——这是进一步的提示。这个函数中的"`Copy`"是一个提示，该函数接受一个`CFBagRef`参数并生成一个对象的副本。源集合的元素对象还会被复制到新创建的`bag`中。函数名的子串"`Copy`"和"`NoCopy`"指示如何处理源对象所拥有的对象——也就是说，它们是否被复制。

## The Get Rule(对象获取引用规则)

如果你从"`Create`"或`Copy`函数以外的任何`Core Foundation`函数(例如`Get`函数)接收对象，则你不拥有该对象，也不能确定对象的生命周期。如果你想确保这样的对象在使用时不会被丢弃，则必须声明所有权(使用`CFRetain`函数)。然后，当你用完它后，你有责任放弃所有权。

参考`CFAttributedStringGetString`函数，它根据一个`attributed string`返回普通字符串。

`CFStringRef CFAttributedStringGetString (CFAttributedStringRef aStr);`

如果释放了`attributed string`，那么就意味着放弃了返回字符串的所有权。如果`attributed string`是它唯一的所有者，那么这个返回的字符串就没有了其他所有者，它会立即被释放。因此，如果你需要在`attributed string`对象销毁后继续使用返回的字符串，需要手动声明所有权(使用`CFRetain`),或者复制它。使用完后，必须使用`CFRelease`，否则会有内存泄漏。

## Instance Variables and Passing Parameters (实例变量和参数传递)

根据基本规则，当你将一个对象传递给另一个对象(作为函数参数)时，如果需要维护它，你应该期望接收方将获得所传递对象的所有权。要理解这一点，请将自己置于接收对象的实现者的位置。当函数接收一个对象作为参数时，接收方最初并不拥有该对象。因此，对象可以在任何时候被释放——除非接收方获得它的所有权(使用`CFRetain`)。当接收方完成使用该对象时，接收方负责放弃所有权(使用`CFRelease`)。

## Ownership Examples

为了防止运行时错误和内存泄漏，您应该确保在接收、传递或返回`Core Foundation`对象的任何地方一致地应用`Core Foundation`所有权策略。理解为什么有时候需要成为一个非自己创建对象的所有者，可以参考以下示例。假设你从另一个对象获取到了一个值对象，然后这个值对象的唯一所有者释放了，那么该值对象没有所有者，也会被释放。现在你就引用了一个已经释放的对象，如果尝试使用它，应用程序将崩溃。

下面的代码列出3种常见情况:`set`函数、`get`函数以及让`Core Foundation`对象保持有效直到一个特定条件发生的函数。第一个，`set`函数：

```c
static CFStringRef title = NULL;
void SetTitle(CFStringRef newTitle) {
    CFStringRef temp = title;
    title = CFStringCreateCopy(kCFAllocatorDefault , newTitle);
    CFRelease(temp);
}
```

上面的示例使用一个静态`CFStringRef`变量来保存`retained`的`CFString`对象。你可以使用其他方法来存储它，但是你必须将它放在接收函数以外的其他地方。这里先用一个临时局部变量保存当前`title`,再进行新值的`copy`和释放旧的(当前的)`title`。如果参数`newTitle`与当前`title`是同一对象，则做了一次复制和释放操作，平衡等效。

请注意，在上面的示例中，对象是`copied`的，而不是简单地`retained`。(回想一下，从所有权的角度来看，它们是等价的——参见[基本内容](#basic)。)这样做的原因是，`title` property可能被认为是一个`attribute`。除了通过访问器方法，它不应该被其他方法更改。即使参数类型为`CFStringRef`，也可能传入对`CFMutableString`对象的引用，这将允许从外部更改值。因此，您可以复制对象，以便在您持有它时它不会被更改。你应该复制对象，当你需要自己持有一个不可变更的对象。如果对象被认为是一个持有关系，那么你应该`retain`它。

对应的`get`函数，要简单得多:

```c
CFStringRef GetTitle() {
    return title;
}
```

通过简单地返回一个对象，您将返回对它的弱引用。换句话说，指针值被复制到接收方的变量中，但引用计数不变。**当返回集合中的元素时，也会发生同样的事情**。

下面的函数`retain`从集合中取到的对象，直到不再需要它，然后释放它。对象被假定为不可变的。

```objc
static CFStringRef title = NULL;
void MyFunction(CFDictionary dict, Boolean aFlag) {
    if (!title && !aFlag) {
        title = (CFStringRef)CFDictionaryGetValue(dict, CFSTR(“title”));
        title = CFRetain(title);
    }
    /* Do something with title here. */
    if (aFlag) {
        CFRelease(title);
    }
}
```

下面的示例演示了将一个`number`对象传递给数组。数组的回调函数表示添加到集合中的对象将被`retain`(集合持有它们)，因此`number`对象可以在添加到数组后,调用`CFRelease`函数。

```c
float myFloat = 10.523987;
CFNumberRef myNumber = CFNumberCreate(kCFAllocatorDefault,
                                    kCFNumberFloatType, &myFloat);
CFMutableArrayRef myArray = CFArrayCreateMutable(kCFAllocatorDefault, 2, &kCFTypeArrayCallBacks);
CFArrayAppendValue(myArray, myNumber);
CFRelease(myNumber);
// code continues...
```

注意，如果(a)释放数组，而(b)在释放数组后继续使用`number`，这里有一个潜在的陷阱:

```c
CFRelease(myArray);
CFNumberRef otherNumber = // ... ;
CFComparisonResult comparison = CFNumberCompare(myNumber, otherNumber, NULL);
```

除非你`retain`了`number`或`array`，或者传递给其他保持其所有权的对象，否则代码将在比较函数中出现错误。如果没有其他对象持有`array`或`number`，当数组被释放时，它释放它的元素。在这种情况下，这也会导致`number`的释放，因此比较函数将对释放的对象进行操作，从而崩溃。

# Core Foundation Object Lifecycle Management

`Core Foundation`对象的生命周期由其引用计数确定——一个内部数字，这个数字描述了有多少数量的`clients`希望该对象存在。在`Core Foundation`中创建或复制对象时，其引用计数将设置为1。后续有需要的`clients`可以通过`CFRetain`来声明对象的所有权，这会增加引用计数。稍后，当不再需要该对象时，调用`CFRelease`。当引用计数达到0时，对象的`allocator`释放对象的内存。

## Retaining Object References

要增加`Core Foundation`对象的引用计数，需要将对该对象的引用作为`CFRetain`函数的形参传递:

```c
/* myString is a CFStringRef received from elsewhere */
myString = (CFStringRef)CFRetain(myString);
```

## Releasing Object References

要减少`Core Foundation`对象的引用计数，传递一个对该对象的引用作为`CFRelease`函数的形参:

`CFRelease(myString);`

重要提示:永远不要直接释放`Core Foundation`对象(例如，通过调用`free`)。当你使用完一个对象后，调用`CFRelease`函数，`Core Foundation`将正确地处理它。

## Copying Object References

复制对象时，新生成的对象的引用计数为1，而不管原始对象的引用计数是多少。有关复制对象的更多信息，请参见[Copy Functions](#CF)。

## Determining an Object's Retain Count 

如果你想知道`Core Foundation`对象的当前引用计数，传递一个对该对象的引用作为`CFGetRetainCount`函数的参数:

`CFIndex count = CFGetRetainCount(myString);`

然而，请注意，通常不需要确定`Core Foundation`对象的引用计数，除非在调试中。如果你发现自己需要知道对象的保留计数，请检查是否正确地遵守了所有权策略规则(请参阅[Ownership Policy](#OP))。

# Copy Functions

通常，当使用`=`操作符将一个变量的值赋值给另一个变量时，会发生标准的复制操作，也可以称为简单赋值操作。例如，表达式`myInt2 = myInt1`将`myInt1`的整型内容从`myInt1`使用的内存复制到`myInt2`使用的内存中。复制操作之后，内存中的两个单独区域包含相同的值。但是，如果试图以这种方式复制`Core Foundation`对象，请注意，**你不会复制对象本身，而只复制对该对象的引用**。

例如，`Core Foundation`的初学者可能认为，要复制`CFString`对象，他们可以使用表达式`myCFString2 = myCFString1`。同样，这个表达式实际上并没有复制字符串数据。因为`myCFString1`和`myCFString2`都必须具有CFStringRef类型，所以这个表达式只复制对对象的引用。在复制操作之后，你将拥有对CFString的引用的两个副本。这种类型的复制非常快，因为只复制引用，但重要的是要记住，以这种方式复制可变对象是危险的。**与使用全局变量的程序一样，如果应用程序的某个部分使用引用的副本更改了对象，则程序中拥有该引用副本的其他部分无法知道数据已更改**。

如果要复制对象，必须使用`Core Foundation`为此专门提供的函数之一。继续`CFString`示例，您将使用`CFStringCreateCopy`创建一个全新的`CFString`对象，该对象包含与原始对象相同的数据。具有" `CreateCopy` "函数的`Core Foundation`类型还提供了" `CreateMutableCopy` "函数，它返回一个可以修改的对象的副本。

## Shallow Copy(浅拷贝)

复制复合对象(如**可以包含其他对象的集合对象**)也必须小心。使用`=`操作符对这些对象执行复制会导致复制对象引用。**与CFString和CFData等简单对象相比，为CFArray和CFSet等复合对象提供的"CreateCopy"函数实际上执行的是浅拷贝**。**对于这些对象，浅拷贝意味着创建一个新的集合对象，但不复制原始集合的内容——只复制对象引用到新容器**。这种类型的复制是有用的，例如，你有一个不可变的数组，你想要重新排序它。在这种情况下，你不想复制所有包含的对象，因为不需要更改它们——为什么要使用额外的内存呢?你仅仅需要的是改变这个不可变的容器（这里指代为数组），这里的风险与复制具有简单类型的对象引用的风险相同（也就是开头描述的风险问题）。

## Deep Copy(深拷贝)

当您想要创建一个全新的复合对象时，您必须执行深拷贝。深拷贝复制复合对象以及它包含的所有对象的内容。`Core Foundation`的当前版本包含一个函数`CFPropertyListCreateDeepCopy`，该函数执行属性列表的深度复制。如果要创建其他结构的全新副本，则可以通过递归遍历复合对象并将其一一复制添加到新副本。在实现此功能时要小心，因为复合对象可能是递归的——它们可能直接或间接包含对自身的引用——这可能导致递归循环。

# Allocators

`Core Foundation`抽象的操作系统服务是内存分配。它使用`Allocators`来实现这个目的。

`Allocators`是为你分配和释放内存的`opaque`对象。您永远不必为`Core Foundation`对象直接分配、重新分配或释放内存——也不应该这样做。你将`Allocators`传递给创建对象的函数;这些函数的名称中带有"`Create`"，例如，`CFStringCreateWithPascalString`。创建函数使用`Allocators`为它们创建的对象分配内存。

在对象的生命周期内，`Allocators`与该对象相关联。如果需要重新分配内存，则对象使用该`Allocators`;当需要释放对象时，则使用`Allocators`进行对象的释放。`Allocators`还用于创建最初创建的对象所需的任何对象。有些函数还允许为特殊目的传入`Allocators`，比如释放临时缓冲区的内存。

`Core Foundation`允许你创建自己的自定义`allocators`。`Core Foundation`还提供了一个`system allocator`，并将该`allocator`初始化为当前线程的默认`allocator`。(每个线程有一个默认的`allocator`。)您可以在代码中的任何时候将自定义`allocator`设置为线程的默认值。然而，`system allocator`是一种很好的通用`allocator`，应该足以应付几乎所有情况。在特殊情况下可能需要自定义`allocator`，比如在Mac OS 9上的某些情况下，或者当性能存在问题时，作为批量`allocator`。除了这些罕见的情况，您既不应该使用自定义`allocator`，也不应该将它们设置为默认值，特别是对于`libraries`。

有关`allocator`的更多信息，特别是关于创建自定义`allocator`的信息，请参见[Creating Custom Allocators](https://developer.apple.com/library/archive/documentation/CoreFoundation/Conceptual/CFMemoryMgmt/Tasks/CustomAllocators.html#//apple_ref/doc/uid/20001154-CJBEHAAG)。

# 备注

原本还有一部分小节，看起来不是很长，就不一一对照翻译，看起来使用场景较少。

# 源文档

[Memory Management Programming Guide for Core Foundation](https://developer.apple.com/library/archive/documentation/CoreFoundation/Conceptual/CFMemoryMgmt/CFMemoryMgmt.html#//apple_ref/doc/uid/10000127-SW1)

