---
name: ios-navigation-patterns
description: SwiftUI navigation architecture patterns including NavigationStack, NavigationSplitView, TabView, programmatic navigation, and custom transitions. Use when reviewing or building navigation, fixing navigation bugs, or architecting app flow.
---

# Navigation Patterns

Comprehensive guide for SwiftUI navigation architecture on iOS, iPadOS, and macOS. Covers the modern navigation APIs (iOS 16+/macOS 13+) with patterns for common and advanced use cases.

## When This Skill Activates

- User is building or reviewing navigation architecture
- User has navigation-related bugs (stack not updating, back button issues, state loss)
- User asks about NavigationStack, NavigationSplitView, TabView, or NavigationPath
- User needs programmatic navigation (push, pop, pop-to-root)
- User is implementing deep linking that connects to navigation
- User asks about navigation transitions or animations
- User is choosing between navigation approaches for their app

## Decision Tree

Use this to pick the right navigation container:

```
What is the app structure?
│
├─ Flat sections (3-5 top-level areas)
│  └─ TabView → see references/tab-view.md
│
├─ Hierarchical drill-down (list → detail)
│  └─ NavigationStack → see references/navigation-stack.md
│
├─ Sidebar + content (macOS / iPad)
│  ├─ Two columns → NavigationSplitView → see references/navigation-split-view.md
│  └─ Three columns → NavigationSplitView → see references/navigation-split-view.md
│
└─ Combined (tabs with drill-down, sidebar with stacks)
   └─ TabView + NavigationStack per tab
      OR NavigationSplitView + NavigationStack in detail
```

## Quick Reference

| Pattern | Container | Min OS | Reference |
|---------|-----------|--------|-----------|
| Simple drill-down | `NavigationStack` | iOS 16 | `references/navigation-stack.md` |
| Value-based links | `NavigationLink(value:)` | iOS 16 | `references/navigation-stack.md` |
| Programmatic push/pop | `NavigationPath` | iOS 16 | `references/programmatic-navigation.md` |
| Pop to root | `path = NavigationPath()` | iOS 16 | `references/programmatic-navigation.md` |
| State restoration | `NavigationPath.CodableRepresentation` | iOS 16 | `references/programmatic-navigation.md` |
| Two-column layout | `NavigationSplitView` | iOS 16 | `references/navigation-split-view.md` |
| Three-column layout | `NavigationSplitView` | iOS 16 | `references/navigation-split-view.md` |
| Column visibility | `NavigationSplitViewVisibility` | iOS 16 | `references/navigation-split-view.md` |
| Tab bar | `TabView` | iOS 13 | `references/tab-view.md` |
| Customizable tabs | `Tab` + `TabView` | iOS 18 | `references/tab-view.md` |
| Sidebar tabs (iPad) | `.tabViewStyle(.sidebarAdaptable)` | iOS 18 | `references/tab-view.md` |
| Zoom transition | `.navigationTransition(.zoom)` | iOS 18 | `references/navigation-transitions.md` |
| Custom transitions | `NavigationTransition` | iOS 18 | `references/navigation-transitions.md` |

## Process

### 1. Identify Navigation Needs

Read the user's code or requirements to determine:
- App structure (flat, hierarchical, sidebar-based)
- Target platforms (iOS only, iPad adaptive, macOS)
- Whether programmatic navigation is needed
- Deep linking requirements

### 2. Load Relevant Reference Files

Based on the need, read from this directory:
- `references/navigation-stack.md` — NavigationStack, NavigationLink, navigationDestination
- `references/navigation-split-view.md` — Two/three column layouts, column control, adaptive behavior
- `references/tab-view.md` — TabView, iOS 18 customizable tabs, sidebar mode
- `references/programmatic-navigation.md` — NavigationPath, state restoration, coordinators, pop-to-root
- `references/navigation-transitions.md` — Custom push/pop transitions (iOS 18+)

### 3. Review or Recommend

Apply patterns from the reference files. Check for common mistakes:

- [ ] Using deprecated `NavigationView` instead of `NavigationStack`/`NavigationSplitView`
- [ ] Using `NavigationLink(destination:)` instead of `NavigationLink(value:)` + `.navigationDestination`
- [ ] Placing `NavigationStack` inside `NavigationSplitView` detail (usually wrong)
- [ ] Missing `.navigationDestination` registration for a value type
- [ ] NavigationPath not `@State` or not in the right scope
- [ ] Multiple NavigationStacks competing for the same navigation context
- [ ] Hard-coding navigation instead of using `NavigationPath` for programmatic control
- [ ] Not handling deep links through the navigation system

### 4. Cross-Reference

- For **deep linking URL handling**, see `generators-deep-linking/` skill
- For **navigation animations**, see `design-animation-patterns/references/transitions.md`
- For **macOS sidebar patterns**, see `macos-ui-review-tahoe/references/swiftui-macos.md`

## References

- [NavigationStack](https://developer.apple.com/documentation/swiftui/navigationstack)
- [NavigationSplitView](https://developer.apple.com/documentation/swiftui/navigationsplitview)
- [NavigationPath](https://developer.apple.com/documentation/swiftui/navigationpath)
- [TabView](https://developer.apple.com/documentation/swiftui/tabview)
- [Migrating to new navigation types](https://developer.apple.com/documentation/swiftui/migrating-to-new-navigation-types)
