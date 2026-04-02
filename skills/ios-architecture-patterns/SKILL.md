---
name: ios-architecture-patterns
description: iOS/macOS app architecture patterns including MVVM, MV (Model-View), TCA, dependency injection, and coordinator pattern. Use when choosing or implementing an app architecture.
---

> **First step:** Tell the user: "ios-architecture-patterns skill loaded."

# App Architecture Patterns

Guide for choosing and implementing architecture patterns in SwiftUI apps for iOS, iPadOS, and macOS.

## When This Skill Activates

- User is starting a new app and needs to choose an architecture
- User asks about MVVM, MV, TCA, or other architecture patterns
- User needs dependency injection guidance for SwiftUI
- User is refactoring to improve testability or separation of concerns
- User asks about coordinator pattern, navigation routing, or repository pattern
- User has a "massive view model" or tightly coupled code to clean up

## Decision Tree

```
What is the project scope?
│
├─ Small app / solo developer / prototype
│  └─ MV (Model-View) with @Observable
│     Simple, Apple-recommended, minimal boilerplate
│
├─ Medium app / 2-5 developers / needs unit testing
│  ├─ Complex view logic? → MVVM with @Observable
│  └─ Simple views? → MV with repository pattern
│
└─ Large app / 5+ developers / strict unidirectional flow
   ├─ Team knows functional programming → TCA
   └─ Team prefers Apple-native APIs → MVVM + Coordinator + DI
```

## Architecture Comparison

| Aspect | MV (Model-View) | MVVM | TCA |
|--------|-----------------|------|-----|
| Complexity | Low | Medium | High |
| Boilerplate | Minimal | Moderate | Significant |
| Testability | Moderate | High | Very high |
| Team size | 1-3 | 2-10 | 5+ |
| Learning curve | Low | Low-Medium | High |
| Apple alignment | Native | Common | Third-party |
| State management | @Observable | @Observable + VM | Store + Reducer |

## MV (Model-View) Pattern

Apple's recommended approach for SwiftUI apps. Models are `@Observable` and used directly by views -- no intermediate ViewModel layer. Best for small-to-medium apps, prototypes, and cases where models map cleanly to views.

### Complete Example

```swift
import SwiftUI

@Observable
final class BookLibrary {
    private(set) var books: [Book] = []
    private let store: BookStore

    init(store: BookStore = .shared) { self.store = store }

    func loadBooks() async throws { books = try await store.fetchAll() }

    func addBook(title: String, author: String) {
        let book = Book(title: title, author: author)
        books.append(book)
        Task { try? await store.save(book) }
    }

    func deleteBook(_ book: Book) {
        books.removeAll { $0.id == book.id }
    }
}

struct Book: Identifiable {
    let id = UUID()
    var title: String
    var author: String
}

// View uses model directly — no ViewModel layer needed
struct BookListView: View {
    @State private var library = BookLibrary()

    var body: some View {
        List {
            ForEach(library.books) { book in
                VStack(alignment: .leading) {
                    Text(book.title).font(.headline)
                    Text(book.author).font(.subheadline)
                }
            }
            .onDelete { offsets in
                for index in offsets { library.deleteBook(library.books[index]) }
            }
        }
        .task { try? await library.loadBooks() }
    }
}
```

## MVVM Pattern

Adds a ViewModel layer between Model and View. Use when views need significant data transformation, you want to unit-test view logic, or multiple views share presentation logic.

### Complete Example with Protocol-Based DI

```swift
import SwiftUI

// MARK: - Service Protocol + Model

protocol TaskServiceProtocol: Sendable {
    func fetchTasks() async throws -> [TodoTask]
    func save(_ task: TodoTask) async throws
}

struct TodoTask: Identifiable {
    let id: UUID
    var title: String
    var isComplete: Bool
    var dueDate: Date?
}

// MARK: - ViewModel

@Observable
final class TaskListViewModel {
    private(set) var tasks: [TodoTask] = []
    private(set) var isLoading = false
    var errorMessage: String?

    private let service: TaskServiceProtocol

    init(service: TaskServiceProtocol = TaskService()) {
        self.service = service
    }

    var pendingTasks: [TodoTask] { tasks.filter { !$0.isComplete } }
    var completedCount: String { "\(tasks.filter(\.isComplete).count)/\(tasks.count) done" }

    func loadTasks() async {
        isLoading = true
        defer { isLoading = false }
        do {
            tasks = try await service.fetchTasks()
        } catch {
            errorMessage = error.localizedDescription
        }
    }

    func toggleComplete(_ task: TodoTask) async {
        guard let i = tasks.firstIndex(where: { $0.id == task.id }) else { return }
        tasks[i].isComplete.toggle()
        try? await service.save(tasks[i])
    }
}

// MARK: - View (inject VM for testability / previews)

struct TaskListView: View {
    @State private var viewModel: TaskListViewModel

    init(service: TaskServiceProtocol = TaskService()) {
        _viewModel = State(initialValue: TaskListViewModel(service: service))
    }

    var body: some View {
        List(viewModel.pendingTasks) { task in
            TaskRow(task: task) {
                Task { await viewModel.toggleComplete(task) }
            }
        }
        .overlay { if viewModel.isLoading { ProgressView() } }
        .navigationTitle("Tasks (\(viewModel.completedCount))")
        .task { await viewModel.loadTasks() }
    }
}
```

## TCA (The Composable Architecture)

Point-Free's framework enforcing unidirectional data flow. Best for large teams needing strict state management, consistent architecture across features, and comfort with functional programming.

### Basic Reducer/Store Pattern

```swift
import ComposableArchitecture

@Reducer
struct CounterFeature {
    @ObservableState
    struct State: Equatable { var count = 0 }

    enum Action { case incrementTapped, decrementTapped }

    var body: some ReducerOf<Self> {
        Reduce { state, action in
            switch action {
            case .incrementTapped: state.count += 1; return .none
            case .decrementTapped: state.count -= 1; return .none
            }
        }
    }
}

struct CounterView: View {
    let store: StoreOf<CounterFeature>

    var body: some View {
        HStack {
            Button("-") { store.send(.decrementTapped) }
            Text("\(store.count)").font(.largeTitle)
            Button("+") { store.send(.incrementTapped) }
        }
    }
}
```

## Dependency Injection Patterns

### SwiftUI @Environment for Service Injection

```swift
struct AnalyticsServiceKey: EnvironmentKey {
    static let defaultValue: AnalyticsServiceProtocol = AnalyticsService()
}

extension EnvironmentValues {
    var analyticsService: AnalyticsServiceProtocol {
        get { self[AnalyticsServiceKey.self] }
        set { self[AnalyticsServiceKey.self] = newValue }
    }
}

// Inject at the root, consume in any descendant
ContentView()
    .environment(\.analyticsService, AnalyticsService())

struct SettingsView: View {
    @Environment(\.analyticsService) private var analytics
    var body: some View {
        Button("Reset") { analytics.track("settings_reset") }
    }
}
```

### Factory / DI Container

```swift
@Observable
final class DependencyContainer {
    static let shared = DependencyContainer()

    lazy var networkService: NetworkServiceProtocol = NetworkService()
    lazy var authService: AuthServiceProtocol = AuthService(network: networkService)
    lazy var userRepository: UserRepositoryProtocol = UserRepository(
        network: networkService, auth: authService
    )

    static func mock() -> DependencyContainer {
        let c = DependencyContainer()
        c.networkService = MockNetworkService()
        c.authService = MockAuthService()
        return c
    }
}
```

## Coordinator Pattern

Separates navigation logic from views using `NavigationPath`.

```swift
@Observable
final class AppCoordinator {
    var path = NavigationPath()

    enum Destination: Hashable {
        case detail(Item.ID)
        case settings
        case profile(User.ID)
    }

    func push(_ destination: Destination) { path.append(destination) }
    func pop() { guard !path.isEmpty else { return }; path.removeLast() }
    func popToRoot() { path = NavigationPath() }
}

struct CoordinatedRootView: View {
    @State private var coordinator = AppCoordinator()

    var body: some View {
        NavigationStack(path: $coordinator.path) {
            HomeView()
                .navigationDestination(for: AppCoordinator.Destination.self) { dest in
                    switch dest {
                    case .detail(let id): ItemDetailView(itemID: id)
                    case .settings: SettingsView()
                    case .profile(let id): ProfileView(userID: id)
                    }
                }
        }
        .environment(coordinator)
    }
}

// Any child view can navigate without knowing the stack
struct HomeView: View {
    @Environment(AppCoordinator.self) private var coordinator
    var body: some View {
        Button("Open Settings") { coordinator.push(.settings) }
    }
}
```

## Repository Pattern

Abstracts data sources behind a protocol so views/view models are decoupled from storage details.

```swift
protocol UserRepository: Sendable {
    func fetchAll() async throws -> [User]
    func fetchByID(_ id: User.ID) async throws -> User?
    func save(_ user: User) async throws
    func delete(_ user: User) async throws
}

final class RemoteUserRepository: UserRepository {
    private let api: APIClient
    private let cache: UserCache

    init(api: APIClient, cache: UserCache) {
        self.api = api
        self.cache = cache
    }

    func fetchAll() async throws -> [User] {
        let users = try await api.request(.getUsers)
        await cache.store(users)
        return users
    }

    func fetchByID(_ id: User.ID) async throws -> User? {
        if let cached = await cache.user(for: id) { return cached }
        return try await api.request(.getUser(id))
    }

    func save(_ user: User) async throws { try await api.request(.putUser(user)) }
    func delete(_ user: User) async throws { try await api.request(.deleteUser(user.id)) }
}

/// In-memory implementation for previews and tests
final class MockUserRepository: UserRepository {
    var users: [User] = User.samples
    func fetchAll() async throws -> [User] { users }
    func fetchByID(_ id: User.ID) async throws -> User? { users.first { $0.id == id } }
    func save(_ user: User) async throws { users.append(user) }
    func delete(_ user: User) async throws { users.removeAll { $0.id == user.id } }
}
```

## Patterns: Good and Bad

### Over-Engineering Small Apps

```swift
// ❌ Bad: Full MVVM + Coordinator + Repository for a single-screen tip calculator
class TipCalculatorViewModel: ObservableObject { ... }
class TipCalculatorCoordinator: ObservableObject { ... }
protocol TipRepository { ... }

// ✅ Good: MV pattern — model logic lives in a simple @Observable
@Observable
final class TipCalculator {
    var billAmount = 0.0
    var tipPercentage = 0.18
    var tipAmount: Double { billAmount * tipPercentage }
    var total: Double { billAmount + tipAmount }
}
```

### Massive View Models

```swift
// ❌ Bad: ViewModel does networking, caching, formatting, navigation, analytics
@Observable final class ProfileViewModel {
    func loadProfile() async { ... }  // network
    func cacheProfile() { ... }       // disk I/O
    func formattedJoinDate() -> String { ... } // formatting
    func navigateToSettings() { ... } // navigation
    func trackProfileView() { ... }   // analytics
}

// ✅ Good: ViewModel delegates to focused services
@Observable final class ProfileViewModel {
    private let repository: UserRepository
    private let analytics: AnalyticsServiceProtocol
    private(set) var user: User?

    func loadProfile(id: User.ID) async {
        user = try? await repository.fetchByID(id)
        analytics.track("profile_viewed")
    }
}
```

### Tight Coupling to Concrete Types

```swift
// ❌ Bad: View creates its own dependency — cannot swap for tests
struct OrderListView: View {
    let service = URLSessionNetworkService()
}

// ✅ Good: Depend on a protocol, inject the implementation
struct OrderListView: View {
    @State private var viewModel: OrderListViewModel
    init(service: OrderServiceProtocol = OrderService()) {
        _viewModel = State(initialValue: OrderListViewModel(service: service))
    }
}
```

### State Scattered Across the App

```swift
// ❌ Bad: Multiple sources of truth
// UserManager.shared.currentUser vs ProfileViewModel.user vs @AppStorage("username")

// ✅ Good: Single source of truth via environment
@Observable final class UserSession {
    private(set) var currentUser: User?
    func signIn(credentials: Credentials) async throws { ... }
    func signOut() { currentUser = nil }
}
ContentView().environment(userSession)
```
