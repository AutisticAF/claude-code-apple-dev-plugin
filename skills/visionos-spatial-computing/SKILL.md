---
name: visionos-spatial-computing
description: visionOS spatial computing patterns including windows, volumes, immersive spaces, RealityKit entities, hand tracking, and spatial interactions. Use when building visionOS apps beyond basic widgets.
---

> **First step:** Tell the user: "visionos-spatial-computing skill loaded."

# visionOS Spatial Computing

Comprehensive guide for building spatial experiences on Apple Vision Pro using SwiftUI, RealityKit, and ARKit. Covers windows, volumes, immersive spaces, hand tracking, spatial gestures, and 3D content integration.

## When This Skill Activates

- User is building a visionOS app or adding Vision Pro support
- User asks about windows, volumes, or immersive spaces
- User wants to display 3D models (USDZ, Reality files) in their app
- User needs hand tracking, eye tracking, or spatial gestures
- User asks about RealityKit entities, anchoring, or scene reconstruction
- User is implementing spatial audio or hover effects
- User wants ornaments or 3D UI chrome around windows
- User is choosing between immersion styles (.mixed, .full, .progressive)
- User asks about ARKit on visionOS (plane detection, hand tracking)

## Decision Tree

```
What spatial experience do you need?
│
├─ 2D interface (familiar app with depth)
│  └─ Window (.plain style) → see Windows section
│
├─ 3D object viewer (bounded, shared space)
│  └─ Volume (.volumetric style) → see Volumes section
│
├─ Blend 3D content with passthrough
│  └─ ImmersiveSpace (.mixed) → see Immersive Spaces section
│
├─ Gradually expand into immersion
│  └─ ImmersiveSpace (.progressive) → see Immersive Spaces section
│
└─ Fully virtual environment
   └─ ImmersiveSpace (.full) → see Immersive Spaces section
```

## API Availability

| Feature | Framework | Minimum OS | Notes |
|---------|-----------|------------|-------|
| WindowGroup | SwiftUI | visionOS 1.0 | Standard and volumetric |
| ImmersiveSpace | SwiftUI | visionOS 1.0 | Mixed, progressive, full |
| RealityView | SwiftUI | visionOS 1.0 | Bridge to RealityKit |
| SpatialTapGesture | SwiftUI | visionOS 1.0 | Eyes + pinch |
| DragGesture (3D) | SwiftUI | visionOS 1.0 | Spatial dragging |
| RotateGesture3D | SwiftUI | visionOS 1.0 | Two-hand rotation |
| MagnifyGesture | SwiftUI | visionOS 1.0 | Two-hand scale |
| Hand tracking | ARKit | visionOS 1.0 | Requires entitlement |
| Scene reconstruction | ARKit | visionOS 1.0 | Mesh of surroundings |
| Plane detection | ARKit | visionOS 1.0 | Horizontal/vertical |
| SpatialAudioComponent | RealityKit | visionOS 1.0 | Positional audio |
| HoverEffectComponent | RealityKit | visionOS 1.0 | Gaze highlight |
| Portal | RealityKit | visionOS 2.0 | Window into virtual world |

## Windows

Windows are the default presentation. They behave like familiar SwiftUI views with automatic depth and placement by the system.

### Window Styles

```swift
@main
struct MyApp: App {
    var body: some Scene {
        // Standard 2D window
        WindowGroup {
            ContentView()
        }

        // Volumetric window (3D bounded box)
        WindowGroup(id: "3d-viewer") {
            VolumeContentView()
        }
        .windowStyle(.volumetric)
        .defaultSize(width: 0.5, height: 0.5, depth: 0.5, in: .meters)
    }
}
```

### Opening Windows Programmatically

```swift
@Environment(\.openWindow) private var openWindow

Button("Show 3D Viewer") {
    openWindow(id: "3d-viewer")
}
```

## Volumes

Volumes display 3D content in a bounded region. They exist in the Shared Space alongside other apps. Use `RealityView` to place RealityKit entities.

```swift
struct VolumeContentView: View {
    var body: some View {
        RealityView { content in
            // Load a USDZ model
            if let model = try? await ModelEntity(named: "toy_robot") {
                model.position = [0, 0, 0]
                model.scale = [0.01, 0.01, 0.01]
                content.add(model)
            }
        }
    }
}
```

Keep content within the declared `defaultSize` bounds. Use meters for sizing (0.5m is roughly arm's length). Volumes are repositionable by the user via the window bar.

## Immersive Spaces

Immersive spaces extend content beyond a bounded volume into the user's surroundings or a fully virtual environment.

### Declaring an Immersive Space

```swift
ImmersiveSpace(id: "solar-system") {
    SolarSystemView()
}
.immersionStyle(selection: .constant(.mixed), in: .mixed)
```

### Immersion Styles

| Style | Behavior |
|-------|----------|
| `.mixed` | 3D content blends with passthrough; user sees real world |
| `.progressive` | Starts mixed, user can dial immersion with Digital Crown |
| `.full` | Complete virtual environment replaces passthrough |

### Opening and Dismissing

```swift
@Environment(\.openImmersiveSpace) private var openImmersiveSpace
@Environment(\.dismissImmersiveSpace) private var dismissImmersiveSpace

// Open — always check the result
let result = await openImmersiveSpace(id: "solar-system")
switch result {
case .opened: isImmersed = true
case .userCancelled: break
case .error: showError = true
@unknown default: break
}

// Dismiss
await dismissImmersiveSpace()
```

## RealityKit Integration

### Loading and Building Entities

```swift
// Load from app bundle (USDZ)
let model = try await ModelEntity(named: "scene")

// Load from Reality Composer Pro project
let scene = try await Entity(named: "MyScene", in: realityKitContentBundle)

// Build in code
let box = ModelEntity(
    mesh: .generateBox(size: 0.2),
    materials: [SimpleMaterial(color: .blue, isMetallic: true)]
)
box.components.set(CollisionComponent(shapes: [.generateBox(size: [0.2, 0.2, 0.2])]))
box.components.set(InputTargetComponent()) // Required for gestures
```

### RealityView with Attachments

SwiftUI views can be placed in 3D space as attachments:

```swift
RealityView { content, attachments in
    if let model = try? await ModelEntity(named: "globe") {
        content.add(model)
    }
    if let label = attachments.entity(for: "info-label") {
        label.position = [0, 0.3, 0]
        content.add(label)
    }
} attachments: {
    Attachment(id: "info-label") {
        Text("Earth")
            .font(.extraLargeTitle)
            .padding()
            .glassBackgroundEffect()
    }
}
```

## Hand Tracking and Spatial Gestures

### Spatial Tap Gesture (Eyes + Pinch)

```swift
RealityView { content in
    let sphere = ModelEntity(
        mesh: .generateSphere(radius: 0.1),
        materials: [SimpleMaterial(color: .green, isMetallic: false)]
    )
    sphere.components.set(CollisionComponent(shapes: [.generateSphere(radius: 0.1)]))
    sphere.components.set(InputTargetComponent())
    content.add(sphere)
}
.gesture(SpatialTapGesture().targetedToAnyEntity().onEnded { value in
    value.entity.scale *= 1.2
})
```

### Drag, Rotate, Scale

```swift
// Drag in 3D
.gesture(DragGesture().targetedToAnyEntity().onChanged { value in
    value.entity.position = value.convert(value.location3D, from: .local, to: .scene)
})

// Two-hand rotation
.gesture(RotateGesture3D().targetedToAnyEntity().onChanged { value in
    value.entity.orientation = simd_quatf(value.rotation)
})

// Two-hand pinch to scale
.gesture(MagnifyGesture().targetedToAnyEntity().onChanged { value in
    let scale = Float(value.magnification)
    value.entity.scale = [scale, scale, scale]
})

// Combine: drag + rotate simultaneously
.gesture(DragGesture().targetedToAnyEntity()
    .simultaneously(with: RotateGesture3D().targetedToAnyEntity()))
```

### ARKit Hand Tracking (Advanced)

Requires `com.apple.developer.arkit.hand-tracking.provider` entitlement:

```swift
let session = ARKitSession()
let handTracking = HandTrackingProvider()
try await session.run([handTracking])

for await update in handTracking.anchorUpdates {
    let hand = update.anchor
    if let indexTip = hand.skeleton.joint(.indexFingerTip) {
        // indexTip.anchorFromJointTransform gives finger position
    }
}
```

## Ornaments

Ornaments attach supplementary UI to the edge of a window:

```swift
MainContent()
    .ornament(attachmentAnchor: .scene(.bottom)) {
        HStack {
            Button("Play", systemImage: "play.fill") { }
            Button("Pause", systemImage: "pause.fill") { }
        }
        .padding()
        .glassBackgroundEffect()
    }
```

## Spatial Audio

```swift
let audioSource = Entity()
audioSource.components.set(SpatialAudioComponent())
entity.addChild(audioSource)

if let resource = try? await AudioFileResource(named: "ambient.mp3") {
    audioSource.playAudio(resource)
}
```

## Anchoring Entities

```swift
// Anchor to a horizontal surface (table)
let tableAnchor = AnchorEntity(.plane(.horizontal, classification: .table, minimumBounds: [0.3, 0.3]))
tableAnchor.addChild(modelEntity)
content.add(tableAnchor)

// Anchor to the user's hand
let handAnchor = AnchorEntity(.hand(.left, location: .palm))
handAnchor.addChild(particleEntity)
content.add(handAnchor)
```

## Scene Reconstruction and Plane Detection

Requires immersive space and appropriate entitlements:

```swift
let session = ARKitSession()
let planeDetection = PlaneDetectionProvider(alignments: [.horizontal, .vertical])
try await session.run([SceneReconstructionProvider(), planeDetection])

for await update in planeDetection.anchorUpdates {
    let plane = update.anchor
    // plane.geometry.meshVertices, plane.classification, etc.
}
```

## Hover Effects and Input Targeting

Entities highlight when the user looks at them. Requires `HoverEffectComponent`, `InputTargetComponent`, and `CollisionComponent`:

```swift
entity.components.set(HoverEffectComponent())
entity.components.set(InputTargetComponent())
entity.components.set(CollisionComponent(shapes: [.generateBox(size: [0.2, 0.2, 0.2])]))

// For SwiftUI views:
Button("Tap Me") { }.hoverEffect()
```

## Complete Example: Volume with 3D Model and Spatial Tap

```swift
import SwiftUI
import RealityKit

@main
struct SpatialApp: App {
    var body: some Scene {
        WindowGroup { HomeView() }

        WindowGroup(id: "model-viewer") {
            ModelViewerVolume()
        }
        .windowStyle(.volumetric)
        .defaultSize(width: 0.4, height: 0.4, depth: 0.4, in: .meters)
    }
}

struct ModelViewerVolume: View {
    var body: some View {
        RealityView { content in
            guard let robot = try? await ModelEntity(named: "toy_robot") else { return }
            robot.scale = [0.005, 0.005, 0.005]
            robot.position = [0, -0.1, 0]
            robot.components.set(InputTargetComponent())
            robot.components.set(CollisionComponent(
                shapes: [.generateBox(size: [0.2, 0.3, 0.2])]
            ))
            robot.components.set(HoverEffectComponent())
            content.add(robot)
        }
        .gesture(SpatialTapGesture().targetedToAnyEntity().onEnded { value in
            var transform = value.entity.transform
            transform.scale *= 1.1
            value.entity.move(to: transform, relativeTo: value.entity.parent, duration: 0.15)
        })
    }
}
```

## Good and Bad Patterns

### Scene Declaration

```swift
// ✅ Declare volumes with explicit size in meters
WindowGroup(id: "viewer") { VolumeView() }
    .windowStyle(.volumetric)
    .defaultSize(width: 0.5, height: 0.5, depth: 0.5, in: .meters)

// ❌ Omitting size — system guesses, content may clip
WindowGroup(id: "viewer") { VolumeView() }
    .windowStyle(.volumetric)
```

### Entity Interaction

```swift
// ✅ Both CollisionComponent AND InputTargetComponent required for gestures
entity.components.set(CollisionComponent(shapes: [.generateSphere(radius: 0.1)]))
entity.components.set(InputTargetComponent())

// ❌ Missing CollisionComponent — taps silently fail
entity.components.set(InputTargetComponent()) // won't receive gestures
```

### Immersive Space Lifecycle

```swift
// ✅ Handle all result cases when opening an immersive space
let result = await openImmersiveSpace(id: "mySpace")
switch result {
case .opened: isImmersed = true
case .userCancelled, .error: showError = true
@unknown default: break
}

// ❌ Fire-and-forget — ignores failure and user cancellation
await openImmersiveSpace(id: "mySpace")
```

### Loading Models

```swift
// ✅ Handle loading failures gracefully
RealityView { content in
    do {
        let model = try await ModelEntity(named: "robot")
        content.add(model)
    } catch {
        print("Failed to load model: \(error)")
    }
}

// ❌ Force-unwrap crashes on missing assets
let model = try! await ModelEntity(named: "robot")
```

### Gesture Targeting

```swift
// ✅ targetedToAnyEntity gives you the entity and spatial hit data
.gesture(SpatialTapGesture().targetedToAnyEntity().onEnded { value in
    value.entity.scale *= 1.2
})

// ❌ Plain TapGesture has no entity info or 3D position
.gesture(TapGesture().onEnded { /* which entity? */ })
```

### Shared Space Etiquette

```swift
// ✅ Keep volumes modest — they share space with other apps
.defaultSize(width: 0.5, height: 0.5, depth: 0.5, in: .meters)

// ❌ Oversized volumes crowd the user's shared space
.defaultSize(width: 3.0, height: 3.0, depth: 3.0, in: .meters)
```
