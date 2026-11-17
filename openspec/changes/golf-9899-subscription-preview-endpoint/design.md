# Design

## Context

Application and test paths in this design resolve under `/Users/rushipatel/Documents/Repos/chronogolf-GOLF-9899-openspec`. The change artifacts live in the `golf-specs` planning repository. See `proposal.md` for motivation and `specs/subscription-preview/spec.md` for the behavior contract. `specs/subscription-term-calculation/spec.md` (already archived) covers the calculation this endpoint calls.

What exists today:

| Location | Existing behavior | How the endpoint uses it |
| --- | --- | --- |
| `app/models/subscription/calculator.rb` | `first_term(club:, lines:, schedule:, strategy:, start_month:, first_payment_today:, now:)` returns dates, at most one payment row, an unsaved `Sale::Quote`, and `{field, code}` errors. It does not look up products, check tenancy, or pick a strategy. | Called once per request. This change adds nothing to it. |
| `app/controllers/private_api/clubs/subscriptions_controller.rb` | `create` builds a `Subscription::Request` and calls `authorize! :create, request`. | Add a `preview` action beside it that authorizes the same way. |
| `app/models/abilities/employee/{manager,assistant}.rb` | `can :create, Subscription::Request` for managers by club, and for assistants only if `sell_subscriptions` is permitted at `request.provider`. | Reused as is, so preview and creation cannot disagree on who is allowed. |
| `app/interactors/subscription/create.rb` | Product must be `part_of_mco_or_club?` of the club. Customer is loaded with `User.find` and not checked against the club. Strategy comes from the client. | Preview applies the same product rule and a stricter customer rule (see decision 3). |
| `app/interactors/sale/calculate_quote_price.rb` and `PrivateAPI::Sales::QuotesController` | The existing quote endpoint. Product types are allow-listed to `Membership::Plan` and `Product::Service` before `constantize`. Errors use the `{type, message}` shape. | Reuse the allow-list. The error shape is not reused (decision 5). |
| `frontend/legacy/apps/admin/services/helpers/payment-strategy.helper.ts` | The browser picks the strategy: credit card, bank account, and on account use Monthly (Installment when the club has Vend POS, a legacy integration), Register uses External. | Moves server-side for declared schedules (decision 4). |
| `app/models/sale/quote.rb`, `app/models/concerns/taxable_lines.rb` | Quote totals use the club's inclusive/exclusive and harmonized tax settings. Unharmonized clubs compute tax per rate on the sum of lines, not per line. | Totals are read from the quote. Per-line rows are read from the quote's lines (decision 6). |

## Goals / Non-Goals

**Goals:**

- One thin HTTP layer over the calculator: authorize, resolve the request into calculator inputs, call it, shape the response.
- A response the form can merge into its create request with no remapping, and field errors it can show next to inputs.
- No write and no payment-provider call on any path.

**Non-Goals:**

- Changing the calculator, abilities, the create path, or the existing quote endpoint.
- Persisting a schedule, accepting a schedule in create, or any feature-flag gate (GOLF-9896, GOLF-9900).
- Proration fields (GOLF-9901) and the form (GOLF-9902).
- Checking that a product is active or that the customer has an overlapping membership. Creation keeps owning those rules (decision 7).

## Decisions

### 1. Where the code lives

Add `preview` to `PrivateAPI::Clubs::SubscriptionsController` with a collection route `post :preview` under the existing `resources :subscriptions, only: %i[index create]` in `config/routes/private_api/clubs.rb`. Put the resolving and calling in a new interactor, `Subscription::Preview` in `app/interactors/subscription/preview.rb`, following the `context` pattern of `Subscription::Create` that the repo asks new business logic to use. A controller-only implementation would mix parameter parsing, tenancy checks, and response shaping in an already-used file, and the calculator is deliberately not the place for HTTP concerns.

The controller authorizes, calls the interactor, and renders. Authorization builds a `Subscription::Request` for the club with placeholder dates, because the request's strict attributes reject nil dates and the abilities read only the provider. The interactor returns either the calculator result plus the resolved schedule, or a list of errors.

Alternative rejected: a new `PreviewsController`. It would duplicate the club lookup and error handling for one action, and the route reads naturally as a collection action of subscriptions.

### 2. Request shape

The request follows the TDD's sample, with two deliberate changes:

```json
{
  "subscription": {
    "customer_id": 123,
    "lines": [
      { "subscription_product": { "type": "Membership::Plan", "id": 45 }, "price": "149.99" }
    ],
    "schedule": {
      "billing_cycle": "monthly",
      "start_month": "2026-12",
      "payment_day": 1,
      "payment_month": null,
      "first_payment_today": false
    },
    "sale_agreement": { "payment_setup": { "method_name": "credit_card" } }
  }
}
```

- `lines` has the shape of a create request's lines, so the form can send the same array to both. Bundle fields on a line and `auto_renew` do not change dates or amounts; they are not read here. The controller hands the interactor only `customer_id`, `lines`, `schedule`, and `sale_agreement`, and the interactor reads only the documented keys. Whether auto-renew suits the payment method stays a creation check.
- `schedule.first_payment_today` is the "Take payment today" option. The TDD sample omits it, but the calculator takes it separately from the recurring schedule and GOLF-9902 needs it. It is read from the request only; it is not echoed back and creation does not persist it.
- **Change from the TDD:** the request carries `payment_setup.method_name`, not `strategy_name` (decision 4).
- Numeric schedule fields must be JSON integers. The calculator treats anything else as `invalid`, and the interactor does not coerce strings.
- `start_month` and `payment_month` may be omitted or null: the first defaults to the first eligible month, the second to the start month (Annual).

Missing or non-object `subscription` is a `400` (`params.require`, then `head :bad_request` for a non-object value; no exception is raised by hand). Missing individual fields are `422` field errors. The controller does not filter values with `permit`: `permit` would silently drop a wrongly shaped value (such as a price sent as an object) and the preview would quietly use a default. The interactor validates shapes and reports them as field errors. JSON requests are also wrapped by Rails under the `subscription` key, so an empty body arrives as an empty object and is treated as missing.

### 3. Resolving customer, products, and prices

The calculator wants resolved products and prices and leaves lookup and tenancy to the caller. The interactor therefore:

- **Customer:** requires `customer_id` and an active customership of that user at the club (`club.active_customerships`). Otherwise `customer_id` is `required` or `not_found`. This is stricter than creation, which loads the user without checking the club. The Add form only offers the club's customers, so valid requests are unaffected, and preview should not confirm that arbitrary user IDs exist.
- **Products:** allow-list the type before loading anything, using the calculator's own `SUPPORTED_PRODUCT_TYPES` (`Membership::Plan` and `Product::Service`) rather than a second list, and compare class names so no input is ever constantized (an unknown type is `unsupported`, an unknown ID is `not_found`). Then require `organization.part_of_mco_or_club?(club.id)`, the rule `Subscription::Create` applies, and report a failure as `not_found`. A product that exists but belongs elsewhere must look the same as one that does not exist.
- **Prices:** a supplied price must be a non-negative, finite number; otherwise `lines[i].price` is `invalid`. An absent price means the product's current price. Zero stays zero. The original value is passed through so the quote receives what creation would, and the calculator's own price check still applies.

The interactor collects errors across customer, products, and schedule and returns them together. It always calls the calculator with the lines that resolved, so schedule errors are reported alongside line and customer errors. The calculator indexes only the lines it receives, so the interactor remembers each line's position in the request and maps calculator errors back to it. Errors that are only fallout are dropped: "lines required" when every line was already rejected, payment-row errors built from a partial quote, and strategy errors when the payment method was already rejected.

### 4. Choose the strategy on the server

TDD requirement 7 and GOLF-9900 say the server picks the strategy from the payment method, with today's rule. The TDD's sample preview request sends `strategy_name`, which would let the client decide, and a client could then ask for a deferred Register collection. The preview therefore reads `method_name` and selects:

| Method | Strategy |
| --- | --- |
| `credit_card`, `bank_account`, `on_account` | `monthly` |
| `register` | `external` (no scheduled rows) |
| missing | `method_name` is `required` |
| anything else, including `ecommerce` | `method_name` is `unsupported` |

If the request also sends `sale_agreement.payment_setup.strategy_name` and it differs from the selected strategy, the error is `strategy_name` `unsupported`. This is how "reject deferred Register collection" is expressed: `register` with `monthly` is rejected, and so is a card with `external`. Sending a matching strategy is harmless, which lets a form that already sends one keep working. Separate proration payments do not exist yet (GOLF-9901), so no extra rule is needed here.

Vend POS is deprecated (confirmed by the ticket owner), so there is no `installment` branch. The TDD's "Installment for Vend clubs" is intentionally not carried over, and GOLF-9900 should not add it either. A club that still has a Vend integration gets `monthly`.

The rule is a constant, `Sale::PaymentSetup::DECLARED_SUBSCRIPTION_STRATEGY_MAPPING`, next to the existing method and strategy mappings it relates, not in the interactor. The calculator is not the home: GOLF-9894 keeps strategy selection with the caller. GOLF-9900 reads the same constant.

### 5. Error shape

Errors are `422` with `{"errors": [{"field": "schedule.start_month", "code": "before_first_eligible_month"}]}`. The existing private API error shape (`FormattedServiceMessages`) carries only `type` and `message`, and the form needs to attach an error to an input and branch on its code, so reusing it would force the form to parse text. This is a deliberate local shape, not a new global convention, and it is not translated: the form owns its wording. Authorization and not-found at club level keep the existing `403` and `404` handling.

`field` is the path relative to the `subscription` object. Calculator fields map as follows:

| Calculator field | Request field |
| --- | --- |
| `billing_cycle`, `start_month`, `payment_day`, `payment_month` | `schedule.<same name>` |
| `lines`, `lines[i].price` | unchanged |
| `lines[i].product` | `lines[i].subscription_product` |
| `payments` | `sale_agreement.payment_setup.payments_attributes` |
| `strategy` | `sale_agreement.payment_setup.strategy_name` |

Codes are the calculator's (`required`, `invalid`, `unsupported`, `not_found`, `before_first_eligible_month`) plus the interactor's own, which reuse the same words. Because strategy is chosen on the server, `strategy` should not occur from the calculator; the mapping is there so it is not lost if it does. A zero total with a scheduled method comes back as `payments` `invalid`, the existing limitation recorded in GOLF-10170, and is not changed here.

Quote validity: `Sale::Quote` rejects a mix of tax-inclusive and tax-exclusive lines, which creation would hit when it saves the quote. The calculator does not check this. The interactor runs the quote's own validation and reports a failure as `lines` `invalid`, so the preview cannot succeed where creation is certain to fail.

### 6. Response shape and the per-product breakdown

```json
{
  "subscription": {
    "start_date": "2026-12-01",
    "end_date": "2026-12-31",
    "schedule": { "billing_cycle": "monthly", "payment_day": 1, "payment_month": null },
    "sale_agreement": {
      "payment_setup": { "payments_attributes": [{ "amount": "149.99", "due_date": "2026-12-01" }] }
    }
  },
  "quote": {
    "amount_without_tax": "130.43",
    "amount_tax": "19.56",
    "amount_with_tax": "149.99",
    "lines": [
      {
        "subscription_product": { "type": "Membership::Plan", "id": 45 },
        "name": "Gold membership",
        "price": "149.99",
        "amount_without_tax": "130.43",
        "amount_tax": "19.56",
        "amount_with_tax": "149.99"
      }
    ]
  }
}
```

(The amounts are illustrative; the JSON type of each amount is whatever the existing quote serializer produces today, since the calculator keeps the quote's number types.)

- The `subscription` object has the same keys the create request uses, so the form can deep-merge it. The returned `schedule` leaves out `start_month` and `first_payment_today`, which are inputs rather than stored fields. For Annual, `payment_month` is the resolved month (the request's value, or the start month), so creation stores a concrete month; for Monthly it is `null`.
- The per-product rows are under `quote`, not `subscription`. Arrays replace rather than merge, so putting `lines` inside `subscription` would overwrite the form's own lines when it merges. The TDD says `quote` is not sent, which makes it a safe home.
- Each product row comes from the calculator's unsaved quote line: the line's name, price, and its `amount_without_tax`, `amount_tax`, and `amount_with_tax`. The totals come from the quote itself, not from summing rows, because the club's unharmonized tax mode computes tax per rate on the sum of lines, so row taxes can differ from the quote's tax by a cent. The rows are for display and the quote totals are the amounts that creation will use. This matches GOLF-9894's decision to read totals from the quote.
- Payment amounts are the calculator's `quote.amount`, which under existing conventions is what creation validates payments against. GOLF-9894's design already records that the TDD sample's tax-inclusive payment amount is illustrative. Preview does not recompute it.
- Rows are matched to request lines by the quote's line order, which is the order the interactor added them.

The response is rendered by `Subscription::PreviewSerializer`, with `Subscription::PreviewLineSerializer` for each product row, instead of a hash built in the controller. They reuse `Sale::QuoteSerializer` for the totals and do not change it, since its other callers expect totals only.

### 7. What the preview deliberately does not validate

`Membership::Plan#validate_subscription` also checks that the plan is active at the start date and that the customer has no overlapping membership. Both need a built `Subscription` for the customer, with dates and lines. Running them would turn a read-only calculation into a partial run of the create path and would make the preview's result depend on the customer's other memberships. Creation keeps these checks and still rejects a conflicting request. The cost is that a preview can succeed where creation later fails for one of them. That is already true of today's Custom flow, so it is recorded in GOLF-10170 rather than closed here.

### 8. Clock and time zone

The interactor calls `first_term` with the calculator's default `now: Time.current`. The calculator converts it to the club's time zone before taking the date, so the browser's and server's UTC dates are never used. Specs use `travel_to` with timestamps where UTC and the club's date differ, including November 1 vs November 2.

## Risks / Trade-offs

- [Preview succeeds but creation fails for overlap or inactive product] → Documented as decision 7; the form already handles a creation error today.
- [Per-line tax rows differ from quote tax by a cent in unharmonized clubs] → Totals are authoritative and rows are display-only; a spec covers a multi-line unharmonized club to pin the behavior down.
- [Request fields drift from creation] → The request keeps the create request's names for `customer_id`, `lines`, and `sale_agreement.payment_setup`. GOLF-9900 and GOLF-9902 should review the contract below before either builds on it.
- [Latency] → One active-customership lookup, one lookup per product, and the existing quote tax reads; no new eager loading. The TDD requires p95 at or below `POST /private_api/sales/quotes`. This is measured on staging when the form exists; no benchmark is part of this change.
- [Preview reveals product or price existence across clubs] → Products outside the club or its management company return `not_found`, the same as a missing one.
- [Retail POS reads] → Building quote lines already reads tax and POS references. These are the reads the existing quote endpoint makes. They are not payment-provider calls and are not blocked.

## Migration Plan

No schema or data migration and no flag. The endpoint is additive and has no caller until GOLF-9902, so it can deploy ahead of the form. Rollback removes an unused route.

Before implementation is published, obtain approval for focused verification commands, perform an independent read-only code review, and address material findings. No specs, builds, linters, or runtime checks have been run as part of this planning change.

## Open Questions

- Confirm with GOLF-9900 that creation reads the same `schedule` keys, and that `first_payment_today` is accepted there or sent only to preview. This does not change the preview's behavior.
