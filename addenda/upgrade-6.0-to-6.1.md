# Upgrade 6.0 → 6.1: Lessons Learned

## Detection Commands

### rails-controller-testing usage
```bash
# This gem is commonly removed during 6.1 upgrades
grep -rn "assigns\b\|assert_template" test/ spec/
grep -rn "rails-controller-testing" Gemfile
```

### Session ID API change (Rack::Session::SessionId)
```bash
# session.id changed from string to SessionId object
grep -rn "session_options\[:id\]\|request\.session_options" app/
# Also check for session.id usage without .public_id
grep -rn "session\.id[^.]" app/ | grep -v "session_id\|public_id"
```

### Cookie/session store type
```bash
# CacheStore → CookieStore has 4KB limit implications
grep -rn "session_store\|CacheStore\|cookie_store" config/initializers/
```

### annotate_template_file_names misconfiguration
```bash
# Common: called as method (no-op) instead of assigned
grep -rn "annotate_template_file_names[^=]" config/environments/
```

## The Upgrade Timeline (Real-World)

One production app took approximately **one month** of active work (Nov 10 - Dec 11, 2020), with preparation throughout the 6.0.x era.

### Patch Version Path
| Version | Notes |
|---------|-------|
| 6.0.0 → 6.0.1 | Minor bump, Gemfile only |
| 6.0.1 → 6.0.2.2 | Session ID API change required code fixes |
| 6.0.2.2 → 6.0.3.3 | Security patches (XSS in ActionView, CSRF, ReDoS) |
| 6.0.3.3 → 6.0.3.4 | Patch update |
| 6.0.3.4 → 6.1.0.rc1 | RC testing |
| 6.1.0.rc1 → 6.1.0.rc2 | RC2 testing |
| 6.1.0.rc2 → 6.1.0 | GA release |

**Branch strategy**: The upgrade used a long-lived `rails61` feature branch with regular merges from master. Master continued receiving feature work throughout.

## Session ID Class Change (CRITICAL)

Rails 6.0.2+ changed `session.id` from a raw String to a `Rack::Session::SessionId` object. Code that uses the session ID for database lookups breaks silently (returns nil instead of matching records).

```ruby
# Before (Rails 6.0.0)
List.find_by(session_id: request.session_options[:id])

# After (Rails 6.0.2+) — use .public_id to get the string value
List.find_by(session_id: request.session&.id&.public_id)
```

**This change affected 5+ locations** in one production app (every place that associated anonymous carts/lists with sessions).

**Detection:**
```bash
grep -rn "session_options\[:id\]" app/
grep -rn "request\.session\.id[^.]" app/ | grep -v "public_id"
```

**Reference**: https://github.com/rack/rack/issues/1432

## Cookie/Session Store Migration

### CacheStore → CookieStore Pitfalls
Switching from `ActionDispatch::Session::CacheStore` (memcache-backed) to Rails' default cookie store introduces a **4KB cookie size limit**. If your app stores complex objects in the session, this will silently truncate data or raise errors.

**Symptoms**: Session data gets truncated or raises errors when storing complex objects like lists or browsing history

**Fix pattern**: Store only IDs/keys in the session, move data to cache:
```ruby
# Before (stores full array in session — works with CacheStore, breaks with CookieStore)
session[:history] = @viewed_products

# After (stores only a cache key in session)
session[:history_key] ||= SecureRandom.hex(10)
Rails.cache.write(session[:history_key], @viewed_products)
```

Objects that commonly blow the 4KB limit:
- Shopping cart contents
- Browsing history arrays
- Complex filter/sort state
- Serialized model objects

## rails-controller-testing Gem Removal

This gem provides `assigns()` and `assert_template()` helpers extracted from Rails core in Rails 5. It's often removed during the 6.1 upgrade because:
1. These test patterns are deprecated in favor of integration test assertions
2. The gem may have version constraints that conflict

**Migration pattern:**
```ruby
# Before (controller test with assigns)
assert_equal expected_product, assigns(:product)
assert_template 'products/show'

# After (integration test style)
assert_response :success
assert_select 'h1', expected_product.title
```

**Tip**: Remove `assigns()` and `assert_template()` usage proactively before the upgrade, then remove the gem.

## Webpacker Introduction

Rails 6.0-6.1 was the era where Webpacker became the default JavaScript compiler. The recommended approach was running Sprockets and Webpacker in parallel:

```erb
<!-- Layout runs both pipelines -->
<%= stylesheet_link_tag "application", media: "all", 'data-turbolinks-track': 'reload' %>
<%= javascript_include_tag "application", 'data-turbolinks-track': 'reload' %>
<%= javascript_pack_tag 'application' %>
```

Sprockets handled existing CSS/JS, while Webpacker handled modern JavaScript modules.

## Rails 6.1 Configuration Changes

### config/environments/test.rb
```ruby
# New in 6.1
config.eager_load = ENV["CI"].present?  # Was: false
config.cache_store = :null_store        # Recommended for tests
config.active_support.disallowed_deprecation = :raise
config.active_support.disallowed_deprecation_warnings = []
```

### config/environments/development.rb
```ruby
# New: Enable template annotation
# config.action_view.annotate_rendered_view_with_filenames = true

# New: Server timing headers
config.server_timing = true

# Changed: Explicit hosts configuration recommended
config.hosts << "myapp.localhost"
```

### config/environments/production.rb
```ruby
# Deprecation handling changed
# Old:
config.active_support.deprecation = :notify
# New:
config.active_support.report_deprecations = false
```

## annotate_template_file_names Misconfiguration

A common misconfiguration where the setting is called as a method (no-op) instead of assigned:
```ruby
# Wrong (no-op — raises no error but does nothing)
config.action_view.annotate_template_file_names

# Correct
config.action_view.annotate_template_file_names = true
```

## Subtle Gotchas

- **Zeitwerk autoloader strictened**: Rails 6.1 requires `zeitwerk ~> 2.3` (was `~> 2.2`). File naming must strictly match class names. Check for any non-conventional naming that was tolerated before

- **tzinfo 2.0 timezone names**: `tzinfo` moved from 1.x to 2.0 in Rails 6.1. Some timezone identifiers may have changed. If you have hardcoded timezone strings, verify they still resolve correctly

- **`config.action_view.raise_on_missing_translations`** moved to `config.i18n.raise_on_missing_translations` — the old location still works but is deprecated
