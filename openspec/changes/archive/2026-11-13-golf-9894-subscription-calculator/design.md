# Design

## Context

Application and test paths in this design resolve under `/Users/rushipatel/Documents/Repos/chronogolf`. The change artifacts themselves live in the `golf-specs` planning repository.

See `proposal.md` for motivation and `specs/subscription-term-calculation/spec.md` for the behavior contract. This design is included because the clock, pricing, tax, and reusable result boundaries benefit from decisions before implementation.

Relevant existing code:

| Location | Existing behavior | How the calculator will use it |
| --- | --- | --- |
| `app/models/subscription.rb` | Renewal infers duration; its local date helpers use the provider timezone. | Reuse local-date semantics, not legacy duration inference. Do not change this model's existing methods. |
| `app/use_cases/subscription/expire.rb` | Starts renewal at `Time.current`, derives payment day from the first actual payment, and saves/starts/charges. | Do not call this workflow from calculation. Its integration is GOLF-9898. |
| `app/models/subscription/line.rb` | `price_from_source` uses the current product price except for bundle lines, which retain saved price. | Reuse this exact pricing rule for `next_term`. |
| `app/models/sale/quote.rb` | `add_subscription_product` already builds unsaved quote lines and sale items using product taxes and POS references. | Reuse this behavior unchanged to calculate quote totals. Do not save the preview objects or snapshot their taxes. |
| `app/models/concerns/taxable_lines.rb` | Calculates base amount, subtotal, tax, and total using current inclusive/exclusive and harmonized conventions. | Delegate quote arithmetic rather than write a new tax implementation. |
| `app/models/sale/payment_strategy/base.rb` and `monthly.rb` | Payment `amount` is checked against quote base amount; actual collection uses the resulting sale's tax-inclusive amount. | Do not put tax-inclusive display totals into the payment base amount unconditionally. |
| `spec/use_cases/sale/calculate_quote_price_spec.rb`, `spec/models/subscription/base_spec.rb`, `spec/models/sale/agreement/template_spec.rb` | Existing expectations cover current quote taxes and source-price renewal. | Use these as behavior references; do not refactor or rewrite them. |

## Goals / Non-Goals

**Goals:**

- Keep a small calculation object and result type, with one internal calendar/quote path shared by the two entry points.
- Allow the subsequent-term calculation to be tested before schedule columns exist by accepting the schedule explicitly.
- Preserve current business behavior outside the explicitly declared calendar schedule and initial due-date rules.

**Non-Goals:**

- Do not invoke factories, checkout, agreement templates, renewal, activation, payment strategy setup, or payment providers merely to obtain a preview.
- Do not select a payment method or integration policy, implement sale validation, or serialize an HTTP response here. Those belong to later callers.
- Do not modify existing helpers to support the new calculator, add a contract abstraction, or add proration extension points speculatively.

## Decisions

### 1. What the two calculator methods take and return

Implement `Subscription::Calculator` in `app/models/subscription/calculator.rb`, with a small `Result` type colocated within the class. It is not an Active Record model. A plain calculation object is sufficient because this ticket neither saves records nor coordinates payment execution; an interactor would add an orchestration layer without work to orchestrate.

Use two methods because the initial and subsequent terms have different date rules, not because the TDD lists two methods:

```ruby
Subscription::Calculator.first_term(
  club:,
  lines:,
  schedule:,
  strategy:,
  start_month: nil,
  first_payment_today: false,
  now: Time.current
)

Subscription::Calculator.next_term(previous_term:, schedule:)
```

`first_term` needs the club for timezone and quote behavior, the product-and-price lines to quote, and the selected existing strategy to decide whether payment rows are needed. `start_month` and `first_payment_today` are initial choices, separate from the recurring `schedule`. `now` defaults to the current timestamp, while tests can supply a timestamp to verify timezone boundaries.

`next_term` gets the club, product lines, and existing agreement strategy from `previous_term`. Its start is the local day after that term ends, so it has no `now` or `today` argument. Accepting `schedule:` explicitly lets this ticket proceed before GOLF-9896 adds the corresponding columns and makes the recurring values visible to the caller.

The common result contains only `start_date`, `end_date`, `payments`, `quote`, and `errors`. Read subtotal, tax, and total from the fresh unsaved quote; do not store duplicate totals on the result. The later preview endpoint will decide its HTTP response shape. The existing quote serializer exposes totals only, so per-product display will need that caller to read and map the quote lines explicitly.

Payment rows contain `amount` and `due_date`; dates are `Date` values and errors are field/code pairs. Keep amounts in the number types produced by the existing quote calculation. Do not add a final conversion to Decimal merely because the TDD labels its amounts Decimal. A result with errors has no usable payment schedule; its quote can remain available to explain a pricing or payment error.

Alternative rejected: using the existing create/renewal flow with rollback to preview results. Those flows can activate memberships, charge externally, and publish notifications; rolling back is not a read-only calculation.

### 2. Use resolved products and separate recurring values from initial choices

For `first_term`, `lines:` is an array of already resolved product-and-price objects with the existing `Subscription::Request::Line` shape: `product`, `price`, and optional bundle identity fields. The caller supplies the effective price, using its override when present or the current product price otherwise. Zero must stay zero. Reuse the existing line shape rather than introducing another public line type.

The calculator does not receive raw product IDs, parse HTTP JSON, or call `Subscription::Request#lines` to resolve products. The future endpoint owns allowlisted product lookup, tenant authorization, and mapping request fields to these inputs. Do not copy that existing request method's unrestricted `constantize`. The calculator still checks for missing/unsupported supplied products and invalid prices.

Use `schedule:` for only `billing_cycle` (`monthly` or `annual`), `payment_day` (1 through 28), and `payment_month` (Annual only), matching GOLF-9896. A missing Annual payment month defaults to the term's start month; Monthly ignores it. Pass `start_month` separately as the first-term year-month selection, defaulting to the first eligible month when omitted.

Pass `first_payment_today` separately, defaulting to false. It changes only the initial due date and is not persisted as part of the recurring schedule. The next-term method does not accept it or inspect the previous payment's actual due date.

The calculator accepts an already selected existing strategy (`monthly`, `installment`, or `external`), using existing strategy mappings rather than introducing a second payment policy. External returns no payments; the initial collection option has no scheduled row to move. Strategy selection, including Vend handling, remains with the later caller and existing behavior.

Alternative rejected: storing an overridden initial payment date as the recurring schedule. That would preserve the current date-drift problem rather than establish the declared schedule.

### 3. Determine today's date in the club timezone

Only `first_term` reads the clock. Convert its `now` timestamp to the club timezone once before taking the local date. Use a timestamp input consistently rather than accepting both timestamps and plain dates under a parameter called `today`. Do not use server `Date.current` or ambient `Time.zone` for eligibility.

First-term calculation determines the earliest eligible month, defaults an omitted selection, and rejects an earlier selection. Parse supplied `start_month` strictly as a valid `YYYY-MM`. Use month/year calendar arithmetic to derive the term end and the Annual payment date. The selected Annual month belongs to the start year when it is on or after the start month, and the following year otherwise.

Next-term calculation starts from `previous_term.end_date_in_club_zone + 1.day`. It does not read the current clock, gate that start against today, or shift an overdue payment. Do not call `Subscription#renewal` or construct a successor subscription to obtain these dates.

Alternative rejected: elapsed seconds or copying the worker's timestamp. Both conflate runtime with the intended calendar period.

### 4. Reuse current pricing and tax calculations without writes

For first terms, quote the effective price on each resolved input line and use the product's current price as its original price, matching current quote construction. Keep zero-valued product prices valid; do not silently replace zero with the catalog price.

For subsequent terms, use each existing line's `price_from_source` as its price and original price, matching `Subscription#renewal`. This deliberately preserves saved bundle prices even when standalone catalog prices change. Do not fetch a different bundle price, infer whether its saved value was a discount, or change ordinary-product repricing.

Build a fresh `Sale::Quote` and call its existing subscription-product quote construction for each resolved line. Validate quote/line inputs without `save`, `create`, tax freezing, or agreement/payment construction. Preserve its arithmetic, number types, product taxes, and rounding. Set the regular payment base amount to `quote.amount`; callers read `quote.amount_without_tax`, `quote.amount_tax`, and `quote.amount_with_tax` for subtotal, tax, and total.

Keep the existing distinction between a free product and a free payment. A zero-priced product alongside a positively priced product produces a positive quote and payment. If the entire quote is zero and the selected strategy requires positive payments, retain the zero quote but return a payment validation error and no usable payment schedule. This reflects the current creation outcome: the browser produces no payment rows for zero total, the scheduled-payment strategies require rows, and the payment model rejects `amount: 0`. Do not create or save a payment to check that rule.

External/Register still returns no scheduled payment rows. The calculator only supplies dates and an unsaved quote; it does not open Retail checkout. When staff actually sells through Register, the existing sale and Retail checkout flow must remain unchanged, including its conditions for opening the POS sale and its activation/completion behavior. Preserving that workflow when new schedules are integrated belongs to GOLF-9900 and the related form ticket, not new checkout code in GOLF-9894. Do not infer new Register acceptance rules from a validation that applies to Credit card, Bank account, and On account payment records.

Alternative rejected: summing tax-inclusive totals into `payments[].amount` for all clubs, or implementing custom tax arithmetic to match illustrative TDD JSON. Either changes current payment semantics.

### 5. Keep error reporting local and structured

Validate cycle, strict year-month selection, day/month bounds, supported strategy/product inputs, quote prices, and positive scheduled-payment amounts before returning a usable schedule. Return stable codes with fields such as `billing_cycle`, `start_month`, `payment_day`, `payment_month`, `strategy`, `payments`, `lines[index].price`, and `lines[index].product`. The future endpoint can map these to its request/UI fields. Do not expose translated exception text as the contract or rescue unrelated infrastructure failures as invalid user input.

Suggested local codes are `invalid`, `required`, `before_first_eligible_month`, `unsupported`, and `not_found`; exact constant spelling can stay within the calculator file. This change does not add six-locale messages, HTTP statuses, authorization rules, or endpoint validation.

Alternative rejected: retaining arbitrary client-supplied dates or silently clamping unsupported days. The declared calculation needs an explicit error result while leaving legacy validation untouched.

### 6. Where the TDD or prototype needs correction

The TDD gives the wider direction, not a mandatory Ruby interface or an exact description of every current rule. Keep a detail only when it follows from the feature's required behavior and the existing code.

| Detail to reconsider | Decision and reason |
| --- | --- |
| TDD gives both methods `settings:` and `today:`. | Use recurring `schedule:` and separate initial choices. Only the first term needs the clock; the next term must not depend on when a worker runs. |
| TDD lists separate subtotal/tax/total values, and the earlier plan also returned the quote. | Return the quote once and read its totals, rather than storing the same pricing information twice. |
| TDD labels monetary outputs Decimal. | Reuse existing quote arithmetic, number types, and rounding instead of imposing a conversion unrelated to the requested behavior. |
| The earlier plan treated raw request fields as calculator inputs. | Accept resolved product-and-price lines. HTTP parsing, product lookup, and tenant authorization belong to the future endpoint. |
| TDD says current prices and first-term-only overrides; current bundle lines retain saved prices. | User explicitly confirmed preserving current behavior. Use `price_from_source`, including its bundle exception. |
| TDD Monthly sample places a tax-inclusive total in payment `amount`; existing quote/payment conventions distinguish base amount and amount collected. | The requested scope explicitly preserves current tax conventions. Treat sample values as illustrative, not a change to payment semantics. |
| Prototype offers Last day, Monthly/Annual splits, future editing, and other later controls. | GOLF-9894 limits Q3 to days 1 through 28 and one regular payment. Those controls are excluded, not inferred requirements. |
| TDD refers to a D3 start-month range without supplying a corresponding rule in the extracted text. | Use the ticket's explicit club-local first-eligible-month rule. The prototype's three-year selector is a UI range, not a new backend maximum. |
| TDD's future-contract appendix recommends a contract in its retained design. | The main Q3 design and GOLF-9896 explicitly choose schedule-on-term. No contract work belongs here. |

The supplied prototype was inspected statically. Its zero-tax demo totals and simulated Paid state are not evidence of production tax or payment-provider behavior.

### 7. Deferred edge cases

Maintain the running list in [GOLF-10170 — Edge cases](https://lightspeedhq.atlassian.net/browse/GOLF-10170), under the subscriptions epic. Add a bullet when another case is deliberately deferred; a listed case is not automatically a blocker or an implementation requirement.

- **Every product is $0 with Credit card, Bank account, or On account:** preserve current creation rejection. A zero quote can be calculated, but it does not supply a positive payment. A free product within a positive-total subscription remains valid. Whether to support zero-total creation later is a follow-up decision, not work in GOLF-9894. Register uses a different collection flow and is not included in this rejection.

## Risks / Trade-offs

- [Shared quote behavior differs by tax mode] -> Compare calculator results against real unsaved quotes for inclusive/exclusive, product-specific rates, and existing rounding; do not use only a tax-free fixture.
- [New code accidentally changes input records or saves an associated object] -> Assert unchanged supplied records and no database writes during both successful and invalid calculation, after fixtures have been prepared. Cover quote lines and sale items as well as subscriptions/payments.
- [A date-based test masks timezone errors] -> Include a timestamp where UTC is November 2 while the club is November 1, plus leap-year and daylight-saving-month scenarios.
- [Over-generalization adds future work] -> Keep one calculator file and one spec file as the expected diff. No proration model, split-payment abstraction, public API, or integration changes.
- [Calculator starts duplicating HTTP request handling] -> Accept resolved lines and check only the supplied calculation inputs. Leave product ID lookup and tenant authorization with the future endpoint.

## Migration Plan

No schema or data migration. The new calculator can be deployed without activating any new workflow because no caller is changed in this ticket. Rollback removes an unused calculation capability; existing Custom and WooCommerce behavior is unaffected. Later integration tickets own enablement and deployment ordering.

Before implementation is published, obtain approval for focused verification commands, perform an independent read-only code review, and address material findings. No specs, builds, linters, or runtime checks have been run as part of this planning change.
