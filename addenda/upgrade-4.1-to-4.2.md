# Upgrade 4.1 → 4.2: Lessons Learned

## Detection Commands

### config/secrets.yml missing
```bash
# Rails 4.2 requires config/secrets.yml (replaces secret_token.rb)
ls config/secrets.yml
grep "config.secret_key_base" config/initializers/secret_token.rb
```

### Active Job integration
```bash
# Rails 4.2 introduced Active Job — check if background jobs need migration
grep -rn "Delayed::Job\|Resque\|Sidekiq" app/jobs/ config/
```

### respond_with / respond_to usage
```bash
# respond_with extracted to responders gem in 4.2
grep -rn "respond_with\|respond_to" app/controllers/
```

### Foreign key constraint usage
```bash
# Rails 4.2 added native foreign key support
grep -rn "add_foreign_key\|remove_foreign_key" db/migrate/
```

### Adequate Record optimizations
```bash
# Rails 4.2 includes Adequate Record — test performance-sensitive queries
# No specific detection needed, but be aware of changed query caching behavior
```

### Web Console gem
```bash
# Rails 4.2 added web-console for development debugging
grep "web-console" Gemfile
```

### Config for serve_static_assets
```bash
# Deprecated in Rails 5.0, but introduced in 4.2 era
grep "serve_static_assets\|serve_static_files" config/environments/
```

### Deprecated finder methods
```bash
# find_by_* dynamic finders deprecated (removed in 5.x)
grep -rn "find_by_[a-z_]\+(" app/models/ app/controllers/ | grep -v "find_by(" | head -20
```

## The Upgrade Timeline (Real-World)

One production app completed the Rails 4.1 → 4.2 upgrade in approximately **one day** (December 22, 2014), making it one of the smoothest Rails upgrades. The app stayed on Rails 4.2.x for **12 months** (Dec 2014 – Dec 2015) before upgrading to Rails 5.0.

### Version Path
| Version | Date | Notes |
|---------|------|-------|
| 4.1.0 | 2014-04-11 | "Pre-lim upgrate to 4.1" + finalization |
| 4.1.1 | 2014-05-20 | Patch update |
| 4.1.4 | 2014-07-04 | Patch update |
| 4.1.6 | 2014-10-03 | Final 4.1.x version (2.5 months) |
| **4.2.0** | **2014-12-22** | **"Initial update to Rails 4.2"** |
| 4.2.1 | 2015-04-30 | Patch update |
| 4.2.3 | 2015-07-07 | Patch update |
| 4.2.5 | 2015-12-16 | Final 4.2.x version |
| 5.0.0.beta1 | 2015-12-30 | Rails 5 beta (14 days later) |

**Key strategy**: This was a **single-day upgrade** with minimal code changes. The "Initial update to Rails 4.2" commit (1831a095a) updated 17 files in one commit. The app ran Rails 4.2.0 immediately in production without a long testing/beta period.

**Post-upgrade stability**: The upgrade happened on Dec 22, 2014. Between then and the next Rails patch (4.2.1 on April 30, 2015), there were **512 commits** — mostly feature work, bug fixes, and shop integrations. Only **one commit** explicitly mentioned Rails 4.2, and it was a point release update (4.2.3).

## config/secrets.yml Introduction (Rails 4.2 Requirement)

Rails 4.2 introduced `config/secrets.yml` to replace the older `config/initializers/secret_token.rb` pattern. This file is **required** for Rails 4.2 to boot.

```yaml
# config/secrets.yml (new file)
development:
  secret_key_base: <long hex string>

test:
  secret_key_base: <long hex string>

production:
  secret_key_base: <%= ENV["SECRET_KEY_BASE"] %>
```

**Migration path:**
```ruby
# Before (Rails 4.1 — config/initializers/secret_token.rb)
YourApp::Application.config.secret_key_base = 'long_hex_string_here'

# After (Rails 4.2 — config/secrets.yml)
# Move the secret to secrets.yml and delete secret_token.rb
```

One production app created `secrets.yml` with development/test keys committed to the repository, but production pulled from `ENV["SECRET_KEY_BASE"]`. This is the recommended pattern.

**Lesson**: The upgrade will **fail to boot** without `secrets.yml`. Rails 4.2's `rails new` generator creates this file automatically, but existing apps must create it manually during the upgrade.

## responders Gem Extraction

Rails 4.2 extracted `respond_with` and class-level `respond_to` to the `responders` gem. If your controllers use these methods, you must add the gem:

```ruby
# Gemfile — required for respond_with
gem 'responders', '~> 2.0'
```

**Detection:**
```bash
grep -rn "respond_with" app/controllers/
```

One production app added `responders` in the initial Rails 4.2 upgrade commit. Without this gem, any controller using `respond_with` will raise `NoMethodError`.

**Instance-level `respond_to`** (in controller actions) still works without the gem:
```ruby
# This still works without the responders gem
def show
  @product = Product.find(params[:id])
  respond_to do |format|
    format.html
    format.json { render json: @product }
  end
end
```

**Class-level `respond_to`** and `respond_with` require the gem:
```ruby
# Requires responders gem
class ProductsController < ApplicationController
  respond_to :html, :json  # Class-level — needs gem

  def show
    @product = Product.find(params[:id])
    respond_with @product  # respond_with — needs gem
  end
end
```

**Lesson**: If your app uses REST-style controllers with `respond_with`, add the gem immediately. If you only use instance-level `respond_to`, the gem is not needed.

## web-console Gem (Development Helper)

Rails 4.2 added `web-console` to the default Gemfile for development. This gem provides an interactive console in error pages and view templates.

```ruby
# Gemfile — development group
gem 'web-console', '~> 2.0'
```

One production app added it in the initial 4.2 upgrade commit. It was later commented out during the Rails 5.0 upgrade due to incompatibility with Rails 5 beta.

**Usage in views:**
```erb
<% console %>
```

This drops an interactive Ruby console into the rendered page for debugging.

**Lesson**: This gem is purely a development tool and optional. If your upgrade process includes disabling dev gems to focus on core compatibility, `web-console` can be safely skipped.

## Configuration Changes (Minimal)

The Rails 4.1 → 4.2 upgrade had remarkably minimal configuration changes.

### config/application.rb
```ruby
# Added (but commented out in one production app)
# config.active_record.raise_in_transactional_callbacks = true
```

This was a **transitional setting** for Rails 4.2 to opt into new behavior where after_commit/after_rollback callbacks run immediately instead of being deferred. It became the default (and only) behavior in Rails 5.0.

One production app **commented this out**, meaning they deferred adopting the new callback behavior until Rails 5.0 forced it.

### config/environments/development.rb
No significant changes. The file structure remained the same.

### config/environments/production.rb
No significant changes in one production app's upgrade.

### config/environments/test.rb
No significant changes in one production app's upgrade.

### config/boot.rb
Updated to match Rails 4.2 generator output (minor formatting changes, no functional differences).

### config/initializers/
- **mime_types.rb**: Updated to match Rails 4.2 defaults
- **session_store.rb**: No changes needed
- **inflections.rb**: Updated comments/formatting to match 4.2 defaults

**Lesson**: Rails 4.2 was one of the least disruptive upgrades for configuration files. Most changes were cosmetic (comments, formatting) rather than functional.

## Code Changes (Remarkably Few)

### Controllers

One production app made minimal controller changes:

**AlertsController** (`app/controllers/alerts_controller.rb`):
- Added `@active_alerts` and `@inactive_alerts` instance variables (feature work, not Rails-related)
- Added `format.js` responder to `destroy` action (feature work)

**API Controllers** (`app/controllers/api/v1/lists_controller.rb`):
- Removed SSL enforcement logging (`Rails.logger.error("API List#index without SSL error!")`)
- Changed authentication from email/password to auth_token lookup
- **No Rails 4.2-specific changes**

**Products API Controller** (`app/controllers/api/v1/products_controller.rb`):
- Massive refactoring — removed inline search logic (126 lines deleted)
- Replaced with `Search` service object: `booko_search = Search.new(query, product_type)`
- This was **feature work**, not a Rails 4.2 requirement

### Models
No model changes were required for the Rails 4.2 upgrade in one production app.

### Views
No view changes were required for the Rails 4.2 upgrade in one production app.

### Migrations
No migration changes were required. One production app had no migrations referencing new Rails 4.2 features (foreign keys, etc.).

**Lesson**: The Rails 4.1 → 4.2 upgrade was almost entirely a **Gemfile + config/secrets.yml change**. Code changes during this period were feature work, not upgrade requirements.

## Gemfile Changes

```diff
# Gemfile
-gem 'rails', '=4.1.6'
+gem 'rails', '=4.2.0'

+gem 'responders', '~> 2.0'

group :development do
+  gem 'web-console', '~> 2.0'
   # ...
end
```

**No gems were removed or disabled** during this upgrade, unlike the Rails 5.0 upgrade which required commenting out several incompatible gems.

## Rails 4.2 Features: Adoption Decisions

| Feature | Adopted? | Notes |
|---------|----------|-------|
| **Active Job** | Skipped | Continued using existing Beanstalkd integration directly |
| **config/secrets.yml** | Yes | Required for Rails 4.2 to boot |
| **Foreign key support** | Skipped | No migrations added foreign keys during this period |
| **Adequate Record** | Automatic | Performance improvements applied automatically |
| **responders gem** | Yes | Required for existing `respond_with` usage |
| **web-console** | Yes | Added to development group |
| **GlobalID / Signed GlobalID** | Skipped | Not actively used |
| **raise_in_transactional_callbacks** | Skipped | Commented out, deferred to Rails 5.0 |

**Lesson**: Rails 4.2 had several headline features (Active Job, foreign keys, Adequate Record), but none were **required** for the upgrade. A conservative approach that only updates the Rails version and adds `secrets.yml` + `responders` is valid.

## Rails 4.2.x Point Releases (Stability)

Between Rails 4.2.0 (Dec 22, 2014) and Rails 5.0 (Dec 30, 2015), one production app went through these point releases:

- **4.2.0 → 4.2.1** (April 30, 2015): 4 months on 4.2.0
- **4.2.1 → 4.2.3** (July 7, 2015): 2 months on 4.2.1 (skipped 4.2.2)
- **4.2.3 → 4.2.5** (Dec 16, 2015): 5 months on 4.2.3 (skipped 4.2.4)

Each point release was a **Gemfile-only change** (2 lines: `gem 'rails'` version bump, Gemfile.lock regeneration). No code or config changes were needed.

**Lesson**: Rails 4.2.x had excellent stability. One production app accumulated **512 commits** of feature work between 4.2.0 and 4.2.3 with no Rails-related issues.

## Interleaved Work (The Hidden Complexity)

The git history shows the Rails 4.2 upgrade was done on a `pg` branch that also handled a **MySQL → PostgreSQL database migration**. Between the last Rails 4.1 commit (Oct 3, 2014) and the Rails 4.2 commit (Dec 22, 2014), there were multiple merges between `master` and the `pg` branch:

- `accb212d6` (Nov 19, 2014): "Merge branch 'master' into pg"
- `a1ae66133` (Dec 22, 2014): "Merge branch 'master' into pg"
- Then Rails 4.2 upgrade commit (same day)

This suggests the Rails 4.2 upgrade was **piggybacked onto an active database migration branch**, which is **not recommended**. However, in this case it worked because Rails 4.2 required minimal changes.

**Lesson**: Rails 4.2 was simple enough that it could be combined with other infrastructure work without causing problems. This would **not** have worked for Rails 5.0 (which required 8 months of branch maintenance and extensive compatibility fixes).

## Subtle Gotchas

- **secrets.yml is mandatory**: Without this file, Rails 4.2 **will not boot**. The error message is clear (`Missing `secret_key_base` for 'production' environment`), but it's still a surprise if you're not expecting it

- **responders gem is easy to miss**: If your app uses `respond_with` in just a few controllers, you might not notice it's missing until you visit those specific endpoints. The error (`undefined method 'respond_with'`) is clear, but the fix is not immediately obvious if you're not aware of the gem extraction

- **raise_in_transactional_callbacks deferral**: One production app **commented this out**, meaning they deferred the new callback behavior. This is fine, but it means callbacks run at a different point in the transaction lifecycle. If your app relies on precise callback timing (e.g., sending emails in after_commit), be aware of the behavioral change

- **Active Job adoption pressure**: Rails 4.2 introduced Active Job as the "Rails Way" for background jobs. If you have an existing job system (Beanstalkd, Sidekiq, Resque), there may be pressure to migrate. However, this is **not required** — existing job systems continue to work fine

- **Adequate Record performance changes**: Rails 4.2's Adequate Record optimization changes query caching behavior. If your app relies on specific query caching patterns (e.g., expecting certain queries to hit the database every time), test thoroughly

- **Foreign key constraints**: Rails 4.2 added native foreign key support, but this is **opt-in**. Existing apps without foreign keys will continue to work. However, this is a good time to consider adding them for data integrity

- **4.2.0 shipped on Dec 19, 2014**: One production app upgraded on Dec 22, **3 days after GA release**. This is remarkably fast for a major Rails upgrade. Rails 4.2 was stable enough that early adoption was low-risk

- **One-year lifespan**: Rails 4.2.0 was released Dec 19, 2014. Rails 5.0.0 was released June 30, 2016 (18 months later). One production app stayed on Rails 4.2.x for exactly 12 months before starting the Rails 5.0 beta cycle, which is a reasonable upgrade cadence

- **No branch strategy needed**: Unlike Rails 5.0 (which needed a long-lived feature branch tracking betas), Rails 4.2 could be upgraded in a single commit on `master`. The simplicity of the upgrade meant no extended testing period was required
