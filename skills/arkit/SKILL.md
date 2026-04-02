---
name: arkit
description: ARKit and RealityKit patterns for augmented reality experiences on iOS including world tracking, image detection, face tracking, and 3D content rendering. Use when building AR features.
---

> **First step:** Tell the user: "arkit skill loaded."

# ARKit & RealityKit Development

Guidance for building AR experiences on iOS using ARKit and RealityKit.

## When This Skill Activates

Use this skill when the user:
- Wants to add AR features to an iOS app
- Asks about ARKit session configuration or tracking
- Needs to place 3D content in the real world
- Wants to detect planes, images, faces, or bodies
- Asks about RealityKit entities, materials, or physics
- Needs to load and display `.usdz` or `.reality` files
- Wants people occlusion or scene understanding
- Asks about AR coaching overlay
- Needs to integrate ARView with SwiftUI

## Decision Tree: Which Framework Combination

### ARKit + RealityKit (Recommended for new projects)
- Modern renderer with physically based lighting
- Entity-Component-System (ECS) architecture
- Built-in physics, gestures, and animations
- Requires iOS 13+

### ARKit + SceneKit (Legacy)
- Node-based scene graph, iOS 11+
- No longer receiving major updates; migrate to RealityKit

### RealityKit Standalone (visionOS)
- visionOS uses RealityKit without ARKit via `RealityView`
- See visionOS skill for details

## API Availability

| Feature | Minimum OS | Device Requirement |
|---|---|---|
| ARWorldTrackingConfiguration | iOS 11 | A9+ chip |
| ARFaceTrackingConfiguration | iOS 11 | TrueDepth camera |
| ARImageTrackingConfiguration | iOS 12 | A9+ chip |
| ARBodyTrackingConfiguration | iOS 13 | A12+ chip |
| RealityKit | iOS 13 | A9+ chip |
| People occlusion | iOS 13 | A12+ chip, LiDAR preferred |
| Scene reconstruction | iOS 13.4 | LiDAR scanner |
| Object capture | iOS 17 | LiDAR scanner |

## ARView Setup

### SwiftUI with UIViewRepresentable

```swift
import SwiftUI
import ARKit
import RealityKit

struct ARViewContainer: UIViewRepresentable {
    func makeUIView(context: Context) -> ARView {
        let arView = ARView(frame: .zero)
        let config = ARWorldTrackingConfiguration()
        config.planeDetection = [.horizontal]
        config.environmentTexturing = .automatic
        arView.session.run(config)
        arView.session.delegate = context.coordinator
        return arView
    }

    func updateUIView(_ uiView: ARView, context: Context) {}

    func makeCoordinator() -> Coordinator { Coordinator() }

    class Coordinator: NSObject, ARSessionDelegate {
        func session(_ session: ARSession, didAdd anchors: [ARAnchor]) {
            for anchor in anchors {
                if let planeAnchor = anchor as? ARPlaneAnchor {
                    print("Detected plane: \(planeAnchor.classification)")
                }
            }
        }
    }
}
```

## ARSession Configuration Types

### World Tracking

```swift
let config = ARWorldTrackingConfiguration()
config.planeDetection = [.horizontal, .vertical]
config.environmentTexturing = .automatic
config.sceneReconstruction = .meshWithClassification // LiDAR
config.frameSemantics = [.personSegmentation]
arView.session.run(config)
```

### Face Tracking

```swift
guard ARFaceTrackingConfiguration.isSupported else { return }
let config = ARFaceTrackingConfiguration()
config.maximumNumberOfTrackedFaces = 1
config.isWorldTrackingEnabled = true // iOS 13+, combines face + world
arView.session.run(config)
```

### Image Tracking

```swift
guard let referenceImages = ARReferenceImage.referenceImages(
    inGroupNamed: "ARResources", bundle: nil
) else { return }

let config = ARImageTrackingConfiguration()
config.trackingImages = referenceImages
config.maximumNumberOfTrackedImages = 4
arView.session.run(config)
```

### Body Tracking

```swift
guard ARBodyTrackingConfiguration.isSupported else { return }
let config = ARBodyTrackingConfiguration()
config.automaticSkeletonScaleEstimationEnabled = true
arView.session.run(config)
```

## RealityKit Entities and Models

### Entity Hierarchy

- **Entity**: Base class, has transform but no visual representation
- **ModelEntity**: Entity with a 3D mesh and materials
- **AnchorEntity**: Attaches the entity tree to an AR anchor

### Creating and Placing Entities

```swift
// Primitive shape
let box = ModelEntity(
    mesh: .generateBox(size: 0.1),
    materials: [SimpleMaterial(color: .blue, isMetallic: true)]
)

// Anchor to a horizontal plane
let anchor = AnchorEntity(.plane(.horizontal, classification: .floor, minimumBounds: [0.2, 0.2]))
anchor.addChild(box)
arView.scene.addAnchor(anchor)
```

### Loading USDZ and Reality Files

```swift
// Async loading (preferred)
func loadModel() async throws {
    let entity = try await ModelEntity.load(named: "toy_robot")
    let anchor = AnchorEntity(.plane(.horizontal, classification: .any, minimumBounds: [0.1, 0.1]))
    anchor.addChild(entity)
    arView.scene.addAnchor(anchor)
}

// From a .reality file (exported from Reality Composer Pro)
func loadRealityScene() async throws {
    let scene = try await Entity.load(named: "MyScene", in: Bundle.main)
    let anchor = AnchorEntity(world: .zero)
    anchor.addChild(scene)
    arView.scene.addAnchor(anchor)
}
```

## Placing Content: Plane Detection and Ray Casting

```swift
// Ray cast from screen tap to detected plane
@objc func handleTap(_ recognizer: UITapGestureRecognizer) {
    let location = recognizer.location(in: arView)

    // RealityKit ray cast
    let results = arView.raycast(from: location, allowing: .estimatedPlane, alignment: .horizontal)

    guard let first = results.first else { return }

    let anchor = AnchorEntity(world: first.worldTransform)
    let sphere = ModelEntity(
        mesh: .generateSphere(radius: 0.05),
        materials: [SimpleMaterial(color: .red, isMetallic: false)]
    )
    anchor.addChild(sphere)
    arView.scene.addAnchor(anchor)
}
```

## Image Recognition

```swift
class ImageTrackingCoordinator: NSObject, ARSessionDelegate {
    weak var arView: ARView?

    func session(_ session: ARSession, didAdd anchors: [ARAnchor]) {
        for anchor in anchors {
            guard let imageAnchor = anchor as? ARImageAnchor else { continue }
            let name = imageAnchor.referenceImage.name ?? "unknown"

            let plane = ModelEntity(
                mesh: .generatePlane(
                    width: Float(imageAnchor.referenceImage.physicalSize.width),
                    depth: Float(imageAnchor.referenceImage.physicalSize.height)
                ),
                materials: [SimpleMaterial(color: .green.withAlphaComponent(0.5), isMetallic: false)]
            )

            let anchorEntity = AnchorEntity(anchor: imageAnchor)
            anchorEntity.addChild(plane)
            arView?.scene.addAnchor(anchorEntity)
            print("Detected image: \(name)")
        }
    }
}
```

## Face Tracking: Blend Shapes and Face Mesh

```swift
func session(_ session: ARSession, didUpdate anchors: [ARAnchor]) {
    for anchor in anchors {
        guard let faceAnchor = anchor as? ARFaceAnchor else { continue }

        let blendShapes = faceAnchor.blendShapes
        let mouthOpen = blendShapes[.jawOpen]?.floatValue ?? 0
        let smile = blendShapes[.mouthSmileLeft]?.floatValue ?? 0
        let leftBlink = blendShapes[.eyeBlinkLeft]?.floatValue ?? 0

        // Drive UI or 3D avatar with blend shape values
        DispatchQueue.main.async {
            self.updateAvatar(mouthOpen: mouthOpen, smile: smile)
        }
    }
}
```

## Physics and Collision

```swift
let box = ModelEntity(
    mesh: .generateBox(size: 0.1),
    materials: [SimpleMaterial(color: .blue, isMetallic: true)]
)

// Add physics body (dynamic = affected by forces and gravity)
box.components.set(PhysicsBodyComponent(
    massProperties: .default,
    material: .default,
    mode: .dynamic
))

// Add collision shape for interaction
box.generateCollisionShapes(recursive: true)

// Create a static floor for physics
let floor = ModelEntity(
    mesh: .generatePlane(width: 2, depth: 2),
    materials: [OcclusionMaterial()]
)
floor.components.set(PhysicsBodyComponent(
    massProperties: .default,
    material: .default,
    mode: .static
))
floor.generateCollisionShapes(recursive: true)
```

## Materials

```swift
// Physically based material
let metallic = SimpleMaterial(color: .gray, roughness: 0.2, isMetallic: true)

// Unlit material (not affected by lighting)
let unlit = UnlitMaterial(color: .yellow)

// Occlusion material (invisible but hides content behind it)
let occlusion = OcclusionMaterial()

// Textured material
var textured = SimpleMaterial()
let texture = try TextureResource.load(named: "wood_diffuse")
textured.color = .init(tint: .white, texture: .init(texture))
```

## People Occlusion and Scene Understanding

```swift
let config = ARWorldTrackingConfiguration()

// People occlusion: virtual objects hidden behind real people
if ARWorldTrackingConfiguration.supportsFrameSemantics(.personSegmentationWithDepth) {
    config.frameSemantics.insert(.personSegmentationWithDepth)
}

// Scene reconstruction with LiDAR
if ARWorldTrackingConfiguration.supportsSceneReconstruction(.meshWithClassification) {
    config.sceneReconstruction = .meshWithClassification
}

arView.environment.sceneUnderstanding.options = [
    .occlusion,       // Real-world objects occlude virtual content
    .receiveShadows,  // Virtual shadows cast on real surfaces
    .physics          // Virtual objects interact with real geometry
]

arView.session.run(config)
```

## Coaching Overlay

```swift
let coachingOverlay = ARCoachingOverlayView()
coachingOverlay.autoresizingMask = [.flexibleWidth, .flexibleHeight]
coachingOverlay.session = arView.session
coachingOverlay.goal = .horizontalPlane // .verticalPlane, .anyPlane, .tracking
coachingOverlay.activatesAutomatically = true
arView.addSubview(coachingOverlay)
```

## RealityKit Animations

```swift
// Transform animation
var transform = box.transform
transform.translation = SIMD3<Float>(0, 0.5, 0)
box.move(to: transform, relativeTo: box.parent, duration: 2.0, timingFunction: .easeInOut)

// Playing animations from USDZ files
if let animation = entity.availableAnimations.first {
    entity.playAnimation(animation.repeat()) // or .repeat(count: 3)
}
```

## Patterns

### ✅ Good Patterns

```swift
// ✅ Always check device capability before configuring
guard ARWorldTrackingConfiguration.isSupported else {
    showUnsupportedDeviceMessage()
    return
}

// ✅ Pause session when leaving the AR view
override func viewWillDisappear(_ animated: Bool) {
    super.viewWillDisappear(animated)
    arView.session.pause()
}

// ✅ Use coaching overlay for onboarding
let coaching = ARCoachingOverlayView()
coaching.goal = .horizontalPlane
coaching.activatesAutomatically = true
arView.addSubview(coaching)

// ✅ Handle session interruptions
func sessionWasInterrupted(_ session: ARSession) {
    showInterruptionOverlay()
}

func sessionInterruptionEnded(_ session: ARSession) {
    resetTracking()
}

// ✅ Use async model loading to avoid blocking
Task {
    let model = try await ModelEntity.load(named: "robot")
    anchor.addChild(model)
}
```

### ❌ Bad Patterns

```swift
// ❌ Never skip capability checks
let config = ARFaceTrackingConfiguration() // Crashes on devices without TrueDepth

// ❌ Never forget to pause the session — drains battery and holds camera

// ❌ Never load models synchronously on the main thread
let model = try! ModelEntity.loadModel(named: "large_scene") // Blocks UI

// ❌ Never ignore session errors
// Always implement session(_:didFailWithError:)

// ❌ Never use zero minimumBounds — places content on tiny detected patches
let anchor = AnchorEntity(.plane(.horizontal, classification: .any, minimumBounds: .zero))
```
