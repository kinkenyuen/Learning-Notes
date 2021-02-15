# 两种方案对比
方案A :

```objc
+ (void)a_swizzleTargetCls:(Class)targetClass withClass:(Class)cls withOriginalSEL:(SEL)oriSEL withSwizzleSEL:(SEL)swizzleSEL {
    Method originalMethod = class_getInstanceMethod(targetClass, oriSEL);
    Method swizzledMethod = class_getInstanceMethod(cls, swizzleSEL);
    BOOL didAddMethod =
    class_addMethod(targetClass,
                    oriSEL,
                    method_getImplementation(swizzledMethod),
                    method_getTypeEncoding(swizzledMethod));

    if (didAddMethod) {
        class_replaceMethod(cls,
                            swizzleSEL,
                                           method_getImplementation(originalMethod),
                                           method_getTypeEncoding(originalMethod));
    } else {
        method_exchangeImplementations(originalMethod, swizzledMethod);
    }
}
```

方案B :

```objc
+ (void)b_swizzleTargetCls:(Class)targetClass withClass:(Class)cls withOriginalSEL:(SEL)oriSEL withSwizzleSEL:(SEL)swizzleSEL {
    Method originalMethod = class_getInstanceMethod(targetClass, oriSEL);
    Method swizzledMethod = class_getInstanceMethod(cls, swizzleSEL);
    method_exchangeImplementations(originalMethod, swizzledMethod);
}
```

# 研究前提

* 有3个类，分别为Child、Super、Base,继承关系为Child->Super->Base
* Base默认实现实例方法`instanceMethod`,子类根据情况可选实现该实例方法
* 实现的方法内容为，打印当前调用方法和调用父类的方法，即`[super ...]`
* 根据Hook Child和Super不同顺序、方法是否实现、使用方案A/B三个维度组合测试

# 先Hook Child 再Hook Super

## Child、Super均未实现方法

以下为原始结构图:

![1](./imgs/MethodSwizzling/未hook/Child、Super未实现方法/初始状态.jpg)

### Child用方案A_Super用方案A

![2](./imgs/MethodSwizzling/先Child后Super/Child、Super未实现方法/Child_A-Super_A.jpg)

**红色为Child类经过方案A Hook之后的指向图，蓝色为Super类经过方案A Hook之后的指向图，后同。**

调试执行顺序

```
=====Child实例调用方法=====
-[Child child_instanceMethod]
-[Base instanceMethod]
=====Super实例调用方法=====
-[Super super_instanceMethod]
-[Base instanceMethod]
=====Base实例调用方法=====
-[Base instanceMethod]
```

可以看到顺序与指向图一致

### Child用方案A_Super用方案B

![3](./imgs/MethodSwizzling/先Child后Super/Child、Super未实现方法/Child_A-Super_B.jpg)

```
=====Child实例调用方法=====
-[Child child_instanceMethod]
-[Base instanceMethod]
=====Super实例调用方法=====
-[Super super_instanceMethod]
-[Base instanceMethod]
=====Base实例调用方法=====
-[Super super_instanceMethod]
2021-02-15 16:05:26.994133+0800 MethodSwizzling[11456:925811] -[Base super_instanceMethod]: unrecognized selector sent to instance 0x6000003805c0
```

1. Child、Super实例执行方法路径正常
1. 当Base实例调用方法，因为Method的IMP指向了super_instanceMethod，所以出现
`unrecognized selector`错误

这种错误出现的场景:

1. 要Hook的当前类没有实现目标方法，而父类实现了
2. 在当前类的类别中直接使用`method_exchangeImplementations `操作
3. 在替换方法内部进行了自调用

当方法执行到替换方法内部自调用，就会出现**父类调用子类的SEL，从而出现错误**

# 先Hook Super 再Hook Child