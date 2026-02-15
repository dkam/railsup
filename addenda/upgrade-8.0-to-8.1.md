# Upgrade 8.0 → 8.1: Lessons Learned

## Detection Commands

### SSL configuration detection
```bash
# Rails 8.1 comments out force_ssl and assume_ssl by default for Kamal compatibility
grep -rn 'force_ssl\s*=\s*true' config/environments/production.rb | grep -v "assume_ssl"
# If using Kamal: expect commented out settings
# If NOT using Kamal: expect both present and true
```

### bundler-audit integration
```bash
# Rails 8.1 requires bundler-audit gem
grep -rn "bundle-audit|bundler-audit" Gemfile .github/
```

### load_defaults version detection
```bash
# Check current load_defaults — should be bumped to 8.1
grep -rn "load_defaults" config/application.rb
```

### benchmark gem detection
```bash
# Rails 8.1 removed benchmark from activesupport's dependencies
# If your app uses benchmark directly, you need to add it to Gemfile
grep -rn "require.*benchmark\|Benchmark\." app/ lib/ test/
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

## Gem Compatibility Notes

- **Sidekiq adapter removed**: Rails 8.1 removes built-in Sidekiq adapter from ActiveJob
  - Requires: Sidekiq gem >= 7.3.3
  - Detection: `grep -rn "queue_adapter.*:sidekiq|queue_adapter.*:sucker_punch" config/`
  - Update: Add `gem "sidekiq"` to Gemfile, require: false
  - Remove: Any direct `Sidekiq.new` or `Sidekiq.sucker_punch` references

- **Azure ActiveStorage service removed**: Azure integration completely removed
  - Detection: `grep -rn "service.*azure|azure_storage" config/storage.yml`
  - Action: Switch to S3, GCS, or disk storage

- **bundler-audit gem required**: Security scanning now integrated
  - Add to Gemfile: `gem "bundler-audit", require: false`
  - Create: `config/bundler-audit.yml`
  - Make executable: `chmod +x bin/bundler-audit`

- **benchmark gem now separate**: Rails 8.1 removed `benchmark` from ActiveSupport's dependencies
  - If your app uses `Benchmark.measure` or similar, add `gem "benchmark"` to Gemfile
  - This is easy to miss — the error only shows at runtime, not during `bundle install`

## File Existence Checks

### bin/bundler-audit
```bash
# Should exist after 8.1 upgrade
[ -e bin/bundler-audit ]
```

### config/bundler-audit.yml
```bash
# Should exist after 8.1 upgrade
[ -e config/bundler-audit.yml ]
```

### bin/ci (new in 8.1)
```bash
# New executable in 8.1
[ -e bin/ci ]
```

### config/ci.rb (new in 8.1)
```bash
# New config file in 8.1
[ -e config/ci.rb ]
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

### Solid Queue plugin re-enabled
The default `config/puma.rb` uncomments the Solid Queue plugin:

```ruby
# Before (8.0) — commented out
# plugin :solid_queue if ENV["SOLID_QUEUE_IN_PUMA"]

# After (8.1) — active
plugin :solid_queue if ENV["SOLID_QUEUE_IN_PUMA"]
```

If you removed Solid Queue, make sure this line stays commented out or removed.

## Development Config Simplification

Rails 8.1 further simplifies `config/environments/development.rb`:

### Changes
```ruby
# REMOVED — Bullet gem config (move to initializer if needed)
config.after_initialize do
  Bullet.enable = true
  # ...
end

# REMOVED — barcode_scanner_enabled (app-specific config)
config.barcode_scanner_enabled = true

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

# REMOVED — explicit hosts entries (use Rails default)
config.hosts << "api.myapp.localhost"
# ...

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

## Systemd Service Files

If using systemd with Puma, the `ExecReload` command for phased restarts may need updating:

```ini
# Before — may fail silently
ExecReload=/path/to/pumactl -S /path/to/pumactl.sock phased-restart

# After — use su for proper environment, or add -C config path
ExecReload=/bin/su - appuser -c '/path/to/pumactl -S /path/to/pumactl.sock phased-restart'
```

## Subtle Gotchas

- **Kamal SSL settings commented**: If deploying to Kamal, SSL defaults may be commented out
  - Action: Review `config/environments/production.rb` for `# config.force_ssl = true` and `# config.assume_ssl = true`
  - Action: Uncomment them if you want SSL enforcement

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

- **Docker/CI QEMU compatibility**: Bootsnap parallel compilation must be disabled for QEMU builds
  - Disable: `BOOTSNAP_PARALLEL_COMPILATION=false` in Dockerfile
  - New: Jemalloc loaded via `LD_PRELOAD` instead of entrypoint script
  - File ownership: `COPY --chown=rails:rails` instead of after (`--chown=rails:rails` flag on `COPY`)
  - Impact: Docker builds will fail without this setting

- **Valkey replaces Redis in CI**: GitHub Actions service image changed
  - Old: `redis` image
  - New: `valkey/valkey:8`
  - Update: `.github/workflows/ci.yml`

- **`require File.expand_path("../boot", __FILE__)` removed**: Rails 8.1 only uses `require_relative "boot"` in `config/application.rb`. If you had both (common after previous upgrades), remove the `File.expand_path` version

- **`config.assets.quiet` returned**: This setting was removed in Rails 8.0 but is back in 8.1 for development. If you removed it during the 8.0 upgrade, add it back

- **connection_pool pinning**: Rails 8.1 may require `connection_pool ~> 2.5`. Check your Gemfile.lock for compatibility if you have gems that depend on older versions

- **Nokogiri native gems**: Rails 8.1 ships with native platform-specific Nokogiri gems (e.g., `nokogiri-arm64-darwin`) instead of compiling from source. This removes the `mini_portile2` dependency and speeds up installs, but may affect Dockerfile build strategies that expect source compilation
