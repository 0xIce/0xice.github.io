---
title: RxSwift Subjects
excerpt: RxSwift Subject 及 Repay 的介绍和使用
categories:
  - RxSwift
tags:
  - RxSwift
toc: true
author_profile: false
sidebar:
  nav: RxSwift-docs
---

## 什么是 Subject

Subject 同时表现为 observable 序列和 observer 订阅者，即它可以接收事件，在接收事件之后会将事件传递给它的订阅者

```swift
example(of: "PublishSubject") {
  let subject = PublishSubject<String>()
  subject.onNext("Is anyone listening?") // observer 的身份接收事件
  
  // observable 的身份被订阅
  let subscriptionOne = subject
  .subscribe(onNext: { string in
    print(string)
  })
  
  subject.on(.next("1"))
  subject.onNext("2")
}

// 订阅者只会接收到在开始订阅之后 subject 接收的事件，所以 "Is anyone listening?" 不会打印
输出：
--- Example of: PublishSubject ---
1
2
```

RxSwift 中有四种 subject 类型和两种 relay 类型，relay 类型是对 subject 类型的封装

- **PublishSubject:** 以空白开始，只会将新的元素发送给订阅者
- **BehaviorSubject:** 以一个初始值开始，对新的订阅者会重发初始值或者最后一个元素
- **ReplaySubject:** 以一个缓冲大小开始，对新的订阅会重发初始值或者最后缓冲大小的元素
- **AsyncSubject:** 只在收到 `.completed` 事件时发送最后一个 `.next` 事件，很少用到
- **PublishRelay** 和 **BehaviorRelay:** 它们各自封装了对应的 subject，但是只能接收 `.next` 事件，不能接收 `.completed` 或者 `.error` 事件，所以他们非常适合用在不会终止的序列上

## 使用 publish subject

在上面的例子中新增代码

```swift
let subscriptionTwo = subject
  .subscribe { event in
    print("2)", event.element ?? event)
}

subject.onNext("3")

输出：
3
2) 3
```

当一个订阅 dispose 之后将不会再接收新的事件

```swift
subscriptionOne.dispose()

subject.onNext("4")

输出：
2) 4
```

当一个 publish subject 接收了一个终止事件之后( `.completed` 或者 `.error` )，它将会**重发该终止事件到新的订阅者**并且不会再发出 `.next` 事件

新增代码

```swift
// 1
subject.onCompleted()

// 2
subject.onNext("5")

// 3
subscriptionTwo.dispose()

let disposeBag = DisposeBag()

// 4
subject
  .subscribe {
    print("3)", $0.element ?? $0)
  }
  .disposed(by: disposeBag)

subject.onNext("?")

输出：
2) completed
3) completed
```

> 最好在 publish subject 的订阅者中处理终止事件，以免在订阅的时候 publish subject 已经终止

## 使用 behavior subject