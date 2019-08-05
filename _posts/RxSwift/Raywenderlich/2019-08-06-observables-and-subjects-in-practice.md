---
title: Observables and Subjects in Practice
excerpt: Observable 和 Subject 实战
categories:
  - RxSwift
tags:
  - RxSwift
toc: true
author_profile: false
sidebar:
  nav: RxSwift-docs
---

## Controller 之间的传值

传统的做法可以用代理的方式实现，需要定义协议及遵守协议、实现协议方法等步骤

使用 RxSwift 可以使用一种通用的在任意两个页面之间通信，而不需要协议，因为 Observable 可以传递任意类型的值到订阅者

```swift
private let selectedPhotosSubject = PublishSubject<UIImage>()
var selectedPhotos: Observable<UIImage> {
  return selectedPhotosSubject.asObservable() // asObservable() 之后不会被误改，相当于变成了只读属性
}
```

在订阅者页面使用代码

```swift
photosViewController.selectedPhotos
      .subscribe(
        onNext: { [weak self] (newImage) in
          guard let images = self?.images else { return }
          images.accept(images.value + [newImage])
      }, onDisposed: {
        print("completed photo selection")
      })
      .disposed(by: bag)
```

> 订阅的代码可能会造成问题，因为 bag 是在订阅者页面持有的，所以在 Observable 页面消失之后订阅并没有消失，造成内存泄漏
>
> 解决方法是在 Observable 页面消失的时候调用发送终止消息，调用 onCompleted()

## 创建自定义的 observable

以一个 PhotoWriter 的 class 为例实现自定义的 observable

创建一个保存图片的 Observable 的实现很简单：

1. 如果图片成功写入 disk，发出它的 asset ID 和一个 `.completed` 事件
2. 失败则发出 `.error` 事件

自定义的实现

```swift
class PhotoWriter {
  enum Errors: Error {
    case couldNotSavePhoto
  }

  static func save(_ image: UIImage) -> Observable<String> {
    return Observable.create({ observer  in
      var savedAssetId: String?
      PHPhotoLibrary.shared().performChanges({
        let request = PHAssetChangeRequest.creationRequestForAsset(from: image)
        savedAssetId = request.placeholderForCreatedAsset?.localIdentifier
      }, completionHandler: { (success, error) in
        DispatchQueue.main.async {
          if success,
            let id = savedAssetId {
            observer.onNext(id)
            observer.onCompleted()
          } else {
            observer.onError(error ?? Errors.couldNotSavePhoto)
          }
        }
      })
      
      return Disposable.create()
    })
  }
}
```

## Traits 实战

### Single

只能产生一个 .success(value) 或者 .error 事件

可以对 Observable 通过调用 `asSingle()` 方法转换为 Single 序列

### Maybe

只能产生一个.next(value) 或者 .completed 或者 .error

可以调用 `Maybe.create({ ... })` 直接创建或者对 Observable 调用 `asMaybe()` 方法转换为 Maybe 序列

### Completable

只能产生一个 completed 或者 error

可以使用 `ignoreElements()` 操作符将 Observable 转换为 Completable，这个操作符会忽略所有的 next 事件，只传递 completed 和 error 事件

也可以使用 `Completable.create { … }` 直接创建

completable 的一个应用场景是，App 会在后台保存数据，保存成功和失败的时候有对应的提示

```swift
saveDocument()
  .andThen(Observable.from(createMessage))
  .subscribe(onNext: { message in
    message.display()
  }, onError: {e in
    alert(e.localizedDescription)
  })
```

`andThen` 操作符可以将多个 completable 或者 observable 的 success 事件链接起来，如果有 error 事件产生，会直接跳转到最终订阅的 onError closure 中

## 订阅自定义的 observable

```swift
PhotoWriter.save(image)
	.asSingle()
  .subscribe(onSuccess: { [weak self] (id) in
		self?.showMessage("保存成功，id: \(id)")
    self?.actionClear()
   }) { (error) in
    self.showMessage("Error", description: error.localizedDescription)
   }
  .disposed(by: bag)
```

> asSingle() 保证序列只会产生一个 success 或者 error 事件，如果产生了多于一个事件就会抛出异常

