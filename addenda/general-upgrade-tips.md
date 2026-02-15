# General Upgrade Tips

## Symptom-Based Quick Lookup

| Symptom | Likely Cause | Check / Action |
|-----------|--------------|------------------|
| **My tests are failing after upgrade** | Job queue changes, transaction-aware jobs | `bin/rails test` |
| **"unknown keyword: pool" in database.yml** | Rails 8.0 → 8.1 pool rename | `grep "pool:" config/database.yml` |
| **Assets not loading** | Rails 8.0 Propshaft migration incomplete | `bin/rails assets:precompile` |
| **"undefined method secrets"** | Rails 7.2 secrets removal | `grep -i "secret" app/ lib/ config/` |
| **"pool unknown keyword"** | Rails 8.0 Solid Cache adoption | Solid gems `config/database.yml` |
| **"bundle-audit command not found"** | Rails 8.1 bundler-audit missing | `which bundler-audit` |
| **"API parameters parsed incorrectly"** | Rails 8.1 semicolon separator removed | API clients |

## Multi-Hop Strategy

### Phase Duration

**48 hours between hops** is recommended:
- Deploy to production
- Monitor for 48+ hours
- Ensure stability before next hop

### Risk Assessment

Based on test coverage:
- **< 70%**: Very High Risk - Add extra testing time
- **70-90%**: High Risk - Comprehensive testing required
- **> 90%**: Medium Risk - Testing still important but less critical

### Budget Formula

```
Estimated Hours = Base Estimate × 1.3 + 0.2

Example: 6 hours base estimate = 6 hours
  + 30% overhead (testing, validation) = 8 hours
  + 20% buffer (unknowns) = 2 hours
  = 16 hours total
```

## Detection Tips

### Deprecation Counting

```bash
# Count active deprecations ranked by frequency
bin/rails test 2>&1 | grep -i "deprecation" | sort | uniq -c | sort -rn
```

Top 5 warnings get most attention. Focus on those first.
