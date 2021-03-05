# RunLoop

runloop是与线程相关的基础结构部分。runloop是一个事件处理循环，您可以使用它来安排工作和协调接收处理传入的事件。runloop的目的是使线程有工作任务的时候保持忙碌，在没有工作任务的时候休眠。

runloop的管理并不是完全自动的。您仍然需要设计线程相关的代码，以便在适当的时机启动runloop并响应传入的事件。`Cocoa`和`Core Foundation`都提供了runloop对象来帮助你配置和管理线程的runloop。你的应用程序不需要显示创建这些runloop对象，**每个线程（包括应用程序的主线程）都有一个关联的runloop对象。但是，只有辅助线程需要显示运行它们的runloop**。**应用程序启动时，系统框架会自动在主线程上配置并运行runloop**

以下部分介绍runloop的信息和如何在应用程序中配置它们

# 剖析RunLoop
runloop就像它的名字描述那样，它是线程进入并用于触发回调以响应传入事件的循环。您的代码提供了用于实现runloop的实际循环部分的控制语句——换句话说，您的代码提供了驱动运行循环的while或for循环。在循环中，使用run loop对象“运行”事件处理代码，接收事件并调用已注册的回调。

**runloop从两种不同类型的源接收事件。 输入源传递异步事件，通常是来自另一个线程或其他应用程序的消息。计时器源传递同步事件，这些事件在计划的时间或重复的间隔发生。 两种类型的源都使用特定于应用程序的回调来处理到达时的事件。**

图3-1展示了运行循环的概念结构和各种源。输入源将异步事件传递给相应的处理程序，并促使rununtidate:方法(在线程的相关NSRunLoop对象上调用)退出。计时器源将事件交付给它们的处理程序例程，但不会导致运行循环退出。

图3-1 runloop结构与输入源

![1](./imgs/runloop/runloop.jpg)

除了处理输入源之外，runloop还会生成关于运行runloop的通知。注册的runloop observers可以接收这些通知，并使用它们在线程上做额外的处理。您可以使用Core Foundation在线程上配置runloop observers。

以下部分提供了关于runloop的组件和它们的运行模式的更多信息。它们还描述了在事件处理期间不同时间生成的通知。

## RunLoop 模式

**runloop mode是输入源、计时器以及runloop observers的集合。每次运行runloop时，都需要指定（显式或隐式）特定的模式。在runloop运行过程中，只有与当前该模式相关的source被监听并处理它们的回调(类似地，只有与该模式相关的observers才会被通知runloop的状态)。与其他模式关联的源将保留任何新事件，直到随后以对应的模式进入runloop。**

在你的代码中，可以通过名称来区分runloop模式。`Cocoa`和`Core Foundation`都定义了一个默认模式和几个常用模式，以及用于在代码中指定这些模式的字符串。您可以通过指定自定义字符串来定义自定义模式（一般很少这样做），自定义模式的名称是任意的，但这些模式的内容可不是任意的。**你必须确保添加一个或多个输入源、计时器或runloop observer添加到自己创建的任何模式中，这样runloop模式才会起作用。**

在runloop的一次迭代过程中，可以使用模式过滤掉本次不需要的源事件。大多数情况下，你会希望在系统定义的“默认”模式下运行runloop。然而，一个`modal panel`会在"modal"模式下运行。在此模式下，只有与`modal panel`相关的源才会将事件传递给线程。对于辅助线程，您可以使用自定义模式来防止低优先级源在计时期间传递事件。 

> For secondary threads, you might use custom modes to prevent low-priority sources from delivering events during time-critical operations.

>注意:
模式根据事件的来源进行区分，而不是事件的类型。例如，您不能使用模式仅匹配鼠标点击事件或键盘事件。您可以使用模式来监听一组不同的端口，暂时挂起计时器，或者以其他方式更改源和正在监视的runloop observers。

表3-1列出了Cocoa和Core Foundation定义的标准模式，以及使用该模式的描述。第二列列出了用于在代码中指定模式的实际常量。

|Mode|Name| Description |
|---|:--|---|
| Default |NSDefaultRunLoopMode (Cocoa)<br>kCFRunLoopDefaultMode (Core Foundation) | 默认模式是用于大多数操作的模式。大多数情况下，您应该使用这种模式来启动运行循环并配置输入源。|
| Connection | NSConnectionReplyMode (Cocoa) | Cocoa使用这种模式与NSConnection对象一起监控应答。您自己应该很少需要使用这种模式。|
| Modal |NSModalPanelRunLoopMode (Cocoa)|Cocoa使用这种模式来识别用于模态面板的事件。|
|Event tracking |NSEventTrackingRunLoopMode (Cocoa)| Cocoa使用此模式处理鼠标拖动以及其他一些用户交互事件|
|Common modes| NSRunLoopCommonModes (Cocoa)<br>kCFRunLoopCommonModes (Core Foundation)| 这是一组可配置的常用模式。将输入源与此模式相关联意味着将其与组中的每个模式相关联。对于Cocoa应用程序，该集合默认包括Default、Modal和Event tracking模式。Core Foundation最初只包含Default模式。您可以使用CFRunLoopAddCommonMode函数来添加自定义模式。 |

## 输入源
输入源异步地向线程传递事件。**事件的来源取决于输入源的类型，输入源通常是两类之一**。基于端口的输入源监视应用程序的Mach端口。自定义输入源监视事件的自定义源。就runloop而言，输入源是基于端口的还是自定义的并不重要。系统通常实现两种类型的输入源，您可以按原样使用。这两个源之间唯一的区别是它们如何发出信号。**基于端口的源由内核自动发出信号，自定义源必须从另一个线程手动发出信号。**

当您创建输入源时，您将它分配给runloop的一个或多个模式。模式会影响这些输入源，大多数情况下，您会在default模式下运行runloop，但是您也可以指定自定义模式。如果输入源不在当前监视模式下，则它生成的任何事件都将被保存，直到runloop以输入源对应的模式运行。

以下部分介绍了一些输入源。

### 基于端口的源

Cocoa和Core Foundation为使用与端口相关的对象和函数创建基于端口的输入源提供了内置支持。例如，在Cocoa中，您根本不需要直接创建输入源。您只需创建一个端口对象，并使用NSPort的方法将该端口添加到runloop中。port对象为您处理所需输入源的创建和配置。

在Core Foundation中，您必须手动创建端口及其runloop源。在这两种情况下，都使用与端口不透明类型(CFMachPortRef、CFMessagePortRef或CFSocketRef)相关联的函数来创建适当的对象。

### 自定义源

要创建自定义输入源，必须使用Core Foundation中与CFRunLoopSourceRef不透明类型关联的函数。您可以使用几个回调函数来配置自定义输入源。Core Foundation在不同的时机调用这些函数来配置源，处理任何传入的事件，并在runloop移除源的时候销毁这些源。

>In addition to defining the behavior of the custom source when an event arrives, you must also define the event delivery mechanism. This part of the source runs on a separate thread and is responsible for providing the input source with its data and for signaling it when that data is ready for processing. The event delivery mechanism is up to you but need not be overly complex.

### Cocoa Perform Selector 源

除了基于端口的源，Cocoa还定义了一个自定义的输入源，允许您在任何线程上执行选择器。与基于端口的源一样，在目标线程上序列化执行选择器请求，解决了在一个线程上运行多个方法时可能出现的许多同步问题。与基于端口的源不同，执行选择器源在执行选择器后将自己从runloop中移除。

**当在另一个线程上执行选择器时，目标线程必须有一个活跃的runloop**。对于您创建的线程，这意味着需要你自己手动运行runloop。但是，因为主线程启动了它自己的runloop，所以只要应用程序调用应用程序委托的`applicationDidFinishLaunching:`方法，您就可以开始对该线程发出调用。每次runloop都会处理所有在队列中的Selector，而不是在每次循环迭代中处理一个。

表3-2列出了定义在NSObject上的方法，这些方法可以用于在其他线程上执行选择器。因为这些方法是在NSObject上声明的，你可以在任何你能访问Objective-C对象的线程中使用它们，包括POSIX线程。这些方法实际上并没有创建新线程来执行选择器。

表3-2  在其他线程上执行选择器

| Methods | Description |
|---|---|
| performSelectorOnMainThread:withObject:waitUntilDone:<br>performSelectorOnMainThread:withObject:waitUntilDone:modes: | 在应用程序主线程的下一个runloop中执行指定的选择器。这些方法提供了阻塞当前线程的选项，直到执行选择器。 |
| performSelector:onThread:withObject:waitUntilDone:<br>performSelector:onThread:withObject:waitUntilDone:modes:| 对任何有NSThread对象的线程执行指定的选择器。这些方法提供了阻塞当前线程的选项，直到执行选择器。|
| performSelector:withObject:afterDelay:<br>performSelector:withObject:afterDelay:inModes: | 在下一个运行循环周期和可选的延迟周期之后，对当前线程执行指定的选择器。因为它会等待下一个runloop来执行选择器，所以这些方法会默认提供一个最小延迟，从当前的执行的代码开始计起。如果队列中有多个Selector，会顺序依次执行|
| cancelPreviousPerformRequestsWithTarget:<br>cancelPreviousPerformRequestsWithTarget:selector:object:| 取消在当前线程执行Selector，这些Selector是通过`performSelector:withObject:afterDelay:` 或 `performSelector:withObject:afterDelay:inModes:`方法添加的|

## 计时器源

计时器源在将来的预设时间同步地向线程传递事件。定时器是线程通知自己去做某事的一种方式。

**尽管它生成基于时间的通知，但计时器并不是一种实时机制。与输入源一样，计时器与runloop的特定模式相关联。如果计时器不在runloop当前监视的模式中，则在您以计时器支持的模式之一运行runloop之前，计时器不会触发。**

您可以将定时器配置为只生成一次或多次事件。重复计时器根据预定的触发时间(而不是实际的触发时间)自动重新调度自己。例如，如果一个计时器计划在一个特定的时间触发，并且在那之后每5秒触发一次，那么计划的触发时间将总是落在原来的5秒时间间隔上，即使实际的触发时间被延迟了。如果触发时间延迟太久，以致错过了一个或多个预定的触发次数，则计时器在错过的时间段内只触发一次。在某个时间间隔内定时器错过了触发，它将会重新计算下一个触发时间。

## RunLoop Observers

与在适当的异步或同步事件发生时触发的源相反，runLoop observers在runloop本身执行期间的特定位置触发。 您可以使用runLoop observers来让线程准备处理给定的事件，或者在线程进入睡眠之前做某些操作。你可以将以下事件与runLoop observers关联:

- The entrance to the run loop. 
- When the run loop is about to process a timer.
- When the run loop is about to process an input source.
- When the run loop is about to go to sleep.
- When the run loop has woken up, but before it has processed the event that woke it up.
- The exit from the run loop.

您可以使用Core Foundation将runLoop observers添加到应用程序。 要创建runLoop observers，请创建CFRunLoopObserverRef透明类型的新实例。 此类型跟踪您的自定义回调函数及其感兴趣的活动。 

与计时器类似，runLoop observers可以使用一次或多次。一次性的observer在触发后从runloop中移除自己，而重复的observer仍然在。您可以指定创建一个observer时是只运行一次还是重复运行。

## RunLoop事件序列

每次运行runloop时，都会处理需要处理的事件，并生成通知发送给runloop中的observers，顺序如下:

1. 通知observers已经进入runloop
2. 通知observers有就绪的计时器即将触发
3. 通知observers有非基于port的输入源即将触发
4. 触发处理已经就绪的非基于port的输入源
5. 如果有基于port的输入源就绪和等待处理，立即处理，并跳转到9
6. 通知observers线程即将进入休眠状态
7. 将线程置于休眠状态直到以下任一事件发生:
   * 一个基于port的输入源事件到达
   * 计时器触发
   * 为runloop设置的超时值到期。
   * runloop被显示唤醒
8. 通知observers线程刚被唤醒
9. 处理待办事件
   * 如果用户定义的计时器触发，处理计时器事件并重新启动loop，并跳转到2
   * 如果一个输入源事件触发，传递该事件
   * 如果runloop被显式唤醒但尚未超时，则重新启动loop，并跳转到2
10. 通知observers runloop已经退出

因为计时器和输入源的observer通知先于它们的事件处理，所以这两个事情之间会有时间间隔。（这不是废话吗？）如果这些事件之间的时间非常关键，则可以使用sleep和waking -from-sleep通知来帮助您关联实际事件之间的时间。

因为计时器和其他周期性事件是在运行runloop时交付的，绕过该循环会中断这些事件的交付。当您通过进入循环并重复地向应用程序请求事件来实现鼠标跟踪例程时，就会出现这种行为的典型示例。因为您的代码是直接抓取事件，而不是让应用程序正常地分发这些事件，活动计时器将无法触发，直到鼠标跟踪例程退出并将控制权返回给应用程序。(就是有UI操作的时候，计时器无法工作)

可以使用run loop对象显式地唤醒运行循环。其他事件也可能导致运行循环被唤醒。例如，添加另一个非基于端口的输入源会唤醒运行循环，以便可以立即处理输入源，而不是等待其他事件发生。

# 什么时候使用RunLoop

**只有在为应用程序创建辅助线程时，才需要显式地运行runloop。应用程序主线程的runloop是基础结构的关键部分。应用程序框架提供了运行主应用程序循环的代码，并自动启动该循环。UIApplication在iOS(或NSApplication在OS X)中的run方法作为正常启动序列的一部分来启动应用程序的主循环**。如果你使用Xcode模板项目来创建你的应用，你不应该显式地调用这些例程。



对于辅助线程，您需要决定是否需要runloop，如果需要，请自己配置并启动它。您不需要在所有情况下都启动线程的运行循环。例如，如果您使用一个线程来执行一些长时间运行的预先确定的任务，您可能可以避免启动runloop。运行循环适用于您希望与线程进行更多交互的情况。例如，如果你计划做以下任何一件事，你需要启动一个runloop:

* 使用port或自定义输入源与其他线程通信
* 在线程上使用计时器
* 在Cocoa应用程序中使用任何`performSelector… `方法
* 线程需要周期性执行任务

如果您选择使用runloop，配置和设置是简单的。但是，与所有线程编程一样，您应该有计划在适当的情况下退出辅助线程。让线程退出总比强制终止线程好。

# 使用RunLoop对象

runloop对象提供了添加输入源、定时器和runloop observer到runloop并运行的主要接口。每个线程都有与其相关的runloop对象。在Cocoa中，这个对象是NSRunLoop类的实例。在底层应用程序中，它是一个指针，指向CFRunLoopRef这个不透明类型。

## 获取RunLoop对象

要获取当前线程的runloop，可以使用以下方法:

* 在Cocoa应用中，使用`currentRunLoop`类方法获取一个runloop对象
* 使用`CFRunLoopGetCurrent`函数

虽然两者不是`toll-free bridged`类型，但你可以从`NSRunLoop`对象中获取一个`CFRunLoopRef`不透明类型，`NSRunLoop`类定义了一个`getCFRunLoop`方法，该方法返回一个`CFRunLoopRef`类型，这个返回类型可以传给`Core Foundation`里的函数。因为它们引用的是同一个runloop对象，你可以根据需求混合使用这两种runloop对象。

## 配置RunLoop对象

**在辅助线程上运行runloop之前，你必须为runloop至少添加一个输入源或计时器**。如果runloop没有任何源，它会立刻退出。

除了添加源之外，你也可以添加runloop observers，使用它们可以检测runloop的不同阶段状态。要添加runloop observer，你可以创建一个`CFRunLoopObserverRef`不透明类型，然后使用`CFRunLoopAddObserver`函数添加到你的runloop。**runloop observers必须使用`CoreFoundation`创建，即使是Cocoa应用程序。**

清单3-1展示了在线程入口函数中为当前线程的runloop添加runloop observer。这个例子的目的是展示如何创建runloop observer，代码中简单地设置了一个runloop observer去监视所有的runloop活动。这里的回调只是简单展示runloop的活动当它执行计时器请求时。(回调没有展示出来)

清单3-1 创建runloop observer

```objective-c
- (void)threadMain
{
    // The application uses garbage collection, so no autorelease pool is needed.
    NSRunLoop* myRunLoop = [NSRunLoop currentRunLoop];
 
    // Create a run loop observer and attach it to the run loop.
    CFRunLoopObserverContext  context = {0, self, NULL, NULL, NULL};
    CFRunLoopObserverRef    observer = CFRunLoopObserverCreate(kCFAllocatorDefault,
            kCFRunLoopAllActivities, YES, 0, &myRunLoopObserver, &context);
 
    if (observer)
    {
        CFRunLoopRef    cfLoop = [myRunLoop getCFRunLoop];
        CFRunLoopAddObserver(cfLoop, observer, kCFRunLoopDefaultMode);
    }
 
    // Create and schedule the timer.
    [NSTimer scheduledTimerWithTimeInterval:0.1 target:self
                selector:@selector(doFireTimer:) userInfo:nil repeats:YES];
 
    NSInteger    loopCount = 10;
    do
    {
        // Run the run loop 10 times to let the timer fire.
        [myRunLoop runUntilDate:[NSDate dateWithTimeIntervalSinceNow:1]];
        loopCount--;
    }
    while (loopCount);
}
```

对于长期运行的线程，最好添加至少一个输入源去接收消息。虽然你可以只添加一个计时器源到runloop中，一旦计时器出发，不久后它就会失效，这就会导致runloop退出。附加一个重复计时器可以使运行循环运行更长的时间，但需要定期触发计时器以唤醒线程，这实际上是另一种轮询形式。相反，一个输入源让线程休眠，等待事件发生。

## 开启RunLoop

应用程序中的辅助线程才需要手动启动runloop。一个runloop必须至少有一个输入源或计时器来监视。如果没有附加，则runloop立即退出。有几种方法可以开始runloop，包括以下几种:

* Unconditionally (无条件地)
* With a set time limit
* In a particular mode

无条件地进入runloop是最简单的选择，但也是最不可取的选择。无条件地运行runloop会将线程置于永久循环中，这使您几乎无法控制runloop本身。您可以添加和删除输入源和计时器，但停止runloop的唯一方法是杀死它。没有办法在自定义模式中运行runloop。

与其无条件地运行runloop，不如使用超时值运行runloop。当使用超时值时，runloop将一直运行，直到事件到达或分配的时间到期。如果事件到达，则将该事件分派给处理程序进行处理，然后runloop退出。然后，代码可以重新启动runloop来处理下一个事件。如果所分配的时间过期了，您可以简单地重新启动runloop或使用这段时间来做任何需要的管理工作。

除了超时值之外，您还可以使用特定模式运行runloop。模式和超时值不是互斥的，在启动runloop时都可以使用。模式限制向runloop交付事件的源的类型。

清单3-2展示了线程的主入口例程的框架版本。这个例子的关键部分展示了runloop的基本结构。本质上，您将输入源和计时器添加到runloop中，然后重复调用其中一个例程来启动runloop。每次runloop例程返回时，都要检查是否出现了任何可能导致退出线程的条件。示例使用Core Foundation runloop例程，以便检查返回结果并确定runloop退出的原因。如果您正在使用Cocoa，并且不需要检查返回值，您也可以使用NSRunLoop类的方法以类似的方式运行runloop。

清单3-2 运行一个runloop

```objective-c
- (void)skeletonThreadMain
{
    // Set up an autorelease pool here if not using garbage collection.
    BOOL done = NO;
 
    // Add your sources or timers to the run loop and do any other setup.
 
    do
    {
        // Start the run loop but return after each source is handled.
        SInt32    result = CFRunLoopRunInMode(kCFRunLoopDefaultMode, 10, YES);
 
        // If a source explicitly stopped the run loop, or if there are no
        // sources or timers, go ahead and exit.
        if ((result == kCFRunLoopRunStopped) || (result == kCFRunLoopRunFinished))
            done = YES;
 
        // Check for any other exit conditions here and set the
        // done variable as needed.
    }
    while (!done);
 
    // Clean up code here. Be sure to release any allocated autorelease pools.
}
```

可以递归地运行runloop。换句话说，你可以调用CFRunLoopRun、CFRunLoopRunInMode或任何NSRunLoop方法来从输入源或计时器的处理程序例程中启动runloop。在执行此操作时，您可以使用runloop的任何模式，包括外部runloop正在使用的模式。

## 退出RunLoop

有两种方法可以在runloop处理完一个事件**之前**退出:

* 配置runloop以使用超时值运行。(到点就结束)
* 主动通知runloop要退出

指定超时值可以让runloop在退出之前完成所有正常处理，包括向runloop observer发送通知。

使用CFRunLoopStop函数显式地停止runloop会产生类似超时的结果。runloop发送剩余的runloop notifications，然后退出。不同的是，您可以无条件地在运行中的runloop中使用这种技术。

尽管删除runloop的输入源和计时器也可能导致runloop退出，但这不是停止runloop的可靠方法。一些系统例程将输入源添加到runloop中以处理所需的事件。因为您的代码可能不知道这些输入源，因此无法删除它们，也就不能这样退出runloop。

## 线程安全和RunLoop对象

线程安全性取决于您使用哪个API来操作runloop。Core Foundation中的函数通常是线程安全的，可以从任何线程调用。如果你在配置runloop后做一些线程相关的操作，那么最好只在runloop所在线程上执行。

> If you are performing operations that alter the configuration of the run loop, however, it is still good practice to do so from the thread that owns the run loop whenever possible.

Cocoa NSRunLoop类本身并不像它的核心基础类那样线程安全。如果你使用NSRunLoop类来修改你的runloop，你应该只从拥有那个runloop的同一个线程中这样做。向属于不同线程的runloop添加输入源或计时器可能会导致代码崩溃或以意外的方式运行。

# 配置RunLoop源

下面的小节展示了如何在Cocoa和Core Foundation中设置不同类型的输入源。

## 定义自定义输入源

创建自定义输入源需要定义以下内容:

* The information you want your input source to process.（希望输入源处理的信息。）
* A scheduler routine to let interested clients know how to contact your input source.(调度程序，让感兴趣的客户知道如何联系您的输入源。)
* A handler routine to perform requests sent by any clients.(一个处理程序例程，用于执行任何客户端发送的请求。)
* A cancellation routine to invalidate your input source.(使输入源失效的取消例程。)

因为您创建了一个自定义输入源来处理自定义信息，所以实际的配置被设计得非常灵活。调度程序、处理程序和取消例程是自定义输入源几乎总是需要的关键例程。然而，其余的大部分输入源行为都发生在这些处理程序例程之外。例如，由您定义将数据传递到输入源以及将输入源的存在传递给其他线程的机制。

图3-2显示了自定义输入源的配置示例。在本例中，应用程序的主线程维护对输入源的引用、该输入源的自定义命令缓冲区以及配置了输入源的runloop。**当主线程有一个任务想要交给工作线程时，它会向命令缓冲区发送一个命令以及工作线程启动任务所需的任何信息**。(因为主线程和工作线程的输入源都可以访问命令缓冲区，所以访问必须同步。)  一旦发出命令，主线程向输入源发出信号，并唤醒工作线程的runloop。在接收到唤醒命令后，runloop调用输入源的处理程序，该处理程序处理在命令缓冲区中找到的命令。

图3-2 操作自定义输入源

![custominputsource](./imgs/runloop/custominputsource.jpg)

下面几节将解释实现上图中的自定义输入源，并显示需要实现的关键代码。

### 定义输入源

定义自定义输入源需要使用Core Foundation例程来配置runloop源并将其附加到runloop。尽管基本的处理程序是基于C的函数，但这并不妨碍您封装这些函数，并使用Objective-C或c++来实现代码体。

图3-2中引入的输入源使用一个Objective-C对象来管理命令缓冲区并与runloop协调。**清单3-3展示了该对象的定义。RunLoopSource对象管理一个命令缓冲区，并使用该缓冲区接收来自其他线程的消息。这个清单还展示了RunLoopContext对象的定义，它实际上只是一个容器对象，用于将RunLoopSource对象和runloop引用传递给应用程序的主线程。**

清单3-3 自定义输入源对象定义

```objective-c
@interface RunLoopSource : NSObject
{
    CFRunLoopSourceRef runLoopSource;
    NSMutableArray* commands;
}
 
- (id)init;
- (void)addToCurrentRunLoop;
- (void)invalidate;
 
// Handler method
- (void)sourceFired;
 
// Client interface for registering commands to process
- (void)addCommand:(NSInteger)command withData:(id)data;
- (void)fireAllCommandsOnRunLoop:(CFRunLoopRef)runloop;
 
@end
 
// These are the CFRunLoopSourceRef callback functions.
void RunLoopSourceScheduleRoutine (void *info, CFRunLoopRef rl, CFStringRef mode);
void RunLoopSourcePerformRoutine (void *info);
void RunLoopSourceCancelRoutine (void *info, CFRunLoopRef rl, CFStringRef mode);
 
// RunLoopContext is a container object used during registration of the input source.
@interface RunLoopContext : NSObject
{
    CFRunLoopRef        runLoop;
    RunLoopSource*        source;
}
@property (readonly) CFRunLoopRef runLoop;
@property (readonly) RunLoopSource* source;
 
- (id)initWithSource:(RunLoopSource*)src andLoop:(CFRunLoopRef)loop;
@end
```

尽管Objective-C代码管理输入源的自定义数据，但将输入源附加到runloop需要基于c的回调函数。当您将runloop源附加到runloop时，将调用第一个函数，如清单3-4所示。因为这个输入源只有一个客户机(主线程)，所以它使用调度器函数发送消息，以便将自己注册到该线程上的应用程序委托中。当委托想要与输入源通信时，它使用RunLoopContext对象中的信息来完成此操作。

清单3-4 调度运行循环源 

```objective-c
void RunLoopSourceScheduleRoutine (void *info, CFRunLoopRef rl, CFStringRef mode)
{
    RunLoopSource* obj = (RunLoopSource*)info;
    AppDelegate*   del = [AppDelegate sharedAppDelegate];
    RunLoopContext* theContext = [[RunLoopContext alloc] initWithSource:obj andLoop:rl];
 
    [del performSelectorOnMainThread:@selector(registerSource:)
                                withObject:theContext waitUntilDone:NO];
}
```

最重要的回调例程之一是当输入源有信号时用来处理自定义数据的例程。清单3-5显示了与RunLoopSource对象相关联的执行回调例程。此函数只是将执行工作的请求转发给sourceFired方法，该方法然后处理命令缓冲区中存在的任何命令。

清单3-5 在输入源中执行的工作

```objective-c
void RunLoopSourcePerformRoutine (void *info)
{
    RunLoopSource*  obj = (RunLoopSource*)info;
    [obj sourceFired];
}
```

如果你使用CFRunLoopSourceInvalidate函数从它的runloop中移除输入源，系统会调用你的输入源的取消例程。您可以使用这个例程来通知客户机您的输入源不再有效，并且应该删除对它的任何引用。清单3-6显示了用RunLoopSource对象注册的取消回调例程。此函数将另一个RunLoopContext对象发送给应用程序委托，但这一次要求委托删除对运行循环源的引用。

清单3-6 使输入源失效

```objective-c
void RunLoopSourceCancelRoutine (void *info, CFRunLoopRef rl, CFStringRef mode)
{
    RunLoopSource* obj = (RunLoopSource*)info;
    AppDelegate* del = [AppDelegate sharedAppDelegate];
    RunLoopContext* theContext = [[RunLoopContext alloc] initWithSource:obj andLoop:rl];
 
    [del performSelectorOnMainThread:@selector(removeSource:)
                                withObject:theContext waitUntilDone:YES];
}
```

> **Note:** The code for the application delegate’s `registerSource:` and `removeSource:` methods is shown in [Coordinating with Clients of the Input Source](https://developer.apple.com/library/archive/documentation/Cocoa/Conceptual/Multithreading/RunLoopManagement/RunLoopManagement.html#//apple_ref/doc/uid/10000057i-CH16-SW37). 

### 添加输入源到runloop

清单3-7显示了RunLoopSource类的init和addToCurrentRunLoop方法。init方法创建CFRunLoopSourceRef 不透明类型，该类型实际上必须附加到runloop中。它将RunLoopSource对象本身作为上下文信息传递，这样回调例程就有一个指向该对象的指针。直到工作线程调用addtocurrenttrunloop方法，输入源的添加才会触发，此时会调用RunLoopSourceScheduleRoutine回调函数。输入源添加到runloop后，线程可以运行它的runloop来等待它。

清单3-7 添加runloop源

```objective-c
- (id)init
{
    CFRunLoopSourceContext    context = {0, self, NULL, NULL, NULL, NULL, NULL,
                                        &RunLoopSourceScheduleRoutine,
                                        RunLoopSourceCancelRoutine,
                                        RunLoopSourcePerformRoutine};
 
    runLoopSource = CFRunLoopSourceCreate(NULL, 0, &context);
    commands = [[NSMutableArray alloc] init];
 
    return self;
}
 
- (void)addToCurrentRunLoop
{
    CFRunLoopRef runLoop = CFRunLoopGetCurrent();
    CFRunLoopAddSource(runLoop, runLoopSource, kCFRunLoopDefaultMode);
}

```

### 协调客户端的输入来源

为了使您的输入源有用，您需要对它进行操作，并从另一个线程向它发出信号。输入源的全部意义在于将其关联的线程置于休眠状态，直到有事情要做。这就需要让应用程序中的其他线程知道输入源，并有办法与之通信。输入源的全部目的是让其关联的线程进入休眠状态，直到有事要做。这就需要让应用程序中的其他线程知道输入源，并有办法与之通信。

通知客户端有关输入源的一种方法是在输入源第一次添加到runloop中时发送注册请求。

> You can register your input source with as many clients as you want, or you can simply register it with some central agency that then vends your input source to interested clients. 

清单3-8展示了应用程序委托定义并在调用RunLoopSource对象的scheduler函数时调用的注册方法。该方法接收RunLoopSource对象提供的RunLoopContext对象，并将其添加到源列表中。这个清单还展示了在从其runloop中删除输入源时用于注销输入源的例程。

清单3-8 向应用程序委托注册和删除输入源

```objc
- (void)registerSource:(RunLoopContext*)sourceInfo;
{
    [sourcesToPing addObject:sourceInfo];
}
 
- (void)removeSource:(RunLoopContext*)sourceInfo
{
    id    objToRemove = nil;
 
    for (RunLoopContext* context in sourcesToPing)
    {
        if ([context isEqual:sourceInfo])
        {
            objToRemove = context;
            break;
        }
    }
 
    if (objToRemove)
        [sourcesToPing removeObject:objToRemove];
}
```

> **Note:** The callback functions that call the methods in the preceding listing are shown in [Listing 3-4](https://developer.apple.com/library/archive/documentation/Cocoa/Conceptual/Multithreading/RunLoopManagement/RunLoopManagement.html#//apple_ref/doc/uid/10000057i-CH16-SW32) and [Listing 3-6](https://developer.apple.com/library/archive/documentation/Cocoa/Conceptual/Multithreading/RunLoopManagement/RunLoopManagement.html#//apple_ref/doc/uid/10000057i-CH16-SW34). 

### 向输入源发送信号

在将数据传递给输入源之后，客户端必须向源发出信号，并唤醒它的runloop。向源发送信号可以让runloop知道源可以被处理。而且，由于信号发生时线程可能处于休眠状态，因此您应该总是显式地唤醒runloop。

清单3-9展示了RunLoopSource对象的fireCommandsOnRunLoop方法。当准备为输入源执行添加到buffer中的命令时，客户端会调用该方法。

清单3-9 唤醒runloop

```objective-c
- (void)fireCommandsOnRunLoop:(CFRunLoopRef)runloop
{
    CFRunLoopSourceSignal(runLoopSource);
    CFRunLoopWakeUp(runloop);
}
```

> **Note:** You should never try to handle a `SIGHUP` or other type of process-level signal by messaging a custom input source. The Core Foundation functions for waking up the run loop are not signal safe and should not be used inside your application’s signal handler routines. For more information about signal handler routines, see the `sigaction` man page.

## 配置计时器源

要创建计时器源，您所要做的就是创建一个计时器对象并在运行循环中调度它。在Cocoa中，你可以使用NSTimer类来创建新的计时器对象，而在Core Foundation中，你可以使用CFRunLoopTimerRef opaque类型。在内部，NSTimer类只是Core Foundation的扩展，它提供了一些便利功能，例如使用同一方法创建和安排计时器的功能。

在Cocoa中，你可以使用这些类方法中的任何一种来一次性创建和调度一个计时器:

* `scheduledTimerWithTimeInterval:target:selector:userInfo:repeats:`
* `scheduledTimerWithTimeInterval:invocation:repeats:`

这些方法创建计时器，并将其添加到默认模式(`NSDefaultRunLoopMode`)下的当前线程runloop中。你也可以通过创建你的`NSTimer`对象来手动调度一个计时器，然后使用NSRunLoop的`addTimer:forMode:`方法将它添加到运行循环中。这两种做法基本上是相同的，但后者你可以配置你的计时器。例如，如果您手动创建计时器并将其添加到runloop中，那么您可以使用默认模式之外的模式来完成此操作。清单3-10展示了如何使用这两种技术创建计时器。第一个计时器的初始延迟为1秒，但之后每0.1秒定时触发一次。第二个计时器在最初的0.2秒延迟后开始触发，然后每0.2秒触发一次。

清单3-10 创建并调度计时器

```objective-c
NSRunLoop* myRunLoop = [NSRunLoop currentRunLoop];
 
// Create and schedule the first timer.
NSDate* futureDate = [NSDate dateWithTimeIntervalSinceNow:1.0];
NSTimer* myTimer = [[NSTimer alloc] initWithFireDate:futureDate
                        interval:0.1
                        target:self
                        selector:@selector(myDoFireTimer1:)
                        userInfo:nil
                        repeats:YES];
[myRunLoop addTimer:myTimer forMode:NSDefaultRunLoopMode];
 
// Create and schedule the second timer.
[NSTimer scheduledTimerWithTimeInterval:0.2
                        target:self
                        selector:@selector(myDoFireTimer2:)
                        userInfo:nil
                        repeats:YES];
```

清单3-11展示了使用核心Foundation函数配置计时器所需的代码。虽然这个示例没有在上下文结构中传递任何用户定义的信息，但您可以使用这个结构来传递计时器所需的任何自定义数据。

清单3-11使用Core Foundation创建和调度计时器

```c
CFRunLoopRef runLoop = CFRunLoopGetCurrent();
CFRunLoopTimerContext context = {0, NULL, NULL, NULL, NULL};
CFRunLoopTimerRef timer = CFRunLoopTimerCreate(kCFAllocatorDefault, 0.1, 0.3, 0, 0,
                                        &myCFTimerCallback, &context);
 
CFRunLoopAddTimer(runLoop, timer, kCFRunLoopCommonModes);

```



## 配置基于端口的输入源

Cocoa和Core Foundation都提供了基于端口的对象，用于线程或进程之间的通信。下面几节向您展示如何使用几种不同类型的端口设置端口通信。

### 配置NSMachPort对象

**要使用NSMachPort对象建立本地连接，需要创建port对象并将其添加到主线程的运行循环中。在启动辅助线程时，将相同的对象传递给线程的入口点函数。辅助线程可以使用相同的对象将消息发送回主线程。**

实现主线程代码

清单3-12展示启动辅助工作线程的主线程代码。由于Cocoa框架执行了许多配置端口和运行循环的中间步骤，因此launchThread方法明显比它的核心基础等效方法短(清单3-17);然而，两者的行为几乎是相同的。一个不同之处在于，这个方法不是将本地端口的名称发送给辅助线程，而是直接发送NSPort对象。

清单3-12 主线程启动方法

```objc
- (void)launchThread
{
    NSPort* myPort = [NSMachPort port];
    if (myPort)
    {
        // This class handles incoming port messages.
        [myPort setDelegate:self];
 
        // Install the port as an input source on the current run loop.
        [[NSRunLoop currentRunLoop] addPort:myPort forMode:NSDefaultRunLoopMode];
 
        // Detach the thread. Let the worker release the port.
        [NSThread detachNewThreadSelector:@selector(LaunchThreadWithPort:)
               toTarget:[MyWorkerClass class] withObject:myPort];
    }
}
```

为了在线程之间建立双向通信通道，您可能希望工作线程在签入消息中将其自己的本地端口发送到主线程。 接收到签入消息可以使您的主线程知道在启动第二个线程时一切进展顺利，还为您提供了一种向该线程发送更多消息的方式。

清单3-13展示了主线程的handlePortMessage：方法。 当数据到达线程自己的本地端口时，将调用此方法。 当签入消息到达时，该方法直接从端口消息中检索辅助线程的端口，并将其保存以供以后使用。

```objc
#define kCheckinMessage 100
 
// Handle responses from the worker thread.
- (void)handlePortMessage:(NSPortMessage *)portMessage
{
    unsigned int message = [portMessage msgid];
    NSPort* distantPort = nil;
 
    if (message == kCheckinMessage)
    {
        // Get the worker thread’s communications port.
        distantPort = [portMessage sendPort];
 
        // Retain and save the worker port for later use.
        [self storeDistantPort:distantPort];
    }
    else
    {
        // Handle other messages.
    }
}
```

实现辅助线程代码

对于辅助工作线程，您必须配置该线程并使用指定的端口将信息传回给主线程。

清单3-14 展示了设置工作线程的代码。在为线程创建一个自动释放池之后，该方法创建一个worker对象来驱动线程执行。worker对象的sendCheckinMessage:方法(如清单3-15所示)为工作线程创建一个本地端口，并将签入消息发送回主线程。

```objc
+(void)LaunchThreadWithPort:(id)inData
{
    NSAutoreleasePool*  pool = [[NSAutoreleasePool alloc] init];
 
    // Set up the connection between this thread and the main thread.
    NSPort* distantPort = (NSPort*)inData;
 
    MyWorkerClass*  workerObj = [[self alloc] init];
    [workerObj sendCheckinMessage:distantPort];
    [distantPort release];
 
    // Let the run loop process things.
    do
    {
        [[NSRunLoop currentRunLoop] runMode:NSDefaultRunLoopMode
                            beforeDate:[NSDate distantFuture]];
    }
    while (![workerObj shouldExit]);
 
    [workerObj release];
    [pool release];
}
```

当使用NSMachPort时，本地和远程线程可以使用同一个端口对象在线程之间进行单向通信。换句话说，一个线程创建的本地端口对象成为另一个线程的远程端口对象。

清单3-15 使用Mach端口发送签入消息

```objc
// Worker thread check-in method
- (void)sendCheckinMessage:(NSPort*)outPort
{
    // Retain and save the remote port for future use.
    [self setRemotePort:outPort];
 
    // Create and configure the worker thread port.
    NSPort* myPort = [NSMachPort port];
    [myPort setDelegate:self];
    [[NSRunLoop currentRunLoop] addPort:myPort forMode:NSDefaultRunLoopMode];
 
    // Create the check-in message.
    NSPortMessage* messageObj = [[NSPortMessage alloc] initWithSendPort:outPort
                                         receivePort:myPort components:nil];
 
    if (messageObj)
    {
        // Finish configuring the message and send it immediately.
        [messageObj setMsgId:setMsgid:kCheckinMessage];
        [messageObj sendBeforeDate:[NSDate date]];
    }
}
```

### 配置NSMessagePort对象

要使用NSMessagePort对象建立本地连接，不能简单地在线程之间传递端口对象。必须根据名称获取远程消息端口。在Cocoa中实现这一点需要用特定的名称注册本地端口，然后将该名称传递给远程线程，以便它可以获得适当的端口对象进行通信。清单3-16展示了希望使用消息端口的情况下的端口创建和注册过程。

清单3-16 注册消息端口

```objc
NSPort* localPort = [[NSMessagePort alloc] init];
 
// Configure the object and add it to the current run loop.
[localPort setDelegate:self];
[[NSRunLoop currentRunLoop] addPort:localPort forMode:NSDefaultRunLoopMode];
 
// Register the port using a specific name. The name must be unique.
NSString* localPortName = [NSString stringWithFormat:@"MyPortName"];
[[NSMessagePortNameServer sharedInstance] registerPort:localPort
                     name:localPortName];
```

### 在Core Foundation中配置基于端口的输入源

本节展示如何使用Core Foundation在应用程序的主线程和工作线程之间建立双向通信通道。

清单3-17展示了应用程序主线程调用来启动工作线程的代码。代码要做的第一件事是设置CFMessagePortRef opaque类型，以侦听来自工作线程的消息。工作线程需要端口的名称来进行连接，以便将字符串值传递给工作线程的入口点函数。端口名称在当前用户上下文中通常应该是唯一的;否则，你可能会遇到冲突。

清单3-17 将Core Foundation消息端口附加到一个新线程

```objc
#define kThreadStackSize        (8 *4096)
 
OSStatus MySpawnThread()
{
    // Create a local port for receiving responses.
    CFStringRef myPortName;
    CFMessagePortRef myPort;
    CFRunLoopSourceRef rlSource;
    CFMessagePortContext context = {0, NULL, NULL, NULL, NULL};
    Boolean shouldFreeInfo;
 
    // Create a string with the port name.
    myPortName = CFStringCreateWithFormat(NULL, NULL, CFSTR("com.myapp.MainThread"));
 
    // Create the port.
    myPort = CFMessagePortCreateLocal(NULL,
                myPortName,
                &MainThreadResponseHandler,
                &context,
                &shouldFreeInfo);
 
    if (myPort != NULL)
    {
        // The port was successfully created.
        // Now create a run loop source for it.
        rlSource = CFMessagePortCreateRunLoopSource(NULL, myPort, 0);
 
        if (rlSource)
        {
            // Add the source to the current run loop.
            CFRunLoopAddSource(CFRunLoopGetCurrent(), rlSource, kCFRunLoopDefaultMode);
 
            // Once installed, these can be freed.
            CFRelease(myPort);
            CFRelease(rlSource);
        }
    }
 
    // Create the thread and continue processing.
    MPTaskID        taskID;
    return(MPCreateTask(&ServerThreadEntryPoint,
                    (void*)myPortName,
                    kThreadStackSize,
                    NULL,
                    NULL,
                    NULL,
                    0,
                    &taskID));
}
```

安装了端口并启动了线程后，主线程可以在等待线程签入的同时继续它的常规执行。当签入消息到达时，它被分派到主线程的MainThreadResponseHandler函数，如清单3-18所示。这个函数提取工作线程的端口名，并为未来的通信创建管道。

清单3-18 接收签入消息

```objective-c
#define kCheckinMessage 100
 
// Main thread port message handler
CFDataRef MainThreadResponseHandler(CFMessagePortRef local,
                    SInt32 msgid,
                    CFDataRef data,
                    void* info)
{
    if (msgid == kCheckinMessage)
    {
        CFMessagePortRef messagePort;
        CFStringRef threadPortName;
        CFIndex bufferLength = CFDataGetLength(data);
        UInt8* buffer = CFAllocatorAllocate(NULL, bufferLength, 0);
 
        CFDataGetBytes(data, CFRangeMake(0, bufferLength), buffer);
        threadPortName = CFStringCreateWithBytes (NULL, buffer, bufferLength, kCFStringEncodingASCII, FALSE);
 
        // You must obtain a remote message port by name.
        messagePort = CFMessagePortCreateRemote(NULL, (CFStringRef)threadPortName);
 
        if (messagePort)
        {
            // Retain and save the thread’s comm port for future reference.
            AddPortToListOfActiveThreads(messagePort);
 
            // Since the port is retained by the previous function, release
            // it here.
            CFRelease(messagePort);
        }
 
        // Clean up.
        CFRelease(threadPortName);
        CFAllocatorDeallocate(NULL, buffer);
    }
    else
    {
        // Process other messages.
    }
 
    return NULL;
}
```

主线程配置好之后，剩下的唯一事情就是让新创建的工作线程创建自己的端口并签入。清单3-19展示了工作线程的入口点函数。该函数提取主线程的端口名，并使用它创建回主线程的远程连接。然后，该函数为自己创建一个本地端口，将该端口添加在线程的runloop上，并向包含本地端口名称的主线程发送签入消息。

清单3-19 设置线程结构

```objc
OSStatus ServerThreadEntryPoint(void* param)
{
    // Create the remote port to the main thread.
    CFMessagePortRef mainThreadPort;
    CFStringRef portName = (CFStringRef)param;
 
    mainThreadPort = CFMessagePortCreateRemote(NULL, portName);
 
    // Free the string that was passed in param.
    CFRelease(portName);
 
    // Create a port for the worker thread.
    CFStringRef myPortName = CFStringCreateWithFormat(NULL, NULL, CFSTR("com.MyApp.Thread-%d"), MPCurrentTaskID());
 
    // Store the port in this thread’s context info for later reference.
    CFMessagePortContext context = {0, mainThreadPort, NULL, NULL, NULL};
    Boolean shouldFreeInfo;
    Boolean shouldAbort = TRUE;
 
    CFMessagePortRef myPort = CFMessagePortCreateLocal(NULL,
                myPortName,
                &ProcessClientRequest,
                &context,
                &shouldFreeInfo);
 
    if (shouldFreeInfo)
    {
        // Couldn't create a local port, so kill the thread.
        MPExit(0);
    }
 
    CFRunLoopSourceRef rlSource = CFMessagePortCreateRunLoopSource(NULL, myPort, 0);
    if (!rlSource)
    {
        // Couldn't create a local port, so kill the thread.
        MPExit(0);
    }
 
    // Add the source to the current run loop.
    CFRunLoopAddSource(CFRunLoopGetCurrent(), rlSource, kCFRunLoopDefaultMode);
 
    // Once installed, these can be freed.
    CFRelease(myPort);
    CFRelease(rlSource);
 
    // Package up the port name and send the check-in message.
    CFDataRef returnData = nil;
    CFDataRef outData;
    CFIndex stringLength = CFStringGetLength(myPortName);
    UInt8* buffer = CFAllocatorAllocate(NULL, stringLength, 0);
 
    CFStringGetBytes(myPortName,
                CFRangeMake(0,stringLength),
                kCFStringEncodingASCII,
                0,
                FALSE,
                buffer,
                stringLength,
                NULL);
 
    outData = CFDataCreate(NULL, buffer, stringLength);
 
    CFMessagePortSendRequest(mainThreadPort, kCheckinMessage, outData, 0.1, 0.0, NULL, NULL);
 
    // Clean up thread data structures.
    CFRelease(outData);
    CFAllocatorDeallocate(NULL, buffer);
 
    // Enter the run loop.
    CFRunLoopRun();
}
```

一旦进入runloop，发送到线程端口的所有未来事件都将由`ProcessClientRequest`函数处理。该函数的实现取决于线程所做的工作类型，这里没有显示。

# 源文档

[Run Loops](https://developer.apple.com/library/archive/documentation/Cocoa/Conceptual/Multithreading/RunLoopManagement/RunLoopManagement.html#//apple_ref/doc/uid/10000057i-CH16-SW10)