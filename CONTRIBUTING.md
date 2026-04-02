# Contributing to Claude Code Apple Dev Plugin

Thank you for considering contributing! These skills help the entire Apple development community build better apps with Claude Code.

## Table of Contents

- [Code of Conduct](#code-of-conduct)
- [How Can I Contribute?](#how-can-i-contribute)
- [Getting Started](#getting-started)
- [Skill Development Guidelines](#skill-development-guidelines)
- [Testing Your Changes](#testing-your-changes)
- [Submitting Changes](#submitting-changes)

## Code of Conduct

- Be respectful and inclusive
- Welcome newcomers and learners
- Focus on what is best for the community
- Show empathy towards others

## How Can I Contribute?

### Reporting Bugs

Before creating bug reports, check existing issues to avoid duplicates.

Include:

- **Clear title**: Describe the issue concisely
- **Description**: Detailed explanation of the problem
- **Steps to reproduce**: How to trigger the issue
- **Expected vs actual behavior**
- **Environment**: Claude Code version, OS, Xcode version

### Suggesting Enhancements

Enhancement suggestions are tracked as GitHub issues. Include:

- **Use case**: Why is this enhancement needed?
- **Proposed solution**: How should it work?
- **Examples**: Code examples if applicable

### Contributing New Skills

We welcome new skills! Before creating one:

1. **Check the [roadmap](docs/roadmap.md)**: See if it's already planned
2. **Open an issue**: Discuss the skill idea first
3. **Follow guidelines**: Use `skills/shared-skill-creator/SKILL.md` as a guide

### Improving Existing Skills

- Add missing patterns or anti-patterns
- Update for new Swift/iOS/macOS versions
- Fix incorrect code examples or recommendations
- Improve modularization of large skills

## Getting Started

### Prerequisites

- [Claude Code](https://claude.ai/code) installed
- A Swift/Xcode project to test against

### Setup

1. **Fork the repository** on GitHub
2. **Clone your fork**:
   ```bash
   git clone https://github.com/AutisticAF/claude-code-apple-dev-plugin.git
   cd claude-code-apple-dev-plugin
   ```
3. **Create a branch**:
   ```bash
   git checkout -b feature/your-feature-name
   ```

### Branch Naming

- `feature/skill-name` — New skills
- `fix/issue-description` — Bug fixes
- `docs/update-description` — Documentation
- `refactor/skill-name` — Refactoring

## Skill Development Guidelines

### Directory Structure

All skills use a **flat directory structure** — no nesting:

```
skills/
├── {skill-name}/
│   ├── SKILL.md           # Entry point with YAML frontmatter (required)
│   ├── templates.md       # Code templates (optional)
│   ├── references/        # Supporting reference docs (optional)
│   ├── templates/         # Template directories (optional)
│   └── examples/          # Example output (optional)
```

**Simple skills** (< 400 lines): Single `SKILL.md` file.
**Complex skills** (> 400 lines): `SKILL.md` + `references/` folder.

### Naming Conventions

Skills that belong to a logical group use a `{group}-{name}` prefix. Standalone framework skills have no prefix.

| Prefix | Purpose |
|--------|---------|
| `ios-` | iOS development patterns |
| `macos-` | macOS development patterns |
| `swift-` | Swift language features |
| `swiftui-` | SwiftUI-specific patterns |
| `swiftdata-` | SwiftData patterns |
| `generators-` | Code generators producing production-ready Swift |
| `testing-` | TDD workflows, test infrastructure |
| `product-` | Product development workflows |
| `app-store-` | App Store optimization and management |
| `growth-` | Analytics, press/media, community |
| `design-` | UI/UX design patterns |
| `performance-` | Profiling and debugging |
| `apple-intelligence-` | AI/ML integration |
| `visionos-` | visionOS spatial computing |
| `legal-` | Privacy policies, terms of service |
| `security-` | Security and privacy |
| (none) | Standalone framework skills (e.g., `healthkit`, `arkit`) |

### SKILL.md Frontmatter

Every `SKILL.md` must start with:

```yaml
---
name: skill-name
description: Brief description and when to use it. Should include activation keywords.
---
```

The `name` must match the directory name. The `description` is loaded into Claude's context at session start, so make it specific enough for Claude to know when to activate the skill.

### Required Sections

1. **H1 title** — Human-readable name
2. **"When This Skill Activates"** — Bulleted list of trigger conditions
3. **Core content** — Varies by skill type (see below)
4. **References** (if applicable) — Links to reference files and related skills

### Skill Types

- **Advisory skills** (`ios-*`, `macos-*`): Review code and provide recommendations
- **Generator skills** (`generators-*`): Produce production-ready Swift code with templates. Include "Pre-Generation Checks" and "Configuration Questions" sections.
- **Workflow skills** (`testing-*`, `monetization`): Guide multi-step processes

### Code Examples

All Swift code examples must:

- Be syntactically correct
- Show both ❌ bad and ✅ good patterns
- Use realistic scenarios

### Content Guidelines

**Do:**
- Be clear and concise
- Provide concrete examples
- Explain the "why," not just "what"
- Reference official Apple documentation
- Keep content current with latest OS versions

**Don't:**
- Use vague descriptions
- Include outdated practices
- Copy copyrighted material
- Add project-specific details

## Plugin Structure

This repo is a Claude Code plugin. The structure is:

```
.claude-plugin/plugin.json    # Plugin manifest (name, version, author)
.lsp.json                     # Swift LSP configuration (sourcekit-lsp)
agents/apple-dev.md           # Apple platform developer agent
skills/                       # All 166 skills
docs/                         # Usage guide, roadmap
```

When modifying non-skill files:

- **plugin.json**: Update `version` for releases
- **.lsp.json**: Only modify if LSP configuration needs change
- **agents/apple-dev.md**: Update if new skill categories are added or conventions change

## Testing Your Changes

### 1. Validate Structure

```bash
# Check frontmatter
head -5 skills/your-skill/SKILL.md

# Verify name matches directory
grep "^name:" skills/your-skill/SKILL.md
```

### 2. Test as a Plugin

```bash
# Load the plugin locally in a Swift project
cd /path/to/your-swift-project
claude --plugin-dir /path/to/claude-code-apple-dev-plugin

# Verify your skill appears
# Try invoking it: /apple-dev:your-skill-name
```

### 3. Review Checklist

- [ ] YAML frontmatter is valid (`name` and `description` present)
- [ ] `name` matches the directory name
- [ ] All referenced files exist
- [ ] Code examples are syntactically correct Swift
- [ ] Markdown renders properly
- [ ] Follows naming conventions
- [ ] Tested via plugin in a real project

## Submitting Changes

### Commit Messages

```
type: Brief description

- Detail 1
- Detail 2
```

Types: `feat`, `fix`, `docs`, `refactor`, `chore`

Examples:
```
feat: Add swift-codable skill for JSON and TOON serialization
```
```
fix: Update swiftui-environment for iOS 18 onChange API
```

### Pull Request Process

1. Push to your fork
2. Create a PR with:
   - **Clear title** (under 70 characters)
   - **What changed** and **why**
   - **How you tested it**
3. Respond to feedback

### Approval Criteria

- [ ] Follows skill structure and naming conventions
- [ ] Has valid frontmatter with descriptive `description`
- [ ] Includes clear activation triggers
- [ ] Code examples are correct
- [ ] Doesn't duplicate existing skills
- [ ] Tested in a real project via plugin

## Resources

- [shared-skill-creator](skills/shared-skill-creator/) — Guide for creating new skills
- [docs/roadmap.md](docs/roadmap.md) — Planned skills
- [Apple HIG](https://developer.apple.com/design/human-interface-guidelines/)
- [Claude Code Docs](https://code.claude.com/docs)

## Questions?

Open an issue or join the discussions tab.
