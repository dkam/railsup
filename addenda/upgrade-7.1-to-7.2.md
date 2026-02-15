# Upgrade 7.1 → 7.2: Lessons Learned

## Code Pattern Changes

### Enum syntax (COMMON - most apps need this)

**Detection:**
```bash
# Find old enum syntax (attribute as keyword arg)
grep -rn "enum\s\+\w\+:" app/models/
# Find _prefix or _suffix options
grep -rn "_prefix:\|_suffix:" app/models/
```

**Fix:**
```ruby
# Old
enum status: [:pending, :active, :archived]
enum action: { save: "save", cancel: "cancel" }, _prefix: :action

# New - add colon before attribute, comma after, remove underscore from options
enum :status, [:pending, :active, :archived]
enum :action, { save: "save", cancel: "cancel" }, prefix: :action
```

### Serialize requires explicit coder

**Detection:**
```bash
grep -rn "serialize\s*:" app/models/ | grep -v "coder:"
```

**Fix:**
```ruby
# Old
serialize :settings, type: Hash

# New
serialize :settings, type: Hash, coder: YAML
```

### config.cache_classes → config.enable_reloading

**Detection:**
```bash
grep -rn "cache_classes" config/environments/
```

**Fix:**
```ruby
# config/environments/development.rb
# Old: config.cache_classes = false
config.enable_reloading = true

# config/environments/production.rb
# Old: config.cache_classes = true
config.enable_reloading = false
```

## Detection Commands

### params comparison (CRITICAL breaking change)
```bash
grep -rn "params.*==" app/controllers/ app/graphql/ app/services/ | grep -v "to_h\|permit\|params\.require"
```

### .connection deprecation
```bash
grep -rn "\.connection[^_]" app/ lib/ | grep -v "with_connection\|lease_connection\|@connection\s*=\s*Faraday\|@connection\s*=\s*nil"
```

### show_exceptions boolean values
```bash
grep -rn 'show_exceptions\s*=\s*\(true\|false\)' config/environments/
```

### MissingHelperError constant removal
```bash
grep -rn "MissingHelperError" app/ spec/ test/
```

### fixture_path (singular) removal
```bash
grep -rn "fixture_path[^s]" spec/ test/
```

### Dalli::Client direct usage
```bash
grep -rn "Dalli::Client" config/
# Old: config.cache_store = :mem_cache_store, Dalli::Client.new('localhost')
# New: config.cache_store = :mem_cache_store, 'localhost:11211'
```

### Deprecation without deprecator
```bash
grep -rn "deprecate " app/ lib/ | grep -v "deprecator:"
# Old: deprecate :old_method
# New: deprecate :old_method, deprecator: Rails.deprecator
```

### config/secrets.yml removal
```bash
ls config/secrets.yml 2>/dev/null && echo "⚠️  Remove this file, migrate to Rails.application.credentials"
```

### Existing queue adapter detection
```bash
# Identify current queue adapter before considering Solid Queue
grep -rn "queue_adapter\|good_job\|sidekiq\|resque" config/ Gemfile
```

### Advisory locks and prepared statements
```bash
# Check for PgBouncer or connection pooler usage that conflicts with advisory locks
grep -rn "advisory_locks\|prepared_statements\|pgbouncer" config/database.yml
```

### CSV stdlib dependency
```bash
# Ruby 3.4+ removed CSV from stdlib — check if any gem requires it
grep -rn 'require.*csv' app/ lib/ config/
bundle exec ruby -e "require 'csv'" 2>&1 | grep -i "error"
```

### Turbo Streams broadcast serialization
```bash
# Check for broadcasts_refreshes which may break with strict Sidekiq args
grep -rn "broadcasts_refreshes\|broadcasts_to" app/models/
```

## Gem-Specific Gotchas

- **`mutex_m` removed from ActiveSupport dependencies**: Ruby 3.4 removed `mutex_m` from stdlib. If any gem depends on it transitively through ActiveSupport, add `gem "mutex_m"` to your Gemfile
- **`csv` removed from Ruby stdlib**: Add `require "csv"` in `config/application.rb` if any gem needs it (e.g., meilisearch-rails). Alternatively add `gem "csv"` to Gemfile
- **`mail` gem minimum bumped to 2.8.0**: Check `Gemfile.lock` — if pinned below 2.8.0, update it
- **`concurrent-ruby` minimum bumped to 1.3.1**: May conflict with older gems that pin `concurrent-ruby`
- **`useragent` gem now required by ActionPack**: Added automatically but may conflict if you have a different `useragent` gem
- **`solid_cache` major version bump**: If upgrading from 0.x to 1.x, review the changelog for breaking changes
- **Sidekiq `strict_args!` and Turbo Streams**: If using `broadcasts_refreshes` with Sidekiq, disable strict argument checking:
  ```ruby
  # config/initializers/sidekiq.rb
  Sidekiq.strict_args!(false)
  ```
  See: https://github.com/hotwired/turbo-rails/issues/522

## Multi-Database Setup for Solid Gems

Rails 7.2 is commonly paired with Solid Queue and Solid Cache adoption. This requires multi-database configuration.

### database.yml structure

```yaml
default: &default
  adapter: postgresql
  encoding: unicode
  pool: <%= ENV.fetch("RAILS_MAX_THREADS") { 50 } %>

development:
  primary:
    <<: *default
    database: myapp_dev
  queue:
    <<: *default
    database: myapp_queue_dev
    migrations_paths: db/queue_migrate

production:
  primary:
    <<: *default
    database: myapp_production
    prepared_statements: <%= ENV.fetch("PREPARED_STATEMENTS", true) %>
    advisory_locks: <%= ENV.fetch("ADVISORY_LOCKS", true) %>
  queue:
    <<: *default
    database: solid_queue
    migrations_paths: db/queue_migrate
  cache:
    <<: *default
    database: solid_cache
    migrations_paths: db/cache_migrate
```

### PgBouncer compatibility (CRITICAL)

If using PgBouncer in transaction pooling mode, **workers** must disable prepared statements and advisory locks:

```yaml
production:
  primary:
    prepared_statements: <%= ENV.fetch("PREPARED_STATEMENTS", false) %>
    advisory_locks: <%= ENV.fetch("ADVISORY_LOCKS", false) %>
```

**Why:** PgBouncer in transaction mode reassigns connections between requests. Prepared statements are connection-specific and advisory locks are session-specific — both break silently or cause deadlocks under PgBouncer.

**Detection:**
```bash
# Check if using PgBouncer (non-standard ports are a clue)
grep -n "port:" config/database.yml | grep -v "5432"
```

**Real-world gotcha:** One project had migration deployments deadlocking because advisory locks conflicted with PgBouncer. The fix was `advisory_locks: false` via environment variable so web servers could still use them while workers could not.

### Connection pool sizing

Increase the default connection pool when adding Solid Queue workers:
```yaml
pool: <%= ENV.fetch("RAILS_MAX_THREADS") { 50 } %>  # was 5
```

Solid Queue workers each hold a database connection, so the pool must accommodate both web threads and worker threads.

### Environment config for Solid Queue

Each environment needs the Solid Queue database connection:
```ruby
# config/environments/production.rb
config.solid_queue.connects_to = { database: { writing: :queue } }

# config/environments/development.rb
config.solid_queue.connects_to = { database: { writing: :queue } }
```

### Queue adapter swap

```ruby
# config/application.rb
# Old
config.active_job.queue_adapter = :good_job
config.good_job.execution_mode = :external

# New
config.active_job.queue_adapter = :solid_queue
```

### Mission Control Jobs

If using Mission Control for job monitoring, secure it behind an admin controller:
```ruby
# config/application.rb
config.mission_control.jobs.base_controller_class = "AdminController"
```

```ruby
# config/routes.rb
mount MissionControl::Jobs::Engine, at: "/jobs"
```

## Edge Cases

### includes bug with scoped associations (Rails bug - still exists)

When using `includes` with associations that have scopes containing `where` clauses, Rails 7.2 may raise `NoMethodError` in `references_eager_loaded_tables?`.

**Symptom:** NoMethodError from deep in ActiveRecord when using includes.

**Workaround:** Use `preload` instead:
```ruby
# Broken in Rails 7.2
current_account.tests.includes(:project, tasks: [:overdue_task])

# Workaround
current_account.tests.preload(:project, tasks: [:overdue_task])
```

**Trade-off:** `preload` always uses separate queries instead of LEFT OUTER JOIN. Only apply where you hit the bug.

### Transaction-aware job enqueuing timing change
```bash
# Detect jobs enqueued inside transactions
grep -rn "\.transaction do" app/models/ -A 5 | grep -l "perform_later\|perform_now"
```

If timing is critical, set on affected job class:
```ruby
self.enqueue_after_transaction_commit = :never
```

### Connection pool methods now require role argument
```bash
grep -rn "clear_active_connections\|clear_reloadable_connections\|clear_all_connections!" app/ lib/
```

```ruby
# Old
ActiveRecord::Base.clear_active_connections!
# New
ActiveRecord::Base.clear_active_connections!(:writing)
```

### Recursive CTE method conflicts

Rails 7.2 introduced native `with_recursive` that only supports UNION ALL. If you have custom `with_recursive` implementations or need UNION (without ALL) for deduplication, rename your method to avoid conflicts.

### SQL date casting syntax

PostgreSQL casting syntax is more reliable in Rails 7.2:
```ruby
# Old
where("submitted_at between date :start and date :end", ...)
# New
where("submitted_at between :start::date and :end::date", ...)
```

### Docker server.pid stale file

Containers restarting may fail with "server already running" due to a stale PID file. Add cleanup to your entrypoint:
```bash
# bin/docker-entrypoint
if [ -f tmp/pids/server.pid ]; then
  rm tmp/pids/server.pid
fi
```

### GraphicsMagick as MiniMagick backend

If switching from ImageMagick to GraphicsMagick for better performance:
```ruby
# config/initializers/mini_magick.rb
MiniMagick.configure do |config|
  config.cli = :graphicsmagick
end
```

**Note:** GraphicsMagick has different format support than ImageMagick. Test image processing thoroughly, especially for newer formats like JXL and AVIF.

## Multi-Database: ApplicationRecord connects_to

When using a multi-database setup with primary/replica, you **must** explicitly set `connects_to` in `ApplicationRecord`:

```ruby
# app/models/application_record.rb
class ApplicationRecord < ActiveRecord::Base
  self.abstract_class = true
  connects_to database: { writing: :primary, reading: :primary }
end
```

Without this, Rails may route queries to the wrong database when you have multiple databases configured (e.g., primary + queue + cache). This is easy to miss because it works in development (single database) but fails in production (multiple databases).

## SolidCache: Production Stability Warning

During the 7.1→7.2 era, one production app attempted to adopt SolidCache and went through multiple iterations:
1. Added SolidCache on a standalone host
2. Tried different cache sizes (shrinking, then growing)
3. Used a separate config file for Solid Cache
4. **Switched back to Redis cache** due to stability issues
5. Later switched to SolidCache again, then eventually to DragonflyDB in the 8.0 era

**Lesson**: If adopting SolidCache during 7.2, start with it on a separate database host and have a Redis fallback plan ready. Monitor connection pool usage closely.

## Subtle Gotchas

- **`autoload_lib` syntax**: Rails 7.2 prefers `%w[]` over `%w()` — not breaking, but generators will use square brackets
- **ActiveStorage service name changes**: If renaming storage services (e.g., `:local` → `:bb_covers`), update every environment config file — a mismatch causes `ActiveStorage::ServiceNotFoundError` at boot
- **Registration defaults**: Review any `ENV.fetch` defaults that change security posture (e.g., flipping registration from enabled to disabled by default)
- **Redis port changes**: If restructuring infrastructure alongside the upgrade, update `config/cable.yml` and any Redis-dependent initializers
- **Environment variable typos**: Double-check env var names in `database.yml` — typos like `PREPARTED_STATEMENTS` instead of `PREPARED_STATEMENTS` will silently use the default value instead of the intended override
- **Meilisearch config**: If upgrading meilisearch-rails, the configuration keys changed from `MEILI_HOST`/`MEILI_PORT`/`MEILI_KEY` to `MEILI_URL`/`MEILI_MASTER_KEY`
- **ngrok / tunnel hosts**: Rails 7.2 enforces `config.hosts` more strictly. Add tunnel domains to `config.hosts` in development or requests will be blocked with `Blocked host` errors
