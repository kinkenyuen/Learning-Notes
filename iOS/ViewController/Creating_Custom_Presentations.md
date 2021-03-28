# 目录

   * [Creating Custom Presentations](#creating-custom-presentations)
      * [The Custom Presentation Process](#the-custom-presentation-process)
      * [Creating a Custom Presentation Controller](#creating-a-custom-presentation-controller)
         * [Setting the Frame of the Presented View Controller](#setting-the-frame-of-the-presented-view-controller)
         * [Managing and Animating Custom Views](#managing-and-animating-custom-views)
      * [Vending Your Presentation Controller to UIKit](#vending-your-presentation-controller-to-uikit)
      * [Adapting to Different Size Classes](#adapting-to-different-size-classes)
   * [源文档](#源文档)

# Creating Custom Presentations

`UIKit`将视图控制器的内容与内容在屏幕上的显示和显示方式分开，`presented`视图控制器被一个底层的`presentation controller`对象管理，它管理用来显示视图控制器的视图的视觉样式，`presentation`控制器可以做以下事情:

* 设置`presented`视图控制器的尺寸
* 添加自定义视图以更改所展示内容的视觉外观
* 为任何自定义视图提供过渡动画
* 当应用程序的环境发生变化时，调整展示内容的视觉外观

UIKit为标准的`presentation`样式提供了`presentation`控制器，当你设置一个视图控制器的`presentation`样式为`UIModalPresentationCustom`并提供一个正确的`transitioning delegate`，`UIKit`使用你的自定义`presentation`控制器代替。

## The Custom Presentation Process

当你`present`一个`presentation`样式是`UIModalPresentationCustom`的视图控制器时，`UIKit`寻找一个自定义的``presentation``控制器来管理`present`过程，随着`presentation`的进行，`UIKit`调用`presentation`控制器的方法，让它有机会设置任何自定义视图并将它们动画化到你设置的位置。

`presentation`控制器与`animator`对象一起工作以实现整体的转换，`animator`对象将视图控制器的内容动画到屏幕上，而`presentation`控制器处理其他一切，通常，你的`presentation`控制器会动画化它自己的视图，但你也可以重写`presentation`控制器的`presenttedView`方法并让`animator`对象动画所有或部分那些视图。

在一个`presentation`中，`UIKit`会：

1. 调用`transitioning delegate`实现的`presentationControllerForPresentedViewController:presentingViewController:sourceViewController:`方法来获取你自定义的`presentation`视图控制器

2. 如果有，询问` transitioning delegate`获取`animator`和`interactive animator`对象

3. 调用`presentation`视图控制器的` presentationTransitionWillBegin`方法

   这个方法的实现应该将任何自定义视图添加到视图层次结构中，并为这些视图配置动画。

4. 从`presentation`控制器获取`presentedView`

   这个方法返回的视图被`animator`对象动画到对应位置上，通常，这个方法返回的是`presented`视图控制器的根视图，`presentation`控制器可以根据需要使用自定义背景视图替换该视图，如果你指定了一个不同的视图，你必须嵌入`presented`视图控制器的根视图到你的视图层次结构中

5. 执行转换动画

   转换动画包括由`animator`对象创建的主要动画，以及你配置的与主要动画一起执行的任何动画。

   在动画过程中，`UIKit`调用`presentation`视图控制器的` containerViewWillLayoutSubviews`和`containerViewDidLayoutSubviews`，你可以按需要调整你自定义视图的布局。

6. 调用 `presentationTransitionDidEnd:`方法，当动画转换完成时

在一个`dismissal`中，`UIKit`会：

1. 从当前可见的视图控制器中获取自定义的`presentation`视图控制器

2. 如果有，询问` transitioning delegate`获取`animator`和`interactive animator`对象

3. 调用`presentation`视图控制器的`dismissalTransitionWillBegin`方法

   这个方法的实现应该将任何自定义视图从视图层次结构中移除，并为这些视图配置动画

4. 从`presentation`控制器获取`presentedView`

5. 执行转换动画

   转换动画包括由`animator`对象创建的主要动画，以及你配置的与主要动画一起执行的任何动画。

   在动画过程中，`UIKit`调用`presentation`视图控制器的` containerViewWillLayoutSubviews`和`containerViewDidLayoutSubviews`，你可以按需要移除你自定义视图的布局约束。

6. 调用 `dismissalTransitionDidEnd:` 方法，当转换动画完成时

在``presentation``过程中，你的``presentation``控制器的`frameOfPresentedViewInContainerView`和`presenttedView`方法可能会被调用几次，所以你的实现应该快速返回，同样，你的`presenttedView`方法的实现不应该尝试建立视图层次结构，在调用该方法时，视图层次结构应该已经配置好了。

## Creating a Custom Presentation Controller

要实现一个自定义的`presentation`样式，子类化`UIPresentationController`并添加代码来为你的`presentation`创建视图和动画，在创建自定义`presentation`控制器时，考虑以下问题:

* 你需要添加什么视图？
* 怎样动画化这些视图到屏幕上？
* 需要展示的视图控制器尺寸是多少？
* 在不同的`size class`环境下，它怎样自适应？
* 当`presentation` 完成后，`presenting`视图控制器的视图需要被移除吗？

所有这些点都需要考虑用来重写`UIPresentationController`类的不同方法

### Setting the Frame of the Presented View Controller

你可以修改`presented`视图控制器的`frame`矩形，以便它只填充部分可用空间，默认情况下，一个`presented`视图控制器的大小是完全填充容器视图的`frame`，要更改`frame`矩形，重写`presentation`控制器的`frameOfPresentedViewInContainerView`方法，清单11-1显示了一个将`frame`更改为只覆盖容器视图的右半部分的示例，在本例中，`presentation`控制器使用背景调暗的视图来覆盖容器的另一半。

清单11-1 更改`presented`视图控制器的`frame`

```objective-c
- (CGRect)frameOfPresentedViewInContainerView {
    CGRect presentedViewFrame = CGRectZero;
    CGRect containerBounds = [[self containerView] bounds];
 
    presentedViewFrame.size = CGSizeMake(floorf(containerBounds.size.width / 2.0),
                                         containerBounds.size.height);
    presentedViewFrame.origin.x = containerBounds.size.width -
                                    presentedViewFrame.size.width;
    return presentedViewFrame;
}
```

### Managing and Animating Custom Views

自定义`presentations`通常包括向所显示的内容添加自定义视图，使用自定义视图来实现纯粹的视觉装饰，或者使用它们向`presentation`添加实际行为，例如，背景视图可能包含手势识别器来跟踪所展示内容边界之外的特定动作。

`presentation`控制器负责创建和管理与其`presentation`相关联的所有自定义视图，通常，在初始化`presentations控制器时创建自定义视图，清单11-2展示了创建自己的暗层视图的自定义视图控制器的初始化方法，此方法创建视图并执行一些最小化配置。

清单11-2 初始化`presentation`视图控制器

```objective-c
- (instancetype)initWithPresentedViewController:(UIViewController *)presentedViewController
                    presentingViewController:(UIViewController *)presentingViewController {
    self = [super initWithPresentedViewController:presentedViewController
                         presentingViewController:presentingViewController];
    if(self) {
        // Create the dimming view and set its initial appearance.
        self.dimmingView = [[UIView alloc] init];
        [self.dimmingView setBackgroundColor:[UIColor colorWithWhite:0.0 alpha:0.4]];
        [self.dimmingView setAlpha:0.0];
    }
    return self;
}
```

你可以使用`presentationTransitionWillBegin`方法将自定义视图动画化到屏幕上，在这个方法中，配置你的自定义视图并将它们添加到容器视图中，如清单11-3所示，使用`presenting`或`presented`视图控制器的转换协调器(`transition coordinator`)来创建任何动画，不要在这个方法中修改`presented`视图控制器的视图。`animator`对象负责将`presented`视图控制器动画化为从`frameOfPresentedViewInContainerView`方法返回的`frame`矩形。

清单11-3 动画化暗层视图到屏幕

```objective-c
- (void)presentationTransitionWillBegin {
    // Get critical information about the presentation.
    UIView* containerView = [self containerView];
    UIViewController* presentedViewController = [self presentedViewController];
 
    // Set the dimming view to the size of the container's
    // bounds, and make it transparent initially.
    [[self dimmingView] setFrame:[containerView bounds]];
    [[self dimmingView] setAlpha:0.0];
 
    // Insert the dimming view below everything else.
    [containerView insertSubview:[self dimmingView] atIndex:0];
 
    // Set up the animations for fading in the dimming view.
    if([presentedViewController transitionCoordinator]) {
        [[presentedViewController transitionCoordinator]
               animateAlongsideTransition:^(id<UIViewControllerTransitionCoordinatorContext>
                                            context) {
            // Fade in the dimming view.
            [[self dimmingView] setAlpha:1.0];
        } completion:nil];
    }
    else {
        [[self dimmingView] setAlpha:1.0];
    }
}
```

在`presentation`结束时，使用`presentationTransitionDidEnd:`方法来处理由于取消`presentation`而引起的任何清理。如果不满足阈值条件，`interactive animator`对象可能会取消转换，当那发生时，`UIKit`用一个值`NO`调用`presentationTransitionDidEnd:`方法，当取消操作发生时，删除你在`presentation`开始时添加的任何自定义视图，并将任何其他视图恢复到它们之前的配置，如清单11-4所示

清单11-4 处理被取消的`presentation`

```objective-c
- (void)presentationTransitionDidEnd:(BOOL)completed {
    // If the presentation was canceled, remove the dimming view.
    if (!completed)
        [self.dimmingView removeFromSuperview];
}
```

当视图控制器被移除，使用`dismissalTransitionDidEnd:`方法来从视图层次结构中移除你的自定义视图，如果你想让你的视图消失动画化，在`dismissalTransitionDidEnd:`方法中设置那些动画，清单11-5展示了在前面的示例中移除暗层视图的两种方法的实现，注意，需要检查d`ismissalTransitionDidEnd:`方法的参数，看看移除是成功的还是被取消了的。

清单11-5

```objective-c
- (void)dismissalTransitionWillBegin {
    // Fade the dimming view back out.
    if([[self presentedViewController] transitionCoordinator]) {
        [[[self presentedViewController] transitionCoordinator]
           animateAlongsideTransition:^(id<UIViewControllerTransitionCoordinatorContext>
                                        context) {
            [[self dimmingView] setAlpha:0.0];
        } completion:nil];
    }
    else {
        [[self dimmingView] setAlpha:0.0];
    }
}
 
- (void)dismissalTransitionDidEnd:(BOOL)completed {
    // If the dismissal was successful, remove the dimming view.
    if (completed)
        [self.dimmingView removeFromSuperview];
}
```

## Vending Your Presentation Controller to UIKit

当`present`一个视图控制器时，使用你自定义的`presentation`控制器执行以下步骤来展示它:

* 设置`presented`视图控制器的`modalPresentationStyle` 属性为`UIModalPresentationCustom`
* 为`presented`视图控制器的 `transitioningDelegate`属性设置一个`transitioning delegate`
* 实现`transitioning delegate`的`presentationControllerForPresentedViewController:presentingViewController:sourceViewController:`方法

当需要你的`presentation`控制器时，`UIKit`调用`presentationControllerForPresentedViewController:presentingViewController:sourceViewController:`，这个方法的实现应该像清单11-6中那样简单，只需创建该控制器，配置它，并返回它，如果你从这个方法返回`nil`, `UIKit`使用全屏显示风格展示视图控制器。

清单11-6 创建自定义`presentation`控制器

```objective-c
- (UIPresentationController *)presentationControllerForPresentedViewController:
                                 (UIViewController *)presented
        presentingViewController:(UIViewController *)presenting
            sourceViewController:(UIViewController *)source {
 
    MyPresentationController* myPresentation = [[MyPresentationController]
       initWithPresentedViewController:presented presentingViewController:presenting];
 
    return myPresentation;
}
```

## Adapting to Different Size Classes

当特征(`trait`)或容器视图的尺寸发生变化时，`UIKit`会通知你的`presentation`控制器，这种变化通常发生在设备旋转时，但也可能发生在其他时候，你可以使用`trait`和`size`通知来调整`presentation`的自定义视图，并适当地更新`presentation`样式。

有关如何自适应新的`traits`和`sizes`，请参阅[Building an Adaptive Interface](https://developer.apple.com/library/archive/featuredarticles/ViewControllerPGforiPhoneOS/BuildinganAdaptiveInterface.html#//apple_ref/doc/uid/TP40007457-CH32-SW1)

# 源文档

[Creating Custom Presentations](https://developer.apple.com/library/archive/featuredarticles/ViewControllerPGforiPhoneOS/DefiningCustomPresentations.html#//apple_ref/doc/uid/TP40007457-CH25-SW1)

