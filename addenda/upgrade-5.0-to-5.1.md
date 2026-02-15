# Upgrade 5.0 → 5.1: Lessons Learned

## Detection Commands

### Association force-reload syntax
```bash
# Rails 5.1 removed association(true) for force reload
# Match patterns like: prices(true), products(true), etc.
grep -rn "\.\w\+(true)" app/models/ | grep -v "\.save\|\.valid\|\.destroy\|\.delete\|\.blank\|\.present\|\.nil\|\.empty\|\.freeze\|\.dup\|\.include"
```

### render text: / render nothing: usage
```bash
# render text: deprecated (use render plain:)
grep -rn "render.*:text\s*=>\|render.*text:" app/views/ app/controllers/
# render nothing: true removed (use head :ok or render status:)
grep -rn "render.*nothing:" app/controllers/
```

### Sanitize gem vs Loofah conflict
```bash
# Rails 5.1 bundles Loofah via rails-html-sanitizer — direct sanitize gem may conflict
grep -rn "Sanitize\." app/ lib/
grep -rn "sanitize" Gemfile
```

### jQuery dependency
```bash
# Rails 5.1 removes jQuery from default stack (but jquery-rails gem still works)
grep -rn "jquery" Gemfile
grep -rn '\\$(' app/assets/javascripts/ app/views/ | head -10
```

### erubis references (replaced by erubi)
```bash
# Rails 5.1 switched template engine from erubis to erubi
grep -rn "erubis" Gemfile config/
```

## The Upgrade Timeline (Real-World)

One production app completed the upgrade in approximately **39 days** of branch time (May 13 – Jun 21, 2017), but the actual work was minimal — the branch was mostly dormant between merges from master. The real work happened on **merge day** with 9 rapid-fire fix commits in 24 hours.

### Version Path
| Version | Date | Notes |
|---------|------|-------|
| 5.0.2 → 5.0.3 | 2017-05-13 | Patch update on master (parallel) |
| 5.0.2 → 5.1.1 | 2017-05-13 | `rails51` branch created |
| — | 2017-05-28 | Neat gem rolled back to ~1.8 on branch |
| — | 2017-06-21 | **MERGE: rails51 branch merged into master** |
| — | 2017-06-21/22 | 9 rapid-fire post-merge fixes |
| 5.1.1 → 5.1.3 | 2017-08-23 | Patch update with Passenger |

**Key strategy**: This was a "bump and fix" upgrade — change the Gemfile, merge, and fix what breaks in production. The branch had very few unique commits; it was primarily a holding pen for the version change while master continued receiving feature work.

**Deployment pattern**: Merge and same-day deploy with real-time bug fixing. The typo "Bug fixX" in one commit and "DElete old tests" in another suggest urgency during the deployment session.

## Association Force-Reload Syntax Change (Most Widespread)

Rails 5.1 removed the ability to pass `true` to association methods to force a reload. This was the **single most impactful breaking change**, affecting 9 shop model files.

```ruby
# Before (Rails 5.0)
prod.prices(true)

# After (Rails 5.1)
prod.prices.reload
```

**Detection:**
```bash
# This grep is intentionally broad — manually review results
grep -rn "\.prices(true)\|\.products(true)\|\.works(true)\|\.parts(true)" app/
# More generic (but noisier):
grep -rn "\.\w\+(true)" app/models/
```

**Affected files in one production app:**
- `app/models/shops/angus_and_robertson.rb`
- `app/models/shops/basement_books.rb`
- `app/models/shops/berkelouw_books.rb`
- `app/models/shops/bookware.rb`
- `app/models/shops/clouston_and_hall.rb`
- `app/models/shops/deep_discount.rb`
- `app/models/shops/koorong.rb`
- `app/models/shops/mcgraw_hill.rb`
- `app/models/shops/suomalainen.rb`

**Lesson**: This pattern is extremely common in price scraping/fetching code where you want fresh data after updating prices. Search broadly — it may appear in unexpected places.

## render :text and render nothing: Removal

### render text: → render plain:
```ruby
# Before (in JS.erb views)
$('div#flash').html("<%= escape_javascript(render(:text => @message)) %>");

# After
$('div#flash').html("<%= escape_javascript(render(plain: @message)) %>");
```

Four JS.erb view files needed updating in one production app. The old hash-rocket syntax (`:text =>`) made these harder to grep for — search for both syntaxes.

### render nothing: true → head/render status:
```ruby
# Before
render nothing: true, status: 200

# After (two options)
head :ok
# or
render status: 200
```

## Sanitize Gem → Loofah

Rails 5.1 bundles `loofah` through `rails-html-sanitizer`. If your app directly uses the `sanitize` gem, you may hit version conflicts. One production app switched from Sanitize to Loofah directly:

```ruby
# Before (using sanitize gem)
value = CGI.unescapeHTML(Sanitize.fragment(value, Sanitize::Config::RESTRICTED).strip)

# After (using Loofah, bundled with Rails)
value = CGI.unescapeHTML(Loofah.fragment(value).scrub!(:strip).to_s)
```

**Detection:**
```bash
grep -rn "Sanitize\.\|Sanitize::" app/ lib/
```

**Lesson**: If you only use Sanitize for basic HTML stripping, switch to Loofah and remove the direct dependency. If you use Sanitize's advanced configuration, pin to a compatible version.

## Rails 5.1 Configuration Changes

### Remarkably minimal for this upgrade

One production app made **zero changes** to:
- `config/application.rb`
- `config/environments/development.rb`
- `config/environments/test.rb`

The only production.rb change was expanding ActionCable allowed request origins to include new regional domains (not Rails-version-related).

**No `new_framework_defaults_5_1.rb`** file was created. No `config.load_defaults` was set (the app had been deferring this since Rails 5.0).

**Lesson**: The 5.0→5.1 upgrade was one of the smoothest Rails upgrades in terms of configuration. If your app is mostly config-debt-free, this upgrade is primarily about fixing deprecated method calls.

## erubis → erubi Template Engine

Rails 5.1 silently switched from `erubis` to `erubi` as the ERB template engine. This is handled automatically through the Gemfile.lock — no code changes required unless you:
- Reference `erubis` directly in your Gemfile
- Use erubis-specific template features
- Have custom ERB handlers

Most apps won't notice this change.

## jQuery Not Actually Removed

Despite the headline "jQuery removed from default stack," Rails 5.1 only removed jQuery from the **default generator output** for new apps. Existing apps that have `jquery-rails` in their Gemfile continue to work. One production app kept jQuery throughout the upgrade with a minor version bump (4.2.2 → 4.3.1).

**Don't panic about jQuery during this upgrade.** Only remove it if you're actively migrating away from jQuery-dependent code.

## Rails 5.1 Features: Adoption Decisions

| Feature | Adopted? | Notes |
|---------|----------|-------|
| **Yarn integration** | Skipped | Adopted 2.5 years later during Rails 6.0 |
| **Webpacker** | Skipped | Adopted 2.5 years later during Rails 6.0 |
| **Encrypted secrets** | Skipped | Never adopted (superseded by credentials in 5.2) |
| **form_with helper** | Skipped | Adopted 2+ years later during Rails 6.0 |
| **System tests (Capybara)** | Skipped | Capybara Selenium added during 5.2 era instead |
| **jQuery removal** | Skipped | jQuery kept in Gemfile |

**Lesson**: Rails 5.1's headline features were all optional and could be safely deferred. The upgrade is valuable purely for staying on a supported version and fixing deprecation warnings that become errors in later versions.

## Subtle Gotchas

- **`association(true)` in price scraping code**: This pattern is especially common in scraping/crawling code where you fetch prices and want to reload the association to see the new data. It won't raise an error — `association(true)` silently passes `true` as a scope argument, returning wrong results

- **Hash-rocket syntax in views**: `render(:text => @message)` is harder to grep for than `render(text: @message)`. Search for both old and new hash syntax when finding deprecated patterns

- **Arel version jump**: Arel 7→8 is a major version bump that may affect custom Arel usage. If you build Arel nodes directly, check compatibility

- **nio4r tightening**: ActionCable's `nio4r` dependency tightened from `>= 1.2, < 3.0` to `~> 2.0`. If you have other gems depending on nio4r 1.x, you'll hit a dependency conflict

- **Uninitialized variables surface**: Rails 5.1's stricter evaluation may surface uninitialized variable warnings that were previously silent. One production app had two rapid-fire fixes for uninitialized variables in shipping calculations
