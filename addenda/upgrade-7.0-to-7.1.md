# Upgrade 7.0 → 7.1: Lessons Learned

## Detection Patterns (with exclusions for false positives)

### .connection usage
```bash
# Exclude ActiveRecord .connection calls that are NOT the deprecated ones
grep -rn "\.connection[^_]" app/ lib/ | grep -v "with_connection\|lease_connection|@connection\s*=\s*Faraday|@connection\s*=\s*nil"
# Also exclude Faraday connections (external service)
# Also exclude nil resets
```

### force_ssl default change
```bash
grep -rn 'force_ssl' config/environments/
```

### cache_classes inversion
```bash
grep -rn 'cache_classes' config/environments/
# Gotcha: Boolean is INVERTED
# cache_classes = false means enable_reloading = true
# cache_classes = true means enable_reloading = false
```

## Edge Cases

- **SQLite database not found** after upgrade
  Symptom: `ActiveRecord::NoDatabaseError` in logs
  Diagnosis: SQLite files not moved from `db/` to `storage/`
  Fix: `mkdir -p storage && mv db/*.sqlite3 storage/`
  Also: Verify `.gitignore` includes `storage/` not just `db/*.sqlite3`

## Serialize API Change (Deprecation)

Rails 7.1 changed the `serialize` method signature. The old syntax triggers deprecation warnings:

```ruby
# Old (Rails 7.0) — triggers deprecation warning
serialize :data, Hash, coder: ActiveRecord::Coders::JSON

# New (Rails 7.1) — use keyword argument for type
serialize :data, type: Hash, coder: ActiveRecord::Coders::JSON
```

Detection:
```bash
# Find serialize calls that use the old positional argument for type
grep -rn "serialize\s*:" app/models/ | grep -v "type:"
```

## Cache Namespace Versioning

When upgrading Rails, update your cache namespace to avoid stale cache entries:
```ruby
# config/environments/production.rb
config.cache_store = :redis_cache_store, {
  namespace: "cache_#{Rails.version.gsub('.', '')}"  # e.g., "cache_711"
}
```

One production app used versioned namespaces (`70_`, `711_`, `712_`) to ensure clean cache invalidation after each upgrade.

## Hash Digest Algorithm Change: SHA1 → SHA256

Rails 7.1 defaults change the hash digest algorithm from SHA1 to SHA256. This affects multiple areas of your application and requires careful planning.

### What's Affected

**1. ActiveRecord Encryption** (CRITICAL if you use `encrypts`)
- Default hash digest for `ActiveRecord::Encryption` changes from SHA1 to SHA256
- **Breaking change**: `support_sha1_for_non_deterministic_encryption` now defaults to `false`
- **Impact**: Apps cannot decrypt existing encrypted data unless you:
  - Explicitly enable SHA1 support (temporary), OR
  - Re-encrypt all affected data with SHA256

**2. Cache Keys & ETags**
- Fragment cache keys will change (invalidates all cached fragments)
- ActiveRecord relation cache keys will change
- ETags will change (affects HTTP caching)
- Redis cache key shortening uses new digest

**3. Message Signing**
- ActiveSupport::MessageVerifier and MessageEncryptor use new digest
- Old signed/encrypted messages may not be readable

### Required Configuration

If you have encrypted data from Rails 7.0 or earlier:

```ruby
# config/application.rb or config/environments/production.rb

# Option 1: Enable SHA1 support temporarily (recommended for gradual migration)
config.active_record.encryption.support_sha1_for_non_deterministic_encryption = true
config.active_record.encryption.hash_digest_class = OpenSSL::Digest::SHA256

# Option 2: Keep using SHA1 explicitly (if you're not ready to migrate)
config.active_record.encryption.hash_digest_class = OpenSSL::Digest::SHA1
config.active_record.encryption.support_sha1_for_non_deterministic_encryption = true
```

### Error Symptoms

When encrypted data cannot be decrypted due to the digest mismatch, you'll see:

```
ActiveRecord::Encryption::Errors::Decryption
# or the underlying error:
OpenSSL::Cipher::CipherError
# typically in: ...encryptor.rb:58:in `rescue in decrypt'
```

These errors occur when accessing encrypted column getters on records encrypted with SHA1 when SHA1 support is disabled.

### Known Bug
There was a bug in Rails < 7.1 where some attributes were encrypted using SHA1 even when SHA256 was configured. If your app configured SHA256 before 7.1, you may still need SHA1 support enabled.

### Deployment Impact

**Cache Invalidation**
- All caches will be invalidated on first deployment with 7.1 defaults
- Expect increased database load as caches rebuild
- Consider temporarily scaling up servers/database before deployment

**Data Migration Strategy**
For apps with encrypted data:
1. Deploy with SHA1 support enabled (reads old + writes new)
2. Background job to re-encrypt all records:
   ```ruby
   # Re-encrypts all records with the new SHA256 digest
   MyModelWithEncryptedAttributes.find_each(&:encrypt)
   ```
3. After migration complete, disable SHA1 support
4. Deploy again with `support_sha1_for_non_deterministic_encryption = false`

Detection:
```bash
# Find models using ActiveRecord::Encryption
grep -rn "encrypts\s" app/models/
```

### Testing Before Production
```ruby
# Verify you can still decrypt existing data after upgrade
User.find(123).encrypted_field  # Should work without errors

# Check cache key format change
Rails.cache.fetch("test") { "value" }
# Keys will look different - SHA256 produces longer hashes
```

## Subtle Gotchas

- **`force_ssl` default changed**: Rails 7.1 changes SSL defaults for Kamal compatibility. If not using Kamal, verify your SSL config is still enforced
- **`cache_classes` → `enable_reloading`**: The boolean is **inverted** — `cache_classes = false` becomes `enable_reloading = true`
