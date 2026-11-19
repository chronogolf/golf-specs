# Design

## Context

Application and test paths resolve under `/Users/rushipatel/Documents/Repos/chronogolf`. See `proposal.md` for motivation and `specs/subscription-declared-renewal/spec.md` for the behavior contract.

How renewal works today, and what the declared path takes from it:

| Location | Existing behavior | Declared path |
| --- | --- | --- |
| `app/use_cases/subscription/expire.rb` | Locks the expiring term, stops it, checks eligibility, then builds the next term and agreement and charges overdue payments, all in one transaction. | Keep the lock, stop, eligibility, and charge steps. Only the two "build" steps change. |
| `Subscription#renewal(start_at)` | Next term starts at `start_at` (the job's `Time.current`) and ends after the previous duration. Copies only customer, club, auto-renew, and lines. | Not used for declared terms. A sibling method builds the successor from calculated dates and also copies the three schedule fields. |
| `Sale::AgreementTemplate#new_instance` | Copies the previous payment count, splits the total evenly, and dates payments from the previous first payment's day. Refuses a payment before `sold_at`. | Reused for the quote, payment setup, and the "no payment before sold_at" check. Takes the payments explicitly instead of deriving them. |
| `Subscription::Calculator.next_term` | Already returns start, end, one payment (amount and due date) and the quote for the term after `previous_term`, from an explicit schedule. | Called with the stored schedule. |
| `Subscription` model | The three schedule fields cannot be changed after a term is saved. Renewal copies only the fields in `RENEWAL_ATTRIBUTES` (customer, club, auto-renew), and the schedule fields are not in that list. | The successor must be built with the schedule fields already set, because setting them on a saved term would be rejected. The new successor-building method sets them explicitly. |

## Goals / Non-Goals

**Goals:**

- One renewal flow: the legacy and declared paths differ only in where the dates and payment come from.
- No new abstraction beyond what the declared path needs: no new interactor, no second renewal workflow.
- Make a declared term's renewal correct when the job is late, without the legacy path's date logic leaking in.

**Non-Goals:**

- Do not change the legacy path, its dates, or its specs.
- Do not move charging out of the renewal transaction, add alerts, or change the periodic job.
- Do not handle proration lines or payments on renewal. That is GOLF-9906, which builds on this.
- Do not add a database constraint for duplicate successors. See decision 5.
- Do not change how or when money is collected. The TDD asks that a renewal cannot "collect the same intended payment twice" and lists remedies (idempotency keys on card charges, committing the term before charging). Those are separate work. The charge stays inside the renewal transaction, so a rollback after a card charge behaves as it does today for any renewal. This change only guarantees no duplicate successor term, agreement, or payment row.

## Decisions

### 1. Branch inside `Expire`, keep one flow

`Subscription::Expire` keeps its lock, stop, and eligibility steps and its charge of overdue payments. Right after those, it asks whether the term has a billing cycle:

- declared: calculate the next term with `Calculator.next_term(previous_term:, schedule:)`, where the schedule is the term's stored three fields. Build the successor from the result and its agreement with the calculator's payment.
- not declared: the code that exists today.

`Subscription::Expire` is a legacy use case, and `AGENTS.md` asks for new business logic in interactors, but the change here is a branch inside an existing flow, not new business logic. An interactor around only half of one transaction would split a single flow across two places and need to share its lock and rollback. The new code stays small methods inside the use case.

The predicate is the existing private `schedule_declared?` on `Subscription`, made public. A term is declared when `billing_cycle` is `monthly` or `annual`, which is the same test the validations already use.

Alternative rejected: a separate interactor for declared renewal called from the worker. It would have to take the same row lock and the same transaction to avoid a duplicate successor, which is what `Expire` already does.

### 2. Build the successor term from calculated dates

A new method next to `Subscription#renewal` builds an unsaved successor from the calculation: same customer, club, and auto-renew, the three schedule fields copied, lines built with `price_from_source` as `renewal` does, `starts_at` the start of the calculated start date and `ends_at` the end of the calculated end date, both in the club's timezone. The method is separate from `renewal`, and `RENEWAL_ATTRIBUTES` is unchanged, so legacy callers and specs are unaffected. Copying the schedule into `renewal`'s attributes would not be enough anyway: `renewal` also sets the dates from the duration.

Example: Monthly term ending Dec 31, job runs Jan 2 at 09:00. Calculator returns start Jan 1, end Jan 31. Successor `starts_at` is Jan 1 00:00 club time, `ends_at` Jan 31 23:59:59 club time.

The successor is saved and started as today (`save`, `start`). Its `starts_at` can be a day in the past when the job is late; legacy never produced that, so the start step needs a test with a past start.

### 3. Agreement: reuse the template, pass the payment in

`Sale::AgreementTemplate#new_instance(sold_at, payments_start_on:)` gets an optional `payments:` argument, a list of `{amount:, due_at:}`. When given, the template uses it instead of copying the previous count and splitting the total. The quote still comes from the successor's lines through the existing method, and the existing "no payment before sold_at" check still runs, now against the earliest supplied payment.

The declared path calls it with `sold_at` equal to the successor's `starts_at` and one payment: the calculator's amount, due at the start of the calculated due date in club time.

Register (External) terms never reach this path: only card, bank account, and on account payment methods allow auto-renew (`validate_auto_renewable`), and the prototype says the same ("Only subscriptions paid by credit card or on account can renew"). So the path does not handle a calculation that returns no payment rows.

Example: previous agreement had two payments of 60 and 60.25, stored day 15, term December. The new agreement has one payment of the calculator's amount, due December 15, sold December 1.

The calculator also returns its own unsaved quote. It is not saved. The agreement factory already rejects an agreement whose payments do not add up to its quote (`payments_not_balancing_sale`), so a mismatch between the calculator's amount and the saved quote fails loudly instead of silently saving different numbers.

Alternatives rejected:

- Saving the calculator's quote. It lacks the previous quote's description and club attributes the template copies, and would put two quote-building paths on one agreement.
- Building the agreement without the template. That duplicates payment-setup copying (method, strategy, config, POS references) for no gain.

### 4. Late runs and the "payment before sold_at" check

The calculator places the next term from the previous term's end date, with no clock. A payment is due on the stored day inside the term that starts on the 1st (Monthly) or on the start month's 1st (Annual), so its due date is never before the term start, and the template check cannot fail on a calculated payment. The check stays as a safety net, unchanged, as the ticket says.

The check matters in practice: in the TDD's production numbers, "Requires payments_start_on date to be after sold date" is the cause of 452 of the 1,743 `ExpireWorker` jobs that exhausted their retries since September 1. That is the legacy path, where `sold_at` is the run time. A declared payment cannot hit it, because its date is derived from the same term start as `sold_at`.

A payment whose date is already past when the job runs stays on that date. The existing `process_overdue_payments` step then charges it during renewal, as it does for any overdue payment today.

### 5. Duplicate protection comes from the existing lock

`Expire` locks the expiring term (`FOR UPDATE`) and checks for a successor after taking the lock. A second concurrent run waits for the first to commit, then sees the successor and creates nothing. The worker also uses a `until_executed` unique lock. Both apply to declared terms without change.

The existing spec stubs the lock, so it proves nothing about duplicates. The declared specs run without that stub: one calls `Expire` twice and checks nothing new is created, and one checks the term is locked `FOR UPDATE` first. A true two-thread race is not written, because the specs run in transactional fixtures, which hide uncommitted rows from a second connection, and the repo has no pattern for committed-data database concurrency specs.

No unique index on `renewed_from_subscription_id` is added. It would be a schema change on a large table with legacy rows, and the ticket scopes none. See Open Questions.

### 6. Failure handling is the existing rollback

A calculator error, an invalid successor, or an invalid agreement fails and rolls back through `fail_and_rollback`, the way an invalid legacy agreement does. The expiring term's `stop` is in the same transaction, so it is rolled back too, and the worker raises as it does today. The declared path does not fall back to legacy dates: that would silently reintroduce the drift this ticket exists to remove.

Calculator errors are field/code pairs, not messages. They are turned into the failure message list that the worker already joins into its exception text (for example `payment_day: invalid`).

A declared term that has no sale agreement cannot be calculated (the calculator needs the previous strategy). Declared creation always saves an agreement, so this is not expected, and the declared path reports the error rather than renewing with legacy dates.

### 7. Independent of the creation flag

The declared path reads no ConfigCat flag. Its only gates are the ones `Expire` already applies. The flag does not exist in code yet; GOLF-9900 adds it and must not be consulted from renewal.

### 8. Where the TDD needs correction

| What the TDD says | What we do instead, and why |
| --- | --- |
| The TDD shows renewal calling `next_term` with two inputs: the schedule settings and today's date. | We pass only the term's stored billing cycle, payment day, and payment month. The calculator we already built takes just those, and it needs no date: the next term always starts the day after the previous one ends, so the day the job runs cannot matter. |
| The TDD says to copy the schedule to the next term by adding the three fields to `RENEWAL_ATTRIBUTES`, the list of fields `Subscription#renewal` copies. | That would copy the schedule, but `renewal` would still set the new term's start to the time the job runs and its end from the previous term's length. Those two rules are the drift this ticket removes. So we add a separate method that copies the three fields and takes the dates from the calculator, and leave `renewal` alone for legacy terms. |
| The TDD describes `Subscription::Expire` as unchanged. | It needs one branch, "does this term have a billing cycle?". Everything else in `Expire` stays. |
| The TDD mentions a switch to pause declared renewals in an emergency. | Not in GOLF-9898's criteria. It belongs with rollout in GOLF-9909. |

## Risks / Trade-offs

- [Calculated payment and saved quote disagree] → The agreement factory rejects it; a spec compares both for a tax-inclusive and a tax-exclusive club.
- [Legacy renewal regresses] → The legacy branch is the old code untouched, and the existing `expire_spec` examples stay unedited.
- [Declared term created in the future with no agreement] → Reported as a failure, not renewed with legacy dates.

## Migration Plan

No schema or data change. Deploy this before enabling declared creation (GOLF-9900): until then nothing declared exists, so merging changes no live renewal. Rollback is a revert; terms with no billing cycle never touch the new code.

## Open Questions

- Add a unique index on `renewed_from_subscription_id` as a second defense against duplicate successors? It can be added later without changing the behavior or tasks here.
