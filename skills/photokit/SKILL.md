---
name: photokit
description: PhotoKit and PhotosPicker patterns for photo library access, asset fetching, image loading, and the SwiftUI PhotosPicker. Use when working with the user's photo library.
---

# PhotoKit & PhotosPicker

Patterns for accessing the photo library, picking media, fetching assets, loading images, and saving content. Covers SwiftUI PhotosPicker, PHPickerViewController, and full PHPhotoLibrary access.

## When This Skill Activates

- User needs to pick photos or videos from the library
- User asks about PhotosPicker, PHPickerViewController, or PHPhotoLibrary
- User needs to fetch, display, or cache photo assets
- User is saving images or videos to the photo library
- User asks about photo library permissions or limited access
- User needs to observe photo library changes
- User is building a gallery, image picker, or media browser

## Decision Tree: Which API to Use

```
Need to pick photos/videos from the user's library?
|
+-- SwiftUI app?
|   +-- YES --> PhotosPicker (iOS 16+, no permission needed)
|   +-- NO (UIKit) --> PHPickerViewController (iOS 14+, no permission needed)
|
Need to browse/fetch the full library programmatically?
|   +-- YES --> PHPhotoLibrary + PHAsset (requires authorization)
|
Only need to save photos/videos?
    +-- YES --> PHPhotoLibrary.shared().performChanges (addOnly access)
```

**Key insight:** PhotosPicker and PHPickerViewController run out-of-process. They do NOT require photo library permission. Only use PHPhotoLibrary when you need programmatic browsing, fetching, or observing changes.

## API Availability

| API | Minimum OS | Permission Required |
|-----|-----------|-------------------|
| `PhotosPicker` (SwiftUI) | iOS 16 | No |
| `PHPickerViewController` | iOS 14 | No |
| `PHPhotoLibrary` | iOS 8 | Yes (readWrite) |
| Limited Photo Access | iOS 14 | Yes (limited subset) |
| `PHPhotoLibrary.addOnly` | iOS 11 | Yes (addOnly) |

## Privacy Keys

Add to Info.plist as needed:

```xml
<!-- Required for PHPhotoLibrary read/write access -->
<key>NSPhotoLibraryUsageDescription</key>
<string>We need access to your photos to display your library.</string>

<!-- Required for save-only access (addOnly) -->
<key>NSPhotoLibraryAddUsageDescription</key>
<string>We need permission to save photos to your library.</string>
```

## SwiftUI PhotosPicker

### Basic Usage

```swift
import PhotosUI
import SwiftUI

struct PhotoPickerView: View {
    @State private var selectedItem: PhotosPickerItem?
    @State private var image: Image?

    var body: some View {
        VStack {
            PhotosPicker("Select a Photo", selection: $selectedItem, matching: .images)

            if let image {
                image
                    .resizable()
                    .scaledToFit()
                    .frame(height: 300)
            }
        }
        .onChange(of: selectedItem) { _, newItem in
            Task {
                if let data = try? await newItem?.loadTransferable(type: Data.self),
                   let uiImage = UIImage(data: data) {
                    image = Image(uiImage: uiImage)
                }
            }
        }
    }
}
```

### Filtering and Multi-Selection

```swift
// Multi-select with limit
PhotosPicker(
    "Select Photos",
    selection: $selectedItems,
    maxSelectionCount: 5,
    matching: .images
)

// Filter combinations
PhotosPicker(selection: $item, matching: .videos)
PhotosPicker(selection: $item, matching: .screenshots)
PhotosPicker(selection: $item, matching: .any(of: [.images, .videos]))
PhotosPicker(selection: $item, matching: .not(.videos))
```

### Loading via Transferable

```swift
// Load as Data
if let data = try? await item.loadTransferable(type: Data.self) {
    let uiImage = UIImage(data: data)
}

// Load as Image directly (macOS/iOS)
if let image = try? await item.loadTransferable(type: Image.self) {
    self.image = image
}

// Custom Transferable for controlled decoding
struct PickedImage: Transferable {
    let image: UIImage

    static var transferRepresentation: some TransferRepresentation {
        DataRepresentation(importedContentType: .image) { data in
            guard let image = UIImage(data: data) else {
                throw TransferError.importFailed
            }
            return PickedImage(image: image)
        }
    }
}
```

## PHPickerViewController (UIKit)

```swift
import PhotosUI

var config = PHPickerConfiguration()
config.filter = .images
config.selectionLimit = 3  // 0 = unlimited

let picker = PHPickerViewController(configuration: config)
picker.delegate = self
viewController.present(picker, animated: true)

// PHPickerViewControllerDelegate
func picker(_ picker: PHPickerViewController, didFinishPicking results: [PHPickerResult]) {
    picker.dismiss(animated: true)
    for result in results {
        if result.itemProvider.canLoadObject(ofClass: UIImage.self) {
            result.itemProvider.loadObject(ofClass: UIImage.self) { image, error in
                guard let image = image as? UIImage else { return }
                DispatchQueue.main.async { /* use image */ }
            }
        }
    }
}
```

## PHPhotoLibrary Authorization

### Requesting Access

```swift
import Photos

func requestPhotoAccess() async -> PHAuthorizationStatus {
    let status = await PHPhotoLibrary.requestAuthorization(for: .readWrite)

    switch status {
    case .authorized:
        // Full access to the photo library
        break
    case .limited:
        // User granted access to selected photos only (iOS 14+)
        break
    case .denied, .restricted:
        // No access; guide user to Settings
        break
    case .notDetermined:
        // Should not occur after requesting
        break
    @unknown default:
        break
    }

    return status
}
```

### Access Levels

```swift
// Read and write access (browsing + saving)
PHPhotoLibrary.requestAuthorization(for: .readWrite) { status in }

// Add-only access (saving photos without browsing)
PHPhotoLibrary.requestAuthorization(for: .addOnly) { status in }
```

## Fetching Assets

```swift
import Photos

// Fetch all images, sorted by creation date
let options = PHFetchOptions()
options.sortDescriptors = [NSSortDescriptor(key: "creationDate", ascending: false)]
options.predicate = NSPredicate(format: "mediaType == %d", PHAssetMediaType.image.rawValue)
options.fetchLimit = 50

let result: PHFetchResult<PHAsset> = PHAsset.fetchAssets(with: options)

// Enumerate results
result.enumerateObjects { asset, index, stop in
    print("Asset \(index): \(asset.localIdentifier)")
}

// Fetch a specific album
let albums = PHAssetCollection.fetchAssetCollections(with: .album, subtype: .any, options: nil)
if let album = albums.firstObject {
    let assets = PHAsset.fetchAssets(in: album, options: options)
}
```

## Loading Images with PHImageManager

```swift
let asset: PHAsset = // fetched asset
let manager = PHImageManager.default()

let options = PHImageRequestOptions()
options.deliveryMode = .highQualityFormat   // .fastFormat for thumbnails
options.resizeMode = .exact                 // .fast for approximate sizing
options.isNetworkAccessAllowed = true       // Allow downloading from iCloud
options.isSynchronous = false

let targetSize = CGSize(width: 300, height: 300)

manager.requestImage(
    for: asset,
    targetSize: targetSize,
    contentMode: .aspectFill,
    options: options
) { image, info in
    let isDegraded = info?[PHImageResultIsDegradedKey] as? Bool ?? false
    if !isDegraded, let image {
        // Final high-quality image is ready
    }
}
```

## PHCachingImageManager for Collections

```swift
final class PhotoGridViewModel {
    private let cachingManager = PHCachingImageManager()
    private var assets: PHFetchResult<PHAsset>?
    private let thumbnailSize = CGSize(width: 200, height: 200)

    func startCaching(for indexPaths: [IndexPath]) {
        guard let assets else { return }
        let assetsToCache = indexPaths.compactMap { assets.object(at: $0.item) }
        cachingManager.startCachingImages(
            for: assetsToCache,
            targetSize: thumbnailSize,
            contentMode: .aspectFill,
            options: nil
        )
    }

    func stopCaching(for indexPaths: [IndexPath]) {
        guard let assets else { return }
        let assetsToStop = indexPaths.compactMap { assets.object(at: $0.item) }
        cachingManager.stopCachingImages(
            for: assetsToStop,
            targetSize: thumbnailSize,
            contentMode: .aspectFill,
            options: nil
        )
    }

    func resetCache() {
        cachingManager.stopCachingImagesForAllAssets()
    }
}
```

## Observing Photo Library Changes

```swift
final class PhotoLibraryObserver: NSObject, PHPhotoLibraryChangeObserver {
    private var fetchResult: PHFetchResult<PHAsset>

    init(fetchResult: PHFetchResult<PHAsset>) {
        self.fetchResult = fetchResult
        super.init()
        PHPhotoLibrary.shared().register(self)
    }

    deinit {
        PHPhotoLibrary.shared().unregisterChangeObserver(self)
    }

    func photoLibraryDidChange(_ changeInstance: PHChange) {
        guard let changes = changeInstance.changeDetails(for: fetchResult) else { return }
        DispatchQueue.main.async { [weak self] in
            self?.fetchResult = changes.fetchResultAfterChanges
            // Use changes.removedIndexes, .insertedIndexes, .changedIndexes
            // for incremental UI updates, or reload if !hasIncrementalChanges
        }
    }
}
```

## Saving to the Photo Library

```swift
import Photos

func saveImageToLibrary(_ image: UIImage) async throws {
    try await PHPhotoLibrary.shared().performChanges {
        PHAssetChangeRequest.creationRequestForAsset(from: image)
    }
}

func saveVideoToLibrary(at url: URL) async throws {
    try await PHPhotoLibrary.shared().performChanges {
        PHAssetChangeRequest.creationRequestForAssetFromVideo(atFileURL: url)
    }
}
```

## Patterns

### Good Patterns

```swift
// ✅ Use PhotosPicker for simple selection (no permission needed)
PhotosPicker("Choose Photo", selection: $item, matching: .images)

// ✅ Check authorization before accessing PHPhotoLibrary
let status = await PHPhotoLibrary.requestAuthorization(for: .readWrite)
guard status == .authorized || status == .limited else { return }

// ✅ Use addOnly when you only need to save, not browse
PHPhotoLibrary.requestAuthorization(for: .addOnly) { status in }

// ✅ Use PHCachingImageManager for scrolling collections
let cachingManager = PHCachingImageManager()

// ✅ Handle the degraded callback from requestImage
manager.requestImage(for: asset, targetSize: size, contentMode: .aspectFill, options: nil) { image, info in
    let isDegraded = info?[PHImageResultIsDegradedKey] as? Bool ?? false
    if !isDegraded { /* use final image */ }
}

// ✅ Allow iCloud downloads when needed
let options = PHImageRequestOptions()
options.isNetworkAccessAllowed = true

// ✅ Unregister change observers in deinit
deinit { PHPhotoLibrary.shared().unregisterChangeObserver(self) }
```

### Bad Patterns

```swift
// ❌ Requesting full readWrite access just to pick a photo
PHPhotoLibrary.requestAuthorization(for: .readWrite) { _ in }
// Use PhotosPicker or PHPickerViewController instead

// ❌ Ignoring the degraded image callback
manager.requestImage(for: asset, targetSize: size, contentMode: .aspectFill, options: nil) { image, _ in
    self.image = image  // May set a low-quality placeholder
}

// ❌ Using PHImageManager.default() for large scrolling grids
// Use PHCachingImageManager for prefetching and cache management

// ❌ Forgetting isNetworkAccessAllowed for iCloud Photos users
let options = PHImageRequestOptions()
// options.isNetworkAccessAllowed defaults to false -- iCloud assets will fail

// ❌ Missing Info.plist privacy keys
// App will crash at runtime without NSPhotoLibraryUsageDescription

// ❌ Synchronous image loading on the main thread
let options = PHImageRequestOptions()
options.isSynchronous = true  // Blocks the calling thread
manager.requestImage(for: asset, targetSize: size, contentMode: .aspectFill, options: options) { _, _ in }
```
