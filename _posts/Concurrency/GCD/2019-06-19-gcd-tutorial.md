---
title: GCD 入门
excerpt: 并行并发、同步异步、队列、多读单写、组
tags:
  - GCD
toc: true
---

## Concurrency

iOS 中一个应用进程可以有多个线程。操作系统管理每个线程，每个线程都**可以**并发地执行，但是操作系统决定是否会同时执行，什么时候，以及如何实现。

单核处理器设备通过 **time-slicing(时间片)** 实现，执行一个线程，执行上下文切换，再执行另一个线程。

多核处理器设备，通过 **parallelism(并行)** 同时执行多个线程。

<img width="50%" height="50%" src="/assets/images/Concurrency_vs_Parallelism.png"/>

GCD 建立在线程之上，它维护着一个线程池。在 GCD 中通过代码块添加任务到 **dispatch queues** 然后 GCD 决定用哪个线程去执行它们。

GCD 根据系统和可用的系统资源来决定有多少个并行。

> 并行需要并发，但是并发并不保证并行

**并发并行的区别**

>concurrency is about *structure* while parallelism is about *execution*

## Queues

GCD 以 FIFO 的顺序执行添加进队列的任务，保证先添加进队列的任务比后添加进队列的任务先**开始**执行。

Dispatch queues 是线程安全的，可以多线程同时存取队列。

队列分为**串行**队列和**并发**队列。

串行队列保证在某一时刻只有一个任务在执行，GCD 控制执行时间，你不能确定在任务之间的间隔时间。

<img width="50%" height="50%" src="/assets/images/Serial-Queue-Swift.png"/>

并发队列可以让多个任务在同一时间执行。任务开始执行的顺序还是遵守 FIFO 的规则，任务结束的顺序是不确定的，开始两个任务的时间间隔也是不确定的，同时执行的任务数量也是不确定的。

<img width="50%" height="50%" src="/assets/images/Concurrent-Queue-Swift.png"/>

当两个任务的执行时间重叠时，GCD 会决定是否在多核上执行或者通过时间片的方式执行。

GCD 提供三种主要类型的队列：

1. **Main queue**：在主线程执行并且是一个串行队列
2. **Global queues**: 整个系统共享的并发队列，共有四个不同优先级的队列：`high, default, low, background` 。background 优先级的 queue 在 I/O 活动中被限制以减少对系统的负面影响。
3. **Custom queues**: 可以是串行或者并发队列，*这些队列中的请求最终会在一个全局队列中*

全局队列的优先级属性在 iOS8 上被废弃，替代的方式是使用 QoS。

1. **User-interactive**: 任务必须马上完成以提供一个好的用户体验。
2. **User-initiated**: 用户从UI操作开始这些异步操作。用于用户等待立即的结果和被用户交互依赖的任务。对应 `high` 优先级全局队列。
3. **Default**: 默认，QoS 参数省略时的默认值，对应 `default` 优先级全局队列。
4. **Utility**: 长时间运行的任务，典型的应用是用户可见的进度指示器。可用于计算、I/O、网络、持续数据流等。对应 `low` 优先级全局队列。
5. **Background**: 用户没有意识到的任务。可用于预加载、维护、其它的无用户交互的/非是时间敏感的任务。对应 `background` 优先级的全局队列。

## Synchronous vs. Asynchronous

同步方法在任务完成之后将控制返回给调用者。

异步方法立刻返回，保持任务开始的顺序，但是不会等待任务的结束。因此，异步方法不会阻塞当前的线程执行下一个方法。

**什么时候用 async**

1. **Main Queue**: 子线程结束任务后更新UI可以通过主队列和 async 保证更新UI的操作会在当前方法执行完成之后的某个时刻执行
2. **Global Queue**: 普通非UI操作
3. **Custom Serial Queue**: 连续的后台任务。串行队列一个时刻只有一个任务在执行，解决了资源竞争问题。

**什么时候用 sync**

1. **Main Queue**: 注意死锁的情况
2. **Global Queue**: 可用在 dispatch barrier 中同步任务，或者等待一个任务完成之后再进行其它的操作
3. **Custom Serial Queue**: 注意死锁

## Managing Tasks

GCD 用闭包的方式添加任务，每个提交给 `DispatchQueue` 的任务都是一个 `DispatchWorkItem` 。

可以设置  `DispatchWorkItem` 的 `QoS` 或者是否产生新的线程等。

## Delaying Task Execution

用于希望任务在特定时间运行的时候。

> 可以用在给新启动app的用户一些提示，如果提示太早可能用户关注点在其它地方而忽略了这个提示，所以可以用延时任务完成

```swift
// 1
let delayInSeconds = 2.0

// 2
DispatchQueue.main.asyncAfter(deadline: .now() + delayInSeconds) { [weak self] in
  guard let self = self else {
    return
  }

  if PhotoManager.shared.photos.count > 0 {
    self.navigationItem.prompt = nil
  } else {
    self.navigationItem.prompt = "Add photos with faces to Googlyify them!"
  }

  // 3
  self.navigationController?.viewIfLoaded?.setNeedsLayout()
}
```

**为什么不用 Timer**

1. 可读性，Timer 需要定义一个方法，然后使用定义的方法为 selector 创建一个 Timer。GCD 方法只需要闭包
2. Timer 依赖于 runloop，所以使用 Timer 需要保证它被添加进正确的 runloop mode。
3. Timer 更适合用在需要重复的任务

## Handling the Readers-Writers Problem

可以通过 dispatch barriers 解决[Readers-Writers Problem](http://en.wikipedia.org/wiki/Readers–writers_problem)

当提交 `DispatchWorkItem` 到队列的时候，可以设置一个标识来表示这个任务在执行的时候所在队列只有这个任务在执行。

1. 所有 dispatch barrier 之前的任务都执行完成之后这个 `DispatchWorkItem` 才会执行
2. 当这个`DispatchWorkItem` 执行的时候，所在队列只有这个任务在执行
3. `DispatchWorkItem` 执行完成之后，所在队列回到默认的表现

<img width="50%" height="50%" src="/assets/images/Dispatch-Barrier-Swift.png"/>

> 全局队列为共享，在全局队列中使用 barrier 需要注意
>
> 在自定义的并发队列中使用 barrier 实现线程安全是比较好的实践

barrier 解决了写的问题，在同一个队列进行读操作，但是使用 `sync` 方法来保证方法返回读到数据。

> 当需要**等待**任务结束之后再使用任务闭包处理的数据时，使用  sync

> **死锁 deadlock**
>
> 如果在执行的队列上调用 sync 会导致死锁
>
> sync 会等待闭包执行完成，但是闭包在当前执行的闭包调用完成之前不会调用完成或者调用，但是当前执行的闭包又在等待 sync 的闭包调用结束。

## Dispatch Groups

dispatch group 可以将多个任务组合在一起，然后**等待**所有的任务完成或者在所有的任务完成之后得到**通知**。

一个组中的任务可以是同步的或者异步的，并且可以运行在不同的队列中。

`DispatchGroup` 来管理 dispatch groups。

### **wait** 

这个方法会阻塞当前线程直到所有入队到组中的任务都完成

1. 由于会阻塞当前线程，所有应该使用 GCD 的 `async` 方法和后台队列来保证不会阻塞主线程
2. 创建一个组
3. 在每个任务开始前调用组的 `enter` 方法，在每个任务结束的时候调用组的 `leave` 方法，两个方法调用必须匹配
4. 调用组的 `wait` 方法来阻塞当前线程直到所有的任务都完成(都调用了 `leave`)。或者调用 `wait(timeout:)` 来指定超时时间。
5. `wait` 之后的调用是所有的任务都结束了或者超时了，可以处理任务之后的数据。

### **notify**

使用这个方法可以在在所有组中的任务完成之后得到通知，并且不会像 `wait` 方法一样阻塞当前线程，并且可以在 `notify` 方法中指定接收通知的队列。

1. 创建一个 `DispatchGroup`
2. 在任务开始前调用组的 `enter` 方法，在任务结束之后调用组的 `leave` 方法，同样两个方法的调用需要匹配
3. 调用组的 `notify(queue:work:)` 方法指定接收通知的队列和后续的处理

## Concurrency Looping

`DispatchQueue.concurrentPerform(iterations:execute:)` 可以并发地执行遍历，它是同步的，会在所有的遍历任务完成后退出

> 需要注意遍历迭代的次数和每次迭代的工作量，如果迭代次数很多并且每次迭代的工作量很小会造成很多开销以至于抵消并发迭代的收益。**striding** 的技术可以帮助我们避免这种情况，striding 可以让我们在一次迭代中做多个部分的任务

**什么时候使用**

1. 可以排除串行队列，因为根本没有益处
2. 在包含循环的并发队列中使用是一种好的选择，尤其是当你需要追踪进度的时候

```swift
// 仅为示例，下面的情况并不合适用并发遍历

var storedError: NSError?
let downloadGroup = DispatchGroup()
let addresses = [PhotoURLString.overlyAttachedGirlfriend,
                 PhotoURLString.successKid,
                 PhotoURLString.lotsOfFaces]
// 告诉GCD使用QoS为 .userInitiated的队列来实现并发调用
let _ = DispatchQueue.global(qos: .userInitiated)
DispatchQueue.concurrentPerform(iterations: addresses.count) { index in
  let address = addresses[index]
  let url = URL(string: address)
  downloadGroup.enter()
  let photo = DownloadPhoto(url: url!) { _, error in
    if error != nil {
      storedError = error
    }
    downloadGroup.leave()
  }
  PhotoManager.shared.addPhoto(photo)
}
downloadGroup.notify(queue: DispatchQueue.main) {
  completion?(storedError)
}
```

## Canceling Dispatch Blocks

GCD 的任务以闭包的形式派发，实际是一个 `DispatchWorkItem`，取消就是用到了 `DispatchWorkItem` 来发挥作用

>  **取消必须在任务到达队列的头开始执行之前**。

1. 使用 `DispatchWorkItem(qos:flags:block:)` 创建一个对象，`flags` 可以指定 `.inheritQoS` 来使用派发到的队列的 QoS
2. 派发所有创建的 `DispatchWorkItem` 到一个队列执行，可利用串行队列的特性保证后续的取消操作的在任务开始之前执行
3. 调用 `DispatchWorkItem` 的 `cancel` 方法来取消任务的执行，并调用组的 `leave` 方法

配合[代码](https://github.com/hotchner/Demos/blob/master/GooglyPuff/GooglyPuff/PhotoManager.swift)享用，更多参考 [Apple's documentation](https://developer.apple.com/documentation/dispatch/dispatchworkitem)

## Miscellaneous GCD Fun

### Semaphores

信号量问题讨论参考 [detailed discussion](https://greenteapress.com/wp/semaphores/) 和  [Dining Philosophers Problem](http://en.wikipedia.org/wiki/Dining_philosophers_problem)

1. 创建一个信号量并指定初始值，这个值代表可以同时访问的数量
2. 当一个访问结束的时候，调用信号量的 `signal` 方法增加信号量的值
3. 调用信号量的 `wait` 方法阻塞当前线程直到信号量的值大于0即资源可访问，`wait` 方法还可以指定超时时间。信号量的返回值是枚举，`success` 或者 `timeout`

### Dispatch Sources

dispatch sources 可以用来监听一些事件，包括 Unix signals、file descriptors、Mach ports、VFS Nodes等

**设置 dispatch source**

1. 设置要监听的事件类型、接收事件回调的 dispatch queue
2. 将事件 handler 赋值给 dispatch source
3. 前两步完成后 dispatch source 处于 suspended 状态，从而允许我们进行进一步的设置，比如设置 event handler。设置完成后，调用 source 的 `resume` 方法开始事件处理

**一个🌰**

```swift
// 1
#if DEBUG

  // 2
  var signal: DispatchSourceSignal?

  // 3
  private let setupSignalHandlerFor = { (_ object: AnyObject) in
    let queue = DispatchQueue.main

    // 4
    signal =
      DispatchSource.makeSignalSource(signal: SIGSTOP, queue: queue)
        
    // 5
    signal?.setEventHandler {
      print("Hi, I am: \(object.description!)")
    }

    // 6
    signal?.resume()
  }
#endif

// 在viewDidLoad中调用
#if DEBUG
  setupSignalHandlerFor(self)
#endif
```

DEBUG 模式中，点击 Xcode debug 栏中的 pause 和 continue 可以看到输出信息

**应用场景猜想**

1. 做 app 防护，防止攻击者 attache debugger 到 app 上

2. 做堆栈追踪工具，方便找到想在 debugger 中操作的对象

3. 辅助调试，还是上面的🌰，在 EventHandler 的 print 方法上增加断点，则可以在 app 运行的任意时刻通过 pause 和 continue 进入到此断点，然后进行进一步的调试

   ```swift
   expression let $vc = unsafeBitCast(0x7fd301d0a310, to: GooglyPuff.PhotoCollectionViewController.self)
   expression $vc.navigationItem.prompt = "WOOT!"
   ```

## Reference

1. 不完整翻译自 raywenderlich 的 grand-central-dispatch-tutorial-for-swift-4 [part-1](https://www.raywenderlich.com/5370-grand-central-dispatch-tutorial-for-swift-4-part-1-2)  [part-2](https://www.raywenderlich.com/5371-grand-central-dispatch-tutorial-for-swift-4-part-2-2)
2. [demo](https://github.com/hotchner/Demos/tree/master/GooglyPuff)
