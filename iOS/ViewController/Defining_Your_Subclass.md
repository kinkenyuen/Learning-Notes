# 目录

   * [Defining Your Subclass (自定义子类)](#defining-your-subclass-自定义子类)
      * [Defining Your UI (定义UI元素)](#defining-your-ui-定义ui元素)
      * [Handling User Interactions (处理用户交互)](#handling-user-interactions-处理用户交互)
      * [Displaying Your Views at Runtime](#displaying-your-views-at-runtime)
      * [Managing View Layout (管理视图布局)](#managing-view-layout-管理视图布局)
      * [Managing Memory Efficiently (有效地管理内存)](#managing-memory-efficiently-有效地管理内存)
   * [源文档](#源文档)

# Defining Your Subclass (自定义子类)

你使用`UIViewController`的自定义子类来呈现你的应用程序的内容，大多数自定义视图控制器是**内容视图控制器**——也就是说，它们拥有它们所有的视图并负责那些视图中的数据。相比之下，容器视图控制器并不拥有它所有的视图；它的一些视图被其他视图控制器管理。定义内容和容器视图控制器的大部分步骤是相同的，将在后面的章节中讨论。

对于内容视图控制器，最常见的父类如下:

* 使用`UITableViewController`，特别是当你的视图控制器的主视图是一个`table`
* 使用`UICollectionViewController`，特别是当你的视图控制器的主视图是一个`collection view`
* 使用`UIViewController`，对于其他一些视图控制器

对于容器视图控制器，使用什么父类取决于你是在修改一个现有的容器类还是创建你自己的。对于现有的容器，选择你想要修改的视图控制器类；对于新的容器视图控制器，你通常子类化`UIViewController`。关于创建容器视图控制器的更多信息，请参阅[Implementing a Container View Controller](https://developer.apple.com/library/archive/featuredarticles/ViewControllerPGforiPhoneOS/ImplementingaContainerViewController.html#//apple_ref/doc/uid/TP40007457-CH11-SW1)

## Defining Your UI (定义UI元素)  

在`Xcode`中使用`storyboard`文件为你的视图控制器定义`UI`，虽然你也可以通过编程方式创建你的`UI`，但是`storyboard`是一种很好的方法来可视化你的视图控制器的内容和为不同的环境定制你的视图层次结构(根据需要)。可视化地构建`UI`可以让你快速地进行更改，并让你无需编译和运行应用程序就能看到结果。

图4-1展示一个`storyboard`示例，每个矩形区域代表一个视图控制器及其相关的视图，视图控制器之间的箭头表示视图控制器的`relationships`和`segue`，`relationships`连接一个容器视图控制器到它的子视图控制器，`segue`让你的视图控制器在界面之间导航。

图4-1 

<div align="center">    
<img src="./imgs/storyboard_bird_sightings_2x.png" width="50%" height="50%">
</div>

每个新项目都有一个`main storyboard`，它通常已经包含了一个或多个视图控制器，你可以把新的视图控制器`library`拖到画布上，从而添加到你的`storyboard`中，**新的视图控制器最初没有一个相关的类，所以你必须使用 Identity inspector分配一个**。

要使用`storyboard`：

* 为视图控制器添加、排列和配置视图
* 连接`outlets`和`actions`;参阅 [Handling User Interactions](https://developer.apple.com/library/archive/featuredarticles/ViewControllerPGforiPhoneOS/DefiningYourSubclass.html#//apple_ref/doc/uid/TP40007457-CH7-SW11)
* 控制器之间创建`relationships`和`segues`，参阅[Using Segues](https://developer.apple.com/library/archive/featuredarticles/ViewControllerPGforiPhoneOS/UsingSegues.html#//apple_ref/doc/uid/TP40007457-CH15-SW1)
* 为不同的`size classes`设计自定义布局,参阅[Building an Adaptive Interface](https://developer.apple.com/library/archive/featuredarticles/ViewControllerPGforiPhoneOS/BuildinganAdaptiveInterface.html#//apple_ref/doc/uid/TP40007457-CH32-SW1)
* 添加手势识别器来处理用户与视图的交互

## Handling User Interactions (处理用户交互)

应用程序的`responder`对象处理传入事件并采取适当的操作，虽然视图控制器是`responder`对象，但它们很少直接处理触摸事件，相反，视图控制器通常以以下方式处理事件。

* 视图控制器定义`action`方法来处理更高级别的事件
  * 具体`action`，控件和一些视图调用`action`方法来报告特定的交互
  * 手势识别器(`Gesture recognizers`)，手势识别器调用一个`action`方法来报告手势的当前状态，使用你的视图控制器来处理状态变化或响应完成的手势
* 视图控制器**观察由系统或其他对象发送的通知**，通知反馈某些对象发生了变更，是视图控制器更新其状态的一种方式
* 视图控制器充当另一个对象的`data source`或`delegate`，视图控制器通常用于管理`table`和`collection`视图的数据，你还可以将控制器用作其他对象的`delegate`，例如`CLLocationManager`对象，它将更新的位置值发送给它的`delegate`

响应事件通常涉及到更新视图的内容，这需要对这些视图的引用，你的视图控制器是一个为你以后需要修改的视图定义`outlet`的好地方。使用清单4-1所示的语法将`outlets`声明为属性。清单中的自定义类定义了两个`outlet`(由`IBOutlet`关键字修饰)和一个动作方法(由`IBAction`返回类型修饰)。`outlet`存储对`storyboard`中的`button`和`text field`的引用，而`action`方法响应`button`中的点击。

清单4-1

```objective-c
@interface MyViewController : UIViewController
@property (weak, nonatomic) IBOutlet UIButton *myButton;
@property (weak, nonatomic) IBOutlet UITextField *myTextField;
 
- (IBAction)myButtonAction:(id)sender;
 
@end
```

```swift
class MyViewController: UIViewController {
    @IBOutlet weak var myButton : UIButton!
    @IBOutlet weak var myTextField : UITextField!
    
    @IBAction func myButtonAction(sender: id)
}
```

在`storyboard`中，记得将视图控制器的`outlet`和`action`连接到相应的视图，连接`storyboard`文件中的`outlet`和`action`，确保它们在视图加载时被配置。

## Displaying Your Views at Runtime

`storyboard`使加载和显示视图控制器的视图的过程非常简单，`UIKit`会在需要的时候自动从`storyboard`文件中加载视图。作为加载过程的一部分，`UIKit`执行以下一系列的任务:

1. 使用`storyboard`中的信息实例化视图

2. 连接所有`outlets`和`actions`

3. 将`root view`赋值给视图控制器的`view`属性

4. 调用视图控制的`awakeFromNib`方法

   当这个方法被调用时，视图控制器的`trait collection`是空的，视图可能还不在它们的最终位置

5. 调用视图控制器的`viewDidLoad`方法

   使用此方法可以添加或删除视图、修改布局约束以及为视图加载数据

在屏幕上显示视图控制器的视图之前，`UIKit`给你一些额外的机会来做某些操作，具体来说，`UIKit`执行以下一系列的任务:

1. 调用视图控制器的`viewWillAppear:`方法让它知道它的视图即将出现在屏幕上
2. 更新视图的布局
3. 在屏幕上展示视图
4. 当视图展示在屏幕上，调用`viewDidAppear`方法

当你添加、删除或修改视图的大小或位置时，记得添加和删除应用于这些视图的任何约束。对视图层次结构进行布局相关的更改会导致`UIKit`将布局标记为`dirty`，**在下一个更新周期中，布局引擎使用当前布局约束计算视图的大小和位置，并将这些更改应用到视图层次结构中**。

有关如何在不使用`storyboard`的情况下创建视图的信息，请参阅[UIViewController Class Reference](https://developer.apple.com/documentation/uikit/uiviewcontroller)中的视图管理信息。

## Managing View Layout (管理视图布局)

当视图的大小和位置改变时，`UIKit`为你的视图层次结构更新布局信息，对于使用自动布局配置的视图，`UIKit`使用自动布局引擎，并根据当前的约束更新布局。`UIKit`也让其他感兴趣的对象，如`presentation`控制器，知道布局的变化，以便它们可以相应地响应。

在布局过程中，`UIKit`会在某个时间点通知你，以便你可以执行额外的布局相关的任务。使用这些通知来修改你的布局约束，或在应用布局约束后对布局进行最终调整，在布局过程中，`UIKit`会对每个受影响的视图控制器执行以下操作:

1. 根据需要更新视图控制器及其视图的特征集合,参阅[When Do Trait and Size Changes Happen?](https://developer.apple.com/library/archive/featuredarticles/ViewControllerPGforiPhoneOS/TheAdaptiveModel.html#//apple_ref/doc/uid/TP40007457-CH19-SW6)

2. 调用视图控制器的`viewWillLayoutSubviews`方法

3. 调用当前` UIPresentationController` 对象的`containerViewWillLayoutSubviews`方法

4. 调用控制器根视图的`layoutSubviews`方法

   **此方法的默认实现是计算可用的约束计算新的布局信息，然后，该方法遍历视图层次结构，并为每个子视图调用layoutSubviews**

5. 将计算好的布局信息作用到所有视图上

6. 调用视图控制器的`viewDidLayoutSubViews`方法

7. 调用当前`UIPresentationController`对象的 `containerViewDidLayoutSubviews`方法

视图控制器可以使用`viewWillLayoutSubviews`和`viewDidLayoutSubviews`方法来执行可能影响布局过程的额外更新，在布局之前，你可以添加或删除视图，更新视图的大小或位置，更新约束，或更新其他与视图相关的属性。布局之后，你可以重新加载表数据，更新其他视图的内容，或者对视图的大小和位置进行最后的调整。

以下是一些有效管理你的布局的技巧:

* 使用**Auto Layout**，使用自动布局创建的约束是一种灵活和简单的方法，可以在不同的屏幕大小上定位内容
* 使用` top`和 `bottom layout guides`，按照这些`layout guide`布局内容可以确保内容始终可见。`top layout guide`引导的位置会影响状态栏和导航栏的高度。同样，`bottom layout guide`的位置也会影响标签栏或工具栏的高度。(如今已经不建议使用，取而代之的是`Safe Area Guide`)
* 如果动态地添加或删除视图，请记住更新相应的约束
* 当视图控制器的视图需要动画化时，先暂时性地移除约束，当使用`UIKit Core Animation`动画化视图，在动画持续期间时先移除约束，然后在动画完成后重新添加回约束。记住，如果你的视图的位置或大小在动画过程中发生变化，要更新你的约束

有关`presentation`控制器及其在视图控制器体系结构中扮演的角色的信息，请参阅[The Presentation and Transition Process](https://developer.apple.com/library/archive/featuredarticles/ViewControllerPGforiPhoneOS/PresentingaViewController.html#//apple_ref/doc/uid/TP40007457-CH14-SW7).

## Managing Memory Efficiently (有效地管理内存)

虽然内存分配的大部分内容是由你来决定的，表4-1列出了`UIViewController`的方法，你最有可能分配或释放内存的地方，大多数回收都涉及到删除对对象的强引用，为了移除一个对象的强引用，将指向该对象的属性和变量设置为`nil`。

表4-1 分配和释放内存的位置

| 任务                                 | 方法                      | 描述                                                         |
| ------------------------------------ | ------------------------- | ------------------------------------------------------------ |
| 分配视图控制器所需的关键数据结构内存 | Initialization methods    | 你的自定义初始化方法(不管它是否命名为' `init` '或其他形式)总是负责将你的视图控制器对象放入一个已知的良好状态。使用这些方法来分配任何需要的数据结构，以确保正确的操作。 |
| 分配或加载要在视图中显示的数据       | `viewDidLoad`             | 使用`viewDidLoad`方法来加载你想要显示的任何数据对象。当这个方法被调用时，你的视图对象已经被保证存在并且处于已知的良好状态。 |
| 响应低内存通知                       | `didReceiveMemoryWarning` | 使用这个方法来解除与你的视图控制器关联的所有非关键对象的分配，以便释放尽可能多的内存。 |
| 释放视图控制器所需的关键数据结构     | `dealloc`                 | 只有在执行视图控制器类的最后清理时重写此方法，系统会自动释放存储在类的实例变量和属性中的对象，因此不需要显式释放这些对象。 |

# 源文档

[Defining Your Subclass](https://developer.apple.com/library/archive/featuredarticles/ViewControllerPGforiPhoneOS/DefiningYourSubclass.html#//apple_ref/doc/uid/TP40007457-CH7-SW1)