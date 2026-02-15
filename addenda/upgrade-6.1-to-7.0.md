# Upgrade 6.1 → 7.0: Lessons Learned

This is one of the most complex Rails upgrades due to simultaneous changes in the JavaScript pipeline (Webpacker → Importmap), frontend framework (Turbolinks → Turbo/Stimulus), and Ruby 3.1 breaking changes (Psych 4.0, stdlib gem removal).

## Detection Commands

### Turbolinks references (CRITICAL - most pervasive change)
```bash
# Find turbolinks event listeners in JS files
grep -rn "turbolinks:" app/ lib/ vendor/
# Find turbolinks data attributes in views
grep -rn "data-turbolinks" app/views/ app/javascript/
# Find turbolinks gem references
grep -rn "turbolinks" Gemfile app/assets/javascripts/
# Find turbolinks in JavaScript pack entry points
grep -rn "turbolinks" app/javascript/packs/
```

### jQuery usage
```bash
# Find jQuery patterns in JavaScript
grep -rn "\$\.\|jQuery\.\|\$(" app/assets/javascripts/ app/javascript/
# Find jQuery in views (inline scripts)
grep -rn "\$\.\|jQuery\.\|\$(" app/views/
```

### Webpacker detection
```bash
# Check for Webpacker config
ls config/webpacker.yml babel.config.js postcss.config.js package.json yarn.lock 2>/dev/null
# Check for webpack helpers in views
grep -rn "javascript_pack_tag\|stylesheet_pack_tag" app/views/
```

### YAML.load without permitted_classes (Ruby 3.1 / Psych 4.0)
```bash
# Find unsafe YAML.load calls
grep -rn "YAML\.load\b" app/ lib/ | grep -v "permitted_classes\|safe_load\|YAML\.load_file"
```

### Ruby stdlib gems that need explicit inclusion
```bash
# Check if mail gem is used (needs net-smtp, net-pop, net-imap)
grep -rn "ActionMailer\|deliver_later\|deliver_now" app/
# Check for direct usage
grep -rn "Net::SMTP\|Net::POP\|Net::IMAP" app/ lib/
```

### Kernel#open usage (security)
```bash
# Brakeman flags Kernel#open with URL strings
grep -rn "[^I]open(" app/ lib/ | grep -v "File.open\|URI.open\|IO.open"
```

### SQL injection via string interpolation
```bash
# Find potential SQL injection in where clauses
grep -rn '\.where(".*#{' app/models/ app/controllers/ app/services/
```

### Unsafe redirects
```bash
# Find redirect_to with model URLs (may need URI.parse wrapping)
grep -rn "redirect_to.*\.url" app/controllers/
```

### Cookie serializer
```bash
# Check current cookie serializer setting
grep -rn "cookies_serializer" config/initializers/
# If missing, it defaults to :marshal — needs migration path
```

### form_with / form_for defaults
```bash
# Rails 7 changed form_with default from remote: true to local: true
grep -rn "form_for\|form_tag" app/views/
# Find explicit remote: true (may be unnecessary with Turbo)
grep -rn "remote: true" app/views/
```

### Faraday basic_auth deprecation
```bash
# Faraday 2.0 removed the instance method
grep -rn "\.basic_auth(" app/ lib/
```

### Redis.current removal
```bash
# Redis.current was deprecated in redis-rb 4.6, removed in 5.0
grep -rn "Redis\.current" app/ lib/ config/
```

### constantize on user input (security)
```bash
# Brakeman flags this as remote code execution risk
grep -rn "\.constantize" app/controllers/
```

## The Upgrade Timeline (Real-World)

One production app took **4+ months of active work** (Dec 2021 - Apr 2022), with preparation starting in Aug 2021. Key phases:

1. **Pre-upgrade preparation** (Aug-Sep 2021): Cookie serializer migration, Brakeman security fixes, updated to latest Rails 6.1 patch
2. **First attempt** (Dec 2021): "Another shot at upgrading to rails 7 with import-maps" — the commit message explicitly reveals prior failed attempts
3. **Core upgrade** (Jan 2022): Bumped to Rails 7.0.1, added framework defaults, importmap setup
4. **Frontend migration** (Jan-Mar 2022): Stimulus controllers, jQuery removal, Turbo Stream integration
5. **Production stabilization** (Mar 2022): Analytics fixes, ActionCable endpoint changes, Ruby 3.1 compatibility
6. **Parallel maintenance**: They kept Rails 6.1 updated on a separate branch (`rails7nohb`) as a production fallback throughout

**Lesson**: This is not a weekend upgrade. Plan for months of parallel branch development with a production fallback on 6.1.

## Turbolinks → Turbo Migration

### Event Name Replacement (MOST PERVASIVE)
Every `turbolinks:load` event listener must become `turbo:load`. This affects JavaScript files, inline scripts, analytics integrations, and view templates.

```javascript
// Before
document.addEventListener('turbolinks:load', function(event) { ... })

// After
document.addEventListener('turbo:load', function(event) { ... })
```

**Gotcha**: The event object structure is different:
```javascript
// Turbolinks event
page_location: event.data.url

// Turbo event (different property path)
page_location: event.detail.url
// or use window.location directly
page_location: window.location.href
```

### Data Attributes
```html
<!-- Before -->
<link data-turbolinks-track="true" href="/assets/application.css" rel="stylesheet" />
<%= javascript_include_tag "application", 'data-turbolinks-track': 'reload' %>

<!-- After -->
<link href="/assets/application.css" rel="stylesheet" />
<%= javascript_include_tag "application", 'data-turbo-track': 'reload' %>
```

### Analytics Integration (GA / Matomo)
Analytics break silently with Turbo because page navigations no longer trigger full page loads. Both Google Analytics and Matomo need custom `turbo:load` listeners.

**Google Analytics:**
```javascript
document.addEventListener('turbo:load', function(event) {
  if (typeof gtag === 'function') {
    gtag('config', 'GA-XXXXX', {
      'page_location': window.location.href,
      'page_path': window.location.pathname,
      'page_title': document.title
    });
  }
});
```

**Matomo:**
```javascript
(function() {
  var previousPageUrl = null;
  addEventListener('turbo:load', function(event) {
    if (previousPageUrl) {
      _paq.push(['setReferrerUrl', previousPageUrl]);
      _paq.push(['setCustomUrl', window.location.href]);
      _paq.push(['setDocumentTitle', document.title]);
      _paq.push(['trackPageView']);
    }
    previousPageUrl = window.location.href;
  });
})();
```

Pattern: Skip tracking on the first load (let the default tracker handle it), then track subsequent Turbo navigations manually.

**Gotcha**: If using Matomo, the endpoint URLs also changed from `piwik.php`/`piwik.js` to `matomo.php`/`matomo.js` around this time.

## Turbo Frame Gotchas

### Links Inside Turbo Frames Break
Links inside a `<turbo-frame>` try to navigate WITHIN the frame by default. External links (like outbound retailer links) break because they try to load the external page inside the frame.

```erb
<!-- Before (broken — link tries to load inside the turbo frame) -->
<a href="<%= price.link %>" rel="nofollow" class="...">

<!-- After (works — breaks out of the frame) -->
<a href="<%= price.link %>" rel="nofollow" class="..." data-turbo-frame="_top">
```

Detection:
```bash
# Find turbo_frame_tag usage to audit links inside them
grep -rn "turbo_frame_tag" app/views/
```

### TurboStreams Are Unreliable in Production
Build Stimulus fallbacks that detect when TurboStreams are not functioning and fall back to HTTP polling:

```javascript
// Stimulus controller pattern: embed timestamp in data attribute,
// if TurboStream doesn't update within timeout, force HTTP fetch
tableTargetConnected(element) {
  const state = parseInt(element.dataset['state'], 10)
  if (state == PROCESSING) {
    let ts = Date.parse(element.dataset['ts'])
    let tfe = this.element.getElementsByTagName('turbo-frame')[0]
    setTimeout(() => {
      if (Date.now() - ts >= 2000) {
        tfe.src = this.urlValue  // Force turbo-frame to reload via HTTP
      }
    }, 2000);
  }
}
```

**Lesson**: Never rely solely on TurboStreams for critical UI updates. Always have an HTTP fallback path.

## Webpacker → Importmap Migration

### What Gets Removed
```bash
# All of these are deleted:
yarn.lock          # ~8000 lines
package.json
babel.config.js
postcss.config.js
bin/webpack
bin/webpack-dev-server
config/webpacker.yml
app/javascript/packs/application.js
```

### What Gets Added
```bash
# New files:
config/importmap.rb
app/javascript/application.js    # New entry point (not in packs/)
bin/importmap
```

### Importmap CDN Pin Issues
npm packages need to be pinned to CDN URLs. Packages designed for Node.js require browser shims:

```ruby
# config/importmap.rb
pin "application", preload: true
pin "@hotwired/turbo-rails", to: "turbo.min.js", preload: true
pin "@hotwired/stimulus", to: "stimulus.min.js", preload: true
pin "@hotwired/stimulus-loading", to: "stimulus-loading.js", preload: true
pin_all_from "app/javascript/controllers", under: "controllers"

# CDN-pinned packages
pin "jquery", to: "https://ga.jspm.io/npm:jquery@3.6.0/dist/jquery.js"
pin "@rails/actioncable", to: "actioncable.esm.js"

# Node.js packages that need browser shims
pin "handlebars", to: "https://ga.jspm.io/npm:handlebars@4.7.7/lib/index.js"
pin "fs", to: "https://ga.jspm.io/npm:@jspm/core@2.0.0-beta.19/nodelibs/browser/fs.js"
pin "os", to: "https://ga.jspm.io/npm:@jspm/core@2.0.0-beta.19/nodelibs/browser/os.js"
```

**Gotcha**: npm packages designed for Node.js (like Handlebars) need browser shims for `fs`, `os`, `source-map`, etc. when loaded via importmaps.

**Gotcha**: Local libraries (like Ahoy analytics) may be incorrectly pinned to npm CDN URLs instead of local assets:
```ruby
# Wrong — points to wrong npm package
pin "ahoy", to: "https://ga.jspm.io/npm:ahoy@1.0.1/lib/index.js"

# Correct — points to local asset
pin "ahoy", to: "ahoy.js"
```

### Stimulus Controller Registration
Stimulus auto-loading did not work reliably with importmaps in early Rails 7. Explicit registration was needed:

```javascript
// Auto-loading (unreliable with importmaps in Rails 7.0.x)
import { eagerLoadControllersFrom } from "@hotwired/stimulus-loading"
eagerLoadControllersFrom("controllers", application)

// Explicit registration (reliable)
import ClipboardController from "./clipboard_controller.js"
application.register("clipboard", ClipboardController)
// ... every controller manually listed
```

### Stimulus Import Path Change
Every Stimulus controller must update its import:
```javascript
// Before
import { Controller } from "stimulus"

// After
import { Controller } from "@hotwired/stimulus"
```

## jQuery → Stimulus Migration

### ActionCable Channel Patterns
```javascript
// Before (jQuery)
$('#productPrices').data('prices')[condition]['prices'] = result;
result = $.grep(price_data, function(e) { return e.shop.id !== shop_id; });

// After (vanilla JS)
let prices_json = document.getElementById('productPrices').dataset['prices']
let prices_data = JSON.parse(prices_json)
prices_data[condition]['prices'] = price_data.filter(ele => ele.shop.id !== shop_id)
document.getElementById('productPrices').dataset['prices'] = JSON.stringify(prices_data)
```

### Personalisation via Meta Tags (Caching Pattern)
User-specific content was moved from inline jQuery to Stimulus controllers that read from `<meta>` tags. This pattern enables full-page caching while still personalising content:

```html
<meta name="current-user-name" content="<%= current_user&.name %>">
<meta name="current-user-e" content="<%= current_user&.is_editor? %>">
<body data-controller="personalise">
```

The Stimulus controller uses `document.querySelector("meta[name='current-user-name']")` to personalise the page after load.

### Event Tracking Migration
```javascript
// Before: inline jQuery callbacks
$('.product-link').on('click', function() { ahoy.track(...) })

// After: Stimulus controller with data-action
// app/javascript/controllers/ahoy_controller.js
import { Controller } from "@hotwired/stimulus"
export default class extends Controller {
  track_click(event) {
    let payload = event.target.closest('a').dataset
    delete payload.action  // Remove Stimulus action attribute from tracking
    ahoy.track('Clicked Product', payload);
  }
}
```
```erb
<a href="..." data-action="click->ahoy#track_click" ...>
```

## ActionCable / WebSocket Changes

### Dedicated WSS Server → Puma-Integrated
Rails 7 encourages running ActionCable within the Puma process instead of a separate server (like Faye):

```ruby
# Before (Rails 6.1 — separate Faye WebSocket server)
# faye.ru file with dedicated WS process
config.action_cable.url = "wss://example.com/cable"

# After (Rails 7 — integrated into Puma)
# faye.ru deleted
config.action_cable.url = "ws://rails01/cable"
```

The `faye.ru` (37 lines) was completely removed.

### ActionCable Channel Import Path
```javascript
// Before
import consumer from "channels/consumer"

// After (importmap)
import consumer from "channels/consumer"
// But the consumer itself now imports from:
import { createConsumer } from "@rails/actioncable"
```

## Ruby 3.1 Breaking Changes (Often Done Simultaneously)

### Psych 4.0 / YAML.load Safety
Ruby 3.1 changed `YAML.load` to use safe-loading by default. Code deserializing Ruby objects from YAML breaks with `Psych::DisallowedClass`:

```ruby
# Before (unsafe — now raises Psych::DisallowedClass)
YAML.load(redis_data)

# After — explicitly permit classes
YAML.load(redis_data, permitted_classes: [GlobalID, URI::GID])

# Or use YAML.unsafe_load if you trust the source (not recommended)
```

**Detection**: Any code storing Ruby objects (GlobalID, Time, Date, custom classes) in YAML — especially in Redis, cache stores, or serialized columns.

### Stdlib Gems Removal
Ruby 3.1 removed several gems from the default bundle. The `mail` gem (used by ActionMailer) depends on all three:

```ruby
# Gemfile additions required
gem 'net-smtp', require: false
gem 'net-pop',  require: false
gem 'net-imap', require: false
```

Without these, `bundle exec` may work but email sending fails at runtime with `LoadError`.

## Security Fixes (Commonly Surfaced by Brakeman)

### Kernel#open → URI.open
```ruby
# Before (Brakeman: remote code execution risk)
data = open(url)

# After
data = URI.open(url)
```

In Ruby 3+, `Kernel#open` with a URL string starting with `|` can execute arbitrary commands.

### String Interpolation in SQL
```ruby
# Before (SQL injection)
where("title like '%#{set_no}%'")
where("gtin::bigint > #{gtin}")

# After (parameterized)
where("title like ?", "%#{set_no}%")
where("gtin::bigint > ?", gtin)
```

### Unsafe Redirects
```ruby
# Before (open redirect)
redirect_to @price.url

# After (validates URL structure)
redirect_to URI.parse(@price.url).to_s
```

**Gotcha**: Don't apply `URI.parse` blindly — if the method returns an ActiveRecord object (not a URL string), `URI.parse` will raise an error. One production app had to roll back this fix.

### constantize on User Input
```ruby
# Before (remote code execution risk)
kind = params[:taggable_type]
object = kind.constantize.find(params[:taggable_id])

# After (safe GlobalID resolution)
object = GlobalID::Locator.locate(params[:taggable])
```

Views pass `model.to_global_id` instead of separate type/id hidden fields.

## Cookie Serializer Migration

The recommended migration path: `:marshal` → `:hybrid` → `:json`

```ruby
# config/initializers/cookies_serializer.rb

# Step 1: Switch to hybrid (reads both Marshal and JSON, writes JSON)
Rails.application.config.action_dispatch.cookies_serializer = :hybrid

# Step 2: After all old cookies have expired/been rewritten (~30 days)
Rails.application.config.action_dispatch.cookies_serializer = :json
```

**Gotcha**: One production app switched directly to `:json` and had to switch back to `:hybrid` two days later because existing user cookies were Marshal-serialized. The `:hybrid` serializer handles the transition gracefully.

## Faraday 2.0 Breaking Changes

Faraday's authentication API had a multi-step deprecation that is easy to get wrong:

```ruby
# Old API (Faraday 1.x — removed in 2.0)
conn.basic_auth(user, pass)

# Intermediate API (deprecated in 2.0)
conn.request(:basic_auth, user, pass)

# Correct API (Faraday 2.x)
conn.request(:authorization, :basic, user, pass)
```

**Real-world**: One production app took three consecutive commits to get this right, with multiple commented-out attempts visible in the code.

## Rack::Attack API Change

```ruby
# Before (Rack::Attack 5.x)
Rack::Attack.throttled_response = lambda { |env| ... }

# After (Rack::Attack 6.x)
Rack::Attack.throttled_responder = lambda { |request| ... }
```

Note the method name change AND the block parameter change (from `env` to `request`).

## Kredis Adoption (Optional but Common)

Rails 7 introduced Kredis for typed Redis data structures. Migration from manual Redis/cache calls:

```ruby
# Before: Manual cache-based state management
def price_lookup_state=(value)
  Rails.cache.write("p_#{id}_#{region_id}", value, expires_in: 5.minutes)
end
def price_lookup_state
  Rails.cache.read("p_#{id}_#{region_id}") || READY
end

# After: Kredis enum
kredis_enum :price_lookup_state, values: %w[ready queued processing], default: "ready"
```

Usage changes from constants to enum methods:
```ruby
# Before
product.price_lookup_state == READY
product.price_lookup_state = PROCESSING

# After
product.price_lookup_state.ready?
product.price_lookup_state.value = 'processing'
```

**Gotcha**: Kredis `unique_list` behaves differently from manual array manipulation. One production app tried to migrate a History model to Kredis and had to roll back because the API didn't match the existing behavior.

## Asset Pipeline Changes

### Sprockets Compressor Removal
```ruby
# Removed (Rails 6.1)
Sprockets.register_compressor 'application/javascript', :terser, Terser::Compressor
config.assets.js_compressor = :terser
config.assets.css_compressor = :sass

# Rails 7 — no JS compressor needed (ES modules don't need minification via Sprockets)
```

### Asset Path Helper Required
Hardcoded asset paths stop working with digest assets:
```erb
<!-- Before (broken with digest assets) -->
<link rel="icon" href="/assets/app/favicon-32x32.png">

<!-- After -->
<link rel="icon" href="<%= asset_path "app/favicon-32x32.png" %>">
```

### Manifest File
```javascript
// app/assets/config/manifest.js (new in Rails 7)
//= link_tree ../images
//= link_directory ../javascripts .js
//= link_directory ../stylesheets .css
```

## Configuration Changes

### config/environments/test.rb
```ruby
# New in Rails 7
config.eager_load = ENV["CI"].present?  # Was: false
config.cache_store = :null_store        # Was: Memcache/Redis
config.active_support.disallowed_deprecation = :raise
config.active_support.disallowed_deprecation_warnings = []
```

### config/environments/development.rb
```ruby
# New: Explicit hosts configuration
config.hosts << "myapp.localhost" << "api.myapp.localhost"
# ActionCable forgery protection (commented out by default)
# config.action_cable.disable_request_forgery_protection = true
```

### config/environments/production.rb
```ruby
# Removed: Sprockets compressor config
# Removed: Terser/Sass references
# Added: config.assume_ssl (for reverse proxy setups)
```

### New Config Files
- `config/importmap.rb` — JavaScript module pinning
- `config/initializers/content_security_policy.rb`
- `config/initializers/permissions_policy.rb`
- `config/initializers/filter_parameter_logging.rb`
- `config/initializers/new_framework_defaults_7_0.rb`
- `config/puma.rb` — May need updating for ActionCable
- `Procfile.dev` — For `bin/dev` to start multiple processes

### New Bin Files
- `bin/dev` — Starts Rails + CSS watcher + other processes
- `bin/importmap` — Manage importmap pins
- `bin/setup` — Updated setup script

## Subtle Gotchas

- **String-based event name search is insufficient**: `grep -r "turbolinks"` won't catch all event names if they've been partially updated. Search for both `turbolinks` AND `turbo` to find half-migrated references

- **Matomo event name was missed on first pass**: The Turbolinks→Turbo event name change was caught in most files but missed in the Matomo tracking script. A dedicated grep for each analytics provider is essential

- **`submit_tag` doesn't work well with Turbo**: Turbo expects `f.button` or `button_tag` with `form_with`. The `submit_tag` helper can cause double-submission issues with Turbo's form interception

- **Font paths break with Sprockets changes**: When Sprockets configuration changes, fonts may need to be moved to `app/assets/fonts/` and explicitly linked in the manifest. One app had ~15 commits just fixing font and favicon paths

- **Redis.current removal**: Redis 4.6+ deprecated `Redis.current`. Replace with connection pools:
  ```ruby
  # Before
  Redis.current.get(key)

  # After
  $redis.with { |conn| conn.get(key) }
  ```

- **ActionCable requires explicit allowed_request_origins**: In production, ActionCable now requires `config.action_cable.allowed_request_origins` to include all domains that will connect via WebSockets

- **CSS compression must be disabled carefully**: If you remove Terser/Sass compressors, ensure your CSS pipeline doesn't rely on SCSS compilation. One app had to set `config.assets.css_compressor = nil` explicitly to avoid errors

- **Multiple simultaneous upgrades compound complexity**: Upgrading Rails 6.1→7.0 while also upgrading Ruby 2.7→3.1, Faraday 1.x→2.x, and Psych 3.x→4.x simultaneously creates a combinatorial explosion of breaking changes. Consider staging these upgrades

- **Kredis rollback risk**: Kredis enums and unique_lists are not drop-in replacements for hand-rolled Redis patterns. Test thoroughly before deploying, and keep the old code paths available for quick revert

- **The Dalli gem (Memcached) may be removable**: If migrating caching to Redis during the upgrade, remember to also remove the `dalli` gem from the Gemfile. One app forgot this, leaving a dead dependency
