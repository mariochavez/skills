# Swift patterns

One section per concept: the target, why it holds, when to leave code as it is, and an example. `SKILL.md` indexes these sections by the code shape to spot.

> **Sources.** Distilled from the [Swift API Design Guidelines](https://www.swift.org/documentation/api-design-guidelines/), the [Swift 6.2 release notes](https://www.swift.org/blog/swift-6.2-released/), Swift Evolution SE-0461 (async functions run on the caller's actor) and SE-0466 (default main-actor isolation), Apple's [Observation](https://developer.apple.com/documentation/observation) documentation, the WWDC25 session "Embracing Swift concurrency", and Matt Massicotte's [When should you use an actor?](https://www.massicotte.org/actors). Paraphrased; read the originals for nuance.

## Contents
1. State and data flow
2. Views
3. Async work tied to views
4. Concurrency
5. Errors and load state
6. Types
7. Protocols and dependencies
8. Boundaries: generated bindings, AppKit, files
9. Localization and formatting
10. Testing
11. Package settings and tooling

---

## 1. State and data flow

**Target.** New code uses Observation: `@Observable` classes, `@State` to own them, `@Bindable` to bind, `@Environment` to share. There is one model per feature or screen, named for its domain (`ProjectsModel`, `SyncProgress`); child views take plain values or bindings. `@State` is private to the view that creates it. Derived values are computed properties.

**Why.** Observation tracks the properties a view actually reads, so updates are finer and the boilerplate disappears. A ViewModel per view means two files per screen and state copied between them. A `@State` seeded from a parent's value ignores every later change to that value.

**Leave it** when touched `ObservableObject` code is expensive or risky to migrate. Migrate it when the change is cheap and safe.

```swift
@Observable final class ProjectsModel {
    var filter = ProjectFilter()
    private(set) var projects: [ProjectSummary] = []
    var countText: String { projects.count.formatted() }   // derived, not stored
}

struct ProjectsView: View {
    @State private var model = ProjectsModel()
    var body: some View { ProjectList(projects: model.projects) }
}
```

Sharing through the environment:

```swift
WindowGroup { ContentView() }.environment(projectsModel)

struct FilterBar: View {
    @Environment(ProjectsModel.self) private var model
    var body: some View {
        @Bindable var model = model
        Toggle("filter.active_only", isOn: $model.filter.activeOnly)
    }
}
```

A child that needs data derived from a parent's selection takes the identity and loads by it:

```swift
struct Inspector: View {
    let projectID: ProjectID
    @State private var detail: ProjectDetail?
    @Environment(StoreClient.self) private var store
    var body: some View {
        DetailContent(detail: detail)
            .task(id: projectID) { detail = try? await store.detail(projectID) }
    }
}
```

## 2. Views

**Target.** A body fits on one screen. Large pieces are extracted as `struct` subviews, which give SwiftUI a diffing boundary and you a name. Conditional content uses `@ViewBuilder`, `Group` or `if` / `switch`. A subview receives the minimum it needs. Modifiers repeated across the app become a `ViewModifier` or a small extension; one-off styling stays inline.

**Why.** Views are lightweight value descriptions, so extracting one costs almost nothing. `AnyView` erases the type identity SwiftUI uses to diff.

**Leave it** when a small piece is used once and reads fine inline; a computed property returning a few lines of view is fine.

```swift
var body: some View {
    VStack(spacing: 0) {
        ProjectsHeader(title: model.scopeTitle, count: model.total)
        ProjectList(projects: model.projects, selection: $model.selection)
        StatusBar(summary: model.filterSummary)
    }
}

@ViewBuilder var content: some View {
    if model.projects.isEmpty { EmptyState() } else { ProjectList(projects: model.projects) }
}

ProjectRow(project: project, isSelected: selection.contains(project.id))   // not the whole model
```

## 3. Async work tied to views

**Target.** Work that belongs to a view's lifetime runs in `.task { }` or `.task(id:)`, which cancel automatically on disappear and when the id changes. One-shot async work is `async/await`.

**Why.** A `Task { }` started in `onAppear` is never cancelled and runs again on every appear. Combine for a single call adds operators and cancellables to manage.

```swift
.task(id: model.filter) { await model.reload() }
```

Debouncing without Combine:

```swift
.task(id: searchText) {
    try? await Task.sleep(for: .milliseconds(250))
    guard !Task.isCancelled else { return }
    model.filter.text = searchText
}
```

## 4. Concurrency

**Target.** Follow the target's isolation mode (section 11). With main-actor default isolation, models and views carry no `@MainActor`; only what leaves is marked (`@concurrent`, `nonisolated`). Measured heavy CPU work (decoding, parsing) leaves the main actor explicitly and returns `Sendable` values. An `actor` protects mutable state that several concurrent contexts actually share. Long loops check `Task.isCancelled` or call `try Task.checkCancellation()`.

**Why.** Swift 6 makes data-race safety a compiler guarantee, and Swift 6.2 lets a module default to the main actor and lets nonisolated async functions run on the caller's actor. Most isolation complaints point at unclear ownership. An actor, `Task.detached` or `@unchecked Sendable` can silence them while leaving the problem in place, and actors add suspension points and reentrancy.

**Leave it** when `@unchecked Sendable`, `nonisolated(unsafe)` or `MainActor.assumeIsolated` is backed by a comment proving it is safe, or when `Task.detached` is chosen to drop priority and isolation deliberately. Isolation modes stay as the project set them; switching one is never a drive-by refactor.

```swift
enum Images {
    @concurrent
    static func thumbnail(_ data: Data, maxPixel: Int) async -> CGImage? {
        guard let source = CGImageSourceCreateWithData(data as CFData, nil) else { return nil }
        let options: [CFString: Any] = [
            kCGImageSourceCreateThumbnailFromImageAlways: true,
            kCGImageSourceThumbnailMaxPixelSize: maxPixel,
            kCGImageSourceCreateThumbnailWithTransform: true,
        ]
        return CGImageSourceCreateThumbnailAtIndex(source, 0, options as CFDictionary)
    }
}

func load() async throws { data = try await Files.read(url) }   // Files.read is @concurrent; replaces DispatchQueue hops
```

Fan-out with a task group:

```swift
let thumbnails = await withTaskGroup(of: (AttachmentID, CGImage?).self) { group in
    for (id, data) in batch { group.addTask { (id, await Images.thumbnail(data, maxPixel: 256)) } }
    var result: [AttachmentID: CGImage] = [:]
    for await (id, image) in group { result[id] = image }
    return result
}
```

State only the main actor touches is a plain property on the model, not an actor:

```swift
@Observable final class ProjectsModel { var selection: Set<ProjectID> = [] }
```

## 5. Errors and load state

**Target.** Failures use `throws` and `do / catch`. A user-facing failure becomes state (`LoadState.failed(message)`), and a domain error shown to users is an enum conforming to `LocalizedError`. Input, files, network and FFI values are unwrapped with `guard let` or `throws`. `try?` is used where absence is a valid outcome. Typed throws are used where callers truly match on one error type.

**Why.** Separate `isLoading`, `errorMessage` and data properties allow combinations that mean nothing. A force unwrap on input is a crash waiting for bad data.

**Leave it** when a force unwrap is on a compile-time constant or an IBOutlet-style lifecycle, with a comment.

```swift
enum LoadState: Equatable { case idle, loading, loaded, failed(String) }
private(set) var state: LoadState = .idle

func reload() async {
    state = .loading
    do { projects = try await store.query(filter); state = .loaded }
    catch is CancellationError { /* superseded by a newer request */ }
    catch { state = .failed(error.localizedDescription) }
}
```

```swift
enum StoreError: LocalizedError {
    case locked, unsupportedVersion(String), invalidPath(String)
    var errorDescription: String? {
        switch self {
        case .locked: String(localized: "error.store_locked")
        case .unsupportedVersion(let v): String(localized: "error.unsupported_version \(v)")
        case .invalidPath(let p): String(localized: "error.invalid_path \(p)")
        }
    }
}

guard let url = URL(string: path) else { throw StoreError.invalidPath(path) }
```

## 6. Types

**Target.** `struct` by default; `final class` where identity or reference semantics are needed. Boolean flags and parallel optionals become an enum with associated values. Identifiers that are easy to swap get small wrapper structs. `let` over `var`, and access is `private` / `private(set)` until something needs more.

**Why.** Value types copy predictably, compare structurally and are `Sendable` for free when their contents are. Each invalid combination an enum rules out deletes defensive code elsewhere.

**Leave it** when two id types are never mixed in practice.

```swift
enum Status: Sendable { case draft, active, archived }   // replaces isActive + isArchived
enum Location: Sendable { case unknown, gps(latitude: Double, longitude: Double) }   // replaces hasGPS + two optionals

struct ProjectID: Hashable, Sendable { let rawValue: Int64 }
struct MemberID: Hashable, Sendable { let rawValue: Int64 }
```

## 7. Protocols and dependencies

**Target.** Concrete types by default. A protocol earns its place with two or more real conformers or a real extension point. A closed set of variants is an enum. Generics and `some` come before existentials (`any`), which are for heterogeneous storage. Dependencies reach views through `@Environment` or initializers. Types are named for what they are (`StoreClient`, `SyncProgress`), not a role suffix like `Manager` or `Service`.

**Why.** A protocol per service is a second place to look, and a mock that conforms to it drifts from reality. A view reaching for `.shared` has a dependency nobody can see from outside; injected dependencies document themselves at the call site and let previews and tests supply fixtures.

```swift
@Observable final class StoreClient {   // replaces StoreServiceProtocol with one conformer
    private let store: Store            // generated binding
    init(store: Store) { self.store = store }
    func query(_ filter: ProjectFilter) async throws -> [ProjectSummary] { /* convert + call */ }
}

struct Sidebar: View {
    @Environment(StoreClient.self) private var store   // replaces StoreClient.shared
    let tree: [FolderNode]
    var body: some View { List(tree, children: \.children) { FolderRow(node: $0) } }
}
```

## 8. Boundaries: generated bindings, AppKit, files

Read the subsection for a boundary the code actually has.

**Target.** Foreign types convert to app types once, at the edge. Interop types (binding wrappers, `NSViewRepresentable` coordinators) stay small and free of business logic. Inside the app, code trusts its own types.

### Generated bindings (UniFFI, C)

```swift
extension StoreClient {
    func counts(_ filter: ProjectFilter) async throws -> Counts {
        do { return Counts(try await store.counts(filter: filter.ffi)) }
        catch let error as FFIError { throw StoreError(error) }
    }
}
```
- Only the conversion layer imports the generated module; views and models see app types.
- Byte payloads from the core are decoded off the main actor (section 4).

### AppKit interop

`NSViewRepresentable` is for what SwiftUI can't do at the required scale or behavior, such as collection views with hundreds of thousands of items or custom text systems. State flows one way: values in, bindings out.

```swift
struct ItemGrid: NSViewRepresentable {
    let items: [ItemSummary]
    @Binding var selection: Set<ItemID>

    func makeCoordinator() -> Coordinator { Coordinator(selection: $selection) }

    func makeNSView(context: Context) -> NSScrollView {
        let collection = NSCollectionView()
        collection.collectionViewLayout = Self.layout()
        collection.isSelectable = true
        collection.allowsMultipleSelection = true
        collection.delegate = context.coordinator
        context.coordinator.configureDataSource(for: collection)
        let scroll = NSScrollView(); scroll.documentView = collection
        return scroll
    }

    func updateNSView(_ scroll: NSScrollView, context: Context) {
        context.coordinator.apply(items)   // diffable snapshot
    }
}
```
- The data source is an `NSCollectionViewDiffableDataSource` keyed by `ItemID`.
- The coordinator only translates between AppKit callbacks and SwiftUI bindings; loading and caching live in a model.
- Working AppKit interop stays when SwiftUI can't match its scale or behavior.

### Files

File paths and bookmarks become `URL` once at the edge; inside the app, paths are `URL`s.

## 9. Localization and formatting

**Target.** User-facing strings live in the String Catalog (`Text("projects.empty")`, `String(localized:)`), with interpolation so translators see the whole phrase. Numbers, dates, measurements and lists go through `FormatStyle`. Pluralization lives in the catalog. Keys follow the project's convention, semantic or source-string.

```swift
Text("status.showing \(count) of \(total)")
Text(date, format: .dateTime.year().month().day())
Text(distance.formatted(.measurement(width: .abbreviated)))
```

## 10. Testing

**Target.** New tests use Swift Testing (`@Test`, `#expect`, `#require`), with parameterized tests in place of copied cases. Tests run against real inputs: fixture files, temporary directories, in-memory data. Models and pure functions are tested; views are checked through previews and occasional UI tests. Previews use in-bundle fixture data.

**Why.** A protocol created so a mock can conform tests the mock, not the app.

```swift
import Testing
@testable import MyApp

@Suite struct ProjectsModelTests {
    @Test func reloadFillsProjectsFromFixture() async throws {
        let model = ProjectsModel(store: try StoreClient.fixture())
        await model.reload()
        #expect(model.state == .loaded)
        #expect(model.projects.count == 20)
    }

    @Test(arguments: [(Status.draft, 3), (.active, 16), (.archived, 1)])
    func statusFilter(status: Status, expected: Int) async throws {
        let model = ProjectsModel(store: try StoreClient.fixture())
        model.filter.statuses = [status]
        await model.reload()
        #expect(model.projects.count == expected)
    }
}
```
- `let first = try #require(model.projects.first)` unwraps and stops the test early.
- Previews: `#Preview { ProjectsView().environment(ProjectsModel.preview) }`.

## 11. Package settings and tooling

**Target.** Match the isolation mode the project already uses. For a new package or target, main-actor default isolation with approachable concurrency:

```swift
// swift-tools-version: 6.2
let package = Package(
    name: "MyKit",
    platforms: [.macOS(.v15)],
    targets: [
        .target(
            name: "MyKit",
            swiftSettings: [
                .defaultIsolation(MainActor.self),
                .enableUpcomingFeature("NonisolatedNonsendingByDefault"),
                .enableUpcomingFeature("InferIsolatedConformances"),
            ]
        ),
        .testTarget(name: "MyKitTests", dependencies: ["MyKit"]),
    ]
)
```
Xcode targets: `SWIFT_VERSION = 6`, `SWIFT_DEFAULT_ACTOR_ISOLATION = MainActor`, `SWIFT_APPROACHABLE_CONCURRENCY = YES`.

New warnings, concurrency warnings included, count as failures.
