# Collection Point: Product Requirements Document


## Overview

Collection Point lets a collector receive group payments by ordinary bank transfer and have every payment identified, allocated and reconciled automatically.

Today a collector who gathers money for several purposes (school fees, dues, a wedding, rent) usually receives everything into one bank account. They match each credit to a person by hand, work out which purpose it was for, and keep the books in a spreadsheet. Collection Point replaces that with a virtual account per collection, one intake account per collector, and a ledger that records who paid, how much and for what.

**Vision:** senders pay from any banking app, and the collector sees a live, correct picture of every collection and the combined balance, with most manual accounting, settlement and reconciliation work gone.

**How it works at a glance**

1. The collector creates collections, and each one gets its own virtual account.
2. Senders pay either a collection's account directly or the collector's intake account.
3. Each credit is matched to a sender. Intake payments are split across that sender's collections in the collector's priority order and recorded in the ledger, and the money stays in the intake account until the collector releases it.
4. The collector reviews doubtful matches, releases money to collection accounts when needed, and sees balances, reports and statements.

This is a draft for review. Open decisions are listed under Risks and open questions.

## Goals, non-goals and success measures

The first release succeeds if a collector can run several collections from one dashboard without reconciling by hand.

**Goals**

1. Cut the manual accounting, settlement and reconciliation work collectors do today on a single account.
2. Let senders pay from any banking app, with no login and no account to create.
3. Identify and allocate every credit, or hold it visibly for review, so nothing stays unexplained.
4. Give the collector one view of every collection and the main balance.
5. Use AI to suggest matches and explain discrepancies, never to move money.

**Non-goals for the first release**

- A sender portal or sender login.
- Card collection, card linking or auto-debit.
- Sender-chosen amounts per collection. Allocation follows the collector's priority order.
- Platform-held funds. The collector is the only legal holder of funds.
- Lending, advances or earning on held funds.

**Success measures**

| Measure | Definition |
| --- | --- |
| Auto-match rate | Share of credits matched to a sender without collector review |
| Review rate | Share of credits that enter the review queue |
| Unmatched age | How long a credit stays unmatched or unallocated |
| Reconciliation differences | Ledger-versus-provider differences found per day |
| Close-out time | Time from a collection's last payment to its final statement |
| Collector effort | Hours a collector spends on reconciliation per month, from a pilot survey |

No baseline exists yet, so targets are set after the first pilot.

## Users and roles

Collection Point has one kind of user, the collector. Senders never log in.

| Role | Who | Access | What they do |
| --- | --- | --- | --- |
| Collector | The person who sets up collections and receives payments, such as a school administrator, treasurer or event organiser | Full dashboard; the only logged-in user | Creates collections, registers senders, sets the priority order, reviews doubtful matches, approves payouts, reads reports. Legally holds the funds and completes KYC |
| Sender | Anyone paying a collector | None | Pays from their own banking app to an account number the collector shares |
| Collector's team (later) | An assistant, treasurer or second approver | Limited roles | Works inside the collector's account with restricted permissions |

**Sender registry.** The collector keeps one record per sender: name, optional phone number, and what they owe in each collection (amount, tier, installment schedule). When the collector confirms a payment, the sender's bank account number is remembered, so later payments from it match without review.

Open collections need no registered senders. Each sender is recorded as they pay.

## Account model

Money lands in virtual accounts from the payment provider (Squad by HabariPay is the primary provider), and the ledger decides what each credit means.

| Account | Count | Receives | Holds money |
| --- | --- | --- | --- |
| Collection account | One per collection | Direct transfers for that collection, and money the collector releases from the intake account | Yes: direct payments and money released to it. Its ledger balance can be higher, because allocated money may still sit in the intake account |
| Intake account | One per collector; the same details for every sender | Payments that may cover several collections | Yes: every intake payment, until the collector releases the allocated part to a collection account. It also holds credits that are unidentified, held for review or in excess |
| Main balance | One per collector | Nothing; it is a view | No. It is the sum of every collection's ledger balance, whether the money sits in the intake or a collection account, cross-checked against provider balances |

**Rules**

- The collector is the only legal holder of funds. The money sits in virtual accounts that Squad issues and still owns, so Collection Point never holds or touches funds.
- Collection accounts take direct transfers and released money, and hold what has been released to each collection. The intake account lets a sender with several obligations pay once.
- Intake payments are allocated in the ledger only. The money stays in the intake account until the collector releases it to a collection account by a real transfer with its own reference. A collection's releasable amount is never more than its unreleased ledger balance.
- A direct payment to a collection account counts toward that collection only. It never triggers the priority split.
- Collectors can move money between collection accounts, with a recorded reason and approval.
- Refunds go back to the sender's account details taken from the original payment, or account that is provided with a screenshot (this needs approval).
- Every movement, including doubtful credits, is an append-only ledger entry. Corrections are new entries, never edits.

## Core flows

Every credit is recorded, matched to a sender, allocated in the ledger and reported, and only a confident match is allocated.

*Diagram (see the live document for the drawing): intake payment and release flow, 13 steps, 2 decisions.* A sender pays the intake account. The webhook is verified and recorded. If the match is confident, the payment is allocated in priority order, ledger entries are made per collection, a receipt is sent to the sender, and the money stays in the intake account. If the match is not confident, the credit goes to the review queue, where the collector decides, and a confirmed match continues to allocation. Later, whenever the collector chooses, they pick an amount to release. If it is within the collection's unreleased balance, the money is transferred from the intake account to the collection account and the unreleased balance is reduced once the transfer is confirmed. If it is over the cap, the release is refused.

An intake payment is recorded and matched. A confident match is allocated in priority order and recorded in each collection's ledger while the money stays in the intake account. A doubtful one waits in the review queue until the collector decides.

**Other flows**

1. **Set up a collection.** The collector enters the purpose, target, deadline and mode. Collection Point issues a virtual account and a ready-to-send payment message. The collector registers senders and what each owes.
2. **Direct payment.** A sender transfers to a collection's own account. The credit is recorded and matched, and it counts toward that collection only.
3. **Payout.** The collector picks a collection, beneficiary and amount. Payouts come only from the collection's own account, so money must be released there first. The account name is checked, a second approver signs off above the collector's limit, and the transfer is recorded with an expense category and receipt.
4. **Refund.** Money returns to the sender account details on the original credit, or to an account the sender provides with a screenshot after approval, singly or in bulk.
5. **Close-out.** Any unreleased money is released or refunded, then the balance is settled, a final statement is issued, and the collection account is closed and archived.

**Release.** Allocation changes the ledger only, so each collection shows three figures: allocated, released and unreleased. The collector picks a collection and an amount to move from the intake account to that collection's account. The amount can be anything up to that collection's unreleased balance, and never more, whatever else the intake account holds. A release needs the same second approval as a payout above the collector's limit, carries its own reference and idempotency key, and reduces the unreleased balance only when the transfer is confirmed. For example, if ₦40,000 is allocated as ₦30,000 to school fees and ₦10,000 to levy, the collector can release at most ₦30,000 to school fees and at most ₦10,000 to levy, in any smaller amounts and at any time.

**How payment outcomes are handled**

| Case | Handling |
| --- | --- |
| Exact payment | Allocated to the sender's obligations in priority order |
| Short payment | Allocated in priority order until the money runs out; later obligations stay open |
| More than everything owed | Every obligation is filled, then the excess is refunded or held as credit if the payment is recurring, as the collector chose |
| Duplicate transfer by a sender | Held and flagged for review, might be held till next payment or refunded. |
| Unidentified sender | Held unallocated in the intake account until the collector confirms |
| Someone paying for another sender | Flagged as a possible third-party payment; the collector confirms who it counts for |
| Cash or offline payment | Logged by the collector as a manual payment and allocated like any other |
| Money sent to a closed collection's account | Open question |

## Functional requirements

Requirements are grouped by area and numbered. The release phasing section assigns each ID to a phase.

**Collection setup**

| ID | Requirement |
| --- | --- |
| CS-1 | Create a collection with a purpose, target, deadline and mode: open, pre-registered, tiered, shared or installment |
| CS-2 | Assign each collection its own virtual account |
| CS-3 | Let the collector set one priority order across their collections for allocation |
| CS-4 | Define installment schedules as expected payment dates and amounts |
| CS-5 | Generate a ready-to-send message with the account name, number and purpose |
| CS-6 | Bulk import senders and amounts from a spreadsheet |
| CS-7 | Recalculate installments for senders who join late |
| CS-8 | Clone a collection and support recurring collections such as monthly dues |
| CS-9 | Offer templates for common purposes such as school fees, rent, weddings and dues |

**Sender registry**

| ID | Requirement |
| --- | --- |
| SR-1 | Keep one sender record per person: name, optional phone, bank account numbers learned over time |
| SR-2 | Track each sender's obligations per collection: amount owed, tier and schedule |
| SR-3 | Remember bank account numbers from confirmed payments so repeat payments match deterministically |

**Payment intake and matching**

| ID | Requirement |
| --- | --- |
| PM-1 | Receive provider webhooks with signature verification, idempotency and replay protection |
| PM-2 | Record every credit with sender name, bank, account number, narration, amount and fees |
| PM-3 | Place each credit in one of three states: auto-matched, needs review or unmatched |
| PM-4 | Provide a review queue where the collector confirms, reassigns or rejects a match; every decision is logged |
| PM-5 | Let the collector log cash and offline payments, marked as manual |
| PM-6 | Apply collector-set rules to underpayments, overpayments, duplicates and payments made by someone else |

**Allocation and settlement**

| ID | Requirement |
| --- | --- |
| AL-1 | Split each intake payment across the sender's collections in priority order, capped at the amount still owed in each |
| AL-2 | Apply a direct payment to its own collection only |
| AL-3 | Record each allocation as a ledger entry against its collection; the money stays in the intake account until the collector releases it |
| AL-4 | Transfer money between collection accounts with a reason, an approval and matching ledger entries |
| AL-5 | When every obligation is met, refund the excess or hold it as credit, as the collector chooses |
| AL-6 | Reverse a wrong allocation with new ledger entries |
| AL-7 | Let the collector release money from the intake account to a collection account, in any amount up to that collection's unreleased balance and never more. Each release has one reference and idempotency key, needs second approval above the collector's limit, and counts as complete only when its transfer is confirmed |
| AL-8 | Show each collection's allocated, released and unreleased amounts, and compare the total unreleased amount with the intake account's provider balance |

**Payouts and refunds**

| ID | Requirement |
| --- | --- |
| PO-1 | Pay vendors and beneficiaries from a collection's own account, never from the intake account, with saved beneficiaries and an account-name check before sending |
| PO-2 | Require a second approver above a collector-set amount |
| PO-3 | Refund single or bulk payments to the sender account details on the original credit, or to a different account the sender provides with a screenshot, which needs approval |
| PO-4 | Close a collection: settle the balance, issue a final statement, close the account and archive it |

**Reporting and records**

| ID | Requirement |
| --- | --- |
| RP-1 | Dashboard showing the main balance, each collection's balance, today's inflow and total outstanding |
| RP-2 | Per-collection view of expected, received, outstanding and unmatched amounts |
| RP-3 | Statements per collection, sender and period, exported as CSV, Excel and PDF |
| RP-4 | Receipts to senders by WhatsApp, SMS or email after a confirmed payment |
| RP-5 | Daily check of intake and collection balances against provider balances |
| RP-6 | Daily close-out the collector approves in one step |
| RP-7 | Aging report of who owes what and for how long |
| RP-8 | Fee tracking that shows gross, provider fees and net per payment |
| RP-9 | Expense tracking with a category and receipt on every payout |
| RP-10 | Full reconciliation at three levels: intake account against ledger, each collection account against ledger, and the transfers between them |

**Communication**

| ID | Requirement |
| --- | --- |
| CM-1 | Send reminders before and after due dates by WhatsApp, SMS or email, with wording and timing the collector controls |
| CM-2 | Notify the collector of each credit or send a daily digest |

**Access and trust**

| ID | Requirement |
| --- | --- |
| AC-1 | Collector sign-up with KYC; authentication is built in Python, in the middleware service, and the method is still to be chosen |
| AC-2 | Append-only audit log of every action, with the actor and time |
| AC-3 | Team roles with limited permissions |
| AC-4 | Optional read-only share link showing progress, off by default; senders still need no login |

## AI features

AI suggests and explains; deterministic rules decide who owes what and where money moves. The AI service (Python) reads data only through an MCP server, which exposes read-only tools over the ledger database, and returns suggestions with a confidence and a reason. It never writes to the ledger or starts a transfer.

**Name matching pipeline**

Because every sender pays the same intake account, the sender's name on the credit is the main link to a person. Matching runs in layers:

1. Deterministic rules: an exact match on a remembered bank account number, then a normalised name comparison.
2. A fuzzy score for near matches, such as reordered names, truncation and initials.
3. An LLM only for the ambiguous middle band. It returns a suggested sender, a confidence and a one-line reason.
4. The collector confirms anything below a high-confidence threshold. A confirmed match is remembered (SR-3).

**Features**

| Feature | What it does |
| --- | --- |
| Name matching | The pipeline above, with the sender registry as the candidate list |
| Allocation explanation | Gives a plain-language reason for each split, for example which collections a payment covered and why, per the collector's priority order |
| Discrepancy explainer | Explains short or odd credits, such as a small shortfall that looks like a transfer fee |
| Third-party payment detection | Flags credits where someone appears to be paying for another sender |
| Narration parsing | Reads the transfer narration for hints about the sender or the installment |
| Anomaly detection | Flags duplicates, odd amounts, credits left unallocated too long, and an intake balance that disagrees with the ledger |
| Short-payment suggestions | Proposes how to split a payment that does not cover everything, for the collector to approve |
| Default-risk prediction | Flags senders likely to miss installments so the collector can step in early |
| Reminder drafting | Writes personalised reminders in English or Pidgin, escalating by lateness |
| Failed-transfer diagnosis | Classifies why a release transfer failed and suggests the next step |
| Natural-language setup | Turns a typed description of a collection into a filled form for review |
| Ask your data | Answers questions such as who has not paid, using read-only queries through the MCP server |
| Weekly summaries | Writes a plain-language status for each collection |
| Invoice reading | Extracts vendor, amount and date from an uploaded invoice for expense logging |

**Guardrails**

- Only deterministic code writes allocations. The AI proposes.
- The MCP server exposes read-only tools only, runs on a read-only database role, and scopes every query to one collector. It has no tool that writes, releases or pays.
- A doubtful match is never auto-confirmed.
- A screenshot or a sender's claim never marks anyone as paid. Only a confirmed credit counts.
- Names and account numbers are masked before data goes to an external model wherever possible; the MCP server is the place to apply that masking.
- Each suggestion shows its reason to the collector, and both the suggestion and the decision are logged.
- Matching is evaluated on a labelled test set, tracking accuracy and cost per call.
- Risk scores are tested for bias, and the collector can see why a sender was flagged.

## Payment engine (API for other apps)

Collection Point can also run as a payment engine: another app, the host app, calls an API to create collections, show its users account details, and learn when money is confirmed, while Collection Point does the matching, allocation, ledger and reconciliation. This section is a proposal for review; none of it is decided.

**Principle.** The dashboard and the API are two front doors to the same core. The dashboard is built on the API from the first release, so the engine adds no second code path for money.

**Who is who**

| Term | Meaning |
| --- | --- |
| Host app | The third-party product that integrates, such as a school-fees app or a church-dues app. It holds API keys and never holds money |
| Collector (tenant) | The host app's customer who receives the money, such as one school. Still the legal holder of funds and still completes KYC |
| Sender | Unchanged: pays by bank transfer and never logs in, and never sees Collection Point |

One host app can own many collectors. Every API call is scoped to one collector, and data never crosses collectors.

**How a host app uses it**

1. The host app creates a collector, which starts KYC, then creates collections, senders and obligations, each tagged with the host's own reference ID.
2. Collection Point returns account details. The host app shows them to its user in its own screens.
3. The sender pays by bank transfer. Collection Point matches and allocates the money in the ledger exactly as in Core flows.
4. Collection Point sends the host app a signed webhook. The host app marks an invoice paid only when it receives a confirmed allocation event, never on a sender's claim or screenshot.
5. Doubtful credits go to the review queue, which the host app can embed through the API or leave to Collection Point's own hosted screen.

**Requirements**

| ID | Requirement |
| --- | --- |
| PE-1 | Host app accounts with API keys, in separate test and live modes. Keys are scoped to one host app, shown once, stored hashed, and rotatable without downtime |
| PE-2 | Create and update collectors, collections, senders and obligations through the API. Every object takes the host's own reference ID, unique per host app |
| PE-3 | Every write call takes an idempotency key, so a retried request never creates a second collection or moves money twice |
| PE-4 | Outbound webhooks for payment received, matched, needs review, allocation recorded, release completed, release failed, payout completed and refund completed. Each is signed, retried with backoff, replayable, and visible in a delivery log |
| PE-5 | Read endpoints for balances, collection status, obligations, payments and statements, with pagination and filters |
| PE-6 | Review-queue endpoints so a host app can confirm, reassign or reject a match. Each decision records the host user as the actor |
| PE-7 | Payouts and refunds through the API under the same rules as the dashboard, including second approval above the collector's limit. Approvals are given by the collector, not by the host app's key alone |
| PE-8 | A sandbox that simulates provider credits, failures and duplicate webhooks, so a host app can test without real money |
| PE-9 | Per-key rate limits, usage metering per host app, and an audit entry for every call that names the key as the actor |
| PE-10 | A versioned API with a published OpenAPI description and a changelog. A breaking change ships as a new version and the old one keeps working for a stated period |
| PE-11 | Host-branded payment instructions, as a hosted page or embeddable component, so the host app need not build its own screen |

**Example endpoints**

| Resource | Example calls |
| --- | --- |
| Collectors | `POST /v1/collectors`, `GET /v1/collectors/{id}` |
| Collections | `POST /v1/collections`, `GET /v1/collections/{id}`, `POST /v1/collections/{id}/close` |
| Senders and obligations | `POST /v1/senders`, `PUT /v1/collections/{id}/obligations` |
| Payments | `GET /v1/payments`, `POST /v1/payments/manual` |
| Review items | `GET /v1/review-items`, `POST /v1/review-items/{id}/decision` |
| Balances and statements | `GET /v1/balances`, `GET /v1/statements` |
| Payouts and refunds | `POST /v1/payouts`, `POST /v1/refunds` |
| Webhook endpoints | `POST /v1/webhook-endpoints`, `GET /v1/webhook-deliveries` |

**What changes elsewhere in this document**

- **Authentication.** Authentication, built in the Python middleware, must now cover machine-to-machine access (API keys or client credentials) as well as collector login, and both must map to one audit identity.
- **Data model.** Add Host app, API key, Webhook endpoint and Webhook delivery, plus an optional host reference ID on collector, collection, sender and obligation.
- **Architecture.** The Python middleware exposes the public API. The Java gateway and the Python reporting service, AI service and MCP server stay internal and are never exposed to host apps directly.
- **Non-goals.** Collection Point still holds no funds in its own name. A host app that holds or routes money itself is out of scope.

**Phasing.** First release: the dashboard runs on the same internal API, with no external access. Next: keys, collector and collection endpoints, webhooks and the sandbox, for one design-partner host app. Later: payouts and refunds by API, the hosted payment page, metering and a public developer guide.

**Decisions needed before building the public API**

- Is the host app's customer the collector of record, or is the host app itself? This decides KYC, who the funds in Squad's accounts belong to, and whether a licence is needed.
- Can the provider issue accounts for many collectors under one integration, and at what cost?
- Who pays provider fees, and how does Collection Point charge host apps: per transaction, per collector, or a plan?

## Architecture

Five services work together: a small Java payment gateway talks to Squad, and four Python services (middleware, reporting, AI and an MCP server) do everything else. The Python middleware owns the ledger and is the only service that writes to it; the gateway, reporting, AI and MCP services never do. Messages between the gateway and the middleware pass through a RabbitMQ queue with a dead-letter queue for failures.

*Diagram (see the live document for the drawing): system architecture, 5 services, 2 external providers.* The collector dashboard reaches the Python middleware through authentication (built in Python, method to be chosen). The middleware service contains sender matching, ledger and allocation, the review queue, release and payout rules, reminders and receipts, and the audit log and public API. It exchanges data with the Java payment gateway, sends messages through the messaging providers, and writes to the ledger database. The gateway exchanges data with Squad and publishes verified credits to the queue for the middleware to consume. The reporting service (Python) reads the ledger database read-only. The AI service reaches the ledger database only through the MCP server, which runs read-only queries, and returns its insights and suggestions to the collector dashboard.

Collectors reach the middleware through authentication. Squad sends webhooks to the gateway, which verifies each one, records each credit once and passes it to the middleware through the queue. The middleware instructs releases and payouts; the gateway executes them against Squad and reports the result back. The reporting service and the MCP server read the ledger database read-only.

| Component | Owns |
| --- | --- |
| Payment gateway (Java, Spring Boot) | Receiving provider webhooks and verifying their signatures, recording each credit once (unique provider reference, replay protection) and passing it to the Python middleware, and executing transfers and releases against Squad with an idempotency key on every call, then reporting the result back. It holds no ledger and no matching rules. A mock Squad provider stands in until the real integration is confirmed |
| Middleware service (Python, FastAPI) | The ledger, sender matching rules, allocation, the review queue, release and payout rules and approvals, refunds, reminders and receipts, the audit log, and the public API. It is the only service that writes the ledger and the only one that instructs the gateway |
| Reporting service (Python) | Dashboard figures, statements and exports (CSV, Excel, PDF), aging and close-out reports, reconciliation against provider reports, and the summary endpoints |
| AI service (Python) | Match suggestions, explanations, anomaly flags, reminder drafts and summaries. It reads data only through the MCP server and returns suggestions with a confidence and a reason |
| MCP server (Python) | Read-only tools that give the AI service access to ledger data, scoped to one collector per call, with masking of names and account numbers where possible. It has no write, release or payout tools |
| Ledger database | Payments, allocations, transfers and ledger entries, written only by the Python middleware. A relational database is assumed, with PostgreSQL proposed |
| Payment provider (Squad) | Virtual accounts, incoming payment webhooks (received by the Java gateway), transfers between accounts, payouts and refunds. Squad by HabariPay is the primary provider, with one virtual account per collection |
| Messaging providers | Delivery of WhatsApp, SMS and email messages |

**Authentication (built in Python).** Authentication lives in the Python middleware service and must:

- Support collector sign-up and login. The collector is the only user type at first.
- Tie a collector's KYC status to their ability to receive money (AC-1).
- Give every audit entry an identity, including second approvers on payouts.
- Accept one identity across the Python middleware and the Python reporting service, so a single session works for both.
- Allow team roles with limited permissions later (AC-3).

Which method it uses (for example sessions or tokens) is still to be chosen before collector sign-up is built.

## Data model

The model centres on a payment, its allocations and an append-only ledger, so every naira can be traced from credit to collection.

| Entity | Key fields | Notes |
| --- | --- | --- |
| Collector | Name, contact, KYC status, priority order | The account owner |
| Sender | Collector, name, optional phone, known bank account numbers | One record per person per collector |
| Collection | Collector, purpose, target, deadline, mode, status, priority rank | Has one virtual account |
| Obligation | Sender, collection, amount owed, tier, installment schedule | What a sender owes in one collection |
| Virtual account | Owner (a collection or a collector's intake), account number, provider reference | Issued by the provider |
| Payment | Account, matched sender, amount, sender name, bank, account number, narration, fees, provider reference, match state | One per credit; the provider reference is unique |
| Allocation | Payment, obligation, amount, reason, status | One per split; a ledger entry that moves no money |
| Transfer | From account, to account, amount, reference, idempotency key, status | Created when the collector releases money; status is pending, succeeded or failed |
| Ledger entry | Account, amount, direction, link to the payment, allocation, transfer, payout or refund | Append-only |
| Payout | Collection, beneficiary, amount, approvals, status | Needs a second approver above the collector's limit |
| Refund | Payment, destination account (from the original credit, or provided and approved), amount, status | Single or bulk |
| Review item | Payment, suggested match, confidence, reason, decision, decided by | Created for doubtful matches |
| Audit event | Actor, action, target, time | Append-only |

**Payment states.** A payment moves from received to matched and then allocated; each allocated amount is later released in part or in full. It can also sit as needs review, unmatched, held for excess, or refunded. Each change is a ledger or audit entry, never an overwrite.

## Non-functional requirements

The platform must never lose, duplicate or silently misplace money, and every movement must be explainable afterwards.

**Money correctness**

- Every call that moves money carries an idempotency key and a unique reference, so a retry can never move money twice.
- A release completes only after its transfer is confirmed. A failed release changes nothing: the money stays in the intake account and the collection's unreleased balance is unchanged.
- The ledger and audit log are append-only. Corrections are reversing entries.

**Security**

- Verify the signature on every provider webhook, reject replays, and treat the provider reference as unique per payment.
- Collectors see only their own data. Payouts above a collector-set amount need a second approver.
- The MCP server connects with a read-only database role and cannot change data.
- No card data is stored, because cards are out of scope.
- Provider keys and other secrets are kept outside the code.

**Reconciliation**

- Check intake and collection balances against provider balances every day (RP-5), and run the three-level reconciliation described in RP-10. The intake account's provider balance must equal the sum of every collection's unreleased balance plus unmatched and held credits.
- Any difference opens an item for the collector or the platform team. None is ignored.

**Privacy and compliance**

- Sender names, phone numbers and bank details are personal data. Collect only what is needed, set a retention period, and restrict access. Confirm obligations under Nigerian data protection rules.
- Reminders and receipts need a lawful basis and a way for a sender to opt out.
- Before launch, confirm with the provider and a legal adviser the collector's KYC, how funds in Squad's virtual accounts are treated, and any rules on fees and disclosure.

**Performance**

Throughput and latency targets are not set yet. The webhook endpoint should acknowledge quickly and process each payment asynchronously, so a slow match never delays the provider.

## Risks, assumptions and open questions

The biggest risk is matching accuracy, because every sender pays one intake account and a wrong match can misallocate several collections at once.

| Risk | Why it matters | Mitigation |
| --- | --- | --- |
| Matching errors | One wrong match can touch several obligations | Layered matching, a review queue, doubtful credits held, an evaluation set |
| Webhook lacks sender details | Matching and refunds depend on sender name, bank and account number | Confirm with the provider first; otherwise ask the collector for refund details and verify the account name |
| Transfer cost, speed or limits | Every release is a real transfer, though only when the collector chooses | Confirm costs early. Allocation is ledger-only, so transfers happen only when the collector releases money |
| Failed release | A release transfer may fail or time out | A reference and idempotency key per release, safe retries, and an unreleased balance that changes only when the transfer is confirmed |
| Wrong amounts and third-party payments | The bank cannot cap what a sender transfers | Collector-set rules, the review queue, a refund workflow |
| Provider and regulatory limits | Squad's KYC, settlement and fee disclosure rules apply | Confirm with the provider and a legal adviser before launch |
| Personal data exposure | Sender lists hold names, phones and bank details | Access control, masking before external model calls, a retention period |
| MCP server exposes too much data to the model | The AI can query whatever its tools allow, so a broad tool or a missing collector filter could leak another collector's data | Read-only database role, per-collector scoping on every call, a small fixed set of tools, masking at the MCP layer, and logging of every tool call |
| Authentication method not chosen | It affects sign-up, approvals and audit identity | Choose the method before building collector sign-up, against the requirements in the Architecture section |
| Large balance in the intake account | All intake money stays in one Squad virtual account until released, so a ledger or provider error affects every collection at once | Daily check that unreleased balances equal the intake balance (RP-5), the release cap, and confirming with Squad how unreleased money in the intake account is treated before launch |
| Gateway and middleware out of step | A credit recorded by the Java gateway could be lost before the Python middleware has it, or a release could be executed twice | The gateway keeps each credit until the middleware acknowledges it, retries safely, and uses the provider reference and an idempotency key on every call; the daily balance check catches any gap |

**Assumptions**

- Squad by HabariPay is the primary provider, with one virtual account per collection. Its answers to the open questions below are not yet confirmed.
- The Java gateway keeps its own small store of received credits and transfer attempts, separate from the ledger database, and a mock Squad provider is used in development until Squad's answers to the open questions are confirmed.
- One intake account per collector can receive payments from many senders.
- The provider can issue a virtual account per collection at an acceptable cost.

**Open questions**

- [ ] Do Squad webhooks reliably include the sender's name, bank and account number?
- [ ] Can money move between our own Squad virtual accounts at all, and at what cost, speed and limits? Squad's docs say virtual account payments are credited to the merchant's GTBank account or wallet, which may not support the release design.
- [ ] Can one intake account receive high volumes from many senders, and are there account limits?
- [ ] Can a collector's balance across accounts be read in one place, or must we compute it?
- [ ] What are the refund and payout options, and their costs?
- [ ] Under Squad's terms, whose money are funds in the virtual accounts, and what KYC does the collector need?
- [ ] Which authentication method will the Python middleware use, for example sessions or tokens?
- [ ] What happens when money arrives in a closed collection's account?
- [ ] Which monetisation model do we choose: a fee on contributions or payouts, paid plans for large groups, or paid reminders? Who bears provider costs?
- [ ] Which fee rate applies to Collection Point? Squad's own pages show both 1.20% and 0.01%.

## Release phasing

The first release lets a collector run collections end to end: receive money, match it, allocate it and report on it. Payouts, refunds and fuller reconciliation follow.

| Phase | Scope | Requirements | AI |
| --- | --- | --- | --- |
| First release | Collector sign-up, collections and virtual accounts, sender registry, webhook intake, matching with a review queue, manual cash entry, priority allocation and release, reversals, dashboard, statements, receipts, daily balance check | AC-1, AC-2, CS-1 to CS-5, SR-1 to SR-3, PM-1 to PM-6, AL-1 to AL-3, AL-6 to AL-8, RP-1 to RP-5 | Name matching, allocation explanation, and the MCP server they read through |
| Next | Transfers between collections, excess handling, payouts with approvals, refunds, collection close-out, full reconciliation, fee and expense tracking, aging and close-out reports, bulk import, late joiners, reminders and credit notifications | AL-4, AL-5, PO-1 to PO-4, RP-6 to RP-10, CS-6, CS-7, CM-1, CM-2 | Discrepancy explainer, third-party detection, narration parsing, anomaly detection, short-payment suggestions, failed-transfer diagnosis, reminder drafting |
| Later | Recurring and cloned collections, templates, team roles, optional share link, accounting-software exports | CS-8, CS-9, AC-3, AC-4 | Default-risk prediction, natural-language setup, ask your data, weekly summaries, invoice reading |

**Gate before the first release:** answer the Squad questions on sender details in webhooks, transfers between accounts and intake limits, and choose the authentication method.

## Glossary

These terms mean the same thing in the code, the interface and conversations about the product.

| Term | Meaning |
| --- | --- |
| Collector | The only logged-in user. Sets up collections, receives money and legally holds the funds |
| Sender | Anyone who pays a collector. Never logs in |
| Collection | A pot of money for one purpose, with a target, deadline, mode and its own virtual account |
| Virtual account | An account number issued by Squad, the payment provider, to receive transfers. The account stays Squad's |
| Collection account | The virtual account that belongs to one collection |
| Intake account | The one virtual account per collector that receives payments for any of their collections and holds the money until the collector releases it to a collection account |
| Main balance | The sum of every collection's ledger balance, whether the money sits in the intake or a collection account, shown as one figure |
| Obligation | What one sender owes in one collection |
| Priority order | The collector's ranking of collections, used to split an intake payment |
| Waterfall | Filling obligations in priority order until the money runs out |
| Allocation | The part of a payment assigned to one obligation |
| Release | The transfer, started by the collector, that moves money from the intake account to a collection account, up to that collection's unreleased balance |
| Review queue | Credits whose sender could not be matched with confidence, waiting for the collector |
| MCP server | The internal read-only service that gives the AI service access to ledger data through a fixed set of tools |

## Decision log

These decisions came out of the design discussions in October 2026. Change one only on purpose, and record the change here.

| Decision | Why |
| --- | --- |
| Senders pay by bank transfer from their own banking apps, with no sender login | Collectors want the overview; senders should not need the platform |
| One virtual account per collection, not per member per collection | Per-member accounts would grow expensive |
| One intake account per collector, with the same details for every sender | Lets a sender with several obligations pay once |
| The collector's priority order decides allocation; senders do not choose amounts for now | Simpler for senders and keeps the rules in one place |
| Senders are identified by the name and details on the credit, with AI helping to check registered names | There are no per-sender accounts to tell payments apart |
| The collector is the only legal holder of funds | Collection Point does not hold money in its own name |
| Collectors can move money between collection accounts | Collectors need to rebalance between purposes |
| Refunds use the sender's account details, or an approved account the sender provides with a screenshot | Keeps refunds traceable while allowing for a changed account |
| AI suggests and explains; only deterministic code writes allocations | Protects the ledger from model errors |
| Java is limited to one small Spring Boot payment gateway; the middleware, reporting and AI services are Python (FastAPI) (23 Oct 2026) | Keeps the Java surface small and the ledger, matching and allocation logic in one language. Replaces the earlier plan for a Java middleware |
| Authentication is built in Python, in the middleware service; the method is still to be chosen (23 Oct 2026) | Keeps sign-in in the same language as the ledger and the other services. The method is settled before collector sign-up is built |
| The AI service reads data through a read-only MCP server instead of calling the middleware's APIs (26 Oct 2026) | Gives the AI one narrow, auditable set of read tools in a single place, and keeps it away from any write path. Replaces reading through the middleware |
| Squad by HabariPay is the primary payment provider, with one virtual account per collection | Squad offers virtual accounts through an API with a sandbox. Its dynamic accounts need an exact amount, so they may become an optional mode later. Webhook details and transfers between accounts are still to confirm |
| Allocation is ledger-only; money stays in the intake account until the collector releases it, up to that collection's unreleased balance (17 Oct 2026) | Avoids a real transfer for every payment, so fewer transfer costs and failures; the collector moves money only when needed. Replaces distribution on every payment |

## Acceptance criteria for the first release

The first release is done when each of these holds.

**Worked allocation example.** A sender owes three collections and pays ₦40,000 into the intake account. The payment is matched confidently, then split in the collector's priority order.

| Priority | Collection | Owed | Allocated | Still owed |
| --- | --- | --- | --- | --- |
| 1 | School fees | ₦30,000 | ₦30,000 | ₦0 |
| 2 | Levy | ₦20,000 | ₦10,000 | ₦10,000 |
| 3 | Party fund | ₦10,000 | ₦0 | ₦10,000 |

**Criteria**

1. **Confident match.** Given a registered sender with open obligations, when a credit arrives and matches confidently, then it is allocated in priority order, each allocation is recorded in its collection's ledger while the money stays in the intake account, and the ledger and receipt show it.
2. **Doubtful match.** Given a credit with an ambiguous sender name, when matching is not confident, then nothing is allocated, the credit appears in the review queue with a reason, and allocation happens only after the collector confirms.
3. **Short payment.** Given the example above, when the payment covers only part of what is owed, then allocation stops when the money runs out and the remaining obligations stay open.
4. **Excess payment.** Given every obligation is met, when money is left over, then it is refunded or held as credit under the collector's rule, and the choice is logged.
5. **Direct payment.** Given a credit arrives at a collection's own account, then it counts toward that collection only and does not trigger the priority split.
6. **Repeated webhook.** Given the provider sends the same webhook twice, then one payment is recorded.
7. **Failed release.** Given a release transfer fails, then the unreleased balance is unchanged, the release shows as failed, and a retry never moves money twice.
8. **Manual payment.** Given the collector logs a cash payment, then it is marked manual and counted in balances and statements.
9. **Correction.** Given a wrong allocation, then it is reversed with new ledger entries and the original stays in the history.
10. **Daily balance check.** Given the ledger and provider balances differ, then an item is raised for follow-up.
11. **Release cap.** Given a collection has ₦30,000 unreleased in the intake account, when the collector releases ₦20,000, then the transfer is confirmed and ₦10,000 stays releasable. A release of ₦40,000 is refused, and money allocated to other collections cannot be used for it.
