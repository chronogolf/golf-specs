# Deferred edge cases: Subscription::Calculator

Cases found while comparing the calculator with the existing subscription code. "Deferred" means we decided to leave it for now. "Open" means no decision yet. "Unverified" means not yet compared against the existing code.

## Deferred (decided)

### 1. One membership per subscription
- Existing `Subscription::Create` rejects a subscription with more than one `Membership::Plan` ("Only one membership per subscription allowed").
- The calculator prices it anyway.
- Example: Gold plan $100 plus Silver plan $50 at a 13% club. The calculator returns total $169.50. Create refuses.
- Found by: the random stress test (74 of 300 random cases).
- Decision: the UI already validates this, and the preview does not need to. Revisit if the endpoint is ever called without that UI.

### 2. Customer already has an active subscription in the period
- Existing create rejects a customer who already has an active subscription in the same period ("This customer already has an active subscription within this period").
- The calculator takes no customer, so it cannot know.
- Example: the customer has Gold from Jan 1 to Jan 31. A quote for any plan starting Jan 1 for the same customer is priced normally by the calculator. Create refuses.
- Decision: the calculator could check this, but we are deferring it. It would need a `customer:` argument.

## Open (no decision yet)

### 3. Legacy previous term that does not end on a month's last day
- `next_term` starts the next term the day after the previous one ends, then derives the first payment date from that.
- Example: a previous term of Jan 15 to Feb 14. The next term starts Feb 15, ends Mar 14, and a payment on day 28 lands in February, outside the new term, with no error.
- Options: add a `previous_term` / `unsupported` error in slice 6, or note it on GOLF-10170 for GOLF-9898.

### 4. No price given
- The existing preview and create both fall back to the catalog price when the line has no price.
- The calculator returns `lines[0].price` / `invalid`.
- Example: a plan with catalog $100 and no price. Existing code quotes $100. The calculator errors.
- The future endpoint must fill in the catalog price before calling if the front end can omit it.

### 5. Renewal term length for non-month, non-year terms
- Existing renewal derives the new length from the old one: a month-sized term gives a month, a year-sized term gives a year, anything else repeats the same number of days.
- The calculator only supports `monthly` and `annual`, chosen by the caller.
- Example: a legacy 45-day term renews as 45 days in existing code. Through the calculator it renews as a month or a year.

### 6. Tenant check on products
- Existing create rejects products that belong to another organization (`ensure_lines_part_of_mco`).
- The calculator does not check this. The design puts it in the endpoint.
- Example: a request for club A that includes club B's plan. The calculator prices it.

## Unverified (not yet compared)

### 7. What the real payment strategies charge (slice 4)
- The calculator returns one payment for `quote.amount`, which is the pre-tax base on a tax-exclusive club.
- Example: a $100 plan at 13% tax gives a quote subtotal of 100.00 and total of 113.00, and the payment row is 100.00.
- Read (not executed) in `Sale::PaymentStrategy::Monthly`: a payment's `amount` is that same base, and tax is added when the payment is charged (`amount_to_pay = sale.amount_with_tax`). So the row looks consistent for Monthly.
- Not yet compared for Installment, and not yet exercised by running a real strategy.

### 8. Other checks inside create
- Auto-renew against the payment method type, `billing_date_exceeds_ends_at`, `validate_on_products`, and whatever `Subscription::Factory` enforces.

### 9. Bundle line references
- `Sale::Quote#add_line` accepts `subscription_bundle_id` and `subscription_bundle_reference` but does not store them.
- The old and new paths therefore agree trivially, so PR 1 says nothing about bundle references.

### 10. Renewal `original_price` for bundle lines
- The calculator uses the line's own price as the original price on renewal, matching `Subscription#renewal`.
- The existing quote use case would use the catalog price. Only `original_amount` differs. Totals and payments do not.
- Example: a bundle line saved at $80 with catalog $100 shows original $80 in the calculator and $100 in the use case.

### 11. Not yet tested in the quote comparison
- Rounding on odd prices combined with stacked tax rates, and harmonized-tax clubs.
