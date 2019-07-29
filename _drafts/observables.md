---
title: RxSwift Observables
excerpt: 创建和订阅 Observable 实践
categories:
  - RxSwift
tags:
  - RxSwift
toc: true
author_profile: false
sidebar:
  nav: RxSwift-docs
---

帮助方法

```swift
public func example(of description: String, action: () -> Void) {
  print("\n--- Example of:", description, "---")
  action()
}
```

## 什么是 Observable?

> Observable 即序列

Observable 是 Rx 的核心，你可能会听到多 `observable`、`observable sequence`、`sequence`，它们都是指同一个东西。

## Observable 的生命周期

1. Observable 发出 `next` 事件，携带着元素
2. 重复产生 `next` 事件，直到被终止，无论是 `completed` 事件还是 `error` 事件
3.  被终止之后，不会再产生事件

可以直接看一下 RxSwift 中对这三种事件的定义

```swift
public enum Event<Element> {
    /// Next element is produced.
    case next(Element)

    /// Sequence terminated with an error.
    case error(Swift.Error)
    
    /// Sequence completed successfully.
    case completed
}
```

## 创建 Observable

查看一个例子

```swift
example(of: "just, of, from") {
  // 1
  let one = 1
  let two = 2
  let three = 3
  
  // 2
  let observable = Observable<Int>.just(one)
}
```

使用 `just` 方法创建一个 Int 类型的序列，其中只有一个 `one` 值

`just` 的作用就是创建一个只包含一个值的序列，它是一个 Observable 的类型方法，在 RxSwift 的世界里，这类方法被称作 **操作符 Operator**

在代码中增加一行

```swift
let observable2 = Observable.of(one, two, three)
```

observable2 没有像 observable1 一样指定类型，它的类型也是 Int 而不是 [Int]，因为 `of` 方法接受可变参数类型，Swift 的类型推断特性会推断出正确的类型

如果你确实想要创建一个 Array 类型的序列，那么就向 `of` 方法传递数组参数就可以了

```swift
let observable3 = Observable.of([one, two, three])
```

另一个创建序列的操作符是 `from`

```swift
let observable4 = Observable.from([one, two, three])
```

`from` 操作符会依次取数组参数重的元素作为序列的元素，而不是把数组当作一个整体，所以 observable4 的类型是 Int，而不是 [Int]

## 订阅 Observable

使用 `subscribe` 方法订阅序列

> 在有订阅者之前，observable 不会发出事件

订阅者可以直接调用

```swift
observable.subscribe { event in
  if let element = event.element {
    print(element)
  }
}
```

也可以直接传递关心的事件类型的block，比如说 `next`

```swift
observable.subscribe(onNext: { element in
  print(element)
})
```

**Empty 操作符**

有一个特殊的操作符 `empty`，它不会产生 `next` 事件，只有 `completed`事件

```swift
example(of: "empty") {
  let observable = Observable<Void>.empty() // empty 操作符无法进行类型推断，必须明确确定范型的类型
  observable.subscribe(
  // 1
  onNext: { element in
    print(element)
  },
  
  // 2
  onCompleted: {
    print("Completed")
  }
  )
}

只会输出 "Completed"
```

empty 操作符可以用在，产生一个立即结束的序列，或者有 0 个元素的序列

**Never 操作符**

与 empty 操作符相反，never 操作符永远也不会产生 completed 事件

**Range 操作符**

使用 range 操作符可以创建一个范围的序列

```swift
example(of: "range") {
  // 1
  let observable = Observable<Int>.range(start: 1, count: 10)
  
  observable
    .subscribe(onNext: { i in  
      // 2
      let n = Double(i)
      let fibonacci = Int(((pow(1.61803, n) - pow(0.61803, n)) / 2.23606).rounded())
      print(fibonacci)
  })
}
```

## 清理和终止

每个订阅都会返回一个 `Disposable` 类型的  `subscription` 对象，可以对其调用 `dipose` 立即终止序列，或者调用 `disposed(by:)` 将其放到一个 DispseBag 中，跟随 bag 总之序列

**为什么需要清理**

如果我们没有调用上述方法，并且又没有在合适的时机对序列进行终止，那么就会造成内存泄露

下面看一个使用 `create` 操作符的例子

```swift
example(of: "create") {
  let disposeBag = DisposeBag()
  
  Observable<String>.create { observer in
  	// 1
  	observer.onNext("1") // onNext 是 on(.next(_:)) 的便利方法
  
  	// 2
  	observer.onCompleted() // onCompleted 是 on(.completed) 的便利方法
  
  	// 3
  	observer.onNext("?")
  
  	// 4
  	return Disposables.create()
  }
  .subscribe(
  	onNext: { print($0) },
  	onError: { print($0) },
  	onCompleted: { print("Completed") },
  	onDisposed: { print("Disposed") }
	)
	.disposed(by: disposeBag) 
}

输出：
--- Example of: create ---
1
Completed
Disposed
```

从输出可以看出，在 `completed` 事件之后，`next` 事件将不会发生作用，并且会终止序列

`error` 事件和 `completed` 事件类似，会终止序列，同时伴随着一个 Error 值

```swift
enum MyError: Error {
  case anError
}
observer.onError(MyError.anError)
```

> 如果没有产生 completed 事件和 error 事件，又没有将返回的 Disposable 对象放到 disposeBag 中，那么将会产生内存泄漏
>

## 创建 Observable 工厂

可以使用 `deferred` 操作符创建序列工厂，根据参数的不同返回不同的序列，对于订阅者来说可以直接使用订阅序列工厂的方式来订阅最终的序列

```swift
example(of: "deferred") {
  let disposeBag = DisposeBag()

  // 1
  var flip = false

  // 2
  let factory: Observable<Int> = Observable.deferred {
    
	// 3
	flip.toggle()

	// 4
	if flip {
  	return Observable.of(1, 2, 3)
	} else {
  	return Observable.of(4, 5, 6)
	}

  }
}
```

```swift
for _ in 0...3 {
  factory.subscribe(onNext: {
    print($0, terminator: "")
  })
  .disposed(by: disposeBag)

  print()
}

输出：
--- Example of: deferred ---
123
456
123
456
```

## 使用特征序列 Traits

特征序列比普通的序列有更有限的表现
特征序列是可选的，可以被普通序列替代，但是使用特征序列可读性更强

RxSwift 中共有 3 个特征序列，Single、Maybe、Completable

Single 序列只能产生一个 .success(value) 或者 .error 事件然后终止序列，success 事件实际是 next 事件和 completed 事件的结合，Single 可以用在网络请求中代表成功或者失败

Completable 序列只能产生一个 .completed 或者 .error 事件，然后终止，不会产生值，可以用在只关心成功或失败的地方，比如文件的写入

Maybe 是 Single 和 Completable 的混合，它可以产生 .success(value) 、.completed、.error 三种事件之一然后终止，使用场景是操作可能会成功或失败，还可能产生一个值

下面看一个使用 Single 的例子，从一个文件中读取内容

```swift
example(of: "Single") {
  // 1
  let disposeBag = DisposeBag()

  // 2
  enum FileReadError: Error {
    case fileNotFound, unreadable, encodingFailed
  }
  
  // 3
  func loadText(from name: String) -> Single<String> {
    // 4
    return Single.create { single in
			// 1
			let disposable = Disposables.create()

			// 2
			guard let path = Bundle.main.path(forResource: name, ofType: "txt") else {
  			single(.error(FileReadError.fileNotFound))
  			return disposable
			}

			// 3
			guard let data = FileManager.default.contents(atPath: path) else {
  			single(.error(FileReadError.unreadable))
  			return disposable
			}

			// 4
			guard let contents = String(data: data, encoding: .utf8) else {
  			single(.error(FileReadError.encodingFailed))
  			return disposable
			}

			// 5
			single(.success(contents))
			return disposable
		}
  }
}
```

在发出 .success(value) 事件之前，会经过多次错误检查

每个错误检查不通过的时候发出 .error 事件，伴随着不同的错误类型，终止序列，返回 Disposable 对象

所有的错误检查都通过之后发出 .success(value) 事件，返回读取到的内容

## Challenge

### Perform side effects

使用 do 操作符可以插入 side effects，他不会以任何方式改变原始的事件，只是将事件原封不动的传递下去

do 操作符还包括一个 onSubscribe handler，这个是在 subscribe 方法中不存在的

```swift
do(onNext:onError:onCompleted:onSubscribe:onDispose) 
```

```swift
example(of: "never") {
  let observable = Observable<Any>.never()
  let disposeBag = DisposeBag()

  observable
    .do(onSubscribe: {
      print("Subscribed")
    })
    .subscribe(
      onNext: { element in
        print(element)
      },
      onCompleted: {
        print("Completed")
      },
      onDisposed: {
        print("Disposed")
      }
    )
    .disposed(by: disposeBag)
}
```

### Print debug info

RxSwift 有一个专门用于调试的操作符，就是 debug 操作符

debug 操作符有很多参数，最有用的可能是标识符参数，它会被打印在每一行中

debug 参数可以插入在多个位置，配合标识符参数可以查看序列信息，比如在 transform 之前和之后分别插入 debug 操作符查看效果

```swift
example(of: "never") {
  let observable = Observable<Any>.never()
  let disposeBag = DisposeBag()

  observable
    .debug("observable")
    .subscribe(
      onNext: { element in
        print(element)
      },
      onCompleted: {
        print("Completed")
      },
      onDisposed: {
        print("Disposed")
      }
    )
    .disposed(by: disposeBag)
}
```

