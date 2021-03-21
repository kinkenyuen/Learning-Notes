# 目录

   * [Design Tips](#design-tips)
      * [Use System-Supplied View Controllers Whenever Possible (尽可能使用系统提供的视图控制器)](#use-system-supplied-view-controllers-whenever-possible-尽可能使用系统提供的视图控制器)
      * [Make Each View Controller an Island](#make-each-view-controller-an-island)
      * [Use the Root View Only as a Container for Other Views](#use-the-root-view-only-as-a-container-for-other-views)
      * [Know Where Your Data Lives](#know-where-your-data-lives)
      * [Adapt to Changes](#adapt-to-changes)
   * [源文档](#源文档)

# Design Tips

视图控制器是运行在iOS上的应用程序的必要工具，`UIKit`的视图控制器基础组件使它很容易创建复杂的界面，而不需要编写大量的代码。当实现你自己的视图控制器时，使用下面的提示和指南来确保你不会做一些可能会干扰系统预期自然行为的事情。

## Use System-Supplied View Controllers Whenever Possible (尽可能使用系统提供的视图控制器)

许多iOS框架定义了视图控制器，你可以在你的应用程序中原样使用，使用这些系统提供的视图控制器为你节省了时间，并确保了用户一致的体验

大多数系统视图控制器是为特定的任务而设计的。一些视图控制器提供对用户数据的访问，比如联系人。其他的可能提供对硬件的访问，或者提供专门调优的接口来管理媒体。例如，`UIKit`中的`UIImagePickerController`类提供了一个用于捕捉图片和视频以及访问用户设备相机的标准接口。

在你创建自己的自定义视图控制器之前，查看现有的框架，看看你想要执行的任务是否已经存在一个符合(合适)的视图控制器

* `UIKit`框架为在`iCloud`上显示警报、拍摄图片和视频以及管理文件提供了视图控制器。UIKit还定义了许多标准的容器视图控制器，你可以用它们来组织你的内容。
* `GameKit`框架为匹配玩家、管理排行榜、成就和其他游戏功能提供了视图控制器。
* `Address Book UI`框架为显示和选择联系人信息提供了视图控制器。
* `MediaPlayer`框架为播放和管理视频以及从用户库中选择媒体提供了视图控制器。
* `EventKit UI`框架为显示和编辑用户的日历数据提供了视图控制器。
* `GLKit`框架提供了一个视图控制器来管理`OpenGL`渲染层。
* `Multipeer Connectivity`框架提供了视图控制器来检测其他用户并邀请他们连接。
* `Message UI`框架为编辑电子邮件和SMS消息提供了视图控制器。
* `PassKit`框架提供了视图控制器来显示通行证并将它们添加到`Passbook`中。
* `Social framework`提供了视图控制器，用于为`Twitter`、`Facebook`和其他社交媒体站点编写消息。
* `AVFoundation`框架为显示媒体资产提供了一个视图控制器。

> 永远不要修改系统提供的视图控制器的视图层次结构，每个视图控制器拥有它的视图层次结构，并负责维护该层次结构的完整，作出改变可能会在你的代码中引入错误或阻止所属的视图控制器正确运行。在使用系统视图控制器的情况下，总是依赖于视图控制器的公共可用的方法和属性来做所有的修改。

关于使用特定视图控制器的信息，请参阅相应框架的参考文档。

## Make Each View Controller an Island

> 译注: 标题的意思可理解为，使视图控制器更多独立

视图控制器总是`self-contained`的对象，任何视图控制器都不应该知道另一个视图控制器的内部工作或视图层次结构，**在两个视图控制器需要通信或来回传递数据的情况下，它们应该总是使用显式定义的公共接口来这样做**。

`delegation`设计模式经常被用于管理视图控制器之间的通信，使用`delegate`，一个对象定义一个协议，用于与关联的`delegate`对象通信，该对象是任何遵循该协议(**遵循该协议表示实现了协议的接口**)的对象，委托对象的确切类型不重要，重要的是它实现了协议的方法。

## Use the Root View Only as a Container for Other Views

> 译注：标题：只使用根视图作为其他视图的容器

尽可能使用你的视图控制器的根视图作为你其余内容的容器，使用根视图作为容器为所有视图提供了一个通用的父视图，这使得许多布局操作更简单，许多自动布局约束需要一个公共的父视图来正确布局视图。

## Know Where Your Data Lives

在MVC设计模式中，视图控制器的作用是促进数据在模型对象和视图对象之间的移动，视图控制器可能在临时变量中存储一些数据并执行一些验证，但它的主要职责是确保它的视图包含准确的信息，数据对象负责管理实际数据，并确保数据的整体完整性。

`UIDocument`和`UIViewController`类之间的关系就是数据和接口分离的一个例子，具体来说，两者之间不存在什么缺省关系，`UIDocument`对象协调控制数据的加载和保存，而`UIViewController`对象协调控制视图在屏幕上的显示。如果你创建了两个对象之间的关系，记住视图控制器应该只缓存文档中的信息以提高效率，实际数据仍然属于`document`对象。

## Adapt to Changes

应用程序可以在各种iOS设备上运行，视图控制器被设计为适应这些设备上不同大小的屏幕，与其使用单独的视图控制器来管理不同屏幕上的内容，不如使用内置的自适应支持来响应视图控制器中的大小和`size class`的变化。`UIKit`发送的通知让你有机会对你的用户界面进行大规模和小规模的更改，而不需要更改你的视图控制器代码的其余部分。

更多关于处理自适应性的变化，请参阅[The Adaptive Model](https://developer.apple.com/library/archive/featuredarticles/ViewControllerPGforiPhoneOS/TheAdaptiveModel.html#//apple_ref/doc/uid/TP40007457-CH19-SW1)

# 源文档

[Design Tips](https://developer.apple.com/library/archive/featuredarticles/ViewControllerPGforiPhoneOS/DesignTips.html#//apple_ref/doc/uid/TP40007457-CH5-SW1)

