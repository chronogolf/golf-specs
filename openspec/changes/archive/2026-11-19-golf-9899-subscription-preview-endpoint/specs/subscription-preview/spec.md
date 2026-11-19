# Spec Delta

## Purpose

Let staff see exactly what a Monthly or Annual subscription will contain before it is created: term dates, payments, per-product amounts, and totals. The preview is permission-checked, read-only, and reports invalid input by field and code.

## ADDED Requirements

### Requirement: Preview endpoint
The system SHALL provide `POST /private_api/clubs/:club_id/subscriptions/preview`. For a valid request it SHALL respond `200` with the term start and end dates, the recurring schedule, the regular payment rows, and the quote totals and per-product breakdown.

#### Scenario: Valid Monthly preview
- **GIVEN** a club-local date of November 2, 2026 and a credit card payment method
- **WHEN** an employee previews a Monthly subscription with one plan priced at 100, start month 2026-12, and payment day 5
- **THEN** the response has start date 2026-12-01 and end date 2026-12-31
- **AND** one payment row due 2026-12-05
- **AND** the quote totals and one per-product row for the plan

#### Scenario: Omitted start month
- **GIVEN** a club-local date of November 2, 2026
- **WHEN** a valid preview omits the start month
- **THEN** the term starts 2026-12-01, the first eligible month

### Requirement: Permission to sell subscriptions
The system SHALL allow a preview only for an employee who is permitted to sell subscriptions at the requested club, under the same rule that governs creating a subscription there. It SHALL NOT reveal prices or customer data to anyone else.

#### Scenario: Employee without the permission
- **WHEN** an employee without the sell-subscriptions permission at that club requests a preview
- **THEN** the response is `403` and contains no quote or term data

#### Scenario: Employee of another club
- **WHEN** an employee who has no role at the requested club requests a preview
- **THEN** the response is `403`

#### Scenario: Unauthenticated request
- **WHEN** a request carries no valid session
- **THEN** the response is `401`

### Requirement: Field and code validation errors
The system SHALL respond `422` with an `errors` list of `{field, code}` entries when any input is invalid, reporting every problem found in the inputs it checked rather than only the first. The schedule is checked once the products and the payment method are accepted. Each `field` SHALL be the path of the offending input relative to the `subscription` object of the request, such as `schedule.start_month` or `lines[0].price`.

#### Scenario: Several invalid inputs
- **WHEN** a request has payment day 31 and a negative price on its first line
- **THEN** the response is `422`
- **AND** the errors include `schedule.payment_day` and `lines[0].price`, each with a code

#### Scenario: Rejected product hides schedule errors
- **WHEN** a request has an unknown product and payment day 31
- **THEN** the errors include only the product's `lines[0].subscription_product` `not_found`
- **AND** the payment day is reported once the product is fixed

#### Scenario: Missing required input
- **WHEN** a request omits the billing cycle
- **THEN** the errors include `schedule.billing_cycle` with code `required`

#### Scenario: Calculator errors are reported
- **WHEN** the schedule calculation reports an error
- **THEN** the response reports it under the matching request field and the same code

#### Scenario: Request without a subscription
- **WHEN** the request body has no `subscription`, or it is not an object
- **THEN** the response is `400`

### Requirement: Customer and product belong to the club
The system SHALL accept only a customer who has an active customership at the requested club, and only supported subscription products (membership plans and services) that belong to that club or to its management company. Anything else SHALL be reported as a field error without revealing whether the record exists elsewhere.

#### Scenario: Customer of another club
- **WHEN** the customer has no active customership at the requested club, including one that is archived or at another club
- **THEN** the errors include `customer_id` with code `not_found`

#### Scenario: Product of an unrelated club
- **WHEN** a line references a plan owned by a club that is neither the requested club nor its management company
- **THEN** the errors include `lines[0].subscription_product` with code `not_found`

#### Scenario: Unsupported product type
- **WHEN** a line names a product type other than a membership plan or service
- **THEN** the errors include `lines[0].subscription_product` with code `unsupported`

#### Scenario: Product that is not an object
- **WHEN** a line's `subscription_product` is present but is not an object
- **THEN** the errors include `lines[0].subscription_product` with code `invalid`

#### Scenario: Management company product
- **WHEN** a line references a plan owned by the club's management company
- **THEN** the product is accepted and priced

### Requirement: Product prices
The system SHALL price each line at its supplied price when present and at the product's current price otherwise. A supplied price of zero SHALL stay zero. A price that is not a non-negative number SHALL be rejected as `invalid`.

#### Scenario: Override price
- **WHEN** a line supplies price 80 for a plan whose catalog price is 100
- **THEN** the quote uses 80 for that line

#### Scenario: Zero price
- **WHEN** a line supplies price 0 alongside a paid line
- **THEN** that line contributes zero and the other line is unchanged

#### Scenario: Invalid price
- **WHEN** a line supplies price "abc" or a negative number
- **THEN** the errors include `lines[0].price` with code `invalid`

### Requirement: First eligible start month
The system SHALL reject an initial start month earlier than the first eligible month, judged on the club's local date rather than the browser's or the server's UTC date, and SHALL NOT return a payment schedule when it does.

#### Scenario: Current month after its first day
- **GIVEN** a club-local date of November 2, 2026
- **WHEN** the start month is 2026-11
- **THEN** the errors include `schedule.start_month` with code `before_first_eligible_month`
- **AND** the response has no payment rows

#### Scenario: Current month on its first day
- **GIVEN** a club-local date of November 1, 2026
- **WHEN** the start month is 2026-11
- **THEN** the preview succeeds with a term starting 2026-11-01

#### Scenario: Club date differs from UTC date
- **GIVEN** a club in a time zone where it is still November 1 while UTC is already November 2
- **WHEN** the start month is 2026-11
- **THEN** the preview succeeds

### Requirement: Annual payment date
The system SHALL place the Annual payment on the selected payment day in the selected payment month within the twelve-month term, using the following calendar year when that month is earlier than the start month. When the payment month is omitted it SHALL default to the start month.

#### Scenario: Payment month in the next calendar year
- **WHEN** an Annual term starts 2026-12 with payment month 1 and payment day 15
- **THEN** the payment is due 2027-01-15 and the term ends 2027-11-30

#### Scenario: Omitted payment month
- **WHEN** an Annual term starts 2026-12 and the payment month is omitted
- **THEN** the payment is due in December 2026
- **AND** the returned schedule states payment month 12

### Requirement: Initial payment today
The system SHALL accept an option to collect the first payment today. It SHALL change only the due date of that payment, not its amount, the term dates, or the stored payment day and month.

#### Scenario: First payment today
- **GIVEN** a club-local date of November 2, 2026
- **WHEN** a Monthly preview for December has payment day 5 and the option set
- **THEN** the payment is due 2026-11-02 with the same amount as without the option
- **AND** the returned schedule still states payment day 5

### Requirement: Payment method and strategy
The system SHALL choose the payment strategy on the server from the payment method and SHALL NOT take it from the client. Credit card, Bank account, and On account SHALL produce one scheduled payment row. Register SHALL return the full quote for immediate Retail POS collection and no scheduled payment rows.

#### Scenario: Scheduled method
- **WHEN** the payment method is bank account
- **THEN** the response has exactly one payment row for the term

#### Scenario: Register
- **WHEN** the payment method is register
- **THEN** the response has an empty payment list and the full quote totals

#### Scenario: Deferred Register collection rejected
- **WHEN** the payment method is register and the request names a scheduled strategy
- **THEN** the errors include `sale_agreement.payment_setup.strategy_name` with code `unsupported`

#### Scenario: Unsupported payment method
- **WHEN** the payment method is missing or is not one of the supported methods
- **THEN** the errors include `sale_agreement.payment_setup.method_name` with code `required` or `unsupported`

### Requirement: Existing tax conventions
The system SHALL calculate totals under the club's existing tax conventions: the same inclusive or exclusive treatment, tax rates, and rounding as the quote the club would produce for the same products. The quote totals SHALL be the amounts before tax, tax, and amount after tax of that quote. Each per-product row SHALL show that line's price and its amounts before tax, tax, and after tax.

#### Scenario: Tax-exclusive club
- **WHEN** a tax-exclusive club previews one plan priced at 100 with 15 percent tax
- **THEN** the quote totals are 100 before tax, 15 tax, and 115 after tax

#### Scenario: Tax-inclusive club
- **WHEN** a tax-inclusive club previews one plan priced at 115 with 15 percent tax
- **THEN** the quote totals are 100 before tax, 15 tax, and 115 after tax

#### Scenario: Several products
- **WHEN** a plan and a service are previewed together
- **THEN** each has its own per-product row
- **AND** the quote totals are the quote's own totals, not a separate sum of the rows

### Requirement: Read-only preview
The system SHALL NOT save any record and SHALL NOT call any payment provider while previewing, whether the request is valid or not.

#### Scenario: Valid preview writes nothing
- **WHEN** a valid preview is served
- **THEN** no subscription, quote, sale item, agreement, payment, or transaction exists afterwards that did not exist before

#### Scenario: Rejected preview writes nothing
- **WHEN** a preview is rejected with validation errors
- **THEN** no record is created and no payment provider is called

### Requirement: Response can be merged into a create request
The system SHALL return the term dates, recurring schedule, and payment rows under a `subscription` object whose shape matches the corresponding parts of the create request, and the quote totals and per-product rows under a separate `quote` object. The returned schedule SHALL state the billing cycle, payment day, and the resolved payment month (empty for Monthly).

#### Scenario: Shape of a successful response
- **WHEN** a valid Monthly preview succeeds
- **THEN** `subscription` contains `start_date`, `end_date`, `schedule`, and `sale_agreement.payment_setup.payments_attributes`
- **AND** `quote` contains the totals and the per-product rows
- **AND** the per-product rows are not inside `subscription`
