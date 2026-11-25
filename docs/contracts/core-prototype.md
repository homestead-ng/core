+++
title = 'Core Prototype Contract'
weight = 10
+++
# CORE: Household Budget And Purchase Approval Prototype

**Status:** Draft with prototype scope and explicit owner directions approved on
2026-11-19 and subsequently recorded dates. Detailed requirements, API shapes,
security/session details, and verification criteria remain proposals unless
explicitly stated below.

**Ownership review required, 2026-11-25:** The owner subsequently chose a
modular platform with independently deployable modules, including optional
Budgeting. This draft predates that direction. Its recorded budgeting and
procurement decisions are preserved, but statements assigning all financial
behavior to Core and its proposed API routes require review before implementation.
Do not treat this note as approval of a new module contract.

## Purpose And Scope

Describe a first usable Go backend and WebUI slice: establish one two-adult
household per server, allocate funds to envelopes, submit purchase requests with
requester-selected envelope splits, and let the other member approve or reject.
Import account transactions through SimpleFIN Bridge or CSV/OFX, assign or split
transactions across envelopes, and suggest matches to approved requests for
member confirmation. Plaid is deferred. The Android companion belongs in a
separate repository after a usable main-app prototype and API contract exist.

## Ownership

The Go backend owns authentication enforcement, household authorization, money
calculations, transaction ingestion, and request state transitions. The WebUI is
the primary client in this repository. Clients display server-provided state;
they do not calculate authoritative balances or perform state transitions
locally. Google owns the external identity provider; successful Google sign-in
alone does not grant household access. Bank-provider credentials and imported
financial data require restricted access and secure storage.

## Requirements

### CORE-001: Prototype Scope

**Owner direction:** The first usable prototype covers a two-adult household,
envelopes and allocation, purchase requests, approval or rejection, reserved
funds, bank transaction import, envelope assignment/splitting, and suggested
request matching with member confirmation. Bank-data import and request matching
were added to the prototype scope on 2026-11-19. Exact behavior remains subject
to review. Broader membership, meal planning, pantry, assets, and family records
are outside this slice.

**Owner direction, 2026-11-24:** One self-hosted server serves one household.
Multi-tenant operation is not planned. The WebUI is desktop-first; full mobile
browser parity is not required for the prototype. The separate native Android
companion remains a later client.

### CORE-002: Google Sign-In

**Owner direction:** Google sign-in is the shared identity provider for WebUI
and future Android app. The same Google account must resolve to the same
application user across clients without manual account matching. Household
membership remains separate; matching identity alone does not join a household.

**Proposal:** Use Google's stable issuer and subject (`iss`, `sub`) as the
external identity key, not email. The backend verifies the credential, creates
or loads the same application user, and establishes a client-appropriate app
session. **Owner direction:** The WebUI uses a secure server session cookie;
Android sends a Google ID token to the Go API and receives a native app session.
Both sessions resolve to the same Google identity but remain separate client
sessions. Exact OAuth clients, token audiences, session format and expiry,
refresh, account recovery, and logout behavior remain open. An unverified
identity must not access household data.

### CORE-003: Household Boundary

**Proposal:** Every household-owned read or mutation is authorized against the
authenticated member's household membership. A client-supplied household or
resource ID is not proof of access. The prototype has one household per server
and two adult members; roles beyond this case are out of scope.
The owner selected a single-use invite link for the second member, valid seven
days and bound to the partner's intended verified Google email. The link does
not itself authenticate or grant access before that identity check.

**Owner direction, 2026-11-24:** Both adults can see and manage household
accounts, envelopes, and requests. The requester still cannot approve their own
request. Private accounts and categories are outside the first prototype.
The operator controls household creation; Google sign-in alone must not grant an
arbitrary user permission to create a household. The setup authorization and
recovery mechanism remain open.

**Owner direction, 2026-11-24:** The operator obtains a single-use setup code
from the server, and the intended creator enters it after Google sign-in to
authorize creation of the household. Code generation, expiry, delivery, and
operator recovery mechanics remain open; the code must not appear in logs.

### CORE-004: Envelope Amounts

**Proposal:** Envelope allocations, request amounts, and transaction amounts use
integer minor currency units, not floating-point values. The prototype uses one
household currency. **Owner direction:** The first prototype uses USD per
household; currency conversion is out of scope. Handling an imported non-USD
account or transaction, formatting, and whether negative allocations are
allowed remain open.

### CORE-005: Available Funds (Superseded)

**Superseded owner direction, 2026-11-19:** The initial prototype direction
defined available funds as manual allocation minus approved, still-open request
reservations, with posted spending deferred. The owner later expanded the
prototype to include imported and assigned transactions. CORE-014 records the
current proposal. This historical requirement is retained; do not implement it
as a separate balance rule.

### CORE-006: Purchase Request Lifecycle

**Proposal:** A submitted request awaits a decision. Approval or rejection moves
it to a decided state. Only an approved, still-open request reserves funds;
rejection does not. **Owner direction:** The requester may cancel a pending or
approved request until a transaction match is confirmed. Cancelling an approved
request releases every envelope reservation atomically. Cancellation after a
confirmed match, cooling-off periods, partial purchases, and other lifecycle
details remain open. Transaction matching behavior is addressed in CORE-012.

**Owner direction, 2026-11-24:** The requester may edit and resubmit a pending
or approved request before matching. A changed request requires a fresh decision
by the other member; editing an approved request releases its reservation while
the revised request awaits review. Exact edit history and the interaction with
concurrent decisions remain open.

**Owner direction, 2026-11-24:** Approved requests stay open and reserved until
matched or cancelled; automatic request expiry is not part of the prototype.

### CORE-007: Approval Authority

**Owner direction:** The requester cannot decide their own request. In the
two-adult prototype household, the other member may approve or reject. Broader
approver roles and exceptions are out of scope for this prototype.

### CORE-008: Atomic Decisions

**Proposal:** Approval and reservation of every request envelope share are one
atomic server-side state change. Repeated or concurrent decisions must not
create duplicate reservations or silently overwrite a terminal decision. The
owner selected that a request may be submitted when funds are insufficient, but
approval is blocked until every selected envelope share is available. Exact
conflict responses remain open.

### CORE-009: Versioned Client API

**Proposal:** Expose household, envelope, purchase-request, account-connection,
and transaction operations through a documented, versioned HTTP API, initially
under `/api/v1/`. Candidate operations include invites, envelope allocations,
request approval/rejection, bank connection setup, transaction import, envelope
splits, and match suggestions/confirmation. Exact routes, payloads, errors,
pagination, and session transport are not yet approved. This API will be the
future integration boundary for the separate Android repository.

Candidate routes for review:

| Method | Path | Purpose |
| --- | --- | --- |
| `POST` | `/api/v1/auth/google` | Android exchanges a Google ID token for an app session |
| `GET` | `/api/v1/me` | Current user and household membership |
| `POST` | `/api/v1/households` | Create the household |
| `POST` | `/api/v1/household-invitations` | Create a single-use invite |
| `POST` | `/api/v1/household-invitations/accept` | Accept an invite |
| `GET` | `/api/v1/accounts` | List linked accounts and budget status |
| `PUT` | `/api/v1/accounts/{id}/budget-status` | Include/exclude account from budget |
| `POST` | `/api/v1/budget/start` | Capture opening balances and start the budget |
| `GET`, `POST` | `/api/v1/envelopes` | List or create envelopes |
| `PUT` | `/api/v1/envelopes/{id}/allocation` | Set an envelope allocation |
| `GET`, `POST` | `/api/v1/purchase-requests` | List or submit requests |
| `POST` | `/api/v1/purchase-requests/{id}/approve` | Approve a request |
| `POST` | `/api/v1/purchase-requests/{id}/reject` | Reject a request |
| `POST` | `/api/v1/bank-connections/simplefin` | Exchange a Setup Token |
| `POST` | `/api/v1/bank-connections/{id}/sync` | Manually request a provider sync |
| `POST` | `/api/v1/transactions/imports` | Import CSV/OFX data |
| `GET` | `/api/v1/transactions` | List imported transactions |
| `PUT` | `/api/v1/transactions/{id}/envelope-splits` | Assign or split a transaction |
| `GET` | `/api/v1/transactions/{id}/match-suggestions` | Suggest request matches |
| `POST` | `/api/v1/transactions/{id}/matches/{request_id}/confirm` | Confirm a match |

Candidate payload conventions: currency amounts are integer minor units with an
explicit currency code; request creation includes positive envelope shares that
sum to the request amount; transaction assignment shares sum to its amount.
Candidate error classes are unauthenticated, forbidden/not-found, invalid input,
and conflicting state. Household scoping, exact response shapes, error format,
invite-token handling, CSV upload limits, pagination, and API session transport
remain open. These routes are discussion material, not compatibility promises.

Candidate JSON fields for review:

```json
{
  "name": "Household",
  "currency_code": "USD"
}
```

The household currency is selected once at creation. `currency_code` is an
uppercase ISO 4217 code. Whether to default it from locale or require explicit
selection remains open.

```json
{
  "on_budget": true
}
```

The account budget-status operation explicitly includes or excludes that
account from envelope budgeting. The budget-start request may include a
`start_date`; exact date semantics and opening-balance response shape remain
open.

```json
{
  "name": "Food",
  "group_name": "Living",
  "allocated_minor": 80000
}
```

```json
{
  "title": "Kitchen mixer",
  "merchant": "Example Shop",
  "item_url": "https://shop.example/item",
  "justification": "Replace the broken one",
  "amount_minor": 12500,
  "envelope_splits": [
    {"envelope_id": "env_example_1", "amount_minor": 10000},
    {"envelope_id": "env_example_2", "amount_minor": 2500}
  ]
}
```

Request shares use the household currency, are positive, and sum exactly to
`amount_minor`. `merchant`, `item_url`, and `justification` are optional.

```json
{
  "direction": "outflow",
  "amount_minor": 12500,
  "currency_code": "USD",
  "posted_at": "2026-11-19T15:04:05Z",
  "envelope_splits": [
    {"envelope_id": "env_example_1", "amount_minor": 10000},
    {"envelope_id": "env_example_2", "amount_minor": 2500}
  ]
}
```

Transactions use a non-negative integer amount plus an explicit direction, so
provider sign conventions do not leak into envelope math. `posted_at` uses
RFC 3339 UTC. Candidate response objects use opaque string IDs and include
server-calculated `allocated_minor`, `posted_minor`, `encumbered_minor`, and
`available_minor`; clients must not derive these totals themselves.

Candidate error response:

```json
{
  "error": {
    "code": "insufficient_available_funds",
    "message": "The selected envelopes do not have enough available funds."
  }
}
```

Candidate status mapping: `401` unauthenticated, `404` missing or inaccessible
resource, `409` current-state conflict (including insufficient funds), `413`
upload too large, `422` invalid fields or split totals, and `502` provider
failure. Exact codes, messages, validation boundaries, rate limits, response
envelopes, and pagination remain open. All candidate response fields and examples
are proposals, not stable API guarantees.

### CORE-010: Household Invitation

**Owner direction:** The creator invites the second adult using a single-use
invite link that expires after seven days. The link can be accepted once.
Acceptance requires a Google identity with the intended partner's verified
email, as specified in CORE-003. Revocation and link generation remain open.

### CORE-011: Bank Providers And Import

**Owner direction:** SimpleFIN Bridge and CSV/OFX import are the initial provider
scope; Plaid is deferred. This direction reflects self-hosting fit and current
public pricing, not a guarantee of bank coverage or a final cost model.

**Owner direction:** A member manually starts each provider sync; scheduled
automatic polling is out of scope for the first prototype. CSV/OFX import is
also member initiated.

**Owner direction, 2026-11-24:** A household can start without connecting
SimpleFIN by creating accounts and opening balances manually; CSV/OFX import
remains available. SimpleFIN support is still part of the prototype. Members can
enter and correct manual transactions without importing them. Exact editing and
audit details remain open.

**Owner direction, 2026-11-24:** Automatically skip proven duplicate imports;
hold possible duplicates for a member to review rather than silently discarding
an uncertain transaction. Exact source-specific identity and update rules remain
open.

**Proposal:** The backend imports account and transaction data through a provider
boundary. SimpleFIN Setup Tokens are exchanged server-side for Access URLs;
stored Access URLs are treated as secrets and are never logged. CSV/OFX import
remains available when an institution is unsupported. Connection ownership,
sync status, manual-sync limits, duplicate identity rules, and account removal
behavior remain open. The
owner reports that their joint account can access both partners' individual
accounts, so one joint-account connection is expected to cover this household;
confirm actual institution coverage during setup. Subscription coverage is not
a decision gate for this prototype.

Public pricing checked 2026-11-19: the SimpleFIN Bridge site lists $1.50/month
or $15/year plus tax, up to 25 institutions and 25 apps. It does not establish
per-connection pricing. The owner confirmed their joint account can access both
partners' individual accounts; the prototype will use that joint-account
connection and verify institution access during setup. Plaid's Trial is limited
to 10 Items; exact paid rates are not publicly disclosed. Pricing does not block
the prototype provider choice.
Sources: <https://bridge.simplefin.org/>, <https://simplefin.org/protocol.html>,
<https://plaid.com/pricing/>, <https://plaid.com/docs/account/billing/>.

### CORE-012: Transaction Splits And Request Matching

**Owner direction:** Imported transactions can be split across envelopes.
Suggested matches to approved requests require member confirmation. A suggestion
alone does not change balances or close a request. The prototype uses a one-to-one
match: one request may match one transaction, and one transaction may match one
request. A transaction can still be split across envelopes.

**Owner direction, 2026-11-24:** Pending bank charges are displayed separately
as a warning or preview; they do not change official envelope balances and cannot
be confirmed as request matches until posted. Matching pending and posted
records and communicating the risk of unposted charges remain open.

**Proposal:** A confirmed match records posted spending against the selected
envelope shares and releases the request's encumbrance without double-counting.
If the transaction amount differs from the request amount, show the variance and
require a member to confirm; use the actual transaction amount for posted spend.
No automatic amount tolerance is used in the prototype. Refunds, pending
transactions, and the suggestion algorithm remain open.

**Owner direction, 2026-11-24:** Either household member may choose the actual
transaction's envelope split and confirm a match, including when it differs from
the approved request's split. A separate partner approval is not required for
that reconciliation. The original approved request and the bank transaction
remain distinct records.

**Above-estimate example for review:** A household has $100 in a home envelope,
with no other spending or reservations. Approving an $80 request reserves $80,
leaving $20 available. A $120 debit-card charge is imported. A member reviews
and confirms the $40 variance and assigns the actual charge to that envelope.
The system releases the $80 reservation, records $120 of spending, closes the
match, and shows a $20 deficit. It must not reject or alter the bank charge to
make the plan balance.

**Changed-split example for review:** A request reserves $60 from home and $20
from personal spending. Its actual bank charge is $100. Either member may
confirm a $70 home and $30 personal split. The $80 in original reservations is
released and the $100 transaction is recorded against the chosen envelopes.
**Proposal:** Retain the approved and actual splits for later review.

**Proposal:** Match confirmation and requester cancellation compete for the
same open request atomically. Only one may succeed; either path releases its
reservation once. If matching wins, record the transaction's actual envelope
spending once. If cancellation wins, keep the imported transaction available
for review and categorization; do not delete or silently ignore it. A repeated
confirmation must not create another match or post spending twice. Exact API
conflict and retry responses remain open.

**Owner direction, 2026-11-24:** A member may undo an incorrect match and review
it again. Keep the bank transaction, restore the request's reservation, retain
correction history, and flag any resulting deficit. **Proposal:** Keep actual
spending recorded until explicitly recategorized; restoring an open request's
reservation represents a separate future commitment, not another bank charge.
Exact concurrent undo, cancellation, and rematching behavior remains open.

### CORE-013: Request Envelope Splits

**Owner direction:** The requester selects how a request amount is split across
envelopes rather than the system applying an automatic coverage order.

**Proposal:** Positive shares sum to the request total, and each envelope belongs
to the same household. Approval reserves every share atomically; it does not
partially encumber a request. Editing shares after submission and approver
adjustments remain open.

### CORE-014: Available Funds With Imported Spending

**Proposal:** For a budget month, available funds reflect carried envelope funds
and assignments or reallocations, minus posted envelope spending and approved
open request encumbrances. See CORE-021 for proposed rollover behavior. A
confirmed match moves value from encumbered to posted spending without counting
it twice. Imported spending without a matched request is assigned or split by a
member. Actual posted spending is recorded even if it puts an envelope below
zero; show the resulting deficit for review. Treatment of income, transfers,
unassigned transactions, and transaction corrections remains open.

### CORE-015: Budget Opening Baseline

**Owner direction:** Budget setup records opening account balances and a budget
start date. Transactions before the start are history-only. Opening cash is
available for allocation, rather than added to envelope allocations as a second
source of funds. The treatment of later income, transfers, and card debt is
proposed in CORE-020.

**Proposal:** Capture provider balances for explicitly on-budget accounts when
the budget starts, and store a baseline timestamp. The exact relationship
between the user-visible start date, provider balance timestamp, and transactions
posted on the cutoff date needs definition before implementation. Whether setup
requires every opening dollar to be allocated before normal use also remains open.
SimpleFIN provides a `balance-date`; pending transactions may have `posted` set
to zero, and pending data is optional. A date alone cannot safely determine
whether every charge is reflected in the opening balance. See the
[SimpleFIN protocol](https://simplefin.org/protocol.html). Reconciliation at
the cutoff requires an explicit design and acceptance cases. **Proposal:** Show
the opening balances for household review and hold ambiguous transactions for
review rather than silently counting them again or discarding them.

**Proposed opening workflow:** For each selected on-budget account, show the
reported balance and its available timestamp, let a member confirm or correct
the opening amount, and retain the confirmed amount and reference point. Import
older transactions as history. If a transaction posts later but could already
be reflected in the confirmed opening balance, ask a member whether it belongs
to the opening balance or should change the budget; retain the decision so a
later sync does not apply the transaction twice. Do not treat a pending charge's
missing posted timestamp as evidence that it is new spending. Exact provider
balance types, cutoff boundaries, and later corrections remain open.

### CORE-016: On-Budget Account Selection

**Owner direction:** A member explicitly marks each linked account on-budget or
tracking-only. Only on-budget accounts contribute opening balances and
budget-relevant transactions to envelope calculations.

**Proposal:** Newly connected accounts default to tracking-only until selected.
Changing account status after the budget starts must not silently rewrite past
allocations or transaction assignments; migration behavior remains open.

**Owner direction, 2026-11-24:** Members can add accounts and change their
on-budget or tracking-only status after budget setup, with explicit review.
The application must not silently rewrite historical allocations or spending.
When a new on-budget cash account is added, its reviewed opening balance becomes
unassigned money; earlier transactions are history-only. Opening card debt is
tracked separately, as described in CORE-020. Cutoff ambiguities still require
the review described in CORE-015.

### CORE-017: SimpleFIN Credential Encryption

**Owner direction:** Encrypt stored SimpleFIN Access URLs in SQLite using a key
file stored outside the database volume. The key must persist across app restarts
and remain separately protected from database backups.

**Proposal:** Use authenticated encryption with a 32-byte random key, read from a
secret file mounted read-only for the app. Never log Setup Tokens, Access URLs,
or key material. Missing or invalid key material must fail closed for affected
connections, not fall back to plaintext. Key generation, path/config name,
permissions, rotation, and recovery procedures remain open for implementation
design.

### CORE-018: Durable Database

**Owner direction from the original stack handoff:** Use SQLite in WAL mode with
`modernc.org/sqlite` for a pure-Go build. Persist the database outside the
container's writable layer. The exact database path, migration runner, backup
format, and restore procedure remain open for implementation design.

### CORE-019: Free And Open-Source Access

**Owner direction, 2026-11-23:** The software is free of charge to use and open
source. No license fee, required subscription, paid account, or paywall may gate
the self-hosted core or its core features. Donations and sponsorships are
welcome. Future hosted service or paid support may be offered separately and
must remain optional for self-hosters. Third-party services and infrastructure
may charge independently; their costs must not be presented as project fees.

### CORE-020: Budgeting Reference And Credit Cards

**Owner direction, 2026-11-24:** Use YNAB-like budgeting behavior as the
default reference when refining financial decisions, except where household
procurement needs a different rule. Credit cards are essential to the first
usable prototype. This is product direction, not blanket approval of YNAB's
behavior or of the proposals below.

**Proposal:** Treat an existing credit-card balance as debt, not spendable
cash. Money can be assigned toward that existing balance. When a covered card
purchase is recorded against a spending envelope, move the covered money to a
card-payment category. A payment from an on-budget cash account to the card is
a transfer, not a second expense. Record and flag overspending as directed in
CORE-014. For an uncovered card purchase, propose moving only covered funds to
the card-payment category; the uncovered amount remains card debt. Cash
overspending instead means cash has left the budget without sufficient envelope
coverage. Exact mixed cash/card overspending, month-end handling, partial refunds,
and matching pending charges to posted transactions remain open for review.

**Owner direction, 2026-11-24:** A member reviews an imported refund and
assigns it back to the original spending envelope; card-payment funds are
adjusted accordingly. The exact handling of a missing original purchase,
multi-envelope refunds, and partial refunds remains open.

**Cash-flow example for review:** A household has $1,000 in checking and no
money assigned yet. The budget shows $1,000 available to assign. Assigning $300
to groceries leaves $700 available to assign and $300 in groceries. A $500
paycheck increases available-to-assign money to $1,200. Moving $200 from
checking to an on-budget savings account does not create income or spending and
does not change those budget amounts. Opening balances must not be added again
when historical transactions are imported.

**Proposed reconciliation invariant:** When all on-budget cash-account activity
has been accounted for and there is no overspending or unresolved correction,
the total of unassigned cash and envelope balances, including reserved amounts
and card-payment balances, equals the cash-account balance. Credit-card debt is
tracked separately. A purchase request reservation reduces an envelope's
available-to-spend amount but does not remove cash or create spending. Treatment
of deficits and unresolved transactions must be specified before relying on this
invariant generally.

For example, with $1,000 in checking and $200 already owed on a card, propose
starting with $1,000 available to assign, not $800. Assign $200 to the card
payment and $50 to groceries. A covered $50 grocery card purchase moves that
$50 to the card-payment category: $250 is set aside for the $250 card balance.
Paying $250 transfers money from checking to the card without another grocery
expense. Exact opening-balance timing and corrections still need definition.

**Procurement example for review:** A groceries envelope has $100 available.
Approving a $50 request reserves $50, leaving $50 available. A member confirms
a matching $45 card charge. The $50 reservation is released, $45 is recorded as
groceries spending, and $45 is set aside for the card payment; the groceries
envelope now has $55 available. The member sees and confirms the $5 variance.
This example assumes the purchase is fully covered and does not settle rules for
partial charges, shortfalls, corrections, or refunds.

Reference: <https://www.ynab.com/blog/how-to-do-credit-cards-in-ynab>.
Overspending reference:
<https://support.ynab.com/en_us/overspending-in-ynab-a-guide-ryWoxEyi>.

### CORE-021: Monthly Rollover And Open Reservations

**Proposal:** Treat each budget month as part of a continuous plan. Positive
unspent envelope balances carry into the next month; a new month does not create
cash or reset an approved, still-open request's reservation. Keep the reservation
against its original envelope shares until the request is matched or cancelled.

**YNAB-inspired proposal:** Uncovered cash overspending reduces the following
month's unassigned cash. Uncovered credit-card overspending remains card debt
and does not become cash available to assign. Do not book a second charge when
the month changes. Mixed cash/card overspending, later edits to prior months,
and how a member covers an overspend across the boundary require further design.

**Proposal:** Distinguish an envelope's balance before reservations from its
available-to-spend amount. A negative available amount caused solely by an open
reservation exceeding the remaining envelope balance is a commitment shortfall,
not an additional cash purchase. Carry the balance and reservation forward and
show the shortfall; do not also deduct it from next month's unassigned cash as
cash overspending. Actual cash spending beyond the envelope balance still needs
the cash-overspending treatment above.

**Reservation example for review:** An envelope has $100 before reservations
and an approved open request reserves $40. At month end, $60 is available to
spend. With no other activity, the next month starts with $100 in the envelope,
$40 still reserved, and $60 available. Matching or cancelling that request later
must release its reservation once; a match also records the actual spending.

**Commitment-shortfall example for review:** An envelope has $100 and an open
request reserves $80. An unplanned $50 cash purchase leaves $50 in the envelope
before reservations and $30 below zero available. If unresolved at month end,
carry the $50 balance and $80 reservation forward and continue showing the $30
shortfall. Do not also deduct $30 from next month's unassigned cash solely for
the reservation.

**Overspending examples for review:** With $20 allocated and a $30 cash purchase,
the envelope is $10 overspent; if left uncovered, the next month's unassigned
cash is reduced by $10. For a $30 card purchase against a $20 envelope, $20 is
set aside toward the card payment and $10 remains unfunded card debt. These cases
assume no other transactions, prior-month adjustments, or reservations.

Reference for overspending:
<https://support.ynab.com/en_us/overspending-in-ynab-a-guide-ryWoxEyi>.

### CORE-022: Sign-In And Invitation Security

**Proposal:** The browser sign-in uses an OIDC authorization-code flow with
PKCE, a one-time state value, and a nonce. The server verifies the token's
signature, issuer, intended client audience, expiry, and nonce before creating
an application session. Identify a Google user by verified issuer and subject;
do not merge users solely because their email matches.

An invite token is unpredictable, single-use, and stored as a verifier rather
than plaintext. Acceptance requires a signed-in Google identity whose verified
email matches the intended recipient. The server checks expiry, household
capacity, and prior use atomically with membership creation. An invite alone
does not create a session or authorize household access. Exact email comparison,
revocation, recovery, rate limits, and session duration remain open.

The browser session uses a secure, HttpOnly cookie and an appropriate SameSite
policy. Unsafe cookie-authenticated requests require CSRF protection; GET and
HEAD do not make state changes. Remote access requires HTTPS. Local development
and reverse-proxy configuration must be specified without weakening production
session security. The later Android session protocol is a separate proposal.

### CORE-023: Bank Connection Request Safety

**Proposal:** Treat the SimpleFIN Setup Token, claim URL, Access URL, and any
embedded credentials as secrets and untrusted input. Never return them in API
responses, logs, traces, or unsanitized provider errors. Store Access URLs only
after authenticated encryption as described in CORE-017.

For the hosted Bridge flow, require HTTPS and reject requests to loopback,
private, link-local, multicast, or otherwise non-public destinations by default.
Apply the same checks to redirects and to the address actually dialed, including
IPv4-mapped IPv6 addresses, so DNS changes cannot bypass validation. Configure
outbound proxies so they cannot silently bypass this policy. Bound redirects,
request duration, and response size. Local-network SimpleFIN servers may need a
future explicit, separately reviewed exception; they are not silently allowed.

Surface a sanitized connection failure without exposing credentials. A failed
or repeated sync must not create partial budget changes or duplicate imported
transactions. Exact provider retries, identity matching across imports, and
operator-configured network policy remain open.

### CORE-024: Backup And Recovery

**Owner direction, 2026-11-24:** The first prototype uses local backups, with
scheduled and operator-initiated backups. An authorized operator can restore a
local backup without Google sign-in or internet access; normal application access
still requires the configured sign-in. Offline restore uses a documented local
server command. Google Drive backup is deferred. Full local backups default to
daily, with the most recent seven successful copies retained; schedule and
retention are configurable.

**Proposal:** Provide an operator-documented, WAL-consistent SQLite backup and
restore process. A backup must include committed data still present in the WAL;
copying only a live database file is not a sufficient backup procedure. Verify
restores against a separate destination before replacing live data. Report
scheduled-backup failures and retain a previous successful backup when a new one
fails. Destination configuration, time of day, migration/downgrade compatibility,
and incremental-backup retention remain open.

The operator must separately protect and back up the external encryption key
file. Database backups without that key cannot recover encrypted bank-connection
credentials; loss of the key must not destroy household records or trigger a
plaintext fallback. Recovery may require reconnecting the bank provider. The
database contains sensitive financial data even when connection credentials are
encrypted, so backups and host storage require appropriate access controls.

**Future direction, not prototype scope:** Either adult may connect a Google
account for optional Drive backups after separate authorization. Encrypt cloud
backups before upload and use a recovery key stored outside the server and Drive.
This backup recovery key is distinct from the runtime key that protects SimpleFIN
credentials. Whether the latter is separately backed up or included only in an
encrypted backup needs design; recovery must not depend solely on Google access.
Multiple connected destinations and Google OAuth configuration remain open.

**Retention discussion:** The daily/seven-copy default applies to full local
backups. If incremental backups are later supported, the owner suggested
considering increasingly sparse tiers. Incremental retention, backup-size
limits, and tier design are not yet decisions.

### CORE-025: Envelope Reallocation And Concurrent Spending

**Proposal:** A household member may move assigned money between envelopes
without changing total household cash. A move draws from the source envelope's
currently available, unreserved funds; it cannot take money already reserved
for an approved request. Approval and reallocation both check and update their
affected envelopes atomically, so concurrent actions cannot spend the same
available funds twice.

**Owner direction, 2026-11-24:** An imported unplanned purchase records actual
spending even if that creates a deficit. It does not cancel an approved request
or remove its existing reservation. Show the deficit and the open reservation so
the household can move money or cancel the request. Exact rules for modifying an
approved request and how to resolve deficits across months remain open.

### CORE-026: Web Approval Alerts

**Owner direction, 2026-11-24:** The first WebUI prototype shows pending
requests in an approval inbox with a visible indicator for the other member.
Changes made by either member should appear in the other member's WebUI without
a page reload when connected. Bank sync remains manually initiated. Email
notifications and Android push are deferred. Reconnection, delivery, and refresh
mechanics remain open.

### CORE-027: Optional Cash Wallets

**Owner direction, 2026-11-24:** Cash tracking is optional. The household may
individually enable a shared cash wallet and a personal cash wallet for each
adult; none is mandatory. Personal ownership identifies whose cash is held, not
privacy: both adults can see and manage the wallets and their transactions.
Each wallet can be on-budget or tracking-only, following CORE-016. A personal
wallet does not automatically create a personal allowance or bypass purchase
approval rules.

Members may record cash purchases without a prior request, or optionally match
one to an approved request under CORE-012. Unmatched spending is flagged when
applicable under the household's rules. Moving cash between accounts or wallets
is recorded as one reviewed transfer; a bank withdrawal can be linked after
review rather than counted as a second purchase. An ATM fee is recorded
separately from the transfer principal. The specific treatment of transfers
across the on-budget/tracking-only boundary remains open.

Members can compare a wallet's recorded balance with a physical count. When it
differs, show the difference and require a member to confirm the adjustment and
how it affects the budget. The adjustment's categorization and correction
history need further design.

**Transfer example for review:** A checking-account withdrawal totals $103:
$100 cash and a $3 ATM fee. Link the bank withdrawal to a $100 transfer into the
cash wallet and record the $3 fee as an expense. Do not record another $103 bank
withdrawal or a second $3 bank charge. If both accounts are on-budget, the $100
principal moves between them without creating income or spending.

## Dependencies

Depends on the repository's security, testing, and delivery requirements in
REPO. Future Android client requirements consume this contract rather than
defining a second source of household or money behavior. No Android repository or
client implementation is authorized by this draft.

## Acceptance Cases

These are proposed cases for discussion, not approved test obligations.

- Given a household member without a valid Google-backed application session,
  household data is not returned.
- Given a valid member, requests using unknown or inaccessible envelope, request,
  account, or transaction IDs do not reveal or mutate unrelated data.
- Given an authenticated Google user who is not authorized by the operator to
  create the server's household, sign-in alone does not permit household creation.
- Given connected household accounts, only explicitly on-budget accounts
  contribute opening balances or budget-relevant transactions.
- Given a new on-budget cash account after setup, its reviewed opening balance
  becomes unassigned money and earlier transactions remain history-only.
- Given a transaction before the budget start, it is history-only and does not
  reduce envelope availability.
- Given an uncertain relationship between an opening balance and a transaction,
  the transaction is presented for review rather than silently counted twice.
- Given a pending charge later posts and may have been included in the opening
  balance, a member can decide whether it was already included or affects the
  budget; repeating the import does not apply that decision twice.
- Given $1,000 in checking and $200 in existing card debt, $1,000 is available
  to assign; assigning $200 to card payment does not create $200 of new expense.
- Given a covered $50 card purchase assigned to groceries, $50 moves from
  groceries to the card-payment category; paying the card is a transfer.
- Given a $50 approved reservation and a confirmed $45 card purchase, release
  the reservation, record $45 of spending, and show the $5 variance.
- Given an open $40 request reserved in a $100 envelope at month end, the next
  month retains $40 reserved and $60 available without creating new funds.
- Given a $50 remaining envelope balance and an $80 open reservation at month
  end, carry both forward, flag the $30 shortfall, and do not deduct that
  reservation-driven shortfall again from next month's unassigned cash.
- Given uncovered $10 cash overspending at month end, next month's unassigned
  cash is reduced by $10 without recording the purchase a second time.
- Given a $30 card purchase with only $20 budgeted, the proposed card-payment
  funding is $20 and the remaining $10 is unfunded debt.
- Given an invite intended for one verified email, a different Google identity
  cannot accept it.
- Given an invalid OIDC state, nonce, issuer, audience, signature, or expiry,
  sign-in does not create an application session.
- Given concurrent attempts to accept the same invite, at most one membership
  is created, and an expired or previously used token is rejected.
- Given an unsafe cross-site request using a browser session, household state
  does not change.
- Given a stored SimpleFIN Access URL, the database does not contain its
  plaintext; missing encryption-key material never triggers plaintext fallback.
- Given a Setup Token or redirect pointing to a prohibited internal address,
  the server does not connect to that destination or reveal token contents.
- Given provider errors containing credentials, user-visible errors and logs
  omit those credentials.
- Given committed changes in SQLite's WAL, the documented backup and restore
  process retains those changes in a separate restored database.
- Given a local backup and the necessary separately protected key material, an
  authorized operator can restore with a local server command without Google or
  internet access.
- Given default backup settings, full local backups run daily and retain the
  seven most recent successful copies; the operator can change both settings.
- Given a scheduled backup fails, report the failure and preserve a previous
  successful backup; exact retention is still to be decided.
- Given a database backup without its encryption key, household records remain
  recoverable but encrypted bank credentials are not silently replaced or
  exposed; reconnecting may be required.
- Given a request split across multiple envelopes, approval reserves every share
  or none of them.
- Given a pending or approved request that has not been matched, requester
  cancellation changes it to cancelled and releases all reservations once.
- Given an approved request is edited and resubmitted before matching, its prior
  reservation is released and the revised request awaits the other member's
  decision.
- Given a request with a confirmed transaction match, requester cancellation is
  not available; correction behavior remains open.
- Given an invite link older than seven days or already accepted, it cannot be
  accepted again.
- Given insufficient available funds, a request may be submitted but cannot be
  approved until its full split is available.
- Given duplicate or concurrent approval attempts, the request has one decision
  and at most one reservation.
- Given two pending requests for $70 and $60 against $100 available, concurrent
  approvals result in at most one approval; the other remains unapproved.
- Given $80 reserved in an envelope with $100 before reservations, a member may
  move at most $20 out without changing or cancelling the reservation.
- Given an unplanned $50 expense against that envelope, record the expense and
  retain the $80 reservation; available funds show a $30 deficit for review.
- Given an imported transaction split, assigned shares sum to the transaction
  amount before it is considered reconciled.
- Given a pending bank charge, show it separately without changing official
  envelope balances or permitting a confirmed request match before posting.
- Given a reviewed refund, return it to the original spending envelope and
  adjust card-payment funds when applicable; partial-refund details remain open.
- Given a household without SimpleFIN, members can create manual accounts with
  opening balances, enter and correct manual transactions, and use CSV/OFX import.
- Given cash tracking is not enabled, the household can use the prototype without
  creating a cash wallet.
- Given a personal cash wallet, both adults can see and manage it; its ownership
  does not automatically grant a personal-allowance approval exception.
- Given an on-budget cash wallet, its balance contributes to budget funds; a
  tracking-only wallet does not.
- Given a $103 bank withdrawal consisting of $100 cash and a $3 fee, linking
  it records one $100 transfer and one $3 fee without a duplicate bank charge.
- Given a cash purchase without a request, a member can record actual spending;
  an approved request may instead be matched when applicable.
- Given a physical cash count differs from the wallet balance, the system shows
  the difference and waits for a member to confirm an adjustment and its budget
  treatment.
- Given a request awaiting review, the other member can see it in the WebUI
  approval inbox and its indicator.
- Given a household change while the other member's WebUI is connected, the
  affected view updates without a page reload or an automatic bank sync.
- Given a proposed transaction match, no request or balance changes until a
  household member confirms it.
- Given a confirmed match, posted spending and released encumbrance do not
  double-count the purchase.
- Given a request and transaction with different amounts, the variance is shown
  and requires member confirmation; posted spending uses the actual amount.
- Given $100 in an envelope, an $80 approved reservation, and a confirmed $120
  debit-card match, release the reservation, record $120 of spending, and show a
  $20 deficit without altering or rejecting the bank charge.
- Given an approved request split $60/$20 and a $100 matched transaction,
  either member can confirm actual envelope shares of $70/$30 without another
  partner approval; release the original reservations once.
- Given cancellation and match confirmation race on one approved request, only
  one succeeds, the reservation is released once, and the bank transaction is
  retained whether matched or available for further review.
- Given a repeated confirmation of the same transaction/request match, no
  additional spending or reservation release is created.
- Given an incorrect confirmed match, a member can undo it, retain the bank
  transaction and correction history, and restore the request reservation; any
  resulting deficit is flagged.
- Given an imported transaction without a request match, a member can assign or
  split it across envelopes; actual spending posts and any resulting deficit is
  flagged rather than rejected.
- Given a repeated provider sync or CSV/OFX import, an already-imported source
  transaction is not duplicated; exact identity rules remain open.
- Given a possible duplicate that cannot be proven, retain it for member review
  rather than silently discarding a potentially legitimate charge.
- Given an account is added or its on-budget status changes after setup, require
  explicit review without silently rewriting historical allocations or spending.
- Given a self-hosted installation, no core feature requires a subscription,
  paid account, or project payment.
- Given a user declines donations or paid optional services, self-hosted core
  features remain available.
- Given a third-party bank or infrastructure charge, it is identified as an
  external cost rather than a Homestead-NG software fee.

## Decisions

- Owner approval, 2026-11-19: Proceed with the prototype scope and explicit
  owner directions in this contract. This does not blanket-approve the remaining
  proposed requirements, API details, or security/session behavior.
- Owner direction, 2026-11-19: This repository contains the Go backend and
  primary WebUI. The Android companion belongs in a separate repository after
  an API contract and usable main-app prototype exist.
- Owner direction, 2026-11-19: Start with a two-adult household, envelopes and
  allocations, purchase requests, approval/rejection, reserved funds, bank
  transaction import, envelope splits, and confirmed request matching.
- Owner direction, 2026-11-19: Google OIDC is the sign-in direction.
- Owner direction, 2026-11-19: WebUI and Android resolve the same Google
  identity without manual matching, but use separate client sessions. WebUI
  uses a secure server cookie; Android exchanges a Google ID token for a native
  app session through the Go API.
- Owner direction, 2026-11-19: The creator invites the second adult by single-use
  invite link valid for seven days; the requester cannot decide their own request.
- Owner direction, 2026-11-19: The requester may cancel a pending or approved
  request until a transaction is matched; cancellation releases reservations.
- Owner direction, 2026-11-19: Requests may be submitted when funds are short;
  approval waits until all selected envelope shares are available.
- Owner direction, 2026-11-19: Use SimpleFIN Bridge and CSV/OFX first; defer
  Plaid pending provider coverage and cost evaluation.
- Owner direction, 2026-11-19: Use USD as the first prototype's single household
  currency.
- Owner direction, 2026-11-19: A household member manually starts provider sync;
  scheduled automatic polling is deferred.
- Owner context, 2026-11-19: The household's joint bank account can access both
  partners' individual accounts; a single joint-account connection is expected
  to cover the prototype household.
- Owner direction, 2026-11-19: Requesters choose envelope splits; imported
  transactions may be split; suggested request matches require member confirmation.
- Owner direction, 2026-11-19: The first prototype matches one request to one
  transaction; a transaction may still be split across envelopes.
- Owner direction, 2026-11-19: Amount differences require confirmation, use the
  actual transaction amount for posted spending, and have no automatic tolerance.
- Owner direction, 2026-11-19: Unplanned imported spending can make an envelope
  negative; record actual spend and flag the deficit rather than reject it.
- Owner direction, 2026-11-23: Record opening balances and a budget start date;
  earlier transactions are history-only.
- Owner direction, 2026-11-23: Require explicit on-budget/tracking-only selection
  for each linked account.
- Owner direction, 2026-11-23: Bind single-use invites to the intended partner's
  verified Google email.
- Owner direction, 2026-11-23: Encrypt SimpleFIN Access URLs in SQLite with a
  persistent key file stored outside the database volume.
- Owner direction from original stack handoff: SQLite WAL with
  `modernc.org/sqlite`; persist database state outside the container layer.
- Scope revision, 2026-11-19: Bank import and transaction matching moved into
  the prototype baseline. CORE-005 is superseded by CORE-014; its ID is retained.
- Owner direction, 2026-11-23: Homestead-NG is free-to-use FOSS. Donations and
  sponsorships are welcome; future hosting or paid support remains optional and
  does not gate core self-hosted functionality.
- Owner direction, 2026-11-24: Use YNAB-like budgeting as the default reference
  for financial decisions; credit cards are essential to the prototype.
- Owner direction, 2026-11-24: Unplanned spending that creates an envelope
  deficit does not cancel an approved request or remove its reservation; show
  the deficit for the household to resolve.
- Owner direction, 2026-11-24: Either member can choose the actual envelope
  split and confirm a transaction match even when it differs from the request.
- Owner direction, 2026-11-24: Display pending charges separately; exclude them
  from official envelope balances and confirmed request matches until posted.
- Owner direction, 2026-11-24: Allow manual account setup and opening balances
  without a SimpleFIN connection; retain CSV/OFX and SimpleFIN support.
- Owner direction, 2026-11-24: Return reviewed refunds to their original
  spending envelope and adjust card-payment funds when applicable.
- Owner direction, 2026-11-24: A requester may edit and resubmit before matching;
  a changed request needs a new partner decision and releases any old reservation.
- Owner direction, 2026-11-24: Both adults share household data and management
  access; private accounts and categories are deferred.
- Owner direction, 2026-11-24: The WebUI has an approval inbox and indicator;
  email notifications and Android push are deferred.
- Owner correction, 2026-11-24: One server serves one household; multi-tenancy
  is not planned. The operator controls household creation.
- Owner direction, 2026-11-24: The prototype WebUI is desktop-first. Connected
  members see each other's household changes without a page reload; bank sync
  is still manual.
- Owner direction, 2026-11-24: Approved requests remain reserved until matched
  or cancelled; automatic expiry is outside the prototype.
- Owner direction, 2026-11-24: Skip proven duplicate imports and hold ambiguous
  possible duplicates for member review.
- Owner direction, 2026-11-24: Permit reviewed account additions and budget-status
  changes after setup without silently rewriting historical allocations or spend.
- Owner direction, 2026-11-24: A new on-budget cash account's reviewed opening
  balance becomes unassigned money; older transactions are history-only.
- Owner direction, 2026-11-24: The operator authorizes household creation using
  a single-use setup code after the creator signs in with Google.
- Owner direction, 2026-11-24: Members may enter and correct manual transactions.
- Owner direction, 2026-11-24: Members may undo incorrect matches, retain the
  bank transaction and correction history, restore the reservation, and flag any
  resulting deficit.
- Owner direction, 2026-11-24: Full local backups default to daily with seven
  successful copies; schedule and retention are configurable. Offline restore
  uses a documented local server command.
- Owner direction, 2026-11-24: Cash tracking is optional, with individually
  enabled shared and adult-owned wallets visible to both members. Each wallet
  can be on-budget or tracking-only.
- Owner direction, 2026-11-24: Review cash transfers once, record cash purchases
  with or without a request, and confirm physical-count adjustments.
- Owner direction, 2026-11-24: Start with scheduled and manual local backups and
  offline operator restore; defer Google Drive backup.
- Future direction, 2026-11-24: Either adult may authorize optional Google Drive
  backups; encrypt cloud backups and keep the recovery key separate.
- No blanket approval of all detailed CORE requirements or API routes is implied;
  explicit owner directions are recorded separately above.

## Verification

No implementation or runtime verification exists. Contract structure checks
only confirm document consistency; they do not approve requirements or prove
security and behavior.
