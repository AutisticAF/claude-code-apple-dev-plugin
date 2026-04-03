# Claude Code Apple Dev Plugin

A Claude Code plugin with 166 skills for Apple platform development — iOS, macOS, iPadOS, watchOS, tvOS, and visionOS. Covers everything from product ideation to App Store submission, with code generators, architecture patterns, framework guides, and workflow tools.

This plugin is heavily based on the skills from[claude-code-apple-skills](https://github.com/rshankras/claude-code-apple-skills). I added a bunch around building CLIs, creating macros, using AVFoundation and other first-party libraries, etc. I also re-structured the skills according to Anthropic's [Claude Code Plugins reference](https://code.claude.com/docs/en/plugins-reference). And lastly, I made it into an easy-to-install plugin with an `apple-dev:dev` agent that's built to use these skills.

[![License: PolyForm Noncommercial](https://img.shields.io/badge/License-PolyForm%20Noncommercial-blue.svg)](https://polyformproject.org/licenses/noncommercial/1.0.0)

## Installation

### As a Plugin (recommended)

```bash
# Install from GitHub
claude plugin install github:AutisticAF/claude-code-apple-dev-plugin

# Skills are available as /apple-dev:skill-name in any project
```

### Local Development

```bash
git clone https://github.com/AutisticAF/claude-code-apple-dev-plugin.git
claude --plugin-dir ./claude-code-apple-dev-plugin
```

### With the Apple Dev Agent

The plugin includes an `apple-dev:dev` agent — an Opus-powered Apple platform expert that proactively uses the plugin's skills:

```bash
# Use as session agent
claude --agent apple-dev:dev

# Or with local plugin during development
claude --plugin-dir ./claude-code-apple-dev-plugin --agent apple-dev:dev
```

## What's Included

### Plugin Structure

```
.claude-plugin/plugin.json    # Plugin manifest
.lsp.json                     # Swift LSP (sourcekit-lsp) configuration
agents/dev.md                 # Apple platform developer agent
skills/                       # 166 skills (flat directory structure)
```

### Skills by Category

| Category                                        | Count | Purpose                                                                  |
| ----------------------------------------------- | ----- | ------------------------------------------------------------------------ |
| **Generators** (`generators-*`)                 | 62    | Production-ready Swift code for common features                          |
| **Product** (`product-*`)                       | 13    | Idea discovery to App Store workflow                                     |
| **SwiftUI** (`swiftui-*`)                       | 11    | Layout, environment, focus, toolbars, Transferable, Charts 3D            |
| **iOS** (`ios-*`)                               | 9     | Architecture, navigation, iPad, migration, accessibility, UI review      |
| **Swift** (`swift-*`)                           | 8     | Codable/JSON/TOON, concurrency, macros, testing, memory, CLI, regex, SPM |
| **macOS** (`macos-*`)                           | 8     | Tahoe APIs, SwiftData architecture, AppKit-SwiftUI bridge                |
| **Testing** (`testing-*`)                       | 8     | TDD workflows, snapshot tests, test data factories                       |
| **App Store** (`app-store-*`)                   | 7     | ASO, descriptions, keywords, reviews, search ads, rejections             |
| **Growth** (`growth-*`)                         | 4     | Analytics, press/media, community, indie business                        |
| **Apple Intelligence** (`apple-intelligence-*`) | 3     | Foundation Models, Visual Intelligence, App Intents                      |
| **Design** (`design-*`)                         | 2     | Liquid Glass, animation patterns                                         |
| **Performance** (`performance-*`)               | 2     | Instruments profiling, SwiftUI debugging                                 |
| **SwiftData** (`swiftdata-*`)                   | 2     | Modeling patterns, class inheritance                                     |
| **visionOS** (`visionos-*`)                     | 2     | Spatial computing, widgets                                               |
| **Security** (`security-*`)                     | 1     | Privacy manifests                                                        |
| **Legal** (`legal-*`)                           | 1     | Privacy policies, terms of service, EULAs                                |
| **Standalone frameworks**                       | 23    | HealthKit, ARKit, CarPlay, Core ML, MapKit, MusicKit, and more           |

**Total: 166 skills**

## Quick Start

**No idea yet?** Say: _"I don't know what to build"_

**New app?** Say: _"I have an idea for a macOS app that does X. Should I build it?"_

**Existing app?** Say: _"Review my code"_ or _"Add [feature]"_

See **[docs/usage.md](docs/usage.md)** for a complete guide.

## Generator Skills

Generate production-ready Swift code that adapts to your project:

| Generator                 | What It Creates                                    |
| ------------------------- | -------------------------------------------------- |
| `networking-layer`        | Async/await API client with Codable                |
| `auth-flow`               | Sign in with Apple + biometrics                    |
| `paywall-generator`       | StoreKit 2 subscriptions                           |
| `widget-generator`        | WidgetKit widgets with templates                   |
| `live-activity-generator` | ActivityKit Live Activities + Dynamic Island       |
| `push-notifications`      | APNs setup                                         |
| `deep-linking`            | URL schemes, universal links                       |
| `cloudkit-sync`           | CKSyncEngine CloudKit sync                         |
| `background-processing`   | BGTaskScheduler, downloads, silent push            |
| `app-extensions`          | Share, Action, Keyboard, Safari extensions         |
| `app-clip`                | App Clip target with invocation handling           |
| `test-generator`          | Unit/UI tests (Swift Testing + XCTest)             |
| `accessibility-generator` | VoiceOver, Dynamic Type                            |
| `logging-setup`           | Apple Logger infrastructure                        |
| `analytics-setup`         | Protocol-based analytics (TelemetryDeck, Firebase) |
| `persistence-setup`       | SwiftData + optional iCloud                        |
| `settings-kit`            | Complete preferences UI                            |
| `onboarding-generator`    | Multi-step welcome flow                            |
| `feature-flags`           | Local/remote feature flags                         |
| `ci-cd-setup`             | GitHub Actions / Xcode Cloud                       |
| `localization-setup`      | String catalogs, i18n                              |
| `error-monitoring`        | Crash reporting (Sentry/Crashlytics)               |
| `consent-flow`            | GDPR/CCPA consent with ATT                         |
| `data-export`             | JSON/CSV/PDF export, GDPR portability              |
| `offline-queue`           | Offline operation queue with retry                 |
| `screenshot-automation`   | Automated App Store screenshots                    |
| ... and 36 more           | See `skills/generators-*/`                         |

## Standalone Framework Skills

| Skill                          | Framework                                  |
| ------------------------------ | ------------------------------------------ |
| `healthkit`                    | HealthKit workouts, vitals, health records |
| `arkit`                        | ARKit scene understanding, anchors         |
| `core-ml`                      | Vision, NaturalLanguage, model integration |
| `mapkit-geotoolbox`            | MapKit, GeoToolbox, place descriptors      |
| `musickit`                     | MusicKit playback, catalog, library        |
| `carplay`                      | CarPlay templates, navigation              |
| `avfoundation`                 | Audio/video capture, playback              |
| `gamekit`                      | Game Center, leaderboards, achievements    |
| `eventkit`                     | Calendar, reminders                        |
| `photokit`                     | Photos framework, asset management         |
| `passkit`                      | Wallet passes, Apple Pay                   |
| `weatherkit`                   | WeatherKit forecasts, conditions           |
| `multipeer-connectivity`       | Peer-to-peer networking                    |
| `coretext`                     | CoreText layout, fonts                     |
| `textkit`                      | TextKit 2 text layout                      |
| `foundation-attributed-string` | AttributedString API                       |
| `watchos`                      | Watch apps, complications, health/fitness  |
| `tvos`                         | tvOS apps, focus engine                    |

## Features

### Swift LSP Integration

The plugin configures `sourcekit-lsp` automatically, giving Claude access to Swift type information, diagnostics, and go-to-definition in your projects.

### Apple Dev Agent

The `apple-dev:dev` agent establishes modern Apple development defaults:

- SwiftUI-first, Swift 6+ with strict concurrency
- MVVM with `@Observable`, `NavigationStack`, protocol-based DI
- SwiftData for persistence, Swift Testing for tests
- Proactively uses plugin skills for framework-specific patterns

## Documentation

| Doc                                | Description                         |
| ---------------------------------- | ----------------------------------- |
| [docs/usage.md](docs/usage.md)     | How to use for new vs existing apps |
| [docs/roadmap.md](docs/roadmap.md) | Skills roadmap and status           |
| [CONTRIBUTING.md](CONTRIBUTING.md) | How to contribute                   |

## Contributing

Contributions welcome! See [CONTRIBUTING.md](CONTRIBUTING.md).

## Disclaimer

Skills in this repository were generated with the assistance of [Claude Code](https://claude.ai/code). Content may contain inaccuracies — contributions and corrections are welcome.

## License

PolyForm Noncommercial License 1.0.0 — see [LICENSE](LICENSE).
