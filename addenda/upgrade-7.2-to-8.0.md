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

### Solid gems configuration
```bash
# Rails 8.0 defaults to Solid Cache/Queue/Cable
# Check if already using Solid gems
grep -rn "cache_store.*solid|queue_adapter.*solid|action_cable.server.*solid" config/environments/
```

### Existing queue adapter detection
```bash
# Identify current queue adapter before migrating to Solid Queue
grep -rn "queue_adapter" config/
grep -rn "GoodJob\|Sidekiq\|good_job\|sidekiq" config/ Gemfile app/
```

### Job processing engine routes
```bash
# Check for mounted job engine dashboards that need removal
grep -rn "mount.*GoodJob\|mount.*Sidekiq\|mount.*Resque" config/routes.rb
```

### Puma configuration
```bash
# Check for old Puma config patterns that changed in Rails 8
grep -n "RAILS_MAX_THREADS\|worker_timeout\|PIDFILE" config/puma.rb
```

### Dotenv explicit loading
```bash
# Check for explicit Dotenv loading that may conflict with Rails 8
grep -rn "Dotenv::Rails" config/application.rb
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

## Gem Compatibility Notes

- **Propshaft required**: Rails 8.0 default asset pipeline
  - Add `gem "propshaft"` to Gemfile if removing Sprockets
  - Remove `gem "sprockets-rails"` and `gem "sass-rails"`
  - Remove `gem "autoprefixer-rails"` (Tailwind v4 handles vendor prefixing)
  - Delete `app/assets/config/manifest.js` (Sprockets-only file)
  - Delete `config/webpack/` directory and `bin/webpack` if present
  - SCSS files must be converted to plain CSS (Propshaft does not compile SCSS)
  - See: https://github.com/rails/propshaft

- **Solid Cache (optional but evaluate carefully)**:
  - Database-backed caching; good for simple setups
  - **WARNING**: Can exhaust database connection pools under high concurrency. One production app saw SolidCache exceptions from connection pool exhaustion, requiring pool increases from 5 to 100 (SQLite) and 20 to 50 (PostgreSQL)
  - If your app has high cache throughput, consider Redis/DragonflyDB/Valkey instead
  - See: https://github.com/rails/solid_cache

- **Solid Queue (optional but evaluate carefully)**:
  - Database-backed job queue; simpler ops than Sidekiq/GoodJob
  - **WARNING**: If using PgBouncer, Solid Queue needs a direct PostgreSQL connection (uses `LISTEN/NOTIFY`)
  - One production app adopted Solid Queue then migrated back to Sidekiq after encountering issues with concurrency limits and job throughput
  - For high-volume job processing, Sidekiq may remain the better choice
  - See: https://github.com/rails/solid_queue

- **Solid Cable (optional)**:
  - Consider using Solid Cable for WebSocket connections
  - See: https://github.com/rails/solid_cable

- **Thruster**: New in Rails 8, provides HTTP/2, asset compression, and X-Sendfile
  - Add `gem "thruster"` to Gemfile
  - Creates `bin/thrust` binstub
  - Replaces need for separate nginx/caddy for basic HTTP serving
  - See: https://github.com/basecamp/thruster

- **Tailwind CSS**: If upgrading alongside Rails 8, note Tailwind v4 ships with `tailwindcss-rails`
  - May need both `gem "tailwindcss-rails"` and `gem "tailwindcss-ruby"` for v4
  - See Tailwind 3→4 section below for breaking changes

## Edge Cases

- **Multi-database config required**: If using multi-database features with Solid gems, `config/database.yml` needs restructuring

- **Docker/CI**: Rails 8.0 introduces Thruster gem
  - Disable Bootsnap parallel compilation: `bundle exec bootsnap precompile -j 1 --gemfile` (note: `-j1` flag)
  - New Jemalloc options via ENV

- **New PWA routes**: Rails 8.0 adds `allow_browser` to ApplicationController
  - Check for existing custom middleware before adding
  - May block old browsers - review carefully

- **Action Cable API changes**: WebSocket adapter and logger adapter properties moved
  - Update: `ActionCable.WebSocket = MyWebSocket`
  - Update: `ActionCable.adapters.logger = myLogger`

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
gem "autoprefixer-rails"  # Not needed with Tailwind v4

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

## Tailwind CSS 3 → 4 Migration (if applicable)

If upgrading Tailwind alongside Rails 8, expect a **large diff** (one production app changed 127 files). Key changes:

### CSS entry point rewrite
```css
/* Before (Tailwind v3) */
@tailwind base;
@tailwind components;
@tailwind utilities;

/* After (Tailwind v4) */
@import 'tailwindcss';
@config "../../../config/tailwind.config.js";
```

### Border color default changed
Tailwind v4 changed the default border color from `gray-200` to `currentColor`. Add a compatibility layer:
```css
@layer base {
  *, ::after, ::before, ::backdrop, ::file-selector-button {
    border-color: var(--color-gray-200, currentColor);
  }
}
```

### Class renames (search & replace across all views)
| Tailwind v3 | Tailwind v4 |
|---|---|
| `shadow` | `shadow-sm` |
| `shadow-sm` | `shadow-xs` |
| `rounded` | `rounded-sm` |
| `focus:outline-none` | `focus:outline-hidden` |
| `flex-shrink-0` | `shrink-0` |
| `ring-opacity-5` | `ring-black/5` (use opacity modifier) |
| `max-h-1/2` | `max-h-[50vh]` (use arbitrary values) |

### Custom utilities syntax changed
```css
/* Before (Tailwind v3) — inside @layer utilities */
@layer utilities {
  .columns-2 { columns: 2 }
}

/* After (Tailwind v4) — use @utility directive */
@utility columns-2 {
  columns: 2;
}
```

### Detection commands for Tailwind v3 classes needing update
```bash
# Find shadow classes that need renaming
grep -rn 'shadow"' app/views/ app/components/ | grep -v 'shadow-sm\|shadow-lg\|shadow-xl\|shadow-md\|shadow-xs\|shadow-none'
# Find outline-none that needs updating
grep -rn 'outline-none' app/views/ app/components/
# Find flex-shrink-0 that needs updating
grep -rn 'flex-shrink' app/views/ app/components/
# Find ring-opacity that needs updating
grep -rn 'ring-opacity' app/views/ app/components/
```

## Solid Queue Migration

Migrating from GoodJob, Sidekiq, or Resque to Solid Queue is a common Rails 8 upgrade task. This is error-prone — take it step by step.

### Queue adapter swap
```ruby
# config/environments/production.rb
config.active_job.queue_adapter = :solid_queue  # was :good_job, :sidekiq, etc.
```

### Database setup for Solid gems
Each Solid gem needs its own database entry in `config/database.yml`:

```yaml
production:
  primary:
    <<: *default
    database: myapp_production

  queue:
    <<: *default
    database: solid_queue
    migrations_paths: db/queue_migrate

  cache:
    <<: *default
    database: solid_cache
    migrations_paths: db/cache_migrate

  cable:
    <<: *default
    database: solid_cable
    migrations_paths: db/cable_migrate
```

**Gotchas:**
- If using PgBouncer, Solid Queue may need a direct PostgreSQL connection (not through PgBouncer) since it uses `LISTEN/NOTIFY`
- Create the databases and roles before running migrations
- Each database needs its own `migrations_paths` — do not mix with primary migrations
- **Connection pool sizing is critical**: SolidCache with SQLite can exhaust pools under load. Set `pool: 100` or higher for cache databases in high-throughput apps

### Solid Queue configuration (`config/queue.yml`)
```yaml
default: &default
  dispatchers:
    - polling_interval: 1
      batch_size: 500
  workers:
    - queues: "*"
      threads: 3
      polling_interval: 1

development:
  <<: *default

production:
  <<: *default
  workers:
    - queues: "*"
      threads: <%= ENV.fetch("SOLID_QUEUE_THREADS", 4) %>
      processes: <%= ENV.fetch("JOB_CONCURRENCY", 2) %>
      polling_interval: 1
```

**Tuning tips:**
- `polling_interval: 0.1` hammers the database — start with `1` (seconds) and lower only if needed
- `batch_size` controls how many jobs are dispatched per poll — higher values use more memory but are more efficient
- In production, set explicit `processes` count rather than using `Etc.nprocessors` for predictable resource usage

### Puma Solid Queue plugin
For single-server deployments, run queue workers inside Puma:
```ruby
# config/puma.rb
plugin :solid_queue if ENV["SOLID_QUEUE_IN_PUMA"]
```

### Recurring jobs (`config/recurring.yml`)
Rails 8 has built-in recurring job support — replaces cron, whenever, sidekiq-cron:
```yaml
production:
  clear_finished_jobs:
    class: ClearFinishedJobsJob
    queue: default
    schedule: every 12 hours

  my_periodic_task:
    class: MyPeriodicJob
    queue: low
    schedule: every 1 hour
```

### Cleaning up finished jobs
Solid Queue keeps finished job records by default. Add a cleanup job:
```ruby
# app/jobs/clear_finished_jobs_job.rb
class ClearFinishedJobsJob < ApplicationJob
  def perform
    SolidQueue::Job.clear_finished_in_batches
  end
end
```

Control this with: `config.solid_queue.preserve_finished_jobs = true` in `application.rb`.

### Remove old queue engine routes
```ruby
# config/routes.rb — DELETE these:
mount GoodJob::Engine => "good_job"
mount Sidekiq::Web => "/sidekiq"
```

### Reverting from Solid Queue (escape hatch)
One production app adopted Solid Queue then migrated back to Sidekiq after 4+ months due to:
- Database connection pool exhaustion under high concurrency
- Concurrency limiting issues
- Job throughput bottlenecks

If you need to revert:
1. **Do not clear Solid Queue jobs from Sidekiq** — they use different serialization
2. Move jobs incrementally (a few queues at a time, not all at once)
3. Remove `solid_queue` and `mission_control-jobs` gems
4. Clean up Solid Queue database config from `database.yml`
5. Remove `plugin :solid_queue` from `config/puma.rb`

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

### Concurrency limits with Solid Queue
Prevent duplicate work on the same resource:
```ruby
class MyJob < ApplicationJob
  limits_concurrency(
    key: ->(record_id) { record_id },
    duration: 5.minutes
  )

  def perform(record_id)
    # Only one job per record_id runs at a time
  end
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
Review queue names used across all jobs — ensure they match what's configured in `config/queue.yml`. Mismatched queue names mean jobs silently sit unprocessed.

## Sidekiq Web UI Authentication (Rails 8 gotcha)

If keeping Sidekiq and need to authenticate its web UI, Rails 8 makes session-based auth in route constraints harder. Sessions are not reliably accessible in Rack middleware context.

### Failed approaches (documented to save you time)
1. **Rack middleware session access**: `request.session` returns empty hash
2. **ActionDispatch::Request in middleware**: Still doesn't load session data properly
3. **Route constraints with session check**: Sessions unreliable in constraint lambdas
4. **Cookie-based remember_token fallback**: Complex and fragile

### Working solution: Cache-based admin token
```ruby
# In your SessionsController (on login)
def set_admin_cache_cookie(user)
  return unless user&.is_editor
  admin_token = SecureRandom.hex(16)
  Rails.cache.write("admin_auth:#{admin_token}", true, expires_in: 1.hour)
  cookies[:admin_auth] = { value: admin_token, expires: 1.hour.from_now, httponly: true }
end

# In config/routes.rb
mount Sidekiq::Web, at: "/sidekiq", constraints: lambda { |request|
  admin_token = request.cookies["admin_auth"]
  admin_token.present? && Rails.cache.read("admin_auth:#{admin_token}")
}
```

**Tip**: Refresh the cache token on page loads (e.g., user menu) to prevent timeout during active sessions.

## Production Config Changes

### config/environments/production.rb simplification
Rails 8 significantly simplifies production config. Key changes:

```ruby
# REMOVED — delete these lines:
config.public_file_server.enabled = false  # Now true by default (Thruster serves files)
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
# Rails.configuration.recaptcha_enabled  # Use FeatureSwitch pattern instead
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
- **Dotenv explicit loading**: If `config/application.rb` has `Dotenv::Rails.overwrite = true` or `Dotenv::Rails.load`, remove it — Rails 8 handles env loading via the gem's Railtie
- **Logger initialization**: The verbose `.tap { }.then { }` chain for tagged logging is replaced by `ActiveSupport::TaggedLogging.logger(STDOUT)`
- **Cache store toggle removed**: Rails 8 dev config no longer uses the `tmp/caching-dev.txt` file toggle. Cache store is set directly in `config/environments/development.rb`
- **`config.active_support.disallowed_deprecation` removed**: This setting and `disallowed_deprecation_warnings` are gone from environment configs. Remove them to avoid unknown config warnings
- **`config.assets.*` settings removed**: All asset pipeline config (`config.assets.quiet`, `config.assets.compile`, etc.) is gone. Remove from all environment files
- **Trusted proxy ranges**: If you customized `config.action_dispatch.trusted_proxies`, verify IP ranges are still correct — the defaults may have changed
- **`bin/setup` auto-starts server**: Rails 8 `bin/setup` calls `exec "bin/dev"` at the end. Use `bin/setup --skip-server` if you don't want this behavior
- **ActiveStorage service names**: If you rename storage services in `config/storage.yml`, update all references in every environment config file — a mismatch causes silent failures
- **Filter parameter additions**: Rails 8 adds `:email`, `:cvv`, `:cvc` to `config.filter_parameters`. Check that your filtered params list is complete
- **Email case sensitivity**: Rails 8's `has_secure_password` and authentication generators are case-sensitive by default. Always `.downcase` email on login and registration to prevent "user not found" bugs for users who type their email with different casing
- **`new_framework_defaults_7_0.rb` cleanup**: Once you've set `config.load_defaults 8.0`, delete the old `new_framework_defaults_7_0.rb` file — it's no longer needed and adds confusion
- **CSP initializer**: `config/initializers/content_security_policy.rb` format changes in Rails 8.0/8.1 — regenerate from the Rails template if you have a custom CSP
- **Doorkeeper removal**: If migrating from Doorkeeper to a custom OAuth implementation, drop the old Doorkeeper tables (`oauth_access_grants`, `oauth_access_tokens`, `oauth_applications`) via migration, not by removing the gem first
- **`public_file_server.enabled`**: Changed from `false` to `true` in production — Thruster now handles static file serving. If you relied on nginx for this, review your reverse proxy config
