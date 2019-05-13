# Swift 性能优化

## 1. 使用struct 而不是使用class

- 引用语义
- 值语义

## 2. 方法的静态分发和动态分发

### 1. Static Dispatch

- 在编译阶段确定了方法的调用， 没有了方法的查找过程，提高效率

- 编译器有可能进行方法内联进行进一步优化

### 2. Dynamic Dispatch (Virtual-Table Dispatch/V-Table Dispatch)

- 类默认用Dynamic Dispatch
- 可以手动标记为final取消动态派发，表示没有子类

## 3. 面向协议编程

1. The Protocol Witness Table (PWT)
2. The Existential Container
3. The Value Witness Table (VWT)

