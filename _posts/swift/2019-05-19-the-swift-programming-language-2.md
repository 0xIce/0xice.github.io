```yaml
---
title: 重读 Swift (5.0) 第二篇
excerpt: Control Flow、Closures、Structures and Classes、Properties、Subscripts
tags:
  - swift
toc: true
---
```

## Control Flow

### 1. stride

```swift
let minutes = 60
let minuteInterval = 5
for tickMark in stride(from: 0, to: minutes, by: minuteInterval) {
    // render the tick mark every 5 minutes (0, 5, 10, 15 ... 45, 50, 55)
}

let hours = 12
let hourInterval = 3
for tickMark in stride(from: 3, through: hours, by: hourInterval) {
    // render the tick mark every 3 hours (3, 6, 9, 12)
}
```

### 2. Switch

**Compound Cases/Value Bindings**

Compound  Cases 的 value binding 的变量名和类型必须一致

```swift
let stillAnotherPoint = (9, 0)
switch stillAnotherPoint {
case (let distance, 0), (0, let distance): // distance 的变量名和类型必须一致
    print("On an axis, \(distance) from the origin")
default:
    print("Not on an axis")
}
```



**Labeled Statements**

当有控制流嵌套的时候可以使用 `Label` 来标记控制流，调用流控制语句时可以指定 `Label`

```swift
gameLoop: while square != finalSquare {
    diceRoll += 1
    if diceRoll == 7 { diceRoll = 1 }
    switch square + diceRoll {
    case finalSquare:
        break gameLoop // 如果不指定 gameLoop 将会 break switch 语句
    case let newSquare where newSquare > finalSquare:
        continue gameLoop // switch 没有continue 语法，可以省略 gameLoop
    default:
        square += diceRoll
        square += board[square]
    }
}
print("Game over!")
```



## Closures

**closure 是引用类型**

**global and nested functions are special cases of closures**

- Global functions 是没有变量捕获的closure
- Nested functions 是可以有变量捕获的closure

## Structures and Classes

### 1. Compare

**common**

1. 定义属性
2. 定义方法
3. 定义subscripts
4. 定义构造方法
5. extended
6. 遵守协议

**diff( class 独有)**

1. 继承
2. type cast
3. 析构方法
4. 引用类型，一个对象可以有多个引用

**diff(structure 独有)**

1. 默认有一个`按成员初始化方法`，新增自定义初始化方法后失效，在extension中新增初始化方法不影响

2. 值类型，在赋值时拷贝 

   _实际是copy-on-write_

   > Collections defined by the standard library like arrays, dictionaries, and strings use an optimization to reduce the performance cost of copying. Instead of making a copy immediately, these collections share the memory where the elements are stored between the original instance and any copies. If one of the copies of the collection is modified, the elements are copied just before the modification. The behavior you see in your code is always as if a copy took place immediately.
   >

**structure is preferred**

## Properties

### 1. Lazy Stored Properties

lazy标记的属性，在第一次被访问的时候才会被初始化

在lazy属性没有被初始化之前多线程同时访问，属性可能会被多次初始化

### 2. Property Observers

- 如果在一个属性的didSet observer中给这个属性赋值，赋的新值会替换到刚刚赋的值

- 在init方法中对本类属性属性赋值不会触发observer，在调用super的init方法后对父类属性进行赋值会触发observer

- 如果observer的属性被当作inout参数传递进一个函数中，即使没有值的改变也会调用observer，因为当函数结束的时候，inout参数会被write back

  ```swift
  class Father {
    var fatherValue: Int {
      didSet {
        print("fatherValue: \(fatherValue)")
      }
    }
    
    init() {
      print("father init")
      fatherValue = 0
    }
  }
  
  class Son: Father {
    var sonValue: Int {
      didSet {
        print("sonValue: \(sonValue)")
      }
    }
    override init() {
      print("son init")
      sonValue = 0
      super.init()
      fatherValue = 1
      sonValue = 1
    }
  
    func echo(_ value: inout Int) {
      print(#function, ": \(value)")
    }
  }
  
  let father = Father()
  let son = Son()
  son.echo(&(son.sonValue))
  /**
  father init
  son init
  father init
  fatherValue: 1
  echo(_:) : 1
  sonValue: 1
  */
  ```

### 3. Global and Local Variables

#### 1. 定义

- global variables 是指在 function, method, closure, or type context 之外定义的变量
- local variables 是指在 function, method, or closure context 中定义的变量

#### 2. observer

global variables 和 local variables 也可以定义observer

#### 3. lazy

- global variables 是lazy计算的，不用显式标记为`lazy`
- local variables  是always 非lazy计算的

### 4. Type Property Syntax

1. 用`static`关键字定义
2. 对于 class 类型的计算型属性可以用 class 关键字以允许子类重写

## Subscripts

**class、 structure、 enum可以定义subscript**

### 1. 语法

```swift
subscript(index: Int) -> Int {
    get {
        // return an appropriate subscript value here
    }
    set(newValue) { // newValue变量名可以自定义，省略的时候使用newValue
        // perform a suitable setting action here
    }
}
```

### 2. Subscript Options

- 一个类型可以定义多个subscripts, 每个subscript可以定义多个多种类型的参数，调用时根据参数个数和类型确定具体哪个subscript，实现subscript的重载
- subscript可以有read-write和read-only属性，与计算型属性相同