---
name: carplay
description: CarPlay development patterns including CPTemplateApplicationScene, navigation, audio, communication, and EV charging templates. Use when building or adapting apps for CarPlay.
---

> **First step:** Tell the user: "carplay skill loaded."

# CarPlay Development

Guidance for building CarPlay-enabled apps using the CarPlay framework, including template selection, scene lifecycle, navigation, audio, communication, and testing.

## When This Skill Activates

Use this skill when the user:
- Asks about CarPlay development or integration
- Wants to build a navigation, audio, communication, or EV charging app for CarPlay
- Needs help choosing CarPlay templates (CPListTemplate, CPMapTemplate, etc.)
- Asks about CPTemplateApplicationScene or multi-scene lifecycle
- Wants to support instrument cluster / multi-screen display
- Needs to handle Siri intents within CarPlay
- Asks about CarPlay Simulator testing
- Wants to understand CarPlay entitlement and provisioning requirements
- Is adapting an existing iPhone app for the car

## Decision Tree: Choosing Your CarPlay App Type

```
What does your app do?
├── Turn-by-turn navigation ──────────► Navigation app (CPMapTemplate)
├── Stream or play audio ─────────────► Audio app (CPNowPlayingTemplate)
├── Messaging / VoIP calls ───────────► Communication app (CPMessageListItem, CPContact)
├── EV charging station finder ───────► EV Charging app (CPPointOfInterestTemplate)
├── Quick food ordering ──────────────► Quick Food Ordering app (CPListTemplate)
├── Fueling station finder ───────────► Fueling app (CPPointOfInterestTemplate)
├── Parking garage finder ────────────► Parking app (CPPointOfInterestTemplate)
└── Driving task (toll, road condition)► Driving Task app (CPListTemplate)
```

Each category requires a **specific CarPlay entitlement** from Apple. You cannot mix categories.

## API Availability

| API / Feature | Minimum OS | Notes |
|---|---|---|
| CarPlay framework (templates) | iOS 12 | Replaced `UIScreen`-based approach |
| CPTemplateApplicationScene | iOS 13 | Scene-based lifecycle (required) |
| CPTemplateApplicationDashboardScene | iOS 13.4 | Instrument cluster support |
| CPNowPlayingTemplate | iOS 14 | Standalone now-playing screen |
| CPPointOfInterestTemplate | iOS 14 | POI display with map pins |
| CPInformationTemplate | iOS 14 | Key-value detail screens |
| CPTabBarTemplate (5 tabs max) | iOS 14 | Tab-based root navigation |
| CPListTemplate sectioned headers | iOS 15 | Section headers and filtering |
| Instrument cluster map | iOS 15.4 | Navigation apps only |

## Entitlement and Provisioning Setup

1. **Request the CarPlay entitlement** from Apple at [developer.apple.com/carplay](https://developer.apple.com/contact/carplay/).
2. Apple grants a **category-specific** entitlement (e.g., `com.apple.developer.carplay-audio`).
3. Add the entitlement to your provisioning profile and `Entitlements.plist`:

```swift
// Entitlements.plist key (audio example)
// com.apple.developer.carplay-audio = true
```

4. Declare the CPTemplateApplicationScene in `Info.plist`:

```xml
<key>UIApplicationSceneManifest</key>
<dict>
    <key>UISceneConfigurations</key>
    <dict>
        <key>CPTemplateApplicationSceneSessionRoleApplication</key>
        <array>
            <dict>
                <key>UISceneClassName</key>
                <string>CPTemplateApplicationScene</string>
                <key>UISceneConfigurationName</key>
                <string>CarPlay</string>
                <key>UISceneDelegateClassName</key>
                <string>$(PRODUCT_MODULE_NAME).CarPlaySceneDelegate</string>
            </dict>
        </array>
    </dict>
</dict>
```

## CPTemplateApplicationScene Lifecycle

```swift
import CarPlay

class CarPlaySceneDelegate: UIResponder, CPTemplateApplicationSceneDelegate {
    var interfaceController: CPInterfaceController?

    func templateApplicationScene(
        _ templateApplicationScene: CPTemplateApplicationScene,
        didConnect interfaceController: CPInterfaceController
    ) {
        self.interfaceController = interfaceController
        let rootTemplate = buildRootTemplate()
        interfaceController.setRootTemplate(rootTemplate, animated: true, completion: nil)
    }

    func templateApplicationScene(
        _ templateApplicationScene: CPTemplateApplicationScene,
        didDisconnectInterfaceController interfaceController: CPInterfaceController
    ) {
        self.interfaceController = nil
    }
}
```

### Dashboard Scene (Instrument Cluster)

```swift
class CarPlayDashboardDelegate: UIResponder, CPTemplateApplicationDashboardSceneDelegate {
    func templateApplicationDashboardScene(
        _ templateApplicationDashboardScene: CPTemplateApplicationDashboardScene,
        didConnect dashboardController: CPDashboardController
    ) {
        let button = CPDashboardButton(
            titleVariants: ["Navigate"],
            subtitleVariants: ["To destination"],
            image: UIImage(systemName: "location.fill")!
        ) { _ in
            // Handle tap — opens your app's main CarPlay scene
        }
        dashboardController.shortcutButtons = [button]
    }
}
```

## Template Types

### CPTabBarTemplate (Root Container)

```swift
func buildRootTemplate() -> CPTabBarTemplate {
    let favoritesSection = CPListItem(text: "Home", detailText: "123 Main St")
    let favoritesTemplate = CPListTemplate(
        title: "Favorites",
        sections: [CPListSection(items: [favoritesSection])]
    )
    favoritesTemplate.tabImage = UIImage(systemName: "star.fill")

    let recentsTemplate = CPListTemplate(
        title: "Recents",
        sections: []
    )
    recentsTemplate.tabImage = UIImage(systemName: "clock.fill")

    return CPTabBarTemplate(templates: [favoritesTemplate, recentsTemplate])
}
```

### CPListTemplate

```swift
let item = CPListItem(text: "Coffee Shop", detailText: "0.3 mi away")
item.handler = { listItem, completion in
    // Navigate or show detail
    completion()
}
let section = CPListSection(items: [item], header: "Nearby", sectionIndexTitle: "N")
let listTemplate = CPListTemplate(title: "Places", sections: [section])
```

### CPGridTemplate

```swift
let gridButton = CPGridButton(
    titleVariants: ["Gas"],
    image: UIImage(systemName: "fuelpump.fill")!
) { _ in
    // Handle selection
}
let gridTemplate = CPGridTemplate(title: "Categories", gridButtons: [gridButton])
```

### CPInformationTemplate and CPPointOfInterestTemplate

```swift
// Information template for detail screens
let infoItem = CPInformationItem(title: "Distance", detail: "2.4 mi")
let infoTemplate = CPInformationTemplate(
    title: "Station Details",
    layout: .leading,
    items: [infoItem],
    actions: [CPTextButton(title: "Navigate", textStyle: .confirm) { _ in }]
)

// Point of Interest template
let poi = CPPointOfInterest(
    location: MKMapItem(placemark: MKPlacemark(coordinate: CLLocationCoordinate2D(latitude: 37.7749, longitude: -122.4194))),
    title: "Charger A",
    subtitle: "Available",
    summary: "50 kW DC Fast",
    detailTitle: "Charger A",
    detailSubtitle: "50 kW DC Fast Charger",
    detailSummary: "Available now — 2 of 4 ports open",
    pinImage: UIImage(systemName: "bolt.fill")
)
let poiTemplate = CPPointOfInterestTemplate(title: "Chargers", pointsOfInterest: [poi], selectedIndex: 0)
```

### CPAlertTemplate and CPActionSheetTemplate

```swift
let alert = CPAlertTemplate(
    titleVariants: ["Reroute?"],
    actions: [
        CPAlertAction(title: "Yes", style: .default) { _ in },
        CPAlertAction(title: "No", style: .cancel) { _ in }
    ]
)

let actionSheet = CPActionSheetTemplate(
    title: "Options",
    message: "Choose an action",
    actions: [
        CPAlertAction(title: "Share ETA", style: .default) { _ in },
        CPAlertAction(title: "Cancel", style: .cancel) { _ in }
    ]
)
```

## Navigation Apps

```swift
class NavigationManager: CPMapTemplateDelegate {
    func startNavigation(on mapTemplate: CPMapTemplate) {
        let trip = CPTrip(
            origin: MKMapItem.forCurrentLocation(),
            destination: destinationMapItem,
            routeChoices: [
                CPRouteChoice(
                    summaryVariants: ["Fastest — 25 min"],
                    additionalInformationVariants: ["via Highway 101"],
                    selectionSummaryVariants: ["25 min via 101"]
                )
            ]
        )

        mapTemplate.showTripPreviews([trip])

        // After user selects route:
        let session = mapTemplate.startNavigationSession(for: trip)
        session.pauseTrip(for: .loading, description: "Calculating…")

        let maneuver = CPManeuver()
        maneuver.instructionVariants = ["Turn right onto Oak St"]
        maneuver.symbolImage = UIImage(systemName: "arrow.turn.up.right")
        session.upcomingManeuvers = [maneuver]
        session.resumeTrip(with: .routeGuidance, description: "Navigating")
    }

    // MARK: - CPMapTemplateDelegate

    func mapTemplate(_ mapTemplate: CPMapTemplate, panWith direction: CPMapTemplate.PanDirection) {
        // Handle map panning
    }

    func mapTemplateDidBeginPanGesture(_ mapTemplate: CPMapTemplate) {
        // Show re-center button
    }
}
```

### Instrument Cluster Map (iOS 15.4+)

```swift
func templateApplicationScene(
    _ scene: CPTemplateApplicationScene,
    didConnect interfaceController: CPInterfaceController,
    to window: CPWindow
) {
    // The window is for the instrument cluster display
    let clusterVC = ClusterMapViewController()
    window.rootViewController = clusterVC
}
```

## Audio Apps

```swift
class AudioCarPlayManager: CPTemplateApplicationSceneDelegate {
    func templateApplicationScene(
        _ scene: CPTemplateApplicationScene,
        didConnect interfaceController: CPInterfaceController
    ) {
        let playlists = CPListItem(text: "My Playlists", detailText: "12 playlists")
        playlists.handler = { _, completion in
            // Push playlist detail template
            completion()
        }
        playlists.accessoryType = .disclosureIndicator

        let browseTemplate = CPListTemplate(
            title: "Music",
            sections: [CPListSection(items: [playlists])]
        )
        browseTemplate.tabImage = UIImage(systemName: "music.note.list")

        let nowPlaying = CPNowPlayingTemplate.shared
        nowPlaying.updateNowPlayingButtons([
            CPNowPlayingShuffleButton { _ in /* toggle shuffle */ },
            CPNowPlayingRepeatButton { _ in /* toggle repeat */ }
        ])

        let tabBar = CPTabBarTemplate(templates: [browseTemplate, nowPlaying])
        interfaceController.setRootTemplate(tabBar, animated: true, completion: nil)
    }
}
```

Audio apps must use `MPNowPlayingInfoCenter` and `MPRemoteCommandCenter` for playback metadata and controls.

## Communication Apps

```swift
let message = CPMessageListItem(
    conversationIdentifier: "conv-123",
    text: "Alice",
    leadingConfiguration: CPMessageListItem.LeadingConfiguration(
        leadingItem: .init(text: "A", textStyle: .abbreviation),
        unread: true
    ),
    trailingConfiguration: nil,
    trailingText: "2m ago"
)

let contact = CPContact(name: PersonNameComponents(givenName: "Alice", familyName: "Smith"))
contact.actions = [
    CPButton(
        image: UIImage(systemName: "phone.fill")!,
        handler: { _ in /* initiate call */ }
    )
]
```

Communication apps use `INSendMessageIntent` and `INStartCallIntent` for Siri integration.

## Handling Siri and CarPlay Voice Commands

```swift
// In your IntentHandler or AppIntent
class IntentHandler: INExtension, INStartAudioMediaIntentHandling {
    func handle(intent: INStartAudioMediaIntent) async -> INStartAudioMediaIntentResponse {
        guard let mediaSearch = intent.mediaSearch else {
            return INStartAudioMediaIntentResponse(code: .failure, userActivity: nil)
        }
        // Start playback based on mediaSearch
        return INStartAudioMediaIntentResponse(code: .success, userActivity: nil)
    }
}
```

Declare supported intents in `Info.plist` under `INIntentsSupported`.

## Testing in CarPlay Simulator

1. **Xcode CarPlay Simulator**: Window > Devices and Simulators > select your simulator > enable CarPlay from Features menu.
2. **External display simulation**: Use `Additional Simulator` in Xcode to test instrument cluster.
3. **Hardware testing**: Use a CarPlay-capable head unit or Apple's MFi CarPlay development kit.

Key testing checklist:
- Verify scene connect/disconnect lifecycle
- Test template stack depth (max 5 pushed templates)
- Validate all list items call their `completion()` handler
- Test with Siri ("Hey Siri, play X on MyApp")
- Test interruptions (phone calls, other audio sources)

## Common Pitfalls

| Pitfall | Detail |
|---|---|
| Template stack limit | Maximum 5 templates on the navigation stack. Plan your hierarchy. |
| No custom UI | You cannot draw custom UIKit views on the CarPlay screen (except map content in navigation apps). |
| Entitlement approval | Apple reviews CarPlay entitlement requests manually — apply early. |
| Forgetting completion handlers | Every `CPListItem.handler` must call `completion()` or the UI freezes. |
| Missing scene config | If `Info.plist` is missing the scene role, CarPlay silently never connects. |
| Audio session category | Audio apps must set `.playback` category with `.mixWithOthers` or `.duckOthers` policy. |

## Patterns

### ✅ Good Patterns

```swift
// ✅ Always call completion in list item handlers
item.handler = { item, completion in
    self.showDetail(for: item)
    completion()
}

// ✅ Provide multiple title variants (short to long) for different display sizes
let maneuver = CPManeuver()
maneuver.instructionVariants = [
    "Turn right onto Oak Street",
    "Turn right — Oak St",
    "Right on Oak"
]

// ✅ Check template stack before pushing
if (interfaceController.templates.count < 5) {
    interfaceController.pushTemplate(detailTemplate, animated: true, completion: nil)
}

// ✅ Clean up state on disconnect
func templateApplicationScene(_ scene: CPTemplateApplicationScene,
                              didDisconnectInterfaceController controller: CPInterfaceController) {
    self.interfaceController = nil
    self.navigationSession = nil
}
```

### ❌ Bad Patterns

```swift
// ❌ Never forget to call completion — this freezes the CarPlay UI
item.handler = { item, completion in
    self.showDetail(for: item)
    // missing completion() call!
}

// ❌ Don't try to present custom UIKit views on CarPlay
interfaceController.pushTemplate(myCustomViewController, animated: true) // Won't compile

// ❌ Don't exceed the template stack limit without checking
for category in allCategories {
    interfaceController.pushTemplate(category.template, animated: true, completion: nil)
    // Will crash after 5 pushes
}

// ❌ Don't hardcode a single title variant — truncation varies by head unit
maneuver.instructionVariants = ["Turn right onto Oak Street and merge onto Highway 101 North"]
```
