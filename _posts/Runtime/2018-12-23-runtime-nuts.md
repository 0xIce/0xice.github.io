---
title: 神经病院objc runtime入院考试
categories:
  - Runtime
tags:
  - Runtime
toc: true
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
答：`[self class]`没什么疑问就不说了，研究一下`super`的调用，先看一下`objc4-750`中的解释:
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
结构体中有两个属性，一个是`receiver`用来存储当前的对象，一个是`super_class`用来存储当前对象的父类类型，所以`[super class]`的`receiver`还是当前对象，那么调用的结果和`[self class]`也是一样的了。

虽然结果一样，但是过程有点区别，`class`方法在`NSObject`类中定义。
那么`[self class]`的调用是在`Son`类的方法列表开始找，然后逐级到父类寻找直到`NSObject`,而`[super class]`是在`Father`类开始找，逐级找到`NSObject`。

## 2. 下面代码的结果？
```
BOOL res1 = [(id)[NSObject class] isKindOfClass:[NSObject class]];
BOOL res2 = [(id)[NSObject class] isMemberOfClass:[NSObject class]];
BOOL res3 = [(id)[Sark class] isKindOfClass:[Sark class]];
BOOL res4 = [(id)[Sark class] isMemberOfClass:[Sark class]];
```
答：先说答案，`YES/NO/NO/NO`，然后看一下原因，在OC中class也是对象，是metaClass类型的，instance-class-metaClass的关系可以通过一幅图来看一下：![](/assets/images/isa_superclass_metaclass.png)(图片来自[这里](http://www.sealiesoftware.com/blog/class%20diagram.pdf))对于本题来说`NSObject`就对应图中的`Root class`, `Sark`就对应图中的`Superclass`。
那么上边的问题返回true的条件就是方法的参数传入的是调用者class的metaClass类型，具体看一下。

1. `BOOL res1 = [(id)[NSObject class] isKindOfClass:[NSObject class]];`
`NSObject`的metaClass类型应该是`[NSObject metaClass]`(代指，实际获取metaClass需要用objc_getMetaClass)，它又是`NSObject`子类，所以参数传入`[NSObject class]`也是YES。
2. `BOOL res2 = [(id)[NSObject class] isMemberOfClass:[NSObject class]];`
跟第一条一样，但是传入了父类对象，所以返回NO。
3. 剩下两条传入的参数是`[Sark class]`, 完全不在`metaClass`之列，所以都返回NO。

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
答：都会调用到`-foo`, 对象方法的调用是在class的方法列表中查找(不考虑cache)，类方法的调用是在metaClass的方法列表中查找，找不到的话逐级到父类查找，那么跟上题类似，`NSObject`的metaClass的superclass就是`NSObject`, 所以最终还是会找到`NSObject`的方法列表中，成功调用。

## 4. 下面的代码会？Compile Error / Runtime Crash / NSLog…?
```
@interface Sark : NSObject
@property (nonatomic, copy) NSString *name;
@end
@implementation Sark
- (void)speak {
    NSLog(@"my name's %@", self.name);
}
@end
@implementation ViewController
- (void)viewDidLoad {
    [super viewDidLoad];
    id cls = [Sark class];
    void *obj = &cls;
    [(__bridge id)obj speak];
}
@end
```
答：
### (1). 会不会crash？
不会,
`id`类型在`objc`中的定义是: `typedef struct objc_object *id;`
那么
```
id cls = [Sark class];
void *obj = &cls;
```
可以理解为
```
objc_object *cls = [Sark class];
void *obj = &cls;
```
`[Sark class]`的返回`objc_class`类型，`objc_class`继承自`objc_object`, 所以用`objc_object`类型的`cls`接收也是完全可以满足`objc_object`的赋值要求的了。
再经过`c`类型和`oc`类型的转换，就获得了一个`Sark`类型的实例对象。可以正常调用实例方法。

### (2). 会输出什么？
会输出什么的关键问题是对象是怎么找到它的属性的，站在巨人的肩膀上，直接看一下[霜神的研究](https://www.jianshu.com/p/9d649ce6d0b8)结论吧
> Objc中的对象是一个指向ClassObject地址的变量，即 id obj = &ClassObject ， 而对象的实例变量 void *ivar = &obj + offset(N)

 所以只要找出入栈顺序，找出`obj`在栈中的相邻值就可以了。
 `viewDidLoad`调用了`super`, `objc_super`的结构是
```
struct objc_super {
    /// Specifies an instance of a class.
    __unsafe_unretained id receiver;

    /// Specifies the particular superclass of the instance to message. 
#if !defined(__cplusplus)  &&  !__OBJC2__
    /* For compatibility with old objc-runtime.h header */
    __unsafe_unretained Class class;
#else
    __unsafe_unretained Class super_class;
#endif
    /* super_class is the first class to search */
};
#endif
```
 得出入栈顺序是`self, _cmd(SEL, viewDidLoad), super_class, self(receiver), cls, obj`，输出的结果就是`cls`偏移一个地址(32位是4字节，64位是8字节)，就取到了`self`的值

### (3). 变种1
 ```
@interface Sark : NSObject
@property (nonatomic, copy) NSString *sex;
@property (nonatomic, copy) NSString *name;
@end
@implementation Sark
- (void)speak {
    NSLog(@"my name's %@", self.name);
}
@end
@implementation ViewController
- (void)viewDidLoad {
    [super viewDidLoad];
    id cls = [Sark class];
    void *obj = &cls;
    [(__bridge id)obj speak];
    NSLog(@"super: %@", [super class]);
}
@end
 ```
输出:
```
my name's ViewController
super: ViewController
```
按照入栈顺序，取到的值应该是`super_class`

### (4). 变种2
 ```
@interface Sark : NSObject
@property (nonatomic, copy) NSString *name;
@end
@implementation Sark
- (void)speak {
    NSLog(@"my name's %@", self.name);
}
@end
@implementation ViewController
- (void)viewDidLoad {
    [super viewDidLoad];
    NSString *myName = @"hotchner";
    id cls = [Sark class];
    void *obj = &cls;
    [(__bridge id)obj speak];
    [(__bridge id)obj speak];
}
@end
 ```
输出:
```
my name's hotchner
```
新的入栈顺序变成了`self, _cmd(SEL, viewDidLoad), super_class, self(receiver), myName, cls, obj`

## 结语
真的是神经病院！

## 参考链接
1. [Objective-C对象模型及应用](http://blog.devtang.com/2013/10/15/objective-c-object-model)
2. [神经病院Objective-C Runtime入院第一天——isa和Class](https://www.jianshu.com/p/9d649ce6d0b8)
3. [神经病院Objective-C Runtime住院第二天——消息发送与转发](https://www.jianshu.com/p/4d619b097e20)
4. [神经病院Objective-C Runtime出院第三天——如何正确使用Runtime](https://www.jianshu.com/p/db6dc23834e3)
5. [Dissecting objc_msgSend on ARM64](https://www.mikeash.com/pyblog/friday-qa-2017-06-30-dissecting-objc_msgsend-on-arm64.html#comment-b5af064e508623e696292572c8bfd75c)
6. [深入解析 ObjC 中方法的结构](https://github.com/draveness/analyze/blob/master/contents/objc/%E6%B7%B1%E5%85%A5%E8%A7%A3%E6%9E%90%20ObjC%20%E4%B8%AD%E6%96%B9%E6%B3%95%E7%9A%84%E7%BB%93%E6%9E%84.md#%E6%B7%B1%E5%85%A5%E8%A7%A3%E6%9E%90-objc-%E4%B8%AD%E6%96%B9%E6%B3%95%E7%9A%84%E7%BB%93%E6%9E%84)
7. [Objc 对象的今生今世](https://www.jianshu.com/p/f725d2828a2f)
8. [sunnyxx视频](https://v.youku.com/v_show/id_XODIzMzI3NjAw.html)
