---
title: Swift 范型入门
excerpt: 什么是范型，为什么有用，怎么写范型的方法和数据结构，如何使用类型约束，如何扩展范型类型
categories:
  - Swift
tags:
  - Generic
toc: true
---

## 引言

**范型解决了代码复用的问题**

使用范型可以避免写出类似于下面这样的重复代码

```swift
func addInt(a: Int, b: Int) -> Int {
  return a + b
}

func addDouble(a: Double, b: Double) -> Double {
  return a + b
}
```

Swift 标准库中很多常见的数据结构都使用了范型，比如 **array, dictionary, optional, result**

## Swift 标准库中的范型

### Array

Swift 中 Array 的定义为 

```swift
public struct Array<Element> { }
```

`Element` 为类型参数，每一个范型的数据结构结构中都至少有一个范型参数，代表还未确定的类型。

类型参数确定之后，Array 中只能存储该类型的数据。

### Dictionary

Swift 中 Dictionary 的定义为

```swift
public struct Dictionary<Key, Value> where Key : Hashable { }
```

这里有两个类型参数，并且 Key 限定为必须满足 Hashable 协议。

```swift
let countryCodes = ["Arendelle": "AR", "Genovia": "GN", "Freedonia": "FD"]
let countryCode = countryCodes["Freedonia"]
```

上述代码会被类型推断为 `[String : String]` 类型

### Optional

Swift 中 Optional 的定义为

```swift
public enum Optional<Wrapped> : ExpressibleByNilLiteral {
  case none
  case some(Wrapped)
}
```

Wrapped 为类型参数，同时遵守了 ExpressibleByNilLiteral 协议，可以直接通过赋值 nil 初始化

其中定义了两个 case，`none` 和 `some(Wrapped)`，some 中通过关联值保存了范型类型的值

```swift
let optionalName = Optional<String>.some("Princess Moana")
if let name = optionalName {}
```

### Result

Result 是 Swift 5 中新增的类型，定义如下

```swift
public enum Result<Success, Failure> where Failure : Error {
  case success(Success)
  case failure(Failure)
}
```

直接看一个例子

```swift
enum MagicError: Error {
  case spellFailure
}

func cast(_ spell: String) -> Result<String, MagicError> {
  switch spell {
  case "flowers":
    return .success("💐")
  case "stars":
    return .success("✨")
  default:
    return .failure(.spellFailure)
  }
}
```

## 范型数据结构

以一个队列为例查看一下如何自定义一个范型的数据结构

```swift
struct Queue<Element> {
  private var elements: [Element] = []
  mutating func enqueue(newElement: Element) {
    elements.append(newElement)
  }
  mutating func dequeue() -> Element? {
    guard !elements.isEmpty else { return nil }
    return elements.remove(at: 0)
  }
}
```

> 如上的 Elemen 类型在整个结构体中都是可以使用的，即使是在方法的内部

使用👇

```swift
var q = Queue<Int>()

q.enqueue(newElement: 4)
q.enqueue(newElement: 2)

q.dequeue()
q.dequeue()
q.dequeue()
q.dequeue()
```

## 范型函数

下面的方法实现将字典转换元祖数组

```swift
func pairs<Key, Value>(from dictionary: [Key: Value]) -> [(Key, Value)] {
  return Array(dictionary)
}
```

使用👇

```swift
let somePairs = pairs(from: ["minimum": 199, "maximum": 299])
// result is [("maximum", 299), ("minimum", 199)]

let morePairs = pairs(from: [1: "Swift", 2: "Generics", 3: "Rule"])
// result is [(1, "Swift"), (2, "Generics"), (3, "Rule")]
```

## 范型的类型约束

实现查找中位数的方法

```swift
func mid<T>(array: [T]) -> T? {
  guard !array.isEmpty else { return nil }
  return array.sorted()[(array.count - 1) / 2]
}
```

上述代码会出现编译错误，因为 `sorted` 方法要求元素遵守 `Comparable` 协议，所以需要对范型类型增加一个约束

```swift
func mid<T: Comparable>(array: [T]) -> T? {
  guard !array.isEmpty else { return nil }
  return array.sorted()[(array.count - 1) / 2]
}
```

使用约束的语法和遵守协议的语法相同，在类型后面使用冒号 `:` 跟着要遵守的协议就可以了

## 清理求和函数

使用范型来清理文章开始提到的求和方法

定义一个协议，约束类型必须实现 `+` 方法，然后让 `Int` 和 `Double` 遵守协议

```swift
protocol Summable { static func +(lhs: Self, rhs: Self) -> Self }
extension Int: Summable {}
extension Double: Summable {}
```

定义范型的求和方法

```swift
func add<T: Summable>(x: T, y: T) -> T {
  return x + y
}
```

可以对 `Int` 类型和 `Double` 使用 `add` 方法

```swift
let addIntSum = add(x: 1, y: 2) // 3
let addDoubleSum = add(x: 1.0, y: 2.0) // 3.0
```

由于使用协议约束，所以并不限定范型的具体类型必须是数值，遵守了 `Summable` 协议的任何类型都可以

```swift
extension String: Summable {}
let addString = add(x: "Generics", y: " are Awesome!!! :]")
```

## 扩展范型

范型类型在范型数据结构的扩展中也是可以使用，扩展上面的范型队列

```swift
extension Queue {
  func peek() -> Element? {
    return elements.first
  }
}
```

## 继承范型

范型的类可以被继承，一个使用场景是继承并实现抽象的类

```swift
class Box<T> {
  // Just a plain old box.
}
```

`Box` 可以存储任何类型，因为是范型定义的。

对范型类的继承可以有两种方式

1. 可以继续保持范型，那么依旧可以存储任何类型

2. 可以把范型具体化，那么该子类就只能存储一个确定的类型

   > Swift 对于两种继承方式都是支持的

```swift
class Gift<T>: Box<T> {
  // By default, a gift box is wrapped with plain white paper
  func wrap() {
    print("Wrap with plain white paper.")
  }
}

class Rose {
  // Flower of choice for fairytale dramas
}

class ValentinesBox: Gift<Rose> {
  // A rose for your valentine
}

class Shoe {
  // Just regular footwear
}

class GlassSlipper: Shoe {
  // A single shoe, destined for a princess
}

class ShoeBox: Box<Shoe> {
  // A box that can contain shoes
}
```

## 带关联值的枚举

枚举的关联值也可以范型化

```swift
enum Reward<T> {
  case treasureChest(T)
  case medal

  var message: String {
    switch self {
    case .treasureChest(let treasure):
      return "You got a chest filled with \(treasure)."
    case .medal:
      return "Stand proud, you earned a medal!"
    }
  }
}
```

可以像下面这样使用

```swift
let message = Reward.treasureChest("💰").message
print(message)
```

> Swift 5 中新出的 Result 类型就是范型枚举的一个应用

## Reference

1. <https://www.raywenderlich.com/3535703-swift-generics-tutorial-getting-started>