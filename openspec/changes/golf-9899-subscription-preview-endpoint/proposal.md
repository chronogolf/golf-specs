# Proposal

## Why

Staff creating a Monthly or Annual subscription need to see exactly what will be created (term dates, payment dates and amounts, per-product amounts, and totals) before they submit, from the same backend calculation that creation and renewal will use. [GOLF-9894](https://lightspeedhq.atlassian.net/browse/GOLF-9894) delivered that calculation, but nothing exposes it over HTTP yet. This change adds the read-only endpoint the Add subscription form ([GOLF-9902](https://lightspeedhq.atlassian.net/browse/GOLF-9902)) will call on every form change.

## What Changes

- Add `POST /private_api/clubs/:club_id/subscriptions/preview`. It takes the customer, products with optional price overrides, the Monthly or Annual schedule, and the payment method, and returns the term dates, the single regular payment (or none for Register), the per-product breakdown, and the quote totals.
- Allow the request only for employees who may sell subscriptions at that club, using the same permission as subscription creation.
- Resolve the request into the calculator's inputs: the customer must belong to the club, and each product must be a supported subscription product of the club or its management company.
- Return `422` with a list of `{field, code}` errors when any input is invalid, including every error the calculator reports. This includes a start month earlier than the first eligible month, judged on the club's local date.
- Choose the payment strategy on the server from the payment method, using the same rule creation uses today. Register previews the full quote for immediate Retail POS collection, with no scheduled payment rows. A request that tries to defer Register collection is rejected.
- Follow the club's existing tax conventions. Quote totals come from the existing quote; each product row reports the amount before tax, tax, and amount after tax for that line.
- Save nothing and call no payment provider.
- Add request and unit specs for the endpoint. This proposal does not run tests or implement code.

Out of scope, belonging to sibling tickets: persisting the schedule and creating declared subscriptions ([GOLF-9896](https://lightspeedhq.atlassian.net/browse/GOLF-9896), [GOLF-9900](https://lightspeedhq.atlassian.net/browse/GOLF-9900)), first-term proration ([GOLF-9901](https://lightspeedhq.atlassian.net/browse/GOLF-9901)), the form ([GOLF-9902](https://lightspeedhq.atlassian.net/browse/GOLF-9902)), renewal ([GOLF-9898](https://lightspeedhq.atlassian.net/browse/GOLF-9898)), and the creation feature flag. The M1 UI revamp, subscription contracts, and Q4 features are also out of scope.

## Capabilities

### New Capabilities

- `subscription-preview`: A read-only, permission-checked HTTP preview of a Monthly or Annual subscription: what is accepted, how invalid input is reported, and what the response contains.

### Modified Capabilities

None. `subscription-term-calculation` already covers the calculation; the endpoint consumes it without changing its requirements.

## Impact

- Implementation repository: `/Users/rushipatel/Documents/Repos/chronogolf-GOLF-9899-openspec`. All `app/`, `config/`, and `spec/` paths in this change refer to that checkout; planning artifacts live in `golf-specs`.
- New route in `config/routes/private_api/clubs.rb` and a `preview` action in `app/controllers/private_api/clubs/subscriptions_controller.rb`.
- New interactor `Subscription::Preview` under `app/interactors/subscription/`, which resolves the request and calls the existing `Subscription::Calculator.first_term`. It adds a serializer for the response and a method-to-strategy mapping next to the existing payment setup mappings. The calculator, subscription models, abilities, and existing create path are not changed.
- New specs for the controller action and interactor.
- No migrations, feature flags, locale strings, frontend edits, or payment-provider calls. The private API has no OpenAPI entry for subscriptions today, so no swagger change is planned.
- The preview and create requests must agree on field names. The form will merge the preview's `subscription` section into its create request, so [GOLF-9900](https://lightspeedhq.atlassian.net/browse/GOLF-9900) and [GOLF-9902](https://lightspeedhq.atlassian.net/browse/GOLF-9902) need the contract in `design.md`.

Sources: [GOLF-9899](https://lightspeedhq.atlassian.net/browse/GOLF-9899) and its linked tickets (9894, 9900, 9901, 9902), and the user-supplied `/Users/rushipatel/Documents/TDD_ Subscriptions major upgrade (Final).docx`. The design records where this change departs from the TDD's sample request.
