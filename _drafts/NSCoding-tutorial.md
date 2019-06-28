---

title: NSCoding 入门
excerpt: NSCoding、FileManager、NSSecureCoding、硬盘读写、图片存取
tags:
  - Persistence
  - NSCoding
toc: true
---

## 实现 NSCoding

自定义的数据类通过遵守 `NSCoding` 协议来支持 encoding 和 decoding 数据从而可以将数据持久化存储在硬盘中。

`NSCoding` 包括两个方法，`encode(with:)` 作为 encoder，`init(coder:)` 作为decoder

首先定义一个枚举用来保存 encode 需要用到的 key，避免拼写错误

```swift
enum Keys: String {
  case title = "Title"
  case rating = "Rating"
}
```

实现 encode 方法

```swift
func encode(with aCoder: NSCoder) {
  aCoder.encode(title, forKey: Keys.title.rawValue)
  aCoder.encode(rating, forKey: Keys.rating.rawValue)
}
```

> encode 方法将要保存的值作为第一个参数传入，并指定一个 key 作为第二个参数
>
> 要保存的值也**必须**遵守 NSCoding 协议

实现 decode 的初始化方法

```swift
required convenience init?(coder aDecoder: NSCoder) {
  let title = aDecoder.decodeObject(forKey: Keys.title.rawValue) as! String
  let rating = aDecoder.decodeFloat(forKey: Keys.rating.rawValue)
  self.init(title: title, rating: rating)
}
```

> convenience 不是必须的，也不是 NSCoding 协议要求的，只是为了方便调用 Designed 初始化方法

> decode 的 初始化方法正好与 encode 方法相反，是按照指定的 key 从 NSCoder 对象中获取对应的值

encode 和 decode 的方法支持的数据类型分为基本数据类型、byte类型和 object 类型，基本数据类型包括`Bool, Int32, Int64, Float, Double`，object 类型可以在 decode 的时候 type cast 到具体的类型

## 加载和保存到硬盘

> 出于表现效率考虑，不应该一次性加载所有的数据

### 添加初始化方法



## 参考

1. 不完整翻译自 [raywenderlich](https://www.raywenderlich.com/6733-nscoding-tutorial-for-ios-how-to-permanently-save-app-data)
2. [demo](https://github.com/hotchner/Demos/tree/master/ScaryCreatures)