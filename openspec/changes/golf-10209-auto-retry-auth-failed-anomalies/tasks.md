## 1. Eligibility

- [ ] 1.1 Add a scope to `Anomaly::MissingPaymentTransaction` for unresolved, unignored house-account anomalies within `RETRY_WINDOW`
- [ ] 1.2 Add a helper that returns true when the latest failed report audit for a payment has a 401
- [ ] 1.3 Add unit specs for the scope and the 401 helper, including non-401 and outside-window cases

## 2. Retry worker

- [ ] 2.1 Create `app/workers/periodic/retry_auth_failed_payment_anomalies_worker.rb` following the existing periodic worker pattern
- [ ] 2.2 Implement the skip guard, reading references from the primary database
- [ ] 2.3 Implement the per-club circuit breaker with `MAX_CONSECUTIVE_FAILURES`
- [ ] 2.4 Log per-club counts with `Loggable` (no `message:` key in details)
- [ ] 2.5 Add specs for: retry on 401, skip on existing reference, skip outside window, skip non-401, stop after consecutive failures

## 3. Schedule

- [ ] 3.1 Register the periodic job in `config/initializers/sidekiq.rb` (times in EST)
- [ ] 3.2 Confirm the queue and `retry: 0` match the detection worker

## 4. Verification

- [ ] 4.1 Run `bundle exec standardrb` on every changed file
- [ ] 4.2 Run the new specs and the existing anomaly specs
- [ ] 4.3 Manual check in staging: expire a test token, confirm no retries beyond the limit, restore it, confirm recovery on the next run
