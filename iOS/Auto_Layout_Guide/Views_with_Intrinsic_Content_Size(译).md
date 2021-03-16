# View with Intrinsic Content Size

接下来的实例演示具有`intrinsic content size`的视图如何布局，通常地，`intrinsic content size`可以简化布局，减少你需要的约束的数量。但是，使用`intrinsic content size`通常需要设置视图的`content-hugging`和`compression-resistance`优先级（简称`CHCR`优先级），这可能会增加额外的复杂性。

## Simple Label and Text Field

以下例子演示布局一对简单的`label`和`text field`，在这个例子中，`label`的宽度基于它文字属性的尺寸，`text field`伸缩填充剩余的空间

<div align="center">    
<img src="./imgs/Label_and_Text_Field_Pair_2x.png" width="60%" height="60%">
</div>

因为本例使用视图的`intrinsic content size`，所以你只需要5个约束来布局，但是，你必须确保设置正确的`CHCR`优先级来让系统布局。

更多有关`intrinsic content sizes`和`CHCR`优先级，请参阅[Intrinsic Content Size](https://developer.apple.com/library/archive/documentation/UserExperience/Conceptual/AutolayoutPG/AnatomyofaConstraint.html#//apple_ref/doc/uid/TP40010853-CH9-SW21)

### Views and Constraints

在`Interface Builder`中，设置以下约束：

<div align="center">    
<img src="./imgs/simple_label_and_text_field_2x.png" width="60%" height="60%">
</div>

```
1. Name Label.Leading = Superview.LeadingMargin
2. Name Text Field.Trailing = Superview.TrailingMargin
3. Name Text Field.Leading = Name Label.Trailing + Standard
4. Name Text Field.Top = Top Layout Guide.Bottom + 20.0
5. Name label.Baseline = Name Text Field.Baseline
```

### Attributes

为了让`text field`拉伸占据剩余的空间，它的`content hugging `优先级必须低于`label`的`content hugging `优先级，默认地，`Interface Builder`会设置`label`的`content hugging `优先级为**251**，`text field`的为**250**，你可以在`Size inspector`中查看它们

| Name            | Horizontal hugging | Vertical hugging | Horizontal resistance | Vertical resistance |
| --------------- | ------------------ | ---------------- | --------------------- | ------------------- |
| Name Label      | 251                | 251              | 750                   | 750                 |
| Name Text Field | 250                | 250              | 750                   | 750                 |

### Discussion

注意，这个布局只使用两个约束(4和5)来定义垂直布局，使用三个约束(1、2和3)来定义水平布局。文章 [Creating Nonambiguous, Satisfiable Layouts](https://developer.apple.com/library/archive/documentation/UserExperience/Conceptual/AutolayoutPG/AnatomyofaConstraint.html#//apple_ref/doc/uid/TP40010853-CH9-SW16) 提及到每个视图在每个方向上需要两个约束确定布局，但是，`label`和`text field`的`intrinsic content size`提供了它们的高度和`label`的宽度，因此这3个约束没有必要了。

这个布局还有一个假设点，即`text field`总是比`label`高，并且它使用`text field`来定义到`top layout guide`的距离。因为`label`和`text field`都用于显示文本，所以本例使用文本的`baseline`对它们进行对齐。

水平方向上，你仍需确定哪个`view`需要延伸来填充剩余的空间，通过`CHCR`优先级来做这件事，在本例中，`Interface Builder`会设置`label`的`horizontal`和 ` vertical hugging priority`为 **251**，因为它比`text field`的**250**大，所以text field会拉伸来填充剩余的空间。（要理解还是挺费心思，`content hugging`优先级越大，那么这个视图的`intrinsic content size`会被优先确定，在这里的行为就是，`label`的宽度先被确认，这样一来`label`所占的空间就确定，所以剩下的空间全给`text field`（换句话说就是`text field`被拉伸了））

> 注意：如果布局需要显示在对于控件来说太小的空间中，你还需要修改compression resistances优先级的值，compression resistances定义了当没有足够空间时，哪些视图应该被截断。
>
> 在本例中，修改compression resistances留给读者练习，如果label的文字或字体足够大，屏幕没有足够空间放下，就会产生布局模糊的问题，然后，The system then selects a constraint to break，text field或label会被截断
>
> 理想情况下，你希望创建一个对于可用空间来说不会太大的布局——根据需要使用`size classes`来代替自动布局。设计支持多语言的应用时，很难准确预测label可能会变多大。以防万一，修改compression resistance是一个好的选择。

## Dynamic Height Label and Text Field

### Views and Constraints

### Attributes

### Discussion

## Fixed Height Columns

### Views and Constraints

### Attributes

### Discussion

## Dynamic Height Columns

### Views and Constraints

### Attributes

### Discussion

## Two Equal-Width Buttons

###  Views and Constraints

### Attributes

### Discussion

## Three Equal-Width Buttons

### Views and Constraints

### Attributes

### Discussion

## Two Buttons with Equal Spacing

### Views and Constraints

### Attributes

### Discussion

## Two Buttons with Size Class-Based Layouts

### Constraints

### Attributes

### Discussion



