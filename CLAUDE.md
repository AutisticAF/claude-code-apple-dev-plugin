# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository Purpose

This is a collection of Claude Code skills for Apple platform development (iOS, macOS, iPadOS). Skills are markdown files that provide domain knowledge and workflows to Claude Code instances.

**This is NOT a code project** - it contains only markdown documentation files organized as skills.

## Architecture

### Skill Structure

All skills use a flat directory structure — no nesting:

```
skills/
├── {skill-name}/
│   ├── SKILL.md           # Entry point with YAML frontmatter (required)
│   ├── templates.md       # Code templates (optional)
│   ├── references/        # Supporting reference docs (optional)
│   ├── templates/         # Template directories (optional)
│   └── examples/          # Example output (optional)
```

Skills that belong to a logical group use a `{group}-{name}` prefix (e.g., `generators-auth-flow`, `ios-navigation-patterns`). Standalone skills have no prefix (e.g., `healthkit`, `arkit`).

### YAML Frontmatter (Required)

Every SKILL.md must have:

```yaml
---
name: skill-name
description: Brief description and when to use it
allowed-tools: [Read, Write, Edit, Glob, Grep, Bash]
---
```

### Skill Prefixes

| Prefix | Purpose |
|--------|---------|
| `ios-` | iOS development patterns |
| `macos-` | macOS development patterns |
| `product-` | Product development workflow (idea to App Store), beta testing, localization |
| `generators-` | Code generators producing production-ready Swift |
| `growth-` | Analytics, press/media, community building, indie business |
| `legal-` | Privacy policies, terms of service, EULAs |
| `apple-intelligence-` | Foundation Models, Visual Intelligence |
| `design-` | Liquid Glass, modern design patterns |
| `app-store-` | ASO, descriptions, screenshots, reviews, search ads, rejections |
| `swift-` | Swift language features (concurrency, macros, testing, CLI) |
| `swiftui-` | SwiftUI-specific patterns (layout, focus, environment, etc.) |
| `swiftdata-` | SwiftData modeling, queries, patterns |
| `testing-` | TDD workflows, test infrastructure, snapshot tests |
| `performance-` | Profiling, SwiftUI debugging |
| `visionos-` | visionOS spatial computing, widgets |
| (none) | Standalone framework skills (healthkit, arkit, carplay, etc.) |

### Generator vs Advisory vs Workflow Skills

- **Advisory skills** (`ios-*`, `macos-*`, `release-review`): Review code and provide recommendations
- **Generator skills** (`generators-*`): Produce production-ready Swift code with templates
- **Workflow skills** (`testing-*`, `monetization`): Guide multi-step processes with methodology and code generation

## Creating New Skills

Use `skills/shared-skill-creator/SKILL.md` as the guide. Key requirements:

1. **Simple skills** (< 400 lines): Single SKILL.md file
2. **Complex skills** (> 400 lines): Modularize into SKILL.md + `references/` folder
3. **Naming**: Use `kebab-case` for skill names and files
4. **Activation triggers**: Clear "When This Skill Activates" section
5. **Examples**: Always show ✅ good and ❌ bad patterns

## Apple Docs Reference

Apple documentation files were used as source material when creating these skills. Skills contain the relevant API patterns inline.

See `docs/ROADMAP.md` for planned skills based on available Apple docs.

## Conventions

- Code examples must be syntactically correct Swift
- Use emoji sparingly (✅ ❌ for patterns, priority indicators for issues)
- Output formats should use: 🔴 Critical, 🟠 High, 🟡 Medium, 🟢 Low, ✅ Strengths
- Generator templates use protocol-based architecture for swappable implementations
