---
name: healthkit
description: HealthKit patterns for reading/writing health data, workout sessions, background delivery, and HealthKit UI. Use when integrating health and fitness data in iOS or watchOS apps.
---

# HealthKit Integration

Patterns and guidance for reading, writing, and observing health and fitness data on iOS and watchOS using HealthKit.

## When This Skill Activates

Use this skill when the user:
- Asks about reading or writing health data (steps, heart rate, sleep, etc.)
- Wants to integrate HealthKit into an iOS or watchOS app
- Needs to request health data authorization
- Is building workout tracking features
- Asks about background delivery of health data
- Wants to display Activity Rings or health summaries
- Mentions `HKHealthStore`, `HKSampleQuery`, or `HKWorkoutSession`
- Needs incremental health data sync with anchored queries

## Decision Tree: Choosing the Right HealthKit API

```
Need health data?
├── One-time read → HKSampleQuery or HKStatisticsQuery
├── Aggregated over time → HKStatisticsCollectionQuery
├── Incremental updates (sync) → HKAnchoredObjectQuery
├── Real-time monitoring → HKObserverQuery + enableBackgroundDelivery
├── Tracking a workout → HKWorkoutSession + HKWorkoutBuilder
└── Writing data → HKQuantitySample / HKCategorySample → save()
```

## API Availability

| API | iOS | watchOS | Notes |
|-----|-----|---------|-------|
| HKHealthStore | 8.0+ | 2.0+ | Central access point |
| HKSampleQuery | 8.0+ | 2.0+ | One-shot data reads |
| HKStatisticsQuery | 8.0+ | 2.0+ | Sum, avg, min, max |
| HKStatisticsCollectionQuery | 8.0+ | 2.0+ | Time-bucketed stats |
| HKAnchoredObjectQuery | 8.0+ | 2.0+ | Incremental sync |
| HKObserverQuery | 8.0+ | 2.0+ | Change notifications |
| HKWorkoutSession | -- | 2.0+ | watchOS only |
| HKWorkoutBuilder | 11.0+ | 5.0+ | Construct workouts |
| HKLiveWorkoutBuilder | -- | 5.0+ | Real-time workout data |
| HKActivityRingView | -- | 7.0+ | watchOS Activity Rings |

## Privacy and Entitlements Setup

### Info.plist Keys (Required)

```xml
<!-- Reading health data -->
<key>NSHealthShareUsageDescription</key>
<string>We read your step count and heart rate to show fitness trends.</string>

<!-- Writing health data -->
<key>NSHealthUpdateUsageDescription</key>
<string>We save your workouts and nutrition logs to Apple Health.</string>
```

### Entitlements

Enable the HealthKit capability in Xcode, which adds:

```xml
<key>com.apple.developer.healthkit</key>
<true/>
<key>com.apple.developer.healthkit.access</key>
<array/>
```

For background delivery, also add:

```xml
<key>com.apple.developer.healthkit.background-delivery</key>
<true/>
```

### Patterns

```swift
// ✅ Always check availability before using HealthKit
guard HKHealthStore.isHealthDataAvailable() else {
    // Device does not support HealthKit (e.g., iPad)
    return
}
```

```swift
// ❌ Never assume HealthKit is available on all devices
let store = HKHealthStore() // Crashes on iPad if you proceed without checking
```

## Authorization

```swift
import HealthKit

final class HealthManager {
    private let store = HKHealthStore()

    func requestAuthorization() async throws {
        let readTypes: Set<HKObjectType> = [
            HKQuantityType(.stepCount),
            HKQuantityType(.heartRate),
            HKQuantityType(.activeEnergyBurned),
            HKCategoryType(.sleepAnalysis)
        ]

        let writeTypes: Set<HKSampleType> = [
            HKQuantityType(.stepCount),
            HKQuantityType(.activeEnergyBurned)
        ]

        try await store.requestAuthorization(toShare: writeTypes, read: readTypes)
    }
}
```

### Checking Authorization Status

```swift
// ✅ Check write authorization (HealthKit only reveals write status for privacy)
let status = store.authorizationStatus(for: HKQuantityType(.stepCount))
switch status {
case .notDetermined:
    // Request authorization
    break
case .sharingDenied:
    // User denied write access
    break
case .sharingAuthorized:
    // Write access granted
    break
@unknown default:
    break
}
```

```swift
// ❌ Don't try to check read authorization — HealthKit intentionally hides it
// There is no API to check if the user granted read access.
// Queries simply return no data if read access was denied.
```

## Reading Data

### HKSampleQuery — One-Time Read

```swift
func fetchRecentHeartRates() async throws -> [HKQuantitySample] {
    let heartRateType = HKQuantityType(.heartRate)
    let sortDescriptor = SortDescriptor(\HKSample.startDate, order: .reverse)
    let predicate = HKQuery.predicateForSamples(
        withStart: Calendar.current.date(byAdding: .hour, value: -24, to: .now),
        end: .now,
        options: .strictStartDate
    )

    let query = HKSampleQueryDescriptor(
        predicates: [.quantitySample(type: heartRateType, predicate: predicate)],
        sortDescriptors: [sortDescriptor],
        limit: 100
    )

    return try await query.result(for: store)
}
```

### HKStatisticsQuery — Aggregated Value

```swift
func fetchTodaySteps() async throws -> Double {
    let stepsType = HKQuantityType(.stepCount)
    let startOfDay = Calendar.current.startOfDay(for: .now)
    let predicate = HKQuery.predicateForSamples(
        withStart: startOfDay, end: .now, options: .strictStartDate
    )

    let query = HKStatisticsQueryDescriptor(
        predicate: .quantitySample(type: stepsType, predicate: predicate),
        options: .cumulativeSum
    )

    let result = try await query.result(for: store)
    return result?.sumQuantity()?.doubleValue(for: .count()) ?? 0
}
```

### HKStatisticsCollectionQuery — Time-Bucketed Aggregation

```swift
func fetchWeeklySteps() async throws -> [(date: Date, steps: Double)] {
    let stepsType = HKQuantityType(.stepCount)
    let calendar = Calendar.current
    let endDate = Date.now
    let startDate = calendar.date(byAdding: .day, value: -7, to: endDate)!

    let query = HKStatisticsCollectionQueryDescriptor(
        predicate: .quantitySample(type: stepsType),
        options: .cumulativeSum,
        anchorDate: calendar.startOfDay(for: endDate),
        intervalComponents: DateComponents(day: 1)
    )

    let collection = try await query.result(for: store)
    var results: [(date: Date, steps: Double)] = []

    collection.enumerateStatistics(from: startDate, to: endDate) { statistics, _ in
        let steps = statistics.sumQuantity()?.doubleValue(for: .count()) ?? 0
        results.append((date: statistics.startDate, steps: steps))
    }
    return results
}
```

## Writing Data

### Saving a Quantity Sample

```swift
func saveSteps(count: Double, start: Date, end: Date) async throws {
    let stepsType = HKQuantityType(.stepCount)
    let quantity = HKQuantity(unit: .count(), doubleValue: count)
    let sample = HKQuantitySample(
        type: stepsType,
        quantity: quantity,
        start: start,
        end: end
    )

    try await store.save(sample)
}
```

### Saving a Category Sample (Sleep)

```swift
func saveSleepAnalysis(start: Date, end: Date) async throws {
    let sleepType = HKCategoryType(.sleepAnalysis)
    let sample = HKCategorySample(
        type: sleepType,
        value: HKCategoryValueSleepAnalysis.asleepCore.rawValue,
        start: start,
        end: end
    )

    try await store.save(sample)
}
```

## Common Quantity Types and Units

| Type | Identifier | Unit | Statistics Option |
|------|-----------|------|-------------------|
| Steps | `.stepCount` | `.count()` | `.cumulativeSum` |
| Heart Rate | `.heartRate` | `.count()/.minute()` | `.discreteAverage` |
| Active Energy | `.activeEnergyBurned` | `.kilocalorie()` | `.cumulativeSum` |
| Distance Walking | `.distanceWalkingRunning` | `.meter()` | `.cumulativeSum` |
| Sleep | `.sleepAnalysis` | Category type | N/A |
| Body Mass | `.bodyMass` | `.gramUnit(with: .kilo)` | `.discreteAverage` |
| Blood Oxygen | `.oxygenSaturation` | `.percent()` | `.discreteAverage` |

## Workout Sessions (watchOS)

### Building a Workout with HKWorkoutBuilder

```swift
import HealthKit

func startWorkout() async throws {
    let configuration = HKWorkoutConfiguration()
    configuration.activityType = .running
    configuration.locationType = .outdoor

    let builder = HKWorkoutBuilder(healthStore: store, configuration: configuration, device: .local())
    try await builder.beginCollection(at: .now)

    // Add samples during the workout
    let heartRateType = HKQuantityType(.heartRate)
    let heartRateSample = HKQuantitySample(
        type: heartRateType,
        quantity: HKQuantity(unit: .count().unitDivided(by: .minute()), doubleValue: 145),
        start: .now,
        end: .now
    )
    try await builder.addSamples([heartRateSample])

    // End and save
    try await builder.endCollection(at: .now)
    try await builder.finishWorkout()
}
```

### HKWorkoutSession + HKLiveWorkoutBuilder (watchOS)

```swift
import HealthKit

class WorkoutManager: NSObject, HKWorkoutSessionDelegate, HKLiveWorkoutBuilderDelegate {
    private let store = HKHealthStore()
    private var session: HKWorkoutSession?
    private var builder: HKLiveWorkoutBuilder?

    func startLiveWorkout() async throws {
        let configuration = HKWorkoutConfiguration()
        configuration.activityType = .cycling
        configuration.locationType = .outdoor

        session = try HKWorkoutSession(healthStore: store, configuration: configuration)
        builder = session?.associatedWorkoutBuilder()

        session?.delegate = self
        builder?.delegate = self
        builder?.dataSource = HKLiveWorkoutDataSource(healthStore: store, workoutConfiguration: configuration)

        let startDate = Date.now
        session?.startActivity(with: startDate)
        try await builder?.beginCollection(at: startDate)
    }

    func endLiveWorkout() async throws {
        let endDate = Date.now
        session?.end()
        try await builder?.endCollection(at: endDate)
        try await builder?.finishWorkout()
    }

    // MARK: - HKWorkoutSessionDelegate
    func workoutSession(_ workoutSession: HKWorkoutSession, didChangeTo toState: HKWorkoutSessionState,
                        from fromState: HKWorkoutSessionState, date: Date) {
        // Handle state transitions
    }

    func workoutSession(_ workoutSession: HKWorkoutSession, didFailWithError error: Error) {
        // Handle errors
    }

    // MARK: - HKLiveWorkoutBuilderDelegate
    func workoutBuilderDidCollectEvent(_ workoutBuilder: HKLiveWorkoutBuilder) { }

    func workoutBuilder(_ workoutBuilder: HKLiveWorkoutBuilder, didCollectDataOf collectedTypes: Set<HKSampleType>) {
        for type in collectedTypes {
            guard let quantityType = type as? HKQuantityType else { continue }
            if let statistics = workoutBuilder.statistics(for: quantityType) {
                // Update UI with latest statistics (heart rate, calories, distance)
            }
        }
    }
}
```

## Background Delivery

```swift
func enableStepCountBackgroundDelivery() {
    let stepsType = HKQuantityType(.stepCount)

    store.enableBackgroundDelivery(for: stepsType, frequency: .hourly) { success, error in
        if let error {
            print("Background delivery setup failed: \(error.localizedDescription)")
        }
    }

    let query = HKObserverQuery(sampleType: stepsType, predicate: nil) { [weak self] _, completionHandler, error in
        guard error == nil else {
            completionHandler()
            return
        }
        // Fetch new data and update your app
        Task {
            // Perform data fetch here
        }
        completionHandler() // Must call to let HealthKit know you're done
    }

    store.execute(query)
}
```

```swift
// ✅ Always call the completion handler in observer queries
store.execute(HKObserverQuery(sampleType: type, predicate: nil) { _, completionHandler, _ in
    defer { completionHandler() }
    // Process update
})
```

```swift
// ❌ Forgetting to call completionHandler stops future deliveries
store.execute(HKObserverQuery(sampleType: type, predicate: nil) { _, completionHandler, _ in
    // Process update but never call completionHandler — background delivery breaks
})
```

## HKAnchoredObjectQuery — Incremental Updates

```swift
func fetchNewSamples(type: HKSampleType, anchor: HKQueryAnchor?) async throws -> (samples: [HKSample], anchor: HKQueryAnchor) {
    return try await withCheckedThrowingContinuation { continuation in
        let query = HKAnchoredObjectQuery(
            type: type,
            predicate: nil,
            anchor: anchor,
            limit: HKObjectQueryNoLimit
        ) { _, added, deleted, newAnchor, error in
            if let error {
                continuation.resume(throwing: error)
                return
            }
            continuation.resume(returning: (samples: added ?? [], anchor: newAnchor!))
        }
        store.execute(query)
    }
}
```

```swift
// ✅ Persist the anchor between app launches for true incremental sync
let anchorData = try NSKeyedArchiver.archivedData(withRootObject: newAnchor, requiringSecureCoding: true)
UserDefaults.standard.set(anchorData, forKey: "stepCountAnchor")
```

```swift
// ❌ Re-querying all data every time wastes resources and battery
let query = HKAnchoredObjectQuery(type: type, predicate: nil, anchor: nil, limit: HKObjectQueryNoLimit) { ... }
// Passing nil anchor fetches everything from scratch
```

## HealthKit UI — Activity Ring View (watchOS)

```swift
import HealthKitUI
import SwiftUI

struct ActivityRingsView: WKInterfaceObjectRepresentable {
    let store: HKHealthStore

    func makeWKInterfaceObject(context: Context) -> some WKInterfaceObject {
        let ring = WKInterfaceActivityRing()
        let query = HKActivitySummaryQuery(predicate: todayPredicate()) { _, summaries, _ in
            if let summary = summaries?.first {
                ring.setActivitySummary(summary, animated: true)
            }
        }
        store.execute(query)
        return ring
    }

    func updateWKInterfaceObject(_ ring: WKInterfaceObjectType, context: Context) { }

    private func todayPredicate() -> NSPredicate {
        let calendar = Calendar.current
        let components = calendar.dateComponents([.year, .month, .day], from: .now)
        return HKQuery.predicateForActivitySummary(with: components)
    }
}
```

## Good and Bad Patterns Summary

```swift
// ✅ Request only the types you actually need
let readTypes: Set<HKObjectType> = [HKQuantityType(.stepCount)]
try await store.requestAuthorization(toShare: [], read: readTypes)
```

```swift
// ❌ Requesting every type — users distrust broad health data requests
let readTypes: Set<HKObjectType> = [
    HKQuantityType(.stepCount), HKQuantityType(.heartRate),
    HKQuantityType(.bloodGlucose), HKQuantityType(.bodyMass),
    HKQuantityType(.oxygenSaturation), HKQuantityType(.bodyTemperature),
    // ... dozens more
]
```

```swift
// ✅ Use async/await descriptor APIs on iOS 15.4+ / watchOS 8.5+
let descriptor = HKSampleQueryDescriptor(
    predicates: [.quantitySample(type: HKQuantityType(.stepCount))],
    sortDescriptors: [SortDescriptor(\.startDate, order: .reverse)],
    limit: 10
)
let samples = try await descriptor.result(for: store)
```

```swift
// ❌ Using legacy completion-handler queries when async versions are available
store.execute(HKSampleQuery(sampleType: type, predicate: nil, limit: 10,
    sortDescriptors: nil) { _, samples, error in
    // Callback hell, harder to manage errors
})
```

```swift
// ✅ Handle the case where HealthKit data is unavailable gracefully
func getSteps() async -> Double {
    guard HKHealthStore.isHealthDataAvailable() else { return 0 }
    do {
        return try await fetchTodaySteps()
    } catch {
        return 0 // Degrade gracefully
    }
}
```

```swift
// ❌ Force-unwrapping HealthKit results
let steps = result!.sumQuantity()!.doubleValue(for: .count()) // Crash if nil
```
