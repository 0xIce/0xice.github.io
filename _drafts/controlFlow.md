```yaml
---
title: Swift 控制流
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

## Structures and Classes

### Compare

**common**

1. 定义属性
2. 定义方法
3. 定义subscripts
4. 定义构造方法
5. extended
6. 遵守协议

**diff( class才有)**

1. 继承
2. 



