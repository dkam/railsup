# Upgrade 5.1 → 5.2: Lessons Learned

## Detection Commands

### ActiveRecord::Base instead of ApplicationRecord
```bash
# Rails 5.0 introduced ApplicationRecord but many apps deferred migration
grep -rn "< ActiveRecord::Base" app/models/
```

### dalli_store cache namespace (pre-6.0 cache busting)
```bash
# If upgrading cache format, change the namespace to bust stale entries
grep -rn "cache_store.*namespace" config/environments/
```

### Fixnum usage (Ruby 2.4 deprecated)
```bash
# Fixnum was unified into Integer in Ruby 2.4 (commonly upgraded alongside 5.2)
grep -rn "Fixnum" app/ lib/ test/
```

### cached_find / cached_find_by patterns
```bash
# Dynamic finders like find_by_* were deprecated; cached wrappers doubly so
grep -rn "cached_find\|find_by_\w\+" app/ | grep -v "find_by(" | grep -v "find_by_sql"
```

### factory_girl gem (renamed to factory_bot)
```bash
grep -rn "factory_girl" Gemfile test/ spec/
```

### secrets.yml usage (5.2 introduces credentials)
```bash
# Check if still using secrets.yml instead of credentials
ls config/secrets.yml config/credentials.yml.enc 2>/dev/null
```

### config.load_defaults missing or old
```bash
# Rails 5.2 apps should have config.load_defaults 5.2
grep -rn "load_defaults" config/application.rb
```

### ActiveStorage pulled in as dependency
```bash
# Rails 5.2 adds activestorage to the rails gem — check if it's being used or just present
grep -rn "active_storage\|activestorage" Gemfile.lock config/
```

## The Upgrade Timeline (Real-World)

One production app took approximately **5 months** from first beta testing to completion (Dec 2017 – Apr 2018), using a long-lived `r52b` feature branch with periodic merges from master.

### Version Path
| Version | Date | Notes |
|---------|------|-------|
| 5.1.1 → 5.1.3 | 2017-08-23 | Patch update on master |
| 5.1.3 → 5.1.4 | late 2017 | Patch update on master |
| 5.2.0.beta2 | 2017-12-22 | First 5.2 test on `r52b` branch |
| 5.2.0.rc1 | 2018-02-06 | RC1 testing, cache namespace changed |
| 5.2.0.rc2 | 2018-03-22 | RC2, deprecated SQL removed |
| 5.1.4 → 5.1.6 | 2018-04-03 | Parallel patch maintenance on master |
| 5.2.0 | 2018-04-10 | GA release, timezone change, Rubocop added |
| — | 2018-04-17 | **MERGE: r52b branch merged into master** |

**Key strategy**: Master was kept on Rails 5.1.x throughout the upgrade period, receiving patch updates independently. The `r52b` branch tracked every beta/RC release. The final merge was just a Gemfile.lock update — all code changes had been done incrementally.

**Branch strategy**: Long-lived `r52b` feature branch with at least 3 merges from master over 2 months.

## ActiveRecord::Base → ApplicationRecord

Rails 5.0 introduced `ApplicationRecord` as the base class for models, but many apps deferred this migration. One production app did it during the 5.2 upgrade — **all 24 models updated in a single commit**.

```ruby
# Before (every model)
class Product < ActiveRecord::Base

# After
class Product < ApplicationRecord
```

**Detection:**
```bash
grep -rn "< ActiveRecord::Base" app/models/ | wc -l
```

**Lesson**: Do this migration early (it's mechanical and safe) rather than carrying the tech debt through multiple upgrades. It's a prerequisite for many Rails 5.x+ features.

## factory_girl → factory_bot

The FactoryGirl gem was renamed to FactoryBot. This is a straightforward find-and-replace:

```ruby
# Gemfile
# Before
gem 'factory_girl'
gem 'factory_girl_rails'

# After
gem 'factory_bot'
gem 'factory_bot_rails'

# test_helper.rb
# Before
require 'factory_girl_rails'
include FactoryGirl::Syntax::Methods

# After
require 'factory_bot_rails'
include FactoryBot::Syntax::Methods
```

## Cache Namespace Busting

When upgrading Rails, change your cache namespace to prevent stale/incompatible cached data from being read:

```ruby
# Before
config.cache_store = :dalli_store, '127.0.0.1', { namespace: 'rails5', compress: true }

# After (on 5.2 branch)
config.cache_store = :dalli_store, '127.0.0.1', { namespace: 'r52', compress: true }
```

One production app changed namespaces from `'pg'` → `'r52'` → `'rails6'` across successive upgrades. This is cheap insurance against serialization incompatibilities.

## Timezone Configuration Change

Rails 5.2 is a good time to switch from local timezone to UTC for ActiveRecord:

```ruby
# config/application.rb
# Before
config.active_record.default_timezone = :local

# After
config.active_record.default_timezone = :utc
```

**Warning**: This was a contentious change in one production app — committed, reverted ("Set timezone back to local"), then re-applied. If your database has existing timestamps in local time, this change affects how they're read. Plan the migration carefully:

1. Audit all timestamp columns for existing data
2. Consider a data migration to convert existing timestamps
3. Test thoroughly — timezone bugs are subtle and often only surface in edge cases (DST transitions, cross-timezone queries)

## Deprecated SQL Patterns

Rails 5.2 (via Arel 9.0) deprecated certain raw SQL patterns in scopes and queries:

```ruby
# Before (deprecated)
scope :ordered, -> { order("COALESCE(prices.found_at, '1970-01-01') DESC") }

# After — use Arel or restructure
scope :ordered, -> { order(Arel.sql("COALESCE(prices.found_at, '1970-01-01') DESC")) }
```

**Detection:**
```bash
# Find raw SQL strings in order/where/having clauses
grep -rn "\.order(\"\|\.order('" app/models/ | grep -v "Arel.sql"
grep -rn "\.where(\"\|\.where('" app/models/ | grep -v "?"
```

One production app commented out affected ORDER BY clauses during RC2 testing rather than wrapping them in `Arel.sql()` — a quick fix that may leave functionality disabled.

## Dynamic Finders Removal

Rails 5.2 continued removing dynamic finders like `find_by_*`:

```ruby
# Before (deprecated)
User.find_by_auth_token!(cookies.signed[:auth_token])
User.find_by_id(access_key)

# After
User.find_by(auth_token: cookies.signed[:auth_token])
User.find_by(id: access_key)
```

**Also**: The bang variant (`find_by_*!`) was changed to the non-bang version in one production app's authentication code, changing behavior from raising `ActiveRecord::RecordNotFound` to returning `nil`. Make sure this is intentional:

```ruby
# Changed from raising to returning nil — is this the desired behavior?
@current_user ||= User.find_by_auth_token(cookies.signed[:auth_token])
# vs
@current_user ||= User.find_by(auth_token: cookies.signed[:auth_token])
```

## Cover Model / Image Handling (Instead of ActiveStorage)

Rails 5.2 introduced ActiveStorage, but one production app built a custom `Cover` model instead. This is a valid choice when:
- You have an existing image pipeline (CDN, cover art servers)
- You need SVG placeholder generation
- You need ETag/Last-Modified caching control
- Your images aren't user-uploaded files

**Lesson**: Don't feel obligated to adopt ActiveStorage just because it shipped with 5.2. If your existing image handling works, keep it.

## config.load_defaults Debt

One production app had **no `config.load_defaults` setting at all** during the 5.2 upgrade. The setting was added later and then immediately commented out because it enabled `belongs_to_required_by_default`, which broke the app.

```ruby
# This pattern shows the evolution of config.load_defaults debt:
# config.load_defaults 5.0    # Added then commented out
# config.load_defaults 5.2    # Added then commented out
config.load_defaults 6.0       # Finally kept, but after fixing belongs_to issues
```

**Lesson**: Each skipped `load_defaults` version accumulates configuration debt. When you eventually enable a later version, you get hit by all the defaults you've been avoiding. Prefer enabling each version's defaults during its upgrade.

## Rails 5.2 Features: Adoption Decisions

| Feature | Adopted? | Notes |
|---------|----------|-------|
| **ActiveStorage** | Skipped | Custom Cover model used instead |
| **Credentials** | Skipped | Continued with `config/secrets.yml` |
| **Content Security Policy DSL** | Skipped | No CSP initializer created |
| **Redis Cache Store** | Skipped | Continued with Memcached/Dalli |
| **Bootsnap** | Skipped | No changes to `config/boot.rb` |
| **Early Hints** | Skipped | No evidence of adoption |

**Lesson**: A conservative upgrade that only changes the Rails version — without adopting new features — is a valid strategy. Adopt features in separate PRs after the upgrade is stable.

## Concurrent Work Patterns

The 5.1→5.2 upgrade period included significant concurrent work that was mixed into the upgrade branch:
- PostgreSQL migration completion (long-running `pg` branch finally merged)
- 5 new shop integrations (Audible GB, Booky, Catch, Kennys, Reformers)
- Cover art system built from scratch
- Currency exchange provider migration (to OpenExchangeRates)
- Chartkick integration
- Rubocop adoption

**Lesson**: Avoid building features on the upgrade branch. Keep the upgrade branch focused on compatibility fixes and merge feature work from master. One production app's 987-commit diff between tags made it nearly impossible to separate upgrade changes from feature work retroactively.

## Subtle Gotchas

- **`activestorage` in Gemfile.lock**: Even if you don't use ActiveStorage, it appears in your Gemfile.lock as a dependency of the `rails` gem. This pulls in `marcel` for MIME type detection. It's harmless but can confuse dependency audits

- **`sanitize` gem version conflicts**: Rails 5.2 bundles `loofah` through `rails-html-sanitizer`. If you have a direct dependency on the `sanitize` gem, version conflicts may force a downgrade or a switch to Loofah directly

- **VCR test framework compatibility**: VCR may need updating or reconfiguration for Rails 5.2. One production app disabled it entirely during the upgrade

- **Secrets in config/secrets.yml**: If you defer the credentials migration, at least move secrets to environment variables. One production app had API keys committed directly to `secrets.yml` across all environments

- **Minitest version pinning**: Rails 5.2 may pull in a newer Minitest that changes test behavior. One production app pinned `minitest` to `'5.10.3'` to avoid surprises

- **`association(true)` reload patterns**: If you missed fixing these during the 5.1 upgrade, they'll definitely break in 5.2. The pattern is `model.association(true)` → `model.association.reload`
