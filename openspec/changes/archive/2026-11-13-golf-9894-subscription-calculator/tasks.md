# Tasks

Implement only the calculator and its focused specs. Read `design.md` and `specs/subscription-term-calculation/spec.md` before applying. Preserve current prices, tax calculations, and payment rules, including saved bundle prices. Track deferred cases in [GOLF-10170 — Edge cases](https://lightspeedhq.atlassian.net/browse/GOLF-10170); they are not additional implementation requirements. Verification commands require explicit approval before execution; this plan does not authorize them.

Estimates below include implementation and specs, counting additions and deletions. They are forecasts based on the planned methods and test cases, not measured diffs. About 200 lines is a ceiling, not a target: split further even below that when both parts remain meaningful and independently shippable. Allow for uncertainty when fixtures or existing quote behavior require more setup. Check the split before writing code; move boundaries when necessary instead of removing tests or adding abstractions to satisfy a number. More than six slices needs a stated reason.

PRs 1 through 4 build the quote, calendar, and payment portions already needed by the design, inside `app/models/subscription/calculator.rb`, with their corresponding tests in `spec/models/subscription/calculator_spec.rb`. They do not expose stubbed `first_term` or `next_term` methods. PRs 5 and 6 add those public methods only when their supporting behavior is ready. No extra helper classes, services, files, or refactors are introduced to create these slices. Existing callers stay untouched throughout the stack; every slice must be independently reviewable and shippable, with master green.

The boundaries separate quote building, term dates, recurring payment dates, payment-row choices, the complete first-term call, and the next-term call. Do not split result/input setup away from quoting, or source-price selection away from the next-term call: those smaller pieces need their neighboring behavior to explain their purpose. Each slice's own tests must make its behavior understandable without opening another slice's diff.

## PR 1: Quote building and shared result (~190 changed lines)

Depends on: master. Estimated split: ~45 implementation lines + ~145 spec lines. Review concern: how resolved product-and-price inputs become an unsaved quote and the common result, without changing existing pricing or tax arithmetic.

- [x] 1.1 Before any implementation, check all six line estimates against the planned implementation and test setup. Verify every heading shows an estimate including specs, the slices are balanced, each diff is meaningful on its own, and each can ship without incomplete public methods. Split further wherever that remains true, including below 200 lines; merge a boundary that needs its neighboring diff to make sense. Revisit the grouping before coding if these assumptions do not hold; do not add or drop feature work.
- [x] 1.2 Add `Subscription::Calculator`, its colocated result, and the shared quote/input portion of the planned implementation. Accept resolved product-and-price lines and define dates/payments/quote/errors without duplicate totals. Add result-field and unsupported-product/price input examples. Verify calculation does not look up products by ID and no existing caller is edited; public first/next-term method checks belong to PRs 5 and 6.
- [x] 1.3 Build a fresh unsaved quote, reusing current tax calculations, number types, and rounding. Add ordinary override, zero-priced-product-with-paid-product, mixed tax-rate, inclusive/exclusive, and rounding examples. Verify subtotal/tax/total are read from the quote and match existing arithmetic, no monetary-type conversion is introduced, and calculation saves no quote, quote line, or sale item.
- [x] 1.4 Obtain approval for the verification commands below, then run the calculator specs, StandardRB on changed Ruby files, and the backend typecheck for this slice. Verify they pass or record the precise blocker and any unavailable gate; do not count unrun commands as passing.
- [x] 1.5 Request a fresh independent read-only review of this slice's calculator/spec diff and relevant quote/pricing callers. Verify every material finding is addressed, approved affected checks are rerun, and a fresh focused re-review covers any resulting fixes.

## PR 2: Calendar term boundaries and start eligibility (~125 changed lines)

Depends on: PR 1, using its branch as this slice's base. Estimated split: ~50 implementation lines + ~75 spec lines. Review concern: when a term may start and where it ends, including club-local dates, without payment-day or collection rules.

- [x] 2.1 Implement the shared calendar portion: first eligible month selection/defaulting, strict start-month validation, and Monthly/Annual boundaries. Include conversion of the first-term timestamp to the club timezone. Add November 1 versus November 2, earlier/later month, leap/non-leap February, Annual year-boundary, UTC-versus-Los-Angeles, and daylight-saving-month examples. Verify expected local dates and field/code errors; expose the complete `first_term` method and its default-clock tests in PR 5.
- [x] 2.2 Run the same approved calculator specs, StandardRB on changed Ruby files, and backend typecheck for this slice; obtain approval if not already granted. Verify they pass or record the precise blocker and any unavailable gate; do not count unrun commands as passing.
- [x] 2.3 Request a fresh independent read-only review of this slice's term-boundary and eligibility code/specs. Verify every material finding is addressed, approved affected checks are rerun, and a fresh focused re-review covers any resulting fixes.

## PR 3: Recurring payment dates (~85 changed lines)

Depends on: PR 2, using its branch as this slice's base. Estimated split: ~25 implementation lines + ~60 spec lines. Review concern: placing the selected payment day/month inside a known term. The inputs and date expectations must be clear in this slice's tests, without needing the earlier date-rule diff.

- [x] 3.1 Calculate recurring payment dates for day 1 through 28 and Annual payment-month resolution/defaulting. Add day 28, invalid day/month, December-to-January Annual payment, and Monthly-ignores-payment-month examples. Verify dates fall inside the term. Combining these dates with payment amounts and initial collection options belongs to PR 4.
- [x] 3.2 Run the same approved calculator specs, StandardRB on changed Ruby files, and backend typecheck for this slice; obtain approval if not already granted. Verify they pass or record the precise blocker and any unavailable gate; do not count unrun commands as passing.
- [x] 3.3 Request a fresh independent read-only review of this slice's recurring payment-date code/specs. Verify every material finding is addressed, approved affected checks are rerun, and a fresh focused re-review covers any resulting fixes.

## PR 4: Payment rows and initial collection choices (~105 changed lines)

Depends on: PR 3, using its branch as this slice's base. Estimated split: ~25 implementation lines + ~80 spec lines. Review concern: whether to return a payment row, its amount, and whether its initial date moves to today. Keep these payment rules separate from assembling the public first-term call.

- [x] 4.1 Build one regular payment for supported strategies using `quote.amount` and the calculated recurring date. Add `first_payment_today` as a separate initial option and External/Register's empty-payment result. Cover amount reconciliation, unchanged amounts/term dates, and the existing zero-total limitation for Credit card, Bank account, and On account: retain a zero quote but return a payment error and no usable schedule. Verify no deferred Register row, no provider/Retail checkout calls, and no changed Register checkout or free-subscription handling.
- [x] 4.2 Run the same approved calculator specs, StandardRB on changed Ruby files, and backend typecheck for this slice; obtain approval if not already granted. Verify they pass or record the precise blocker and any unavailable gate; do not count unrun commands as passing.
- [x] 4.3 Request a fresh independent read-only review of this slice's payment-row and collection-option code/specs. Verify every material finding is addressed, approved affected checks are rerun, and a fresh focused re-review covers any resulting fixes.

## PR 5: Complete first-term call and no-side-effect checks (~85 changed lines)

Depends on: PR 4, using its branch as this slice's base. Estimated split: ~25 implementation lines + ~60 spec lines. Review concern: the complete caller-facing result and its read-only behavior. Tests show concrete dates, amounts, and errors so the combined behavior can be judged in this diff, rather than relying on readers to open the foundation PRs.

- [x] 5.1 Add `first_term` with the documented arguments, including `now: Time.current`, reusing the completed quote, calendar, and payment portions. Add first-term input/output, unsupported-input, default-clock, and supplied-clock examples. Verify returned fields, amount reconciliation, field/code errors with no usable payments, and the absence of product-ID lookup or existing caller changes.
- [x] 5.2 Add no-side-effect examples for valid and invalid first-term calls, preparing fixtures before observation. Verify no writes across subscription, quote, item, agreement, sale, payment, and transaction tables, no mutation of supplied records, and no provider/Retail checkout/job/notification calls.
- [x] 5.3 Run the same approved calculator specs, StandardRB on changed Ruby files, and backend typecheck for this slice; obtain approval if not already granted. Verify they pass or record the precise blocker and any unavailable gate; do not count unrun commands as passing.
- [x] 5.4 Request a fresh independent read-only review of this slice's complete first-term call and specs. Verify every material finding is addressed, approved affected checks are rerun, and a fresh focused re-review covers any resulting fixes.

## PR 6: Next-term calculation with existing renewal pricing (~180 changed lines)

Depends on: PR 5, using its branch as this slice's base. Estimated split: ~45 implementation lines + ~135 spec lines. Review concern: consecutive terms and preserved renewal prices, reusing the existing quote/calendar/payment rules without changing the renewal caller. Keep price selection with the next-term call, where its effect is directly visible in the result tests.

- [x] 6.1 Implement `next_term(previous_term:, schedule:)` from the previous term's club-local end date plus one day, without reading the current clock, calling renewal, or requiring new schema columns. Extend input/output and unsupported-input examples to this method, and add late Monthly, Annual successor, year-boundary, and initial-override-isolation examples. Verify both methods return the same fields, `next_term` has no clock argument or current-time calculation, and January 2 execution still produces January 1–31 with a January 1 payment, independent of the first term's actual payment date.
- [x] 6.2 Price subsequent-term lines through existing `price_from_source`, using that value for price and original price as current renewal does, and retain current product taxes. Add ordinary repricing, saved bundle price, and changed-tax examples. Verify an ordinary saved price of 80 with current source price 100 uses 100, while a bundled saved price of 80 remains 80.
- [x] 6.3 Add no-side-effect examples for valid and invalid subsequent-term calls, preparing fixtures before observation. Verify no writes across subscription, quote, item, agreement, sale, payment, and transaction tables, no mutation of supplied records, and no provider calls or enqueue/notification effects.
- [x] 6.4 Run the same approved calculator specs, StandardRB on changed Ruby files, and backend typecheck for this slice; obtain approval if not already granted. Verify they pass or record the precise blocker and any unavailable gate; do not count unrun commands as passing.
- [x] 6.5 Request a fresh independent read-only review of this slice's next-term code/spec diff and relevant quote/pricing callers. Verify every material finding is addressed, approved affected checks are rerun, and a fresh focused re-review covers any resulting fixes.
- [x] 6.6 Review the final stack diff and document completed evidence in the existing change artifacts. Verify the expected application diff is limited to the calculator and its spec, current callers remain untouched, and no persistence, endpoint, integration, frontend, feature-flag, or proration implementation slipped in.

Planned commands, to run only after approval:

```bash
bundle exec rspec spec/models/subscription/calculator_spec.rb
bundle exec standardrb app/models/subscription/calculator.rb spec/models/subscription/calculator_spec.rb
bundle exec srb tc
```

No UI workflow or E2E seeder is needed for this read-only calculation ticket. No backend RSpec, build, lint, typecheck, or runtime verification has been run during proposal creation.

## Implementation evidence

Stack, each a draft PR based on the previous branch: [#15664](https://github.com/chronogolf/chronogolf/pull/15664) `GOLF-9894-1-calculator-quote` (base master), [#15665](https://github.com/chronogolf/chronogolf/pull/15665) `-2-calculator-calendar`, [#15666](https://github.com/chronogolf/chronogolf/pull/15666) `-3-calculator-payment-dates`, [#15667](https://github.com/chronogolf/chronogolf/pull/15667) `-4-calculator-payment-rows`, [#15668](https://github.com/chronogolf/chronogolf/pull/15668) `-5-calculator-first-term`, [#15669](https://github.com/chronogolf/chronogolf/pull/15669) `-6-calculator-next-term`.

| PR | Forecast (changed lines) | Actual (impl + spec) | Calculator spec examples |
| --- | --- | --- | --- |
| 1 | ~190 | 199 (44 + 155) | 15 |
| 2 | ~125 | 170 (30 + 140) | 42 |
| 3 | ~85 | 144 (23 + 121) | 61 |
| 4 | ~105 | 133 (11 + 122) | 79 |
| 5 | ~85 | 229 (26 + 204) | 94 |
| 6 | ~180 | 241 (40 + 206) | 112 |

PRs 5 and 6 exceed the 200-line ceiling, almost entirely in spec lines (the shared no-side-effects example, error-aggregation examples, and review follow-ups). The six-slice grouping was kept because the tasks forbid splitting source-price selection from `next_term` and the side-effect checks from `first_term`.

- **Verification:** after every slice the approved commands ran and passed: `bundle exec rspec spec/models/subscription/calculator_spec.rb`, `bundle exec standardrb` on both files, and `bundle exec srb tc`. Each rebased slice was also rerun on its own. The very first rspec run of slice 1 failed every example with a transient `can't quote Hash` in the shared fixture setup and passed on rerun with no change. Running rspec regenerates the tracked `sorbet/rbi/dsl/subscription.rbi`; that drift was restored each time and never committed.
- **Per-slice reviews** (fresh read-only `reviewer`) led to root-cause fixes in slice 1 (a line with an unusable or non-finite price is removed from the unsaved quote and reported, because reading `quote.amount` raised for a nil price and `Infinity` passed validation), a non-December Annual payment example (slice 3), a `price: nil` first-term example and a non-reloading side-effect snapshot (slice 5), and a guard so `next_term` does not raise for a missing product with no agreement (slice 6). Mutation checks confirmed the side-effect example, the renewal-price example and the bundle example fail when their behavior is broken.
- **Final whole-stack review** (fresh `reviewer-opus`): no high findings, verdict ship. Its low findings (POS-reference assertion, tax-inclusive and mixed-rate payment amounts, an unmapped agreement strategy, `before` blocks creating data) were fixed in spec-only commits and a fresh focused re-review found nothing further.
- **Scope:** `git diff master...GOLF-9894-6-calculator-next-term --stat` lists only `app/models/subscription/calculator.rb` and `spec/models/subscription/calculator_spec.rb`. No existing caller, persistence, endpoint, integration, frontend, feature-flag or proration change, and no generated file, is in any commit.
- **Deviation from the plan:** PRs 1 to 4 ship private helpers that nothing calls yet, and their specs call them with `send`, because the tasks forbid public `first_term`/`next_term` stubs before PRs 5 and 6.
- **Accepted limits, not implemented:** prices near the Float maximum can overflow when summed or taxed, and a non-finite catalog `product.price` is not rejected; the extra `payments`/`invalid` error appears after the line error when every line is unquotable; the DST-month scenario is exercised through pure date arithmetic.
- **Open decision:** `next_term` assumes the previous term ended on a month's last day. A legacy term ending mid-month produces a payment date outside the next term with no error. Options are an explicit `previous_term`/`unsupported` error in `next_term`, or a note in [GOLF-10170](https://lightspeedhq.atlassian.net/browse/GOLF-10170) for GOLF-9898 to own.
