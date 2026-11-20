# Proposal

## Why

Monthly and Annual subscriptions store their billing schedule on each term ([GOLF-9896](https://lightspeedhq.atlassian.net/browse/GOLF-9896)) and have a calculator that places a term from that schedule ([GOLF-9894](https://lightspeedhq.atlassian.net/browse/GOLF-9894)). Renewal does not use either yet. Today `Subscription::Expire` starts the next term at the time the job runs, guesses its length from the previous term's duration, and takes the payment day from the previous term's first payment. For a declared schedule that drifts: a renewal run on January 2 would start the term January 2, and a customer who chose "charge the first payment today" would carry that day forward forever.

[GOLF-9898](https://lightspeedhq.atlassian.net/browse/GOLF-9898) makes renewal follow the stored schedule. It must ship before declared creation ([GOLF-9900](https://lightspeedhq.atlassian.net/browse/GOLF-9900)) is turned on, so no declared subscription ever renews through the legacy path.

## What Changes

- When the expiring term has a billing cycle, `Subscription::Expire` places the next term with `Subscription::Calculator.next_term` instead of the legacy duration and payment-day guesses. Its stop, eligibility, auto-renew, and already-renewed checks stay as they are.
- The next term copies the billing cycle, payment day, and payment month, and starts the day after the previous term ends in the club's timezone, so a late run still produces consecutive terms. Its agreement's `sold_at` is that scheduled start, not the run time.
- The next agreement gets one regular payment, dated by the stored payment day (and month), for the calculator's amount at current product prices. The previous agreement's payment dates and count are not read.
- `Sale::AgreementTemplate` accepts an explicit list of payments, so the declared path can supply the calculator's payment instead of the template's "copy the previous count" rule. Legacy calls pass nothing and behave as before.
- Terms with no billing cycle, including Custom terms created later, keep the existing renewal path unchanged.
- Declared renewal does not check the declared-creation flag, so turning creation off does not stop existing declared subscriptions from renewing.
- Tests cover the on-time and late runs, first-payment-today, current prices, repeated and concurrent attempts, flag independence, and an unchanged legacy renewal.
- Not in this change: skipping the proration quote line and payment on renewal ([GOLF-9906](https://lightspeedhq.atlassian.net/browse/GOLF-9906)), creation ([GOLF-9900](https://lightspeedhq.atlassian.net/browse/GOLF-9900)), the declared-creation flag, display and reports ([GOLF-9903](https://lightspeedhq.atlassian.net/browse/GOLF-9903)), renewal alerts and rollout ([GOLF-9909](https://lightspeedhq.atlassian.net/browse/GOLF-9909)), and moving payment collection out of the renewal transaction.

## Capabilities

### New Capabilities

- `subscription-declared-renewal`: How a term with a declared Monthly or Annual schedule renews: which schedule values carry over, how the next term's dates and payment are placed, and when a renewal must not create a successor.

### Modified Capabilities

None. `subscription-term-calculation` and `subscription-term-schedule` stay as written; this change is their first caller on the renewal side. `subscription-term-schedule` still says terms with no billing cycle are never calculated from a schedule, which this change preserves.

## Impact

- Implementation repository: `/Users/rushipatel/Documents/Repos/chronogolf`. Paths below refer to that checkout; planning artifacts live in `golf-specs`.
- `app/use_cases/subscription/expire.rb`: branch on the expiring term's billing cycle.
- `app/models/subscription.rb`: build the declared successor term next to the existing `renewal`.
- `app/models/sale/agreement_template.rb`: optional explicit payments.
- Specs: `spec/use_cases/subscription/expire_spec.rb`, `spec/models/subscription/base_spec.rb`, `spec/models/sale/agreement/template_spec.rb`.
- No migration, endpoint, serializer, frontend, feature flag, worker, or Sorbet RBI change. Nothing declared exists until GOLF-9900 is enabled, so merging changes no live renewal.

Sources: [GOLF-9898](https://lightspeedhq.atlassian.net/browse/GOLF-9898) and its linked tickets (9896, 9894, 9906, 9900); the archived `subscription-term-calculation` and `subscription-term-schedule` changes; the user-supplied `/Users/rushipatel/Documents/TDD_ Subscriptions major upgrade (Final).docx` (renewal and "How renewal works today" sections). The design records where this change departs from the TDD.
