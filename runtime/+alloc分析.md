# +alloc调用分析
**调试分析基于objc4可编译源码**

objc代码

```objective-c
Person *p = [Person alloc];
```

给+alloc下断点，得到以下调用栈

```objective-c
frame #0: 0x00000001003371e0 libobjc.A.dylib`+[NSObject alloc](self=Person, _cmd="alloc") at NSObject.mm:2330:28
frame #1: 0x0000000100335445 libobjc.A.dylib`objc_alloc [inlined] callAlloc(cls=Person, checkNil=true, allocWithZone=false) at NSObject.mm:1714:12
frame #2: 0x000000010033539e libobjc.A.dylib`objc_alloc(cls=Person) at NSObject.mm:1730
frame #3: 0x0000000100003bdb ObjcDebugDemo`main(argc=<unavailable>, argv=<unavailable>) + 43 [opt]
frame #4: 0x00007fff71017cc9 libdyld.dylib`start + 1
```

即

```objective-c
|objc_alloc
	|callAlloc
		|+alloc
```

可以看到调用`alloc`方法时，runtime内部最先调用`objc_alloc`函数，这个函数内部很简单，可以直接跳到`callAlloc`,为了调试方便，我加入部分过滤代码来梳理执行路径

```objective-c
#if __OBJC2__
    if (slowpath(checkNil && !cls)) return nil;
    
    //**********Debug**********
    bool hasCustomAWZ = cls->ISA()->hasCustomAWZ();
    const char *name = cls->mangledName();
    if (!strcmp(name, "Person")) {
        printf("kk | %s hasCustomAWZ : %d\n",name, hasCustomAWZ);
    }
    //**********Debug**********
    
    //第一次判断hasCustomAWZ为YES，没有进入
    //第二次判断hasCustomAWZ为NO,进入
    if (fastpath(!cls->ISA()->hasCustomAWZ())) {
        return _objc_rootAllocWithZone(cls, nil);
    }
#endif

    // No shortcuts available.
    if (allocWithZone) {
        return ((id(*)(id, SEL, struct _NSZone *))objc_msgSend)(cls, @selector(allocWithZone:), nil);
    }
    return ((id(*)(id, SEL))objc_msgSend)(cls, @selector(alloc));

```

```
kk | Person hasCustomAWZ : 1
kk | Person hasCustomAWZ : 0
```

>这里暂时忽略hasCustomAWZ的含义，只需关注if判断结果

程序执行后,会发现该函数执行了两次，根据输出日志以及动态调试，得到以下更为详细的调用栈

```objective-c
|objc_alloc
	|callAlloc
		|+alloc
			|_objc_rootAlloc
				|callAlloc
```
不同的是，第二次进入`callAlloc`时，`cls->ISA()->hasCustomAWZ()`返回0，意味着进入了`_objc_rootAllocWithZone`函数，继续跟进，内部只是简单地继续调用函数。因此，继续扩展调用栈

```objective-c
|objc_alloc
	|callAlloc
		|+alloc
			|_objc_rootAlloc
				|callAlloc
					|_objc_rootAllocWithZone
						|_class_createInstancesFromZone
							|instanceSize		//计算内存大小
							|calloc			//开辟空间
							|initInstanceIsa	//初始化isa
								|initIsa

```

整个流程可划分为三个:


**1. 计算对象实例所占内存大小**

**2. 根据对象大小开辟堆内存**

**3. 初始化对象的第一个实例变量isa**

上述执行路径仅为生成普通对象的其中一条路径，还有其他分支没覆盖.



