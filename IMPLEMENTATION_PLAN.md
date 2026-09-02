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
  - Metadata source, timestamp, worker identifier, transaction reference, integrity hash
- `VerificationJob`
  - Requested, processing, passed, failed, needs review
- `Payment`
  - Amount, currency, provider, authorization, transfer, refund status
- `AuditEvent`
  - Immutable record of important actions and external responses
- `Notification`

Use a server-generated opaque worker identifier in external metadata. Do not expose email addresses, legal names, or payment information to blockchain networks.

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

1. Backend creates a verification session.
2. A provider or loading network receives the external worker identifier.
3. Metadata is collected by an approved integration.
4. The integration submits evidence references and transaction IDs.
5. A worker process retrieves and normalizes the evidence.
6. Rules validate time, worker identity, task requirements, and data integrity.
7. Optional AI analysis produces a recommendation with confidence and explanation.
8. Low-confidence or failed cases go to manual review.
9. The final decision is written to the assignment and audit log.

Important safeguards:

- Never treat a blockchain transaction alone as proof that work was performed.
- Store provider name, network, transaction hash, block height, retrieval time, and evidence hash.
- Make verification reproducible and version the rules/model used.
- Avoid collecting precise location or behavioral data unless necessary and consented to.

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
- Provide data retention and deletion controls
- Address employment classification and wage/payment regulations
- Confirm money-transmission, KYC, AML, tax, and marketplace obligations
- Enforce the README’s prohibition on employing minors through age verification and eligibility rules
- Threat-model spoofed metadata, replayed transactions, account takeover, fraudulent employers, and payment disputes
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
6. Add evidence storage, verification jobs, retries, and manual review.
7. Integrate one real payment provider in sandbox mode.
8. Add one real metadata/blockchain provider.
9. Add compliance controls, security review, accessibility, and observability.
10. Pilot with synthetic jobs and a small approved user group before handling real funds.

The first concrete milestone should be a vertical slice: an employer creates a job, a worker accepts and submits it, a mock verifier returns a result, and a sandbox payment completes. That will validate the core architecture before investing in multiple blockchain networks or AI verification.
