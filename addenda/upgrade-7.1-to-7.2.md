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

## Gem-Specific Gotchas

- **`mail` gem minimum bumped to 2.8.0**: Check `Gemfile.lock` — if pinned below 2.8.0, update it
- **`concurrent-ruby` minimum bumped to 1.3.1**: May conflict with older gems that pin `concurrent-ruby`
- **`useragent` gem now required by ActionPack**: Added automatically but may conflict if you have a different `useragent` gem

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

## Subtle Gotchas

- **`autoload_lib` syntax**: Rails 7.2 prefers `%w[]` over `%w()` — not breaking, but generators will use square brackets
- **ngrok / tunnel hosts**: Rails 7.2 enforces `config.hosts` more strictly. Add tunnel domains to `config.hosts` in development or requests will be blocked with `Blocked host` errors
