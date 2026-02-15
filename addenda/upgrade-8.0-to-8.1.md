# Upgrade 8.0 → 8.1: Lessons Learned

## Detection Commands

### load_defaults version detection
```bash
# Check current load_defaults — should be bumped to 8.1
grep -rn "load_defaults" config/application.rb
```

### STDOUT vs $stdout detection
```bash
# Rails 8.1 changes STDOUT to $stdout in logger config
grep -rn "STDOUT" config/environments/
```

### Cache store syntax detection
```bash
# Rails 8.1 uses shorthand syntax for cache_store config
# But custom config attributes may NOT work with shorthand
grep -rn "cache_store.*=" config/environments/
```

## config.load_defaults Jump

### Jumping from 7.1 to 8.1
If your app was still on `config.load_defaults 7.1` (common if you skipped incremental default upgrades), jumping directly to 8.1 is possible but requires care:

```ruby
# config/application.rb
# Before
config.load_defaults 7.1

# After
config.load_defaults 8.1
```

When making this jump:
- Delete `config/initializers/new_framework_defaults_7_0.rb` (117 lines of old defaults you no longer need)
- The `to_time_preserves_timezone = :zone` setting that you may have added manually is now the default — remove it from `config/application.rb`
- Review all settings from 7.1 → 7.2 → 8.0 → 8.1 defaults in the [Rails upgrade guide](https://guides.rubyonrails.org/upgrading_ruby_on_rails.html)

## Logger Changes

### STDOUT → $stdout
Rails 8.1 uses `$stdout` (global variable) instead of `STDOUT` (constant):

```ruby
# Before (8.0)
config.logger = ActiveSupport::TaggedLogging.logger(STDOUT)

# After (8.1)
config.logger = ActiveSupport::TaggedLogging.logger($stdout)
```

This is a subtle change. `$stdout` is reassignable (useful for test capture), while `STDOUT` is a constant.

## Cache Store Syntax (Gotcha)

### Shorthand works for config.cache_store
```ruby
# Rails 8.1 shorthand — works fine
config.cache_store = :redis_cache_store, { redis: -> { MyApp.redis }, namespace: "cache" }
```

### Shorthand does NOT work for custom config attributes
If you have custom cache store config attributes (e.g., for rate limiting), the shorthand syntax **silently fails**:

```ruby
# BROKEN — custom attribute with shorthand tuple
config.rate_limit_cache_store = :redis_cache_store, { redis: -> { MyApp.redis }, namespace: "rate_limits" }

# WORKS — must use explicit instantiation for custom attributes
config.rate_limit_cache_store = ActiveSupport::Cache::RedisCacheStore.new(
  redis: -> { MyApp.redis },
  namespace: "rate_limits"
)
```

The shorthand `:symbol, {options}` syntax is only handled by Rails internally for `config.cache_store`. Any custom config attributes that store cache instances must use `ActiveSupport::Cache::RedisCacheStore.new(...)` directly.

## Puma Configuration Changes

### WEB_CONCURRENCY=auto
Rails 8.1 adds support for `WEB_CONCURRENCY=auto` which automatically starts one worker per available processor:

```ruby
# config/puma.rb comment updated:
# You can set it to `auto` to automatically start a worker
# for each available processor.
```

## Development Config Simplification

Rails 8.1 further simplifies `config/environments/development.rb`:

### Changes
```ruby
# REMOVED — assets.debug (no longer needed with Propshaft)
config.assets.debug = true

# REMOVED — extensive trusted_proxies config (simplified)
config.action_dispatch.trusted_proxies = [
  IPAddr.new("172.64.66.1"),
  IPAddr.new("172.16.0.0/12"),
  # ...
]
# Replaced with:
config.action_dispatch.trusted_proxies = %w[127.0.0.1 ::1].map { |proxy| IPAddr.new(proxy) }

# CHANGED — ActionCable disable_request_forgery_protection now commented out by default
# config.action_cable.disable_request_forgery_protection = true

# NEW
config.action_dispatch.verbose_redirect_logs = true
config.assets.quiet = true  # Re-added (was removed in 8.0, back in 8.1)
```

## Content Security Policy

Rails 8.1 adds a new CSP option in the initializer:

```ruby
# config/initializers/content_security_policy.rb
# NEW — auto-nonce for script/style tags
# config.content_security_policy_nonce_auto = true
```

This automatically adds `nonce` attributes to `javascript_tag`, `javascript_include_tag`, and `stylesheet_link_tag` if the corresponding directives are listed in `content_security_policy_nonce_directives`.

## Error Pages Updated

Rails 8.1 completely rewrites `public/422.html` and updates `public/406-unsupported-browser.html`:
- Modern responsive design with CSS grid
- Dark mode support via `prefers-color-scheme: dark`
- Uses `100dvh` instead of `100vh` (dynamic viewport height)
- SVG error codes use named IDs (`#error-id`, `#error-description`) for dark mode color switching
- Consider regenerating from the Rails template: `bin/rails app:update`

## Subtle Gotchas

- **pool renamed to max_connections**: Database configuration syntax changed
  - Old: `pool: <%= ENV.fetch("RAILS_MAX_THREADS") { 5 } %>`
  - New: `max_connections: <%= ENV.fetch("RAILS_MAX_THREADS") { 5 } %>`
  - New options also available:
    - `min_connections: 0`      (NEW)
    - `keepalive: 60`          (NEW, seconds)
    - `max_age: 3600`          (NEW, seconds)

- **Semicolon query string separator removed**:
  - Old: `"foo=bar;baz=qux"` parsed as two params
  - New: `"foo=bar;baz=qux"` parsed as ONE param with bracket stripping
  - Detection: `grep -rn ";" app/ | grep "params\|query"` to find usage
  - Impact: Any API receiving semicolon-separated strings from external clients

- **Missing controllers now return 500**: Not a breaking change, but behavior change
  - Old: Routes pointing to non-existent controllers returned 404
  - New: Returns 500 Internal Server Error
  - Detection: `bin/rails routes | grep "#"` - audit that all referenced controllers exist
  - Impact: Error monitoring now catches 404s for missing controllers

- **MySQL unsigned_float and unsigned_decimal removed**:
  - Old: `add_column :products, :price, :unsigned_decimal`
  - New: `add_column :products, :price, :decimal + add_check_constraint :products, "price >= 0"`
  - Detection: `grep -rn "unsigned_float\|unsigned_decimal" db/migrate/`

- **`require File.expand_path("../boot", __FILE__)` removed**: Rails 8.1 only uses `require_relative "boot"` in `config/application.rb`. If you had both (common after previous upgrades), remove the `File.expand_path` version

- **`config.assets.quiet` returned**: This setting was removed in Rails 8.0 but is back in 8.1 for development. If you removed it during the 8.0 upgrade, add it back
