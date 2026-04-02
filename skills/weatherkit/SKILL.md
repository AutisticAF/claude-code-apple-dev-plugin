---
name: weatherkit
description: WeatherKit patterns for current conditions, forecasts, weather alerts, and attribution. Use when integrating weather data from Apple's weather service.
---

> **First step:** Tell the user: "weatherkit skill loaded."

# WeatherKit

Apple's framework for accessing weather data including current conditions, forecasts, severe weather alerts, and historical weather. Available as a native Swift API and a REST API.

## When This Skill Activates

Use this skill when the user:
- Wants to fetch current weather conditions or forecasts
- Needs hourly, daily, or minute-by-minute precipitation data
- Asks about weather alerts or severe weather notifications
- Needs to display weather data with proper Apple attribution
- Asks about WeatherKit setup, entitlements, or rate limits
- Wants to choose between the Swift API and the REST API
- Needs to check weather data availability by region

## Decision Tree: Swift API vs REST API

| Criteria | Swift API | REST API |
|----------|-----------|----------|
| Platform | iOS 16+, macOS 13+, watchOS 9+, tvOS 16+ | Any platform (web, Android, server) |
| Auth | Automatic via entitlement | JWT signed with App Store Connect key |
| Language | Swift only | Any (JSON responses) |
| Attribution | `WeatherAttribution` built-in | Must implement manually |
| Recommended | Apple-platform apps | Cross-platform or server-side |

**Rule of thumb:** Use the Swift API for Apple-platform apps. Use the REST API only when you need weather data outside Apple ecosystems.

## API Availability

| Platform | Minimum Version |
|----------|----------------|
| iOS | 16+ |
| macOS | 13+ |
| watchOS | 9+ |
| tvOS | 16+ |

All datasets (current, minute, hourly, daily, alerts, availability) share the same minimum versions.

## Setup

### 1. Enable WeatherKit Capability

In Xcode: Target > Signing & Capabilities > + Capability > WeatherKit.

This adds the `com.apple.developer.weatherkit` entitlement to your app.

### 2. Enable in App Store Connect

Go to Certificates, Identifiers & Profiles > Identifiers > your App ID > App Services, and enable **WeatherKit**.

### 3. Import the Framework

```swift
import WeatherKit
import CoreLocation
```

## WeatherService: Fetching Weather Data

Use the shared `WeatherService` singleton. All calls are async and require a `CLLocation`.

```swift
let weatherService = WeatherService.shared
let location = CLLocation(latitude: 37.7749, longitude: -122.4194)

// Fetch specific datasets (recommended -- each dataset counts as one API call)
let (current, daily, hourly) = try await weatherService.weather(
    for: location,
    including: .current, .daily, .hourly
)
```

## Current Weather

`CurrentWeather` provides a snapshot of conditions right now.

```swift
let current = try await weatherService.weather(
    for: location,
    including: .current
)

let temperature = current.temperature              // Measurement<UnitTemperature>
let condition = current.condition                   // WeatherCondition enum
let humidity = current.humidity                     // 0.0...1.0
let uvIndex = current.uvIndex                      // UVIndex (.value, .category)
let wind = current.wind                            // Wind (.speed, .direction, .gust)
let isDaylight = current.isDaylight                // Bool
```

## Hourly Forecast

`Forecast<HourWeather>` provides hour-by-hour data, typically up to 240 hours.

```swift
let hourly = try await weatherService.weather(
    for: location,
    including: .hourly
)

for hour in hourly.forecast.prefix(24) {
    let date = hour.date
    let temp = hour.temperature
    let condition = hour.condition
    let precipChance = hour.precipitationChance    // 0.0...1.0
    let windSpeed = hour.wind.speed
}
```

## Daily Forecast

`Forecast<DayWeather>` provides daily summaries, typically up to 10 days.

```swift
let daily = try await weatherService.weather(for: location, including: .daily)

for day in daily.forecast {
    let highTemp = day.highTemperature
    let lowTemp = day.lowTemperature
    let precipChance = day.precipitationChance     // 0.0...1.0
    let sunrise = day.sun.sunrise                  // Date?
    let sunset = day.sun.sunset                    // Date?
}
```

## Minute Forecast

`Forecast<MinuteWeather>` provides precipitation intensity minute-by-minute for the next hour. **Only available in select regions** -- always check availability first.

```swift
if let minuteForecast = try await weatherService.weather(for: location, including: .minute) {
    for entry in minuteForecast.forecast {
        let intensity = entry.precipitationIntensity  // Measurement<UnitSpeed>
        let chance = entry.precipitationChance
    }
    let summary = minuteForecast.summary              // Localized summary string
}
```

## Weather Alerts

`WeatherAlert` provides government-issued severe weather warnings for a location.

```swift
if let alerts = try await weatherService.weather(for: location, including: .alerts) {
    for alert in alerts {
        let severity = alert.severity       // .minor, .moderate, .severe, .extreme
        let summary = alert.summary         // Localized description
        let region = alert.region           // Affected area name
        let detailsURL = alert.detailsURL   // URL to full advisory
    }
}
```

## Checking Data Availability

Not all datasets are available in every region. Always check before requesting.

```swift
let availability = try await weatherService.availability(for: location)
if availability.contains(.minuteForecast) {
    let minute = try await weatherService.weather(for: location, including: .minute)
}
```

## Attribution Requirements

**Apple requires visible attribution** whenever you display WeatherKit data. Failure to include attribution can result in App Review rejection.

```swift
let attribution = try await weatherService.attribution

let legalPageURL = attribution.legalPageURL         // Must be accessible to users
let markURL = attribution.combinedMarkDarkURL       // Apple Weather logo (dark mode)
let markLightURL = attribution.combinedMarkLightURL // Apple Weather logo (light mode)
```

### SwiftUI Attribution View

```swift
struct WeatherAttributionView: View {
    @Environment(\.colorScheme) var colorScheme
    @State private var attribution: WeatherAttribution?

    var body: some View {
        if let attribution {
            VStack {
                AsyncImage(url: colorScheme == .dark
                    ? attribution.combinedMarkDarkURL
                    : attribution.combinedMarkLightURL
                ) { image in
                    image.resizable().scaledToFit()
                } placeholder: { ProgressView() }
                .frame(height: 20)
                Link("Legal", destination: attribution.legalPageURL).font(.caption2)
            }
        }
    }
}
```

## SwiftUI Display Patterns

### Current Conditions Card

```swift
struct CurrentWeatherView: View {
    let weather: CurrentWeather

    var body: some View {
        VStack(spacing: 8) {
            Image(systemName: weather.symbolName)
                .font(.system(size: 48))
                .symbolRenderingMode(.multicolor)
            Text(weather.temperature.formatted(
                .measurement(width: .abbreviated, usage: .weather)
            ))
            .font(.system(size: 52, weight: .thin))
            Text(weather.condition.description).font(.headline)
            HStack(spacing: 16) {
                Label(weather.humidity.formatted(.percent), systemImage: "humidity")
                Label("UV \(weather.uvIndex.value)", systemImage: "sun.max")
            }
            .font(.subheadline)
        }
    }
}
```

### Hourly Forecast Row

```swift
struct HourlyForecastRow: View {
    let hours: [HourWeather]

    var body: some View {
        ScrollView(.horizontal, showsIndicators: false) {
            HStack(spacing: 16) {
                ForEach(hours, id: \.date) { hour in
                    VStack(spacing: 6) {
                        Text(hour.date.formatted(.dateTime.hour()))
                        Image(systemName: hour.symbolName)
                            .symbolRenderingMode(.multicolor)
                        Text(hour.temperature.formatted(
                            .measurement(width: .narrow, usage: .weather)
                        ))
                    }
                    .font(.caption)
                }
            }
            .padding(.horizontal)
        }
    }
}
```

## Rate Limits and Billing

- **Free tier:** 500,000 calls/month per Apple Developer account
- Each `weather(for:including:)` call with N datasets counts as **N API calls**
- Request only the datasets you need to minimize usage
- Cache responses locally; weather data is typically valid for 5-15 minutes
- Monitor usage in App Store Connect under WeatherKit metrics

## Patterns

### ✅ Good Patterns

```swift
// ✅ Request only needed datasets
let (current, daily) = try await weatherService.weather(
    for: location,
    including: .current, .daily
)

// ✅ Check availability before requesting regional data
let availability = try await weatherService.availability(for: location)
if availability.contains(.minuteForecast) {
    let minute = try await weatherService.weather(for: location, including: .minute)
}

// ✅ Cache weather data to reduce API calls (10-15 min TTL is reasonable)

// ✅ Always include attribution when displaying weather data
WeatherAttributionView()
```

### ❌ Bad Patterns

```swift
// ❌ Fetching all datasets when you only need current conditions
let weather = try await weatherService.weather(for: location)

// ❌ Polling weather data every few seconds
Timer.scheduledTimer(withTimeInterval: 5, repeats: true) { _ in
    Task { let _ = try? await weatherService.weather(for: location, including: .current) }
}

// ❌ Ignoring that minute forecast may not be available
let minute = try await weatherService.weather(for: location, including: .minute)
// This returns nil in unsupported regions - always check availability first

// ❌ Displaying weather data without attribution
// Apple requires the Apple Weather mark and legal link
```
