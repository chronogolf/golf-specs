# Proposal

## Why

Monthly and Annual subscriptions need one backend calculation of calendar term dates, payment dates, and quote totals that future preview and renewal callers can share. GOLF-9894 establishes that read-only calculation without changing existing subscription workflows or the money engine.

## What Changes

- Add two read-only calculator methods: `first_term` works out the initial term and `next_term` works out the term after an existing one. Both return local dates, payment rows, an unsaved quote, and field/code validation errors. Quote totals remain on the quote rather than being duplicated on the result.
- Calculate Monthly calendar months and Annual twelve-month periods, including leap years and year boundaries.
- Validate the first eligible start month using the club's local date; default a missing initial selection to that month. Do not apply creation's start-month restriction to subsequent terms.
- Place the regular payment on the selected day from 1 through 28 and, for Annual, the selected month within the term. Moving the first payment to today does not change its amount or the recurring schedule.
- Return one regular payment for existing supported scheduled-payment strategies when the quote permits a positive payment, and no scheduled payments for External/Register. Calculation does not open Retail checkout; the existing Register sale and collection flow remains unchanged.
- Reuse current quote/tax behavior and renewal pricing. Preserve saved bundle prices through `Subscription::Line#price_from_source`, as explicitly confirmed by the user.
- Accept already resolved product-and-price lines instead of parsing HTTP payloads or looking up product IDs. Keep the existing quote's number types and rounding; do not introduce a monetary-type conversion.
- Preserve zero-priced products within a positive-total subscription. A term whose entire quote is $0 cannot provide a positive scheduled payment for Credit card, Bank account, or On account; do not add support for creating that subscription. Track the potential behavior change in [GOLF-10170 — Edge cases](https://lightspeedhq.atlassian.net/browse/GOLF-10170).
- Add focused calculator specs, including no-write/no-provider-call assertions. This proposal does not run tests or implement code.

## Capabilities

### New Capabilities

- `subscription-term-calculation`: Read-only calculation of declared calendar subscription terms, regular payments, quote totals, and validation errors.

### Modified Capabilities

None. The current OpenSpec capability inventory is empty; no existing durable spec is being modified.

## Impact

- Implementation repository: `/Users/rushipatel/Documents/Repos/chronogolf`. All `app/` and `spec/` paths below refer to that checkout; planning artifacts live in `golf-specs`.
- Planned application addition: `app/models/subscription/calculator.rb`. A small colocated result type may be defined within it; no new architecture layer is required.
- Planned tests: `spec/models/subscription/calculator_spec.rb`.
- Reused behavior: `Sale::Quote`, its subscription-product quote construction and tax calculation, and `Subscription::Line#price_from_source`.
- Coordinate recurring `billing_cycle`, `payment_day`, and `payment_month` values with [GOLF-9896](https://lightspeedhq.atlassian.net/browse/GOLF-9896). Pass them as `schedule:`. Pass the initial start-month selection and first-payment-today option separately; they are not new persisted fields. Only `first_term` needs the current timestamp, which defaults to `Time.current` and can be supplied in tests.
- Later consumers are [GOLF-9899](https://lightspeedhq.atlassian.net/browse/GOLF-9899) preview, [GOLF-9898](https://lightspeedhq.atlassian.net/browse/GOLF-9898) renewal, and [GOLF-9900](https://lightspeedhq.atlassian.net/browse/GOLF-9900) creation. This change does not wire any of them up.
- No migrations, persisted schedules, API endpoints or serializers, authorization changes, creation/renewal integration, frontend edits, feature flags, jobs, payment execution, or proration. Existing Custom and WooCommerce callers remain untouched.

Sources: [GOLF-9894](https://lightspeedhq.atlassian.net/browse/GOLF-9894), the user-supplied `/Users/rushipatel/Documents/TDD_ Subscriptions major upgrade (Final).docx`, and `/Users/rushipatel/Documents/subscriptions-prototype (Final).html`. The design records source discrepancies and the confirmed compatibility decisions.
