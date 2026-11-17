# Spec Delta

## Purpose

Define which billing schedule a subscription term can declare, how that declaration is validated, and that terms without a declaration stay undeclared and unchanged, so renewal, creation, and display can rely on a stored schedule instead of inferring one from dates.

## ADDED Requirements

### Requirement: Optional declared schedule
A subscription term SHALL be able to store a billing cycle, a payment day, and a payment month. The billing cycle SHALL be Monthly, Annual, or absent. A term with no billing cycle has no declared schedule and follows the existing creation, renewal, and payment behaviour. This includes Custom terms created after the schedule fields exist: Custom is not a stored billing cycle.

#### Scenario: Declared Monthly term
- **WHEN** a term is saved with billing cycle Monthly and payment day 15
- **THEN** the term stores those values and has a declared schedule

#### Scenario: Declared Annual term
- **WHEN** a term is saved with billing cycle Annual, payment day 5, and payment month 1
- **THEN** the term stores those values and has a declared schedule

#### Scenario: Term with no schedule
- **WHEN** a term is saved with no billing cycle, payment day, or payment month
- **THEN** the term is valid and has no declared schedule

#### Scenario: Unsupported billing cycle
- **WHEN** a term is saved with a billing cycle other than Monthly or Annual, including Custom
- **THEN** the term is rejected with an error on the billing cycle

### Requirement: Payment day for a declared schedule
A term with a billing cycle SHALL have a payment day that is a whole number from 1 through 28. The system SHALL NOT accept Last day or any day above 28.

#### Scenario: Boundary days
- **WHEN** a Monthly term is saved with payment day 1, and another with payment day 28
- **THEN** both are valid

#### Scenario: Missing payment day
- **WHEN** a term with a billing cycle is saved with no payment day
- **THEN** the term is rejected with an error on the payment day

#### Scenario: Out-of-range or fractional day
- **WHEN** a term with a billing cycle is saved with payment day 0, 29, 31, or 3.5
- **THEN** the term is rejected with an error on the payment day

### Requirement: Payment month for an Annual schedule
An Annual term SHALL have a payment month that is a whole number from 1 through 12. A Monthly term SHALL NOT have a payment month.

#### Scenario: Boundary months
- **WHEN** an Annual term is saved with payment month 1, and another with payment month 12
- **THEN** both are valid

#### Scenario: Annual term without a payment month
- **WHEN** an Annual term is saved with no payment month
- **THEN** the term is rejected with an error on the payment month
- **AND** no month is chosen on the caller's behalf

#### Scenario: Out-of-range month
- **WHEN** an Annual term is saved with payment month 0 or 13
- **THEN** the term is rejected with an error on the payment month

#### Scenario: Monthly term with a payment month
- **WHEN** a Monthly term is saved with a payment month
- **THEN** the term is rejected with an error on the payment month

### Requirement: No schedule values without a billing cycle
A term with no billing cycle SHALL have no payment day and no payment month. A day or month with no cycle would describe a schedule that was never declared.

#### Scenario: Orphan payment day
- **WHEN** a term with no billing cycle is saved with payment day 10
- **THEN** the term is rejected with an error on the payment day

#### Scenario: Orphan payment month
- **WHEN** a term with no billing cycle is saved with payment month 3
- **THEN** the term is rejected with an error on the payment month

### Requirement: A term with no billing cycle keeps existing behaviour
A term with no billing cycle, whether it existed before the schedule fields or was created later as Custom, SHALL remain valid and unchanged when loaded and saved. The system SHALL NOT infer a schedule from its dates, and SHALL NOT calculate it from a schedule, until a separate migration gives it a billing cycle.

#### Scenario: Existing term keeps no schedule
- **WHEN** a term that existed before the schedule fields is loaded, saved, and reloaded
- **THEN** its billing cycle, payment day, and payment month are still absent
- **AND** its other attributes are unchanged

#### Scenario: Calendar-aligned dates do not imply a schedule
- **WHEN** an existing term runs from the first to the last day of a calendar month
- **THEN** it still has no declared schedule

#### Scenario: Custom term created later
- **WHEN** a Custom subscription is created through the existing Custom flow after the schedule fields exist
- **THEN** it is stored with no billing cycle, payment day, or payment month
- **AND** it is not calculated from a schedule when it is created or renewed

### Requirement: Schedule fixed once the term exists
The billing cycle, payment day, and payment month of a saved term SHALL NOT be changed, added, or removed by updating that term. This includes a term that has no schedule.

#### Scenario: Change a declared schedule
- **WHEN** a saved Monthly term with payment day 5 is updated to payment day 6, or to Annual
- **THEN** the update is rejected with an error on the changed field
- **AND** the stored schedule is unchanged

#### Scenario: Add a schedule to an existing term
- **WHEN** a saved term with no schedule is updated to declare Monthly with payment day 5
- **THEN** the update is rejected with an error on the billing cycle
- **AND** the term still has no declared schedule

#### Scenario: Unrelated update
- **WHEN** a saved term's auto-renew setting is updated and the schedule fields are not touched
- **THEN** the update succeeds and the stored schedule is unchanged

### Requirement: End date fixed on a declared term
A saved term that has a billing cycle SHALL NOT have its end date changed by updating it, so its dates stay aligned with its schedule. A term with no billing cycle SHALL keep its existing ability to have its end date edited.

#### Scenario: Edit the end date of a declared term
- **WHEN** a saved Monthly or Annual term has its end date amended to a different date
- **THEN** the update is rejected with an error on the end date
- **AND** the stored end date is unchanged

#### Scenario: Change auto-renew without moving the end date
- **WHEN** a saved Monthly or Annual term is amended with a new auto-renew setting and its current end date
- **THEN** the update succeeds
- **AND** the end date is unchanged

#### Scenario: Edit the end date of a term with no schedule
- **WHEN** a saved term with no billing cycle has its end date amended
- **THEN** the update succeeds as it does today

### Requirement: Schedule rules match the calculation
The accepted billing cycles, payment days, and payment months SHALL be the same ones the term calculation accepts, so a stored schedule can always be passed to the calculation without being rejected for its own values.

#### Scenario: Stored schedule is calculable
- **WHEN** a term with a valid stored schedule has that schedule passed to the calculation
- **THEN** the calculation raises no billing-cycle, payment-day, or payment-month error for it

### Requirement: Additive and non-disruptive
Adding the schedule fields SHALL NOT change how existing subscriptions are created, renewed, paid, listed, or exported, and SHALL NOT require a backfill. The fields SHALL NOT be exposed by any endpoint or accepted from request parameters as part of this capability.

#### Scenario: Existing creation and renewal behave as before
- **WHEN** a Custom subscription is created and later renewed
- **THEN** neither term has a declared schedule
- **AND** dates, payments, and renewal behavior are as they were before the fields existed

#### Scenario: Fields are not reachable through the API
- **WHEN** a client sends a billing cycle, payment day, or payment month on an existing subscription endpoint
- **THEN** the values are not stored and no response includes them
