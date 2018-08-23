# README

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
- 因为有分类的存在,并且分类中也实现了`initialize`方法,因为分类的方法在运行时会别合并到类的方法列表中,并且后编译的分类的方法会被优先调用.
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

