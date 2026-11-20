# Spec Delta

## Purpose

Defines how a subscription term with a declared Monthly or Annual schedule renews, so the next term follows the stored schedule instead of the time the renewal job runs or the previous term's payment dates.

## ADDED Requirements

### Requirement: Renewal path follows the stored billing cycle
A term that has a billing cycle SHALL renew through the declared path, which places the next term from the stored schedule. A term with no billing cycle SHALL renew through the existing path, with the same dates, payments, and errors as before. The renewal eligibility checks (the term is expired, not cancelled, not already renewed, auto-renew is on, and the club has subscriptions enabled) SHALL be the same on both paths.

#### Scenario: Declared term uses the declared path
- **WHEN** an expired Monthly term with auto-renew on is renewed
- **THEN** its next term is placed from the stored schedule, not from the previous term's duration or first payment

#### Scenario: Term with no billing cycle is unchanged
- **WHEN** an expired term with no billing cycle, including a Custom term created after the schedule fields exist, is renewed
- **THEN** its next term starts when the job runs and its payment day follows the previous first payment, exactly as before

#### Scenario: Eligibility is not widened
- **WHEN** an expired declared term is cancelled, has auto-renew off, belongs to a club without subscriptions enabled, or already has a successor
- **THEN** no successor is created

### Requirement: Schedule copied to the next term
The next term SHALL store the same billing cycle, payment day, and payment month as the term it renews. Its customer, club, auto-renew setting, and products SHALL carry over as they do today.

#### Scenario: Monthly schedule copied
- **WHEN** a Monthly term with payment day 15 renews
- **THEN** the next term is Monthly with payment day 15 and no payment month

#### Scenario: Annual schedule copied
- **WHEN** an Annual term with payment day 5 and payment month 1 renews
- **THEN** the next term is Annual with payment day 5 and payment month 1

### Requirement: Consecutive terms when the job runs late
The next term SHALL start on the club-local day after the previous term ends and SHALL end one cycle later, whatever time the renewal job runs. Its agreement SHALL be sold at the start of that scheduled term, not at the time the job runs and not at the previous agreement's sale time. Later terms SHALL NOT drift from the schedule.

#### Scenario: Run on time
- **WHEN** a Monthly term ending November 30 is renewed on December 1
- **THEN** the next term runs December 1 through December 31 and its agreement is sold December 1

#### Scenario: Run a day late
- **WHEN** a Monthly term ending December 31 is renewed on January 2
- **THEN** the next term still runs January 1 through January 31
- **AND** its agreement is sold January 1, not January 2

#### Scenario: Annual term
- **WHEN** an Annual term ending November 30, 2027 is renewed
- **THEN** the next term runs December 1, 2027 through November 30, 2028

#### Scenario: Club timezone decides the day
- **WHEN** a term ends at the end of November 30 in America/Los_Angeles and the job runs at 02:00 UTC on December 1
- **THEN** the club-local date is still November 30, the term is not yet expired, and nothing renews until the club's December 1 begins

### Requirement: Payment follows the stored payment day
The next agreement SHALL have one regular payment due on the stored payment day (and, for Annual, the stored payment month inside the new term). Neither the previous agreement's payment dates nor whether its first payment was moved to the day of sale SHALL change it. A payment whose date has already passed when the job runs SHALL stay on its scheduled date; it SHALL NOT move to the following month. A payment date SHALL NOT be earlier than the agreement's sale date.

#### Scenario: Monthly payment on the stored day
- **WHEN** a Monthly term with payment day 15 renews into December
- **THEN** the next payment is due December 15

#### Scenario: First payment today on the original subscription
- **WHEN** the original subscription's first payment was moved to the day it was sold, which was the 20th, and its stored payment day is 5
- **THEN** the renewal's payment is due on the 5th of its term, not the 20th

#### Scenario: Payment day already passed when the job runs late
- **WHEN** a Monthly term with payment day 1 is renewed on January 2 for the term starting January 1
- **THEN** the payment stays due January 1 and is overdue
- **AND** it is not moved to February 1

#### Scenario: Annual payment month
- **WHEN** an Annual term starting December 1, 2026 with payment day 5 and payment month 1 renews
- **THEN** the next term's payment is due January 5, 2028

### Requirement: Current prices and one regular payment per term
The next term SHALL use current product prices (a bundle line keeps its saved price), as renewal does today. The next agreement SHALL have exactly one regular payment per term, whatever number of payments the previous agreement had, and its amount SHALL be the amount the calculation returns for that term.

#### Scenario: Price changed since the previous term
- **WHEN** a product's price was 80 on the previous term and is 100 now
- **THEN** the next term's line and regular payment use 100

#### Scenario: Previous agreement had several payments
- **WHEN** the previous agreement had more than one payment row
- **THEN** the next agreement still has exactly one regular payment

### Requirement: No duplicate successor
Repeated or concurrent attempts to renew the same term SHALL create at most one successor term, one agreement, and one regular payment. An attempt that finds the term already renewed SHALL leave it as it is and report success.

#### Scenario: Renewal attempted twice
- **WHEN** a declared term is renewed and the same renewal runs again
- **THEN** only the first creates a term, agreement, and payment

#### Scenario: Two renewals at the same time
- **WHEN** two workers renew the same declared term at once
- **THEN** one creates the successor and the other finds it and creates nothing

### Requirement: Independent of the declared-creation flag
Declared renewal SHALL NOT depend on whether creating new declared subscriptions is enabled.

#### Scenario: Creation turned off
- **WHEN** declared creation is off and an existing declared term expires with auto-renew on
- **THEN** it renews through the declared path

### Requirement: Failure leaves nothing partial
If the next term cannot be calculated or its term, agreement, or payment cannot be saved, renewal SHALL save none of them, SHALL leave the expiring term as it was before the attempt, and SHALL report the failure. It SHALL NOT fall back to the legacy dates for a declared term.

#### Scenario: Calculation error
- **WHEN** the stored schedule or the previous agreement's strategy makes the calculation return an error
- **THEN** no term, agreement, or payment is created
- **AND** the expiring term is not stopped by the failed attempt
- **AND** the failure is reported so the job can be retried
