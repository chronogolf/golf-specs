# Tasks

Implement the schedule columns, then their rules, then the behaviour that keeps a stored schedule consistent over a term's life. Read `design.md` and `specs/subscription-term-schedule/spec.md` before applying. Nothing reads or writes the columns outside the model: no endpoint, serializer, parameter, or frontend change, no change to renewal (GOLF-9898 owns copying the schedule and dating the renewal), and no change to how the calculator calculates: only where its three constants are defined moves. The update endpoint is not edited; it starts rejecting an end-date change on a declared term only because of the model guard. Verification commands require explicit approval before execution; this plan does not authorize them.

The work is three slices, about 213 changed lines in total, which is over the 200-line ceiling for one reviewable slice. The estimates are forecasts measured from a first implementation written as one change; they count migration, `db/schema.rb`, the model, and specs.

- PR 1 is the migration alone. It is deliberately tiny: with canary deploys, pods running new model code before the columns exist would fail, so the schema must be deployed and complete before any code that uses it. The repository's earlier stacks, for example GOLF-9497 (1/4), ship a migration the same way. Merging it into PR 2 would defeat that.
- PR 2 and PR 3 split on concern: what a term may declare, then what happens to a stored schedule afterwards. PR 3's two behaviours, immutability and the end-date lock, are each small and both need the declared state PR 2 defines, so they stay together.
- PR 2 includes a 6-line move of the calculator's three constants onto the model. It is a refactor, kept with the validations because it has no purpose without them; a separate slice would be a diff that says little on its own.

## PR 1: Schedule columns migration (~14 changed lines)

Depends on: master. Estimated split: 9 migration lines and 5 `db/schema.rb` lines. Review concern: the migration is additive, reversible, and safe to deploy ahead of any code that uses it. It has no spec counterpart because nothing uses the columns yet; the examples that exercise them arrive in PR 2.

- [x] 1.1 Add one migration (`ActiveRecord::Migration[7.1]`, timestamp after `20261109120000`) adding nullable `billing_cycle` (string), `payment_day` (integer), and `payment_month` (integer) to `subscriptions`, with no default and no backfill, and update `db/schema.rb` to match.
- [x] 1.2 On this slice's own branch, based on master, obtain approval for the verification commands below, then run them. Verify the migration applies, rolls back, and re-applies; that `strong_migrations` raises nothing and `safety_assured` is not used; that the `db/schema.rb` diff is only those three columns and the version bump; and that the existing subscription specs still pass with the columns present, which shows master stays green. Record the precise blocker and any unavailable gate; do not count unrun commands as passing.
- [x] 1.3 Request a fresh independent read-only review of the migration and schema diff. Verify every material finding is addressed, approved affected checks are rerun, and a fresh focused re-review covers any resulting fixes. Use the reviewer model that did not implement the change.
- [x] 1.4 Review this slice's final diff. Verify it is limited to the migration and `db/schema.rb`, with no model, spec, serializer, controller, strong parameter, calculator, frontend, feature-flag, or RBI change.

Planned commands for this slice, to run only after approval:

```bash
bin/rails db:migrate
bin/rails db:rollback
bin/rails db:migrate
bundle exec rspec spec/models/subscription/validations_spec.rb spec/models/subscription/base_spec.rb spec/models/subscription/calculator_spec.rb spec/controllers/private_api/clubs/subscriptions_controller_spec.rb
bundle exec standardrb db/migrate
bundle exec srb tc
```

## PR 2: Schedule validation (~110 changed lines)

Depends on: PR 1, using its branch as this slice's base. Estimated split: ~27 model lines, ~6 calculator lines, and ~77 spec lines. Review concern: which combinations of the three fields are valid, where the allowed values are defined, that an existing term stays valid and undeclared, and that the stored values are acceptable to the calculator.

- [x] 2.1 Define `BILLING_CYCLES` (`monthly`, `annual`), `PAYMENT_DAYS` (1 through 28), and `PAYMENT_MONTHS` (1 through 12) as constants on `Subscription` in `app/models/subscription.rb`, and point the calculator's constants of the same names in `app/models/subscription/calculator.rb` at them, leaving the rest of the calculator untouched. Add the schedule validations to the model as described in design decision 2: `billing_cycle` NULL, `monthly`, or `annual`; day required and 1 through 28 for a declared schedule; month required and 1 through 12 for Annual; day and month absent when there is no cycle; month absent for Monthly. Check the value as assigned so a fractional day is rejected rather than cast to a whole number. In `spec/models/subscription/validations_spec.rb`, add examples for NULL schedule, Monthly days 1 and 28, days 0, 29, 31, and 3.5, Annual months 1 and 12, months 0 and 13, Annual without a month, Monthly with a month, orphan day and orphan month, and cycle `custom`. Verify each rejected case reports its error on the named field, and that the calculator spec passes unedited.
- [x] 2.2 In the same spec file, add examples that an existing-style subscription (built with no schedule fields) loads, saves, and reloads with all three fields still NULL and its other attributes unchanged, and that a valid stored schedule passed as the calculator's `schedule:` raises no billing-cycle, payment-day, or payment-month error. Verify the calculator example uses the stored record's values rather than literals.
- [x] 2.3 On this slice's own branch, based on PR 1's branch, obtain approval for the verification commands below, then run them. Verify they pass or record the precise blocker and any unavailable gate; do not count unrun commands as passing. Because the changed Ruby files are not in `.standard_todo.yml`, StandardRB offenses there fail `lint_backend`.
- [x] 2.4 Request a fresh independent read-only review of this slice's model and spec diff and of the callers that build or update subscriptions. Verify every material finding is addressed, approved affected checks are rerun, and a fresh focused re-review covers any resulting fixes. Use the reviewer model that did not implement the change.
- [x] 2.5 Review this slice's final diff. Verify it is limited to `app/models/subscription.rb`, `app/models/subscription/calculator.rb` (only its three constants), and `spec/models/subscription/validations_spec.rb`, and that it contains no protected-fields change, end-date guard, renewal change, migration, serializer, controller, strong parameter, frontend, feature-flag, or RBI change, and no change to how the calculator calculates.

Planned commands for this slice, to run only after approval:

```bash
bundle exec rspec spec/models/subscription/validations_spec.rb spec/models/subscription/base_spec.rb spec/models/subscription/calculator_spec.rb spec/controllers/private_api/clubs/subscriptions_controller_spec.rb
bundle exec standardrb app/models/subscription.rb app/models/subscription/calculator.rb spec/models/subscription/validations_spec.rb
bundle exec srb tc
```

## PR 3: A saved schedule and end date stay fixed (~89 changed lines)

Depends on: PR 2, using its branch as this slice's base. Estimated split: ~11 model lines and ~78 spec lines. Review concern: after a term is saved, its schedule cannot be edited and, once it has a billing cycle, its end date cannot move off the schedule.

- [x] 3.1 Add `billing_cycle`, `payment_day`, and `payment_month` to `protected_attributes` in `app/models/subscription.rb`. In `spec/models/subscription/validations_spec.rb`, following the existing `refuses changing starts_at` example, add: changing a declared day is rejected, changing Monthly to Annual is rejected, adding a schedule to a saved term with none is rejected, and updating `auto_renew` on a declared term still succeeds. Verify the stored schedule is unchanged after each rejected update.
- [x] 3.2 Reject a changed end date on a term whose stored billing cycle is present, with a validation in `app/models/subscription.rb` that compares the end date as a club-local calendar date rather than as a raw timestamp (resending the same date still differs in the sub-microsecond digits), and that reports a missing end date as invalid instead of raising. In the existing `#amend` examples in `spec/models/subscription/base_spec.rb`, add: amending a declared Monthly term's end date and an Annual term's end date, each to a different calendar day, are rejected with an error on `ends_at` and leave the stored date unchanged; amending only auto-renew with the term's current end date succeeds on both a Monthly and an Annual term (the legacy Manage form may resend the end date, so run this through the amend call rather than a plain assignment); amending the end date of a term with no schedule still succeeds. Add a validation example for an update that sets the end date to nil. Also add to the existing `#renewal` examples that a term with no schedule renews to a term with none, which shows renewal is unchanged. Verify the existing `#amend` and `#renewal` examples pass unedited.
- [x] 3.3 On this slice's own branch, based on PR 2's branch, obtain approval for the verification commands below, then run them. Verify they pass or record the precise blocker and any unavailable gate; do not count unrun commands as passing.
- [x] 3.4 Request a fresh independent read-only review of this slice's model and spec diff and of the callers that update, amend, import, or renew subscriptions. Verify every material finding is addressed, approved affected checks are rerun, and a fresh focused re-review covers any resulting fixes. Use the reviewer model that did not implement the change.
- [x] 3.5 Review this slice's final diff. Verify the application change is limited to `app/models/subscription.rb`, `spec/models/subscription/validations_spec.rb`, and `spec/models/subscription/base_spec.rb`, that no migration, serializer, controller, strong parameter, calculator, frontend, feature-flag, or RBI change slipped in, and that `RENEWAL_ATTRIBUTES`, `renewal`, and `Subscription::Expire` are unchanged.

Planned commands for this slice, to run only after approval:

```bash
bundle exec rspec spec/models/subscription/validations_spec.rb spec/models/subscription/base_spec.rb spec/models/subscription/calculator_spec.rb spec/controllers/private_api/clubs/subscriptions_controller_spec.rb
bundle exec standardrb app/models/subscription.rb spec/models/subscription/validations_spec.rb spec/models/subscription/base_spec.rb
bundle exec srb tc
```

`calculator_spec.rb` and the controller spec are included only to show that existing calculation and creation behavior is unchanged. They are not edited. No UI workflow or E2E seeder is needed: the fields are not visible anywhere. Each slice was run on its own branch with these commands: PR 1 (211 examples) and PR 2 (223) with 0 failures and with StandardRB and `srb tc` clean, tracked by tasks 1.2 and 2.3. PR 3 is tracked by task 3.3.

## Scenarios without a dedicated example

Three spec scenarios are verified by inspection rather than by a new example, because they hold by construction and the existing controller spec still passes: "Calendar-aligned dates do not imply a schedule" (nothing infers a schedule, so no code path could set one), "Custom term created later" (the factory builds a term only from the attributes it lists, which include no schedule field), and "Fields are not reachable through the API" (the update and create endpoints permit a fixed list of fields that excludes them, and no serializer dumps every column). A dedicated example for each can be added if a reviewer wants one.

## Workflow follow-up

- Archive the change after the project's review requirements are satisfied.
- Sync `specs/subscription-term-schedule/spec.md` into the main specs on archive.
