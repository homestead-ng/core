+++
title = 'Core Prototype Contract'
weight = 10
+++
# CORE: Household Budget And Purchase Approval Prototype

**Status:** Draft (prototype scope and explicit owner directions approved on
2026-11-19. Detailed requirements, API shapes, security/session details, and
verification criteria remain proposals unless explicitly stated below.

## Purpose And Scope

Describe a first usable Go backend and WebUI slice: establish a two-adult
household, allocate funds to envelopes, submit purchase requests with
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
resource ID is not proof of access. The initial prototype has two adult members;
roles beyond this case and membership in multiple households remain out of scope.
The owner selected a single-use invite link for the second member, valid seven
days and bound to the partner's intended verified Google email. The link does
not itself authenticate or grant access before that identity check.

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

**Proposal:** The backend imports account and transaction data through a provider
boundary. SimpleFIN Setup Tokens are exchanged server-side for Access URLs;
stored Access URLs are treated as secrets and are never logged. CSV/OFX import
remains available when an institution is unsupported. Connection ownership,
sync cadence, duplicate handling, and account removal behavior remain open. The
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

**Proposal:** A confirmed match records posted spending against the selected
envelope shares and releases the request's encumbrance without double-counting.
If the transaction amount differs from the request amount, show the variance and
require a member to confirm; use the actual transaction amount for posted spend.
No automatic amount tolerance is used in the prototype. Refunds, pending
transactions, and the suggestion algorithm remain open.

### CORE-013: Request Envelope Splits

**Owner direction:** The requester selects how a request amount is split across
envelopes rather than the system applying an automatic coverage order.

**Proposal:** Positive shares sum to the request total, and each envelope belongs
to the same household. Approval reserves every share atomically; it does not
partially encumber a request. Editing shares after submission and approver
adjustments remain open.

### CORE-014: Available Funds With Imported Spending

**Proposal:** After posted transactions are assigned to envelopes, available
funds equal allocated funds minus posted envelope spending minus approved open
request encumbrances. A confirmed match moves value from encumbered to posted
spending without counting it twice. Imported spending without a matched request
is assigned or split by a member. Actual posted spending is recorded even if it
puts an envelope below zero; show the resulting deficit for review. Treatment of
income, transfers, unassigned transactions, and transaction corrections remains
open.

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
coverage. Exact mixed cash/card overspending, month-end handling, refunds, and
pending charges remain open for review.

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

**Reservation example for review:** An envelope has $100 before reservations
and an approved open request reserves $40. At month end, $60 is available to
spend. With no other activity, the next month starts with $100 in the envelope,
$40 still reserved, and $60 available. Matching or cancelling that request later
must release its reservation once; a match also records the actual spending.

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

**Proposal:** Provide an operator-documented, WAL-consistent SQLite backup and
restore process. A backup must include committed data still present in the WAL;
copying only a live database file is not a sufficient backup procedure. Verify
restores against a separate destination before replacing live data. Migration
and downgrade compatibility, retention, and backup scheduling remain open.

The operator must separately protect and back up the external encryption key
file. Database backups without that key cannot recover encrypted bank-connection
credentials; loss of the key must not destroy household records or trigger a
plaintext fallback. Recovery may require reconnecting the bank provider. The
database contains sensitive financial data even when connection credentials are
encrypted, so backups and host storage require appropriate access controls.

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

## Dependencies

Depends on the repository's security, testing, and delivery requirements in
REPO. Future Android client requirements consume this contract rather than
defining a second source of household or money behavior. No Android repository or
client implementation is authorized by this draft.

## Acceptance Cases

These are proposed cases for discussion, not approved test obligations.

- Given a household member without a valid Google-backed application session,
  household data is not returned.
- Given a valid member, requests for another household's envelope, request,
  account, or transaction do not reveal or mutate that data.
- Given connected household accounts, only explicitly on-budget accounts
  contribute opening balances or budget-relevant transactions.
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
- Given a database backup without its encryption key, household records remain
  recoverable but encrypted bank credentials are not silently replaced or
  exposed; reconnecting may be required.
- Given a request split across multiple envelopes, approval reserves every share
  or none of them.
- Given a pending or approved request that has not been matched, requester
  cancellation changes it to cancelled and releases all reservations once.
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
- Given a proposed transaction match, no request or balance changes until a
  household member confirms it.
- Given a confirmed match, posted spending and released encumbrance do not
  double-count the purchase.
- Given a request and transaction with different amounts, the variance is shown
  and requires member confirmation; posted spending uses the actual amount.
- Given an imported transaction without a request match, a member can assign or
  split it across envelopes; actual spending posts and any resulting deficit is
  flagged rather than rejected.
- Given a repeated provider sync or CSV/OFX import, an already-imported source
  transaction is not duplicated; exact identity rules remain open.
- Given a self-hosted installation, no core feature requires a subscription,
  paid account, or project payment.
- Given a user declines donations or paid optional services, self-hosted core
  features remain available.
- Given a third-party bank or infrastructure charge, it is identified as an
  external cost rather than a Household ERP software fee.

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
- Owner direction, 2026-11-23: Household ERP is free-to-use FOSS. Donations and
  sponsorships are welcome; future hosting or paid support remains optional and
  does not gate core self-hosted functionality.
- Owner direction, 2026-11-24: Use YNAB-like budgeting as the default reference
  for financial decisions; credit cards are essential to the prototype.
- Owner direction, 2026-11-24: Unplanned spending that creates an envelope
  deficit does not cancel an approved request or remove its reservation; show
  the deficit for the household to resolve.
- No detailed CORE requirement or API route in this draft is approved yet.

## Verification

No implementation or runtime verification exists. Contract structure checks
only confirm document consistency; they do not approve requirements or prove
security and behavior.
