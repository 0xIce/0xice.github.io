---
title: RxSwift Filtering Operators
excerpt: 讲解 Filter 操作符的使用
categories:
  - RxSwift
tags:
  - RxSwift
toc: true
author_profile: false
sidebar:
  nav: RxSwift-docs
---

## Ignoring operators

### ignoreElements

使用 `ignoreElements` 操作符可以忽略所有的 `.next` 事件，而让终止事件通过，比如 `.error` 和 `.completed`

看一个例子

```swift
example(of: "ignoreElements") {
  // 1
  let strikes = PublishSubject<String>()

  let disposeBag = DisposeBag()

  // 2
  strikes
    .ignoreElements()
    .subscribe { _ in
      print("You're out!")
    }
    .disposed(by: disposeBag)
  
  strikes.onNext("X")
	strikes.onNext("X")
	strikes.onNext("X")
  strikes.onCompleted()
}

输出：
--- Example of: ignoreElements ---
You're out!
```

上面的例子可以看出，所有的 next 事件都被忽略了，只剩 completed 事件通过，所以只打印了一次

> ignoreElements 操作符会返回一个 Completable 序列

### elementAt

在只希望处理序列中的第 n 个事件的时候可以使用 `elementAt` 操作符，它会只通过指定 index 的事件，忽略其它的事件

```swift
example(of: "elementAt") {

  // 1
  let strikes = PublishSubject<String>()

  let disposeBag = DisposeBag()

  //  2
  strikes
    .elementAt(2)
    .subscribe(onNext: { _ in
      print("You're out!")
    })
    .disposed(by: disposeBag)
  
  strikes.onNext("X")
	strikes.onNext("X")
	strikes.onNext("X")
}

输出：
--- Example of: elementAt ---
You're out!
```

> 在发出指定 index 的事件之后，订阅将会终止

### filter

filter 操作符和 Swift 标准库中的 filter 方法基本相同，根据 filter block 的返回值做筛选，只允许通过返回值为 true 的元素

```swift
example(of: "filter") {
  let disposeBag = DisposeBag()

  // 1
  Observable.of(1, 2, 3, 4, 5, 6)
    // 2
    .filter { $0 % 2 == 0 }
    // 3
    .subscribe(onNext: {
      print($0)
    })
    .disposed(by: disposeBag)
}

输出：
--- Example of: filter ---
2
4
6
```

## Skipping operators

### skip

skip 操作符可以跳过前 n 个事件，比如 `skip(2)` 就是跳过前两个事件

```swift
example(of: "skip") {
  let disposeBag = DisposeBag()

  // 1
  Observable.of("A", "B", "C", "D", "E", "F")
    // 2
    .skip(3)
    .subscribe(onNext: {
      print($0)
    })
    .disposed(by: disposeBag)
}

输出：
--- Example of: skip ---
D
E
F
```

### skipWhile

skipWhile 与 filter 类似的地方是接收一个筛选条件

不同的是 skipWhile 在通过第一个元素之后便不再筛选，会让之后的元素都通过

skipWhile 的筛选条件为 true 时会跳过事件，为 false 时会让事件通过

```swift
example(of: "skipWhile") {
  let disposeBag = DisposeBag()

  // 1
  Observable.of(2, 2, 3, 4, 4)
    // 2
    .skipWhile { $0 % 2 == 0 }
    .subscribe(onNext: {
      print($0)
    })
    .disposed(by: disposeBag)
}

输出：
--- Example of: skipWhile ---
3
4
4
```

### skipUntil

之前的筛选操作符都是基于静态的条件，skipUntil 操作符及后面的一些操作符可以把其它的序列作为筛选条件

skipUntil 接收一个 Observable 的 trigger 参数，在 trigger 发出一个 `.next` 事件之前，skipUntil 会把所有的事件跳过

```swift
example(of: "skipUntil") {
  let disposeBag = DisposeBag()

  // 1
  let subject = PublishSubject<String>()
  let trigger = PublishSubject<String>()

  // 2
  subject
    .skipUntil(trigger)
    .subscribe(onNext: {
      print($0)
    })
    .disposed(by: disposeBag)
  
  subject.onNext("A")
	subject.onNext("B")
  trigger.onNext("X")
  subject.onNext("C")
}

输出：
--- Example of: skipUntil ---
C
```

## Taking operators

Taking 是和 skipping 相反的操作符

### take

take 操作符让前 n 个操作符通过

```swift
example(of: "take") {
  let disposeBag = DisposeBag()

  // 1
  Observable.of(1, 2, 3, 4, 5, 6)
    // 2
    .take(3)
    .subscribe(onNext: {
      print($0)
    })
    .disposed(by: disposeBag)
}

输出：
--- Example of: take ---
1
2
3
```

### takeWhile

takeWhile 和 skipWhile 相似，但是是通过而不是跳过

takeWhile 在遇到以一个不通过的事件的时候后面的元素都会跳过

```swift
example(of: "takeWhile") {
  let disposeBag = DisposeBag()

  // 1
  Observable.of(2, 2, 4, 4, 6, 6)
    // 2
    .enumerated()
    // 3
    .takeWhile { index, integer in
      // 4
      integer % 2 == 0 && index < 3
    }
    // 5
    .map { $0.element }
    // 6
    .subscribe(onNext: {
      print($0)
    })
    .disposed(by: disposeBag)
}

输出：
--- Example of: takeWhile ---
2
2
4
```

### takeUntil

类似于 skipUntil 接收一个 trigger 参数，在 trigger 发出 next 之前会一直让事件通过，跳过 trigger 发出 next 事件之后的事件

```swift
example(of: "takeUntil") {
  let disposeBag = DisposeBag()

  // 1
  let subject = PublishSubject<String>()
  let trigger = PublishSubject<String>()

  // 2
  subject
    .takeUntil(trigger)
    .subscribe(onNext: {
      print($0)
    })
    .disposed(by: disposeBag)

  // 3
  subject.onNext("1")
  subject.onNext("2")
  
  // trigger
  trigger.onNext("X")
	subject.onNext("3")
}

输出：
--- Example of: takeUntil ---
1
2
```

> takeUntil 的另一个用途是处理订阅的生命周期，如果没有把订阅放到一个 bag 中，可以使用这种方式，但是一般还是使用 DisposeBag
>
> ```swift
> _ = someObservable
>     .takeUntil(self.rx.deallocated)
>     .subscribe(onNext: {
>         print($0)
>     })
> ```

## Distinct operators

Distinct 操作符可以避免**相邻**的相同的元素

```swift
example(of: "distinctUntilChanged") {
  let disposeBag = DisposeBag()

  // 1
  Observable.of("A", "A", "B", "B", "A")
    // 2
    .distinctUntilChanged()
    .subscribe(onNext: {
      print($0)
    })
    .disposed(by: disposeBag)
}

输出：
--- Example of: distinctUntilChanged ---
A
B
A
```

上面例子的元素类型是 String，本身是遵守并实现了 Equatable 协议的类型，所以可以比较，`distinctUntilChanged` 也可以接受自定义的比较

![distinctUntilChanged](/assets/images/distinctUntilChanged.png)

```swift
example(of: "distinctUntilChanged(_:)") {
  let disposeBag = DisposeBag()
  
  // 1
  let formatter = NumberFormatter()
  formatter.numberStyle = .spellOut

  // 2
  Observable<NSNumber>.of(10, 110, 20, 200, 210, 310)
    // 3
    .distinctUntilChanged { a, b in
      // 4
      guard let aWords = formatter.string(from: a)?.components(separatedBy: " "),
      let bWords = formatter.string(from: b)?.components(separatedBy: " ")
        else {
          return false
      }

      var containsMatch = false

      // 5
      for aWord in aWords where bWords.contains(aWord) {
        containsMatch = true
        break
      }

      return containsMatch
    }
    // 4
    .subscribe(onNext: {
      print($0)
    })
    .disposed(by: disposeBag)
}

输出：
--- Example of: distinctUntilChanged(_:) ---
10
20
200
```

