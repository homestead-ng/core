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
The owner selected a single-use invite link for the second member. Expiration,
revocation, email binding, and acceptance by a specific Google identity remain
open.

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
| `GET`, `POST` | `/api/v1/envelopes` | List or create envelopes |
| `PUT` | `/api/v1/envelopes/{id}/allocation` | Set an envelope allocation |
| `GET`, `POST` | `/api/v1/purchase-requests` | List or submit requests |
| `POST` | `/api/v1/purchase-requests/{id}/approve` | Approve a request |
| `POST` | `/api/v1/purchase-requests/{id}/reject` | Reject a request |
| `POST` | `/api/v1/bank-connections/simplefin` | Exchange a Setup Token |
| `POST` | `/api/v1/bank-connections/{id}/sync` | Request a provider sync |
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
Revocation, email binding, link generation, and whether the invite must be
accepted by a specific Google identity remain open.

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
- Scope revision, 2026-11-19: Bank import and transaction matching moved into
  the prototype baseline. CORE-005 is superseded by CORE-014; its ID is retained.
- No detailed CORE requirement or API route in this draft is approved yet.

## Verification

No implementation or runtime verification exists. Contract structure checks
only confirm document consistency; they do not approve requirements or prove
security and behavior.
