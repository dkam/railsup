# Rails Upgrade Assistant

A Claude Code skill for upgrading Ruby on Rails applications.

## Acknowledgments

This skill is inspired by [rails-upgrade-skill](https://github.com/maquina-app/rails-upgrade-skill) by Mario Alberto Chávez Cárdenas. His original work has been incorporated into the addenda.

## Contributing

Add your learnings to `addenda/` files and submit a PR.

Use your coding agent to extract upgrade tips from your own Rails app's git history. When you've successfully upgraded your app, those hard-won lessons can help everyone. Your real-world migration experience—the edge cases, gotchas, and solutions you discovered—becomes collective knowledge that makes future upgrades smoother for the entire community.

## Install

```bash
cd ~/.claude/skills
git clone git@github.com:dkam/railsup.git
```

Claude Code will automatically find the skill.

## Update

```bash
cd ~/.claude/skills/railsup
git pull
```

## Usage

### Auto-activation

Claude activates when you ask about Rails upgrades, planning upgrades, or investigating version-specific issues.

### Manual invocation

```bash
# Specify target version
/railsup 8.1

# Auto-detect and prompt
/railsup upgrade
```

## What It Does

1. Analyzes your Rails application for breaking changes
2. Generates detection scripts to scan your codebase
3. Provides comprehensive upgrade reports with step-by-step migration guides
4. Flags custom code and potential issues
5. Offers to apply fixes directly

## Important: Check Your Application Defaults

**Your Rails gem version and application defaults may not match!**

A common scenario that causes unexpected issues:
- Your app runs Rails 7.1 (gem version)
- But `config/application.rb` still has `config.load_defaults 7.0`
- When upgrading to Rails 7.2, you review 7.1→7.2 changes
- Then you bump defaults from 7.0→7.1
- **Breaking changes from 7.0→7.1 defaults surprise you**

**Before upgrading:**
1. Check your current `config.load_defaults` version in `config/application.rb`
2. Review upgrade notes for **both**:
   - Your gem version upgrade (e.g., 7.1→7.2)
   - Your defaults upgrade (e.g., 7.0→7.1 if you've been postponing it)
3. Consider upgrading defaults separately from the gem version to isolate issues

The skill will help identify changes for both, but be aware that you may need to review multiple version transitions.

## Supported Versions

Rails 3.0 through 8.2
