---
title: 为什么使用 Rx
categories:
  - RxSwift
tags:
  - RxSwift
toc: true
author_profile: false
sidebar:
  nav: RxSwift-docs
---

## 为什么使用 Rx

**Rx 可以使用声明式的语法构建 App**

### 绑定

```swift
Observable.combineLatest(firstName.rx.text, lastName.rx.text) { $0 + " " + $1 }
    .map { "Greetings, \($0)" }
    .bind(to: greetingLabel.rx.text)
```

同样适用于 `UITableView` 和 `UICollectionView`

```swift
viewModel
    .rows
    .bind(to: resultsTableView.rx.items(cellIdentifier: "WikipediaSearchCell", cellType: WikipediaSearchCell.self)) { (_, viewModel, cell) in
        cell.title = viewModel.title
        cell.url = viewModel.url
    }
    .disposed(by: disposeBag)
```

> 官方建议始终使用 `.disposed(by: disposeBag)`， 即使对于简单的绑定来说是非必要的

### 重试

考虑 API 会有失败的情况。比如下面的 API :

```swift
func doSomethingIncredible(forWho: String) throws -> IncredibleThing
```

像这样的方法调用很难失败的时候需要重试的情况，更不用说建模[指数退避](https://en.wikipedia.org/wiki/Exponential_backoff)的复杂性了。如果一定要实现错误处理的话，需要保存很多的暂态，而这些状态是我们不关心的，而且不可以重用。

RxSwift 中，可以直接抓住重试的本质，而不必保存一系列暂时的状态，并且可以应用在任何的操作上。

用 RxSwift 可以这样写

```swift
doSomethingIncredible("me")
    .retry(3)
```

你也可以方便地创建自定义的重试操作。

### Delegates

传统的写法冗长且表意不明

```swift
public func scrollViewDidScroll(scrollView: UIScrollView) { [weak self] // what scroll view is this bound to?
    self?.leftPositionConstraint.constant = scrollView.contentOffset.x
}
```

RxSwift 的写法

```swift
self.resultsTableView
    .rx.contentOffset
    .map { $0.x }
    .bind(to: self.leftPositionConstraint.rx.constant)
```

### KVO

传统的写法，被观察的对象释放的时候，`observers` 依然在注册，造成了内存泄漏，甚至被错误地被绑定在其它的对象上。

而且写法不友好

```objc
-(void)observeValueForKeyPath:(NSString *)keyPath
                     ofObject:(id)object
                       change:(NSDictionary *)change
                      context:(void *)context
```

用 RxSwift 的  [`rx.observe` and `rx.observeWeakly`](GettingStarted.md#kvo)

```swift
view.rx.observe(CGRect.self, "frame")
    .subscribe(onNext: { frame in
        print("Got new frame \(frame)")
    })
    .disposed(by: disposeBag)
```

```swift
someSuspiciousViewController
    .rx.observeWeakly(Bool.self, "behavingOk")
    .subscribe(onNext: { behavingOk in
        print("Cats can purr? \(behavingOk)")
    })
    .disposed(by: disposeBag)
```

### Notifications

传统的写法

```swift
@available(iOS 4.0, *)
public func addObserverForName(name: String?, object obj: AnyObject?, queue: NSOperationQueue?, usingBlock block: (NSNotification) -> Void) -> NSObjectProtocol
```

RxSwift 的写法

```swift
NotificationCenter.default
    .rx.notification(NSNotification.Name.UITextViewTextDidBeginEditing, object: myTextView)
    .map {  /*do something with data*/ }
    ....
```

### 暂态

在编写异步程序时，暂态会带来很多问题，一个典型的问题是自动完成的搜索框。

如果你用 RxSwift 编写自动完成的代码，第一个要解决的问题是当 `abc` 中的 `c` 被输入后，`ab` 还未发生的时候被取消了，要解决这个问题需要保存一下额外的变量来持有将要发生的请求。

下一个问题问题是如果请求失败，需要做麻烦的重试逻辑。同时，多个捕获重试次数的字段需要被清理。

最好程序能够在发起请求之前能等待一定的时间。总之，有可能有用户输入非常离谱，但是我们不想向服务器发送垃圾请求。添加一个计时器吗？

另一个问题是，当请求发生的时候屏幕上应该展示什么，所有的重试请求都失败的时候又应该展示什么。

用传统的方法写出这样的需求及相应的测试是繁琐的。

RxSwift 的方式是：

```swift
searchTextField.rx.text
    .throttle(.milliseconds(300), scheduler: MainScheduler.instance)
    .distinctUntilChanged()
    .flatMapLatest { query in
        API.getSearchResults(query)
            .retry(3)
            .startWith([]) // clears results on new search term
            .catchErrorJustReturn([])
    }
    .subscribe(onNext: { results in
      // bind to ui
    })
    .disposed(by: disposeBag)
```

再不需要额外的标记和字段了，Rx 会处理所有的暂态

### 成分清理

我们假定一个场景是需要再一个 table view 中展示模糊化的图片。
首先，图片需要从网络获取，然后解码和模糊化处理。

我们希望整个处理过程是可以被取消的，因为宽带和模糊化处理时间都是比较消耗资源的。

我们也希望获取图片的请求不是再 cell 展示出来的时候立刻发生的，因为如果用户滑动的非常快就会有非常多的请求被发起和取消。

我们还希望图片处理有一个并发量的限制，同样因为图片的模糊化处理是比较消耗资源的操作。

Rx 写法是:

```swift
// this is a conceptual solution
let imageSubscription = imageURLs
    .throttle(.milliseconds(200), scheduler: MainScheduler.instance)
    .flatMapLatest { imageURL in
        API.fetchImage(imageURL)
    }
    .observeOn(operationScheduler)
    .map { imageData in
        return decodeAndBlurImage(imageData)
    }
    .observeOn(MainScheduler.instance)
    .subscribe(onNext: { blurredImage in
        imageView.image = blurredImage
    })
    .disposed(by: reuseDisposeBag)
```

这段代码可以实现上述所有的需求，并且 `imageSubscription` 被清理了，它将会取消所有依赖的异步操作并保证不会有错误的图片展示在 UI 上。

### 聚合网络请求

如果我们想发起两个请求并且在它们都完成的时候聚合它们的结果呢？

Rx 的 `zip` 运算符就是来解决这个问题的

```swift
let userRequest: Observable<User> = API.getUser("me")
let friendsRequest: Observable<[Friend]> = API.getFriends("me")

Observable.zip(userRequest, friendsRequest) { user, friends in
    return (user, friends)
}
.subscribe(onNext: { user, friends in
    // bind them to the user interface
})
.disposed(by: disposeBag)
```

如果这些 API 在后台线程返回结果，绑定操作又必须发生在主线成呢？

可以用 `observeOn` 来实现

```swift
let userRequest: Observable<User> = API.getUser("me")
let friendsRequest: Observable<[Friend]> = API.getFriends("me")

Observable.zip(userRequest, friendsRequest) { user, friends in
    return (user, friends)
}
.observeOn(MainScheduler.instance)
.subscribe(onNext: { user, friends in
    // bind them to the user interface
})
.disposed(by: disposeBag)
```

### 状态

编程语言允许修改使我们可以获取全局的状态并修改它。不受控制的对全局状态的修改很容易引起 [组合性爆炸](https://en.wikipedia.org/wiki/Combinatorial_explosion#Computing)

另一方面，正确地使用的时候，[命令式语言](https://en.wikipedia.org/wiki/Imperative_programming)可以编写更加更加接近硬件的更有效率的代码

通常的对抗组合行爆炸的方式是保持状态极可能地简单，并且使用[单向数据流](https://developer.apple.com/videos/play/wwdc2014-229)来对衍生的数据建模

这里是 Rx 出色的地方

Rx 是一个在函数式和命令式的世界中一个闪光点，它让你能够以一种可靠的组合的方式使用不可变的定义和纯粹的函数去处理可变状态的快照

### 集成简单

创建自定义的观察的方式也很简单，下面是一个 RxCocoa 中的例子，仅仅这些就实现了对 `URLSesstion` 的 HTTP 请求的封装

```swift
extension Reactive where Base: URLSession {
    public func response(request: URLRequest) -> Observable<(Data, HTTPURLResponse)> {
        return Observable.create { observer in
            let task = self.base.dataTask(with: request) { (data, response, error) in

                guard let response = response, let data = data else {
                    observer.on(.error(error ?? RxCocoaURLError.unknown))
                    return
                }
    
                guard let httpResponse = response as? HTTPURLResponse else {
                    observer.on(.error(RxCocoaURLError.nonHTTPResponse(response: response)))
                    return
                }
    
                observer.on(.next(data, httpResponse))
                observer.on(.completed)
            }
    
            task.resume()
    
            return Disposables.create(with: task.cancel)
        }
    }
}
```

### 收益

简言之，使用 Rx 可以使你的代码：

- 可组合的 <- 因为 Rx 就是组合的别名
- 可重用的 <- 因为可组合的

* 声明式的 <- 因为定义是不可变的，只有数据在变
* 可理解和简洁的 <- 提高抽象等级，移除暂态
* 稳定的 <- 因为 Rx 的代码经过了彻底的单元测试
* 更少的状态 <- 因为你在对应用以单向数据流的方式建模
* 没有内存泄漏 <- 因为资源的管理简单

### It's not all or nothing

尽可能多地使用 Rx 对你的应用进行建模是一个好的想法

但是如果你不知道所有的操作符或者不确定是不是存在一些操作符可以满足特定的需求呢？

其实，所有的 Rx 操作符都是基于**数学**，并且都是直观的

好消息是大概 10-15 个操作符就可以覆盖通常的需求，包括常见的 `map`, `filter`, `zip`, `observeOn` 等

这里是庞大的[所有 Rx 操作符](http://reactivex.io/documentation/operators.html)的列表

每一个操作符，都有一个[弹珠图](http://reactivex.io/documentation/operators/retry.html)来帮助解释它的实现

如果你需要一些没有在列表中的操作符，可以自定义操作符

如果觉得新建一个操作符非常困难，或者你需要和遗留的状态的代码片段配合，那么你陷入了困境，但是你可以容易地[跳出困境](https://github.com/ReactiveX/RxSwift/blob/5.0.1/Documentation/GettingStarted.md#life-happens)，处理数据，然后返回它。

What if creating that kind of operator is really hard for some reason, or you have some legacy stateful piece of code that you need to work with? Well, you've got yourself in a mess, but you can [jump out of Rx monads](GettingStarted.md#life-happens) easily, process the data, and return back into it.

## 参考

1. [5.0.1 doc](https://github.com/ReactiveX/RxSwift/blob/5.0.1/Documentation/Why.md)
2.   [命令式编程（Imperative） vs声明式编程（ Declarative)](https://zhuanlan.zhihu.com/p/34445114)
3.   [stackoverflow](https://stackoverflow.com/questions/17826380/what-is-difference-between-functional-and-imperative-programming-languages)