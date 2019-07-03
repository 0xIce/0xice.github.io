---
title: RxSwift 冷热信号
categories:
  - RxSwift
tags:
  - RxSwift
author_profile: false
sidebar:
  nav: RxSwift-docs
---

介绍 RxSwift 中的冷热信号

我建议将冷热信号看成是序列的属性，而不是单独的类型，因为它们和  `Observable`  序列代表了同样的抽象

下面是一个 [ReactiveX.io](http://reactivex.io/documentation/observable.html) 中的定义

> `Observable` 什么时候会发出序列元素呢？取决于这个 `Observable`，一个 “hot” Observable 可能会在元素创建的时候就发出，所以当观察者订阅这个 `Observable` 的时候可能会在整个序列的中部开始观察。一个 “cold” Observable 不同，会在有观察者订阅之后才开始发出元素，所以可以保证订阅者从序列的开始获取整个序列

| Hot Observables                                             | Cold observables                               |
| ----------------------------------------------------------- | ---------------------------------------------- |
| ... 是序列                                                  | ... 是序列                                     |
| 无论是否有观察者添加了订阅都会使用资源（“产生热量”）        | 在有观察者订阅之前不会使用资源（“不产生热量”） |
| 变量/属性/常量，点击坐标，鼠标坐标，UI control 值，当前时间 | 异步操作，HTTP 连接，TCP 连接，流              |
| 通常包含 N 个元素                                           | 通常包含 1 个元素                              |
| 无论是否有订阅者都会产生序列元素                            | 只有在有订阅者的时候才产生序列元素             |
| 序列的计算资源通常是是所有的订阅者共享                      | 序列计算资源通常是每个订阅者单独分配           |
| 通常是有状态的                                              | 通常是无状态的                                 |