# Upgrade 7.2 → 8.0: Lessons Learned

## Detection Commands

### Sprockets detection
```bash
# Sprockets is no longer default in Rails 8.0
# Exclude propshaft references
grep -rn "sprockets|Sprockets" config/ Gemfile app/assets/
```

### importmap detection
```bash
# Rails 8.0 uses importmap by default
# Check layout files for old javascript_include_tag
grep -rn "javascript_include_tag" app/views/layouts/
```

### Deprecated environment config
```bash
# These settings were removed in Rails 8 environment configs
grep -rn "disallowed_deprecation\|assets\.quiet\|config\.assets\." config/environments/
```

### Method name conflict detection
```bash
# Rails 8 introduces built-in `authorize` method
# Check for custom `authorize` methods that will conflict
grep -rn "def authorize\b" app/controllers/
grep -rn "before_action :authorize\b" app/controllers/
```

### Webpack remnants detection
```bash
# Rails 8 uses Propshaft + importmap; remove all webpack config
ls bin/webpack config/webpack/ 2>/dev/null
grep -rn "webpacker\|webpack" Gemfile config/ bin/
```

### Sprockets manifest detection
```bash
# app/assets/config/manifest.js is Sprockets-specific and should be removed
# Propshaft does not use this file
ls app/assets/config/manifest.js 2>/dev/null
```

### Asset path hardcoding detection
```bash
# Propshaft requires using asset_path helper instead of hardcoded paths
# Hardcoded /assets/ paths will break with digest-stamped filenames
grep -rn "'/assets/" app/views/ app/components/
grep -rn '"/assets/' app/views/ app/components/
```

## Method Name Conflicts (Critical)

Rails 8 introduces built-in authentication helpers that can conflict with existing custom methods.

### `authorize` → `authorize_user`
If your app defines a custom `authorize` method in ApplicationController (common pattern for checking if a user is logged in), it **will conflict** with Rails 8's built-in authorization methods. Rename it:

```ruby
# Before (Rails 7.x) — WILL BREAK in Rails 8
class ApplicationController < ActionController::Base
  def authorize
    return if user_signed_in?
    redirect_to login_path, notice: "Please log in"
  end
end

# After (Rails 8) — rename to avoid conflict
class ApplicationController < ActionController::Base
  def authorize_user
    return if user_signed_in?
    redirect_to login_path, notice: "Please log in"
  end
end
```

Then update **all** `before_action :authorize` calls across every controller:
```bash
# Find all controllers that need updating
grep -rn "before_action :authorize\b" app/controllers/
# Common pattern: sed -i 's/before_action :authorize\b/before_action :authorize_user/g'
```

### `authorize_jwt` → `authenticate_request`
If using JWT authentication in API controllers, rename to follow Rails 8 conventions:
```ruby
# Before
before_action :authorize_jwt
# After
before_action :authenticate_request
```

### `has_secure_password` adoption
Rails 8 encourages using the built-in `has_secure_password`:
```ruby
# Before: Manual BCrypt in SessionsController
return nil unless BCrypt::Password.new(user.password_digest) == password

# After: Use Rails' built-in authenticate method
return nil unless user.authenticate(password)
```

## Asset Pipeline Migration (Sprockets/Webpack → Propshaft)

Rails 8.0 changes the default asset pipeline from Sprockets to Propshaft.

### Step 1: Remove old asset tooling
```bash
# Delete these files/directories
rm -rf bin/webpack config/webpack/
rm -f app/assets/config/manifest.js
```

### Step 2: Update Gemfile
```ruby
# Remove
gem "sprockets-rails"
gem "sass-rails"

# Add
gem "propshaft"
```

### Step 3: Convert SCSS to CSS
Propshaft does not compile SCSS. Convert `.scss` files to `.css`:
- Replace SCSS variables with CSS custom properties
- Replace SCSS nesting with flat selectors
- Replace SCSS imports with CSS `@import`

### Step 4: Fix hardcoded asset paths
Propshaft uses digest-stamped filenames. Replace hardcoded paths with helpers:
```erb
<!-- Before: Breaks with Propshaft -->
<img src="/assets/app/apple-touch-icon.png">

<!-- After: Works with Propshaft -->
<img src="<%= asset_path('app/apple-touch-icon.png') %>">
```

### Step 5: Update environment configs
```ruby
# Remove from all environment files:
config.assets.css_compressor = nil
config.assets.compile = false
config.assets.quiet = true

# Add to production.rb:
config.public_file_server.enabled = true
config.public_file_server.headers = { "cache-control" => "public, max-age=#{1.year.to_i}" }
```

## Job Migration

### GlobalID support in job arguments
Rails 8 jobs can accept ActiveRecord objects directly via GlobalID serialization:
```ruby
# Old pattern
def perform(product_id)
  product = Product.find(product_id)
end

# New pattern — accepts both for backwards compatibility
def perform(product)
  @product = product.is_a?(Product) ? product : Product.find(product)
end
```

### Defensive nil checks
After queue migration, jobs may be enqueued with unexpected arguments. Add nil guards:
```ruby
def perform(record_id)
  return if record_id.nil?
  # ...
end
```

### Queue name audit
Review queue names used across all jobs — ensure they match what's configured in your queue adapter. Mismatched queue names mean jobs silently sit unprocessed.

## Production Config Changes

### config/environments/production.rb simplification
Rails 8 significantly simplifies production config. Key changes:

```ruby
# REMOVED — delete these lines:
config.public_file_server.enabled = false  # Now true by default
config.assets.css_compressor = nil
config.assets.compile = false
config.active_support.disallowed_deprecation = :raise
config.active_support.disallowed_deprecation_warnings = []

# CHANGED — update these:
# Old verbose logger setup:
config.logger = ActiveSupport::Logger.new(STDOUT)
  .tap { |logger| logger.formatter = ::Logger::Formatter.new }
  .then { |logger| ActiveSupport::TaggedLogging.new(logger) }
# New simplified logger:
config.log_tags = [ :request_id ]
config.logger = ActiveSupport::TaggedLogging.logger(STDOUT)

# NEW — add these:
config.silence_healthcheck_path = "/up"
config.active_support.report_deprecations = false
config.public_file_server.headers = { "cache-control" => "public, max-age=#{1.year.to_i}" }
```

### config/puma.rb overhaul
Rails 8 completely rewrites the default Puma config:

```ruby
# Old pattern (Rails 7.x) — remove:
workers 2
threads 1, 6
app_dir = File.expand_path("../..", __FILE__)
shared_dir = "#{app_dir}/shared"
rails_env = ENV['RAILS_ENV'] || "production"
environment rails_env
pidfile "#{shared_dir}/pids/puma.pid"
state_path "#{shared_dir}/pids/puma.state"
on_worker_boot do
  ActiveRecord::Base.connection.disconnect! rescue ActiveRecord::ConnectionNotEstablished
  ActiveRecord::Base.establish_connection(...)
end

# New pattern (Rails 8) — replace with:
threads_count = ENV.fetch("RAILS_MAX_THREADS", 3)
threads threads_count, threads_count
port ENV.fetch("PORT", 3000)
plugin :tmp_restart
pidfile ENV["PIDFILE"] if ENV["PIDFILE"]
```

**Key differences:**
- Default threads reduced from 5 to 3
- No more manual `on_worker_boot` database reconnection (Rails handles this)
- No more hardcoded shared directory paths
- Workers configured via `WEB_CONCURRENCY` env var instead of hardcoded

### config/environments/development.rb
```ruby
# REMOVED:
config.assets.quiet = true
```

## API Session/Cookie Handling

Rails 8 apps should explicitly skip sessions and cookies for API endpoints:

```ruby
# app/controllers/api/v3/api_controller.rb
class Api::V3::ApiController < ActionController::Base
  skip_before_action :track_ahoy_visit
  skip_before_action :set_csrf_cookie_header, raise: false
  before_action :skip_session

  private

  def skip_session
    request.session_options[:skip] = true
  end
end
```

This prevents unnecessary session/cookie overhead for stateless API requests and avoids conflicts with JWT-based authentication.

## Framework Defaults Rollout

Rails 8 generates `config/initializers/new_framework_defaults_8_0.rb` with all new defaults commented out. This is the safe upgrade path.

### Recommended rollout strategy
1. Start with all defaults commented out (compatibility mode)
2. Uncomment one setting at a time in `new_framework_defaults_8_0.rb`
3. Run the full test suite after each change
4. Once all settings are enabled and tests pass, delete the file and set `config.load_defaults 8.0` in `application.rb`

### Key defaults to evaluate carefully
- **`to_time_preserves_timezone = :zone`** — Changes timezone handling when converting to Time objects. Test any code that relies on timezone behavior.
- **`strict_freshness = true`** — Changes HTTP caching freshness checks. Test conditional GET requests.
- **`Regexp.timeout = 1`** — Adds a 1-second timeout to all regexes. Prevents ReDoS but may break complex patterns. Test any heavy regex usage.

## New Rails 8 Features to Adopt

### Query log tags
Appends controller/action info to SQL queries in logs — invaluable for debugging:
```ruby
# config/environments/production.rb (and development.rb)
config.active_record.query_log_tags_enabled = true
config.log_tags = [:request_id]
```

### Production inspect safety
Prevents PII from leaking in production logs via `#inspect`:
```ruby
# config/environments/production.rb
config.active_record.attributes_for_inspect = [:id]
```

### Health check log silencing
Prevents `/up` health checks from flooding logs:
```ruby
# config/environments/production.rb
config.silence_healthcheck_path = "/up"
```

## Subtle Gotchas

- **Propshaft migration**: Ensure `app/assets/manifest.json` exists for PWA support
- **Bootsnap with Zeitwerk**: No special changes needed, should work out of the box
- **Puma thread default reduced**: Rails 8 default is 3 threads (was 5). If relying on defaults, set `RAILS_MAX_THREADS` explicitly or adjust `config/puma.rb`
- **Puma config simplified**: `pidfile`, `worker_timeout`, and multi-worker config removed from default `puma.rb`. Review your config against the new Rails 8 template
- **Logger initialization**: The verbose `.tap { }.then { }` chain for tagged logging is replaced by `ActiveSupport::TaggedLogging.logger(STDOUT)`
- **`config.active_support.disallowed_deprecation` removed**: This setting and `disallowed_deprecation_warnings` are gone from environment configs. Remove them to avoid unknown config warnings
- **`config.assets.*` settings removed**: All asset pipeline config (`config.assets.quiet`, `config.assets.compile`, etc.) is gone. Remove from all environment files
- **Trusted proxy ranges**: If you customized `config.action_dispatch.trusted_proxies`, verify IP ranges are still correct — the defaults may have changed
- **`bin/setup` auto-starts server**: Rails 8 `bin/setup` calls `exec "bin/dev"` at the end. Use `bin/setup --skip-server` if you don't want this behavior
- **ActiveStorage service names**: If you rename storage services in `config/storage.yml`, update all references in every environment config file — a mismatch causes silent failures
- **Filter parameter additions**: Rails 8 adds `:email`, `:cvv`, `:cvc` to `config.filter_parameters`. Check that your filtered params list is complete
- **Email case sensitivity**: Rails 8's `has_secure_password` and authentication generators are case-sensitive by default. Always `.downcase` email on login and registration to prevent "user not found" bugs for users who type their email with different casing
- **`new_framework_defaults_7_0.rb` cleanup**: Once you've set `config.load_defaults 8.0`, delete the old `new_framework_defaults_7_0.rb` file — it's no longer needed and adds confusion
- **CSP initializer**: `config/initializers/content_security_policy.rb` format changes in Rails 8.0/8.1 — regenerate from the Rails template if you have a custom CSP
- **`public_file_server.enabled`**: Changed from `false` to `true` in production. If you relied on nginx for static file serving, review your reverse proxy config
