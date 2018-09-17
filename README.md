# README

> 总结:
> 
> 1. `initialize` 方法会在`类`第一次接受到消息的时候调用
> 2. 强制性的要`先调用父类的 initialize 方法,再调用子类的 initialize 方法`.
> 3. 因为`initialize`方法是通过`objc_msgSend`调用的,那么当子类没有实现`initialize`方法的时候,就会去调用父类的`initialize`方法,所以父类的 `initialize` 方法可能会被调用多次(看子类个数).
> 4. 如果分类实现了`initialize`方法,那么会优先调用.


## 一.`+(void)initialize`方法什么时候被调用?

**结论**

- `initialize`方法会在`类`第一次接受到消息的时候调用.

**证明**

假设只有一个类`TYPerson`,及其两个分类`TYPerson+Test1`和`TYPerson+Test2`的情况.里面都实现了`+ (void)initialize`方法.

```objc
+ (void)initialize {
    NSLog(@"%s",__func__);
}
```

1. 当`TYPerson`没有调用任何方法时.`也就是 TYPerson 没有接收到任何消息的时候`,运行程序,发现`initialize`方法并没有像`load`方法那样执行.所以验证了一个问题:

- `initialize`方法并不同`load`方法一样,`load`方法是当运行时加载类及分类时就会被调用,而`initialize`方法这时不会被调用.

2.当给`TYPerson`发送一条消息的时候,如`[TYPerson alloc]`,也就说明`TYPerson`接收到了一条消息的时候.这时候发现`initialize`方法执行了.而打印结果为:

```c
+[TYPerson(Test1) initialize]
```

3.当`TYPerson`第一次收到消息时,我们发现先调用的是其分类的`initialize`方法.由此可以推断出如下结论:

- 当`[TYPerson alloc]`执行的时候,其本质就是`objc_msgSend([TYPerson class],@selector(alloc))`.
- 那么首先就会通过`TYPerson`类的 `isa指针`去其`元类`对象的`类方法列表`中找`alloc`方法.
- 因为有分类的存在,并且分类中也实现了`initialize`方法,因为分类的方法在运行时会被合并到类的方法列表中,并且后编译的分类的方法会被优先调用.
- 所以,当找到`alloc`方法过程中,发现`TYPerson+Test1`中实现了`initialize`方法,所以将其调用.

4.当给`TYPerson`发送多条消息的时候,如下

```objc
[TYPerson alloc];
[TYPerson alloc];
[TYPerson alloc];
```

- 发现`initialize`方法确实只执行一次,且是当`TYPerson 第一次接收到消息的时候执行的`

## 二.当出现继承的时候,`initialize`方法的调用情况?

现在有一个类`TYStudent`,它继承自`TYPerson`.并且`TYStudent`有两个分类:`TYStudent+Test1`和`TYStudent+Test2`.且它们都实现了`+ (void)initialize`方法.

那么当给`TYStudent`发送消息的时候: `[TYStudent alloc]`,`initialize`方法的执行情况打印如下: 

```c
+[TYPerson(Test1) initialize]
+[TYStudent(Test2) initialize]
```

**结论**

- 当出现继承关系的时候,会优先调用父类的`initialize`方法,然后再调用子类的`initialize`方法.

**证明**

`[TYStudent alloc]` 时,调用了`+[TYStudent(Test2) initialize]`,可以由上面的推断得出.但是为什么会调用父类的`initialize`方法呢? 我们猜测在`objc_msgSend`这个方法中,它会首先主动的去调用父类的方法.下面开始验证我们的猜想:

#### 1.探究 initialize 方法的调用本质?

**结论**

- 在查找方法的过程中,会通过`objc_msgSend`来发送`initialize`方法.

**验证过程**

如执行 `[TYPerson alloc]`时,会调用其分类的`initialize`方法.因为其本质是`objc_msgSend([TYPerson class],@selector(alloc))`.所以很有可能是`objc_msgSend`方法内部`主动调用了 initialize方法`.

- 通过查看`objc`的 runtime 源码.找到`objc_msgSend`方法的实现,发现其是由`汇编`实现的,有点看不太懂....
- 那么看`objc_msgSend`的过程,发现是`通过 TYPerson 的 isa 指针,找到其元类对象,然后开始找 alloc 方法.如果找到,调用,如果没有找到, 通过 superclass 指针继续找其父类,找到后调用.那么 initialize 方法的调用时机应该是在找 alloc 方法时,或者找到 alloc 方法调用时, 被主动调用的.`
- 那么就看一下`objc_msgSend`寻找方法的这个过程中,是不是主动的调用了`initialize`方法.

在`objc_msgSend`的源码注释下,看到如下一行注释:

```objc
 * id objc_msgSend(id self, SEL _cmd, ...);
 * IMP objc_msgLookup(id self, SEL _cmd, ...);
```

注意下面的`objc_msgLookup`应该就是`查找方法`.发现其仍旧是汇编实现.经过前人的研究发现,其实还有一个由 c 语言实现的查找方法的方法: `class_getInstanceMethod`.

**class_getInstanceMethod: 找对象方法的实现**

因为上面是找的类方法.那么可以看到其本质还是调用`class_getInstanceMethod`

```c
Method class_getClassMethod(Class cls, SEL sel)
{
    if (!cls  ||  !sel) return nil;

    return class_getInstanceMethod(cls->getMeta(), sel);
}
```

那么看`class_getInstanceMethod`方法:

```c
Method class_getInstanceMethod(Class cls, SEL sel)
{
    if (!cls  ||  !sel) return nil;

    // This deliberately avoids +initialize because it historically did so.

    // This implementation is a bit weird because it's the only place that 
    // wants a Method instead of an IMP.

#warning fixme build and search caches
        
    // Search method lists, try method resolver, etc.
    // 搜索方法列表
    lookUpImpOrNil(cls, sel, nil, 
                   NO/*initialize*/, NO/*cache*/, YES/*resolver*/);

#warning fixme build and search caches

    return _class_getMethod(cls, sel);
}
```

其中的`lookUpImpOrNil`方法:

```c
IMP lookUpImpOrNil(Class cls, SEL sel, id inst, 
                   bool initialize, bool cache, bool resolver)
{
    IMP imp = lookUpImpOrForward(cls, sel, inst, initialize, cache, resolver);
    if (imp == _objc_msgForward_impcache) return nil;
    else return imp;
}
```

找到`lookUpImpOrForward`方法.

```c
IMP lookUpImpOrForward(Class cls, SEL sel, id inst, 
                       bool initialize, bool cache, bool resolver)
{
    IMP imp = nil;
    bool triedResolver = NO;

    runtimeLock.assertUnlocked();

    // Optimistic cache lookup
    // 试一下缓存查找,也许会找到,找到了就返回
    if (cache) {
        imp = cache_getImp(cls, sel);
        if (imp) return imp;
    }

    // runtimeLock is held during isRealized and isInitialized checking
    // to prevent races against concurrent realization.

    // runtimeLock is held during method search to make
    // method-lookup + cache-fill atomic with respect to method addition.
    // Otherwise, a category could be added but ignored indefinitely because
    // the cache was re-filled with the old value after the cache flush on
    // behalf of the category.
    
    /** 翻译:
     * runtimeLock 在检查 isRealized 和 isInitialized 期间,要保留下来.
     * 来阻止并发时的资源抢夺.
     *
     * 在搜索方法这个过程中要保持 runtimeLock.
     * method-loopup 和 cache-fill 关于方法添加时使用 atomic 原子操作.
     * 否则,添加一个 category 会被无限期的忽略
     * 因为代表 category 的缓存刷新后,缓存会被旧值重新填充
     */

    runtimeLock.read();

    if (!cls->isRealized()) {
        // Drop the read-lock and acquire the write-lock.
        // realizeClass() checks isRealized() again to prevent
        // a race while the lock is down.
        
        /**
         * 删除 read-lock 并且获取 write-lock
         * realizeClass() 再次检查 isRealized() 来防止 lock 的失败
         */
        
        runtimeLock.unlockRead();
        runtimeLock.write();

        realizeClass(cls);

        runtimeLock.unlockWrite();
        runtimeLock.read();
    }

    // initialize 我们要找的部分!
    
    // 如果需要初始化(initialize)并且这个类(cls)还没有被初始化
    if (initialize  &&  !cls->isInitialized()) {
        runtimeLock.unlockRead();
        // 那么就调用_class_initialize 方法将其初始化
        _class_initialize (_class_getNonMetaClass(cls, inst));
        runtimeLock.read();
        // If sel == initialize, _class_initialize will send +initialize and 
        // then the messenger will send +initialize again after this 
        // procedure finishes. Of course, if this is not being called 
        // from the messenger then it won't happen. 2778172
    }

    
 // ...省略 n 行非看代码

    return imp;
}
```

那么看`_class_initialize(`方法, 这个方法会根据需要将`+initialize`这条消息发送给未被初始化的类 class, 并且要强制性的先发送给父类 superclass(如果存在父类).

```c
/***********************************************************************
* class_initialize.  Send the '+initialize' message on demand to any
* uninitialized class. Force initialization of superclasses first.

* class_initialize. 根据需要将 `+initialize` 这条消息发送给未被初始化的类 class.
* 强制先初始化父类 superclasses
**********************************************************************/
void _class_initialize(Class cls)
{
    // 省略 n 行内部实现...

    // Make sure super is done initializing BEFORE beginning to initialize cls.
    // See note about deadlock above.
    supercls = cls->superclass;
    if (supercls  &&  !supercls->isInitialized()) {
        _class_initialize(supercls);
    }
    
     {
            callInitialize(cls);

            if (PrintInitializing) {
                _objc_inform("INITIALIZE: thread %p: finished +[%s initialize]",
                             pthread_self(), cls->nameForLogging());
            }
        }
    
    // 省略 n 行内部实现...
}
```

最后`callInitialize(cls)` 看其实现:**最后通过 objc_msgSend 将 initialize 方法发送出去**

```c
void callInitialize(Class cls)
{
    ((void(*)(Class, SEL))objc_msgSend)(cls, SEL_initialize);
    asm("");
}
```

### 2.继承时调用情况

上边已经知道了 initialize 的调用本质,那么假设有3个类:

```objc
// 1. 第一个类 及其分类
TYPerson.
TYPerson+Test1
TYPerson+Test2

// 2. 第二个类 TYStudent 继承自 TYPerson,及其分类
TYStudent : TYPerson
TYStudent+Test1
TYStudent+Test2

// 3. 第三个类 TYZhangSan 继承自 TYStudent, 及其分类
TYZhangSan : TYStudent
TYZhangSan+Test1
TYZhangSan+Test2
```

##### 2.1 第一种情况.子类实现了 initialize 的情况.

`TYStudent`  及 `TYZhangSan` 及它们的分类均实现了`initialize`方法时.

**执行方法 [TYZhangSan alloc], 见证 initialize 方法的调用情况**

```c
+[TYPerson(Test1) initialize]
+[TYStudent(Test2) initialize]
+[TYZhangSan(Test2) initialize]
```

##### 2.2 第二种情况, 父类`TYStudent及其分类都没有实现initialize方法`的情况.

只有`TYZhangSan 和 TYPerson 实现了 initialize 方法`.

**执行方法 [TYZhangSan alloc], 见证 initialize 方法的调用情况**

```c
+[TYPerson(Test1) initialize]
+[TYPerson(Test1) initialize]
+[TYZhangSan(Test2) initialize]
```

##### 2.3 第三种情况, 子类都没有实现 initialize 方法.`TYZhangSan 及 TYStudent 都没有实现 initialize 方法`的情况

**执行方法 [TYZhangSan alloc], 见证 initialize 方法的调用情况**

```c
+[TYPerson(Test1) initialize]
+[TYPerson(Test1) initialize]
+[TYPerson(Test1) initialize]
```

由上面的三种情况对比,我们得出**结论**: 

**结论1 : 
如果子类没有实现 `+(void)initialize` 方法的话, 那么就会调用父类的 `+(void)initialize` 的方法, 那么父类的 `+(void)initialize` 方法就有可能调用多次**.

**结论2 :如果分类实现了 `+(void)initialize` 方法,那么会优先调用分类的**

#### 3.验证上面的结论

`[TYZhangSan alloc]` 执行的时候,其实是`objc_msgSend([TYZhangSan class], @selector(alloc))`.

因为在调用`initialize`方法时要`先强制初始化父类的`.所以

- 先初始化`TYPerson` 的.调用一次
- 再初始化`TYStudent`的,发现其没有,那么就找到`TYPerson 的 initialize 方法`,调用一次
- 最后`TYZhangSan`的,自己没有,往上找 `TYStudent`的,也没有,再往上找`TYPerson`,有,调用.

