## Context

- `Admin::DetectMissingPaymentTransactionsWorker` creates one anomaly per missing payment and skips payments that already have one. It never re-checks existing anomalies.
- `Anomaly::MissingPaymentTransaction#attempt_to_report_sale` returns false when already resolved, runs `Integration::POS::ReportHouseAccountPaymentWorker.perform_inline`, and marks the anomaly resolved on success. On failure it logs and leaves the anomaly unresolved.
- Report failures are written to `critical_worker_audit_logs` with `status: "failed"`, `job_arguments` set to `[payment_id]`, and an `error_message` that contains the Retail HTTP code (for example `"httpCode"=>"401"`).
- Replica lag: check-then-act guards must read from the primary (see the `primary_db_workers` allowlist in `config/initializers/sidekiq.rb`). Do not rely on `Current.stick_to_primary`.

## Goals / Non-Goals

Goals: automatic recovery after a token is reauthorized; bounded blast radius per club; no double reports.

Non-goals: retrying non-401 failures, changing report payloads, changing the detection worker, fixing family account anomalies.

## Decisions

1. **Eligibility** is an unresolved, unignored anomaly for a `Sale::HouseAccountPayment` whose `paid_at` falls within `RETRY_WINDOW` (default 7 days) and whose latest failed audit entry for the report workers contains a 401.
2. **Skip guard** reads references from the primary database and skips the payment if a Retail reference already exists. This covers the replica-lag case.
3. **Circuit breaker** stops a club after `MAX_CONSECUTIVE_FAILURES` (default 3) failed attempts in one run. The run continues with other clubs.
4. **Reuse** `attempt_to_report_sale`, so the success path (resolving the anomaly) is the same one the Report button uses.
5. **Schedule** as a periodic worker in `app/workers/periodic/`, using the existing `config.periodic` pattern, on `SK_HA_BATCH`, with `retry: 0`, matching the detection worker.
6. **Logging** uses `Loggable` with distinct keys (no `message:` key in `details`).

## Risks

- A token that stays expired retries every run. The circuit breaker limits this to 3 calls per club per run, and the 401 filter stops non-auth failures.
- A payment reported at the POS but not marked synced could be re-sent. The skip guard on the primary database prevents this.
- Window default (7 days) is a judgment call. Confirm before merge.
