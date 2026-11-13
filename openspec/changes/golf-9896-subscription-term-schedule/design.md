# Design

## Context

Application, migration, and test paths in this design resolve under `/Users/rushipatel/Documents/Repos/chronogolf`. The change artifacts live in the `golf-specs` planning repository.

See `proposal.md` for motivation and `specs/subscription-term-schedule/spec.md` for the behavior contract. A design is included because the placement of the rules (model versus database), the handling of values an integer column would silently coerce, and the immutability choice each have a credible alternative.

Relevant existing code:

| Location | Existing behavior | How this change uses it |
| --- | --- | --- |
| `db/schema.rb` (`subscriptions`) | A term is one row with `starts_at`, `ends_at`, `auto_renew`, and `renewed_from_subscription_id`. There is no schedule column. | Add three nullable columns here. No separate table: the schedule belongs to the term, and legacy rows must read as "none". |
| `app/models/subscription.rb` (`validates`, `ends_at_after_start`) | Presence and date-order validations on create and update. | Add the schedule validations alongside them. |
| `app/models/subscription.rb` (`protected_attributes`) with `ProtectedAttributesValidator` | On update, any change to `starts_at`, `customer_id`, or `provider_id` is rejected. The validator compares `record.changes` against the list, so a nil-to-value change counts. | Add the three schedule fields to the list. This gives immutability, including for terms that have no schedule, with no new mechanism. |
| `app/models/subscription.rb` (`RENEWAL_ATTRIBUTES`, `renewal`) | Renewal copies `customer_id`, `provider_id`, `auto_renew` and recomputes dates from the previous term's length. | Not edited. Copying the schedule to the next term and using it to date the renewal are GOLF-9898. |
| `app/models/subscription.rb` (`amend`) with `app/use_cases/subscription/amend.rb` | Staff can set a new end date on a saved term through the update endpoint, and it is saved as sent. | A validation rejects a changed end date for a term with a billing cycle. The endpoint and use case are not edited. |
| `app/interactors/subscription/create.rb` | After saving a new term, adjusts its end date when auto-renew is on. With a checkout, a later update saves the adjustment; without one, it is never saved. | Not edited. Declared creation (GOLF-9900) must skip the adjustment, because with a checkout the guard would reject it. |
| `app/models/subscription/calculator.rb` | Defines `BILLING_CYCLES`, `PAYMENT_DAYS`, and `PAYMENT_MONTHS`, and takes `schedule:` with exactly these three keys. | The three constants move to the model, and the calculator points its own names at them. Nothing about how it calculates changes. |
| `app/controllers/private_api/clubs/subscriptions_controller.rb` | Creation builds a `Subscription::Request` from an explicit list of permitted fields. | No change. The new columns are not permitted, so they cannot be set from a request. |
| `app/serializers/subscription_serializer.rb`, `partner_api/v2/subscription_serializer.rb`, `subscription_report_serializer.rb` | Attributes are listed explicitly. | No change. New columns are not rendered until GOLF-9903 adds them deliberately. |
| `spec/models/subscription/validations_spec.rb` | Existing home for validation and immutability examples (`refuses changing starts_at`). | Add the schedule examples here, following its structure. |

## Goals / Non-Goals

**Goals:**

- Store the declared schedule on the term in the exact vocabulary the calculator already consumes, so GOLF-9898 can build `schedule:` directly from the record.
- Make an invalid or contradictory schedule unsavable, and make a saved schedule, and a declared term's end date, impossible to alter through ordinary updates.
- Ship as additive deploys that existing code, running mid-deploy, does not notice.

**Non-Goals:**

- No endpoint, serializer, parameter, or UI use of the columns.
- No helper that turns the three columns into the calculator's `schedule:` hash. GOLF-9898 is the first reader and can shape it to its needs.
- No change to how renewal calculates dates or payments, or to creation or payment, and no change to how the calculator calculates. Only where its three constants are defined moves.
- No backfill and no attempt to derive a schedule for existing terms.

## Decisions

### 1. Three nullable columns on `subscriptions`

Add `billing_cycle` (string), `payment_day` (integer), and `payment_month` (integer), all nullable, with no default. `billing_cycle` stores `"monthly"` or `"annual"`, the same strings the calculator already compares against. The numeric columns use `integer`, like the table's other numeric columns (for example `customer_id` and `provider_id`). The TDD's data model lists small integers for the day and month; `db/schema.rb` has no small-integer column anywhere and the saving would be two bytes per row, so this follows the schema's existing convention. A string column also leaves room for a later `custom` value without a type change.

Nullable with no default is the cheapest possible change in PostgreSQL: existing rows are not rewritten, and `strong_migrations` raises nothing. Making NULL the legacy state also removes any need for a backfill: an existing term and a Custom term are already correctly described as "no schedule".

Alternatives considered:
- A separate `subscription_schedules` table: needs a join on every read and a row-versus-no-row distinction for legacy terms, for no benefit. The ticket and the Q3 planning notes both place the schedule on the term.
- A Rails enum for `billing_cycle`: adds `monthly`, `annual`, `monthly?`, and `annual?` to a model that is already large, plus its own error behavior on invalid assignment. A plain validation is smaller and keeps unsupported values reportable as validation errors.
- A default of `"custom"` or a sentinel for legacy rows: rejected by the ticket. Custom and legacy must both remain NULL.

### 2. Validate in the model, which owns the allowed values

The validations apply to every create and update:

| Billing cycle | Payment day | Payment month |
| --- | --- | --- |
| NULL | must be NULL | must be NULL |
| `monthly` | whole number 1 through 28 | must be NULL |
| `annual` | whole number 1 through 28 | whole number 1 through 12 |
| anything else | rejected on `billing_cycle` | not evaluated further |

The allowed billing cycles, payment days, and payment months are defined as constants on the model (`BILLING_CYCLES`, `PAYMENT_DAYS`, `PAYMENT_MONTHS`), and the calculator's own constants of the same names now point at them. The model owns what a stored row may contain, and the calculator, which consumes a schedule but does not own what a stored row may contain, reads from it. One list serves both, which is what the "stored schedule is calculable" requirement needs, and the calculator can no longer drift from the model because it holds no list of its own. The calculator keeps its constant names, so nothing in it or its spec changes.

Alternatives considered:
- Have the model read the calculator's constants, which was the first draft of this design. The dependency then points from the data to a service, and the calculator already depends on the model, so it is circular. It also needed lazy lookups, because the calculator loads the product and payment classes and could do so while the model is still being defined. And it ties the stored values to the calculator's input vocabulary, which will not always match: the prototype's "Last day" payment day needs a stored representation of its own in an integer column, and a stored `custom` cycle will have its own per-cycle rules.
- Duplicate the constants on the model and add a spec that fails if they differ. This removes the circularity at the cost of two lists and a test to keep them in step. Pointing the calculator at the model gets the same independence with a single list.

Later Custom work: adding `custom` or a stored form of "Last day" to the model's lists also widens what the calculator accepts, because it points at the same lists. If the calculator must keep rejecting a value, as `subscription-term-calculation` requires for Custom today, that change has to give the calculator its own list again. The existing calculator specs that expect `custom` to be rejected guard against doing this by accident.

Whole-number checks must look at the value as assigned, not the cast value. An integer attribute casts `3.5` to `3` and `"3.5"` to `3`, so a range check on the cast value would accept a fractional day and store a different number than the caller sent. Use a numericality-style validation, which evaluates the raw input, rather than an inclusion check on the attribute.

The strict rule that a Monthly term must have no payment month follows the calculator, which ignores it. A stored value the calculator never reads would only mislead a later reader of the record. See the first risk below.

### 3. Fixed schedule through the protected list, fixed end date through its own check

Add `billing_cycle`, `payment_day`, and `payment_month` to `protected_attributes`. `ProtectedAttributesValidator` runs on update and rejects any change to a listed field. A nil-to-value change is a change, so a legacy term also cannot gain a schedule. That is what makes "existing records remain unchanged" enforced rather than merely observed, and it matches the Q3 plan's "protected schedule/dates".

No new mechanism is added for the schedule fields: the list that already protects `starts_at` and the two ownership fields protects them too. Terms are created with their schedule, so this does not block creating the term itself.

The end date is protected as well, but only for a term that already has a billing cycle. Staff can edit a term's end date today, and for a Monthly or Annual term that breaks the schedule: a term that should run Dec 1 to Dec 31 and is cut to Dec 20 makes the next term start Dec 21 and end Jan 20, off the calendar for good. The TDD keeps end-date editing for Custom and makes it read-only for declared terms, and the Manage page will hide the field in GOLF-9903. The guard belongs on the backend too, because the update endpoint accepts an end date whatever the page shows, and declared terms can be created (GOLF-9900) before the page changes (GOLF-9903).

The end date is guarded by its own validation rather than the protected list, because the list compares raw timestamps. The Manage form resends the end date when staff only change auto-renew: a date such as Dec 31 becomes 23:59:59.999999999, the database keeps only microseconds, and Rails then sees the same date as changed (checked against Rails' change detection). A raw-timestamp guard would therefore reject a harmless auto-renew change on a declared term. The guard compares the end date as a calendar date in the club's timezone instead, and applies only when the term already has a billing cycle stored. A term with no cycle behaves exactly as before, and a term given a cycle and a new end date in the same update is rejected anyway by the schedule rule.

Two things save a new end date on an existing term today: the amend action, and a step in the creation flow that adjusts a freshly created term's end date when auto-renew is on. The guard rejects both for a declared term, with a caveat on the second: the adjusted date is only saved when creation includes a checkout, because a later update then saves the whole record. Without a checkout the adjustment is never saved, and the guard never fires. Rejecting the first is the intent. The second means GOLF-9900 must skip that adjustment for declared terms, which its plan already calls for ("bypass Custom date snapping"). If it forgets, the later update raises a record-invalid error inside the creation transaction, an unhandled exception rather than a validation failure, instead of silently moving a declared term off its calendar.

Alternatives considered:
- Leave the fields editable. Simpler, but a later bug could change a term's payment day and silently desynchronize it from its payments and from what renewal will copy. Immutability is cheap to relax later and expensive to retrofit after data exists.
- Protect only against changing a declared schedule, but allow nil-to-value. This reopens the legacy-term case the ticket explicitly rules out.
- Lock the end date in the Manage page only (GOLF-9903). This leaves the update endpoint open and a window between 9900 and 9903.
- Lock the end date of every term. The TDD keeps end-date editing for Custom terms.
- Add the end date to the protected list for declared terms. It compares raw timestamps, so the auto-renew case above would be rejected.

### 4. No database check constraints in this change

The invariants in decision 2 are enforced by model validations only. A `CHECK` constraint could guard against writes that skip validations (`update_columns`, raw SQL), but nothing writes these columns that way, and it would add a second migration: `strong_migrations` requires `add_check_constraint ... validate: false` followed by a separate `validate_check_constraint` migration, as in `db/migrate/20260812115028_add_uuid_not_null_check_to_discount_rules.rb` and `db/migrate/20260812115520_validate_uuid_not_null_check_on_discount_rules.rb`. For three columns that are all NULL on day one, that is more ceremony than protection. The allowed set is also expected to grow: a later Custom milestone stores `custom`, and a `CHECK` listing `monthly` and `annual` would need its own migration to change.

This is the one choice most worth revisiting. If a later ticket adds a writer that bypasses validations (a bulk import or a data fix), add the constraint pair then. The columns and rules are designed so it can be added without changing either.

### 5. Leave renewal to GOLF-9898

Renewal is not touched. [GOLF-9898](https://lightspeedhq.atlassian.net/browse/GOLF-9898)'s first acceptance criterion is "Renewal copies the billing cycle, payment day, and payment month", and that ticket also replaces the date logic a declared term must not reuse. The TDD's remark that the schedule is "copied at renewal through RENEWAL_ATTRIBUTES" describes the epic, not this ticket. Copying here would half-build renewal: a successor that carries the schedule but is still dated by the old logic.

Consequence: until 9898, renewing a declared term through the existing path produces a successor with no schedule. No declared term can exist before GOLF-9900 creates one, and 9898's deployment order requires renewal support before creation is enabled.

Alternative considered: add the three fields to the renewal attributes here, which was an earlier draft of this design. That moves one of 9898's criteria into 9896 and leaves renewal half-built in between.

### 6. Custom is not stored; no billing cycle marks the existing behaviour

In Q3 a Custom subscription created from the new Add form still posts today's Custom request, so it is stored with no billing cycle, exactly like an existing row. The TDD says the same: "billing_cycle NULL means legacy", and Custom created from the form "behave[s] as today". NULL is therefore the routing marker: readers such as GOLF-9898 take the existing path when there is no cycle and the calculator path when there is one. "Legacy" is defined by that path, not by creation date, so a Custom term created in Q3 stays on the existing path.

A later Custom milestone (TDD M5) is expected to store `custom` for newly created Custom terms and calculate them with the calculator. Terms with no billing cycle, legacy and Q3 Custom alike, are not moved there automatically, because the existing path, which this change leaves alone, behaves differently from the calculator in three ways:

- Today, with auto-renew on, creation moves the end date of a Custom term that is about a month long (26 to 33 days) or a year long (363 to 367 days), so the term finishes the day before the same date a month or a year later. A term entered as Jan 5 to Feb 3 is saved as Jan 5 to Feb 4. The calculator has no such adjustment.
- Today, a renewal copies the payment day from the first payment. If the first payment happened to be collected on the 17th, every renewal is charged on the 17th. The calculator uses the declared payment day instead.
- Today, a renewal guesses its length from the previous term. A term of about a month renews for a month, about a year for a year, and anything else for the same number of days. The calculator uses the declared billing cycle instead.

Putting a legacy term on the calculator would therefore change its dates and payments, so these terms move only through a deliberate migration. The existing behaviour lives in `app/interactors/subscription/create.rb`, `app/use_cases/subscription/expire.rb`, and `app/models/subscription.rb`.

Alternatives considered:
- Store `custom` now. The calculator deliberately rejects Custom, Q3 does not upgrade Custom, and a stored value would imply a calculation path that does not exist yet.
- Backfill existing rows as Custom or infer a cycle from their dates. This would route legacy terms through the calculator and change their dates and payments, which the TDD lists as out of scope for Q3.

## Risks / Trade-offs

- [Creation must resolve the Annual payment month itself] The calculator defaults a missing Annual payment month to the term's start month, but a stored Annual term requires one. → GOLF-9900 resolves the default before saving. Recorded in the proposal's Impact so that ticket's author sees it.
- [Strict Monthly rule] Rejecting a payment month on a Monthly term means a form that keeps an Annual month in its state after the user switches to Monthly must clear it before saving. → The failure is a clear validation error on `payment_month`, not silent bad data. Loosening to "ignored" later is backward compatible; tightening later is not.
- [Immutability blocks a future edit feature] If a later ticket lets staff change a schedule, it must remove the fields from `protected_attributes`. → The Q3 plan treats calendar dates as read-only, and relaxing a protection is a one-line change.
- [Rolling deploy with the new columns] Pods running old code do not know the columns and never write them; pods running new code treat NULL as valid. → Nothing reads or writes the columns outside the model until later tickets, so there is no mixed-version behavior to guard.
- [Validations can be skipped by direct writes] `update_columns` or raw SQL bypass them. → Accepted per decision 4; no current code path writes these fields that way. The customer CSV import also saves with validations off. It selects terms by having no sale agreement, not by having no schedule, and a declared term without an agreement is valid model state, so the model does not prevent the import from changing one. No production creation path supplies a schedule yet, so no such term exists to reach. If GOLF-9900 lets a declared term be created without a sale agreement, the import needs its own check first.
- [Moving legacy terms to a cycle later conflicts with immutability] A deliberate migration that sets a billing cycle on existing terms would be rejected if it saved through the model, because the schedule fields are protected. → That migration is a one-time operation that writes outside ordinary updates, or it adjusts the protection explicitly. Decide when the migration is designed; nothing here needs to anticipate it.
- [Adding `custom` later changes the per-cycle rules] The rules in decision 2 are written for Monthly and Annual. → The Custom milestone adds its own requirements as a delta to this capability. The "no cycle means no day or month" rule is unaffected, since it protects the existing path.
- [A declared term renewed before GOLF-9898 loses its schedule] The existing renewal copies only the attributes it always has, so a declared term's successor would have no schedule. → No declared term exists until GOLF-9900 creates one, and 9898's deployment order requires renewal support before creation is enabled.
- [Creation adjusts the end date after saving] The existing creation flow adjusts a new term's end date after saving it. When creation includes a checkout, a later update saves that adjustment, and the guard rejects it for a declared term by raising a record-invalid error inside the creation transaction. Without a checkout the adjustment is never saved. → GOLF-9900 must skip that adjustment for declared terms, which its plan already requires. Failing loudly is the intended outcome if it forgets, but it will surface as an exception rather than a validation message.
- [Manage shows a generic error until GOLF-9903] The page still offers the end date for every term, so staff editing a declared term see "cannot be changed". → Acceptable: declared creation is gated, and 9903 hides the field.
- [The auto-renew toggle may resend the end date] The legacy Manage form saves the whole subscription, so toggling auto-renew may send the end date unchanged. → The guard only rejects a changed value, and a spec example covers amending auto-renew off with the current end date. Turning auto-renew on recomputes the end date, which for calendar-aligned Monthly and Annual terms equals the current date (traced by hand, not covered by an example); that path needs an automatic payment method, so it is exercised once GOLF-9900 creates declared terms.

## Migration Plan

1. One migration adds the three nullable columns; `db/schema.rb` is updated to match. No backfill, no data migration, no second step.
2. Ship in three slices (see `tasks.md`). Slice 1 deploys the columns alone, so the schema is in place on every pod before any code that uses it, which matters for canary deploys. Slice 2 adds the validation that gives the columns meaning. Slice 3 adds immutability and the end-date lock. Until slice 2 nothing knows about the columns, and until slice 3 nothing writes them after save and nothing yet prevents it. Existing rows read as undeclared and stay valid throughout.
3. Rollback: revert the model change first, slice 3 before slice 2. The columns can stay in place unused; dropping them later, if ever wanted, follows the usual two-step removal because the model has loaded them.
