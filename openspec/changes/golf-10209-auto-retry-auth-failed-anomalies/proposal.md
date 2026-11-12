## Why

On Nov 1, 2026, about 03:55 EST, the batch that reports paid house-account payments to Lightspeed Retail failed for 7 clubs. Most failures (555 of 628 in the window) were HTTP 401 with "Token has expired". The payments were recorded as `Anomaly::MissingPaymentTransaction` rows, but nothing re-attempts an unresolved anomaly. The missing reports were only recovered by hand from superadmin and the console (GOLF-10174, GOLF-9936).

## What Changes

- Add a periodic worker that retries unresolved `Anomaly::MissingPaymentTransaction` rows whose most recent report failure was an authorization failure (HTTP 401).
- Retry only payments within a recent window. Older anomalies are left untouched unless someone opts in.
- Never retry a payment that already has a synced Retail reference (read from the primary database).
- Stop retrying a club after repeated consecutive failures and log the club instead of looping.
- Log per-club counts (retried, resolved, failed, skipped) with `Loggable`.

## Impact

- Affected code: `app/models/anomaly/missing_payment_transaction.rb` (a new scoped retry helper), a new periodic worker under `app/workers/periodic/`, and the periodic schedule in `config/initializers/sidekiq.rb`.
- No change to report behavior. The worker calls the existing `attempt_to_report_sale`.
- No change to non-authorization failures, family account anomalies, or the detection worker.
