# Tasks

Build the two pieces the declared renewal needs, then wire them into `Subscription::Expire`. Read `design.md` and `specs/subscription-declared-renewal/spec.md` before applying. Nothing here changes creation, proration, the periodic job, the declared-creation flag, serializers, or the frontend. Verification commands require explicit approval before execution; this plan does not authorize them.

The work is two slices, about 100 changed lines in total including specs (forecasts, not measurements):

- PR 1 adds the building blocks and nothing calls them, so master behaves exactly as before.
- PR 2 turns on the declared branch. Its diff is small because the pieces were reviewed in PR 1; a reviewer judges it by the behavior scenarios in `expire_spec.rb`.

## PR 1: Declared successor term and explicit agreement payments (~55 changed lines)

Depends on: master. Estimated split: ~10 lines in `app/models/subscription.rb`, ~12 in `app/models/sale/agreement_template.rb`, ~33 in specs. Review concern: the successor term takes its dates and schedule from a calculation, and the agreement template can take its payments explicitly while legacy calls are unchanged.

- [x] 1.1 In `app/models/subscription.rb`, make the existing `schedule_declared?` public, and add a method next to `renewal` that builds an unsaved successor from a `Subscription::Calculator.next_term` result: same customer, club, and auto-renew, the three schedule fields copied, lines built with `price_from_source`, `starts_at` the start of the calculated start date and `ends_at` the end of the calculated end date in the club's timezone. The method calls the existing `renewal` and then overrides the end date and adds the schedule fields, so `RENEWAL_ATTRIBUTES` and `renewal` stay untouched. In `spec/models/subscription/base_spec.rb`, add examples for a Monthly and an Annual term, a term ending on a daylight-saving boundary, and a club in a non-UTC timezone. Verify the successor's dates are club-local day boundaries and the existing `renewal` examples pass unedited.
- [x] 1.2 In `app/models/sale/agreement_template.rb`, add an optional `payments:` argument to `new_instance`, a list of `{amount:, due_at:}`. When given, use it instead of copying the previous count and splitting the total; keep the "payment before `sold_at`" check, now against the earliest supplied payment. Without it, behavior is unchanged. In `spec/models/sale/agreement/template_spec.rb`, add examples: one supplied payment replaces two previous payments; and a payment dated before `sold_at` is still refused. Verify the existing template examples, which cover the legacy call without `payments:`, pass unedited.
- [x] 1.3 On this slice's own branch, based on master, obtain approval for the verification commands below, then run them. Verify they pass or record the precise blocker and any unavailable gate; do not count unrun commands as passing. Changed Ruby files that are not in `.standard_todo.yml` fail `lint_backend` on any offense.
- [x] 1.4 Request a fresh independent read-only review of this slice's diff and of the other callers of `AgreementTemplate#new_instance`. Verify every material finding is addressed, approved checks are rerun, and a fresh focused re-review covers any fixes. Use the reviewer model that did not implement the change.
- [x] 1.5 Review this slice's final diff. Verify it is limited to the three files above and their two specs, that `Subscription::Expire` is unchanged, and that no migration, serializer, controller, calculator, flag, frontend, or RBI change slipped in.

Planned commands for this slice, to run only after approval:

```bash
bundle exec rspec spec/models/subscription/base_spec.rb spec/models/sale/agreement/template_spec.rb spec/use_cases/subscription/expire_spec.rb
bundle exec standardrb app/models/subscription.rb app/models/sale/agreement_template.rb spec/models/subscription/base_spec.rb spec/models/sale/agreement/template_spec.rb
bundle exec srb tc
```

## PR 2: Renew declared terms from their stored schedule (~50 changed lines)

Depends on: PR 1, using its branch as this slice's base. Estimated split: ~30 lines in `app/use_cases/subscription/expire.rb` and ~20 in `spec/use_cases/subscription/expire_spec.rb`, plus the scenario examples. Review concern: a declared term renews from its schedule, every other term renews exactly as before, and a second attempt never creates a second successor.

- [x] 2.1 In `app/use_cases/subscription/expire.rb`, after the existing lock, stop, and eligibility steps, branch on `schedule_declared?`. Declared: call `Subscription::Calculator.next_term` with the stored schedule, fail and roll back on calculator errors through `fail_and_rollback`, save and start the successor from 1.1, and build its agreement with `sold_at` equal to the successor's `starts_at` and the calculation's one payment through 1.2. Not declared: the existing code, untouched. Do not read any feature flag. Verify the existing legacy `expire_spec.rb` examples pass unedited.
- [x] 2.2 In `spec/use_cases/subscription/expire_spec.rb`, add a "declared schedule" context with a real (unstubbed) lock covering each spec scenario: copied Monthly and Annual schedule; on-time and one-day-late runs, asserting `starts_at`, `ends_at`, and the agreement's `sold_at` and payment due date; a club in another timezone at the day boundary; first payment moved to the sale day on the original; a payment day that has already passed stays overdue and is charged; current prices after a price change, for a tax-inclusive and a tax-exclusive club; one regular payment when the previous agreement had several; a Custom term still on the legacy path; cancelled, auto-renew off, subscriptions disabled, and already-renewed terms creating nothing; a calculator error leaving nothing created and the expiring term unstopped; and a club-timezone example. The "declared creation off still renewing" example is not written: the flag does not exist yet (GOLF-9900 adds it), and the declared path reads no flag. Verify each example asserts on persisted records, not only the returned result.
- [x] 2.3 Add a repeated-attempt example (call `Expire` twice against real, unstubbed lock and find) and an example that the expiring term is locked `FOR UPDATE` before a successor is looked for, to the same spec. Verify exactly one successor, one agreement, and one payment exist after the second call. A true two-thread race is not written: the specs run in transactional fixtures, which hide uncommitted rows from a second connection, and the repo has no committed-data pattern for database concurrency. The lock plus the already-renewed check is what prevents the duplicate; the repeated-attempt example exercises the same check against committed state.
- [x] 2.4 On this slice's own branch, based on PR 1's branch, obtain approval for the verification commands below, then run them. Verify they pass or record the precise blocker and any unavailable gate; do not count unrun commands as passing.
- [x] 2.5 Request a fresh independent read-only review of this slice's diff, of `Subscription::ExpireWorker` and `Periodic::SubscriptionWorker` as callers, and of the SonarCloud coverage of new lines. Verify every material finding is addressed, approved checks are rerun, and a fresh focused re-review covers any fixes. Use the reviewer model that did not implement the change.
- [x] 2.6 Review this slice's final diff. Verify it is limited to `expire.rb` and its spec, that the legacy branch is the old code unmodified, and that no flag, worker, migration, serializer, or RBI change slipped in.

Planned commands for this slice, to run only after approval:

```bash
bundle exec rspec spec/use_cases/subscription/expire_spec.rb spec/workers/subscription/expire_worker_spec.rb spec/workers/periodic/subscription_worker_spec.rb spec/models/subscription/base_spec.rb spec/models/sale/agreement/template_spec.rb
bundle exec standardrb app/use_cases/subscription/expire.rb spec/use_cases/subscription/expire_spec.rb
bundle exec srb tc
```

## Workflow follow-up

- After PR 2 merges, GOLF-9906 builds on this path to skip proration on renewal.
- Archive the change after the project's review requirements are satisfied, and sync `specs/subscription-declared-renewal/spec.md` into the main specs on archive.
