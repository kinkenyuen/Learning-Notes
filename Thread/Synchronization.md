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



# 使用条件(Conditions)



