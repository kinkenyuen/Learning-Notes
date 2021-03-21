# The Role of View Controllers

`View controllers`(以下称视图控制器)是应用程序内部结构的基础，每个应用程序至少有一个视图控制器，大多数应用程序有多个。每个视图控制器管理应用程序的用户界面，以及该界面和底层数据之间的交互。视图控制器还具备了用户界面不同部分之间的转换。

它们在你的应用中扮演重要的角色，视图控制器几乎是你做任何事情的核心。`UIViewController`类定义方法和属性来管理你的视图，处理事件，从一个视图控制器转换到另一个，并与应用程序的其他部分协调。你子类化`UIViewController`(或它的子类之一)，并添加自定义代码实现你的应用程序的行为。

主要有两种类型的`view controllers`：

* **内容视图控制器**(*`Content view controllers`* )管理你的应用程序的内容的一个离散的片段，它是你创建的视图控制器的主要类型
* **容器视图控制器**(`Container view controllers`)从其他视图控制器(称为子视图控制器)收集信息，并以方便导航或以不同的方式显示那些视图控制器的内容来呈现它。

大多数App混合使用这两种类型

## View Management (管理视图)

视图控制器最重要的角色是管理视图的层次结构，每个视图控制器都有一个单独的根视图，它包含了视图控制器的所有内容，在根视图中，添加显示内容所需的视图。图1-1说明了视图控制器和它的视图之间的内置关系。**视图控制器总是有一个对它的根视图的引用，而每个视图都有对它的子视图的强引用**。

图1-1 视图控制器和它的视图之间关系

<div align="center">    
<img src="./imgs/VCPG_ControllerHierarchy_fig_1-1_2x.png" width="50%" height="50%">
</div>

> 使用outlet访问视图控制器的视图层次结构中的其他视图是一种常见的做法。因为视图控制器管理它所有视图的内容，所以outlet允许你存储对你需要的视图的引用。当从`storyboard`加载视图时，outlet本身会自动连接到实际的视图对象。

内容视图控制器自己管理它所有的视图，一个容器视图控制器管理它自己的视图以及来自它的一个或多个子视图控制器的根视图。容器不管理其子容器的内容，它只管理根视图，根据容器的设计调整大小并放置它。图1-2说明了分屏视图控制器(`split view controller`)及其子控制器之间的关系，分屏视图控制器管理它的子视图的总体大小和位置，但子视图控制器管理那些视图的实际内容。

图1-2 视图控制器可以管理来自其他视图控制器的内容

<div align="center">    
<img src="./imgs/VCPG_ContainerViewController_fig_1-2_2x.png" width="50%" height="50%">
</div>

有关管理视图控制器的视图的信息，请参阅[Managing View Layout](https://developer.apple.com/library/archive/featuredarticles/ViewControllerPGforiPhoneOS/DefiningYourSubclass.html#//apple_ref/doc/uid/TP40007457-CH7-SW6)

