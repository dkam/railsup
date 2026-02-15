# Upgrade 4.2 → 5.0: Lessons Learned

## Detection Commands

### before_filter / after_filter usage
```bash
# Rails 5 renamed these to before_action / after_action
grep -rn "before_filter\|after_filter\|around_filter\|skip_before_filter\|skip_after_filter" app/controllers/
```

### render nothing: true / render text:
```bash
# render nothing: true removed in Rails 5 (use head :ok)
grep -rn "render.*nothing:" app/controllers/
# render text: deprecated (use render plain:)
grep -rn "render.*:text\s*=>\|render.*text:" app/controllers/ app/views/
```

### Association force-reload syntax
```bash
# association(true) deprecated in 5.0, removed in 5.1
grep -rn "\.\w\+(true)" app/models/ | grep -v "\.save\|\.valid\|\.destroy\|\.delete\|\.blank\|\.present\|\.nil\|\.empty\|\.freeze\|\.dup\|\.include"
```

### ActiveRecord::Base instead of ApplicationRecord
```bash
# Rails 5.0 introduced ApplicationRecord as the new base class
grep -rn "< ActiveRecord::Base" app/models/
```

### Catch-all dynamic segment routes
```bash
# Rails 5 removed support for catch-all controller/action routes
grep -rn '":controller\|:controller(' config/routes.rb
```

### Test keyword arguments (params:, session:)
```bash
# Rails 5 requires keyword args for test HTTP methods
grep -rn "get(\s*:\w\+,\s*{" test/controllers/
grep -rn "post(\s*:\w\+,\s*{" test/controllers/
# Also: get_via_redirect removed
grep -rn "get_via_redirect\|post_via_redirect" test/
```

### serve_static_files deprecation
```bash
# Renamed to config.public_file_server.enabled
grep -rn "serve_static_files\|serve_static_assets" config/environments/
```

### raise_in_transactional_callbacks
```bash
# Removed in Rails 5 — was a transitional setting from 4.2
grep -rn "raise_in_transactional_callbacks" config/
```

### ActiveModel::Serializer API changes
```bash
# AMS 0.10 renamed ArraySerializer to CollectionSerializer
grep -rn "ArraySerializer" app/
```

### Incompatible gems
```bash
# These commonly need updates or disabling for Rails 5
grep -rn "rack-attack\|will_paginate\|web-console\|better_errors\|binding_of_caller" Gemfile
```

## The Upgrade Timeline (Real-World)

One production app took approximately **8 months** from initial beta testing to completion (Dec 2015 – Aug 2016), using a long-lived `rails5` branch that tracked every beta and RC release. The upgrade was interleaved with a concurrent MySQL-to-PostgreSQL database migration on a `pg` branch.

### Version Path
| Version | Date | Notes |
|---------|------|-------|
| 4.2.5 | 2015-12-19 | Last commit on Rails 4.2 |
| 5.0.0.beta1 | 2015-12-30 | `rails5` branch created |
| 5.0.0.beta2 | 2016-02-10 | Beta tracking continues |
| 5.0.0.beta3 | 2016-03-16 | Merged master, upgraded to beta3 |
| — | 2016-05-05 | Merge branch 'master' into rails5 |
| — | 2016-05-24 | Merge branch 'master' into rails5 |
| 5.0.0 | 2016-07-01 | GA release — updated from beta next day |
| — | 2016-07-07 | Major batch of deprecation fixes |
| — | 2016-07-13 | Config and controller fixes |
| 5.0.0.1 | 2016-08-17 | Security patch applied |
| — | 2016-08-16–18 | **Landing: 3-day sprint to merge into master** |
| 5.0.1 | 2017-01-06 | Patch update (also Ruby 2.3.1 → 2.4.0) |
| 5.0.2 | 2017-03-15 | Patch update |
| 5.0.3 | 2017-05-13 | Patch update |

## before_filter → before_action (Most Widespread)

Rails 5 renamed all callback filter methods. This is typically one of the most widespread changes in the upgrade.

```ruby
# Before (Rails 4.2)
before_filter :authenticate_user
after_filter :log_campaigns
skip_before_filter :verify_authenticity_token

# After (Rails 5.0)
before_action :authenticate_user
after_action :log_campaigns
skip_before_action :verify_authenticity_token
```

**Detection:**
```bash
grep -rn "before_filter\|after_filter\|around_filter" app/controllers/
```

**Lesson**: This is a mechanical find-and-replace but easy to miss in less-visited controllers. Do it in one sweep.

## render nothing: true / render text: Removal

### render nothing: true → head
```ruby
# Before (Rails 4.2)
render(status: :forbidden, nothing: true)
render nothing: true, status: 200

# After (Rails 5.0)
head :forbidden
head :ok
```

### render text: → render plain:
```ruby
# Before (Rails 4.2)
render(status: :unauthorized, text: 'User Authentication Failed')

# After (Rails 5.0)
render(status: :unauthorized, plain: 'User Authentication Failed')
```

**Detection:**
```bash
grep -rn "render.*nothing:" app/controllers/
grep -rn "render.*:text\s*=>" app/controllers/ app/views/
grep -rn "render.*text:" app/controllers/ | grep -v "plain:"
```

**Lesson**: Search for both hash-rocket (`:text =>`) and symbol-key (`text:`) syntax. One production app had both styles scattered across controllers and JS.erb views.

## ApplicationRecord Base Class

Rails 5.0 introduced `ApplicationRecord` as the recommended base class for all models. The file was created in the very first commit of the upgrade:

```ruby
# app/models/application_record.rb (new file)
class ApplicationRecord < ActiveRecord::Base
  self.abstract_class = true
end
```

One production app created the file early but only migrated **one model** (`Price`) during the 5.0 upgrade. The remaining ~24 models were migrated later during the 5.2 upgrade.

```ruby
# Before
class Price < ActiveRecord::Base

# After
class Price < ApplicationRecord
```

**Lesson**: The file must exist for Rails 5, but migrating all models is not strictly required for the upgrade to work. However, deferring the migration creates tech debt — doing it incrementally or all-at-once during the 5.0 upgrade is recommended.

## Catch-All Dynamic Segment Routes Removed

Rails 5 removed support for the classic Rails 3/4 catch-all route pattern:

```ruby
# config/routes.rb — REMOVED in Rails 5
get ":controller(/:action(/:id(.:format)))"
get ":controller(/:action(.:format))"
```

One production app had these catch-all routes serving several user-facing endpoints. They all needed explicit route definitions:

```ruby
# Routes previously served by catch-all now need explicit definitions
get 'users/bits'
get 'users/toolbar'
get 'users/oauth_login'
get 'users/oauth'
get 'users/complete'
get 'users/index'
```

**Detection:**
```bash
grep -rn '":controller\|:controller(' config/routes.rb
```

**Also affected**: `url_for` with dynamic segments was deprecated. One production app had to replace:
```ruby
# Before (Rails 4.2)
url_for(controller: 'users', action: 'index', only_path: false, protocol: 'http')

# After (Rails 5.0)
users_url
```

**Lesson**: If your app has catch-all routes, you'll need to add explicit routes for every controller/action combination that was previously served implicitly. Audit your app for `url_for` calls using `:controller`/`:action` parameters.

## Test Framework Changes (Major)

Rails 5 made several breaking changes to the test framework.

### Keyword Arguments Required

All test HTTP methods now require `params:` and `session:` keyword arguments:

```ruby
# Before (Rails 4.2 — positional arguments)
get :show, nil, { user_id: user.id }
post :update, { user: { name: 'New' } }, { user_id: user.id }
get(:index, { id: prod.gtin, format: :json })

# After (Rails 5.0 — keyword arguments)
get :show, session: { user_id: user.id }
post :update, params: { user: { name: 'New' } }, session: { user_id: user.id }
get(:index, params: { id: prod.gtin, format: :json })
```

### ActionController::TestCase → ActionDispatch::IntegrationTest

Controller tests should inherit from `ActionDispatch::IntegrationTest` instead of `ActionController::TestCase`:

```ruby
# Before (Rails 4.2)
class ProductsControllerTest < ActionController::TestCase
  test "should get show" do
    get :show, id: product.id
  end
end

# After (Rails 5.0)
class ProductsControllerTest < ActionDispatch::IntegrationTest
  test "should get show" do
    get product_path(product)
  end
end
```

Key differences:
- Use **route helpers** (`product_path`, `alerts_path`) instead of action symbols (`:show`, `:index`)
- Use **`sign_in(user)`** helper instead of setting session directly
- Session manipulation is no longer directly available — use helper methods

### get_via_redirect Removed

```ruby
# Before (Rails 4.2)
get_via_redirect '/alerts'

# After (Rails 5.0)
get '/alerts'
follow_redirect!
```

### rails-controller-testing Gem

If your tests use `assigns()` or `assert_template()`, add the extraction gem:

```ruby
# Gemfile — test group
gem 'rails-controller-testing'
```

These methods were extracted from Rails core in Rails 5. The gem provides backward compatibility, but migrating away from these assertions is recommended.

**Lesson**: The test migration is the most time-consuming part of the Rails 5 upgrade. One production app had multiple commits across several days fixing test files. Consider converting tests incrementally — `ActionController::TestCase` still works in Rails 5 (just deprecated), so you can migrate tests file-by-file.

## Association Force-Reload Syntax

Passing `true` to association methods for force reload was deprecated in Rails 5.0:

```ruby
# Before (Rails 4.2)
prod.prices(true)
product.prices(true)

# After (Rails 5.0)
prod.prices.reload
product.prices.reload
```

One production app fixed this in **3 separate commits** across different files — it kept being discovered in new locations:
- `app/models/shops/koorong.rb` (Jul 7, 2016)
- `app/models/shop.rb` (Aug 15, 2016)
- `booko_util/fetcher/fetcher.rb` (Aug 15, 2016)
- `app/models/shops/qbd.rb` (Aug 17, 2016)

**Lesson**: This pattern hides in scraping/fetching code where fresh data is needed after updates. It doesn't raise an error in 5.0 — it silently passes `true` as a scope argument, returning wrong results. Search broadly and test thoroughly. It will finally be **removed** in 5.1.

## ActiveModelSerializers 0.10 API Change

AMS 0.10 (released alongside Rails 5) renamed `ArraySerializer` to `CollectionSerializer`:

```ruby
# Before (AMS 0.8/0.9)
ActiveModel::Serializer::ArraySerializer.new(prices)

# After (AMS 0.10)
ActiveModel::Serializer::CollectionSerializer.new(prices)
```

One production app had this in the `Product#price_json` method, used three times (for NEW, USED, and NOT_FOUND price collections).

**Detection:**
```bash
grep -rn "ArraySerializer" app/
```

**Lesson**: If you use `active_model_serializers`, the 0.8→0.10 jump is significant — it's a complete rewrite. Pin to a compatible version during the upgrade and plan the AMS migration separately if possible.

## Configuration Changes

### config/application.rb
```ruby
# Removed (was transitional from 4.1→4.2)
config.active_record.raise_in_transactional_callbacks = true

# Added (re-enables autoloading in production — Rails 5 disabled it by default)
config.enable_dependency_loading = true

# Commented out (incompatible)
# config.middleware.use Rack::Attack
```

### config/environments/development.rb
```ruby
# Renamed
# Before:
config.serve_static_files = true
# After:
config.public_file_server.enabled = true

# Cache namespace changed (bust stale data)
# Before:
{ namespace: 'rails421' }
# After:
{ namespace: 'rails5' }
```

### config/environments/production.rb
```ruby
# Renamed
# Before:
config.serve_static_files = ENV['RAILS_SERVE_STATIC_FILES'].present?
# After:
config.public_file_server.enabled = false

# Log level tightened
# Before:
config.log_level = :debug
# After:
config.log_level = :info

# Cache namespace busted
{ namespace: 'rails5' }

# HTTPS protocol default added
Rails.application.routes.default_url_options[:protocol] = 'https'
```

### config/environments/test.rb
```ruby
# Renamed
# Before:
config.serve_static_files = true
config.static_cache_control = 'public, max-age=3600'
# After:
config.public_file_server.enabled = true
config.public_file_server.headers = { 'Cache-Control' => 'public, max-age=3600' }
```

## Gems Disabled During Upgrade

Several gems were incompatible with Rails 5 beta and needed to be commented out:

| Gem | Status | Notes |
|-----|--------|-------|
| **rack-attack** | Commented out | Middleware and initializer both disabled |
| **will_paginate** | Commented out | Incompatible with Rails 5 beta |
| **web-console** | Commented out | Needed update for Rails 5 |
| **better_errors** | Commented out | Incompatible |
| **binding_of_caller** | Commented out | Dependency of better_errors |
| **rails-erd** | Commented out | Incompatible |
| **meta_request** | Commented out | Incompatible |

**Gems added:**
- `rails-controller-testing` (extracted `assigns`/`assert_template`)

**Gems updated:**
- `active_model_serializers` 0.8.1 → 0.10.1

**Lesson**: Start the upgrade by commenting out non-essential gems to get Rails booting. Re-enable them one at a time after the core upgrade works. One production app's first commit was literally "Disable some stuff to get rails 5 running."

## Rails 5.0 Features: Adoption Decisions

| Feature | Adopted? | Notes |
|---------|----------|-------|
| **ApplicationRecord** | Partial | File created, but only 1 model migrated (rest deferred to 5.2) |
| **API mode** | Skipped | Existing API controllers kept as regular controllers |
| **ActionCable** | Skipped | Had existing Faye/Redis for WebSocket needs |
| **Turbolinks 5** | Skipped | No evidence of adoption |
| **rails-controller-testing** | Added | Needed to preserve existing `assigns` tests |
| **config.load_defaults** | Skipped | Not set (tech debt carried forward) |

**Lesson**: Rails 5.0 was the biggest "new features" release since Rails 3, but a conservative upgrade that only ensures compatibility is a valid approach. Adopt new features in separate PRs after the upgrade is stable.

## Subtle Gotchas

- **`enable_dependency_loading = true`**: Rails 5 disabled autoloading in production by default. If your app loads classes from `lib/` at runtime, you'll get `NameError` in production without this setting. The proper fix is to use `config.eager_load_paths` instead, but `enable_dependency_loading = true` is the quick workaround

- **`raise_in_transactional_callbacks` removal**: This was a transitional setting added in Rails 4.2 to opt into the new behavior (callbacks run inside the transaction). In Rails 5 the new behavior is the only behavior, so the setting must be removed or you'll get a deprecation warning → error

- **Association `(true)` silently breaks**: In Rails 5.0, `prices(true)` doesn't raise an error — it silently passes `true` as a scope argument, returning incorrect results. This makes it extremely hard to detect without thorough testing of data correctness

- **Catch-all routes served implicit endpoints**: If your app relied on the `":controller(/:action(/:id))"` catch-all pattern, you won't get routing errors during development — those routes simply won't exist. Audit traffic logs for 404s after deployment

- **AMS 0.10 is a full rewrite**: The jump from AMS 0.8/0.9 to 0.10 changes the entire serialization API, not just `ArraySerializer` → `CollectionSerializer`. If you use custom serializers extensively, consider pinning AMS to 0.9.x during the upgrade

- **`head` syntax change**: The old-style `head :status => :not_found` doesn't work in Rails 5. Use `head(:not_found)` or `head :not_found` instead

- **VCR cassettes**: VCR may need cassette re-recording after the upgrade due to changes in how Rails makes HTTP requests internally. One production app re-recorded cassettes in multiple commits

- **will_paginate incompatibility**: will_paginate was not compatible with early Rails 5 betas. If using will_paginate, check for a compatible version before upgrading, or consider switching to `kaminari` or `pagy`
