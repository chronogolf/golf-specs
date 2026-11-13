# Spec Delta

## Purpose

Provide a read-only, consistent calculation of Monthly and Annual subscription terms and regular payments for future preview, creation, and renewal consumers while preserving existing pricing and tax behavior.

## ADDED Requirements

### Requirement: Common calculation result
The system SHALL return term start and end dates, regular payment amounts and due dates, an unsaved quote, and field/code validation errors. Subtotal, tax, and total SHALL be available from the quote without separate copies on the result. Dates SHALL be club-local calendar dates; amounts SHALL preserve the existing quote calculation's number types and rounding.

#### Scenario: Valid calculation
- **WHEN** a valid Monthly or Annual term is calculated
- **THEN** the result contains its start date, end date, payment rows, and unsaved quote
- **AND** subtotal, tax, and total are read from that quote
- **AND** its errors are empty

### Requirement: Resolved product-and-price inputs
The system SHALL calculate using product-and-price inputs whose products have already been resolved by the caller. It SHALL not parse an HTTP product payload or resolve product IDs as part of calculation. Product lookup and tenant authorization SHALL remain the caller's responsibility.

#### Scenario: Calculate a supplied product
- **WHEN** the caller supplies a resolved membership product and its selected price of 80
- **THEN** the quote uses that product and price without looking up the product again by ID
- **AND** existing tax and POS-reference reads remain available

### Requirement: First eligible start month
For the first term, the system SHALL use the current month as the earliest full-term start when the club-local date is the first of the month, and next month otherwise. An omitted start month SHALL default to that earliest month. Earlier start months SHALL be rejected.

#### Scenario: Current month eligible on its first day
- **WHEN** the club-local date is November 1, 2026 and no start month is supplied
- **THEN** the first full term starts November 1, 2026

#### Scenario: Next month eligible after the first day
- **WHEN** the club-local date is November 2, 2026 and no start month is supplied
- **THEN** the first full term starts December 1, 2026

#### Scenario: Earlier month rejected
- **WHEN** the club-local date is November 2, 2026 and November 2026 is selected
- **THEN** calculation returns a validation error identifying the start month
- **AND** does not return a usable payment schedule

#### Scenario: Later month accepted
- **WHEN** the club-local date is November 2, 2026 and March 2027 is selected
- **THEN** the first full term starts March 1, 2027

### Requirement: Monthly calendar terms
The system SHALL calculate a Monthly term from the first through the last calendar day of its selected month, including February in leap and non-leap years.

#### Scenario: Leap February
- **WHEN** February 2028 is selected for a valid first Monthly term
- **THEN** its dates are February 1 through February 29, 2028

#### Scenario: Non-leap February
- **WHEN** February 2027 is selected for a valid first Monthly term
- **THEN** its dates are February 1 through February 28, 2027

### Requirement: Annual calendar terms
The system SHALL calculate an Annual term as twelve calendar months beginning on the first of the selected month and ending the day before the same month starts in the following year.

#### Scenario: Annual term across a leap year
- **WHEN** December 2027 is selected for a valid first Annual term
- **THEN** its dates are December 1, 2027 through November 30, 2028
- **AND** its period includes February 29, 2028 without shifting either boundary

### Requirement: Recurring payment date
For a supported scheduled-payment strategy and a valid positive quote total, the system SHALL return one regular payment per term on a selected day from 1 through 28. Annual calculations SHALL use the selected payment month within the term, defaulting an omitted payment month to the term's start month. Monthly calculations SHALL ignore payment month.

#### Scenario: Monthly payment day
- **WHEN** a December 2026 Monthly term has payment day 28
- **THEN** its regular payment is due December 28, 2026

#### Scenario: Annual payment month defaults to start month
- **WHEN** a December 2026 Annual term has payment day 5 and no payment month
- **THEN** its regular payment is due December 5, 2026

#### Scenario: Annual payment month crosses the year boundary
- **WHEN** a December 2026 Annual term has payment month January and payment day 5
- **THEN** its regular payment is due January 5, 2027
- **AND** not January 5, 2026

### Requirement: Initial collection override
The system SHALL support moving the first regular payment's due date to the club-local date without changing its amount, term boundaries, or recurring payment day and month. Subsequent-term calculation SHALL not inherit that initial collection override or derive recurring dates from the first term's actual payment date.

#### Scenario: Collect before the selected full term starts
- **WHEN** the club-local date is November 10, 2026, a December Monthly term uses payment day 1, and first-payment-today is enabled
- **THEN** its regular payment is due November 10, 2026
- **AND** its amount is unchanged and the term remains December 1 through December 31
- **AND** a subsequent January term uses January 1 as its payment date

### Requirement: Consecutive subsequent terms
The system SHALL start the next term on the club-local day after the previous term ends and apply the declared Monthly or Annual calendar duration. It SHALL not apply first-term eligibility restrictions to subsequent terms or move an overdue scheduled payment into a later month because calculation occurs late.

#### Scenario: Late Monthly calculation
- **WHEN** the previous Monthly term ends December 31, 2026, payment day is 1, and calculation occurs January 2, 2027
- **THEN** the next term runs January 1 through January 31, 2027
- **AND** its payment remains due January 1, 2027

#### Scenario: Annual successor
- **WHEN** the previous Annual term ends November 30, 2027, payment day is 5, and payment month is January
- **THEN** the next term runs December 1, 2027 through November 30, 2028
- **AND** its payment is due January 5, 2028

### Requirement: Club-local date semantics
The system SHALL determine eligibility and today-based payment dates in the club's timezone. Date boundaries SHALL use calendar arithmetic rather than fixed elapsed-hour durations.

#### Scenario: UTC and club disagree about the month
- **WHEN** calculation receives November 2, 2026 at 00:30 UTC for a club in America/Los_Angeles
- **THEN** its local date is November 1 and November remains an eligible first-term month
- **AND** first-payment-today means November 1, not November 2

#### Scenario: Month containing a daylight-saving transition
- **WHEN** a valid March Monthly term is calculated for a club whose timezone changes offset during March
- **THEN** its dates remain March 1 through March 31

### Requirement: Existing pricing and tax conventions
The system SHALL use supplied first-term price overrides and existing product taxes. Subsequent terms SHALL use current renewal pricing rules: ordinary products use their current source price and bundle products retain their saved line price. Regular payment amounts SHALL reconcile to the quote's existing base amount, not unconditionally to its tax-inclusive total.

#### Scenario: Ordinary product repriced for next term
- **WHEN** an ordinary product was sold for an overridden price of 80 and its current source price is 100
- **THEN** the subsequent term is calculated using 100 for that product

#### Scenario: Existing bundle price preserved
- **WHEN** a bundle product's saved line price is 80 and its standalone source price is now 100
- **THEN** the subsequent term is calculated using the saved bundle line price of 80

#### Scenario: Tax-exclusive quote
- **WHEN** a tax-exclusive product has base price 100 and an applicable tax rate of 13 percent
- **THEN** the regular payment base amount is 100
- **AND** the quote subtotal is 100, tax is 13, and total is 113

#### Scenario: Tax-inclusive quote
- **WHEN** a tax-inclusive product has base price 113 and an applicable tax rate of 13 percent
- **THEN** the regular payment base amount is 113
- **AND** the quote subtotal is 100, tax is 13, and total is 113

#### Scenario: Mixed product rates
- **WHEN** the term contains products with different applicable tax rates
- **THEN** quote totals match the existing quote calculation's product tax and rounding conventions
- **AND** the regular payment base amount equals the quote base amount

### Requirement: Preserve zero-price and positive-payment rules
The system SHALL retain zero-priced product lines when the overall subscription has a positive payment amount. For a term whose entire quote is zero and whose payment strategy requires positive scheduled payments, calculation SHALL retain the zero quote but return a payment validation error and no usable payment schedule. It SHALL not introduce free-subscription creation behavior.

#### Scenario: Free product alongside a paid product
- **WHEN** a tax-free term contains one product priced at 0 and another priced at 100
- **THEN** both quote lines remain present with those prices
- **AND** the regular payment amount is 100

#### Scenario: Every product is free with a scheduled payment method
- **WHEN** every product is priced at 0 and the selected strategy handles Credit card, Bank account, or On account payments
- **THEN** the quote total remains 0
- **AND** a field/code error indicates that no positive payment can be produced
- **AND** there is no usable payment schedule or payment-provider call

### Requirement: External Register calculation
For External/Register, calculation SHALL return the full term and quote totals with no scheduled payment rows. It SHALL not open Retail checkout or model immediate Retail collection as a deferred regular payment. The existing Register sale and collection workflow SHALL remain unchanged outside calculation.

#### Scenario: Calculate Register dates and totals without starting checkout
- **WHEN** the calculator is asked to calculate a valid Monthly or Annual term using External/Register
- **THEN** it returns the term dates and unsaved quote with full totals
- **AND** it returns no scheduled payment rows
- **AND** the calculator itself does not open Retail checkout or collect money

### Requirement: Structured validation
The system SHALL report invalid cycle, start month, payment day, Annual payment month, product, price, or calculation strategy inputs using field identifiers and stable codes. Only Monthly and Annual cycles and payment days from 1 through 28 are supported. Invalid input SHALL not yield a usable payment schedule.

#### Scenario: Unsupported cycle
- **WHEN** a Custom or unknown cycle is passed to declared-term calculation
- **THEN** a billing-cycle validation error is returned without routing Custom through the new calculator

#### Scenario: Invalid payment day
- **WHEN** the selected payment day is missing, 0, 29, 31, or Last day
- **THEN** a payment-day validation error is returned

#### Scenario: Invalid Annual payment month
- **WHEN** an Annual payment month is supplied outside 1 through 12
- **THEN** a payment-month validation error is returned

#### Scenario: Invalid month representation
- **WHEN** the selected start month is not a valid year-month value
- **THEN** a start-month validation error is returned

#### Scenario: Invalid quote input
- **WHEN** a supplied product is missing or unsupported, or its price is malformed or negative
- **THEN** a field/code error identifies the affected line
- **AND** calculation does not persist a partial quote or payment schedule

### Requirement: Read-only execution and compatibility
Calculation SHALL create no database records, mutate no supplied persisted records, call no payment providers, enqueue no work, and emit no notifications. Introducing the calculator SHALL leave existing Custom, WooCommerce, creation, renewal, and payment callers unchanged until separate integration changes are delivered.

#### Scenario: Successful calculation has no side effects
- **WHEN** either a first or subsequent term is successfully calculated
- **THEN** no subscription, quote, quote line, sale item, agreement, sale, payment, or transaction is persisted or changed
- **AND** no payment-provider call, job, or notification occurs

#### Scenario: Invalid calculation has no side effects
- **WHEN** calculation returns validation errors
- **THEN** it also leaves supplied records and external systems unchanged
