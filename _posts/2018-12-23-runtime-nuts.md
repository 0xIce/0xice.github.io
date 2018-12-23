---
title: 神经病院objc runtime入院考试
categories:
  - iOS
tags:
  - objc runtime
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

## 2. 下面代码的结果？
```
BOOL res1 = [(id)[NSObject class] isKindOfClass:[NSObject class]];
BOOL res2 = [(id)[NSObject class] isMemberOfClass:[NSObject class]];
BOOL res3 = [(id)[Sark class] isKindOfClass:[Sark class]];
BOOL res4 = [(id)[Sark class] isMemberOfClass:[Sark class]];
```
答：先说答案，`YES/NO/NO/NO`，然后看一下原因，在OC中class也是对象，是metaClass类型的，instance-class-metaClass的关系可以通过一幅图来看一下：![](/assets/images/isa_superclass_metaclass.png)对于本题来说`NSObject`就对应图中的`Root class`, `Sark`就对应图中的`Superclass`。
那么上边的问题返回true的条件就是方法的参数传入的是调用者class的metaClass类型，具体看一下。

1. `BOOL res1 = [(id)[NSObject class] isKindOfClass:[NSObject class]];`
`NSObject`的metaClass类型应该是`[NSObject metaClass]`(代指，实际获取metaClass需要用objc_getMetaClass)，它又是`NSObject`子类，所以参数传入`[NSObject class]`也是YES。
2. `BOOL res2 = [(id)[NSObject class] isMemberOfClass:[NSObject class]];`
跟第一条一样，但是传入了父类对象，所以返回NO。
3. 剩下两条传入的参数是`[Sark class]`, 完全不在`metaClass`之列，所以都返回NO。

## 3. 下面的代码会？Compile Error / Runtime Crash / NSLog…?
```
@interface NSObject (Sark)
+ (void)foo;
@end
@implementation NSObject (Sark)
- (void)foo {
    NSLog(@"IMP: -[NSObject (Sark) foo]");
}
@end
// 测试代码
[NSObject foo];
[[NSObject new] foo];
```
答：都会调用到`-foo`, 对象方法的调用是在class的方法列表中查找(不考虑cache)，类方法的调用是在metaClass的方法列表中查找，找不到的话逐级到父类查找，那么跟上题类似，`NSObject`的metaClass的superclass就是`NSObject`, 所以最终还是会找到`NSObject`的方法列表中，成功调用。

