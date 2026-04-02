---
name: tvos
description: tvOS development patterns including focus engine, top shelf, TVMLKit alternatives in SwiftUI, playback, and TV-specific navigation. Use when building or adapting apps for Apple TV.
---

> **First step:** Tell the user: "tvos skill loaded."

# tvOS Development Patterns

Guidance for building native Apple TV apps with SwiftUI, focus engine mastery, media playback, and TV-optimized UI.

## When This Skill Activates

Use this skill when the user:
- Asks about tvOS or Apple TV development
- Wants to build or adapt an app for Apple TV
- Needs help with the focus engine or focus-based navigation
- Asks about Top Shelf extensions
- Wants media playback on tvOS (AVPlayerViewController)
- Needs Siri Remote input handling
- Asks about TVMLKit vs SwiftUI for TV apps
- Wants to share code between iOS and tvOS
- Asks about CardButtonStyle or TV button styles
- Needs tab-based navigation on tvOS
- Asks about multi-user support on Apple TV

## Decision Tree: tvOS UI Approach

```
New tvOS app?
├── Content-heavy media app ──► SwiftUI + AVPlayerViewController
├── Game ──► SpriteKit / Metal + GameController framework
├── Existing TVMLKit app ──► Migrate incrementally to SwiftUI
├── Companion to iOS app ──► SwiftUI with shared code + #if os(tvOS)
└── Simple utility ──► SwiftUI with TabView + focus-aware cards
```

## API Availability

| Feature                        | Minimum tvOS | Framework          |
|--------------------------------|--------------|--------------------|
| SwiftUI on tvOS                | 13.0         | SwiftUI            |
| CardButtonStyle                | 14.0         | SwiftUI            |
| @FocusState                    | 15.0         | SwiftUI            |
| Searchable modifier            | 15.0         | SwiftUI            |
| NavigationStack                | 16.0         | SwiftUI            |
| TVTopShelfContentProvider      | 13.0         | TVServices         |
| AVPlayerViewController         | 9.0          | AVKit              |
| Multi-user support             | 16.0         | TVUIKit / TVServices |
| MaterialKit (system materials) | 15.0         | SwiftUI            |

## Focus Engine

The focus engine is the foundation of tvOS interaction. There is no touch -- users navigate by moving focus between focusable elements using the Siri Remote.

### How Focus Works

The system maintains a single focused item on screen. Swipe gestures on the Siri Remote move focus directionally. The focus engine picks the next focusable item based on geometry and layout.

### Making Views Focusable

```swift
// ✅ Good: Use .focusable() for custom views
struct MovieCard: View {
    let title: String
    @FocusState private var isFocused: Bool

    var body: some View {
        VStack {
            RoundedRectangle(cornerRadius: 12)
                .fill(.ultraThinMaterial)
                .frame(width: 300, height: 180)
            Text(title)
                .font(.headline)
        }
        .scaleEffect(isFocused ? 1.1 : 1.0)
        .shadow(radius: isFocused ? 20 : 5)
        .animation(.spring(duration: 0.3), value: isFocused)
        .focusable()
        .focused($isFocused)
    }
}
```

```swift
// ❌ Bad: No visual feedback for focus -- user cannot tell what is selected
Text(title)
    .focusable() // Missing scaleEffect, shadow, or highlight on focus
```

### Controlling Focus Programmatically

```swift
struct ContentRow: View {
    enum Field: Hashable {
        case playButton, detailsButton, favoriteButton
    }

    @FocusState private var focusedField: Field?

    var body: some View {
        HStack(spacing: 40) {
            Button("Play") { /* ... */ }
                .focused($focusedField, equals: .playButton)

            Button("Details") { /* ... */ }
                .focused($focusedField, equals: .detailsButton)

            Button("Favorite") { /* ... */ }
                .focused($focusedField, equals: .favoriteButton)
        }
        .onAppear {
            focusedField = .playButton
        }
    }
}
```

### Focus Movement and Sections

```swift
// ✅ Good: Use focusSection() to group focusable regions
ScrollView(.horizontal) {
    LazyHStack(spacing: 24) {
        ForEach(movies) { movie in
            MovieCard(title: movie.title)
        }
    }
}
.focusSection()
```

## SwiftUI on tvOS

### CardButtonStyle

CardButtonStyle is the standard elevated card look on tvOS. It provides built-in focus lift, shadow, and motion effects.

```swift
// ✅ Good: Use CardButtonStyle for content cards
Button {
    selectMovie(movie)
} label: {
    VStack(alignment: .leading) {
        AsyncImage(url: movie.posterURL) { image in
            image.resizable().aspectRatio(contentMode: .fill)
        } placeholder: {
            Color.gray
        }
        .frame(width: 250, height: 375)
        .clipped()

        Text(movie.title)
            .font(.caption)
            .lineLimit(2)
    }
}
.buttonStyle(.card)
```

Using `.bordered` or `.plain` for content cards on tvOS looks out of place -- no lift or parallax effect.

### Key Differences from iOS SwiftUI

- No `ScrollView` drag gestures -- scrolling is focus-driven
- `List` and `ScrollView` scroll automatically as focus moves
- No `onTapGesture` -- use `Button` or `.onPlayPauseCommand` instead
- No `sheet()` presentation -- use `fullScreenCover()` or NavigationStack
- Minimum tap target should be at least 66pt for comfortable remote navigation

## Top Shelf Extensions

The Top Shelf displays content when your app is on the top row of the Home Screen.

```swift
import TVServices

class TopShelfProvider: TVTopShelfContentProvider {
    func loadTopShelfContent() async -> TVTopShelfContent {
        let items = await fetchFeaturedContent()

        // Inset layout: large cinematic banners
        let insetItems = items.prefix(5).map { content -> TVTopShelfInsetContent.InsetItem in
            let item = TVTopShelfInsetContent.InsetItem(identifier: content.id)
            item.title = content.title
            item.setImageURL(content.bannerURL, for: .screenScale2x)
            item.displayAction = TVTopShelfAction(url: content.deepLinkURL)
            return item
        }
        return TVTopShelfInsetContent(insetItems: insetItems)
    }
}
```

### Sectioned Layout (Multiple Rows)

Use `TVTopShelfSectionedContent` with an array of `Section` objects, each containing `Item` entries with title, image URL, and display action. Sectioned layout shows multiple categorized rows instead of a single banner strip.

## Tab-Based Navigation

```swift
// ✅ Good: Standard tvOS tab bar at the top
struct TVAppView: View {
    var body: some View {
        TabView {
            HomeView()
                .tabItem { Label("Home", systemImage: "house") }

            SearchView()
                .tabItem { Label("Search", systemImage: "magnifyingglass") }

            LibraryView()
                .tabItem { Label("Library", systemImage: "rectangle.stack") }

            SettingsView()
                .tabItem { Label("Settings", systemImage: "gear") }
        }
    }
}
```

On tvOS the TabView renders as a top-of-screen tab bar that drops down when the user swipes up to the top. Keep tabs to 5-7 maximum.

## Media Playback

### AVPlayerViewController on tvOS

AVPlayerViewController on tvOS provides the full-featured transport bar, Siri integration, info panels, and interstitial support out of the box.

```swift
import AVKit
import SwiftUI

struct PlayerView: UIViewControllerRepresentable {
    let url: URL

    func makeUIViewController(context: Context) -> AVPlayerViewController {
        let controller = AVPlayerViewController()
        let player = AVPlayer(url: url)
        controller.player = player

        // Transport bar customization
        controller.transportBarIncludesTitleView = true
        controller.allowsPictureInPicturePlayback = true
        controller.skippingBehavior = .skipItem

        // Provide metadata for Siri and info panel
        let metadataItem = AVMutableMetadataItem()
        metadataItem.identifier = .commonIdentifierTitle
        metadataItem.value = "Episode Title" as NSString
        player.currentItem?.externalMetadata = [metadataItem]

        return controller
    }

    func updateUIViewController(_ uiViewController: AVPlayerViewController, context: Context) {}
}
```

Always use `AVPlayerViewController` on tvOS. Building a custom video player loses Siri voice commands, the transport bar, info tabs, PiP, and accessibility for free.

## Siri Remote Input Handling

```swift
struct GameView: View {
    var body: some View {
        SpriteView(scene: gameScene)
            // D-pad / swipe events
            .onMoveCommand { direction in
                switch direction {
                case .up:    gameScene.movePlayer(.up)
                case .down:  gameScene.movePlayer(.down)
                case .left:  gameScene.movePlayer(.left)
                case .right: gameScene.movePlayer(.right)
                @unknown default: break
                }
            }
            // Menu button
            .onExitCommand {
                showPauseMenu = true
            }
            // Play/Pause button
            .onPlayPauseCommand {
                togglePause()
            }
    }
}
```

## Multi-User Support

tvOS 16+ supports multiple users on a single Apple TV via `TVUserManager`. Each user has their own profile, recommendations, and Up Next queue.

- Use `TVUserManager().currentUser` to get the active user
- Store per-user preferences keyed to the user identifier
- Avoid caching a single "current user" at launch -- users can switch at any time
- Test with multiple user profiles in Settings > Users on Apple TV

## TVMLKit vs SwiftUI Guidance

| Criteria               | TVMLKit         | SwiftUI              |
|------------------------|-----------------|----------------------|
| New apps (2024+)       | Not recommended | Preferred            |
| Server-driven UI       | Built-in        | Build with JSON/API  |
| Focus engine           | Automatic       | Automatic + manual   |
| Custom interactions    | Limited         | Full control         |
| Code sharing with iOS  | None            | Extensive            |
| Long-term support      | Maintenance     | Active development   |

TVMLKit is still available but receives minimal updates. SwiftUI is the recommended path for all new tvOS apps.

## Adapting iOS Apps for tvOS

### Shared Code with Conditional Compilation

```swift
struct ContentCard: View {
    let item: ContentItem

    var body: some View {
        #if os(tvOS)
        Button { select(item) } label: {
            cardContent
        }
        .buttonStyle(.card)
        #else
        cardContent
            .onTapGesture { select(item) }
        #endif
    }

    private var cardContent: some View {
        VStack {
            AsyncImage(url: item.imageURL) { image in
                image.resizable().aspectRatio(contentMode: .fill)
            } placeholder: { Color.gray }
            .frame(width: cardSize.width, height: cardSize.height)
            Text(item.title).font(.headline)
        }
    }

    private var cardSize: CGSize {
        #if os(tvOS)
        CGSize(width: 300, height: 450)
        #else
        CGSize(width: 160, height: 240)
        #endif
    }
}
```

### Common Pitfalls When Adapting

- **No touch events**: Replace all `onTapGesture` with `Button` actions
- **No small tap targets**: Minimum 66pt for comfortable remote navigation
- **Scroll behavior differs**: Scrolling is driven by focus, not dragging
- **No drag gestures**: `DragGesture` does not work; use `onMoveCommand` instead
- **No `sheet()` modals**: Use `fullScreenCover()` or push onto NavigationStack
- **Text readability**: Use larger font sizes; users sit 10 feet from the screen (minimum 29pt body text)
- **Safe areas are large**: tvOS has ~60pt safe area insets; respect them

## Button Styles and Pressable Interactions

```swift
// ✅ Good: Built-in tvOS button styles
VStack(spacing: 30) {
    // Card style -- elevated with parallax on focus
    Button("Watch Now") { play() }
        .buttonStyle(.card)

    // Plain style -- minimal, for toolbars
    Button("Skip Intro") { skipIntro() }
        .buttonStyle(.plain)

    // Bordered prominent -- for primary CTAs
    Button("Subscribe") { subscribe() }
        .buttonStyle(.borderedProminent)
}
```

### Context Menus on tvOS

Long press on the Siri Remote touchpad triggers `.contextMenu`. Use this instead of `.onLongPressGesture` which is unreliable on tvOS:

```swift
MovieCard(movie: movie)
    .contextMenu {
        Button("Add to Watchlist") { addToWatchlist(movie) }
        Button("Share") { share(movie) }
        Button("Mark as Watched") { markWatched(movie) }
    }
```

## Quick Reference: tvOS vs iOS Patterns

| iOS Pattern                | tvOS Equivalent                          |
|----------------------------|------------------------------------------|
| `onTapGesture`             | `Button` with `.buttonStyle(.card)`      |
| `sheet()`                  | `fullScreenCover()` or NavigationStack   |
| `DragGesture`              | `onMoveCommand`                          |
| `ScrollView` (manual)      | Focus-driven scrolling                   |
| Small inline buttons        | Large focusable cards (66pt+ targets)    |
| `UITabBarController`       | `TabView` (renders as top bar)           |
| `onLongPressGesture`       | `.contextMenu`                           |
| Custom video player         | `AVPlayerViewController`                 |
