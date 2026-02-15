---
name: rails-upgrade-assistant
description: >
  Assists with upgrading Ruby on Rails applications. Analyzes codebase
  for breaking changes, generates detection scripts, and provides migration
  guidance. Covers Rails 3.0 through 8.2.
  Use when upgrading Rails, planning upgrades, or investigating version-specific issues.
allowed-tools: Bash, Read, Write, Edit, Glob, Grep
argument-hint: [target-version]
---

# Rails Upgrade Assistant

## What This Skill Does

When you ask about upgrading Rails, this skill:

1. **Detects** your current Rails version (via `bundle exec rails --version` or Gemfile.lock)
2. **Analyzes** your codebase for breaking changes using grep patterns
3. **Generates** reports based on your actual code (not generic examples)
4. **Provides** step-by-step migration guidance
5. **Offers** to apply fixes directly using Edit tool

## Core Workflow

```
User: "Upgrade my Rails app to 8.1"
    ↓
Claude: Detects version (e.g., 7.0.8)
Claude: Reads version guide (7.0→7.1 section from upgrading_ruby_on_rails.md)
Claude: Reads addendum for edge cases (addenda/upgrade-7.0-to-7.1.md)
Claude: Scans codebase using detection commands
Claude: Parses findings, generates report
Claude: Offers to apply fixes via Edit tool
```

## Rules

1. **Sequential upgrades only** - Never skip Rails versions. Upgrade 7.0 → 7.1 → 7.2 → 8.0 → 8.1
2. **Use actual code** - Reports must include your real code with file:line references
3. **Flag custom code** - Mark user-specific customizations with ⚠️ warnings
4. **Follow official guide** - upgrading_ruby_on_rails.md is the canonical source

## Quick Reference

```bash
# Detect current version
bundle exec rails --version

# Alternative: Read Gemfile.lock
grep "rails (" Gemfile.lock
```

## Usage

### Auto-activation
Claude detects when you ask about Rails upgrades, planning upgrades, or version-specific issues.

### Manual invocation
```
/rails-upgrade-assistant 8.1          # Specify target version
/rails-upgrade-assistant upgrade                # Auto-detect and prompt for target
```

## Supporting Files

- **upgrading_ruby_on_rails.md** - Official Rails upgrade guide (canonical source)
- **addenda/**** - Community-learned edge cases, detection patterns, gotchas
  - upgrade-4.1-to-4.2.md
  - upgrade-4.2-to-5.0.md
  - upgrade-5.0-to-5.1.md
  - upgrade-5.1-to-5.2.md
  - upgrade-5.2-to-6.0.md
  - upgrade-6.0-to-6.1.md
  - upgrade-6.1-to-7.0.md
  - upgrade-7.0-to-7.1.md
  - upgrade-7.1-to-7.2.md
  - upgrade-7.2-to-8.0.md
  - upgrade-8.0-to-8.1.md
  - general-upgrade-tips.md
