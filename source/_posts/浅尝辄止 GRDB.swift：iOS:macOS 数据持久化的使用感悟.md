---
title: 浅尝辄止 GRDB.swift：iOS/macOS 数据持久化的使用感悟
date: 2026-04-03 14:00:00
tags:
  - iOS
  - Swift
  - GRDB
  - SQLite
categories:
  - iOS
---

# 浅尝辄止 GRDB.swift：iOS/macOS 数据持久化的使用感悟

> GRDB.swift 是一个基于 SQLite 的 Swift 数据库工具包，专注于应用开发体验。本文将从核心概念、关键类、使用方法、最佳实践等维度全面介绍 GRDB。

---

## 目录

- [为什么选择 GRDB](#为什么选择-grdb)
- [快速开始](#快速开始)
- [核心协议与关键类](#核心协议与关键类)
- [数据库连接管理](#数据库连接管理)
- [数据模型定义](#数据模型定义)
- [数据库迁移](#数据库迁移)
- [CRUD 操作](#crud-操作)
- [查询构建](#查询构建)
- [关联关系（Associations）](#关联关系associations)
- [响应式数据观察（ValueObservation）](#响应式数据观察valueobservation)
- [全文搜索（FTS5）](#全文搜索fts5)
- [与 Core Data / Realm 的对比](#与-core-data--realm-的对比)
- [最佳实践](#最佳实践)
- [总结](#总结)

---

## 为什么选择 GRDB

在 iOS/macOS 数据持久化领域，开发者通常面临以下选择：

| 方案 | 优势 | 劣势 |
|---|---|---|
| **Core Data** | Apple 原生、与 SwiftUI 深度集成 | 学习曲线陡峭、性能开销大、调试困难 |
| **Realm** | API 简洁、实时同步 | 内存占用高、闭源、版本迁移复杂 |
| **FMDB** | 轻量、成熟 | 缺乏类型安全、API 偏 Objective-C 风格 |
| **SQLite.swift** | 类型安全、轻量 | 功能相对基础、缺少迁移和响应式 |
| **GRDB.swift** | **类型安全 + 功能完整 + 高性能 + 纯 Swift** | 学习成本略高于 SQLite.swift |

**GRDB 的核心优势：**

1. **纯 Swift 设计** — 完全契合 Swift 的类型系统和惯用范式
2. **高性能** — 复杂查询性能比 Realm 快约 4.5 倍，批量插入快 2.3 倍
3. **类型安全** — 通过 `FetchableRecord`、`PersistableRecord` 协议和 `Column` 泛型实现编译时检查
4. **响应式编程** — 内置 `ValueObservation`，原生支持 Combine
5. **数据库迁移** — 内置 `DatabaseMigrator`，支持增量式 Schema 变更
6. **FTS5 全文搜索** — 一等公民支持，无需手写 SQL
7. **轻量无依赖** — 基于 SQLite C API，无需额外运行时
8. **活跃维护** — 自 2015 年起持续维护，社区活跃

---

## 快速开始

### 安装

**Swift Package Manager:**

```swift
dependencies: [
    .package(url: "https://github.com/groue/GRDB.swift.git", from: "7.0.0")
]
```

**CocoaPods:**

```ruby
pod 'GRDB.swift'
```

---

## 核心协议与关键类

GRDB 的 API 围绕一组核心协议和类构建，理解它们是高效使用 GRDB 的前提：

### 协议（Protocols）

| 协议 | 作用 | 说明 |
|---|---|---|
| `FetchableRecord` | 从数据库行读取数据 | 定义如何将查询结果映射为 Swift 类型 |
| `PersistableRecord` | 将数据写入数据库 | 定义如何将 Swift 类型持久化 |
| `MutablePersistableRecord` | 可变持久化记录 | 支持插入后自增 ID 回写等场景 |
| `TableRecord` | 表记录 | 声明表名，提供查询入口（如 FTS5 的 `.matching()`） |
| `DatabaseValueConvertible` | 数据库值转换 | 自定义类型与 SQLite 值的双向转换 |

### 类（Classes）

| 类 | 作用 | 说明 |
|---|---|---|
| `DatabasePool` | 连接池（推荐） | 支持并发读写，多读连接 + 单写连接 |
| `DatabaseQueue` | 串行队列 | 适用于需要严格串行化访问的场景 |
| `DatabaseMigrator` | 数据库迁移 | 管理增量式 Schema 变更 |
| `ValueObservation` | 响应式观察 | 追踪数据库变化，自动触发重新查询 |
| `Configuration` | 配置 | 外键约束、日志、WAL 模式等 |
| `DatabaseError` | 错误类型 | 封装 SQLite 错误码和信息 |

### 辅助类型

| 类型 | 作用 |
|---|---|
| `Column` | 类型安全的列引用，支持链式查询 |
| `ForeignKey` | 外键定义，用于关联查询 |
| `FTS5Pattern` | FTS5 搜索模式 |
| `SQLLiteral` | 安全的 SQL 片段构建 |

---

## 数据库连接管理

### DatabasePool vs DatabaseQueue

```swift
import GRDB

// 推荐：DatabasePool 支持并发读，适合大多数应用场景
var config = Configuration()
config.foreignKeysEnabled = true  // 启用外键约束
let dbPool = try DatabasePool(path: "/path/to/db.sqlite", configuration: config)

// 替代方案：DatabaseQueue 串行访问
let dbQueue = try DatabaseQueue(path: "/path/to/db.sqlite", configuration: config)
```

**选择建议：**
- 优先使用 `DatabasePool` — 读操作可以并发执行，性能更好
- 仅在需要严格串行化时使用 `DatabaseQueue`

### 读写操作

```swift
// 读操作
let users = try dbPool.read { db in
    try User.fetchAll(db)
}

// 写操作
try dbPool.write { db in
    try user.insert(db)
}

// 批量写入（整个闭包自动包裹在事务中，非常方便）
try dbPool.write { db in
    for todo in todos {
        try todo.insert(db)
    }
}
```

> **提示：** `DatabasePool.write { }` 闭包默认就是一个事务，中途抛出异常会自动回滚，无需手动管理。

---

## 数据模型定义

GRDB 推荐使用 **struct + Codable** 模式定义模型。只要你的 struct 遵循 `Codable`，就能极简地接入 GRDB：

### 最简模型

```swift
import GRDB

struct User: Codable, FetchableRecord, PersistableRecord, Identifiable {
    var id: Int64
    var name: String
    var email: String
    var createdAt: Date

    static let databaseTableName = "users"
}
```

就这样，一个可用的 GRDB 模型就定义好了。`Codable` 负责自动编解码，`FetchableRecord` 支持查询，`PersistableRecord` 支持写入。

### 完整模型（带列名映射）

当 Swift 属性名（camelCase）和数据库列名（snake_case）不一致时，用 `CodingKeys` 映射：

```swift
struct Todo: Codable, FetchableRecord, PersistableRecord, Identifiable, Sendable {
    var id: String
    var title: String
    var isCompleted: Bool
    var createdAt: Date

    static let databaseTableName = "todos"

    // Swift 属性名 → 数据库列名映射
    enum CodingKeys: String, CodingKey {
        case id, title
        case isCompleted = "is_completed"
        case createdAt = "created_at"
    }

    // 类型安全的列引用，用于查询构建
    enum Columns {
        static let id = Column(CodingKeys.id)
        static let title = Column(CodingKeys.title)
        static let isCompleted = Column(CodingKeys.isCompleted)
        static let createdAt = Column(CodingKeys.createdAt)
    }
}
```

**三个组件各司其职：**

| 组件 | 职责 |
|---|---|
| `CodingKeys` | 属性名 ↔ 列名映射，Codable 自动使用 |
| `Columns` | 为查询提供类型安全引用，编译时检查列名 |
| `databaseTableName` | 声明对应的 SQLite 表名 |

---

## 数据库迁移

`DatabaseMigrator` 是 GRDB 的迁移系统，支持增量式 Schema 变更。每个迁移有唯一标识符，只会执行一次：

```swift
var migrator = DatabaseMigrator()

// v1: 创建初始表结构
migrator.registerMigration("v1_create_tables") { db in
    try db.create(table: "todos") { t in
        t.column("id", .text).primaryKey()
        t.column("title", .text).notNull()
        t.column("is_completed", .integer).notNull().defaults(to: 0)
        t.column("created_at", .datetime).notNull().defaults(to: Date())
    }

    try db.create(table: "tags") { t in
        t.column("id", .text).primaryKey()
        t.column("name", .text).notNull()
        t.column("color", .text)
    }

    // 外键 + 级联删除
    try db.create(table: "todo_tags") { t in
        t.column("todo_id", .text).notNull()
            .references("todos", onDelete: .cascade)
        t.column("tag_id", .text).notNull()
            .references("tags", onDelete: .cascade)
        t.primaryKey(["todo_id", "tag_id"])
    }
}

// v2: 增量添加新字段
migrator.registerMigration("v2_add_priority") { db in
    // 防御性检查：避免重复迁移导致崩溃
    let columns = try db.columns(in: "todos")
    if !columns.contains(where: { $0.name == "priority" }) {
        try db.alter(table: "todos") { t in
            t.add(column: "priority", .integer).defaults(to: 0)
        }
    }
}

// 执行迁移（自动执行所有尚未执行的版本）
try migrator.migrate(dbPool)
```

**最佳实践：**
- 每个迁移使用唯一标识符（如 `"v1_create_tables"`, `"v2_add_priority"`）
- 迁移中加入防御性检查，避免重复执行崩溃
- 新增列使用 `defaults(to:)` 设置默认值，保证旧数据兼容
- 合理使用 `.references(..., onDelete: .cascade)` 自动清理关联数据

---

## CRUD 操作

### 插入

```swift
let todo = Todo(id: UUID().uuidString, title: "学习 GRDB", isCompleted: false, createdAt: Date())

// 基本插入
try dbPool.write { db in
    try todo.insert(db)
}

// 冲突时忽略（适合幂等写入，比如初始种子数据）
try todo.insert(db, onConflict: .ignore)

// Upsert（INSERT OR REPLACE，存在则更新）
try todo.save(db)
```

### 查询

```swift
// 按 ID 查询单条
let todo: Todo? = try dbPool.read { db in
    try Todo.fetchOne(db, key: id)
}

// 条件查询 + 排序
let activeTodos: [Todo] = try dbPool.read { db in
    try Todo
        .filter(Todo.Columns.isCompleted == false)
        .order(Todo.Columns.createdAt.desc)
        .fetchAll(db)
}

// 计数
let total = try dbPool.read { db in
    try Todo.fetchCount(db)
}

// 聚合查询（如获取最大排序序号）
let maxOrder: Int? = try dbPool.read { db in
    try Tag.select(max(Tag.Columns.sortOrder))
        .asRequest(of: Int?.self)
        .fetchOne(db)
}
```

### 更新

```swift
// 更新单个对象
try dbPool.write { db in
    var todo = try Todo.fetchOne(db, key: id)!
    todo.isCompleted = true
    try todo.update(db)
}

// 批量更新（高效：一条 SQL 搞定）
try dbPool.write { db in
    try Todo
        .filter(ids.contains(Todo.Columns.id))
        .updateAll(db, [
            Todo.Columns.isCompleted.set(to: true)
        ])
}
```

### 删除

```swift
// 删除单条
try dbPool.write { db in
    try Todo.deleteOne(db, key: id)
}

// 条件批量删除
try dbPool.write { db in
    try Todo.filter(Todo.Columns.isCompleted == true).deleteAll(db)
}
```

---

## 查询构建

GRDB 提供了强大的类型安全查询接口，基于 `Column` 泛型，告别字符串拼接 SQL：

### 常用查询模式

```swift
// WHERE + ORDER BY
let results = try Todo
    .filter(Todo.Columns.isCompleted == false)
    .order(Todo.Columns.createdAt.desc)
    .fetchAll(db)

// 多条件组合
let results = try Todo
    .filter(Todo.Columns.isCompleted == false)
    .filter(Todo.Columns.createdAt > yesterday)
    .order(Todo.Columns.createdAt.desc)
    .fetchAll(db)

// LIKE 模糊查询
let results = try Todo
    .filter(Todo.Columns.title.like("%GRDB%"))
    .fetchAll(db)

// IN 查询
let results = try Todo
    .filter([id1, id2, id3].contains(Todo.Columns.id))
    .fetchAll(db)
```

### 分页查询

```swift
// 第 3 页，每页 20 条
let page = try Todo
    .order(Todo.Columns.createdAt.desc)
    .limit(20, offset: 40)
    .fetchAll(db)
```

> 相比手写 `SELECT * FROM todos LIMIT 20 OFFSET 40`，类型安全查询能在编译期发现列名拼写错误，代码也更易读。

---

## 关联关系（Associations）

GRDB 提供了原生的关联系统，支持一对多和多对多关系。以经典的「文章-标签」多对多关系为例：

### 定义关联

```swift
// 标签
struct Tag: Codable, FetchableRecord, PersistableRecord, Identifiable {
    var id: String
    var name: String
    var color: String?
    static let databaseTableName = "tags"
}

// 中间表
struct PostTag: Codable, FetchableRecord, PersistableRecord {
    var postId: String
    var tagId: String
    static let databaseTableName = "post_tags"

    // 声明外键
    static let post = belongsTo(Post.self, using: ForeignKey([Columns.postId.name]))
    static let tag = belongsTo(Tag.self, using: ForeignKey([Columns.tagId.name]))
}

// 文章侧扩展
extension Post {
    static let postTags = hasMany(PostTag.self, using: ForeignKey([PostTag.Columns.postId.name]))
    static let tags = hasMany(Tag.self, through: postTags, using: PostTag.tag)
}

// 标签侧扩展
extension Tag {
    static let postTags = hasMany(PostTag.self, using: ForeignKey([PostTag.Columns.tagId.name]))
    static let posts = hasMany(Post.self, through: postTags, using: PostTag.post)
}
```

### 使用关联查询

```swift
// 查找某个标签下的所有文章（Join 过滤）
let posts = try Post
    .joining(required: Post.tags.filter(Tag.Columns.id == tagId))
    .fetchAll(db)

// 预加载关联数据（Eager Loading，避免 N+1 查询）
struct PostInfo: Decodable, FetchableRecord {
    var post: Post
    var tag: Tag
}

let results = try PostTag
    .including(required: PostTag.tag)
    .including(required: PostTag.post)
    .asRequest(of: PostInfo.self)
    .fetchAll(db)
```

---

## 响应式数据观察（ValueObservation）

`ValueObservation` 是 GRDB 最强大的特性之一 — 它能自动追踪查询依赖的表，当数据发生变化时自动重新执行查询，真正实现**数据驱动 UI**。

### 基本用法

```swift
import Combine

// 定义观察器
let observation = ValueObservation.tracking { db in
    try Todo
        .filter(Todo.Columns.isCompleted == false)
        .order(Todo.Columns.createdAt.desc)
        .fetchAll(db)
}

// 转为 Combine Publisher
let cancellable = observation
    .publisher(in: dbPool, scheduling: .immediate)
    .sink(
        receiveCompletion: { print("完成: \($0)") },
        receiveValue: { todos in
            // 每次数据库中 todos 表变化，这里会自动收到最新数据
            print("当前待办: \(todos.map(\.title))")
        }
    )
```

### 在 SwiftUI 中使用

```swift
// ViewModel / Store 中订阅
@MainActor
class TodoStore: ObservableObject {
    @Published var todos: [Todo] = []
    private var cancellables = Set<AnyCancellable>()

    init(dbPool: DatabasePool) {
        ValueObservation.tracking { db in
            try Todo.order(Todo.Columns.createdAt.desc).fetchAll(db)
        }
        .publisher(in: dbPool, scheduling: .immediate)
        .receive(on: RunLoop.main)
        .sink { [weak self] todos in
            self?.todos = todos  // 自动触发 SwiftUI 视图刷新
        }
        .store(in: &cancellables)
    }
}

// SwiftUI 视图
struct TodoListView: View {
    @StateObject private var store: TodoStore

    var body: some View {
        List(store.todos) { todo in
            Text(todo.title)
        }
        // 不需要手动刷新，数据变化时列表自动更新
    }
}
```

**核心机制：** `ValueObservation.tracking` 会自动分析闭包中访问了哪些表。当这些表发生写入操作时，闭包会自动重新执行并发送新值。无需手动调用 `reload` 或发送通知。

---

## 全文搜索（FTS5）

GRDB 对 SQLite FTS5 全文搜索提供了开箱即用的支持：

### 创建 FTS5 虚拟表

```swift
migrator.registerMigration("create_search_index") { db in
    try db.create(virtualTable: "articles_fts", using: FTS5()) { t in
        t.tokenizer = .unicode61()  // Unicode 分词器
        t.column("title")
        t.column("body")
        t.column("author_id").notIndexed()  // 不参与搜索，仅用于关联
    }
}
```

### 执行搜索

```swift
struct ArticleFTS: Codable, FetchableRecord, PersistableRecord, TableRecord {
    var title: String
    var body: String
    var authorId: String
    static let databaseTableName = "articles_fts"
}

// 前缀搜索（输入 "swi" 能匹配 "swift"）
let pattern = try FTS5Pattern(rawPattern: "swi*")
let results = try ArticleFTS
    .matching(pattern)
    .fetchAll(db)
```

> `TableRecord` 协议为 FTS5 虚拟表提供了 `.matching()` 方法入口。

### 关于中文搜索

FTS5 的 `unicode61` 分词器对中文支持有限（按空格/标点分词）。如果需要中文搜索，常见做法是：
- 使用第三方中文分词器（如 `simple` tokenizer）
- 或结合 LIKE 模糊匹配作为兜底

---

## 与 Core Data / Realm 的对比

| 维度 | GRDB | Core Data | Realm |
|---|---|---|---|
| **学习曲线** | 低-中 | 高 | 低 |
| **类型安全** | 编译时检查 | 运行时 | 编译时检查 |
| **性能（查询）** | 极快（原生 SQLite） | 中等 | 较慢（内存映射） |
| **内存占用** | 低 | 中 | 高 |
| **响应式更新** | ValueObservation + Combine | NSFetchedResultsController | 内置 LiveData |
| **数据库迁移** | DatabaseMigrator | 轻量级迁移 | 自动但有限 |
| **全文搜索** | FTS5 原生支持 | 需第三方 | 需第三方 |
| **跨平台** | Apple 全平台 | Apple 全平台 | 全平台 |
| **开源** | MIT | Apple 框架 | Apache 2.0 |
| **包大小** | ~2MB | 系统内置 | ~10MB+ |

**适用场景建议：**
- **GRDB** — 中小型到大型应用，需要高性能和类型安全，希望完全控制数据库
- **Core Data** — 已有 CoreData 遗留项目，或需要 iCloud 同步
- **Realm** — 快速原型、需要实时跨设备同步、复杂的对象图管理

---

## 最佳实践

### 1. 模型层：统一使用 Codable + 协议组合

```swift
// 所有模型遵循统一的协议组合
struct MyModel: Codable, FetchableRecord, PersistableRecord, Identifiable, Sendable {
    // ...
}
```

### 2. 数据层：按领域拆分 Extension

当项目规模增长后，建议将数据库操作按领域拆分，保持单一职责：

```
Database/
├── DatabaseManager.swift           // 核心：连接、迁移、通用方法
├── DatabaseManager+Users.swift      // 用户相关
├── DatabaseManager+Orders.swift     // 订单相关
├── DatabaseManager+Search.swift     // 搜索逻辑
└── DatabaseManager+Observers.swift  // 响应式订阅
```

### 3. 列名映射：始终使用 CodingKeys + Columns

```swift
// CodingKeys 负责属性名 ↔ 列名映射
enum CodingKeys: String, CodingKey {
    case parentId = "parent_id"
}

// Columns 负责查询构建（类型安全）
enum Columns {
    static let parentId = Column(CodingKeys.parentId)
}

// 查询时直接用 Column，编译器会帮你检查拼写
try Model.filter(Model.Columns.parentId == targetId).fetchAll(db)
```

### 4. 迁移中加入防御性检查

```swift
migrator.registerMigration("v2_add_column") { db in
    // 检查列是否已存在，避免重复迁移导致崩溃
    let columns = try db.columns(in: "my_table")
    if !columns.contains(where: { $0.name == "new_column" }) {
        try db.alter(table: "my_table") { t in
            t.add(column: "new_column", .text)
        }
    }
}
```

### 5. 批量操作优先于循环单条操作

```swift
// 推荐：一条 SQL 搞定批量更新
try Todo
    .filter(ids.contains(Todo.Columns.id))
    .updateAll(db, [Todo.Columns.isCompleted.set(to: true)])

// 避免：N 次数据库往返
for id in ids {
    var todo = try Todo.fetchOne(db, key: id)!
    todo.isCompleted = true
    try todo.update(db)
}
```

### 6. 利用事务保证数据一致性

```swift
try dbPool.write { db in
    // 以下操作在同一个事务中，要么全部成功，要么全部回滚
    try order.insert(db)
    for item in orderItems {
        try item.insert(db)
    }
    try updateInventory(db, for: orderItems)  // 扣减库存
}
```

### 7. 善用 ValueObservation 驱动 UI

```swift
// 写入后不需要手动刷新 UI
// ValueObservation 会自动检测到表变化并重新发送数据
try dbPool.write { db in
    try newTodo.insert(db)  // 写入
}
// → Observer 自动收到新的 [Todo]，SwiftUI 视图自动刷新
```

### 8. 软删除配合级联操作

```swift
// 软删除：保留恢复能力
try Todo
    .filter(ids.contains(Todo.Columns.id))
    .updateAll(db, [
        Todo.Columns.isDeleted.set(to: true),
        Todo.Columns.deletedAt.set(to: Date())
    ])

// 硬删除：依靠外键级联自动清理关联数据
t.column("todo_id", .text)
    .references("todos", onDelete: .cascade)  // 删除 todo 时自动清理关联
```

### 9. Configuration 中开启常用选项

```swift
var config = Configuration()

// 推荐开启外键约束
config.foreignKeysEnabled = true

// 开发阶段可以开启 SQL 日志
config.prepareDatabase { db in
    db.trace { print("SQL: \($0)") }
}

let dbPool = try DatabasePool(path: url.path, configuration: config)
```

---

## 总结

GRDB.swift 是一个设计精良的 SQLite 工具包，在类型安全、性能和 API 易用性之间取得了优秀的平衡。其核心优势在于：

- **协议驱动设计**（`FetchableRecord` / `PersistableRecord`）让模型定义简洁直观，一个 struct + Codable 就能开始
- **类型安全的查询构建**（`Column` 泛型）消除了字符串拼接 SQL 的隐患，编译时就能发现列名拼写错误
- **`ValueObservation` + Combine** 实现了真正的响应式数据驱动，写入数据后 UI 自动刷新
- **`DatabaseMigrator`** 让数据库 Schema 演进变得可管理，支持增量迁移和防御性编程
- **FTS5 原生支持** 开箱即用全文搜索能力
- **批量操作和事务** 让 CRUD 代码既简洁又高效

如果你正在为 iOS/macOS 应用选择数据持久化方案，GRDB 是一个值得认真考虑的优秀选择。它既不像 Core Data 那样复杂，也不像 FMDB 那样缺乏类型安全，而是在功能和易用性之间提供了一个恰到好处的平衡点。

---

## 参考资料

- [GRDB.swift GitHub 仓库](https://github.com/groue/GRDB.swift)
- [GRDB 官方文档](https://swiftpackageindex.com/groue/GRDB.swift)
- [GRDB 性能测试报告](https://github.com/groue/GRDB.swift/wiki/Performance)
