---
title: Operation 和 OperationQueue 入门
excerpt: 基本概念、对比GCD、暂停/恢复、任务依赖
tags:
  - Operation
toc: true
---

## Concepts

> 在 iOS 和 macOS 中，thread 功能是由 POSIX Threads API(pthreads) 提供的，并且是操作系统的一部分。pthreads 比较底层，容易出错且不易发现。

## Operation vs. GCD

GCD 结合了语言特性、运行时库、系统增强，提供了系统级的多线程实现，充分利用 iOS 和 macOS 设备的多核。

Operation 是构建于 GCD 之上的，更高级别的封装。

**How to Choose**

1. GCD 是一个轻量级的实现多线程的方式。不用手动安排任务，系统会帮助我们安排。不易于处理任务间的依赖。对于取消和暂停任务的执行需要做一些额外的工作
2. Operation 在 GCD 之上增加了更高级的特性，可以方便的添加任务的依赖，还可以重用、取消、暂停任务

**How to Use**

1. `Operation` 是一个抽象类，每个继承的子类表示一个具体的任务

2. 重写 `main` 方法来进行具体的任务

3. 在 `main` 方法任务开始前和结束后查看 `isCancelled` 检查任务是否被取消来决定是否需要进行后续的操作

4. `main` 方法在网络任务结束后可以对数据进行进一步的处理

5. 开始任务时，创建一个 `Operation` 子类的实例，添加到对应的队列中(异步执行)或者调用 `Operation` 的 `start` 方法(同步执行)

   > 在耗时操作开始前检查任务是否被取消是一个好的习惯

## Suspend and Cancel Operation

可以设置 `OperationQueue` 的 `isSuspended` 属性为 `true` 或 `false` 来暂停/开始任务，不可以对单独的任务设置

调用 `Operation` 的 `cancel` 方法取消任务的执行

## Dependency

可以调用 `Operation` 的 `addDependency(op:)` 和 `removeDependency(op:)` 方法来添加和移除依赖

> 一个被依赖的任务被取消的时候，依赖的任务依旧会执行，就像被依赖的任务正常完成一样

## Reference

1. 不完整翻译自 [raywenderlich](https://www.raywenderlich.com/5293-operation-and-operationqueue-tutorial-in-swift)