---
name: avfoundation
description: AVFoundation patterns for camera capture, video/audio playback, media editing, and audio sessions. Use when working with camera, microphone, video playback, or media processing.
---

> **First step:** Tell the user: "avfoundation skill loaded."

# AVFoundation Development Guide

Patterns and best practices for camera capture, media playback, audio sessions, and media composition on Apple platforms.

## When This Skill Activates

Use this skill when the user:
- Wants to capture photos or video with the camera
- Needs a camera preview in SwiftUI or UIKit
- Asks about video or audio playback with AVPlayer
- Needs to configure audio sessions (AVAudioSession)
- Wants to compose or edit video/audio tracks
- Asks about exporting media with AVAssetExportSession
- Mentions camera permissions or microphone access
- Needs to process video frames in real time

## Decision Tree: Choosing the Right API

```
What do you need?
├── Play video/audio
│   ├── Simple playback in SwiftUI → VideoPlayer (AVKit)
│   ├── Full-screen with controls → AVPlayerViewController (AVKit)
│   └── Custom player UI → AVPlayer + AVPlayerLayer
├── Capture from camera/mic
│   ├── Photos only → AVCaptureSession + AVCapturePhotoOutput
│   ├── Video recording → AVCaptureSession + AVCaptureMovieFileOutput
│   └── Real-time frame processing → AVCaptureSession + AVCaptureVideoDataOutput
├── Edit/compose media
│   ├── Combine tracks → AVMutableComposition
│   ├── Apply effects → AVVideoComposition
│   └── Export result → AVAssetExportSession
└── Audio session management → AVAudioSession
```

## API Availability

| API | iOS | macOS | visionOS | Notes |
|-----|-----|-------|----------|-------|
| AVCaptureSession | 4.0+ | 10.7+ | - | Camera/mic capture pipeline |
| AVCapturePhotoOutput | 10.0+ | 13.0+ | - | Replaces AVStillImageOutput |
| AVCaptureVideoDataOutput | 4.0+ | 10.7+ | - | Raw frame access |
| AVPlayer | 4.0+ | 10.7+ | 1.0+ | Media playback engine |
| AVPlayerViewController | 8.0+ | 10.15+ | 1.0+ | System playback UI (AVKit) |
| VideoPlayer (SwiftUI) | 14.0+ | 11.0+ | 1.0+ | SwiftUI wrapper (AVKit) |
| AVAudioSession | 3.0+ | - | - | iOS/watchOS only |
| AVMutableComposition | 4.0+ | 10.7+ | 1.0+ | Media editing/export |

## Camera Capture Pipeline

### Session Setup with Photo Capture

```swift
import AVFoundation

class CameraManager: NSObject, ObservableObject, AVCapturePhotoCaptureDelegate {
    let session = AVCaptureSession()
    private let sessionQueue = DispatchQueue(label: "camera.session")
    private let photoOutput = AVCapturePhotoOutput()
    @Published var capturedImage: UIImage?

    func configure() {
        sessionQueue.async { [self] in
            session.beginConfiguration()
            session.sessionPreset = .photo

            guard let camera = AVCaptureDevice.default(
                .builtInWideAngleCamera, for: .video, position: .back
            ) else { return }

            do {
                let input = try AVCaptureDeviceInput(device: camera)
                if session.canAddInput(input) { session.addInput(input) }
            } catch {
                print("Camera input error: \(error)")
                return
            }

            if session.canAddOutput(photoOutput) {
                session.addOutput(photoOutput)
            }

            session.commitConfiguration()
        }
    }

    func start() {
        sessionQueue.async { if !self.session.isRunning { self.session.startRunning() } }
    }

    func stop() {
        sessionQueue.async { if self.session.isRunning { self.session.stopRunning() } }
    }

    func capturePhoto() {
        let settings = AVCapturePhotoSettings()
        settings.flashMode = .auto
        photoOutput.capturePhoto(with: settings, delegate: self)
    }

    func photoOutput(_ output: AVCapturePhotoOutput,
                     didFinishProcessingPhoto photo: AVCapturePhoto, error: Error?) {
        guard let data = photo.fileDataRepresentation(),
              let image = UIImage(data: data) else { return }
        DispatchQueue.main.async { self.capturedImage = image }
    }
}
```

### SwiftUI Camera Preview

```swift
import SwiftUI
import AVFoundation

struct CameraPreview: UIViewRepresentable {
    let session: AVCaptureSession

    func makeUIView(context: Context) -> UIView {
        let view = UIView(frame: .zero)
        let previewLayer = AVCaptureVideoPreviewLayer(session: session)
        previewLayer.videoGravity = .resizeAspectFill
        view.layer.addSublayer(previewLayer)
        context.coordinator.previewLayer = previewLayer
        return view
    }

    func updateUIView(_ uiView: UIView, context: Context) {
        context.coordinator.previewLayer?.frame = uiView.bounds
    }

    func makeCoordinator() -> Coordinator { Coordinator() }

    class Coordinator { var previewLayer: AVCaptureVideoPreviewLayer? }
}

struct CameraView: View {
    @StateObject private var camera = CameraManager()

    var body: some View {
        CameraPreview(session: camera.session)
            .ignoresSafeArea()
            .onAppear { camera.configure(); camera.start() }
            .onDisappear { camera.stop() }
    }
}
```

### Video Recording

```swift
class VideoRecordingManager: NSObject, ObservableObject, AVCaptureFileOutputRecordingDelegate {
    let session = AVCaptureSession()
    private let sessionQueue = DispatchQueue(label: "video.session")
    private let movieOutput = AVCaptureMovieFileOutput()
    @Published var isRecording = false

    func configure() {
        sessionQueue.async { [self] in
            session.beginConfiguration()
            session.sessionPreset = .high

            if let camera = AVCaptureDevice.default(.builtInWideAngleCamera, for: .video, position: .back),
               let videoInput = try? AVCaptureDeviceInput(device: camera),
               session.canAddInput(videoInput) {
                session.addInput(videoInput)
            }
            if let mic = AVCaptureDevice.default(for: .audio),
               let audioInput = try? AVCaptureDeviceInput(device: mic),
               session.canAddInput(audioInput) {
                session.addInput(audioInput)
            }
            if session.canAddOutput(movieOutput) { session.addOutput(movieOutput) }

            session.commitConfiguration()
        }
    }

    func startRecording() {
        let url = FileManager.default.temporaryDirectory
            .appendingPathComponent(UUID().uuidString).appendingPathExtension("mov")
        movieOutput.startRecording(to: url, recordingDelegate: self)
        DispatchQueue.main.async { self.isRecording = true }
    }

    func stopRecording() { movieOutput.stopRecording() }

    func fileOutput(_ output: AVCaptureFileOutput, didFinishRecordingTo url: URL,
                    from connections: [AVCaptureConnection], error: Error?) {
        DispatchQueue.main.async { self.isRecording = false }
        if let error { print("Recording error: \(error)") }
    }
}
```

## Video Playback

### AVPlayerViewController (UIKit)

```swift
import AVKit

func presentPlayer(from vc: UIViewController, url: URL) {
    let player = AVPlayer(url: url)
    let playerVC = AVPlayerViewController()
    playerVC.player = player
    vc.present(playerVC, animated: true) { player.play() }
}
```

### SwiftUI VideoPlayer with Status Observation

```swift
import SwiftUI
import AVKit

struct VideoPlayerView: View {
    @State private var player = AVPlayer(url: URL(string: "https://example.com/video.mp4")!)

    var body: some View {
        VideoPlayer(player: player)
            .frame(height: 300)
            .onAppear { player.play() }
            .onDisappear { player.pause() }
    }
}

class PlayerViewModel: ObservableObject {
    let player = AVPlayer()
    private var statusObservation: NSKeyValueObservation?
    private var timeObserver: Any?

    func load(url: URL) {
        let item = AVPlayerItem(url: url)
        statusObservation = item.observe(\.status) { item, _ in
            if item.status == .failed { print("Failed: \(item.error?.localizedDescription ?? "")") }
        }
        player.replaceCurrentItem(with: item)
        timeObserver = player.addPeriodicTimeObserver(
            forInterval: CMTime(seconds: 0.5, preferredTimescale: 600),
            queue: .main) { time in print("Time: \(time.seconds)") }
    }

    deinit { if let obs = timeObserver { player.removeTimeObserver(obs) } }
}
```

## Audio Session Configuration

Configure AVAudioSession before capture or playback on iOS:

```swift
import AVFoundation

func configureAudioSession(for purpose: AudioPurpose) throws {
    let session = AVAudioSession.sharedInstance()
    switch purpose {
    case .playback:
        try session.setCategory(.playback, mode: .default)
    case .recording:
        try session.setCategory(.record, mode: .default)
    case .videoChat:
        try session.setCategory(.playAndRecord, mode: .videoChat,
                                options: [.defaultToSpeaker, .allowBluetooth])
    case .backgroundAudio:
        try session.setCategory(.playback, mode: .default, options: [.mixWithOthers])
    }
    try session.setActive(true)
}

enum AudioPurpose { case playback, recording, videoChat, backgroundAudio }
```

### Handling Audio Interruptions

```swift
NotificationCenter.default.addObserver(
    forName: AVAudioSession.interruptionNotification,
    object: AVAudioSession.sharedInstance(), queue: .main
) { notification in
    guard let info = notification.userInfo,
          let typeValue = info[AVAudioSessionInterruptionTypeKey] as? UInt,
          let type = AVAudioSession.InterruptionType(rawValue: typeValue) else { return }

    switch type {
    case .began:
        // Pause playback
        break
    case .ended:
        let opts = info[AVAudioSessionInterruptionOptionKey] as? UInt ?? 0
        if AVAudioSession.InterruptionOptions(rawValue: opts).contains(.shouldResume) {
            // Resume playback
        }
    @unknown default: break
    }
}
```

## Media Composition and Export

### Combining Video and Audio Tracks

```swift
func composeMedia(videoURL: URL, audioURL: URL) async throws -> AVMutableComposition {
    let composition = AVMutableComposition()
    let videoAsset = AVURLAsset(url: videoURL)
    let audioAsset = AVURLAsset(url: audioURL)

    let videoTracks = try await videoAsset.loadTracks(withMediaType: .video)
    let audioTracks = try await audioAsset.loadTracks(withMediaType: .audio)

    guard let srcVideo = videoTracks.first, let srcAudio = audioTracks.first else {
        throw NSError(domain: "Composition", code: -1)
    }

    let duration = try await videoAsset.load(.duration)

    let compVideoTrack = composition.addMutableTrack(
        withMediaType: .video, preferredTrackID: kCMPersistentTrackID_Invalid)
    try compVideoTrack?.insertTimeRange(
        CMTimeRange(start: .zero, duration: duration), of: srcVideo, at: .zero)

    let compAudioTrack = composition.addMutableTrack(
        withMediaType: .audio, preferredTrackID: kCMPersistentTrackID_Invalid)
    try compAudioTrack?.insertTimeRange(
        CMTimeRange(start: .zero, duration: duration), of: srcAudio, at: .zero)

    return composition
}
```

### Exporting with AVAssetExportSession

```swift
func exportComposition(_ composition: AVComposition, to outputURL: URL) async throws {
    try? FileManager.default.removeItem(at: outputURL)

    guard let session = AVAssetExportSession(
        asset: composition, presetName: AVAssetExportPresetHighestQuality
    ) else { throw NSError(domain: "Export", code: -1) }

    session.outputURL = outputURL
    session.outputFileType = .mp4
    await session.export()

    if session.status == .failed {
        throw session.error ?? NSError(domain: "Export", code: -2)
    }
}
```

## Privacy Permissions

### Required Info.plist Keys

| Key | Required When |
|-----|---------------|
| `NSCameraUsageDescription` | Using AVCaptureDevice for video |
| `NSMicrophoneUsageDescription` | Capturing audio input |
| `NSPhotoLibraryUsageDescription` | Saving to or reading from Photos |

### Requesting Permission

```swift
func requestCameraAccess() async -> Bool {
    switch AVCaptureDevice.authorizationStatus(for: .video) {
    case .authorized: return true
    case .notDetermined: return await AVCaptureDevice.requestAccess(for: .video)
    case .denied, .restricted: return false
    @unknown default: return false
    }
}
```

## Common Pitfalls and Patterns

### Session Threading

```swift
// ❌ Bad: blocking the main thread
captureSession.startRunning()

// ✅ Good: dedicated serial queue
let sessionQueue = DispatchQueue(label: "camera.session")
sessionQueue.async { captureSession.startRunning() }
```

### Lifecycle Balance

```swift
// ❌ Bad: not stopping session when view disappears
CameraPreview(session: manager.session)
    .onAppear { manager.start() }

// ✅ Good: balanced start/stop
CameraPreview(session: manager.session)
    .onAppear { manager.start() }
    .onDisappear { manager.stop() }
```

### Permission Before Configuration

```swift
// ❌ Bad: configuring without checking permission
func setupCamera() {
    let input = try! AVCaptureDeviceInput(device: camera)
    session.addInput(input)
}

// ✅ Good: check permission first
func setupCamera() async {
    guard await requestCameraAccess() else { return }
    sessionQueue.async { [self] in configureSession() }
}
```

### Audio Session Deactivation

```swift
// ❌ Bad: forgetting to deactivate audio session
func stopPlayback() { player.pause() }

// ✅ Good: deactivate to release audio resources
func stopPlayback() {
    player.pause()
    try? AVAudioSession.sharedInstance().setActive(false, options: .notifyOthersOnDeactivation)
}
```

### Time Observer Cleanup

```swift
// ❌ Bad: leaking time observers
player.addPeriodicTimeObserver(forInterval: interval, queue: .main) { _ in }

// ✅ Good: store and remove
private var timeObserver: Any?
func observe() {
    timeObserver = player.addPeriodicTimeObserver(forInterval: interval, queue: .main) { _ in }
}
deinit {
    if let obs = timeObserver { player.removeTimeObserver(obs) }
}
```
