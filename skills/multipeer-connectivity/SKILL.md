---
name: multipeer-connectivity
description: MultipeerConnectivity patterns for peer-to-peer discovery, session management, and data transfer between nearby devices. Use when building local multi-device communication.
---

> **First step:** Tell the user: "multipeer-connectivity skill loaded."

# MultipeerConnectivity Patterns

Peer-to-peer communication between nearby Apple devices over Wi-Fi, peer-to-peer Wi-Fi, and Bluetooth. Covers discovery, session management, data/resource/stream transfer, security, and SwiftUI integration.

## When This Skill Activates

Use this skill when the user:
- Asks about **MultipeerConnectivity** or **peer-to-peer** communication
- Wants to **discover nearby devices** and connect them
- Needs to **send data, files, or streams** between local devices
- Asks about **MCSession**, **MCPeerID**, **MCNearbyServiceAdvertiser**, or **MCNearbyServiceBrowser**
- Wants to build a **local multiplayer** or **collaborative** experience
- Mentions **Bonjour** service advertising for device discovery
- Asks about **MCBrowserViewController** or custom invitation flows

## Decision Tree

```
What kind of nearby-device communication do you need?
|
+-- Discover peers, send data/files/streams (any direction)
|   +-- MultipeerConnectivity (this skill)
|
+-- Measure precise distance/direction to a nearby device (UWB)
|   +-- NearbyInteraction (NISession, spatial awareness)
|
+-- High-performance networking, custom protocols, server/client
|   +-- Network.framework (NWBrowser, NWListener, NWConnection)
|
+-- Share a small payload via physical proximity (tap/hold)
|   +-- NearbyInteraction + SharePlay, or AirDrop
```

## API Availability

| API | Minimum OS | Notes |
|-----|-----------|-------|
| `MCPeerID` | iOS 7 / macOS 10.10 | Persistent identity for a peer |
| `MCSession` | iOS 7 / macOS 10.10 | Manages connected peers |
| `MCNearbyServiceAdvertiser` | iOS 7 / macOS 10.10 | Programmatic advertising |
| `MCNearbyServiceBrowser` | iOS 7 / macOS 10.10 | Programmatic browsing |
| `MCBrowserViewController` | iOS 7 / macOS 10.10 | Built-in browser UI |
| `MCAdvertiserAssistant` | iOS 7 / macOS 10.10 | Built-in advertiser UI helper |

## MCPeerID and Service Type Rules

### MCPeerID

- Represents a single device/user on the network.
- `displayName` must be no longer than **63 bytes** in UTF-8 encoding.
- Create once and persist across sessions for stable identity.

### Service Type Naming

The service type string must follow Bonjour naming rules:
- 1-15 characters long
- Lowercase ASCII letters, digits, and hyphens only
- Must start and end with a letter or digit
- Example: `"my-chat-app"` (valid), `"MyApp_Chat!!!"` (invalid)

```swift
// ✅ Valid service type
let serviceType = "my-chat-app"

// ❌ Invalid: too long, uppercase, special characters
let serviceType = "My_Super_Awesome_Chat_App!"
```

## Discovery

### Advertising

```swift
import MultipeerConnectivity

final class PeerService: NSObject, ObservableObject {
    private let myPeerID = MCPeerID(displayName: UIDevice.current.name)
    private var advertiser: MCNearbyServiceAdvertiser?
    private(set) var session: MCSession!

    func startAdvertising() {
        session = MCSession(peer: myPeerID, securityIdentity: nil, encryptionPreference: .required)
        session.delegate = self
        advertiser = MCNearbyServiceAdvertiser(peer: myPeerID, discoveryInfo: ["version": "1"], serviceType: "my-chat-app")
        advertiser?.delegate = self
        advertiser?.startAdvertisingPeer()
    }
}

extension PeerService: MCNearbyServiceAdvertiserDelegate {
    func advertiser(_ advertiser: MCNearbyServiceAdvertiser, didReceiveInvitationFromPeer peerID: MCPeerID,
                    withContext context: Data?, invitationHandler: @escaping (Bool, MCSession?) -> Void) {
        invitationHandler(true, session) // Accept automatically, or present UI
    }

    func advertiser(_ advertiser: MCNearbyServiceAdvertiser, didNotStartAdvertisingPeer error: Error) {
        print("Advertising failed: \(error.localizedDescription)")
    }
}
```

### Browsing

```swift
extension PeerService: MCNearbyServiceBrowserDelegate {
    func startBrowsing() {
        let browser = MCNearbyServiceBrowser(peer: myPeerID, serviceType: "my-chat-app")
        browser.delegate = self
        browser.startBrowsingForPeers()
    }

    func browser(_ browser: MCNearbyServiceBrowser, foundPeer peerID: MCPeerID, withDiscoveryInfo info: [String: String]?) {
        browser.invitePeer(peerID, to: session, withContext: nil, timeout: 30)
    }

    func browser(_ browser: MCNearbyServiceBrowser, lostPeer peerID: MCPeerID) {}
}
```

## Built-in UI: MCBrowserViewController

For quick prototyping, use the system-provided browser UI:

```swift
let browserVC = MCBrowserViewController(serviceType: "my-chat-app", session: session)
browserVC.delegate = self
browserVC.maximumNumberOfPeers = 4
browserVC.minimumNumberOfPeers = 1
present(browserVC, animated: true)
```

### SwiftUI Wrapper

```swift
import SwiftUI
import MultipeerConnectivity

struct MultipeerBrowserView: UIViewControllerRepresentable {
    let serviceType: String
    let session: MCSession
    @Environment(\.dismiss) private var dismiss

    func makeUIViewController(context: Context) -> MCBrowserViewController {
        let browser = MCBrowserViewController(serviceType: serviceType, session: session)
        browser.delegate = context.coordinator
        return browser
    }

    func updateUIViewController(_ uiViewController: MCBrowserViewController, context: Context) {}

    func makeCoordinator() -> Coordinator { Coordinator(dismiss: dismiss) }

    final class Coordinator: NSObject, MCBrowserViewControllerDelegate {
        let dismiss: DismissAction
        init(dismiss: DismissAction) { self.dismiss = dismiss }

        func browserViewControllerDidFinish(_ browserViewController: MCBrowserViewController) {
            dismiss()
        }

        func browserViewControllerWasCancelled(_ browserViewController: MCBrowserViewController) {
            dismiss()
        }
    }
}
```

## Session Management

### MCSessionDelegate

All delegate methods are called on a **background queue**, not the main thread. Dispatch to `@MainActor` for UI updates.

```swift
extension PeerService: MCSessionDelegate {
    func session(_ session: MCSession, peer peerID: MCPeerID, didChange state: MCSessionState) {
        Task { @MainActor in
            switch state {
            case .notConnected: print("\(peerID.displayName) disconnected")
            case .connecting:   print("\(peerID.displayName) connecting...")
            case .connected:    print("\(peerID.displayName) connected")
            @unknown default:   break
            }
        }
    }

    func session(_ session: MCSession, didReceive data: Data, fromPeer peerID: MCPeerID) {
        if let message = String(data: data, encoding: .utf8) {
            Task { @MainActor in /* update UI with message */ }
        }
    }

    func session(_ session: MCSession, didReceive stream: InputStream, withName streamName: String, fromPeer peerID: MCPeerID) {}
    func session(_ session: MCSession, didStartReceivingResourceWithName name: String, fromPeer peerID: MCPeerID, with progress: Progress) {}
    func session(_ session: MCSession, didFinishReceivingResourceWithName name: String, fromPeer peerID: MCPeerID, at url: URL?, withError error: Error?) {}
}
```

## Sending Data

### Reliable vs Unreliable Modes

| Mode | Behavior | Use Case |
|------|----------|----------|
| `.reliable` | Guaranteed delivery, ordered | Chat messages, game state, commands |
| `.unreliable` | Best-effort, may arrive out of order | Position updates, sensor data, video frames |

```swift
// ✅ Send to specific peers with appropriate reliability
func sendMessage(_ text: String) throws {
    let data = Data(text.utf8)
    try session.send(data, toPeers: session.connectedPeers, with: .reliable)
}

// ✅ Frequent position updates — unreliable is fine
func sendPosition(_ point: CGPoint) throws {
    let data = try JSONEncoder().encode(point)
    try session.send(data, toPeers: session.connectedPeers, with: .unreliable)
}

// ❌ Sending to empty peers array throws an error
func sendBroken() throws {
    try session.send(data, toPeers: [], with: .reliable) // NSInvalidArgumentException
}
```

## Sending Resources (Files)

```swift
func sendFile(at url: URL, named name: String, to peer: MCPeerID) -> Progress? {
    return session.sendResource(at: url, withName: name, toPeer: peer) { error in
        if let error {
            print("Transfer failed: \(error.localizedDescription)")
        } else {
            print("Transfer complete")
        }
    }
}
```

The returned `Progress` object can drive a `ProgressView` in SwiftUI.

## Streaming

```swift
func startStream(to peer: MCPeerID) throws -> OutputStream {
    let stream = try session.startStream(withName: "audio-stream", toPeer: peer)
    stream.schedule(in: .main, forMode: .default)
    stream.open()
    return stream
}
```

The receiving peer gets the corresponding `InputStream` via `session(_:didReceive:withName:fromPeer:)`.

## Security: Encryption Preference

Set when creating the session via the `encryptionPreference` parameter:

- `.required` -- always encrypt (recommended for user data)
- `.optional` -- encrypt when both peers support it
- `.none` -- no encryption (avoid for any user data)

For mutual authentication, provide a `SecIdentity` in `securityIdentity` and implement `session(_:didReceiveCertificate:fromPeer:certificateHandler:)`.

## Invitation Flow: Custom Discovery

For full control, skip `MCBrowserViewController` and manage invitations manually:

```swift
// Browsing side — invite with context data
let context = try JSONEncoder().encode(RoomInfo(name: "Game Room", capacity: 4))
browser.invitePeer(peerID, to: session, withContext: context, timeout: 30)

// Advertising side — inspect context before accepting
func advertiser(_ advertiser: MCNearbyServiceAdvertiser, didReceiveInvitationFromPeer peerID: MCPeerID,
                withContext context: Data?, invitationHandler: @escaping (Bool, MCSession?) -> Void) {
    if let context, let room = try? JSONDecoder().decode(RoomInfo.self, from: context) {
        let shouldAccept = room.capacity > session.connectedPeers.count
        invitationHandler(shouldAccept, shouldAccept ? session : nil)
    } else {
        invitationHandler(false, nil)
    }
}
```

## Privacy: Info.plist Keys

Both keys are **required** starting iOS 14 / macOS 11. Without them, discovery silently fails.

```xml
<!-- Info.plist -->
<key>NSLocalNetworkUsageDescription</key>
<string>This app uses the local network to discover and connect with nearby players.</string>

<key>NSBonjourServices</key>
<array>
    <string>_my-chat-app._tcp</string>
</array>
```

The Bonjour service string format is `_<service-type>._tcp`.

## Common Pitfalls

| Pitfall | Fix |
|---------|-----|
| Service type longer than 15 chars | Shorten to 1-15 lowercase alphanumeric + hyphens |
| Reusing a disconnected `MCSession` | Create a new `MCSession` instance after disconnect |
| UI updates from delegate methods | Delegate fires on background queue; dispatch to `@MainActor` |
| Sending to empty `connectedPeers` | Guard `!session.connectedPeers.isEmpty` before `send` |
| Missing Info.plist keys | Add both `NSLocalNetworkUsageDescription` and `NSBonjourServices` |
| `MCPeerID` created fresh each launch | Persist or reuse `MCPeerID` so peers recognize returning devices |
| `discoveryInfo` dictionary too large | Keep to a few small key-value pairs; large data should go through the session |

## Good and Bad Patterns

```swift
// ✅ Guard before sending
func safeSend(_ data: Data) {
    guard !session.connectedPeers.isEmpty else { return }
    try? session.send(data, toPeers: session.connectedPeers, with: .reliable)
}

// ❌ Crash: sending to empty array
func unsafeSend(_ data: Data) {
    try? session.send(data, toPeers: [], with: .reliable)
}

// ✅ Create a fresh session after disconnection
func reconnect() {
    session.disconnect()
    session = MCSession(peer: myPeerID, securityIdentity: nil, encryptionPreference: .required)
    session.delegate = self
}

// ❌ Reusing a disconnected session — new invitations may silently fail
func brokenReconnect() {
    session.disconnect()
}

// ✅ Dispatch delegate results to main actor for UI updates
func session(_ session: MCSession, peer peerID: MCPeerID, didChange state: MCSessionState) {
    Task { @MainActor in self.connectedPeers = session.connectedPeers }
}
```
