# Upgrade 5.2 → 6.0: Lessons Learned

## Detection Commands

### responders gem usage
```bash
# Rails 6 removes respond_to/respond_with from controllers
grep -rn "respond_to :json\|respond_with" app/controllers/
grep -rn "responders" Gemfile
```

### dalli_store cache configuration
```bash
# Rails 6 deprecates :dalli_store in favor of :mem_cache_store
grep -rn "dalli_store" config/environments/
```

### belongs_to without optional: true
```bash
# Rails 6 with load_defaults 6.0 enforces belongs_to required by default
grep -rn "belongs_to :" app/models/ | grep -v "optional:"
```

### form_for usage (deprecated in favor of form_with)
```bash
grep -rn "form_for" app/views/
# Also check for JS that references form_for-generated IDs like #edit_model_123
grep -rn '#edit_\|#new_' app/views/ app/assets/javascripts/
```

### Class names incompatible with Zeitwerk conventions
```bash
# Files named foo_bar.rb must define FooBar, not FOOBar or FooBAR
# Common offenders: HTML -> Html, US -> Us, API -> Api
grep -rn "class [A-Z]\{2,\}" app/ lib/ | grep -v "# \|#$"
```

### Rack::Attack notification subscriber
```bash
# Notification API changed in Rails 6 — event name and payload structure
grep -rn "rack.attack\|rack_attack" config/initializers/
```

### ActionController::TestCase usage
```bash
# Deprecated in favor of ActionDispatch::IntegrationTest
grep -rn "ActionController::TestCase" test/
```

### config.hosts DNS rebinding protection
```bash
# Rails 6 blocks requests to unknown hosts in development
grep -rn "config.hosts" config/environments/development.rb
```

## Responders Gem Removal (CRITICAL)

Rails 6 removed `respond_to :format` and `respond_with` from controllers (extracted to the `responders` gem which then became incompatible). This was the single largest code change by file count.

**Two-step migration:**

1. Remove `respond_to :json` declarations from all controllers
2. Replace all `respond_with` calls with explicit `render json:`

```ruby
# Before
class Api::V1::ListsController < Api::V1::BaseController
  respond_to :json

  def index
    respond_with @lists, root: false
  end
end

# After
class Api::V1::ListsController < Api::V1::BaseController
  def index
    render json: @lists, root: false
  end
end
```

**Watch for edge cases:**
```ruby
# respond_with wraps non-hash values differently than render json:
# Before:
respond_with(product: product)
# After:
render json: { product: product }
```

## Cache Store: dalli_store → mem_cache_store

Rails 6 deprecated `:dalli_store` in favor of the built-in `:mem_cache_store` (which uses the `dalli` gem internally). This required changes across **all three environments**.

```ruby
# Before (all environments)
config.cache_store = :dalli_store, '127.0.0.1', { namespace: 'rails5', compress: true }

# After
config.cache_store = :mem_cache_store, '127.0.0.1', { namespace: 'rails6', compress: true }
```

**Critical**: Change the cache namespace (e.g., `rails5` → `rails6`, `r52` → `r6`) to avoid stale cache data. The serialization format may differ between the two stores, and old cache entries will cause silent failures.

## belongs_to Required by Default

`config.load_defaults 6.0` enables `belongs_to_required_by_default = true`.

**Detection:**
```bash
# Find all belongs_to without optional: true
grep -rn "belongs_to :" app/models/ | grep -v "optional: true"

# Test which ones fail
bundle exec rails test 2>&1 | grep "must exist"
```

**Models that commonly need `optional: true`:**
```ruby
# Polymorphic associations
belongs_to :collection, polymorphic: true, optional: true

# Nullable foreign keys
belongs_to :cover, optional: true
belongs_to :series, optional: true

# Associations set after creation
belongs_to :language, optional: true
```

**Tests break too** — factories must supply required associations:
```ruby
# Before (worked when belongs_to wasn't enforced):
list.list_prices << create(:list_price)

# After (list_price.list is now required):
list.list_prices << create(:list_price, list: list)
```

**Test setup may need explicit record creation:**
```ruby
def setup
  # Required because belongs_to enforcement is now active
  Region.create(id: 1, language: Language.first_or_create)
end
```

## form_for → form_with Migration

Rails 6 deprecated `form_for` in favor of `form_with`. The critical difference: **`form_with` does not generate HTML element IDs by default**.

If your JavaScript references form IDs like `#edit_product_123` or `#new_product`, these will silently break.

**Fix (recommended):**
```ruby
# config/application.rb — opt in to ID generation
config.action_view.form_with_generates_ids = true
```

**Manual ID assignment when needed:**
```erb
<!-- Before -->
<%= form_for @product do |f| %>

<!-- After — note custom ID to avoid conflicts -->
<%= form_with(model: @product, id: "editer_#{@product.id}") do |f| %>
```

**JavaScript references must be updated:**
```erb
<!-- Before (form_for generated #edit_product_123) -->
onclick='$("#edit_product_<%= @product.id %>").submit()'

<!-- After (using custom ID) -->
onclick='$("#editer_<%= @product.id %>").submit()'
```

**Lesson**: Set `form_with_generates_ids = true` first, then migrate forms incrementally. Don't change forms and IDs simultaneously.

## Rack::Attack Notification Breaking Change

The Rack::Attack notification API changed in **two ways simultaneously**, requiring two rapid-fire fixes:

**Change 1: Payload structure** — the request is no longer passed as the last block argument; it's in the `payload` hash:
```ruby
# Before (broken in Rails 6)
ActiveSupport::Notifications.subscribe('rack.attack') do |name, start, finish, request_id, req|
  Rails.logger.info("Blocked: #{req.env['REMOTE_ADDR']}")
end

# After
ActiveSupport::Notifications.subscribe(/rack_attack/) do |name, start, finish, request_id, payload|
  req = payload[:request]
  Rails.logger.info("Blocked: #{req.env['REMOTE_ADDR']}")
end
```

**Change 2: Notification name** — changed from string `'rack.attack'` to a different format, requiring a regex subscriber (`/rack_attack/`).

## Class Naming for Zeitwerk Compatibility

Rails 6 introduced the Zeitwerk autoloader. Even without explicitly enabling it (`config.autoloader = :zeitwerk`), the stricter naming conventions started being enforced. File names must match class names using Rails' `String#camelize` rules.

**Common pattern**: Acronyms in class names must be titlecased, not all-caps:

```ruby
# lib/html_page.rb
# Before (worked with classic autoloader)
class HTMLPage  # File html_page.rb expects HtmlPage

# After
class HtmlPage
```

```ruby
# app/models/shops/amazon_us.rb
# Before
AmazonUS::cart_links(...)

# After
AmazonUs::cart_links(...)
```

**Detection:**
```bash
# Find class names with consecutive uppercase letters
grep -rn "class [A-Z][A-Z]" app/ lib/ | grep -v "# "

# Common offenders:
# HTMLPage -> HtmlPage
# AmazonUS -> AmazonUs
# XMLParser -> XmlParser
# HTTPClient -> HttpClient
# APIController -> ApiController (usually already correct)
```

**Lesson**: These failures can happen well after the initial upgrade — a class that's autoloaded via a different path may work initially, then break when loaded directly by Zeitwerk.

## CSRF and CSP Meta Tags

Rails 6 added Content Security Policy support. Update your layout:

```erb
<!-- Before -->
<%= csrf_meta_tags unless response.cache_control[:public] %>

<!-- After -->
<%= csrf_meta_tags %>
<%= csp_meta_tag %>
```

**API controllers**: If you have API endpoints that accept form submissions, you may need to skip CSRF verification:
```ruby
skip_before_action :verify_authenticity_token, only: [:update, :create]
```

## Test Migration: ActionController → ActionDispatch

Rails 6 deprecates `ActionController::TestCase` in favor of `ActionDispatch::IntegrationTest`. The key differences:

```ruby
# Before
class Api::V1::ProductsControllerTest < ActionController::TestCase
  test "show product" do
    get(:show, params: { id: prod.gtin, api_user: user.id, format: :json })
  end
end

# After
class Api::V1::ProductsControllerTest < ActionDispatch::IntegrationTest
  test "show product" do
    get api_v1_product_path(product), headers: { 'Authorization' => "Bearer #{token}" }
  end
end
```

**Key changes:**
- Use URL helpers instead of action symbols
- Pass auth via HTTP headers instead of params
- `response.body` replaces `@response.body`
- No more `assigns()` or `assert_template()` without the `rails-controller-testing` gem

## config.hosts (DNS Rebinding Protection)

Rails 6 added host authorization that blocks requests from unrecognized hostnames in development:

```ruby
# config/environments/development.rb
config.hosts << "my-app.local"
config.hosts << "chips-n-bits.local"
```

**Symptom**: `Blocked host: my-app.local` error in development. Does not affect production (where `config.hosts` is typically empty/permissive).

## Rails 6.0 Configuration Changes

### config/application.rb
```ruby
config.load_defaults 6.0

# Opt out of cookie metadata (if needed for backward compatibility)
# config.action_dispatch.use_cookies_with_metadata = false

# Maintain form_for-style ID generation with form_with
config.action_view.form_with_generates_ids = true
```

### config/environments/development.rb
```ruby
# Changed from true to false (Rails 6 default)
config.eager_load = false

# New: DNS rebinding protection
config.hosts << "my-app.local"
```

### config/environments/test.rb
```ruby
# Removed (deprecated in Rails 6.0 final)
# config.action_view.finalize_compiled_template_methods = false
```

### Cache store (all environments)
```ruby
# dalli_store → mem_cache_store (Rails 6 uses dalli internally)
config.cache_store = :mem_cache_store, '127.0.0.1', { namespace: 'r6', compress: true }
```

## ApplicationController Refactoring

Rails 6's stricter autoloading and initialization order may break monkeypatched controller methods. One production app had to extract `current_user`, `current_region`, and `current_history` from a monkeypatched `ActionController::Base.class_exec` block in an initializer into a proper module:

```ruby
# Before (config/initializers/current_values.rb) — fragile with Zeitwerk
ActionController::Base.class_exec do
  def current_user
    # ...
  end
end

# After (app/helpers/current_helper.rb) — proper module
module CurrentHelper
  def current_user
    # ...
  end
end

# app/controllers/application_controller.rb
class ApplicationController < ActionController::Base
  include CurrentHelper
end
```

**Lesson**: Initializer-based monkeypatching of Rails classes is fragile and often breaks during upgrades. Extract to modules included in the proper place.

## Subtle Gotchas

- **`finalize_compiled_template_methods` whiplash**: This config was added in Rails 6 betas (`config.action_view.finalize_compiled_template_methods = false`) then **deprecated in the final release**. If you followed beta upgrade guides, you'll need to remove it

- **Cache namespace invalidation**: Changing from `:dalli_store` to `:mem_cache_store` isn't enough — change the namespace too (e.g., `r52` → `r6`) to avoid deserializing incompatible cached objects

- **`Time.now` → `Time.current`**: Rails 6 is stricter about timezone-aware time methods in tests. Replace `Time.now` with `Time.current` and `Time.now + 1.year` with `Time.current + 1.year`

- **Test sign_in helpers may need REMOTE_ADDR**: If your authentication checks IP addresses (for rate limiting, logging, or Rack::Attack), test helpers need to include the header: `headers: { 'REMOTE_ADDR' => '1.2.3.4' }`

- **Enum naming conflicts**: Rails 6 is stricter about enum method name collisions. If you have enums like `condition: { new: 0, used: 1 }`, the `new` value conflicts with `Model.new`. Use prefixed names: `condition: { cond_new: 0, cond_used: 1 }`
