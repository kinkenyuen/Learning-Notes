# Anatomy of a Constraint(剖析约束)

视图层次结构的布局被定义为一系列**线性方程**。每个约束都代表一个方程。你的目标是声明一系列有且只有一个可能解的方程。

示例方程如下所示。

<div align="center">    
<img src="./imgs/view_formula_2x.png" width="100%" height="100%">
</div>

这个约束规定，红色视图的`leading`(前面)必须距离蓝色视图`trailing`(后面)8个屏幕点。它的方程有几个部分:

* **Item 1** 等式中的第一项——在本例中是红色视图。`item`必须是视图或`layout guide`。
* **Attribute 1** 要约束在第一个`item`上的属性——在本例中，是红色视图的`leading`。
* **Relationship** 左右两边的关系。关系可以是三个值中的其中一个:**等于**、**大于或等于**或**小于或等于**。在这种情况下，左右两边相等。
* **Multiplier** 属性2的值乘以这个浮点数。在本例中，乘数是1.0。
* **Item 2** 等式中的第二项——在本例中是蓝色视图。与第一项不同的是，可以将其留空。
* **Attribute 2** 限制在第二个`item`上的属性——在本例中是蓝色视图的`trai`。如果第二项为空，则这必须不是一个属性。
* **Constant** 一个常量，浮点偏移量——在本例中为8.0。这个值被添加到属性2的值中。

在我们的用户界面中，大多数约束定义了两个**item**之间的关系。这些`item`可以是视图或`layout guide`。约束还可以定义单个`item`的两个不同属性之间的关系，例如，设置`item`的高度和宽度之间的宽高比。还可以将常量值分配给`item`的高度或宽度。当使用常量值时，第二项为空，第二个属性设置为非属性，乘数设置为0.0。

## Auto Layout Attributes(自动布局属性)

在自动布局中，属性定义了一个可以被约束的特性。一般来说，这包括四边(`edge`)(前面(`leading`)、后面(`trailing`)、顶部(`top`)和底部(`bottom`))，以及高度(`height`)、宽度(`width`)和垂直(`vertical`)和水平(`horizontal`)中心。文本`items`也有一个或多个`baseline`属性。

<div align="center">    
<img src="./imgs/attributes_2x.png" width="60%" height="60%">
</div>



有关属性的完整列表，请参阅`NSLayoutAttribute`枚举。

>注意：尽管OS X和iOS都使用NSLayoutAttribute枚举，但它们定义的值稍有不同。要查看完整的属性列表，请确保查看的是正确的平台文档。

## Sample Equations(方程示例)

这些方程中广泛的参数和属性可以让你创建许多不同类型的约束。你可以定义视图之间的空间，对齐视图的边缘，定义两个视图的相对大小，甚至定义一个视图的宽高比。然而，并不是所有的属性都是兼容的。

属性有两种基本类型。大小属性(例如，高度和宽度)和位置属性(例如，`Leading`、`Left`和`Top`)。大小属性用于指定`item`的大小，而不指示其位置。位置属性用于指定`item`相对于其他东西的位置。然而，他们没有显示项目的大小。

考虑到这些差异，需要遵循以下规则:

* 不能将大小属性约束为位置属性。
* 不能将常量值分配给位置属性。
* 不能在位置属性中使用非标识乘数(1.0以外的值)。
* 对于位置属性，不能将垂直属性约束为水平属性。
* 对于位置属性，不能将`Leading`属性或`Trailing`属性约束为`Left`属性或`Right`属性。

例如，如果没有额外的上下文，将`item`的`Top`设置为常量值20.0就没有意义。你必须总是定义一个`item`的位置属性与其他`item`的关系，例如，在父视图顶部的20.0屏幕点以下。但是，将`item`的高度设置为20.0是完全有效的。有关更多信息，请参阅[Interpreting Values](https://developer.apple.com/library/archive/documentation/UserExperience/Conceptual/AutolayoutPG/AnatomyofaConstraint.html#//apple_ref/doc/uid/TP40010853-CH9-SW22)。

清单3-1列出了各种常见约束的示例方程。

>注意：本章所有的例子方程都是以伪代码形式给出，要查看使用实际代码的实际约束，请参阅 [Programmatically Creating Constraints](https://developer.apple.com/library/archive/documentation/UserExperience/Conceptual/AutolayoutPG/ProgrammaticallyCreatingConstraints.html#//apple_ref/doc/uid/TP40010853-CH16-SW1)或 [Auto Layout Cookbook](https://developer.apple.com/library/archive/documentation/UserExperience/Conceptual/AutolayoutPG/LayoutUsingStackViews.html#//apple_ref/doc/uid/TP40010853-CH3-SW1).

清单3-1 约束方程

```
/ Setting a constant height
View.height = 0.0 * NotAnAttribute + 40.0
 
// Setting a fixed distance between two buttons
Button_2.leading = 1.0 * Button_1.trailing + 8.0
 
// Aligning the leading edge of two buttons
Button_1.leading = 1.0 * Button_2.leading + 0.0
 
// Give two buttons the same width
Button_1.width = 1.0 * Button_2.width + 0.0
 
// Center a view in its superview
View.centerX = 1.0 * Superview.centerX + 0.0
View.centerY = 1.0 * Superview.centerY + 0.0
 
// Give a view a constant aspect ratio
View.height = 2.0 * View.width + 0.0

```

> 注意：在重新排序item时，请确保将乘数和常数颠倒过来。例如，常数8变成-8。2.0的乘数变成0.5。常数0.0和乘数1.0保持不变。

你会发现自动布局经常提供多种方法来解决相同的问题。理想情况下，你应该选择最清楚地描述你意图的解决方案。然而，不同的开发人员无疑会对哪种解决方案是最好的产生分歧。在这一点上，保持一致性比正确要好。如果你选择了一种方法并一直坚持下去，从长远来看，你将会经历较少的问题。例如，本教程使用以下经验法则:

1. **整数乘数比分数乘数更受欢迎**。
2. **正的常数比负的常数更受欢迎**。
3. **在可能的情况下，视图应该按照布局顺序出现:从上到下依次排列**。

## Creating Nonambiguous, Satisfiable Layouts(创建明确的、可满足的布局)

当使用自动布局时，目标是提供一系列方程，其中只有一个可能的解。模糊约束有不止一个可能的解。不可满足的约束没有有效解。

通常，约束必须定义每个视图的大小和位置(两个维度)。假设父视图的大小已经设置好了(例如，iOS中的一个场景的根视图)，**一个无歧义的、可满足的布局需要每个视图每个维度两个约束**(不包括父视图)。然而，在选择要使用哪些约束时，你有很多选择。例如，以下三种布局都能产生明确的、可满足的布局(只展示水平约束):

<div align="center">    
<img src="./imgs/constraint_examples_2x.png" width="100%" height="100%">
</div>
* 第一个布局限制了视图的`leading`相对于父视图的`leading`。它也给视图一个固定的宽度。`trailing`的位置可以根据父视图的大小和其他约束来计算。
* 第二个布局限制了视图的`leading`相对于父视图的`leading`。它也限制了视图的`trailing`相对于父视图的`trailing`。视图的宽度可以根据父视图的大小和其他约束来计算。
* 第三个布局限制了视图的`leading`相对于父视图的`leading`。它也会让视图和父视图居中对齐。宽度和`trailing`的位置都可以根据父视图的大小和其他约束来计算。

注意，每个布局都有一个视图和两个水平约束。在每种情况下，约束完全定义了视图的宽度和水平位置。这意味着所有的布局都会沿着水平轴产生一个明确的、可满足的布局。**然而，这些布局并不是同样有用。考虑一下当父视图的宽度改变时会发生什么**。

在第一个布局中，视图的宽度没有改变。大多数时候，这不是你想要的。实际上，作为一般规则，**你应该避免给视图分配常量大小**。自动布局是设计来创建布局，动态地适应它们的环境。当你约束一个固定大小的视图时，你就会限制这个特性。

这可能不是很明显，但是第二和第三个布局产生了相同的行为:**当父视图的宽度改变时，它们都在视图和它的父视图之间保持一个固定的边界**。然而，他们并不一定是平等的。一般来说，第二个例子更容易理解，但第三个例子可能更有用，特别是当你需要对许多`item`居中对齐时。和往常一样，为你的特定布局选择最佳方法。

现在考虑一些更复杂的事情。假设你想在iPhone上并排显示两个视图，你要确保它们所有的边距都很好，而且它们宽度是一样的，它们也应该在设备旋转时正确地调整大小。

下面的插图显示了纵向和横向的视图:

<div align="center">    
<img src="./imgs/Blocks_Portrait_2x.png" width="30%" height="30%">
</div>

<div align="center">    
<img src="./imgs/Blocks_Landscape_2x.png" width="30%" height="30%">
</div>


那么这些约束应该是什么样的呢?下面的插图展示了一个简单的解决方案:

<div align="center">    
<img src="./imgs/two_view_example_1_2x.png" width="30%" height="30%">
</div>


上述解决方案使用了以下约束条件:

```
// Vertical Constraints
Red.top = 1.0 * Superview.top + 20.0
Superview.bottom = 1.0 * Red.bottom + 20.0
Blue.top = 1.0 * Superview.top + 20.0
Superview.bottom = 1.0 * Blue.bottom + 20.0
 
// Horizontal Constraints
Red.leading = 1.0 * Superview.leading + 20.0
Blue.leading = 1.0 * Red.trailing + 8.0
Superview.trailing = 1.0 * Blue.trailing + 20.0
Red.width = 1.0 * Blue.width + 0.0
```

按照前面的经验法则，这个布局有两个视图、四个水平约束和四个垂直约束。虽然这并不是一个绝对可靠的指南，但它能迅速表明你走在正确的道路上(While this isn’t an infallible guide, it is a quick indication that you’re on the right track. )。更重要的是，约束唯一地指定两个视图的大小和位置，从而产生一个明确的、可满足的布局。删除任何这些约束，布局就会变得不明确。添加额外的约束，就会有引入冲突的风险。

不过，这并不是唯一可能的解决方案。这里有一个同样有效的方法:

<div align="center">    
<img src="./imgs/two_view_example_2_2x.png" width="30%" height="30%">
</div>


你不需要将蓝框的顶部和底部固定到它的父视图中，而是让蓝框的顶部与红框的顶部对齐。同样，将蓝框的底部与红框的底部对齐。约束条件如下所示：

```
// Vertical Constraints
Red.top = 1.0 * Superview.top + 20.0
Superview.bottom = 1.0 * Red.bottom + 20.0
Red.top = 1.0 * Blue.top + 0.0
Red.bottom = 1.0 * Blue.bottom + 0.0
 
//Horizontal Constraints
Red.leading = 1.0 * Superview.leading + 20.0
Blue.leading = 1.0 * Red.trailing + 8.0
Superview.trailing = 1.0 * Blue.trailing + 20.0
Red.width = 1.0 * Blue.width + 0.0
```

该示例仍然有两个视图、四个水平约束和四个垂直约束。它仍然产生一个明确的、可满足的布局。

> 但是哪一种方案更好呢？
>
> 这些解决方案都可以产生有效的布局。那么哪一个更好呢?
>
> 不幸的是，客观地证明一种方法严格地优于另一种方法几乎是不可能的。每一种都有自己的优缺点。
>
> 第一个解决方案在删除视图时更加健壮。从视图层次结构中删除一个视图也会删除引用该视图的所有约束。因此，如果您删除红色视图，蓝色视图将保留三个约束来保持它的位置。你只需要添加一个约束，就可以得到一个有效的布局。在第二个解决方案中，删除红色视图将只给蓝色视图留下一个约束。
>
> 另一方面，在第一个解决方案中，如果你希望视图的顶部和底部对齐，你必须确保它们的顶部和底部约束使用相同的常量值。如果你改变了一个常数，你必须记住也要改变另一个常数。

