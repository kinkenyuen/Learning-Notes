# 同步

一个应用程序中存在多个线程，这就可能出现从多个执行线程安全访问资源的问题。两个修改同一资源的线程可能会以意想不到的方式相互干扰。例如，一个线程可能会覆盖另一个线程的更改，或将应用程序置于未知且可能无效的状态。如果幸运的话，损坏的资源可能会导致明显的性能问题或崩溃，这些问题比较容易跟踪和修复。但是，如果不幸的话，损坏可能会导致一些微妙的错误，这些错误直到很久以后才显现出来，或者这些错误可能需要重写底层基本代码。

说到线程安全，一个好的设计是你拥有的最好的保护。**避免共享资源和最小化线程之间的交互可以减少线程之间相互干扰的可能性**。然而，一个完全无干扰的设计并不总是可能的。在线程必须进行交互的情况下，**您需要使用同步工具来确保它们在交互时能够安全地进行**。

OS X和iOS提供了大量的同步工具供您使用，从提供**互斥访问**的工具到那些在您的应用程序中**正确排序事件**的工具。下面几节将描述这些工具以及如何在代码中使用它们来影响对程序资源的安全访问。

# 同步工具

为了防止不同的线程意外地更改数据，您可以将应用程序设计为不存在同步问题，也可以使用同步工具。尽管完全避免同步问题是可取的，但这并不总是可能的。下面的部分描述了您可以使用的同步工具的基本类别。

## 原子操作

原子操作是对简单数据类型进行同步的一种简单形式。原子操作的优点是它们不会阻塞竞争线程。对于简单的操作，例如递增一个计数器变量，这比使用锁性能好得多。

OS X和iOS包括大量的操作，用于对32位和64位值执行基本的数学和逻辑操作。这些操作包括比较和交换操作、测试和设置操作以及测试和清除操作的原子版本。有关更多原子操作可查看`/usr/include/libkern/OSAtomic.h`头文件或查看[atomic](https://developer.apple.com/library/archive/documentation/System/Conceptual/ManPages_iPhoneOS/man3/atomic.3.html#//apple_ref/doc/man/3/atomic)介绍。

## 内存屏障和易失性变量（Memory Barriers and Volatile Variables）

为了达到最佳性能，编译器经常重新排列汇编级指令，以使处理器的指令管道尽可能满。作为优化的一部分，当编译器认为这样做不会产生不正确的数据时，它可能会对访问主存的指令重新排序。不幸的是，编译器并不总是能够检测到所有与内存相关的操作。如果看似独立的变量实际上相互影响，则编译器优化可能会以错误的顺序更新这些变量，从而产生可能不正确的结果。

内存屏障是一种非阻塞的同步工具，用于确保内存操作以正确的顺序进行。内存屏障的作用就像一个栅栏，使处理器先完成栅栏前面的加载、存储操作，再完成栅栏之后的加载、存储操作。内存屏障通常用于确保一个线程(但对另一个线程可见)的内存操作总是按照预期的顺序进行。在这种情况下，缺少内存屏障可能会让其他线程看到看似不可能的结果。（要查看案例，参见维基百科[Memory barrier](https://en.wikipedia.org/wiki/Memory_barrier)）要使用内存屏障，只需在代码中的适当位置调用OSMemoryBarrier函数。

易失性变量对单个变量应用另一种类型的内存约束。编译器通常通过将变量的值装入寄存器来优化代码。对于局部变量，这通常不是问题。但是，如果变量在另一个线程中是可见的，这样的优化可能会防止其他线程注意到对它的任何更改。将volatile关键字应用于变量，强制编译器在每次使用该变量时从内存加载该变量。如果一个变量的值可以在任何时候被编译器无法检测到的外部源更改，则可以将其声明为volatile。

因为内存屏障和volatile变量都减少了编译器可以执行的优化次数，所以应该尽量少使用它们，并且只在需要的地方使用，以确保正确性。有关使用内存屏障的信息，请参阅[OSMemoryBarrier](https://developer.apple.com/library/archive/documentation/System/Conceptual/ManPages_iPhoneOS/man3/OSMemoryBarrier.3.html#//apple_ref/doc/man/3/OSMemoryBarrier)。

## 锁

**锁是最常用的同步工具之一。您可以使用锁来保护代码的关键部分，这是一段每次只允许一个线程访问的代码。例如，一个临界区可能操作一个特定的数据结构，或者使用一些一次最多支持一个客户端的资源。通过在这个部分周围放置一个锁，可以排除其他线程进行可能影响代码正确性的更改**。

表4-1列出了程序员常用的一些锁。OS X和iOS提供了大多数锁类型的实现，但不是全部。对于不支持的锁类型，描述列解释了为什么这些锁不能直接在平台上实现。

表4-1 锁类型

| Lock                            | Description                                                  |
| ------------------------------- | ------------------------------------------------------------ |
| Mutex(互斥锁)                   | 互斥锁充当资源周围的保护屏障。互斥锁是一种信号量，每次只允许访问一个线程。如果一个互斥锁正在使用中，而另一个线程试图获取它，那么该线程将阻塞，直到互斥锁被其原始持有者释放。如果多个线程竞争同一个互斥锁，那么一次只允许一个线程访问它。 |
| Recursive lock（递归锁）        | 递归锁是互斥锁的变体。递归锁允许单个线程在释放锁之前多次获取锁。其他线程保持阻塞状态，直到锁的所有者释放它获得锁的相同次数的锁。递归锁主要在递归迭代期间使用，但也可以在多个方法各自需要分别获取锁的情况下使用。 |
| Read-write lock(读写锁)         | 读写锁也称为共享排他锁。这种类型的锁通常用于大规模操作，如果经常读取受保护的数据结构，并且只偶尔修改，则可以显著提高性能。正常运行时，多个读取者可以同时访问该数据结构。但是，当一个线程想要写这个结构时，它就会阻塞，直到所有的读取器释放锁，这时它就可以获得锁并更新这个结构。当写线程等待锁时，新的读线程会阻塞，直到写线程完成。系统仅支持使用POSIX线程使用读写锁。更多使用锁的相关信息，参见[pthread](https://developer.apple.com/library/archive/documentation/System/Conceptual/ManPages_iPhoneOS/man3/pthread.3.html#//apple_ref/doc/man/3/pthread) |
| Distributed lock(分布式锁)      | 分布式锁在进程级别上提供互斥访问。与真正的互斥锁不同，分布式锁不会阻塞进程或阻止进程运行。它只是报告锁何时繁忙，并让进程决定如何继续。 |
| Spin lock(自旋锁)               | 自旋锁反复轮询其锁条件，直到该条件为真。自旋锁最常用于多处理器系统，在这些系统中，锁的预期等待时间很短。在这些情况下，轮询通常比阻塞线程更有效，阻塞线程涉及上下文切换和线程数据结构的更新。由于旋转锁的轮询特性，系统不提供任何旋转锁的实现，但是您可以在特定的情况下轻松地实现它们。有关在内核中实现自旋锁的信息,请参阅[Kernel Programming Guide](https://developer.apple.com/library/archive/documentation/Darwin/Conceptual/KernelProgramming/About/About.html#//apple_ref/doc/uid/TP30000905) |
| Double-checked lock(双重检查锁) | 双重检查锁是一种尝试，通过在获取锁之前测试锁定条件来减少获取锁的开销。由于双重检查锁可能不安全，系统不提供显式支持，因此不鼓励使用双重检查锁。 |

> 注意:大多数类型的锁也包含一个内存屏障，以确保在进入临界区之前，任何加载和存储指令都已完成。

有关如何使用锁的信息，请参见[Using Locks](https://developer.apple.com/library/archive/documentation/Cocoa/Conceptual/Multithreading/ThreadSafety/ThreadSafety.html#//apple_ref/doc/uid/10000057i-CH8-SW16)。

## 条件(Conditions)

**条件是另一种类型的信号量，它允许线程在某个条件为真时相互发出信号。条件通常用于指示资源的可用性或确保任务按照特定的顺序执行**。当线程测试一个条件时，它会阻塞，除非该条件已经为真。它将保持阻塞状态，直到其他线程显式地更改并发出信号。**条件和互斥锁的区别在于可以允许多个线程同时访问条件**。条件就像是看门人，只有满足设定的标准才让不同的线程访问。

使用条件的一种方法是管理待处理事件池。当事件队列中有事件时，事件队列将使用条件变量来通知等待线程。如果一个事件到达，则队列将适当地发出条件信号。如果一个线程已经在等待，它将被唤醒，然后从队列中拉出事件并处理它。如果两个事件大约在同一时间进入队列，队列将发出两次条件信号，以唤醒两个线程。

系统提供多种不同技术实现条件。但是，条件的正确实现需要仔细编码，所以在您自己的代码中使用条件之前，应该先看看使用条件的示例[Using Conditions](https://developer.apple.com/library/archive/documentation/Cocoa/Conceptual/Multithreading/ThreadSafety/ThreadSafety.html#//apple_ref/doc/uid/10000057i-CH8-SW4)。

## 执行Selector例程

Cocoa应用程序可以很方便地以同步的方式将消息传递给单个线程。NSObject类声明了在应用程序的一个活动线程上执行选择器的方法。这些方法让您的线程异步交付消息，并保证目标线程将同步执行这些消息。例如，可以使用执行Selector消息将分布式计算的结果传递给应用程序的主线程或指定的辅助线程。

有关执行Selector例程的摘要和有关如何使用它们的更多信息，请参见[Cocoa Perform Selector Sources](https://developer.apple.com/library/archive/documentation/Cocoa/Conceptual/Multithreading/RunLoopManagement/RunLoopManagement.html#//apple_ref/doc/uid/10000057i-CH16-SW44)

# 同步开销与性能

同步有助于确保代码的正确性，但这是以牺牲性能为代价的。使用同步工具会带来延迟，即使在没有竞争的情况下也是如此。锁和原子操作通常涉及使用内存屏障和内核级同步，以确保代码得到适当的保护。如果存在竞争锁，那么您的线程可能会阻塞并经历更大的延迟。

表4-2列出了在无争用情况下与互斥锁和原子操作相关的一些大致开销。这些测量代表了数千个样本的平均时间。但是，与线程创建时间一样，获取互斥锁的时间(即使在无竞争的情况下)可能会根据处理器负载、计算机速度以及可用的系统和程序内存数量有很大差异。

表4-2 互斥和原子操作开销

| 项             | 大致开销     | 描述                                                         |
| -------------- | ------------ | ------------------------------------------------------------ |
| 互斥锁获取时间 | 大约0.2微秒  | 这是无争用情况下的锁获取时间。如果锁被另一个线程持有，那么获取时间可能会大得多。这些数据是通过分析在运行OS X v10.5、2ghz双核处理器和1gb RAM的基于intel的iMac上获取互斥锁时生成的平均值和中值来确定的。 |
| 原子比较和交换 | 大约0.05微秒 | 这是无争用情况下的比较和交换时间。这些数据是通过分析操作的平均值和中值来确定的，是在运行OS X v10.5的基于intel的iMac上生成的，iMac拥有2ghz Core Duo处理器和1gb RAM。 |

在设计并发任务时，正确性始终是最重要的因素，但是您还应该考虑性能因素。在多个线程下正确执行的代码，但比在单个线程上运行的相同代码慢，这算不上什么改进。

如果要对现有的单线程应用程序进行改造，则应该始终采取一组关键任务性能的基准度量。在添加额外的线程之后，您应该对这些相同的任务进行新的度量，并比较多线程情况和单线程情况的性能。如果在调优代码之后，线程并没有提高性能，那么您可能需要重新考虑特定的实现或线程的使用。

有关性能和收集性能指标的工具的信息，请参见 *[Performance Overview](https://developer.apple.com/library/archive/documentation/Performance/Conceptual/PerformanceOverview/Introduction/Introduction.html#//apple_ref/doc/uid/TP40001410)*。有关锁和原子操作成本的具体信息，请参见[Thread Costs](https://developer.apple.com/library/archive/documentation/Cocoa/Conceptual/Multithreading/CreatingThreads/CreatingThreads.html#//apple_ref/doc/uid/10000057i-CH15-SW7)。

# 线程安全与信号

对于线程化的应用程序，没有什么比处理信号的问题更让人担心或困惑的了。信号是一种低层次的BSD机制，可以用来将信息传递给进程或以某种方式操纵它。有些程序使用信号来检测某些事件，例如子进程的退出销毁。系统使用信号来终止失控的进程并与其他类型的信息通信。

在单线程应用程序中，所有信号处理程序都运行在主线程上。在多线程应用程序中，没有绑定到特定硬件错误(如非法指令)的信号被传递给当时正在运行的任何线程。如果多个线程同时运行，信号将被传递给系统选择的任何一个线程。换句话说，信号可以传递到应用程序的任何线程。

在应用程序中实现信号处理程序的第一个规则是避免假设哪个线程在处理信号。如果一个特定的线程想要处理一个给定的信号，那么您需要找到某种方法在信号到达时通知该线程。

> You cannot just assume that installation of a signal handler from that thread will result in the signal being delivered to the same thread.

更多关于信号和配置信号处理程序，参见[signal](https://developer.apple.com/library/archive/documentation/System/Conceptual/ManPages_iPhoneOS/man3/signal.3.html#//apple_ref/doc/man/3/signal)和[sigaction](https://developer.apple.com/library/archive/documentation/System/Conceptual/ManPages_iPhoneOS/man2/sigaction.2.html#//apple_ref/doc/man/2/sigaction)

# 线程安全设计技巧

同步工具是使代码线程安全的一种有用的方法，但它们不是万灵药。如果使用过多，锁和其他类型的同步原语实际上会降低应用程序的线程性能(与非线程性能相比)。在线程安全与性能之间找到合适的平衡点是一门需要经验的艺术。以下部分提供了一些提示，帮助您为应用程序选择适当的同步手段。

## 完全避免同步

对于您所从事的任何新项目，甚至是现有项目，设计代码和数据结构以避免同步需求是可能的最佳解决方案。尽管锁和其他同步工具很有用，但它们确实会影响任何应用程序的性能。如果整个设计在特定资源之间引起高度的争用，那么线程可能会等待更长的时间。

实现并发性的最佳方法是减少并发任务之间的交互和相互依赖。如果每个任务都对自己的私有数据集进行操作，则不需要使用锁来保护这些数据。即使在两个任务共享一个公共数据集的情况下，您也可以查看对该数据集进行分区或为每个任务提供自己的副本的方法。当然，复制数据集也有其成本，所以在做出决定之前，您必须权衡这些成本与同步的成本。

## 理解同步的限制

同步工具只有在应用程序中的所有线程一致使用时才有效。如果您创建了一个互斥锁来限制对特定资源的访问，那么在尝试操作资源之前，您的所有线程都必须获得相同的互斥锁。如果不这样做，就会破坏互斥锁提供的保护，这是一个程序员错误。

## 注意代码正确性的问题

在使用锁和内存屏障时，您应该始终仔细考虑它们在代码中的位置。即使是看似放置得当的锁，实际上也会让你产生一种虚假的安全感。下面的一系列示例试图通过指出看似无害的代码中的缺陷来说明这个问题。基本前提是您有一个包含一组不可变对象的可变数组。假设您想调用数组中第一个对象的方法。你可以使用下面的代码来做到这一点:

```objective-c
NSLock* arrayLock = GetArrayLock();
NSMutableArray* myArray = GetSharedArray();
id anObject;
 
[arrayLock lock];
anObject = [myArray objectAtIndex:0];
[arrayLock unlock];
 
[anObject doSomething];
```

因为数组是可变的，所以数组周围的锁会阻止其他线程修改数组，直到某个线程获得该数组对象。而且由于您的目标对象本身是不可变的，所以在调用`doSomething`方法时不需要加锁。

但是，前面的例子有一个问题。如果你释放了锁，另一个线程进来，并在你有机会执行`doSomething`方法之前从数组中删除所有对象，会发生什么?在没有垃圾回收的应用程序中，代码所持有的对象可能会被释放，留下一个指向无效内存地址的对象。为了解决这个问题，你可以简单地重新安排你现有的代码，并在你调用`doSomething`后释放锁，如下所示:

```objective-c
NSLock* arrayLock = GetArrayLock();
NSMutableArray* myArray = GetSharedArray();
id anObject;
 
[arrayLock lock];
anObject = [myArray objectAtIndex:0];
[anObject doSomething];
[arrayLock unlock];
```

通过将`doSomething`调用移动到锁内，您的代码可以保证在调用方法时对象仍然有效。不幸的是，如果`doSomething`方法需要很长时间来执行，这可能会导致代码长时间持有锁，从而造成性能瓶颈。

代码的问题不是关键区域定义得不好，而是实际的问题没有被理解。真正的问题是内存管理问题，只有其他线程的存在才会触发这个问题。因为它可以由另一个线程释放，所以更好的解决方案是在释放锁之前保留一个对象。这个解决方案解决了对象被释放的实际问题，并且没有引入潜在的性能损失。

```objective-c
NSLock* arrayLock = GetArrayLock();
NSMutableArray* myArray = GetSharedArray();
id anObject;
 
[arrayLock lock];
anObject = [myArray objectAtIndex:0];
[anObject retain];
[arrayLock unlock];
 
[anObject doSomething];
[anObject release];
```

虽然前面的例子本质上很简单，但它们确实说明了一个非常重要的问题。当谈到正确性时，你必须考虑到明显的问题之外。内存管理和设计的其他方面也可能会受到多线程的影响，因此您必须预先考虑这些问题。此外，您应该始终假定编译器在安全方面会做最坏的事情。这种意识和警惕应该可以帮助您避免潜在的问题，并确保您的代码行为正确。

有关如何使程序线程安全的其他示例，请参阅[Thread Safety Summary](https://developer.apple.com/library/archive/documentation/Cocoa/Conceptual/Multithreading/ThreadSafetySummary/ThreadSafetySummary.html#//apple_ref/doc/uid/10000057i-CH12-SW1)。

## 注意死锁和活锁

任何时候一个线程试图同时获取多个锁，都有可能发生死锁。**当两个不同的线程持有另一个线程需要的锁，然后试图获取另一个线程持有的锁时，就会发生死锁**。结果是，每个线程都永久阻塞，因为它永远无法获得另一个锁。活锁类似于死锁，当两个线程竞争同一组资源时发生。在活锁情况下，线程放弃它的第一个锁，试图获得它的第二个锁。一旦它获得了第二个锁，它就返回并再次尝试获取第一个锁。它被锁定是因为它把所有时间都花在释放一个锁和获取另一个锁上，而不是做任何实际的工作。

假设两人正好面对面碰上对方：

- 死锁：两人互不相让，都在等对方先让开。
- 活锁：两人互相礼让，却恰巧站到同一侧，再次让开，又站到同一侧，同样的情况不断重复下去导致双方都无法通过。

**避免死锁和活锁的最好方法是一次只取一个锁。如果您必须同时获取多个锁，那么您应该确保其他线程不会尝试做类似的事情**。

## 正确使用易失性变量

如果您已经在使用互斥锁来保护某个代码段，那么不要想当然地认为您需要使用volatile关键字来保护该代码段中的重要变量。互斥锁包含一个内存屏障，以确保load和store操作的正确顺序。将volatile关键字添加到临界段中的变量中，将强制该值在每次访问时从内存中加载。在特定情况下，这两种同步技术的结合可能是必要的，但也会导致显著的性能损失。如果只有互斥锁就足以保护变量，则省略volatile关键字。

同样重要的是，不要为了避免使用互斥锁而使用易失性变量。通常，互斥锁和其他同步机制是比易失性变量更好的保护数据结构完整性的方法。volatile关键字只确保变量从内存中加载，而不是存储在寄存器中。它不能确保代码正确地访问变量。

# 使用原子操作

非阻塞同步是一种执行某些类型操作和避免锁开销的方法。尽管锁是同步两个线程的有效方法，但获取锁是一个相对昂贵的操作，即使在无争用的情况下也是如此。相比之下，许多原子操作只需要一小部分时间就可以完成，而且可以像锁一样有效。

原子操作允许您对32位或64位值执行简单的数学和逻辑操作。这些操作依赖于特殊的硬件指令(和一个可选的内存屏障)，以确保给定的操作在再次访问受影响的内存之前完成。在多线程的情况下，您应该始终使用包含内存屏障的原子操作，以确保线程之间的内存正确同步。

原子数学运算和逻辑操作及其函数名如表4-3所示。这些函数都在`/usr/include/libkern/ osatomics .h`头文件中声明，你也可以在那里找到完整的语法。这些函数的64位版本仅在64位进程中可用。

表4-3 原子数学运算和逻辑运算

| 操作       | 函数名                                                       | 描述                                                         |
| ---------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| 相加       | OSAtomicAdd32 <br>OSAtomicAdd32Barrier <br/>OSAtomicAdd64<br/>OSAtomicAdd64Barrier | 将两个整数值相加，并将结果存储在指定的一个变量中。           |
| 自增       | OSAtomicIncrement32 <br>OSAtomicIncrement32Barrier<br/>OSAtomicIncrement64<br/>OSAtomicIncrement64Barrier | 将指定的整数值加1。                                          |
| 自减       | OSAtomicDecrement32<br>OSAtomicDecrement32Barrier <br/>OSAtomicDecrement64<br/>OSAtomicDecrement64Barrier | 将指定的整数值递减1。                                        |
| 逻辑或     | OSAtomicOr32<br>OSAtomicOr32Barrier                          | 在指定的32位值和32位掩码之间执行逻辑或。                     |
| 逻辑与     | OSAtomicAnd32<br>OSAtomicAnd32Barrier                        | 在指定的32位值和32位掩码之间执行逻辑与。                     |
| 逻辑异或   | OSAtomicXor32<br>OSAtomicXor32Barrier                        | 在指定的32位值和32位掩码之间执行逻辑异或。                   |
| 比较和交换 | OSAtomicCompareAndSwap32<br>OSAtomicCompareAndSwap32Barrier<br/>OSAtomicCompareAndSwap64<br/>OSAtomicCompareAndSwap64Barrier<br/>OSAtomicCompareAndSwapPtr<br/>OSAtomicCompareAndSwapPtrBarrier<br/>OSAtomicCompareAndSwapInt<br/>OSAtomicCompareAndSwapIntBarrier<br/>OSAtomicCompareAndSwapLong<br/>OSAtomicCompareAndSwapLongBarrier | 将变量与指定的旧值进行比较。如果两个值相等，这个函数将指定的新值赋给变量;否则，它什么也不做。比较和赋值作为一个原子操作完成，函数返回一个布尔值，指示交换是否实际发生。 |
| 测试并设置 | OSAtomicTestAndSet<br>OSAtomicTestAndSetBarrier              | Tests a bit in the specified variable, sets that bit to 1, and returns the value of the old bit as a Boolean value. Bits are tested according to the formula (0x80 >> (n & 7)) of byte ((char*)address + (n >> 3)) where n is the bit number and address is a pointer to the variable. This formula effectively breaks up the variable into 8-bit sized chunks and orders the bits in each chunk in reverse. For example, to test the lowest-order bit (bit 0) of a 32-bit integer, you would actually specify 7 for the bit number; similarly, to test the highest order bit (bit 32), you would specify 24 for the bit number. |
| 测试和清空 | OSAtomicTestAndClear<br>OSAtomicTestAndClearBarrier          | Tests a bit in the specified variable, sets that bit to 0, and returns the value of the old bit as a Boolean value. Bits are tested according to the formula `(0x80 >> (n & 7)) `of byte `((char*)address + (n >> 3))` where `n` is the bit number and `address` is a pointer to the variable. This formula effectively breaks up the variable into 8-bit sized chunks and orders the bits in each chunk in reverse. For example, to test the lowest-order bit (bit 0) of a 32-bit integer, you would actually specify 7 for the bit number; similarly, to test the highest order bit (bit 32), you would specify 24 for the bit number. |

大多数原子函数的行为应该是相对简单的，正如您所期望的那样。然而，清单4-1展示了原子测试与设置操作和比较与交换操作的行为，它们稍微复杂一些。对`OSAtomicTestAndSet`函数的前三个调用演示了如何在整数值上使用位操作公式，其结果可能与您预期的不同。最后两个调用显示了`OSAtomicCompareAndSwap32`函数的行为。在所有情况下，这些函数都是在没有其他线程操作值的无争用情况下调用的。

清单4-1 执行原子操作

```c
int32_t  theValue = 0;
OSAtomicTestAndSet(0, &theValue);
// theValue is now 128.
 
theValue = 0;
OSAtomicTestAndSet(7, &theValue);
// theValue is now 1.
 
theValue = 0;
OSAtomicTestAndSet(15, &theValue)
// theValue is now 256.
 
OSAtomicCompareAndSwap32(256, 512, &theValue);
// theValue is now 512.
 
OSAtomicCompareAndSwap32(256, 1024, &theValue);
// theValue is still 512.
```

有关原子操作的信息，请参阅[atomic](https://developer.apple.com/library/archive/documentation/System/Conceptual/ManPages_iPhoneOS/man3/atomic.3.html#//apple_ref/doc/man/3/atomic)和`/usr/include/libkern/ osatomich`头文件。

# 使用锁

锁是线程编程的基本同步工具。锁使您能够轻松地保护大段代码，从而确保代码的正确性。OS X和iOS为所有应用程序类型提供了基本的互斥锁，Foundation框架为特殊情况定义了互斥锁的一些附加变量。下面几节将向您展示如何使用几种锁类型。

## 使用POSIX互斥锁

`POSIX`互斥锁在任何应用程序中都非常容易使用。要创建互斥锁，需要声明并初始化一个`pthread_mutex_t`结构体。要锁定和解锁互斥锁，可以使用`pthread_mutex_lock`和`pthread_mutex_unlock`函数。清单4-2显示了初始化和使用`POSIX`线程互斥锁所需的基本代码。使用完锁后，只需调用`pthread_mutex_destroy`来释放锁数据结构。

清单4-2 使用posix互斥锁

```c
pthread_mutex_t mutex;
void MyInitFunction()
{
    pthread_mutex_init(&mutex, NULL);
}
 
void MyLockingFunction()
{
    pthread_mutex_lock(&mutex);
    // Do work.
    pthread_mutex_unlock(&mutex);
}
```

> 注意:前面的代码是一个简化的示例，目的是展示POSIX线程互斥函数的基本用法。您自己的代码应该检查这些函数返回的错误代码，并适当地处理它们。

## 使用NSLock类

`NSLock`对象为Cocoa应用实现了一个基本的互斥锁。所有锁的接口(包括`NSLock`)实际上是由`NSLocking protocol`定义的，它定义了锁定和解锁方法。您可以使用这些方法获取和释放锁，就像使用任何互斥锁一样。

除了标准的锁定行为，NSLock类还添加了`tryLock`和`lockBeforeDate:`方法。`tryLock`方法尝试获取锁，但如果锁不可用，则不会阻塞;这种情况下，该方法只返回NO。`lockBeforeDate:`方法尝试获取锁，但如果没有在指定的时间限制内获得锁，则解除线程阻塞(并返回NO)。

下面的例子展示了如何使用NSLock对象来协调界面更新，这些显示的数据是由几个线程计算的。如果线程不能立即获得锁，它将继续计算，直到它能够获得锁并更新界面。

```objc
BOOL moreToDo = YES;
NSLock *theLock = [[NSLock alloc] init];
...
while (moreToDo) {
    /* Do another increment of calculation */
    /* until there’s no more to do. */
    if ([theLock tryLock]) {
        /* Update display used by all threads. */
        [theLock unlock];
    }
}
```

## 使用@synchronized指令

@synchronized指令是Objective-C代码中动态创建互斥锁的一种方便方法。@synchronized指令做了任何其他互斥锁都会做的事情——防止不同的线程在同一时间获取相同的锁。但是，在这种情况下，您不必直接创建互斥锁或锁对象。相反，你可以简单地使用任何Objective-C对象作为锁令牌，如下面的例子所示:

```objc
- (void)myMethod:(id)anObj
{
    @synchronized(anObj)
    {
        // Everything between the braces is protected by the @synchronized directive.
    }
}
```

传递给`@synchronized`指令的对象是用来区分受保护块的唯一标识符。如果在两个不同的线程中执行上述方法，在每个线程中为`anObj`参数传递不同的对象，每个线程都将获得其锁并继续处理，而不会被另一个线程阻塞。但是，如果在这两种情况下传递相同的对象，其中一个线程将首先获得锁，另一个线程将阻塞，直到第一个线程完成任务释放锁。

作为预防措施，@synchronized块隐式地向受保护的代码添加了一个异常处理程序。这个处理程序在抛出异常时自动释放互斥锁。这意味着为了使用@synchronized指令，你还必须在你的代码中启用Objective-C异常处理。如果您不希望隐式异常处理程序造成额外的开销，则应该考虑使用**锁**。

更多关于`@synchronized`指令，参见*[The Objective-C Programming Language](https://developer.apple.com/library/archive/documentation/Cocoa/Conceptual/ObjectiveC/Introduction/introObjectiveC.html#//apple_ref/doc/uid/TP30001163)*.

## 使用其他Cocoa锁

下面几节描述了使用其他几种类型的Cocoa锁的过程。

### 使用NSRecursiveLock（递归锁）

`NSRecursiveLock`类定义了一个锁，可以被同一个线程多次获取，而不会导致线程死锁。递归锁跟踪成功获取锁的次数。每次成功获取锁必须通过相应的调用来平衡以解锁锁。只有当所有的锁和解锁调用都被平衡时，锁才会被释放，以便其他线程可以获得它。

顾名思义，这种类型的锁通常在递归函数中使用，以防止递归阻塞线程。在非递归情况下，您可以类似地使用它来调用函数，这些函数的语义要求它们也必须具有锁定功能。下面是一个简单的递归函数的例子，它通过递归获得锁。如果没有在此代码中使用NSRecursiveLock对象，那么当再次调用该函数时，线程将死锁。

```objc
NSRecursiveLock *theLock = [[NSRecursiveLock alloc] init];
 
void MyRecursiveFunction(int value)
{
    [theLock lock];
    if (value != 0)
    {
        --value;
        MyRecursiveFunction(value);
    }
    [theLock unlock];
}
 
MyRecursiveFunction(5);
```

> 注意:因为在所有锁调用与解锁调用平衡后，才会释放递归锁，所以您应该仔细权衡使用性能锁的决定与潜在的性能影响。在一段较长的时间内保持任何锁都可能导致其他线程阻塞，直到递归完成。如果您可以重写代码以消除递归或消除使用递归锁的需要，那么您可能会获得更好的性能。

### 使用NSConditionLock(条件锁)

`NSConditionLock`对象定义了一个互斥锁，可以用特定的值锁定和解锁。您不应该将这种类型的锁与**条件**混淆。行为与条件有些相似，但实现方式非常不同。

通常，当线程需要按特定顺序执行任务时，例如当一个线程产生另一个线程使用的数据时，您可以使用`NSConditionLock`对象。当生产者执行时，消费者使用特定于程序的条件获取锁。(条件本身只是一个您定义的整数值。)当生产者完成时，它解锁锁并将锁条件设置为适当的整数值来唤醒消费者线程，然后消费者线程继续处理数据。

`NSConditionLock`对象响应的锁定和解锁方法可以以任何组合方式使用。例如，你可以将一个锁定消息和`unlockWithCondition:`匹配，或者一个`lockWhenCondition`消息与解锁匹配。后一种组合可以解锁锁，但可能不会释放等待特定条件值的任何线程。

下面的示例显示如何使用条件锁处理生产者-消费者问题。假设一个应用程序包含一个数据队列。生产者线程将数据添加到队列中，消费者线程从队列中提取数据。生产者不需要等待特定的条件，但它必须等待锁可用，这样它才能安全地向队列添加数据。

```objc
id condLock = [[NSConditionLock alloc] initWithCondition:NO_DATA];
 
while(true)
{
    [condLock lock];
    /* Add data to the queue. */
    [condLock unlockWithCondition:HAS_DATA];
}
```

因为锁的初始条件被设置为NO_DATA，所以生产者线程在初始时获得锁应该没有问题。它用数据填充队列，并将条件设置为HAS_DATA。在随后的迭代中，生产者线程可以在到达时添加新数据，而不管队列是空的还是仍然有一些数据。它唯一阻塞的时候是消费者线程从队列中提取数据的时候。

因为消费者线程必须有数据要处理，所以它使用特定的条件等待队列。当生产者将数据放入队列时，消费者线程唤醒并获取其锁。然后，它可以从队列中提取一些数据并更新队列状态。下面的示例展示了消费者线程处理循环的基本结构。

```objc
while (true)
{
    [condLock lockWhenCondition:HAS_DATA];
    /* Remove data from the queue. */
    [condLock unlockWithCondition:(isEmpty ? NO_DATA : HAS_DATA)];
 
    // Process the data locally.
}
```



### 使用NSDistributedLock(分布式锁)

`NSDistributedLock`类可以被多个主机上的多个应用程序使用，以限制对某些共享资源(如文件)的访问。锁本身实际上是一个互斥锁，它是使用文件系统项(比如文件或目录)实现的。为了让`NSDistributedLock`对象可用，这个锁必须被所有使用它的应用程序写入。这通常意味着将其放在一个文件系统上，运行该应用程序的所有计算机都可以访问该文件系统。

与其他类型的锁不同，`NSDistributedLock`不符合`NSLocking protocol`，因此没有锁定方法。锁定方法将阻止线程的执行，并要求系统以预定速率轮询锁定。 `NSDistributedLock`不会将这种限制加在您的代码上，而是提供一个`tryLock`方法，并让您决定是否进行轮询。

因为它是使用文件系统实现的，所以`NSDistributedLock`对象不会被释放，除非所有者显式地释放它。如果您的应用程序在持有分布式锁时崩溃，其他客户机将无法访问受保护的资源。在这种情况下，您可以使用`breakLock`方法来打破现有的锁，以便获得它。但是，通常应该避免解锁，除非您确定所拥有的进程已经死亡并且不能释放锁。

与其他类型的锁一样，当你使用完`NSDistributedLock`对象后，你可以通过调用`unlock`方法来释放它。


# 使用条件(Conditions)

条件是一种特殊类型的锁，可用于同步操作必须进行的顺序。它们与互斥锁有一个微妙的区别。等待某个条件的线程将保持阻塞状态，直到该条件被另一个线程显式发出信号为止。

由于在实现操作系统时涉及到的微妙之处，即使您的代码实际上没有发出信号，条件锁也允许在错误的成功情况下返回。为了避免这些虚假信号引起的问题，应该始终在条件锁中使用谓词。谓词是确定线程继续运行是否安全的更具体的方法。该条件只是保持线程处于休眠状态，直到信令线程可以设置谓词。下面几节将向您展示如何在代码中使用条件。

## 使用NSCondition类

`NSCondition`类提供了与`POSIX`条件相同的语义，但将所需的锁和条件数据结构封装在一个对象中。结果是一个对象，你可以像锁互斥一样锁住它，然后像等待条件一样等待它。

清单4-3显示了一个代码片段，演示了等待`NSCondition`对象的事件序列。`cocoaconcondition`变量包含一个`NSCondition`对象，`timeToDoWork`变量是一个整数，在发出条件信号之前从另一个线程递增。

清单4-3 使用Cocoa condition

```objc
[cocoaCondition lock];
while (timeToDoWork <= 0)
    [cocoaCondition wait];
 
timeToDoWork--;
 
// Do real work here.
 
[cocoaCondition unlock];
```

清单4-4显示了用于指示Cocoa条件和递增谓词变量的代码。在发出信号之前，您应该总是锁定条件。

```objc
[cocoaCondition lock];
timeToDoWork++;
[cocoaCondition signal];
[cocoaCondition unlock];
```

## 使用POSIX Conditions

POSIX线程条件锁需要同时使用条件数据结构和互斥锁。尽管这两种锁结构是分开的，但互斥锁在运行时与条件结构密切相关。等待信号的线程应该总是同时使用相同的互斥锁和条件结构。更改配对可能会导致错误。

清单4-5显示了条件和谓词的基本初始化和用法。在初始化条件和互斥锁之后，等待线程使用`ready_to_go`变量作为谓词进入`while`循环。只有当谓词被设置并且条件被触发时，等待的线程才会苏醒并开始工作。

清单4-5 使用POSIX condition

```objc
pthread_mutex_t mutex;
pthread_cond_t condition;
Boolean     ready_to_go = true;
 
void MyCondInitFunction()
{
    pthread_mutex_init(&mutex);
    pthread_cond_init(&condition, NULL);
}
 
void MyWaitOnConditionFunction()
{
    // Lock the mutex.
    pthread_mutex_lock(&mutex);
 
    // If the predicate is already set, then the while loop is bypassed;
    // otherwise, the thread sleeps until the predicate is set.
    while(ready_to_go == false)
    {
        pthread_cond_wait(&condition, &mutex);
    }
 
    // Do work. (The mutex should stay locked.)
 
    // Reset the predicate and release the mutex.
    ready_to_go = false;
    pthread_mutex_unlock(&mutex);
}
```

信令线程负责设置谓词和向条件锁发送信号。清单4-6显示了实现此行为的代码。在这个例子中，条件在互斥锁内部被通知，以防止等待条件的线程之间发生竞争。

清单4-6 向条件锁发送信号

```c
void SignalThreadUsingCondition()
{
    // At this point, there should be work for the other thread to do.
    pthread_mutex_lock(&mutex);
    ready_to_go = true;
 
    // Signal the other thread to begin work.
    pthread_cond_signal(&condition);
 
    pthread_mutex_unlock(&mutex);
}
```

> 注意:前面的代码是一个简化的示例，目的是展示POSIX线程条件函数的基本用法。您自己的代码应该检查这些函数返回的错误代码，并适当地处理它们。

# 源文档

[Synchronization](https://developer.apple.com/library/archive/documentation/Cocoa/Conceptual/Multithreading/ThreadSafety/ThreadSafety.html#//apple_ref/doc/uid/10000057i-CH8-SW14)

