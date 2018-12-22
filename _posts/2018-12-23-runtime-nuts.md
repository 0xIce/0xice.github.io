---
title: 神经病院objc runtime入院考试
last_modified_at: 2018-12-24T00:20:02+0800
---

参加sunnyxx的[神经病院objc runtime入院考试](http://blog.sunnyxx.com/2014/11/06/runtime-nuts/)

## 1. 下面的代码输出什么？
 ```
 @implementation Son : Father
 - (id)init {
     self = [super init];
     if (self) {
         NSLog(@"%@", NSStringFromClass([self class]));
         NSLog(@"%@", NSStringFromClass([super class]));
     }
     return self;
 }
 @end
 ```
答：`[self class]`没什么疑问就不说了，研究一下`super`的调用，先看一下`objc4-750`中的解释:
> When it encounters a method call, the compiler generates a call to one of the functions objc_msgSend, objc_msgSend_stret, objc_msgSendSuper, or objc_msgSendSuper_stret. Messages sent to an object’s superclass (using the super keyword) are sent using objc_msgSendSuper; other messages are sent using objc_msgSend. Methods that have data structures as return values *  are sent using objc_msgSendSuper_stret and objc_msgSend_stret.

由上面可以看出用`super`调用实际是调用了
`objc_msgSendSuper(struct objc_super *super, SEL op, ...)`，
再看一下函数中的`super`参数的解释：
> A pointer to an objc_super data structure. Pass values identifying the context the message was sent to, including the instance of the class that is to receive the message and the superclass at which to start searching for the method implementation.

函数中的`super` 参数是一个`objc_super`的结构体类型
```
struct objc_super {
    __unsafe_unretained _Nonnull id receiver;
    __unsafe_unretained _Nonnull Class super_class;
};
```
结构体中有两个属性，一个是`receiver`用来存储当前的对象，一个是`super_class`用来存储当前对象的父类类型，所以`[super class]`的`receiver`还是当前对象，那么调用的结果和`[self class]`也是一样的了。

虽然结果一样，但是过程有点区别，`class`方法在`NSObject`类中定义。
那么`[self class]`的调用是在`Son`类的方法列表开始找，然后逐级到父类寻找直到`NSObject`,而`[super class]`是在`Father`类开始找，逐级找到`NSObject`。





