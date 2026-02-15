# Upgrade 6.0 → 6.1: Lessons Learned

## Detection Commands

### active_model_serializers version constraint
```bash
# AMS < 0.10.12 has upper-bound lock (< 6.1) — blocks the upgrade
grep -rn "active_model_serializers" Gemfile Gemfile.lock
bundle exec ruby -e "puts Gem.loaded_specs['active_model_serializers']&.version"
```

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

### inline_svg deprecation
```bash
# inline_svg() renamed to inline_svg_tag() in 1.7+
grep -rn "inline_svg(" app/views/ | grep -v "inline_svg_tag\|inline_svg_pack_tag"
```

### URI.open deprecation (Ruby 2.7+)
```bash
# Kernel#open with URLs is deprecated/dangerous
grep -rn "[^I]open(" app/ lib/ | grep -v "File\.open\|URI\.open\|IO\.open\|open_uri"
```

### thor version conflict
```bash
# Rails 6.1 requires thor ~> 1.0, many gems pin ~> 0.19
grep -rn "thor" Gemfile.lock | head -10
```

### thread_safe gem removal
```bash
# Rails 6.1 replaces thread_safe with concurrent-ruby via tzinfo 2.0
grep -rn "thread_safe" Gemfile Gemfile.lock
```

### force_ssl configuration
```bash
# May cause redirect loops behind reverse proxies
grep -rn "force_ssl" config/environments/
```

### annotate_template_file_names misconfiguration
```bash
# Common: called as method (no-op) instead of assigned
grep -rn "annotate_template_file_names[^=]" config/environments/
```

## The Upgrade Timeline (Real-World)

One production app took approximately **one month** of active work (Nov 10 - Dec 11, 2020), with preparation throughout the 6.0.x era. The main blocker was a third-party gem version constraint.

### Patch Version Path
| Version | Notes |
|---------|-------|
| 6.0.0 → 6.0.1 | Minor bump, Gemfile only |
| 6.0.1 → 6.0.2.2 | Session ID API change required code fixes |
| 6.0.2.2 → 6.0.3.3 | Security patches (XSS in ActionView, CSRF, ReDoS) |
| 6.0.3.3 → 6.0.3.4 | Combined with Ruby 2.7.2 upgrade |
| 6.0.3.4 → 6.1.0.rc1 | `neat` gem commented out (thor conflict) |
| 6.1.0.rc1 → 6.1.0.rc2 | **BROKEN** — AMS threading gem issue |
| 6.1.0.rc2 → 6.1.0 | AMS 0.10.12 released, unblocking the upgrade |

**Key strategy**: The Ruby 2.7 upgrade was done on 6.0.3.4 (one month before the 6.1 RC testing). This separated Ruby deprecation warnings from Rails version issues.

**Branch strategy**: The upgrade used a long-lived `rails61` feature branch with regular merges from master. Master continued receiving feature work throughout.

## ActiveModelSerializers Threading/Version Blocker

The most significant blocker during this upgrade. AMS 0.10.10 had strict version constraints:
```
actionpack (>= 4.1, < 6.1)
activemodel (>= 4.1, < 6.1)
```

This upper-bound lock prevented the gem from loading with Rails 6.1. The app was committed in a **non-functional state** on the upgrade branch for a week until AMS 0.10.12 was released with relaxed constraints (`< 6.2`).

**Detection:**
```bash
# Check for gems with upper-bound Rails version locks
bundle exec ruby -e "
  Bundler.load.specs.each do |spec|
    spec.dependencies.each do |dep|
      if dep.name =~ /^(action|active|rail)/ && dep.requirement.to_s =~ /<\s*[67]/
        puts \"#{spec.name} #{spec.version}: #{dep.name} #{dep.requirement}\"
      end
    end
  end
"
```

**The deeper issue**: Rails 6.1 replaced the `thread_safe` gem with `concurrent-ruby` (via tzinfo 2.0). AMS transitively depended on `thread_safe`, which was no longer pulled in.

**Lesson**: Before attempting a major Rails upgrade, run `bundle update rails` in a branch and check which gems have upper-bound version constraints that block the upgrade. Wait for gem maintainers to release compatible versions, or fork/patch if urgent.

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

## Gem Compatibility

### Bourbon Neat CSS Framework (thor conflict)
```ruby
# Rails 6.1 requires thor ~> 1.0
# Neat ~> 1.8 depended on thor ~> 0.19

# Fix: Remove version pin or upgrade to Neat 2+
# Old:
gem 'neat', '~> 1.8'
# New:
gem 'neat'  # Resolves to a compatible version
```

The `bourbon` gem also needed upgrading from 6.0.0 to 7.0.0 to resolve the thor conflict.

### rails-controller-testing Gem Removal
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

### Timecop Fork Issue
If using Timecop, check for a fork reference in your Gemfile. One production app needed a fork for a bug fix that hadn't been released. This is common with less actively maintained gems — check if official releases have caught up before upgrading.

## Credentials Migration (Pre-Upgrade Preparation)

Migrating secrets to Rails credentials should be done **before** the version upgrade, not during it:

```ruby
# Before (hardcoded in controller — security risk)
client_id = 'XXXXXX-XXXXXXXXXXXX.apps.googleusercontent.com'
client_secret = 'XXXXXXXXXXXXXXXXXXXX'

# After (Rails credentials)
client_id     = Rails.application.credentials.google_client_id
client_secret = Rails.application.credentials.google_client_secret
```

**Lesson**: Decouple security housekeeping from version upgrades. One production app did this migration months before the 6.1 upgrade, making the actual upgrade smaller and lower-risk.

## SSL Configuration

### force_ssl Toggle
If your SSL is terminated at a reverse proxy (Nginx/Apache/load balancer), `config.force_ssl = true` can cause redirect loops:

```ruby
# Caused redirect loops behind Passenger/Nginx SSL termination
config.force_ssl = true

# Fix: Let the reverse proxy handle SSL
config.force_ssl = false
# Or use assume_ssl if you need the X-Forwarded-Proto trust
```

One production app toggled this twice before settling on `false`.

## Webpacker & Stimulus Introduction

Rails 6.0-6.1 was the era where Webpacker and Stimulus were commonly adopted alongside the upgrade:

### Dual Pipeline Architecture
The recommended approach was running Sprockets and Webpacker in parallel:

```erb
<!-- Layout runs both pipelines -->
<%= stylesheet_link_tag "application", media: "all", 'data-turbolinks-track': 'reload' %>
<%= javascript_include_tag "application", 'data-turbolinks-track': 'reload' %>
<%= javascript_pack_tag 'application' %>
```

Sprockets handled existing CSS/JS (bourbon, neat, sass, coffee), while Webpacker handled new Stimulus controllers and modern JavaScript modules.

### Stimulus v2 Upgrade
If adopting Stimulus during this period, note that Stimulus v2 was released. Controllers added at v1 needed updating for v2's `@hotwired/stimulus` import path (this becomes critical during the later 7.0 upgrade).

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

## Deprecation Patterns

### inline_svg → inline_svg_tag
```erb
<!-- Before -->
<%= inline_svg('icon.svg') %>

<!-- After -->
<%= inline_svg_tag('icon.svg') %>
```

Seven instances in one production app's layout file.

### URI.open (Ruby 2.7 deprecation)
```ruby
# Before (deprecated in Ruby 2.7, dangerous in Ruby 3+)
Feedjira.parse(open('https://blog.example.com/feed/').read)

# After
Feedjira.parse(URI.open('https://blog.example.com/feed/').read)
```

### annotate_template_file_names
A common misconfiguration where the setting is called as a method (no-op) instead of assigned:
```ruby
# Wrong (no-op — raises no error but does nothing)
config.action_view.annotate_template_file_names

# Correct
config.action_view.annotate_template_file_names = true
```

## Multi-Site Authentication Gotchas

If your app serves multiple sites/domains from one codebase, the 6.1 upgrade may expose session isolation issues:

- **User lookup must include site**: `User.find_by(email: email, site: current_site)` instead of just email
- **Session must track site_id**: Store `session[:site_id]` not the full Site object (avoids serialization issues)
- **Password reset must be site-aware**: Reset links must route to the correct domain
- **Cache keys must include site**: Otherwise cached content from one site leaks to another

## Subtle Gotchas

- **Zeitwerk autoloader strictened**: Rails 6.1 requires `zeitwerk ~> 2.3` (was `~> 2.2`). File naming must strictly match class names. Check for any non-conventional naming that was tolerated before

- **tzinfo 2.0 timezone names**: `tzinfo` moved from 1.x to 2.0 in Rails 6.1. Some timezone identifiers may have changed. If you have hardcoded timezone strings, verify they still resolve correctly

- **mimemagic dependency added**: `activestorage` gained a dependency on `mimemagic (~> 0.3.2)`. If not using ActiveStorage, this adds a gem that requires a MIME database file. This later became a famous licensing controversy (mimemagic 0.3.x was relicensed)

- **Action Cable with Turbolinks navigation**: If using ActionCable with Turbolinks, subscriptions must be carefully managed on `turbolinks:load` to prevent double-subscriptions or orphaned connections across page navigations

- **Memcache OOM with WebSocket servers**: If running ActionCable on the same server as Memcache, the WebSocket connections' memory usage can OOM the Memcache process. Consider reducing Memcache size or moving it to a separate host

- **`config.action_view.raise_on_missing_translations`** moved to `config.i18n.raise_on_missing_translations` — the old location still works but is deprecated

- **Distributed locking**: If adopting background job processing (Beanstalkd, Sidekiq) during this period, consider adding distributed locking (e.g., `suo` gem with Redis) to prevent duplicate job execution

- **Private gem servers**: If migrating gems to a private gem server, use `require: false` for gems that are only needed in specific contexts (e.g., CLI scripts, background jobs) to avoid slowing down boot time
