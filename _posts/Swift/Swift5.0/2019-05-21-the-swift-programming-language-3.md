---
title: 重读 Swift (5.0) 第三篇
excerpt: Inheritance、Initialization、Deinitialization
categories:
  - Swift
tags:
  - Swift
toc: true
---

## Inheritance

### 1. Overriding

1. 子类重写的方法，`someMethod()`，可以在子类重写中通过 `super.someMethod()` 调用
2. 子类重写的 `property`，`someProperty`，可以在子类中 `setter/getter` 方法中通过 `super.someProperty` 调用
3. 子类重写的 `subscript`，`someIndex`，可以子类重写的subscript中通过 `super[someIndex]` 调用

### 2. Overriding Properties

#### 1. Overriding Property Getters and Setters

1. 子类可以通过自定义 `getter/setter` 的方式对属性进行重写
2. 可以将 `read-only` 的属性重写为 `read-write` 的属性，反过来则不可以
3. `let` 属性不可以被 `override`

#### 2. Overriding Property Observers

1. `let` 存储属性和` read-only` 计算属性不可以被子类重写 `observer`
2. 重写 `setter` 和 `willSet ` 的 observer 不可以同时存在，可以在重写的 `setter` 中实现 `observer`

#### 3. Preventing Overrides

- 可以通过添加 `final` 关键字禁止重写，如

   `final var, final func, final class func, final subscript`

- `extension`中也可以对添加的 `method，property，subscript`使用 `final` 修饰

- 也可以对整个 `class` 使用 `final` 修饰

## Initialization

### 1. _init_ vs _default value_

Default value is preferred

### 2. Assigning Constant Properties During Initialization

`constant` 属性只能在声明的类的初始化方法中修改，不能在子类初始化方法中修改

### 3. Default Initializers

如果 `structure` 或者 `class` 所有的存储属性都有了默认值，并且没有自定义初始化方法，Swift会提供默认的初始化方法，默认初始化方法没有参数，按照属性的默认值进行初始化

### 4. Initializer Delegation for Value Types

1. 对于值类型，一个初始化方法内部可以通过 `self.init` 调用另外一个初始化方法

2. 自定义初始化方法后，默认的初始化方法（或者 `structure` 的按成员的初始化方法）方法将不可用
3. 可以在 `extension` 中添加自定义初始化方法同时保留默认初始化方法和 `structure` 的按成员初始化方法

### 5. Class Inheritance and Initialization

1. Designated Initializers
   1. 需要将所有的本类声明的属性做初始化，并且调用合适的父类初始化方法
   2. 像漏斗的尖口一样，保持尽量少，保证所有本类定义的属性都被初始化，通往父类初始化方法的入口
   3. 每个类至少有一个，可以从父类中继承
2. Convenience Initializers
   1. 最终需要调用本类的 `Designated Initializers`
   2. 可以根据传入的参数做一些定制的初始化
   3. 可选，数量时0个或多个

### 6. Initializer Delegation for Class Types

#### 1. Rules

1. `Designated Initializer` 必须调用直接父类的  `Designated Initializer`

2. `Convenience Initializer` 必须调用本类的另外一个初始化方法

3.  `Convenience Initializer` 必须最终调用到一个 `Designated Initializer`

   >  Designated initializers must always delegate up.
   >
   > Convenience initializers must always delegate across.

#### 2. Two-Phase Initialization

##### 1. 定义

1. `Phase-1` 每个本类声明的存储类型属性都必须被赋初值
2. `Phase-2` 在每个对象可用之前，类可以对存储属性做进一步的改造

##### 2. 作用

1. 防止属性在被初始化之前被访问
2. 防止属性意外地被其它初始化方法赋了不同的值？

##### 3. Compiler Safety Check

1. `Designated Initializer` 必须保证在调用父类初始化方法之前所有本类声明的存储属性都被初始化了
2. `Designated Initializer` 在给继承的属性赋值之前，必须调用了父类的初始化方法，否则赋的值会被父类的初始化方法重写
3. `Convenience Initializer` 在给属性(本类定义的属性和继承的属性)赋值之前，必须调用了本类其它的初始化方法，否则赋的值会被本类的 `Designated Initializer` 覆盖
4. 在 `Phase-1` 之前，初始化方法不能调用任何实例方法，读取实例属性的值，不能将 `self` 作为值被引用(比如将 `self` 当作参数初始化另一个类型)

##### 4. 过程

1. `Phase-1`

   1. `designated or convenience initializer` 被调用

   2. 分配内存，内存未被初始化

   3. `Designated Initializer` 保证本类声明的存储属性被赋值。这些存储属性的内存被初始化

   4. `Designated Initializer` 提交父类的初始化方法进行同样的操作

   5. 在继承链上持续调用父类的初始化方法直到最终父类

   6. 最终父类调用了初始化方法对它的存储属性进行了赋值。这时候这个新对象的内存被完全地初始化了，`phase-1` 完成

2. `Phase-2`

   1. `Designated Initializer` 这时候就可以对本类对象做一些自定义的赋值和调用，比如改变属性的值，调用实例方法等

   2. `Convenience Initializer` 这时候可以做一些自定义的赋值和调用

### 7. Initializer Inheritance and Overriding

1. swift 的子类不会自动继承父类的初始化方法，以防止子类初始化时调用了父类初始化方法导致对象没有被正确的初始化（满足特定条件的时候可以自动继承）
2. 子类可以重写父类的 `Designated Initializer`，在方法名前加 `override` 关键字，可重写的初始化方法包括默认初始化方法
3. 当子类定义的初始化方法和父类的 `Convenience Initializer` 相同的时候，根据 `Rule 1`, 子类不可以不可以直接调用父类的 `Convenience Initializer`, 所以严格来说并没有重写，所以不需要添加 `override` 关键字
4. 如果子类定义的初始化方法没有在 `Phase 2` 对对象进行修改，并且父类有不需要参数的指定初始化方法，那么子类的初始化方法中可以省略调用父类的初始化方法，实际是对 `super.init()` 的隐式调用

### 8. Automic Initializer Inheritance

#### 前提

自动继承的前提是所有子类子类定义的存储属性都被赋了默认值

#### 继承规则

1. 如果子类没有定义任何指定初始化方法，那么子类会自动继承父类的所有指定初始化方法

2. 如果子类实现了父类的所有指定初始化方法（从规则1继承或者都重写了），那么子类会自动继承父类的所有便利构造方法

   > 规则2中，可以将父类的指定构造方法重写为子类的便利构造方法

### 9. Failable Initializers

1. 定义可失败的初始化方法的方式是在 `init` 后面加一个问号，如 `init?()`

   > 不可以同时定义相同参数的一个可失败的和一个不可失败的方法

2. 当发生初始化失败时，`return nil` 来直接返回

3. 初始化失败的规则可以自定义，比如一些参数不符合要求，或者其它情况发生了失败

4. 可失败的初始化方法返回的是一个 `optional` 对象类型

### 10. Propagation for Initialization Failure

1. 一个可失败的初始化方法可以调用本类的另一个可失败的初始化方法，一个子类的可失败的初始化方法可以调用父类可失败的初始化方法
2. 一个可失败的初始化方法可以调用一个不可失败的初始化方法，可以通过这种形式对不可失败的初始化过程添加可失败的状态和表现

### 11. Overriding a Failable Initializer

1. 可以将父类可失败的初始化方法在子类重写为不可失败的初始化方法，反之不可以
2. 父类可失败的初始化方法重写为不可失败的初始化方法时， **只能用强制解包**的方式调用父类可失败的初始化方法，也可以调用父类不可失败的指定初始化方法

### 12. The init! Failable Initializer

1. init? 可以调用 init!，反之也可以
2. init? 可以重写为 init!，反之也可以
3. init 可以调用 init!，反之也可以

### 13. Required Initializers

1. 被required标记的初始化方法，子类必须重写
2. 子类重写时，前面也要写 `required` 关键字，不写 `override` 关键字
3. 如果子类满足继承初始化方法的条件，可以不用显式重写

## Deinitialization

1. `deinitializer` 在对象` deallocate` 之前被立马调用
2. 父类的 `deinit` 在子类的 `deinit` 结束之后自动调用
3. 对象在 `deinit` 调用结束之前还没有被释放，所以 `deinitializer` 可以访问所有的属性，比如通过保存的文件名关闭打开的文件

## Quiz

<details>
  <summary>Q1: 为什么本类的初始化要放在 super 调用之前？反过来会有什么问题？</summary>
  <p>
    因为子类有可能重写了父类的方法，而父类的初始化方法中又可能调用了重写的方法，如果子类的方法重写中读取了子类新增的属性，父类在子类初始化之前调用就会出错。
  </p>
  <p>
    <b>所以需要满足上述的 <code>Safety Check 1</code></b>
  </p>
</details>

## 参考文档

1. [Medium](https://medium.com/@dineshk1389/swift-why-super-init-is-called-after-setting-all-self-properties-d6827e9f4eb2)