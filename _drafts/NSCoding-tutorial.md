---

title: NSCoding 入门
excerpt: NSCoding、FileManager、NSSecureCoding、归档解档、对象存取、图片存取
tags:
  - Persistence
  - NSCoding
toc: true
---

## 实现 NSCoding 协议

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

## 对象的存取

> 出于表现效率考虑，不应该一次性加载所有的数据

### 添加初始化方法

```swift
var docPath: URL?
  
init(docPath: URL) {
  super.init()
  self.docPath = docPath    
}
```

> 应该在第一次访问的时候把数据加载到内存中，而不是在初始化的时候就加载

当创建一个全新的动物的时 path 将为 `nil`，因为还没有创建对应的文件。下面就来添加代码实现来保证当新的动物被创建的时候 path 被正确设置。

### 添加簿记代码

首先增加枚举记录编解码的 key

```swift
enum Keys: String {
  case dataFile = "Data.plist"
  case thumbImageFile = "thumbImage.png"
  case fullImageFile = "fullImage.png"
}
```

替换获取图片的 `getter`

```swift
get {
  if _thumbImage != nil { return _thumbImage }
  if docPath == nil { return nil }

  let thumbImageURL = docPath!.appendingPathComponent(Keys.thumbImageFile.rawValue)
  guard let imageData = try? Data(contentsOf: thumbImageURL) else { return nil }
  _thumbImage = UIImage(data: imageData)
  return _thumbImage
}
```

为了将不同的动物保存在各自的目录中，创建一个帮助类来提供下一个可用的目录来存储动物的文档

```swift
class ScaryCreatureDatabase: NSObject {
  class func nextScaryCreatureDocPath() -> URL? {
    return nil
  }
}
```

在 **ScaryCreatureDoc.swift** 中新增一个方法

```swift
func createDataPath() throws {
  guard docPath == nil else { return }

  docPath = ScaryCreatureDatabase.nextScaryCreatureDocPath()
  try FileManager.default.createDirectory(at: docPath!,
                                          withIntermediateDirectories: true,
                                          attributes: nil)
}
```

### 保存数据

接下来新增保存数据的逻辑。在 `createDataPath` 方法后面新增一个方法

```swift
func saveData() {
  guard let data = data else { return }
    
  do {
    try createDataPath()
  } catch {
    print("Couldn't create save folder. " + error.localizedDescription)
    return
  }
    
  let dataURL = docPath!.appendingPathComponent(Keys.dataFile.rawValue)
    
  let codedData = try! NSKeyedArchiver.archivedData(withRootObject: data, 
                                                    requiringSecureCoding: false)
  do {
    try codedData.write(to: dataURL)
  } catch {
    print("Couldn't write to save file: " + error.localizedDescription)
  }
}
```

1. 先准备要保存的 path
2. 调用 `NSKeyedArchiver.archivedData(withRootObject:requiringSecureCoding:)` 对数据进行归档， 第二个参数传 `false`，因为现在实现的是 `NSCoding` 而不是 `NSSecureCoding`
3. 归档后的对象调用 `write(to:)` 方法写入硬盘

### 加载数据

> 在使用到信息的时候从硬盘加载到内存而不是在对象初始化的时候，可以优化应用启动时间

数据 `getter` 方法中增加从硬盘读取的逻辑，与 `saveData` 方法相反

```swift
get {
  // 1
  if _data != nil { return _data }
  
  // 2
  let dataURL = docPath!.appendingPathComponent(Keys.dataFile.rawValue)
  guard let codedData = try? Data(contentsOf: dataURL) else { return nil }
  
  // 3
  _data = try! NSKeyedUnarchiver.unarchiveTopLevelObjectWithData(codedData) as?
      ScaryCreatureData
  
  return _data
}
```

### 删除数据

```swift
func deleteDoc() {
  if let docPath = docPath {
    do {
      try FileManager.default.removeItem(at: docPath)
    }catch {
      print("Error Deleting Folder. " + error.localizedDescription)
    }
  }
}
```

直接删除动物所在的目录，目录中存放了动物照片和 title 、rating 数据

## 完成 ScaryCreatureDatabase

之前添加了一个空的方法准备用来获取下一个可用的存储路径来存储新创建的动物，它的另外一个任务是加载所有之前已经存储的动物

首先添加一个帮助方法来创建并返回存储所有动物文件的目录

```swift
static let privateDocsDir: URL = {
  let paths = FileManager.default.urls(for: .documentDirectory, in: .userDomainMask)
  
  let documentsDirectoryURL = paths.first!.appendingPathComponent("PrivateDocuments")
  
  do {
    try FileManager.default.createDirectory(at: documentsDirectoryURL,
                                            withIntermediateDirectories: true,
                                            attributes: nil)
  } catch {
    print("Couldn't create directory")
  }
  return documentsDirectoryURL
}()
```

添加获取已经存储的所有动物的方法

```swift
class func loadScaryCreatureDocs() -> [ScaryCreatureDoc] {
  guard let files = try? FileManager.default.contentsOfDirectory(
    at: privateDocsDir,
    includingPropertiesForKeys: nil,
    options: .skipsHiddenFiles) else { return [] }
  
  return files
    .filter { $0.pathExtension == "scarycreature" }
    .map { ScaryCreatureDoc(docPath: $0) }
}
```

实现之前用 `return nil` 实现的 `nextScaryCreatureDocPath()` 方法

```swift
class func nextScaryCreatureDocPath() -> URL? {
    guard let files = try? FileManager.default.contentsOfDirectory(at: privateDocsDir,
                                                              includingPropertiesForKeys: nil,
                                                              options: .skipsHiddenFiles) else {
                                                                return nil
    }
    
    var maxNumber = 0
    
    files.forEach {
      if $0.pathExtension == "scarycreature" {
        let fileName = $0.deletingPathExtension().lastPathComponent
        maxNumber = max(maxNumber, Int(fileName) ?? 0)
      }
    }
    
    return privateDocsDir.appendingPathComponent("\(maxNumber + 1).scarycreature", isDirectory: true)
  }
```

> 对 files 做 forEach 而不是直接取 count，是因为有可能删除了中间的目录，所以目录名不连续

在 cell 删除的时候也删除硬盘中的存储，在 `tableView(_:commit:forRowAt:)` 方法中实现

```swift
if editingStyle == .delete {
  let creatureToDelete = creatures.remove(at: indexPath.row)
  creatureToDelete.deleteDoc()
  tableView.deleteRows(at: [indexPath], with: .fade)
}
```

在详情页修改动物之后保存到硬盘， 在`rateViewRatingDidChange(rateView:newRating:)` 和 `titleFieldTextChanged(_:)` 方法中调用

```swift
detailItem?.saveData()
```

## 小结

1. `FileManager` 创建目录和删除目录，`NSKeyedArchiver` 将对象类型归档为 `Data` 类型后写入硬盘，`NSKeyedUnarchiver` 将从硬盘读取的 `Data` 类型解档为对对应的对象类型
2. 归档后的数据写入硬盘时，path 的目录一定要存在，文件可以不存在

## 参考

1. 不完整翻译自 [raywenderlich](https://www.raywenderlich.com/6733-nscoding-tutorial-for-ios-how-to-permanently-save-app-data)
2. [demo](https://github.com/hotchner/Demos/tree/master/ScaryCreatures)