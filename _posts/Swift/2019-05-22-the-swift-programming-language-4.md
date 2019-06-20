---
title: 重读 Swift (5.0) 第四篇
excerpt: Type Casting、Protocols、Generics、Automatic Reference Counting、Memory Safety、Access Control
tags:
  - Swift
toc: true
---

## Type Casting

1. `is` 可用于所有的类型，包括 `protocol`，判断是否是某个类型， `class` 的子类， 遵守了某个协议
2. `as` 用于类型转换，可用于所有类型，返回可选型的转换后的类型

## Protocols

1. `protocol ` 可以通过在 `protocol` 前加 `objc` 并且在方法和属性前加  `@objc optional` 关键子定义可选的方法和属性
   - 该 `protocol` 只能用于继承于 `Objective-C` 的 `class` 类型中或者同样被 `@objc` 修饰的类中
   - 可选的方法和属性在调用时都会是 `optional` 类型，可能需要解包
2. `protocol` 可以继承多个其它的 `protocol`
3. `protocol` 的 `extension` 中可以新增方法，可以被遵守 `protocol` 的实例调用
4. 对于 `protocol` 加约束的扩展，如果一个实例满足多个约束，则遵守最严格最具体的约束

## Generics

1. 范型的目的是实现灵活/可重用的函数和类型

2. 对一个范型类型添加 `extension` 的时候，不用在 `extension` 中指定范型类型名，可以直接用原类型中定义好的范型类型名

3. `protocol` 中可以使用 `associatedtype` 实现范型

   ```swift
   protocol SomeProtocol {
       associatedtype Item: Equatable // 可以添加约束
   }
   
   struct SomeStruct: SomeProtocol {
   	typealias Item = Int // 如果满足类型推断，可以省略。swift可以通过使用了Item的协议条件的实现中推断出Item的类型
   }
   ```

4. 可以通过 `where` 对多个范型类型进行限定，多个 `where` 条件之间用逗号分隔

   ```swift
   func allItemsMatch<C1: Container, C2: Container>
       (_ someContainer: C1, _ anotherContainer: C2) -> Bool
       where C1.Item == C2.Item, C1.Item: Equatable {
   
           // Check that both containers contain the same number of items.
           if someContainer.count != anotherContainer.count {
               return false
           }
   
           // Check each pair of items to see if they're equivalent.
           for i in 0..<someContainer.count {
               if someContainer[i] != anotherContainer[i] {
                   return false
               }
           }
   
           // All items match, so return true.
           return true
   }
   ```

## Automatic Reference Counting

1. 当 `ARC` 将一个 `weak` 指针置为 `nil` 时，属性 `observers` 不会被调用

2. 使用捕获列表解决 `closure` 的循环引用问题，列表的多个捕获对使用逗号分隔

   ```swift
   lazy var someClosure: (Int, String) -> String = {
       [unowned self, weak delegate = self.delegate!] (index: Int, stringToProcess: String) -> String in
       // closure body goes here
   }
   ```

## Memory Safety

1. 对内存的访问分为即时的(`instantaneous`)和长期的(`long-term`)

   - `long-term` 的操作是指在一次操作开始后结束前可能会发生其它的操作

2. 发生内存安全的问题的条件是

   1. 至少有一个写操作
   2. 操作的是同一块内存
   3. 和写操作的时间重叠

3. `In-Out` 参数导致的内存安全问题

   1. 所有的非 `In-Out` 参数被评估后，所有的 `In-Out` 参数的写权限开始，直到函数的结束

      ```swift
      var stepSize = 1
      
      func increment(_ number: inout Int) {
          number += stepSize
      }
      
      increment(&stepSize)
      // increment中对stepSize的读写重叠，造成内存冲突
      ```

   2. 对于值类型的属性的修改实际是对整个值的修改，所以也有可能造成内存冲突

      ```swift
      var playerInformation = (health: 10, energy: 20)
      balance(&playerInformation.health, &playerInformation.energy)
      // Error: conflicting access to properties of playerInformation”
      ```

   3. 同时满足以下条件不会发生内存安全问题

      - 只操作了**实例**的存储属性，没有操作计算属性和类属性

      - 结构体是一个局部变量，非全局变量

        ```swift
        func someFunction() {
            var oscar = Player(name: "Oscar", health: 10, energy: 10)
            balance(&oscar.health, &oscar.energy)  // OK
        }
        ```

      - 结构体没有被任何闭包捕获或者只被非逃逸闭包捕获

## Access Control

1. 默认的权限控制是 `internal`

2. 自定义类型

   1. 标记为 `private` 或者 `fileprivate`，它的成员默认也是对应的权限控制
   2. 标记为 `public`，它的成员默认是 `internal`，包括默认初始化方法
   3. 结构的默认的按成员初始化方法的权限为成员中最小的

3. 元组类型的权限控制取成员中小的

4. 方法的权限默认取参数和返回值类型中最小的，如果默认计算出的权限和上下文中默认的不一致，需要显式标记

5. 枚举类型的 `case` 的权限适合枚举的定义一致的

6. 嵌入类型

   1. 被嵌入到 `private` 或者 `fileprivate` 中时，默认取对应的权限
   2. 被嵌入到 `public` 或者 `internal` 中时，默认取 `internal`，如果要扩大需要显式标记

7. 子类不能将父类重写为更高权限的，可以将成员重写为比父类更高的权限

8. 协议类型的成员和协议的权限相同

9. `extension`

   默认情况下，`public` 和 `internal` 的 `extension` 为 `internal`，`private` 和 `fileprivate` 的 `extension` 为对应权限

10. `typealias` 的权限小于等于被 `alias` 的对象