# Proposal

## Why

The Monthly and Annual upgrade needs each subscription term to remember its declared billing schedule, so renewal, creation, and display can use what was chosen rather than inferring it from dates. GOLF-9894 already calculates terms from a `schedule:` of `billing_cycle`, `payment_day`, and `payment_month`, but nothing stores those values. [GOLF-9896](https://lightspeedhq.atlassian.net/browse/GOLF-9896) adds the storage and its rules, and blocks [GOLF-9898](https://lightspeedhq.atlassian.net/browse/GOLF-9898) (renewal), [GOLF-9900](https://lightspeedhq.atlassian.net/browse/GOLF-9900) (creation), and [GOLF-9903](https://lightspeedhq.atlassian.net/browse/GOLF-9903) (Manage, list, reports, CSV).

## What Changes

- Add three nullable columns to `subscriptions`, the table whose rows are terms: `billing_cycle`, `payment_day`, and `payment_month`. The migration is additive: no default, no backfill, no contract table.
- `billing_cycle` accepts `monthly`, `annual`, or NULL. NULL means the term follows the existing creation, renewal, and payment behaviour: every existing subscription, and every Custom subscription created through the current Custom flow, including ones created after this ships. Custom is not stored as a billing cycle in this change. No schedule is inferred from existing start and end dates.
- A declared schedule requires `payment_day` from 1 through 28. An Annual schedule also requires `payment_month` from 1 through 12.
- Reject combinations that contradict the declaration: a day or month without a billing cycle, and a payment month on a Monthly schedule.
- Make the three fields immutable once the record exists, including on existing rows that have none. A term cannot gain, lose, or change a schedule by editing it.
- Make the end date of a term that has a billing cycle read-only on the backend, so the update endpoint cannot move a Monthly or Annual term off its calendar dates. Terms with no billing cycle keep editable end dates, as the TDD requires. Hiding the field in Manage is GOLF-9903.
- Define the allowed billing cycles, payment days and payment months once, on the model, and have the calculator reference them, so the stored schedule and the calculation cannot disagree.
- Add model specs for the rules above and for the unchanged legacy case. Database check constraints are not added; the design records why.
- Not in this change: reading or writing the fields from any endpoint, serializer, or UI; copying them to the next term on renewal, or using the schedule to calculate renewal dates or payments (both GOLF-9898); changing how the calculator calculates (only where its three constants are defined moves); proration; contracts.

## Capabilities

### New Capabilities

- `subscription-term-schedule`: Which billing schedule a subscription term may declare, how the declaration is validated, and that existing terms stay undeclared and unchanged.

### Modified Capabilities

None. `subscription-term-calculation` already says the calculator uses `schedule:` values and leaves existing callers unchanged; this change supplies the stored source for those values without altering any of its requirements.

## Impact

- Implementation repository: `/Users/rushipatel/Documents/Repos/chronogolf`. All `app/`, `db/`, and `spec/` paths below refer to that checkout; planning artifacts live in `golf-specs`.
- New migration in `db/migrate/` adding the three columns, and the corresponding `db/schema.rb` update.
- `app/models/subscription.rb`: validations for the schedule, and the three fields added to `protected_attributes`.
- `spec/models/subscription/validations_spec.rb`: schedule validation and immutability examples. `spec/models/subscription/base_spec.rb`: end-date lock examples in the existing `#amend` block.
- `app/models/subscription/calculator.rb`: its `BILLING_CYCLES`, `PAYMENT_DAYS`, and `PAYMENT_MONTHS` now point at the model's lists, so the allowed values are defined once, on the model. The calculator keeps its constant names, and nothing about how it calculates changes.
- Downstream: 9898 reads the three columns to build `schedule:`, copies them to the next term, and uses the schedule to calculate the renewal; 9900 writes them at creation and must resolve an Annual payment month before saving, because the calculator's default-to-start-month does not apply to a stored Annual schedule; 9903 displays them. Terms with no billing cycle stay on the existing path. A later Custom milestone (TDD M5) will store and calculate `custom` for newly created Custom terms, and terms with no billing cycle will not move to it until a deliberate migration.
- No API, serializer, strong-parameter, frontend, feature-flag, job, or Sorbet RBI changes. Recent column additions in the repo did not regenerate RBIs either.

Sources: [GOLF-9896](https://lightspeedhq.atlassian.net/browse/GOLF-9896); the `subscription-term-calculation` spec and the archived GOLF-9894 design; the user-supplied `/Users/rushipatel/Documents/TDD_ Subscriptions major upgrade (Final).docx` and `/Users/rushipatel/Documents/subscriptions-prototype (Final).html`; and the Q3 planning notes at `.opencode/features/subscriptions-q3/analysis.md` and `plan.md` in the application repository. The design records where this change departs from the TDD.
