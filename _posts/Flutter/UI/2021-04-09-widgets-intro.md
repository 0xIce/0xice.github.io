---
title: Flutter UI 之 Widgets 介绍
excerpt: Widgets 介绍
categories:
  - Flutter
tags:
  - Flutter
toc: true
---

## Hello World
创建一个 Flutter 应用时，可以调用 runApp() 方法，其中可以传入一个 widget 来作为 root widget，root widget 会被强制铺满屏幕

这个过程中没有 vc 的概念，iOS中的 rootVC 的功能也包含中 root widget 中

创建 widget 的主要工作是实现它的 build 方法，build 方法根据其它较低级别的 widget 来描述这个 widget，框架会逐一构建这些 widget，直到最底层的描述 widget 几何形状的 RenderObject

## 基础 widgets

| Widget | 用途   |
| ------ | ------ |
| Text | 用来在应用内创建带样式的文本 |
| Row,Column|flex widgets, 可以在水平和垂直方向创建灵活的布局，它是基于 web 的 flexbox 布局模型设计的 |
| Stack | 按照绘制顺序将widget堆叠在一起，可以使用 Positioned widget 作为 stack 的子 widget，以机相对于 Stack 的上，右，下，左来定位它们。 Stack 是基于 Web 中的绝对位置布局模型设计的 |
| Container | 可以用来创建一个可见的矩形元素，可以使用 BoxDecooratioon 来进行装饰，如背景、边框、阴影等。Container 还可以设置外边距、内边距和尺寸的约束条件等，另外，Container 可以使用矩阵在三维空间进行转换 |

## 使用 Material 组件
Material 组件提供了有用的 widget，其中包括 Navigator、AppBar、Scaffold

使用 Material 组件就是在 runApp() 方法中，返回 Material 组件的 widget，将其作为 app 的 root widget

Scaffold widget 将许多不同的 widget 作为命名参数，每个 widget 都放在了 Scofford 布局中的合适位置。同样的，AppBarr widget 允许我们给 leading、title widget 的 actions 传递 widget。

## 处理手势
使用 GestureDetector 可以为 widget 增加手势，类似于 iOS 中的gestureRecgnizer，但是 iOS 中手势是添加到控件上，Flutter 中的 GestureDetector 是作为父 widget 来为子 widget 增加手势支持，它支持一个 onTap 参数，参数类型为 block，可以在里面写点击操作的处理，还接收一个 child 参数来添加真正要展示的 widget

## 根据用户输入改变 widget
Flutter 中的 widget 分为无状态的和有状态的，无状态的 widget 继承自 StatelessWidgets， 有状态的 widget 继承自 StatefulWidgets。
StatefulWdgets 是一特殊的 widget，它会生成 State 对象用于保存状态。

```
class Counter extends StatefulWidget {
  @override
  _CounterState createState() => _CounterState();
}

class _CounterState extends State<Counter> {
  int _counter = 0;

  void _increment() {
    setState(() {
      _counter++;
    });
  }

  @override
  Widget build(BuildContext context) {
    return Row(
      mainAxisAlignment: MainAxisAlignment.center,
      children: <Widget>[
        ElevatedButton(
          onPressed: _increment,
          child: Text('Increment'),
        ),
        SizedBox(width: 16),
        Text('Count: $_counter'),
      ],
    );
  }
}
```
创建 StatefulWidget 的时候，widget 的创建都会委托给对应的 State 对象去处理，State 对象中存储一些状态信息，在需要更新状态的时候调用 `setState()` 方法来实现状态及 widget 的更新

使用流程是：
1. 声明 widget 继承自 StatefulWidget
2. 实现 `createState()` 方法创建一个对应的 State 对象
3. 声明 State 类型继承自 State<Widget名字> ，state 类型一般为 Widet 名字后面加 State 来成对出现，同时在前面加下划线来表示私有
4. state 类中实现 build 方法，与 statelessWidget 中的 build 方法功能一致
5. state 类中将 widget 的状态记录为属性，在需要更新 widget 状态的时候调用 `setState()` 方法更新属性，之后 Flutter 会自动更新 widget 的状态
```
setState(() {
    _counter++;
});
```

为什么 StatefulWidget 和 State 是独立的对象？因为在 Flutter 中这两种类型的对象有不同的生命周期，Wiget 是临时对象，用于构造应用当前状态的显示，而 State 对象在调用 `build()` 之前是持久的，以此来存储信息