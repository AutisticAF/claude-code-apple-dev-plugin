---
name: swiftdata-patterns
description: SwiftData patterns for modeling, relationships, queries, predicates, sorting, migration, and ModelContainer configuration. Use when working with SwiftData persistence.
---

> **First step:** Tell the user: "swiftdata-patterns skill loaded."

# SwiftData Patterns

## When This Skill Activates

Use this skill when the user:
- Defines or modifies SwiftData `@Model` classes
- Configures `ModelContainer` or `ModelContext`
- Writes queries with `@Query`, `#Predicate`, or `FetchDescriptor`
- Sets up relationships between models
- Plans or performs schema migrations
- Needs background persistence via `ModelActor`
- Asks whether to use SwiftData, Core Data, UserDefaults, or file-based storage

## Persistence Decision Tree

| Scenario | Recommended |
|----------|-------------|
| Simple key-value preferences | `UserDefaults` / `@AppStorage` |
| Single JSON or Codable document | File-based (`FileManager` + `Codable`) |
| Relational object graph, iOS 17+ only | **SwiftData** |
| Relational object graph, must support iOS 16- | Core Data |
| CloudKit sync with SwiftData models | SwiftData + `ModelConfiguration(cloudKitDatabase:)` |
| Existing Core Data stack, incremental adoption | Core Data + SwiftData coexistence via shared `.xcdatamodeld` |

## API Availability

| API | Minimum OS |
|-----|-----------|
| `@Model`, `ModelContainer`, `ModelContext`, `@Query` | iOS 17, macOS 14 |
| `#Predicate`, `VersionedSchema`, `SchemaMigrationPlan` | iOS 17, macOS 14 |
| `ModelActor`, `DefaultSerialModelExecutor` | iOS 17, macOS 14 |
| `#Index` macro, custom data stores | iOS 18, macOS 15 |

---

## 1. @Model Macro

All stored properties are persisted by default. Use `@Transient` to opt out.

```swift
@Model
final class Item {
    var title: String
    var timestamp: Date
    var isComplete: Bool
    @Transient var cachedScore: Int = 0

    init(title: String, timestamp: Date = .now, isComplete: Bool = false) {
        self.title = title
        self.timestamp = timestamp
        self.isComplete = isComplete
    }
}
```

### @Attribute Options

```swift
@Model
final class Attachment {
    @Attribute(.unique) var identifier: String
    @Attribute(.externalStorage) var imageData: Data?
    @Attribute(.spotlight) var searchableTitle: String
    @Attribute(.transformable(by: "ColorTransformer")) var accentColor: UIColor?

    init(identifier: String, searchableTitle: String) {
        self.identifier = identifier
        self.searchableTitle = searchableTitle
    }
}
```

| Attribute | Purpose |
|-----------|---------|
| `.unique` | Enforces uniqueness; upserts on conflict |
| `.externalStorage` | Stores large blobs outside the database file |
| `.spotlight` | Indexes the property for Spotlight search |
| `.transformable(by:)` | Uses a ValueTransformer for non-standard types |

---

## 2. Relationships

```swift
@Model
final class Folder {
    var name: String
    @Relationship(deleteRule: .cascade, inverse: \Item.folder)
    var items: [Item] = []
    init(name: String) { self.name = name }
}

@Model
final class Item {
    var title: String
    var folder: Folder?
    init(title: String, folder: Folder? = nil) {
        self.title = title
        self.folder = folder
    }
}
```

| Delete Rule | Behavior |
|-------------|----------|
| `.cascade` | Deleting parent deletes all children |
| `.nullify` | Sets child's reference to `nil` (default) |
| `.deny` | Prevents deletion while children exist |
| `.noAction` | Removes reference without modifying related object |

---

## 3. ModelContainer

```swift
// Basic setup in App
@main
struct MyApp: App {
    var body: some Scene {
        WindowGroup { ContentView() }
            .modelContainer(for: [Folder.self, Item.self])
    }
}
```

### Custom Configuration with CloudKit

```swift
let config = ModelConfiguration(
    "MyStore", schema: Schema([Folder.self, Item.self]),
    url: URL.documentsDirectory.appending(path: "myapp.store"),
    cloudKitDatabase: .private("iCloud.com.example.myapp")
)
let container = try ModelContainer(for: Folder.self, Item.self, configurations: config)
```

### In-Memory Container for Previews

```swift
@MainActor
let previewContainer: ModelContainer = {
    let config = ModelConfiguration(isStoredInMemoryOnly: true)
    let container = try! ModelContainer(for: Folder.self, Item.self, configurations: config)
    let folder = Folder(name: "Sample")
    container.mainContext.insert(folder)
    folder.items.append(Item(title: "Task 1"))
    return container
}()

#Preview {
    ContentView().modelContainer(previewContainer)
}
```

---

## 4. ModelContext

```swift
@Environment(\.modelContext) private var context

func addItem() {
    context.insert(Item(title: "New Item"))
    // Autosave is on by default; explicit save is optional
}

func removeItem(_ item: Item) { context.delete(item) }
func explicitSave() { try? context.save() }
```

> **Autosave**: `ModelContext` saves on UI lifecycle events. Disable with `context.autosaveEnabled = false` for batch work.

---

## 5. @Query Macro

```swift
// Basic
@Query var items: [Item]

// Filter + sort + animation
@Query(
    filter: #Predicate<Item> { !$0.isComplete },
    sort: \Item.timestamp, order: .reverse,
    animation: .default
) var activeItems: [Item]
```

### Dynamic Query via init

```swift
struct FolderDetailView: View {
    @Query var items: [Item]

    init(folder: Folder) {
        let name = folder.name
        _items = Query(
            filter: #Predicate<Item> { $0.folder?.name == name },
            sort: [SortDescriptor(\Item.timestamp, order: .reverse)]
        )
    }

    var body: some View {
        List(items) { item in Text(item.title) }
    }
}
```

---

## 6. #Predicate Macro

Type-safe, compile-time checked predicates.

```swift
let incomplete = #Predicate<Item> { !$0.isComplete }

let recentAndIncomplete = #Predicate<Item> {
    !$0.isComplete && $0.timestamp > Date.now.addingTimeInterval(-86400)
}

let titleSearch = #Predicate<Item> {
    $0.title.localizedStandardContains("draft")
}
```

> **Limitation**: Only a subset of Swift expressions work inside `#Predicate`. Avoid custom methods or complex computed properties.

---

## 7. SortDescriptor

```swift
let byDate = SortDescriptor(\Item.timestamp, order: .reverse)

// Compound: complete items last, then by recency
let compound: [SortDescriptor<Item>] = [
    SortDescriptor(\.isComplete),
    SortDescriptor(\.timestamp, order: .reverse)
]
```

---

## 8. FetchDescriptor

For fetching outside SwiftUI views with pagination support.

```swift
var descriptor = FetchDescriptor<Item>(
    predicate: #Predicate { !$0.isComplete },
    sortBy: [SortDescriptor(\.timestamp, order: .reverse)]
)
descriptor.fetchLimit = 20
descriptor.fetchOffset = 0
let results = try context.fetch(descriptor)
let count = try context.fetchCount(descriptor) // count without loading objects
```

---

## 9. Schema Migration

### Versioned Schemas

```swift
enum SchemaV1: VersionedSchema {
    static var versionIdentifier = Schema.Version(1, 0, 0)
    static var models: [any PersistentModel.Type] { [ItemV1.self] }

    @Model final class ItemV1 {
        var title: String
        init(title: String) { self.title = title }
    }
}

enum SchemaV2: VersionedSchema {
    static var versionIdentifier = Schema.Version(2, 0, 0)
    static var models: [any PersistentModel.Type] { [ItemV2.self] }

    @Model final class ItemV2 {
        var title: String
        var isComplete: Bool = false  // new property with default
        init(title: String) { self.title = title }
    }
}
```

### Migration Plan

```swift
enum AppMigrationPlan: SchemaMigrationPlan {
    static var schemas: [any VersionedSchema.Type] { [SchemaV1.self, SchemaV2.self] }
    static var stages: [MigrationStage] { [migrateV1toV2] }

    static let migrateV1toV2 = MigrationStage.lightweight(
        fromVersion: SchemaV1.self, toVersion: SchemaV2.self
    )
}

let container = try ModelContainer(
    for: SchemaV2.ItemV2.self, migrationPlan: AppMigrationPlan.self
)
```

> Use `.lightweight` when adding properties with defaults or renaming. Use `.custom` for data transforms requiring logic.

---

## 10. ModelActor for Background Operations

```swift
@ModelActor
actor DataImporter {
    func importRecords(_ records: [[String: Any]]) throws {
        for record in records {
            modelContext.insert(Item(
                title: record["title"] as? String ?? "",
                isComplete: record["done"] as? Bool ?? false
            ))
        }
        try modelContext.save()
    }
}

let importer = DataImporter(modelContainer: container)
try await importer.importRecords(payload)
```

---

## 11. Undo / Redo

```swift
// Enable undo on the main context
container.mainContext.undoManager = UndoManager()

struct EditView: View {
    @Environment(\.modelContext) private var context
    @Environment(\.undoManager) private var undoManager

    var body: some View {
        ContentArea()
            .onAppear { context.undoManager = undoManager }
            .toolbar {
                Button("Undo") { context.undoManager?.undo() }
                    .disabled(!(context.undoManager?.canUndo ?? false))
            }
    }
}
```

---

## Patterns: Good and Bad

```swift
// ✅ Final class, memberwise init, sensible defaults
@Model final class Task {
    var title: String
    var createdAt: Date
    var isComplete: Bool = false
    init(title: String, createdAt: Date = .now) {
        self.title = title; self.createdAt = createdAt
    }
}
// ❌ Struct (SwiftData requires reference types) + missing init
@Model struct Task { var title: String }

// ✅ Always declare inverse to keep the graph consistent
@Relationship(deleteRule: .cascade, inverse: \Note.folder) var notes: [Note] = []
// ❌ Omitting inverse — SwiftData may infer incorrectly
@Relationship(deleteRule: .cascade) var notes: [Note] = []

// ✅ @Query for reactive SwiftUI lists
@Query(sort: \Task.createdAt) var tasks: [Task]
// ❌ Fetching inside body — re-fetches every render
var body: some View {
    let tasks = try? context.fetch(FetchDescriptor<Task>())
    List(tasks ?? []) { t in Text(t.title) }
}

// ✅ @ModelActor for off-main-thread persistence
@ModelActor actor SyncWorker { /* ... */ }
// ❌ Passing model objects across actors — PersistentModel is not Sendable
// Pass PersistentIdentifier instead, then re-fetch in the target context
func process(_ item: Item) async { /* ... */ }

// ✅ In-memory container for previews: fast, isolated, no stale data
ModelConfiguration(isStoredInMemoryOnly: true)
// ❌ On-disk default in previews — accumulates data between sessions
#Preview { ContentView().modelContainer(for: Item.self) }
```
