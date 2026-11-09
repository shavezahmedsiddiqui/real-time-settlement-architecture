Designing Settlement Infrastructure for Real-Time Financial Ecosystems
Reference architecture for high-throughput payment orchestration, double-entry ledger state management, and coordination between domestic instant-payment rails and tokenized assets.

Author: Shavez Ahmed Siddiqui
Topics: fintech · payment-systems · double-entry-accounting · real-time-settlement · fednow · pix · tokenization · distributed-systems
Status: Proposed engineering design — not a deployed payment service, working integration, security audit, or production SDK.
Author website: https://shavezahmedsiddiqui.com/
1. Overview
Fast payment initiation is not the same as final settlement. A payment application must distinguish authorization, network submission, external settlement confirmation, ledger posting, and reconciliation. These events may occur at different times, especially when the transaction crosses institutional or blockchain boundaries.

This repository documents a conceptual payment architecture with:
- An append-only, double-entry journal as the accounting source of truth.
- Posted, reserved, and available balance models.
- Concurrency-safe posting and idempotent requests.
- Separate adapters for FedNow, Pix/SPI, and tokenized rails, where integration arrangements exist.
- Explicit settlement-state transitions and reconciliation of uncertain outcomes.
- Recovery procedures for provider outages, delayed acknowledgments, and non-atomic cross-system flows.
The design is derived from the accompanying technical essay, Designing Settlement Infrastructure for Real-Time Financial Ecosystems. There is no executable application in this initial documentation release.
2. System boundaries
Layer	Responsibility	Important boundary
API / identity	Authenticate requests and enforce user-level permissions	An authenticated request is not a settled payment
Policy and compliance	Validate limits, destination, and applicable checks	Checks depend on jurisdiction and financial institution
Payment orchestration	Manage workflow state, routing, retries, and recovery	Must not infer settlement from a network acknowledgment
Balance/reservation engine	Prevent double spending of available funds	Reservation is not settlement
Double-entry ledger	Atomically journal balanced accounting events	Never mutate committed history silently
Rail adapters	Translate commands and receive external evidence	FedNow and Pix require appropriate eligible participants
Tokenized-asset adapter	Submit authorized on-chain operations and monitor confirmation	Chain inclusion and legal finality vary by system
Reconciliation	Compare external confirmations and internal positions	Unknown outcomes require investigation before retry


3. Reference architecture
<img width="1724" height="2250" alt="mermaid-diagram (8)" src="https://github.com/user-attachments/assets/49fe268b-d720-4a03-b3e4-b1e27770c08e" />
Architecture limitation: The branches represent independent settlement arrangements. The diagram does not imply direct FedNow-to-Pix connectivity, access by non-participants, a universal blockchain bridge, or atomic settlement between independent rails.
4. Accounting model and invariants
A journal transaction must be balanced for each currency:
sum(debits[asset]) == sum(credits[asset])
Consider an internal transfer of USD 500 between two customers whose balances are liabilities on the institution's books:
Ledger account	Debit (USD)	Credit (USD)
Customer A liability	500.00	0.00
Customer B liability	0.00	500.00
Total	500.00	500.00


Liability debits reduce the institution's obligation to a customer; liability credits increase it. A cross-institution transfer requires further asset, clearing, and settlement-related entries as dictated by the accounting model.
Required invariants
1. No committed journal entry is unbalanced in its accounting unit.
2. All legs of a journal transaction commit atomically or not at all.
3. Every committed event has a stable transaction ID and audit metadata.
4. Previously posted entries are not silently edited; corrections use authorized reversals or adjustments.
5. Reservations are enforced atomically against the applicable availability policy.
6. Request retries cannot create an additional payment for the same idempotency identity.
7. An external acknowledgement does not, without appropriate evidence, constitute settlement finality.
8. A transfer with uncertain external status is not blindly resubmitted.
Balance terminology
- Posted balance: Derived from committed accounting entries.
- Reserved amount: Funds committed to outstanding instructions under a reservation policy.
- Available balance: Amount eligible for further spending under the account's policy; in a simplified model, posted balance minus active reservations.
5. Transaction state model
A possible happy path is:
<img width="1172" height="2092" alt="mermaid-diagram (9)" src="https://github.com/user-attachments/assets/623bbc26-1d3c-4116-8f53-0ed72336b97d" />
Modeling note: Accounting reservations or other intermediary postings may occur before external settlement. The diagram is one conceptual lifecycle rather than a universal financial-accounting prescription. The UNKNOWN state is deliberately distinct from FAILED.
6. Illustrative API contracts
Use integer minor units, not floating-point money values. The following JSON is an interface example, not a working endpoint.
{
  "request_id": "pay_2026_000001",
  "idempotency_key": "customer-42-invoice-918",
  "source_account_id": "acct_sender",
  "destination_reference": "beneficiary_ref",
  "currency": "USD",
  "amount_minor": 50000,
  "requested_rail": "fednow",
  "reference": "invoice-918"
}
An adapter result should record independent stages:
{
  "request_id": "pay_2026_000001",
  "orchestration_state": "SUBMITTED",
  "external_reference": "opaque-network-reference",
  "provider_acknowledged": true,
  "settlement_confirmed": false,
  "finality_evidence": null
}
The adapter must not mark settlement_confirmed as true merely because the provider accepted the instruction.
7. Concurrency and idempotency
Concurrent debits
For two withdrawals from the same account, both must not independently reserve the same funds. A potential solution is an atomic balance check and reservation using serializable transactions, account-level serialization, or correctly implemented optimistic concurrency control.
Retry contract
An idempotency record should bind a request key to the payload fingerprint, authorized principal, and durable transaction identity. Reuse of the key with a different payload must be rejected, not interpreted as a new payment.
receive(request)
  validate_identity_and_policy(request)
  existing = find_idempotency_record(request.key)
  if existing:
    assert existing.request_fingerprint == fingerprint(request)
    return existing.transaction_state
  atomically_create_idempotency_record_and_payment(request)
  reserve_funds_if_eligible()
  submit_once_under_controlled_state_machine()
This is pseudocode only. It omits locking implementation, access controls, and failure paths needed in production.
8. Rail-specific integration boundaries
FedNow
FedNow supports eligible US participating financial institutions and operates continuously. Settlement occurs through the relevant Federal Reserve infrastructure. Integration would require an eligible institution or authorized arrangement, applicable messaging and operating rules, and evidence-based status handling.
Pix / SPI
Pix is Brazil's instant-payment ecosystem. Interinstitutional settlement is carried out through the central-bank-operated SPI, subject to its participant rules and settlement arrangements. A generic application cannot treat SPI as an open, direct API.
Tokenized assets
A token transfer needs chain-aware tracking of transaction hashes, inclusion status, confirmation/finality policy, asset identifiers, and relevant issuer or custody arrangements. The legal meaning of a token varies by its terms and underlying system.

Important: A transfer on a fiat rail and a transfer on a blockchain are generally not one atomic transaction. Compensating workflows manage residual obligations; they do not undo settlement finality.
9. Bridge workflow: fiat funds to tokenized assets
A hypothetical, legally supported bridge might follow these steps:
1. Authenticate the participant and validate permissions, limits, and applicable compliance rules.
2. Verify fiat funding or reserve sufficient funds according to the product's accounting model.
3. Submit a fiat movement using an eligible payment provider.
4. Obtain evidence that satisfies the rail's settlement criteria.
5. Authorize token-side issuance, transfer, or release under its separate control rules.
6. Track blockchain execution and apply the relevant confirmation/finality policy.
7. Reconcile bank/provider accounts, token liabilities, journal entries, and exceptions.
A token must not be issued on the unsupported assumption that an external fiat instruction is final. If token-side execution fails after fiat settlement, record the unresolved obligation and follow an authorized recovery process.
10. Failure scenarios and expected handling
Scenario	Risk	Minimum expected response
Duplicate payment request	Double disbursement	Resolve to original idempotency record
Lost acknowledgment	Unknown settlement outcome	Query/reconcile before retry
Stale or contradictory provider status	False success indication	Hold in UNKNOWN; investigate
Concurrent reservations	Overspending	Atomic funds availability check
External provider outage	Unavailable settlement route	Pause or use an eligible alternative with controlled state reconciliation
Chain transaction pending	Premature token-side completion	Await defined confirmation/finality condition
Token action fails after fiat settlement	Unresolved obligation	Exception queue and compensating workflow
Ledger and statement mismatch	Misstated financial position	Flag and reconcile; no silent historical rewrite


11. Test and validation plan
This specification has no measured benchmark results. A future prototype should demonstrate:
- Balanced postings: Property tests verifying that posted debits equal credits per currency.
- Atomicity: Inject a failure between posting legs; no partial journal transaction may commit.
- Race safety: Launch concurrent debits against the same available balance; total approved spending must not exceed limits.
- Idempotency: Retry identical requests, including after simulated network timeouts; only one logical payment may be created.
- Unknown outcome recovery: Lose an external acknowledgment after submission and verify that no blind second submission occurs.
- Non-atomic bridge failures: Simulate confirmed fiat settlement followed by failed token operation and verify explicit reconciliation exceptions.
- Recovery replay: Restart services mid-workflow and reconstruct durable payment state without duplicating value movement.
- Observability: Measure p50/p95/p99 posting latency, reconciliation lag, exception counts, and unresolved transactions.
12. Suggested repository roadmap
real-time-settlement-architecture/
├── README.md                         # This reference specification
├── docs/
│   ├── ledger-accounting.md          # Proposed accounting policy and postings
│   ├── transaction-state-machine.md  # Lifecycle rules and exceptions
│   ├── rail-adapters.md              # External integration boundaries
│   └── threat-model.md               # Access, replay and failure risks
├── schemas/
│   ├── payment-request.schema.json
│   └── settlement-event.schema.json
└── tests/
    ├── accounting-invariants.md
    ├── concurrency-scenarios.md
    └── recovery-scenarios.md
The files beyond README.md are a proposed future repository structure and are not included in this initial release.
13. Further reading
- Federal Reserve Financial Services — FedNow
- Banco Central do Brasil — Pix
- Banco Central do Brasil — Instant Payment System (SPI)
- BIS — Committee on Payment and Market Infrastructures
- PostgreSQL — Transaction Isolation
14. Author and disclaimer
Shavez Ahmed Siddiqui writes about fintech engineering, financial infrastructure, blockchain-based systems, and digital payments. Official website.

Disclaimer: This repository is a conceptual, educational reference architecture. It does not provide functional access to FedNow, Pix, banks, or blockchain payment networks, and it makes no claim of regulatory authorization, production safety, or independently verified performance. Not financial, legal, or regulatory advice.
