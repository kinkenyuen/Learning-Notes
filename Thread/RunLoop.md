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
**模式根据事件的来源进行区分，而不是事件的类型。**例如，您不能使用模式仅匹配鼠标点击事件或键盘事件。您可以使用模式来监听一组不同的端口，暂时挂起计时器，或者以其他方式更改源和正在监视的runloop observers。

表3-1列出了Cocoa和Core Foundation定义的标准模式，以及使用该模式的描述。第二列列出了用于在代码中指定模式的实际常量。

|Mode|Name| Description |
|---|:--|---|
| Default |NSDefaultRunLoopMode (Cocoa)<br>kCFRunLoopDefaultMode (Core Foundation) | 默认模式是用于大多数操作的模式。大多数情况下，您应该使用这种模式来启动运行循环并配置输入源。|
| Connection | NSConnectionReplyMode (Cocoa) | Cocoa使用这种模式与NSConnection对象一起监控应答。您自己应该很少需要使用这种模式。|
| Modal |NSModalPanelRunLoopMode (Cocoa)|Cocoa使用这种模式来识别用于模态面板的事件。|
|Event tracking |NSEventTrackingRunLoopMode (Cocoa)| Cocoa使用此模式处理鼠标拖动以及其他一些用户交互事件|
|Common modes| NSRunLoopCommonModes (Cocoa)<br>kCFRunLoopCommonModes (Core Foundation)| 这是一组可配置的常用模式。**将输入源与此模式相关联意味着将其与组中的每个模式相关联。**对于Cocoa应用程序，该集合默认包括Default、Modal和Event tracking模式。Core Foundation最初只包含Default模式。您可以使用CFRunLoopAddCommonMode函数来添加自定义模式。|

## 输入源
输入源异步地向线程传递事件。**事件的来源取决于输入源的类型，输入源通常是两类之一。**基于端口的输入源监视应用程序的Mach端口。自定义输入源监视事件的自定义源。就runloop而言，输入源是基于端口的还是自定义的并不重要。系统通常实现两种类型的输入源，您可以按原样使用。这两个源之间唯一的区别是它们如何发出信号。**基于端口的源由内核自动发出信号，自定义源必须从另一个线程手动发出信号。**

当您创建输入源时，您将它分配给runloop的一个或多个模式。模式会影响这些输入源，大多数情况下，您会在default模式下运行运行循环，但是您也可以指定自定义模式。如果输入源不在当前监视模式下，则它生成的任何事件都将被保存，直到runloop以输入源对应的模式运行。

以下部分介绍了一些输入源。

### 基于端口的源

Cocoa和Core Foundation为使用与端口相关的对象和函数创建基于端口的输入源提供了内置支持。例如，在Cocoa中，您根本不需要直接创建输入源。您只需创建一个端口对象，并使用NSPort的方法将该端口添加到runloop中。port对象为您处理所需输入源的创建和配置。

在Core Foundation中，您必须手动创建端口及其运行循环源。在这两种情况下，都使用与端口不透明类型(CFMachPortRef、CFMessagePortRef或CFSocketRef)相关联的函数来创建适当的对象。

### 自定义源

要创建自定义输入源，必须使用Core Foundation中与CFRunLoopSourceRef不透明类型关联的函数。您可以使用几个回调函数来配置自定义输入源。Core Foundation在不同的时机调用这些函数来配置源，处理任何传入的事件，并在runloop移除源的时候销毁这些源。

>In addition to defining the behavior of the custom source when an event arrives, you must also define the event delivery mechanism. This part of the source runs on a separate thread and is responsible for providing the input source with its data and for signaling it when that data is ready for processing. The event delivery mechanism is up to you but need not be overly complex.

### Cocoa Perform Selector 源

除了基于端口的源，Cocoa还定义了一个定制的输入源，允许您在任何线程上执行选择器。与基于端口的源一样，在目标线程上序列化执行选择器请求，解决了在一个线程上运行多个方法时可能出现的许多同步问题。与基于端口的源不同，执行选择器源在执行选择器后将自己从runloop中移除。

当在另一个线程上执行选择器时，目标线程必须有一个活跃的runloop。对于您创建的线程，这意味着需要你自己手动运行runloop。但是，因为主线程启动了它自己的runloop，所以只要应用程序调用应用程序委托的`applicationDidFinishLaunching:`方法，您就可以开始对该线程发出调用。每次runloop都会处理所有在队列中的Selector，而不是在每次循环迭代中处理一个。

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

与在适当的异步或同步事件发生时触发的源相反，runLoop o  bservers在runloop本身执行期间的特定位置触发。 您可以使用runLoop observers来让线程准备处理给定的事件，或者在线程进入睡眠之前做某些操作。你可以将以下事件与runLoop observers关联:

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

只有在为应用程序创建辅助线程时，才需要显式地运行runloop。应用程序主线程的runloop是基础结构的关键部分。应用程序框架提供了运行主应用程序循环的代码，并自动启动该循环。UIApplication在iOS(或NSApplication在OS X)中的run方法作为正常启动序列的一部分来启动应用程序的主循环。如果你使用Xcode模板项目来创建你的应用，你不应该显式地调用这些例程。



对于辅助线程，您需要决定是否需要runloop，如果需要，请自己配置并启动它。您不需要在所有情况下都启动线程的运行循环。例如，如果您使用一个线程来执行一些长时间运行的预先确定的任务，您可能可以避免启动runloop。运行循环适用于您希望与线程进行更多交互的情况。例如，如果你计划做以下任何一件事，你需要启动一个runloop:

* 使用port或自定义输入源与其他线程通信
* 在线程上使用计时器
* 在Cocoa应用程序中使用任何``performSelector`… `方法
* 线程需要周期性执行任务

如果您选择使用runloop，配置和设置是简单的。但是，与所有线程编程一样，您应该有计划在适当的情况下退出辅助线程。让线程退出总比强制终止线程好。

# 使用RunLoop对象

## 获取RunLoop对象

## 配置RunLoop对象

## 开启RunLoop

## 退出RunLoop

## 线程安全和RunLoop对象

# 配置RunLoop源

## 定义自定义输入源

## 配置计时器源

## 配置基于端口的输入源

