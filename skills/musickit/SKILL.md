---
name: musickit
description: MusicKit patterns for Apple Music integration, catalog search, library access, playback control, and subscription status. Use when integrating Apple Music or ShazamKit.
---

# MusicKit

Apple Music integration, catalog search, library access, playback control, and audio recognition with ShazamKit.

## When This Skill Activates

Use this skill when the user:
- Wants to integrate Apple Music into their app
- Needs to search the Apple Music catalog or fetch songs, albums, artists, or playlists
- Asks about MusicCatalogSearchRequest, MusicCatalogResourceRequest, or MusicLibraryRequest
- Wants to control music playback with ApplicationMusicPlayer or SystemMusicPlayer
- Needs to check or present Apple Music subscription status
- Asks about ShazamKit for audio recognition or custom catalog matching
- Needs MusicKit authorization or privacy usage descriptions

## Decision Tree: MusicKit vs AVFoundation vs ShazamKit

| Goal | Framework | Key Types |
|------|-----------|-----------|
| Search Apple Music catalog | **MusicKit** | MusicCatalogSearchRequest, Song, Album |
| Access user's Apple Music library | **MusicKit** | MusicLibrary, MusicLibraryRequest |
| Control Apple Music playback | **MusicKit** | ApplicationMusicPlayer, SystemMusicPlayer |
| Play local/streamed audio files | **AVFoundation** | AVAudioPlayer, AVPlayer |
| Record or process audio | **AVFoundation** | AVAudioEngine, AVAudioSession |
| Recognize songs from audio | **ShazamKit** | SHSession, SHManagedSession |
| Match audio against custom catalog | **ShazamKit** | SHCustomCatalog, SHSignature |

## API Availability

| API | iOS | macOS | watchOS | tvOS | Notes |
|-----|-----|-------|---------|------|-------|
| MusicKit (Swift) | 15.0+ | 12.0+ | 8.0+ | 15.0+ | Swift-native framework |
| MusicCatalogSearchRequest | 15.0+ | 12.0+ | 8.0+ | 15.0+ | Catalog search |
| MusicLibraryRequest | 16.0+ | 13.0+ | 9.0+ | 16.0+ | Library queries |
| ApplicationMusicPlayer | 15.0+ | 12.0+ | -- | 15.0+ | In-app playback |
| SystemMusicPlayer | 15.0+ | 12.0+ | 8.0+ | 15.0+ | System-level playback |
| MusicSubscription | 15.0+ | 12.0+ | 8.0+ | 15.0+ | Subscription status |
| SHSession | 15.0+ | 12.0+ | -- | -- | Audio recognition |
| SHManagedSession | 17.0+ | 14.0+ | -- | -- | Simplified recognition |

## Authorization

Always request authorization before accessing MusicKit APIs:

```swift
import MusicKit

let status = await MusicAuthorization.request()
guard status == .authorized else { return }

// Check current status without prompting
let currentStatus = MusicAuthorization.currentStatus
```

## Searching the Apple Music Catalog

```swift
import MusicKit

var request = MusicCatalogSearchRequest(term: "Taylor Swift", types: [Song.self, Album.self, Artist.self])
request.limit = 25
let response = try await request.response()

for song in response.songs {
    print("\(song.title) - \(song.artistName)")
}
```

## Fetching Specific Content

Use `MusicCatalogResourceRequest` to fetch items by ID, and `MusicCatalogChartsRequest` for charts:

```swift
import MusicKit

// Fetch a specific song by ID
func fetchSong(id: MusicItemID) async throws -> Song? {
    let request = MusicCatalogResourceRequest<Song>(matching: \.id, equalTo: id)
    return try await request.response().items.first
}

// Fetch albums with relationships loaded
func fetchAlbums(artistID: MusicItemID) async throws -> [Album] {
    var request = MusicCatalogResourceRequest<Album>(matching: \.artistID, equalTo: artistID)
    request.properties = [.tracks, .artists]
    return Array(try await request.response().items)
}
```

## User's Library

Access the user's personal Apple Music library (requires authorization). `MusicLibraryRequest` is available in iOS 16+:

```swift
import MusicKit

func fetchRecentlyAdded() async throws -> [Album] {
    var request = MusicLibraryRequest<Album>()
    request.sort(by: \.libraryAddedDate, ascending: false)
    request.limit = 50
    return Array(try await request.response().items)
}

func searchLibrary(term: String) async throws -> [Song] {
    var request = MusicLibraryRequest<Song>()
    request.filter(text: term)
    return Array(try await request.response().items)
}

// Add a catalog item to the user's library
try await MusicLibrary.shared.add(song)
```

## Playback

### ApplicationMusicPlayer (In-App Playback)

Plays Apple Music within your app. Audio stops on background unless you configure a background audio session.

```swift
import MusicKit

let player = ApplicationMusicPlayer.shared

// Play a song or album
player.queue = [song]
try await player.play()

// Queue management
try await player.queue.insert(songs, position: .afterCurrentEntry)
try await player.queue.insert(songs, position: .tail)

// Observe state and control transport
let status = player.state.playbackStatus  // .playing, .paused, .stopped
player.pause()
try await player.play()
```

### SystemMusicPlayer

Controls the system Music app. Playback continues independently of your app:

```swift
let systemPlayer = SystemMusicPlayer.shared
systemPlayer.queue = [song]
try await systemPlayer.play()
```

## Subscription Status

Check whether the user has an active Apple Music subscription. Use `MusicSubscription.subscriptionUpdates` as an `AsyncSequence`:

```swift
import MusicKit

func checkSubscription() async -> Bool {
    var updates = MusicSubscription.subscriptionUpdates.makeAsyncIterator()
    guard let subscription = await updates.next() else { return false }
    return subscription.canPlayCatalogContent
}

// Ongoing observation in SwiftUI
struct MusicStatusView: View {
    @State private var subscription: MusicSubscription?

    var body: some View {
        Group {
            if let subscription, subscription.canPlayCatalogContent {
                Text("Apple Music subscriber")
            } else {
                SubscribeButton()
            }
        }
        .task {
            for await sub in MusicSubscription.subscriptionUpdates {
                subscription = sub
            }
        }
    }
}
```

## SwiftUI Integration

### Subscription Offer Sheet

```swift
import SwiftUI
import MusicKit

struct SubscribeButton: View {
    @State private var showOffer = false

    var body: some View {
        Button("Subscribe to Apple Music") {
            showOffer = true
        }
        .musicSubscriptionOffer(isPresented: $showOffer, options: offerOptions)
    }

    private var offerOptions: MusicSubscriptionOffer.Options {
        var options = MusicSubscriptionOffer.Options()
        options.affiliateToken = "YOUR_AFFILIATE_TOKEN"
        options.campaignToken = "YOUR_CAMPAIGN_TOKEN"
        return options
    }
}
```

### Artwork Display

Use `ArtworkImage` for rendering album art in SwiftUI:

```swift
if let artwork = song.artwork {
    ArtworkImage(artwork, width: 50, height: 50)
        .cornerRadius(6)
}
```

## ShazamKit

### Recognizing Songs (iOS 17+ Simplified API)

```swift
import ShazamKit

func recognizeSong() async throws -> SHMatch? {
    let session = SHManagedSession()
    let result = await session.result()
    switch result {
    case .match(let match):
        if let mediaItem = match.mediaItems.first {
            print("Found: \(mediaItem.title ?? "") by \(mediaItem.artist ?? "")")
        }
        return match
    case .noMatch:
        print("No match found")
        return nil
    case .error(let error):
        throw error
    }
}
```

### Manual Session with AVAudioEngine (iOS 15+)

For iOS 15-16 or when you need streaming control, use `SHSession` with `SHSessionDelegate`. Feed audio buffers via `AVAudioEngine`:

```swift
let session = SHSession()
session.delegate = self  // SHSessionDelegate

let inputNode = audioEngine.inputNode
let format = inputNode.outputFormat(forBus: 0)
inputNode.installTap(onBus: 0, bufferSize: 2048, format: format) { buffer, time in
    session.matchStreamingBuffer(buffer, at: time)
}
audioEngine.prepare()
try audioEngine.start()

// Delegate callbacks:
// session(_:didFind:) -> match found
// session(_:didNotFindMatchFor:error:) -> no match
```

### Custom Catalogs

Match audio against your own reference signatures by building an `SHCustomCatalog` and passing it to `SHSession(catalog:)`:

```swift
let catalog = SHCustomCatalog()
let generator = SHSignatureGenerator()
let audioFile = try AVAudioFile(forReading: audioURL)
let buffer = AVAudioPCMBuffer(pcmFormat: audioFile.processingFormat,
                               frameCapacity: AVAudioFrameCount(audioFile.length))!
try audioFile.read(into: buffer)
try generator.append(buffer, at: nil)

let mediaItem = SHMediaItem(properties: [.title: "My Track", .artist: "Artist"])
try catalog.addReferenceSignature(generator.signature(), representing: [mediaItem])
let session = SHSession(catalog: catalog)
```

## Privacy

Required Info.plist keys:

```xml
<!-- Required for MusicKit library access -->
<key>NSAppleMusicUsageDescription</key>
<string>This app needs access to your music library to play and recommend songs.</string>

<!-- Required for ShazamKit microphone access -->
<key>NSMicrophoneUsageDescription</key>
<string>This app uses the microphone to identify songs playing nearby.</string>
```

## Common Pitfalls and Patterns

### Authorization

```swift
// ✅ Good: Request authorization before any MusicKit call
let status = await MusicAuthorization.request()
guard status == .authorized else { return }
let results = try await MusicCatalogSearchRequest(term: "jazz", types: [Song.self]).response()

// ❌ Bad: Calling MusicKit APIs without authorization (throws error)
let results = try await MusicCatalogSearchRequest(term: "jazz", types: [Song.self]).response()
```

### Playback Player Choice

```swift
// ✅ Good: Use ApplicationMusicPlayer for in-app playback you control
let player = ApplicationMusicPlayer.shared
player.queue = [song]
try await player.play()

// ❌ Bad: Using SystemMusicPlayer when you want your app to control the experience
// SystemMusicPlayer controls the Music app, so your app loses control on background
let player = SystemMusicPlayer.shared
```

### Subscription Check Before Playback

```swift
// ✅ Good: Check subscription before playing catalog content
let canPlay = await checkSubscription()
guard canPlay else { showSubscriptionOffer(); return }
player.queue = [song]
try await player.play()

// ❌ Bad: Playing without checking (fails silently or throws if not subscribed)
player.queue = [song]
try await player.play()
```

### Search Request Configuration

```swift
// ✅ Good: Specify types and set a reasonable limit
var request = MusicCatalogSearchRequest(term: query, types: [Song.self, Album.self])
request.limit = 25

// ❌ Bad: Too many types and excessive limit
var request = MusicCatalogSearchRequest(term: query, types: [Song.self, Album.self, Artist.self, Playlist.self])
request.limit = 200
```
