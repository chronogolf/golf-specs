# Tasks

Implement only the preview endpoint and its specs. Read `design.md` and `specs/subscription-preview/spec.md` before applying. Reuse `Subscription::Calculator`, the existing abilities, and the existing quote and tax behavior unchanged. Do not touch the create path, the calculator, abilities, the quote endpoint, or anything in `frontend/`. Deferred cases are tracked in [GOLF-10170 — Edge cases](https://lightspeedhq.atlassian.net/browse/GOLF-10170); they are not additional requirements. Verification commands require explicit approval before execution; this plan does not authorize them.

Estimates include implementation and specs, counting additions and deletions. They are forecasts, not measured diffs. About 200 lines is a ceiling, not a target. Check the split before writing code and move boundaries rather than dropping tests or adding abstractions to meet a number.

The four slices separate four concerns: who and what the request refers to (PR 1), what is calculated and how failures are reported (PR 2), the HTTP surface and permission (PR 3), and the per-product display rows (PR 4). PRs 1 and 2 add one interactor, `Subscription::Preview`, with no caller until PR 3; each is judged by its own specs. Every slice leaves master green and changes no existing caller.

## PR 1: Resolve customer, products, and prices (~155 planned, 204 actual changed lines)

Depends on: master. Estimated split: ~45 implementation lines + ~110 spec lines. Review concern: turning request values into the resolved, tenant-checked inputs the calculator expects, and which input errors are reported.

- [x] 1.1 Before any implementation, check the four line estimates against the planned code and spec setup, confirm each slice is meaningful on its own, and revisit the grouping if not. Verify every heading shows an estimate and that no slice needs another slice's diff to be understood.
- [x] 1.2 Add `Subscription::Preview` in `app/interactors/subscription/preview.rb` with the first step only: resolve `customer_id` and `lines` into a customer and `Subscription::Request::Line` values, collecting `{field, code}` errors. Require an active customership of the customer at the club (`club.active_customerships`). Allow-list product types before loading, require `part_of_mco_or_club?`, report foreign products as `not_found`, and use the product's current price when none is supplied. Price validation is left to the calculator. Verify with specs for: customer of another club, customer with an archived customership, missing customer, unknown/foreign/unsupported product, management company product accepted, override price, zero price, and several errors reported together with the right `lines[i]` indexes.
- [x] 1.3 Verify no record is written and no existing caller is edited, and that an unknown product type is never passed to `constantize`.
- [x] 1.4 Obtain approval for the verification commands below, then run the interactor spec, StandardRB on changed Ruby files, and the backend typecheck. Verify they pass or record the precise blocker and any unavailable gate; do not count unrun commands as passing.
- [x] 1.5 Request a fresh independent read-only review of this slice's code and specs, including the customer and product tenancy rules. Verify every material finding is addressed, approved affected checks are rerun, and a fresh focused re-review covers any resulting fixes.

## PR 2: Strategy selection, calculation, and error mapping (~170 planned, 296 actual changed lines)

Depends on: PR 1, using its branch as this slice's base. Estimated split: ~60 implementation lines + ~110 spec lines. Review concern: what is sent to the calculator, how the payment strategy is chosen on the server, and how calculator errors become request fields.

- [x] 2.1 Extend the interactor to read `method_name` and the optional `strategy_name` and select the strategy: `monthly` for credit card, bank account, and on account, `external` for register. Report a missing method as `required`, an unsupported one as `unsupported`, and a differing `strategy_name` as `unsupported`. Verify with specs for each method, a matching strategy name accepted, register with `monthly` rejected, and a card with `external` rejected.
- [x] 2.2 Read `billing_cycle`, `start_month`, `payment_day`, `payment_month`, and `first_payment_today` from the schedule and call `Subscription::Calculator.first_term` with the resolved lines, the selected strategy, and the default clock. Return the calculator result, the resolved schedule (Annual payment month defaulting to the start month, `null` for Monthly), and the errors. Verify with specs for Monthly, Annual, omitted start month, December start with a January payment in the next year, and first-payment-today changing only the due date.
- [x] 2.3 Call the calculator only when the lines and the payment method resolved, so its error indexes equal the request's, and translate its errors to request paths as `design.md` describes (a `schedule.` prefix, and `payments` to its full path). Verify specs for each, for negative, non-numeric, and non-finite prices reported by the calculator, that a rejected product or payment method reports only its own errors, and that a bad price next to a bad schedule reports both without a misleading payments error.
- [x] 2.4 Add club-local clock examples using `travel_to`: November 1 vs November 2, and a club where local date differs from UTC. Verify the earliest-month rejection returns `before_first_eligible_month` with no payment rows.
- [x] 2.5 Obtain approval if not already granted, then run the same approved commands for this slice. Verify they pass or record the precise blocker and any unavailable gate.
- [x] 2.6 Request a fresh independent read-only review of this slice's code and specs, including the strategy selection and error mapping. Verify every material finding is addressed, approved affected checks are rerun, and a fresh focused re-review covers any resulting fixes.

## PR 3: Route, permission, and response (~130 planned, 191 actual changed lines)

Depends on: PR 2, using its branch as this slice's base. Estimated split: ~30 implementation lines + ~100 spec lines. Review concern: the HTTP surface: who may call it, status codes, and the response shape the form will merge.

- [x] 3.1 Add the `preview` collection route and a `preview` action to `PrivateAPI::Clubs::SubscriptionsController`. Authorize with `authorize! :create` on a `Subscription::Request` for the club, permit only the documented parameters with `params.require(:subscription).permit(...)`, call the interactor, and render `200` or `422 {errors: [{field, code}]}`. Verify the club-level `403`, `404`, and `401` come from the existing handling. If building a request with only a provider fails under its Virtus attributes, record that and use the smallest alternative that reuses the same abilities rather than adding a new permission.
- [x] 3.2 Render the success body described in `design.md` with `subscription` (dates, schedule, payment rows) and `quote` (totals only for now). Do not change `Sale::QuoteSerializer`. Verify with request specs that the shape matches the spec, that the response has no per-product rows yet, and that amounts are serialized like the existing quote endpoint's.
- [x] 3.3 Add permission examples: a manager of the club, an assistant with and without `sell_subscriptions`, an employee of another club, an unauthenticated request, and a body with no `subscription` (`400`). Verify the responses are `200`, `403`, or `401`, with no term or quote data in the denied cases.
- [x] 3.4 Obtain approval if not already granted, then run the controller spec and the same approved commands for this slice. Verify they pass or record the precise blocker and any unavailable gate.
- [x] 3.5 Request a fresh independent read-only review of this slice's code and specs, including the authorization rule and parameter handling. Verify every material finding is addressed, approved affected checks are rerun, and a fresh focused re-review covers any resulting fixes.

## PR 4: Per-product rows, quote validity, and read-only proof (~120 planned, 140 actual changed lines)

Depends on: PR 3, using its branch as this slice's base. Estimated split: ~30 implementation lines + ~90 spec lines. Review concern: what each product row shows, how it relates to the quote totals under the club's tax settings, and proof that previewing changes nothing.

- [x] 4.1 Add `quote.lines` to the response: one row per request line, in request order, with the product reference, name, price, and the line's amounts before tax, tax, and after tax, read from the calculator's unsaved quote lines. Keep the quote totals read from the quote itself. Verify with specs for a tax-exclusive club, a tax-inclusive club, a harmonized club, and an unharmonized club with two lines where row taxes may differ from the quote tax by a cent while the totals match the quote.
- [x] 4.2 Run the quote's own validation and report a failure (mixed inclusive and exclusive taxes) as `lines` `invalid`. Verify with a spec that creation's quote rule and the preview agree.
- [x] 4.3 Add no-side-effect examples for valid and rejected previews, preparing fixtures before observation. Verify no rows are created in subscriptions, quotes, quote lines, sale items, agreements, sales, payments, or transactions, and no payment-provider or POS checkout call is made.
- [x] 4.4 Obtain approval if not already granted, then run the same approved commands for this slice. Verify they pass or record the precise blocker and any unavailable gate.
- [x] 4.5 Request a fresh independent read-only review of this slice's code and specs. Verify every material finding is addressed, approved affected checks are rerun, and a fresh focused re-review covers any resulting fixes.
- [x] 4.6 Review the final stack diff against the proposal, design, and specs, and document completed evidence in the existing change artifacts. Verify the application diff is limited to the route, controller action, the interactor, and their specs; the calculator, abilities, create path, and frontend are untouched; and no flag, migration, locale, or proration work slipped in.

Implementation note: actual sizes exceeded the forecasts because the specs, which cover tax modes, error cases, and permissions, were larger than planned. The implementation itself stayed near the forecast. After review the interactor was simplified: no shape guards beyond `permit`, no price validation of its own, and no line remapping. Approved commands were run for every slice. The Vend branch was dropped and an active customership is required, as decided with the ticket owner.

Planned commands, to run only after approval:

```bash
bundle exec rspec spec/interactors/subscription/preview_spec.rb spec/controllers/private_api/clubs/subscriptions_preview_spec.rb
bundle exec standardrb app/interactors/subscription/preview.rb app/controllers/private_api/clubs/subscriptions_controller.rb config/routes/private_api/clubs.rb spec/interactors/subscription/preview_spec.rb spec/controllers/private_api/clubs/subscriptions_preview_spec.rb
bundle exec srb tc
```

The preview controller examples live in their own file, `subscriptions_preview_spec.rb`, next to the existing subscriptions controller spec.
