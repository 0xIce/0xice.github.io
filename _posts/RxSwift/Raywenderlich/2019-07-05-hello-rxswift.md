---
title: Hello RxSwift!
categories:
  - RxSwift
tags:
  - RxSwift
toc: true
author_profile: false
sidebar:
    nav: RxSwift-docs
---

> RxSwift 是一个使用可观察序列和函数式操作符构成异步的和基于事件的代码，允许调度器带参数地调用

> RxSwift 本质上简化了异步程序的开发，它允许你以序列式的隔离的方式去响应新的数据

## 异步编程术语

### 1. 状态，尤其是共享的可变状态

状态不好定义，可以考虑下面一个例子

一个平板电脑，突然有一天运行不正常了，重启之后恢复正常，软硬件没变，但是状态改变了。

管理 app 中多个异步模块共享的状态，是一个挑战，也是我们需要学习的地方

### 2. 命令式编程

命令式编程是一种编程范式，使用声明来改变程序的状态，就像告诉你的狗：“趴下，装死”，你是用命令式的代码来告诉 app 什么时候以及如何工作

考虑下面的代码

```swift
override func viewDidAppear(_ animated: Bool) {
  super.viewDidAppear(animated)

  setupUI()
  connectUIControls()
  createDataSource()
  listenForChanges()
}
```

这里看不出这些方法干了什么，有没有改变 view controller 的属性。更不安的是，它们以正确的顺序调用了吗？可能有人不小心改动了顺序，改变了 app 的表现

### 3. Side effects

`Side effects` 代表在你当前的代码范围之外的对状态做的更改。

比如，考虑上面的代码的 `connectUIControls()` 可能对一些 UI 控件添加了事件响应，这个就是导致了 `side effects`，它改变了 view 的状态 -> app 再调用 `connectUIControls()` 方法之前和之后的表现不一样了

当你改变存储在硬盘上的值或者更新 label 上的文本的时候，就造成了 side effects

side effects 本身不是坏事。而且，造成 Side effects 是**任何程序的终极目标**。

重要的是，你需要让 Side effects 以一种可控的方式发生。你需要决定哪些代码可以产生 side effects，而那些代码之间简单的处理和输出数据

### 4. 声明式代码

在命令式的编程中，你有意识地改变状态。在函数式编程中，你的目标是最小化代码造成的 side effects，RxSwift 结合了两者的优点。

声明式的代码让你可以定义一些行为，RxSwift 在特定的事件发生的时候运行这些行为，并且提供了一个不可变的独立的数据片

在这种方式下，你可以用简单的 for loop 来实现异步任务：你处理的是不可变的数据，代码的运行方式是一种序列式的，确定的方式

### 5. 响应式系统

响应式的系统是一种更抽象的概念，可以应用于 web 和 iOS 平台的开发

展现出了下面的特性：

- **反应灵敏(Responsive)：**始终保持 UI 最新，代表最新的 app 状态
- **适应能力强(Resilient)：**每一个行为都是独立的，并且提供了灵活的错误恢复机制
- **弹性强(Elastic)：**代码会处理多种的工作量，经常实现一些特性，比如，懒加载的拉取数据来驱动的集合，事件的限流，资源的共享
- **消息驱动(Message driven)：**模块会使用基于消息的通信来提升重用性和解耦

## RxSwift 核心

### Observables

> 还是那句话，**`Observables` 就是序列**

`Observable<T>` 类型可以允许一个或多个订阅者去订阅、实时响应事件，收到事件后可以更新 UI 或者其它的数据处理

`ObservableType` 协议(`Obvervable<T>`遵守的协议)非常简单，仅可以发出 3 种类型的事件：

- **A next event:**  一个携带者最新(下一个)数据的事件，订阅者可以从这里拿到值。一个 `Observable` 在终止之前可能产出无限的这样的值
- **A completed event:** 成功结束的事件，之后不会再产出事件
- **An error event:** 错误地结束，之后不会再产出事件

得益于 `Observable<T>` 的范型，可以写出耦合比较低的代码

#### 有限序列

有些 observables 可以产出 0 个，1  个或多个值，在之后的某个时间，就会成功地结束或者错误地结束

在 iOS app 中，考虑下面这样一个网络下载文件的任务：

1. 开始下载，监听获取到的数据
2. 重复将获取到的一块数据添加到保存的文件上
3. 如果网络发生错误，下载任务会因为超时错误而终止
4. 如果文件可以正常下载，任务会成功结束

对应的代码会像下面这样：

```swift
API.download(file: "http://www...")
	.subscribe(onNext: { data in
		// Append data to temporary file
	},
	onError: {
    // Display error to user
  },
	onCompleted: {
    // Use downloaded file
  })
```

`API.download(fiel:)` 方法会返回一个 `Observable<Data>` 类型的实例，将会通过网络获取并产出 `Data` 发送给订阅者

#### 无限序列

通常来说，UI 事件就是无限序列的一种。

比如处理设备的屏幕方向的事件：

1. 向 `NotificationCenter` 中添加一个 `UIDeviceOrientationDidChange` 的监听
2. 你需要添加一个方法回调来处理屏幕方向事件

这个事件是不会结束的，并且在你监听这个事件的时候始终有一个初始值

RxSwift 处理这个事件的代码会像下面这样：

```swift
UIDevice.rx.orientation
	.subscribe(onNext: { current in
    switch current{
      case .landscape:
      	// Re-arrange UI for landscape
      case .portrait:
      	// Re-arrange UI for protrait
    }
  })
```

`UIDevice.rx.orientation` 将会返回一个 `Observable<Orientation>` 类型的实例，可以订阅它在接收到事件的时候做相应的处理。订阅的时候忽略了 `onError` 和 `onCompleted` 参数，因为这两个事件不会发生

### Operators

`ObservableType` 和 `Observable` 的实现中包含了大量的方法来处理不同的异步任务片段，你可以组合这些方法来处理更加复杂的逻辑。因为它们是高度解耦的，它们通常被称为`操作符(opetators)`

这些操作符大多数接收异步地输入，产出不会造成 side effects 的输出。它们可以容易地组合在一起

上面的观察屏幕方向的例子添加操作符：

```swift
UIDevice.rx.orientation
	.filter { value in
    return value != .landscape
  }
	.map { _ in
    return "Portrait is the best!"
  }
	.subscribe(onNext: { string in
    showAlert(text: string)
  })
```

<img width="50%" height="50%" src="/assets/images/RxSwift_operators.jpeg"/>

操作符是高度**可组合的**，它们都是接收输入然后输出它们自己的结果，可以将它们以不同的方式组合实现不同的逻辑

### Schedulers

`Scheduler` 是 Rx 中等价于 dispath queue 的概念

RxSwift 中预定义了很多的 schedulers，基本上可以覆盖 99% 的使用场景，所以基本不用创建自定义的 scheduler

举个例子，你可以指定在 `SerialDispatchQueueScheduler` 上监听 `next` 事件，这将会使用 Grand Central Dispatch 在指定的队列上运行你的代码

`ConcurrentDispatchQueueScheduler` 将会并发地运行你的代码，同时 `OperationQueueScheduler` 允许你在指定的 OperationQueue 上调度订阅

归功于 RxSwift， 你可以将同一个订阅的不同部分运行在不同的 scheduler 上来达到最优的表现

RxSwift 可以作为一个派发员的身份将订阅(左边)和 scheduler(右边)连接起来，将一部分任务派发给正确的上下文，并且可以让不同的部分无缝地和其它部分的输出一起工作

<img width="100%" height="100%" src="/assets/images/RxSwift_schedulers.jpeg"/>

- 蓝色的网络从 (1) 开始一部分任务，使用自定义的基于 OperationQueue 的 scheduler
- (1) 的输出作为 (2) 的输入，(2) 运行在不同的 scheduler，在一个并发的后台 GCD 队列上
- 最后，任务 (3) 派发到主线程 scheduler 上来使用新数据更新 UI

> 后面章节会有对 scheduler 更详细的解释

## 应用架构

RxSwift 并不会改变 app 的架构，RxSwift 可以用在 MVC、MVP 或 MVVM 上，还可以用在单向数据流的架构上

当然，微软的 MVVM 架构适用于事件驱动的应用开发，并提供了数据绑定。RxSwift 和 MVVM 的组合可以表现地更好一点，因为 ViewModel 允许你暴露 Observable<T> 属性，可以把它绑定到 UIKit 上，这使UI 和数据的绑定变得很简单

<img width="100%" height="100%" src="/assets/images/RxSwift_MVVM_binding.jpeg"/>

> Rx 从微软的 .NET 孵化，MVVM 也是出自微软，他俩的结合更合理也可以理解

## RxCocoa

RxSwift 对 Rx 的实现是普通的、平台无关的，所以它并不知道 Cocoa 和 UIKit 中的类

RxCocoa 是和 RxSwift 相伴的一个库，用来实现 UIKit 和 Cocoa 中的 Rx。RxCocoa 对很多 UI 控件添加了 Rx 的扩展，所以你可以用 Rx 的方式去使用 UI 控件。

比如用 Rx 的方式使用 UISwitch 的例子：

```swift
toggleSwitch.rx.isOn
	.subscribe(onNext: { isOn in
    print(isOn ? "It's ON" : "It's OFF")
  })
```

同样的，RxSwift 向 UITextField，URLSession，UIViewController 等类中添加了 rx 的命名空间，允许你实现类似于对 UISwitch 的 rx 调用，并且你可以在这个 rx 命名空间下定义自己的响应式扩展