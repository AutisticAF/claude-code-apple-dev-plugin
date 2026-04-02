# Quick Start Guide

Get up and running with the Apple Dev plugin for Claude Code in under a minute.

## Prerequisites

- Claude Code installed
- An iOS, macOS, or Swift project

## Installation

### From the Marketplace

Browse the [Claude Code Marketplace](https://github.com/AutisticAF/autisticaf-claude-code-marketplace) and install:

```bash
claude plugin install github:AutisticAF/claude-code-apple-dev-plugin
```

Skills become available as `/apple:skill-name` across all your projects.

### Local Development

```bash
git clone https://github.com/AutisticAF/claude-code-apple-dev-plugin.git
claude --plugin-dir ./claude-code-apple-dev-plugin
```

### With the Apple Dev Agent

The plugin includes a specialized agent that proactively uses skills based on your context:

```bash
claude --agent apple-dev
```

Or during local development:

```bash
claude --plugin-dir ./claude-code-apple-dev-plugin --agent apple-dev
```

## First Use

### Just Describe What You Want

Skills activate automatically — no manual invocation needed:

| Say this... | Skill that activates |
|-------------|---------------------|
| "Review my code for best practices" | `ios-coding-best-practices` or `macos-coding-best-practices` |
| "Review my UI for HIG" | `ios-ui-review` or `macos-ui-review-tahoe` |
| "Add logging to my app" | `generators-logging-setup` |
| "Add subscription paywall" | `generators-paywall-generator` |
| "TDD this new feature" | `testing-tdd-feature` |
| "Fix this bug and add a test" | `testing-tdd-bug-fix` |
| "I need to refactor this safely" | `testing-tdd-refactor-guard` |
| "Should I monetize my app?" | `monetization` |
| "Review for release" | `release-review` |
| "I have an app idea..." | `product-agent` |

## Common Workflows

### New App

```
You: "I have an idea for a macOS app that does X"
→ product-agent validates the idea
→ product-market-research sizes the market
→ product-prd-generator creates the PRD
→ generators add features as you build
→ release-review audits before shipping
```

### Existing App — Add Features

```
You: "Add CloudKit sync to my app"
→ generators-cloudkit-sync generates the code
→ testing-tdd-feature helps you test it
```

### Existing App — Safe Refactoring

```
You: "I need to refactor my data layer"
→ testing-tdd-refactor-guard checks test coverage
→ testing-characterization-test-generator captures current behavior
→ You refactor with confidence
```

### Bug Fix

```
You: "Users report X is broken"
→ testing-tdd-bug-fix writes failing test first
→ Fix + verify + never regress
```

## Tips for Best Results

### Be Specific

```
Good: "Review ExpenseViewModel.swift for best practices"
Vague: "Check my code"
```

### Provide Context

```
Good: "Add a subscription paywall with monthly and yearly tiers"
Vague: "Add payments"
```

## Skill Categories (166 skills across 17 categories + standalone)

| Category | Count | Purpose |
|----------|-------|---------|
| `generators-*` | 63 | Production-ready code for common features |
| `product-*` | 13 | Idea to App Store workflow, beta testing, localization |
| `swiftui-*` | 11 | Layout, environment, focus, observable, presentations, transferable, AlarmKit, WebKit, text editing, toolbars, Charts 3D |
| `ios-*` | 9 | Code review, UI review, planning, architecture, accessibility, iPad, migration, navigation |
| `macos-*` | 8 | macOS development patterns, AppKit-SwiftUI bridge, Tahoe APIs |
| `swift-*` | 8 | Concurrency, Codable, macros, memory, testing, Package Manager, regex, CLI apps |
| `testing-*` | 8 | TDD workflows, test infrastructure, snapshots |
| `app-store-*` | 7 | ASO, descriptions, keywords, search ads, rejections, marketing |
| `growth-*` | 4 | Analytics, press/media, community, indie business |
| `apple-intelligence-*` | 3 | Foundation Models, Visual Intelligence, App Intents |
| `design-*` | 2 | Liquid Glass, animations |
| `performance-*` | 2 | Instruments, SwiftUI debugging |
| `security-*` | 2 | Keychain, biometrics, privacy manifests |
| `swiftdata-*` | 2 | Patterns, class inheritance |
| `visionos-*` | 2 | Spatial computing, widgets |
| `legal-*` | 1 | Privacy policies |
| + 21 standalone | 21 | Core ML, ARKit, AVFoundation, CarPlay, GameKit, HealthKit, MapKit, Monetization, MusicKit, PassKit, WeatherKit, etc. |

## Troubleshooting

### Skill Not Activating

1. Verify the plugin is installed: `claude plugin list`
2. Use explicit trigger phrases from the skill's "When This Skill Activates" section
3. Try: "Use the [skill-name] skill to review this"

### Wrong Skill Activates

- Be more specific in your request
- Skills auto-select based on keyword matching in their descriptions

## Next Steps

- Browse `skills/` to see all available skills
- Read [usage.md](usage.md) for the complete usage guide
- Read [roadmap.md](roadmap.md) for skill coverage tracking
- See [CONTRIBUTING.md](../CONTRIBUTING.md) to contribute
