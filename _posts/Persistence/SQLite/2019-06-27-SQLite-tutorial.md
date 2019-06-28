---
title: SQLite 入门
excerpt: 创建连接数据库、建表、插入记录、更新记录、删除记录、查询、错误处理、Swift 化封装
categories:
  - Persistence
tags:
  - SQLite
toc: true
---

## Why SQLite

1. 集成在 iOS 系统中，不会影响包体积
2. 广泛使用和测试
3. 开源
4. 查询语言与主流数据库类似
5. 跨平台

## The C API

###  Opening a Connection

在进行进一步的操作之前，首先要建立一个数据库的连接

```swift
func openDatabase() -> OpaquePointer? {
  var db: OpaquePointer? = nil
  if sqlite3_open(part1DbPath, &db) == SQLITE_OK {
    print("Successfully opened connection to database at \(part1DbPath)")
    return db
  } else {
    print("Unable to open database. Verify that you created the directory described " +
      "in the Getting Started section.")
    PlaygroundPage.current.finishExecution()
  }
  
}
```

`Sqlite3_open` 方法会打开或者新建一个数据库文件。如果成功返回一个 `OpaquePointer` 类型，数据库的操作者需要持有这个 point 来和数据库进行交互。

> `OpaquePoint` 是一个 Swift 类型，用来表示不能直接用 Swift 类型直接表示的 C 语言指针

许多 SQLite 方法都会返回一个 `Int32` 类型的结果码。这些结果码几乎都有一个 SQLite 中定义好的常量表示，比如 `SQLITE_OK` 代表结果码 `0` ，结果码的列表可以在[这里](https://www.sqlite.org/rescode.html)找到。

### Creating a Table

```swift
let createTableString = """
CREATE TABLE Contact(
Id INT PRIMARY KEY NOT NULL,
Name CHAR(255));
"""

func createTable() {
  // 1
  var createTableStatement: OpaquePointer? = nil
  // 2
  if sqlite3_prepare_v2(db, createTableString, -1, &createTableStatement, nil) == SQLITE_OK {
    // 3
    if sqlite3_step(createTableStatement) == SQLITE_DONE {
      print("Contact table created.")
    } else {
      print("Contact table could not be created.")
    }
  } else {
    print("CREATE TABLE statement could not be prepared.")
  }
  // 4
  sqlite3_finalize(createTableStatement)
}
```

1. 创建一个后续步骤要使用的指针
2. 调用 `sqlite3_prepare_v2()` 方法将 SQL 语句编译成字节码并返回结果码 - 在执行一个随意写的 SQL 语句之前非常重要的步骤。更多信息查看[这里](https://www.sqlite.org/c3ref/prepare.html)。检查一下返回的结果码来确保 SQL 语句被正确编译。
3. 调用 `sqlite3_step()` 方法来执行编译好的 SQL 语句。可以调用一次或者多次
4. 最后必须调用 `sqlite3_finalize()` 方法，参数传入接收 SQL 语句编译结果的指针来删除它，防止资源泄露，当 编译结果指针被删除后，不可再使用它

### Insert

```swift
// 0
let insertStatementString = "INSERT INTO Contact (Id, Name) VALUES (?, ?);"

func insert() {
  var insertStatement: OpaquePointer? = nil

  // 1
  if sqlite3_prepare_v2(db, insertStatementString, -1, &insertStatement, nil) == SQLITE_OK {
    let id: Int32 = 1
    let name: NSString = "Ray"

    // 2
    sqlite3_bind_int(insertStatement, 1, id)
    // 3
    sqlite3_bind_text(insertStatement, 2, name.utf8String, -1, nil)

    // 4
    if sqlite3_step(insertStatement) == SQLITE_DONE {
      print("Successfully inserted row.")
    } else {
      print("Could not insert row.")
    }
  } else {
    print("INSERT statement could not be prepared.")
  }
  // 5
  sqlite3_finalize(insertStatement)
}
```

0. 插入的 SQL 语句，问号为占位符，在调用 `prepare` 方法进行 compile 后，可调用 `bind` 方法设置实际的值，compile 的结果可以多次使用，传入不同的值
1. compile
2. 替换问号占位符的位置为实际的值，`bind` 方法的第二个参数为问号的位置，位置从 1 开始，第三个参数为实际的值
3. 与第二步一样的作用，`bind_text` 增加了两个参数，更多信息查看[这里](https://www.sqlite.org/c3ref/bind_blob.html)
4. 执行语句
5. 释放，如果要插入多条记录，可以持有编译后的语句重用之

### Multiple Inserts

```swift
func multiInsert() {

  var insertStatement: OpaquePointer? = nil
  // 1
  let names: [NSString] = ["Ray", "Chris", "Martha", "Danielle"]

  if sqlite3_prepare_v2(db, insertStatementString, -1, &insertStatement, nil) == SQLITE_OK {

    // 2
    for (index, name) in names.enumerated() {
      // 3
      let id = Int32(index + 1)
      sqlite3_bind_int(insertStatement, 1, id)
      sqlite3_bind_text(insertStatement, 2, name.utf8String, -1, nil)

      if sqlite3_step(insertStatement) == SQLITE_DONE {
        print("Successfully inserted row.")
      } else {
        print("Could not insert row.")
      }
      // 4
      sqlite3_reset(insertStatement)
    }

    sqlite3_finalize(insertStatement)
  } else {
    print("INSERT statement could not be prepared.")
  }
}
```

> 在再次执行语句之前需要调用 `sqlite3_reset()` 方法重置状态

### Query

```swift
// 0
let queryStatementString = "SELECT * FROM Contact;"

func query() {
  var queryStatement: OpaquePointer? = nil
  // 1
  if sqlite3_prepare_v2(db, queryStatementString, -1, &queryStatement, nil) == SQLITE_OK {
    // 2
    if sqlite3_step(queryStatement) == SQLITE_ROW {
      // 3
      let id = sqlite3_column_int(queryStatement, 0)

      // 4
      let queryResultCol1 = sqlite3_column_text(queryStatement, 1)
      let name = String(cString: queryResultCol1!)

      // 5
      print("Query Result:")
      print("\(id) | \(name)")

    } else {
      print("Query returned no results")
    }
  } else {
    print("SELECT statement could not be prepared")
  }

  // 6
  sqlite3_finalize(queryStatement)
}
```

0. 查询语句
1. 编译语句
2. 检查执行的结果是否是 `SQLITE_ROW`， 代表返回一条记录
3. 获取第一个字段，`sqlite3_column_int` 的第二个参数为字段的索引值，从 0 开始
4. 获取第二个字段的值并转换为 Swift 类型
5. 打印结果
6. 结束

### Printing Every Row

```swift
func query() {
  var queryStatement: OpaquePointer? = nil
  if sqlite3_prepare_v2(db, queryStatementString, -1, &queryStatement, nil) == SQLITE_OK {

    while (sqlite3_step(queryStatement) == SQLITE_ROW) {
      let id = sqlite3_column_int(queryStatement, 0)
      let queryResultCol1 = sqlite3_column_text(queryStatement, 1)
      let name = String(cString: queryResultCol1!)
      print("Query Result:")
      print("\(id) | \(name)")
    }

  } else {
    print("SELECT statement could not be prepared")
  }
  sqlite3_finalize(queryStatement)
}
```

在 while 循环中获取所有的记录，当获取最后一条记录后，`sqlite3_step` 会返回 `SQLITE_DONE` 标识结束。

### Update

```swift
// 0
let updateStatementString = "UPDATE Contact SET Name = 'Chris' WHERE Id = 1;"

func update() {
  var updateStatement: OpaquePointer? = nil
  if sqlite3_prepare_v2(db, updateStatementString, -1, &updateStatement, nil) == SQLITE_OK {
    if sqlite3_step(updateStatement) == SQLITE_DONE {
      print("Successfully updated row.")
    } else {
      print("Could not update row.")
    }
  } else {
    print("UPDATE statement could not be prepared")
  }
  sqlite3_finalize(updateStatement)
}
```

0. 更新语句，更新一条的时候直接写死，否则应该用**问号占位**
1. 依次是 prepare、step、finalize

### Delete

```swift
let deleteStatementStirng = "DELETE FROM Contact WHERE Id = 1;"

func delete() {
  var deleteStatement: OpaquePointer? = nil
  if sqlite3_prepare_v2(db, deleteStatementStirng, -1, &deleteStatement, nil) == SQLITE_OK {
    if sqlite3_step(deleteStatement) == SQLITE_DONE {
      print("Successfully deleted row.")
    } else {
      print("Could not delete row.")
    }
  } else {
    print("DELETE statement could not be prepared")
  }
  
  sqlite3_finalize(deleteStatement)
}
```

相同的套路，准备 SQL 语句、prepare、step、finalize

### Handling Errors

合适的错误处理可以节约开发时间，返回更有意义的错误信息。

```swift
let malformedQueryString = "SELECT Stuff from Things WHERE Whatever;"

func prepareMalformedQuery() {
  var malformedStatement: OpaquePointer? = nil
  // 1
  if sqlite3_prepare_v2(db, malformedQueryString, -1, &malformedStatement, nil) == SQLITE_OK {
    print("This should not have happened.")
  } else {
    // 2
    let errorMessage = String.init(cString: sqlite3_errmsg(db))
    print("Query could not be prepared! \(errorMessage)")
  }
  
  // 3
  sqlite3_finalize(malformedStatement)
}

// 输出: Query could not be prepared! no such table: Things
```

调用 `sqlite3_errmsg(db)` 方法获取最近一次错误的文本描述

### Close the Database Connection

调用 `sqlite3_close(db)` 方法关闭连接，返回 `SQLITE_OK` 关闭成功。关闭需要的准备查看[这里](https://www.sqlite.org/c3ref/close.html)

## SQLite with Swift

### Wrap Errors

```swift
enum SQLiteError: Error {
  case OpenDatabase(message: String)
  case Prepare(message: String)
  case Step(message: String)
  case Bind(message: String)
}
```

封装错误类型，关联值记录错误信息

将数据库的操作封装为可以 throw error 的方法

### Wrap the Database Connection

原始的方法中使用到了 `OpaquePointer` 类型，可以做 Swift 化的封装

```swift
class SQLiteDatabase {
  fileprivate let dbPointer: OpaquePointer?

  fileprivate init(dbPointer: OpaquePointer?) {
    self.dbPointer = dbPointer
  }

  deinit {
    sqlite3_close(dbPointer)
  }
  
  fileprivate var errorMessage: String {
    if let errorPointer = sqlite3_errmsg(dbPointer) {
      let errorMessage = String(cString: errorPointer)
      return errorMessage
    } else {
      return "No error message provided from sqlite."
    }
  }
}

let db: SQLiteDatabase
do {
  db = try SQLiteDatabase.open(path: part2DbPath)
  print("Successfully opened connection to database.")
} catch SQLiteError.OpenDatabase(let message) {
  print("Unable to open database: \(message)")
  PlaygroundPage.current.finishExecution()
}
```

### Wrap the Prepare Call

```swift
extension SQLiteDatabase {
  func prepareStatement(sql: String) throws -> OpaquePointer? {
    var statement: OpaquePointer? = nil
    guard sqlite3_prepare_v2(dbPointer, sql, -1, &statement, nil) == SQLITE_OK else {
      throw SQLiteError.Prepare(message: errorMessage)
    }

    return statement
  }
}
```

### Wrap the Table Creation

```swift
struct Contact {
  let id: Int32
  let name: NSString
}
```

```swift
protocol SQLTable {
  static var createStatement: String { get }
}
```

```swift
extension Contact: SQLTable {
  static var createStatement: String {
    return """
    CREATE TABLE Contact(
      Id INT PRIMARY KEY NOT NULL,
      Name CHAR(255)
    );
    """
  }
}
```

```swift
extension SQLiteDatabase {
  func createTable(table: SQLTable.Type) throws {
    // 1
    let createTableStatement = try prepareStatement(sql: table.createStatement)
    // 2
    defer {
      sqlite3_finalize(createTableStatement)
    }
    // 3
    guard sqlite3_step(createTableStatement) == SQLITE_DONE else {
      throw SQLiteError.Step(message: errorMessage)
    }
    print("\(table) table created.")
  }
}
```

> 参数是 SQLTable.Type 而不是 SQLTable 是因为 createStatement 为 static 修饰

```swift
do {
  try db.createTable(table: Contact.self)
} catch {
  print(db.errorMessage)
}
```

### Wrap Insertions

```swift
extension SQLiteDatabase {
  func insertContact(contact: Contact) throws {
    let insertSql = "INSERT INTO Contact (Id, Name) VALUES (?, ?);"
    let insertStatement = try prepareStatement(sql: insertSql)
    defer {
      sqlite3_finalize(insertStatement)
    }
    
    let name: NSString = contact.name as NSString
    guard sqlite3_bind_int(insertStatement, 1, contact.id) == SQLITE_OK,
      sqlite3_bind_text(insertStatement, 2, name.utf8String, -1, nil) == SQLITE_OK  else {
        throw SQLiteError.Bind(message: errorMessage)
    }
    
    guard sqlite3_step(insertStatement) == SQLITE_DONE else {
      throw SQLiteError.Step(message: errorMessage)
    }
    
    print("Successfully inserted row.")
  }
}

do {
  try db.insertContact(contact: Contact(id: 1, name: "Ray"))
} catch {
  print(db.errorMessage)
}
```

### Wrap Reads

```swift
extension SQLiteDatabase {
  func contact(id: Int32) -> Contact? {
    let querySql = "SELECT * FROM Contact WHERE Id = ?;"
    
    guard let queryStatement = try? prepareStatement(sql: querySql) else {
      return nil
    }
    
    defer {
      sqlite3_finalize(queryStatement)
    }
    
    guard sqlite3_bind_int(queryStatement, 1, id) == SQLITE_OK else {
      return nil
    }
    
    guard sqlite3_step(queryStatement) == SQLITE_ROW else {
      return nil
    }
    
    let id = sqlite3_column_int(queryStatement, 0)
    
    let queryResultCol1 = sqlite3_column_text(queryStatement, 1)
    let name = String(cString: queryResultCol1!)
    
    return Contact(id: id, name: name)
  }
}
```

## 小结

1. SQLite 的使用过程，打开数据库、写 SQL 语句、编译语句、[填充语句中的占位值]、执行、[多次使用重置语句，填充占位置，执行]、结束语句、关闭数据库
2. SQLITE_ROW 用在查询结果的记录，SQLITE_DONE 用于增删改查的结束，SQLITE_OK 用在一般语句的结果判断

## Reference

1. 不完整翻译自 [raywenderlich](https://www.raywenderlich.com/385-sqlite-with-swift-tutorial-getting-started)
2. [demo](https://github.com/hotchner/Demos/tree/master/SQLiteTutorial)
3. [SQLite.swift](https://github.com/stephencelis/SQLite.swift)