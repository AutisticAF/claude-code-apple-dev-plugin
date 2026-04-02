---
name: performance-swiftui-debugging
description: Diagnose SwiftUI performance issues including unnecessary re-renders, view identity problems, and slow body evaluations. Use when SwiftUI views are slow, janky, or re-rendering too often.
---

# SwiftUI Performance Debugging

Systematic guide for diagnosing and fixing SwiftUI performance problems: unnecessary view re-evaluations, identity issues, expensive body computations, and lazy loading mistakes.

## When This Skill Activates

Use this skill when the user:
- Reports slow or janky SwiftUI views
- Sees excessive view re-renders or body re-evaluations
- Asks about `Self._printChanges()` or view debugging
- Has scrolling performance issues with lists or grids
- Asks why a view keeps updating when nothing changed
- Mentions `@Observable` or `ObservableObject` performance differences
- Wants to understand SwiftUI view identity or diffing
- Uses `AnyView` and asks about performance implications
- Has a hang or stutter traced to SwiftUI rendering

## Decision Tree

```
What SwiftUI performance problem are you seeing?
|
+- Views re-render when they should not
|  +- Read references/body-reevaluation.md
|     +- Self._printChanges() to identify which property changed
|     +- @Observable vs ObservableObject observation differences
|     +- Splitting views to narrow observation scope
|
+- Scrolling is slow / choppy (lists, grids)
|  +- Read references/lazy-loading.md
|     +- VStack vs LazyVStack, ForEach without lazy container
|     +- List prefetching, grid cell reuse
|
+- Views lose state unexpectedly / animate when they should not
|  +- Read references/view-identity.md
|     +- Structural vs explicit identity
|     +- .id() misuse, conditional view branching
|
+- Known pitfall (AnyView, DateFormatter in body, etc.)
|  +- Read references/common-pitfalls.md
|     +- AnyView type erasure, object creation in body
|     +- Over-observation, expensive computations
|
+- General "my SwiftUI app is slow" (unknown cause)
|  +- Start with references/body-reevaluation.md, then references/common-pitfalls.md
|  +- Use Instruments SwiftUI template (see Debugging Tools below)
```

## API Availability

| API / Technique | Minimum Version | Reference |
|----------------|-----------------|-----------|
| `Self._printChanges()` | iOS 15 | references/body-reevaluation.md |
| `@Observable` | iOS 17 / macOS 14 | references/body-reevaluation.md |
| `@ObservableObject` | iOS 13 | references/body-reevaluation.md |
| `LazyVStack` / `LazyHStack` | iOS 14 | references/lazy-loading.md |
| `LazyVGrid` / `LazyHGrid` | iOS 14 | references/lazy-loading.md |
| `.id()` modifier | iOS 13 | references/view-identity.md |
| Instruments SwiftUI template | Xcode 14+ | SKILL.md |
| `os_signpost` | iOS 12 | SKILL.md |

## Top 5 Mistakes -- Quick Reference

| # | Mistake | Fix | Details |
|---|---------|-----|---------|
| 1 | Large `ForEach` inside `VStack` or `ScrollView` without lazy container | Wrap in `LazyVStack` -- eager `VStack` creates all views upfront | references/lazy-loading.md |
| 2 | Using `AnyView` to erase types | Use `@ViewBuilder`, `Group`, or concrete generic types -- `AnyView` defeats diffing | references/common-pitfalls.md |
| 3 | Creating objects in `body` (`DateFormatter()`, `NumberFormatter()`) | Use `static let` shared instances or `@State` for mutable objects | references/common-pitfalls.md |
| 4 | Observing entire model when only one property is needed | Split into smaller `@Observable` objects or extract subviews | references/body-reevaluation.md |
| 5 | Unstable `.id()` values causing full view recreation every render | Use stable identifiers (database IDs, UUIDs), never array indices or random values | references/view-identity.md |

## Debugging Tools

### Self._printChanges()

Add to any view body to see what triggered re-evaluation:

```swift
var body: some View {
    let _ = Self._printChanges()
    // ... view content
}
```

Output reads: `ViewName: @self, @identity, _propertyName changed.`
See references/body-reevaluation.md for full interpretation guide.

### Instruments SwiftUI Template

1. **Xcode > Product > Profile** (Cmd+I)
2. Choose **SwiftUI** template (includes View Body, View Properties, Core Animation Commits)
3. Record, reproduce the slow interaction, stop
4. **View Body** lane shows which views had their body evaluated and how often
5. **View Properties** lane shows which properties changed

### os_signpost for Custom Measurement

```swift
import os

private let perfLog = OSLog(subsystem: "com.app.perf", category: "SwiftUI")

var body: some View {
    let _ = os_signpost(.event, log: perfLog, name: "MyView.body")
    // ... view content
}
```

View in Instruments with the **os_signpost** instrument to count body evaluations per second.

## Review Checklist

### View Identity
- [ ] No unstable `.id()` values (random, Date(), array index on mutable arrays)
- [ ] Conditional branches (`if`/`else`) do not cause unnecessary view destruction
- [ ] `ForEach` uses stable, unique identifiers from the model

### Body Re-evaluation
- [ ] Views observe only the properties they actually use
- [ ] `@Observable` classes preferred over `ObservableObject` (iOS 17+)
- [ ] No unnecessary `@State` changes that trigger body re-evaluation
- [ ] Large views split into smaller subviews to narrow observation scope

### Lazy Loading
- [ ] Large collections use `LazyVStack` / `LazyHStack`, not `VStack` / `HStack`
- [ ] `List` or lazy stack used for 50+ items
- [ ] No `.frame(maxHeight: .infinity)` on children inside lazy containers (defeats laziness)

### Common Pitfalls
- [ ] No `AnyView` type erasure (use `@ViewBuilder` or `Group`)
- [ ] No object allocation in `body` (`DateFormatter`, `NSPredicate`, view models)
- [ ] Expensive computations moved to background with `task { }` or `Task.detached`
- [ ] Images use `AsyncImage` or `.resizable()` with proper sizing, not raw `UIImage` decoding in body

## Reference Files

| File | Content |
|------|---------|
| [references/view-identity.md](references/view-identity.md) | Structural vs explicit identity, `.id()` usage, conditional branching |
| [references/body-reevaluation.md](references/body-reevaluation.md) | What triggers body, `_printChanges()`, `@Observable` vs `ObservableObject` |
| [references/lazy-loading.md](references/lazy-loading.md) | Lazy vs eager containers, `List`, `ForEach`, grid performance |
| [references/common-pitfalls.md](references/common-pitfalls.md) | `AnyView`, object creation in body, over-observation, expensive computations |
| performance-profiling skill | General Instruments profiling (Time Profiler, Memory, Energy) |
