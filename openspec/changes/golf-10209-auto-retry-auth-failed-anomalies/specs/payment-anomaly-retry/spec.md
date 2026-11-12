## ADDED Requirements

### Requirement: Automatic retry of authorization-failed payment reports

The system SHALL periodically retry reporting for unresolved, unignored `Anomaly::MissingPaymentTransaction` rows of type `Sale::HouseAccountPayment` whose `paid_at` is within the retry window and whose most recent failed report audit entry records an HTTP 401.

#### Scenario: Retry after token is restored
- **WHEN** an eligible anomaly's latest report failure is a 401 and the club's Retail authorization has since been restored
- **THEN** the periodic run reports the payment via `attempt_to_report_sale` and the anomaly is marked resolved

#### Scenario: Payment already synced
- **WHEN** the payment already has a Retail integration reference
- **THEN** the run skips the payment and does not report it again

#### Scenario: Non-authorization failure
- **WHEN** the latest failed report audit entry is not a 401
- **THEN** the run does not retry the payment

#### Scenario: Outside retry window
- **WHEN** the payment's `paid_at` is older than the retry window
- **THEN** the run does not retry the payment

#### Scenario: Repeated failures for one club
- **WHEN** a club fails `MAX_CONSECUTIVE_FAILURES` attempts in a row during one run
- **THEN** the run stops retrying that club, logs the club, and continues with the remaining clubs
