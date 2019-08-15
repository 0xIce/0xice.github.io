---
title: Flutter 入门
excerpt: 使用 GitHub API  构建 Flutter 应用，包括环境搭建、package、自定义 widget、网络请求、主题设置
categories:
  - Flutter
tags:
  - Flutter
toc: true
---

## 环境搭建

安装步骤可以查看[这里](https://flutterchina.club/setup-macos)

本例开发环境为 Mac + VS Code

## 创建新项目

在 VS Code 选择 `View > Command Palette`，或者使用快捷键，Mac 下为 `Cmd-Shift-P`，Linux 和 Windows 下为` Ctrl-Shift-P`

输入 `Flutter: New Project ` 命令回车，之后输入项目名称创建新项目

在 VS Code 中左边的菜单中可以看到项目的结构，有 ios 和 android 目录分别应用于不同的平台，还有 lib 目录包含 main.dart，lib中的文件会应用于所有的平台

本例中只会修改 lib 目录下的文件

替换 main.dart 中的内容如下

```dart
import 'package:flutter/material.dart';

void main() => runApp(new GHFlutterApp());

class GHFlutterApp extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return new MaterialApp(
      title: 'GHFlutter',
      home: new Scaffold(
        appBar: new AppBar(
          title: new Text('GHFlutter'),
        ),
        body: new Center(
          child: new Text('GHFlutter'),
        ),
      ),
    );
  }
}
```

`main()` 方法使用 `=>` 操作符表示单行的方法， 本例中使用 `GHFlutterApp` 类来运行 App

App 本身使用一个 `StatelessWidget`，大多数 App 的入口都是 **widgets**，无论是 stateless 还是 stateful，你可以重写 widget 的 `build()` 方法创建自己的 widget，本例中使用 **MaterialApp** widget 提供的很多控件来实现页面设计

VS Code 中右下角可以选择运行的设备，现在选择 iPhone 的模拟器运行

选择完设备后，点击 `Debug -> Start Debugging` 开始运行，或者使用 `F5` 快捷键开始

## 热加载

flutter 支持热加载功能，可以不必重新运行就可以看到发生的改动

```dart
appBar: new AppBar(
  title: new Text('GHFlutter App'),
),
```

发生改动之后点击 VS Code 工具栏中的热加载按钮查看改动

## 引入文件

在 lib 目录下创建 *strings.dart* 新文件实现文案的国际化

在 main.dart 中引入创建的文件

```swift
import 'strings.dart';
```

更新 AppTitle

```dart
return new MaterialApp(
  title: Strings.appTitle,
```

更改其它使用 string 的地方

```dart
class GHFlutterApp extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return new MaterialApp(
      title: Strings.appTitle,
      home: new Scaffold(
        appBar: new AppBar(
          title: new Text(Strings.appTitle),
        ),
        body: new Center(
          child: new Text(Strings.appTitle),
        ),
      ),
    );
  }
}
```

## Widgets

基本上 Flutter App 中所有的元素都是 widget。Widget 被设计为不可变的(**Immutable**)来保持 UI 的轻量级

有两种基本的 widget 类型可以使用

1. **Stateless:** 只依赖自己的配置信息的 widget，比如在一个 image view 上的静态图片
2. **Stateful:** 需要维护动态信息的 widget，通过和 **State**  对象的交互来实现

在 Flutter App 中，不管是 Stateless 还是 Stateful 的 widget 在每一帧中都会重绘，区别在于 stateful 的 widget 会将配置代理给它们的 **State** 对象

在 main.dart 中添加代码创建新的 widget，新的 widget 继承 StatefulWidget 并重写 createState 方法来创建自己的 **State** 对象

```dart
class GHFlutter extends StatefulWidget {
  @override
  createState() => new GHFlutterState();
}
```

在 `GHFlutter` 类之上新增 `GHFlutterState`

自定义 widget 的主要任务就是重写 `build()` 方法，它会在 widget 渲染到屏幕上的时候被调用

```dart
class GHFlutterState extends State<GHFlutter> {
  @override
	Widget build(BuildContext context) {
  	return new Scaffold (
    	appBar: new AppBar(
      	title: new Text(Strings.appTitle),
    	),
    	body: new Text(Strings.appTitle),
  	);
	}
}
```

**Scaffold** 是一个 UI 视图的容器，表现为 UI 视图的根视图

替换 `GHFlutterApp` 中的实现，使用自定义的 widget 作为 home 属性而不是系统的

```dart
class GHFlutterApp extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return new MaterialApp(
      title: Strings.appTitle,
      home: new GHFlutter(),
    );
  }
}
```

## 网络请求

import 两个 package 来发送 http 请求和解析 JSON 数据

http 的 package 需要解决依赖，先在 `pubspec.yaml` 的 `dependencies` 中增加依赖项 `http: ^0.12.0`

```yaml
dependencies:
  flutter:
    sdk: flutter
  http: ^0.12.0
```

然后在 `main.dart` 中引入

```dart
import 'dart:convert';
import 'package:http/http.dart' as http;
```

Dart app 是单线程的，但是可以使用 *async/await* 模式来实现在其它线程执行代码和异步执行代码而不会阻塞 UI 线程

下面将发送异步的网络请求来获取 GitHub 的 team members，首先在 `GHFlutterState` 中新增一个空的 list 属性，另增一个属性保存 text style

```dart
var _members = [];

final _biggerFont = const TextStyle(fontSize: 18.0);
```

> 变量名前面的下划线来使变量为私有变量(**private**)

在 `GHFlutterState` 中新增一个方法来发送异步网络请求

在方法名后面使用 `async` 关键字表示该方法是异步的，在方法内部对 `http.get()` 使用 `await` 表示这个调用是阻塞的

在 HTTP 调用结束后，你向 `setState()` 方法中传入一个 callback，它将会在 UI 线程中以同步的方式调用，本例中在 callback 中解析 JSON 并将结果复制给 `_members` 属性

```dart
_loadData() async {
  String dataURL = "https://api.github.com/orgs/raywenderlich/members";
  http.Response response = await http.get(dataURL);
  setState(() {
    _members = json.decode(response.body);
  });
}
```

在  `GHFlutterState` 中重写 `initState()` 方法，在其中调用 `_loadData()` 方法

```dart
@override
void initState() {
  super.initState();

  _loadData();
}
```

## 使用 ListView

现在需要将 `_members` 属性中的元素展示到 UI 上，Flutter 提供 **ListView** widget 来展示列表。

**ListView** 类似于 Android 中的 **RecyclerView** 和 iOS 中的 **UITableView**，重复使用 view 来达到在用户滑动列表的时候获得一个顺滑的表现

在 `GHFlutterState` 中新增一个 `_buildRow()` 方法

```dart
Widget _buildRow(int i) {
  return new ListTile(
    title: new Text("${_members[i]["login"]}", style: _biggerFont)
  );
}
```

更新 `build` 方法让它的 body 是一个 **ListView.builder**

```dart
body: new ListView.builder(
  padding: const EdgeInsets.all(16.0),
  itemCount: _members.length,
  itemBuilder: (BuildContext context, int position) {
    return _buildRow(position);
  }),
```

## 添加 dividers

添加 divider 的方式为，删除原来的 padding， 返回 itemCount 设置为 _members length 的两倍，并在单数位置返回 **Divider** widget

```dart
body: new ListView.builder(
  itemCount: _members.length * 2,
  itemBuilder: (BuildContext context, int position) {
    if (position.isOdd) return new Divider();

    final index = position ~/ 2;

    return _buildRow(index);
  }),
```

如果仍然想要使用 padding，可以把 `_buildRow` 中的 `ListTile` 包装在 Padding widget 中

```dart
Widget _buildRow(int i) {
  return new Padding(
    padding: const EdgeInsets.all(16.0),
    child: new ListTile(
      title: new Text("${_members[i]["login"]}", style: _biggerFont)
    )
  );
}
```

现在 ListTile 变成了 Padding 的子 widget

## 封装模型

之前在 `_loadData()` 方法中获取到数据之后直接调用 `json.decode` 方法将数据转换成了 Dart 的 Map 类型，Map 类型类似于 Kotlin 中的 Map 或者 Swift 中的 Dictionary

现在增加自定义的模型类型

```dart
class Member {
  final String login;

  Member(this.login) {
    if (login == null) {
      throw new ArgumentError("login of Member cannot be null. "
          "Received: '$login'");
    }
  }
}
```

更新 `GHFlutterState` 中的 `_members` 类型

```dart
var _members = <Member>[];
```

> 更新之前默认是 dynamic 类型，即  <dynamic>[]

更新 `_buildRow` 方法中的调用

```dart
title: new Text("${_members[i].login}", style: _biggerFont)
```

更新 `_loadData` 中的 `setState` 的回调，实现 Map 转模型

```dart
setState(() {
  final membersJSON = JSON.decode(response.body);

  for (var memberJSON in membersJSON) {
    final member = new Member(memberJSON["login"]);
    _members.add(member);
  }
});
```

## 使用 NetworkImage 下载图片

下面通过显示用户头像的功能展示如何下载网络图片

更新 Member class

```dart
class Member {
  final String login;
  final String avatarUrl;

  Member(this.login, this.avatarUrl) {
    if (login == null) {
      throw new ArgumentError("login of Member cannot be null. "
          "Received: '$login'");
    }
    if (avatarUrl == null) {
      throw new ArgumentError("avatarUrl of Member cannot be null. "
          "Received: '$avatarUrl'");
    }
  }
}
```

更新 `_buildRow` 增加展示头像的代码，使用 NetworkImage 下载，使用 CircleAvatar widget 展示

```dart
Widget _buildRow(int i) {
  return new Padding(
    padding: const EdgeInsets.all(16.0),
    child: new ListTile(
      title: new Text("${_members[i].login}", style: _biggerFont),
      leading: new CircleAvatar(
        backgroundColor: Colors.green,
        backgroundImage: new NetworkImage(_members[i].avatarUrl)
      ),
    )
  );
}
```

使用 CircleAvatar 的时候使用了 leading 属性，这会让头像在 ListTile 中展示在 title 前面

同时使用 Colors 类设置头像的背景色

更新 `_loadData` 中创建 Member 的方法

```dart
final member = new Member(memberJSON["login"], memberJSON["avatar_url"]);
```

## 清理代码

目前所有的代码都是放在 main.dart 文件中，可以将其中新建的类型抽取到单独的文件中

在 lib 目录中新建源文件 **member.dart** 和 **ghflutter.dart**，将 Member 类移动到 member.dart 文件中，GH FlutterState 类和 GHFlutter 类移动到 ghflutter.dart 文件中

在 ghflutter.dart 中增加 import

```dart
import 'dart:convert';

import 'package:flutter/material.dart';
import 'package:http/http.dart' as http;

import 'member.dart';
import 'strings.dart';  
```

更新 main.dart，完整的内容如下

```dart
import 'package:flutter/material.dart';

import 'ghflutter.dart';
import 'strings.dart';

void main() => runApp(new GHFlutterApp());


class GHFlutterApp extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return new MaterialApp(
      title: Strings.appTitle,
      home: new GHFlutter(),
    );
  }
}  
```

## 添加主题

对 App 设置主题的方式很简单，在 MaterialApp 中增加一个 theme 属性就可以

```dart
class GHFlutterApp extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return new MaterialApp(
      title: Strings.appTitle,
      theme: new ThemeData(primaryColor: Colors.green.shade800), 
      home: new GHFlutter(),
    );
  }
}  
```

## Reference

1. [Demo](https://github.com/hotchner/github-flutter)