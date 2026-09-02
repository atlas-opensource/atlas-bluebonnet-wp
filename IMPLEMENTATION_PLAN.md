# Implementation Plan

The current prototype in [main.code](main.code) already sketches worker registration, job acceptance/completion, Firestore listeners, simulated metadata, and market data. The production app should evolve those flows into a secure backend-driven system described in [README.md](README.md).

## 1. Define the product boundary

Build an initial MVP around:

- Employers creating work requests
- Workers registering and setting availability
- A trusted server assigning work
- Workers accepting and completing assignments
- External verification of completion
- Employer approval and worker payout
- Notifications and audit history

Defer direct blockchain payments. Blockchain data should initially be used as verification evidence, while fiat payouts are handled through a regulated payment provider.

## 2. Establish the architecture

Recommended first version:

- Worker web/mobile client
- Employer web dashboard
- Backend API for all state transitions
- PostgreSQL for users, jobs, verification, payments, and audit records
- Background job queue for blockchain lookups, AI analysis, notifications, and payment retries
- Object storage for large metadata evidence
- Provider adapters for:
  - Blockchain/indexing services
  - Market data
  - Payment providers
  - Email/SMS/push notifications

The loading network is outside the app's trust boundary. It is a separate legal and technical entity operating where the worker performs the task. Its equipment collects worker or employee information and connects independently to the WAN. Neither the app frontend nor the app backend has network access, credentials, administrative control, command capability, or a direct integration with the loading network.

The loading network may independently publish signed or otherwise attributable records to a public blockchain, an indexing service, or another WAN-accessible data provider. The app backend can only query those externally published records through read-only provider adapters, subject to the provider's access terms. It must not assume that a record exists, is complete, or was produced honestly merely because it is publicly visible.

The client should never directly mutate job status, payment data, or verification results in Firestore. The server should authorize and record every transition.

## 3. Model the core domain

Key entities:

- `User`
  - Role: worker, employer, administrator
  - Identity and verification status
- `WorkerProfile`
  - Availability, location permissions, payout account status
- `EmployerProfile`
  - Organization information and billing status
- `WorkRequest`
  - Description, required hours, location, rate, eligibility, expiration
- `Assignment`
  - Worker, employer, acceptance, start, completion, and current status
- `EvidenceRecord`
  - External source, timestamp, opaque worker identifier, transaction reference, integrity hash, retrieval details
- `VerificationJob`
  - Requested, processing, passed, failed, needs review
- `Payment`
  - Amount, currency, provider, authorization, transfer, refund status
- `AuditEvent`
  - Immutable record of important actions and external responses
- `Notification`

Use a server-generated opaque worker identifier for the assignment and make it available to the worker and employer under the product's privacy rules. The loading network is responsible for deciding how it associates that identifier with its independently collected records; the app does not transmit commands or metadata to it. Do not expose email addresses, legal names, or payment information to the loading network, blockchain networks, or public data providers.

## 4. Define the assignment state machine

Use explicit, validated transitions:

```text
DRAFT
  -> OPEN
  -> ASSIGNED
  -> ACCEPTED
  -> IN_PROGRESS
  -> SUBMITTED
  -> VERIFYING
  -> VERIFIED
  -> PAYMENT_PENDING
  -> PAID
```

Failure and dispute paths:

```text
OPEN -> EXPIRED
ASSIGNED -> DECLINED
SUBMITTED -> VERIFICATION_FAILED
VERIFICATION_FAILED -> DISPUTED
PAYMENT_PENDING -> PAYMENT_FAILED
```

Each transition should include:

- Authorized actor
- Timestamp
- Request idempotency key
- Audit event
- Notification event where appropriate

The worker’s “complete” action should create a verification request, not immediately mark the job as successful.

## 5. Build the backend first

Initial API surface:

- `POST /auth/register`
- `GET /me`
- `PATCH /workers/me/availability`
- `PATCH /workers/me/payout-account`
- `POST /work-requests`
- `GET /work-requests`
- `POST /assignments/:id/accept`
- `POST /assignments/:id/start`
- `POST /assignments/:id/submit`
- `GET /assignments/:id`
- `GET /assignments/:id/verification`
- `POST /assignments/:id/dispute`
- `POST /payments/:id/authorize`
- `POST /payments/:id/payout`
- `GET /notifications`

Use authenticated sessions, role-based authorization, request validation, rate limiting, structured errors, and idempotency for all mutating operations.

## 6. Implement evidence and verification

When an assignment is accepted:

1. Backend creates a verification job and records the opaque assignment identifier.
2. The worker performs the task while the independent loading network operates at the work location and collects information under its own legal authority, policies, and technical controls.
3. The loading network independently connects to the WAN and may publish records containing the relevant opaque identifier to a blockchain or other external data provider. The app does not initiate, configure, monitor, or control this collection.
4. After the worker submits the assignment, a backend worker queries approved public or authorized indexing endpoints for records matching the assignment identifier and time window.
5. The backend stores the returned records and provenance, then normalizes them without altering the original evidence.
6. Rules validate source authenticity, identifier matching, timestamps, task requirements, data integrity, and the limitations of the external source.
7. Optional AI analysis produces a recommendation with confidence and explanation; it does not replace source validation or human review.
8. Low-confidence, unavailable, contradictory, or failed evidence goes to manual review rather than being treated as proof of misconduct or completion.
9. The final decision is written to the assignment and audit log.

Important safeguards:

- Never treat a blockchain transaction or loading-network record alone as proof that work was performed.
- Treat the loading network as an untrusted external source: record its legal entity, declared provenance, publication endpoint, and known limitations, but do not grant it app credentials or inbound access.
- Store provider name, network, transaction hash, block height, retrieval time, response metadata, and evidence hash.
- Preserve the original externally retrieved payload separately from normalized data and make verification reproducible by versioning the rules and model used.
- Handle missing, delayed, changed, or unavailable public records explicitly; absence of evidence must not silently become evidence of absence.
- Avoid collecting precise location or behavioral data in the app unless necessary and consented to. The loading network's collection requires separate notice, consent, and legal review by that entity.

## 7. Add payment processing

Start with one provider, preferably Stripe Connect or an equivalent marketplace product.

Flow:

1. Employer adds and verifies a payment method.
2. Employer funds or authorizes the job amount.
3. Verification passes.
4. Platform captures funds.
5. Worker receives payout through a connected account.
6. Webhooks update payment state.
7. Failed transfers enter a retry/manual-review queue.

Never store raw bank credentials, card numbers, or wallet private keys. Encrypt sensitive provider identifiers and retain only the minimum required data.

## 8. Build the user experiences

Worker app:

- Registration and identity status
- Availability
- Assignment inbox
- Assignment details and acceptance
- Active work session
- Completion submission
- Verification status
- Payout setup and payment history
- Notifications and disputes

Employer dashboard:

- Create and manage work requests
- View assignments
- See verification progress and evidence summary
- Approve, dispute, or request review
- Fund jobs and track payments
- View worker and job history

Admin/reviewer console:

- Verification failures
- Disputes
- Payment failures
- Provider outages
- Audit trail
- Manual override with mandatory reason

The crypto market feed should be a separate informational feature. It must use a real provider, show source and freshness, and never affect assignment verification or payment calculations unless that behavior is explicitly defined.

## 9. Security, privacy, and compliance

Before collecting real worker data or money:

- Define terms of service and privacy policy
- Establish worker consent for location/device metadata
- Clearly disclose that the app does not operate or control the loading network, what externally published records may be queried, and how those records affect review or payment
- Provide data retention and deletion controls
- Address employment classification and wage/payment regulations
- Confirm money-transmission, KYC, AML, tax, and marketplace obligations
- Enforce the README’s prohibition on employing minors through age verification and eligibility rules
- Threat-model spoofed or fabricated external records, replayed transactions, identifier collisions, provider compromise, inaccessible or withdrawn records, account takeover, fraudulent employers, and payment disputes
- Add encryption in transit and at rest, secret management, least-privilege access, and immutable audit logs

Legal review is required before launch because this system combines labor coordination, behavioral monitoring, verification, and payments.

## 10. Testing and operations

Test at several levels:

- Unit tests for state transitions and verification rules
- API authorization and validation tests
- Contract tests for blockchain, market, payment, and notification providers
- End-to-end tests for the complete worker/employer flow
- Fraud and replay tests
- Payment webhook retry tests
- Accessibility and mobile usability tests
- Load tests for metadata ingestion and verification queues

Operational requirements:

- Structured logs and correlation IDs
- Metrics for assignment, verification, and payment latency
- Alerts for provider failures and queue buildup
- Dead-letter queues and replay tooling
- Feature flags for provider integrations
- Database backups and migration strategy
- Staging environment with sandbox payment accounts

## Suggested delivery sequence

1. Convert the prototype into a real application structure and choose the frontend/backend stack.
2. Implement authentication, roles, PostgreSQL schema, and audit logging.
3. Implement employer work requests and worker assignment flows.
4. Add the assignment state machine and notifications.
5. Add a mock verification provider behind a stable adapter interface.
6. Add evidence storage, read-only external-provider verification jobs, retries, and manual review.
7. Integrate one real payment provider in sandbox mode.
8. Add one real public-ledger or indexing provider as a read-only evidence source; do not integrate directly with the loading network.
9. Add compliance controls, security review, accessibility, and observability.
10. Pilot with synthetic jobs and a small approved user group before handling real funds.

The first concrete milestone should be a vertical slice: an employer creates a job, a worker accepts and submits it, a mock read-only external evidence provider returns a result or an unavailable result, and a sandbox payment completes only for a verified case. That will validate the trust boundary and failure handling before investing in multiple blockchain networks or AI verification.
