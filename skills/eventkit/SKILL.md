---
name: eventkit
description: EventKit patterns for calendar events, reminders, and the EventKitUI views. Use when integrating calendar or reminder functionality.
---

# EventKit Development Guide

Patterns for working with calendar events and reminders using EventKit and EventKitUI on Apple platforms.

## When This Skill Activates

Use this skill when the user:
- Wants to read, create, or modify calendar events
- Needs to work with reminders (fetch, create, complete)
- Asks about EKEventStore, EKEvent, EKReminder, or EKCalendar
- Wants to present system calendar/reminder UI (EventKitUI)
- Needs recurrence rules, alarms, or availability settings
- Asks about calendar/reminder permissions (iOS 17+ full access model)
- Wants to observe calendar database changes
- Asks about virtual conference providers

## Decision Tree: EventKit vs EventKitUI vs CalendarKit

| Framework | Use When |
|-----------|----------|
| **EventKit** | Reading/writing events and reminders programmatically |
| **EventKitUI** | Presenting Apple's built-in event viewing, editing, or calendar chooser UI |
| **CalendarKit** | Building a fully custom calendar UI (third-party package, not Apple) |

Use **EventKit** alone for background sync or data queries. Use **EventKitUI** for standard system UI. Combine both when querying data programmatically and displaying with system UI.

## API Availability

| API | iOS | macOS | watchOS | visionOS |
|-----|-----|-------|---------|----------|
| EKEventStore | 4.0+ | 10.8+ | -- | 1.0+ |
| EKEvent | 4.0+ | 10.8+ | -- | 1.0+ |
| EKReminder | 6.0+ | 10.8+ | -- | 1.0+ |
| EKEventEditViewController | 4.0+ | -- | -- | -- |
| EKEventViewController | 4.0+ | -- | -- | 1.0+ |
| EKCalendarChooser | 5.0+ | -- | -- | 1.0+ |
| Full Access / Write-Only (iOS 17+) | 17.0+ | 14.0+ | -- | 1.0+ |
| EKVirtualConferenceProvider | 15.0+ | 12.0+ | -- | -- |

## EKEventStore: Requesting Access

### iOS 17+ Authorization Model

iOS 17 replaced the single calendar permission with two tiers:

| Access Level | Reads Events | Writes Events | Info.plist Key |
|--------------|-------------|---------------|----------------|
| **Full access** | Yes | Yes | `NSCalendarsFullAccessUsageDescription` |
| **Write-only** | No | Yes | `NSCalendarsWriteOnlyAccessUsageDescription` |
| **Reminders** | Yes | Yes | `NSRemindersFullAccessUsageDescription` |

### Requesting Access

```swift
import EventKit

let store = EKEventStore()

// Full calendar access (read + write)
func requestFullCalendarAccess() async throws -> Bool {
    if #available(iOS 17.0, *) {
        return try await store.requestFullAccessToEvents()
    } else {
        return try await store.requestAccess(to: .event)
    }
}

// Write-only calendar access (no reading)
@available(iOS 17.0, *)
func requestWriteOnlyAccess() async throws -> Bool {
    return try await store.requestWriteOnlyAccessToEvents()
}

// Reminder access
func requestReminderAccess() async throws -> Bool {
    if #available(iOS 17.0, *) {
        return try await store.requestFullAccessToReminders()
    } else {
        return try await store.requestAccess(to: .reminder)
    }
}
```

### Checking Authorization Status

```swift
let status = EKEventStore.authorizationStatus(for: .event)
// Returns: .notDetermined, .fullAccess (iOS 17+), .writeOnly (iOS 17+),
//          .authorized (pre-iOS 17), .restricted, .denied
```

## Reading Calendars

```swift
// All calendars for events
let eventCalendars = store.calendars(for: .event)

// All calendars for reminders
let reminderCalendars = store.calendars(for: .reminder)

// Filter by source type (.local, .exchange, .calDAV, .subscribed, .birthdays)
let iCloudCalendars = eventCalendars.filter { $0.source?.sourceType == .calDAV }

// Filter by type
let birthdayCalendar = eventCalendars.first { $0.type == .birthday }

// Default calendars
let defaultCalendar = store.defaultCalendarForNewEvents
let defaultReminderList = store.defaultCalendarForNewReminders()
```

## Creating and Modifying Events

```swift
let event = EKEvent(eventStore: store)
event.title = "Team Standup"
event.startDate = Date()
event.endDate = Calendar.current.date(byAdding: .hour, value: 1, to: Date())!
event.calendar = store.defaultCalendarForNewEvents
event.location = "Conference Room A"
event.notes = "Weekly sync"
event.availability = .busy  // .free, .busy, .tentative, .unavailable
event.isAllDay = false

// Alarm (negative offset = before event)
event.addAlarm(EKAlarm(relativeOffset: -600)) // 10 min before

// Recurrence rule (weekly on Mon, Wed, Fri for 52 weeks)
let rule = EKRecurrenceRule(
    recurrenceWith: .weekly, interval: 1,
    daysOfTheWeek: [.monday, .wednesday, .friday].map { EKRecurrenceDayOfWeek($0) },
    daysOfTheMonth: nil, monthsOfTheYear: nil,
    weeksOfTheYear: nil, daysOfTheYear: nil, setPositions: nil,
    end: EKRecurrenceEnd(occurrenceCount: 52)
)
event.addRecurrenceRule(rule)

try store.save(event, span: .thisEvent, commit: true)
// span: .thisEvent (single occurrence) or .futureEvents (this + all future)
try store.remove(event, span: .thisEvent, commit: true)
```

## Querying Events

```swift
let startDate = Calendar.current.startOfDay(for: Date())
let endDate = Calendar.current.date(byAdding: .month, value: 1, to: startDate)!

// Create predicate (max 4-year range)
let predicate = store.predicateForEvents(
    withStart: startDate,
    end: endDate,
    calendars: nil // nil = all calendars
)

// Option A: fetch all matching events (returns sorted array)
let events = store.events(matching: predicate)

// Option B: enumerate for large result sets
store.enumerateEvents(matching: predicate) { event, stop in
    if event.title.contains("Standup") {
        // Process event
        stop.pointee = true // stop early if needed
    }
}
```

## Reminders

```swift
let reminder = EKReminder(eventStore: store)
reminder.title = "Buy groceries"
reminder.calendar = store.defaultCalendarForNewReminders()
reminder.priority = 1  // 1-4 high, 5 medium, 6-9 low, 0 none
reminder.dueDateComponents = Calendar.current.dateComponents(
    [.year, .month, .day, .hour, .minute],
    from: Date().addingTimeInterval(86400)
)
reminder.addAlarm(EKAlarm(absoluteDate: Date().addingTimeInterval(86400)))
try store.save(reminder, commit: true)

// Fetch incomplete reminders (callback fires on arbitrary queue)
let predicate = store.predicateForIncompleteReminders(
    withDueDateStarting: nil,
    ending: Date().addingTimeInterval(7 * 86400),
    calendars: nil
)
store.fetchReminders(matching: predicate) { reminders in
    guard let reminders else { return }
    for r in reminders { print(r.title ?? "Untitled") }
}

// Complete a reminder
reminder.isCompleted = true  // sets completionDate automatically
try store.save(reminder, commit: true)
```

## EventKitUI View Controllers

```swift
import EventKitUI

// --- View an event ---
let eventVC = EKEventViewController()
eventVC.event = event
eventVC.allowsEditing = true
eventVC.delegate = self // EKEventViewDelegate
navigationController?.pushViewController(eventVC, animated: true)

// --- Create/edit an event ---
let editVC = EKEventEditViewController()
editVC.eventStore = store
editVC.event = event               // nil for a new event
editVC.editViewDelegate = self     // EKEventEditViewDelegate
present(editVC, animated: true)

// --- Calendar chooser ---
let chooser = EKCalendarChooser(
    selectionStyle: .multiple,
    displayStyle: .allCalendars,
    entityType: .event,
    eventStore: store
)
chooser.showsDoneButton = true
chooser.showsCancelButton = true
chooser.delegate = self            // EKCalendarChooserDelegate
present(UINavigationController(rootViewController: chooser), animated: true)
```

Delegate callbacks:

```swift
// EKEventViewDelegate
func eventViewController(_ controller: EKEventViewController,
                         didCompleteWith action: EKEventViewAction) {
    controller.dismiss(animated: true)
}

// EKEventEditViewDelegate
func eventEditViewController(_ controller: EKEventEditViewController,
                             didCompleteWith action: EKEventEditViewAction) {
    controller.dismiss(animated: true)
}

// EKCalendarChooserDelegate
func calendarChooserDidFinish(_ calendarChooser: EKCalendarChooser) {
    let selected = calendarChooser.selectedCalendars
    calendarChooser.dismiss(animated: true)
}
```

## Observing Changes

Listen for `EKEventStoreChangedNotification` to detect external modifications (other apps, sync). Re-fetch all cached data when received -- object identifiers may have changed.

```swift
NotificationCenter.default.addObserver(
    self, selector: #selector(storeChanged(_:)),
    name: .EKEventStoreChanged, object: store
)

@objc func storeChanged(_ notification: Notification) {
    // Re-fetch events and reminders with fresh predicates
}
```

## Privacy: Info.plist Keys

| Key | Required For |
|-----|-------------|
| `NSCalendarsFullAccessUsageDescription` | Read + write events (iOS 17+) |
| `NSCalendarsWriteOnlyAccessUsageDescription` | Write events only (iOS 17+) |
| `NSRemindersFullAccessUsageDescription` | Read + write reminders (iOS 17+) |
| `NSCalendarsUsageDescription` | Calendar access (pre-iOS 17) |
| `NSRemindersUsageDescription` | Reminder access (pre-iOS 17) |

For apps supporting both iOS 17+ and older versions, include both old and new keys.

## Virtual Conference Provider

Subclass `EKVirtualConferenceProvider` (iOS 15+) and register it in `Info.plist` under the `EKVirtualConferenceProvider` key.

```swift
@available(iOS 15.0, *)
class MyConferenceProvider: EKVirtualConferenceProvider {
    override func fetchVirtualConference(
        for identifier: EKVirtualConferenceDescriptor.Identifier
    ) async throws -> EKVirtualConference {
        let url = EKVirtualConferenceURLDescriptor(
            title: "Join Call",
            url: URL(string: "https://meet.example.com/\(identifier.rawValue)")!
        )
        return EKVirtualConference(title: "My App Meeting",
                                   urlDescriptors: [url],
                                   conferenceDetails: "Tap the link to join.")
    }
}
```

## Patterns

### Good Patterns

```swift
// ✅ Reuse a single EKEventStore instance across your app
class CalendarManager {
    static let shared = CalendarManager()
    let store = EKEventStore()
}

// ✅ Check authorization before every operation
func addEvent() async throws {
    guard try await requestFullCalendarAccess() else { throw CalendarError.accessDenied }
}

// ✅ Commit in batches when saving multiple items
for event in events { try store.save(event, span: .thisEvent, commit: false) }
try store.commit()

// ✅ Use write-only access when you only need to create events
```

### Bad Patterns

```swift
// ❌ Creating a new EKEventStore for every operation -- wasteful, loses change tracking
func fetchEvents() { let store = EKEventStore() }

// ❌ Querying events with a range exceeding 4 years -- returns no results or crashes
store.predicateForEvents(withStart: .distantPast, end: .distantFuture, calendars: nil)

// ❌ Ignoring authorization status -- events(matching:) returns empty with no error
// ❌ Requesting full access when write-only is sufficient -- hurts user trust
// ❌ Updating UI directly from fetchReminders callback -- fires on arbitrary queue
```
