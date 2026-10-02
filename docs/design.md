# Banking design views

These are the revised, canonical design views. Original StarUML files are preserved as historical source material; they have not been synchronized to this revision. Requirements and acceptance scenarios are in [requirements.md](requirements.md).

## Design at a glance

The design covers customer accounts, internal transfers, cash operations, ATM withdrawals, deposits, cheques and loans. One posting service owns financial journals; other workflows request postings rather than changing balances independently.

```mermaid
flowchart LR
    Channel[Bank channels] --> Access[Authenticate and authorize]
    Access --> Workflow[Account and payment workflows]
    Workflow --> Posting[Posting service]
    Posting --> Ledger[(Ledger and holds)]
    Ledger --> Outbox[Committed notifications]
    ATM[ATM outcome] --> Reconcile[Reconciliation]
    Reconcile --> Workflow
```

- **Transfers:** matching request keys return the same result; balanced postings and the result commit together.
- **ATM withdrawals:** reserve funds first. Settle a confirmed dispense, release a confirmed failure, and reconcile an uncertain outcome.
- **Loans:** document verification, approval and disbursement are separate steps.
- **Audit:** financial journals are immutable; corrections use reversals. Notifications follow a committed outbox.

Scope: one bank, internal same-currency transfers and one transactional database. This is a design study; the acceptance scenarios describe expected behavior for a future implementation.

---

## Detailed views

## Use cases

This is a use-case relationship map drawn with Mermaid flowchart syntax, not formal UML include/extend notation. Authentication is a precondition for protected actions; approval is not implicitly part of document verification.

```mermaid
flowchart LR
    customer[Customer]
    receptionist[Receptionist]
    cashier[Cashier]
    accountant[Accountant]
    manager[Branch Manager]
    admin[Administrator]
    subgraph Banking[Banking system boundary]
        onboarding[Open account and verify identity]
        balances[View balances and statements]
        transfer[Request internal transfer]
        cash[Record counter cash transaction]
        atm[Request ATM withdrawal]
        cheque[Issue or stop cheque and process clearing]
        deposit[Open or redeem fixed deposit]
        apply[Submit loan application]
        verify[Verify loan documents]
        decide[Approve or reject loan]
        fund[Disburse approved loan]
        complaint[Raise and resolve complaint]
        products[Set products and rates]
        access[Manage staff access]
    end
    customer --- onboarding & balances & transfer & atm & deposit & apply & complaint
    receptionist --- onboarding & apply & complaint
    cashier --- cash & cheque
    accountant --- verify & fund
    manager --- decide & products
    admin --- access
```

[Diagram source](diagrams/01-use-cases.mmd)

## Domain classes

Account ownership is explicit, including joint-account signing policy. Fixed deposits are separate term products rather than unrestricted withdrawal accounts. Product policies determine overdraft eligibility. Amounts use integer minor units and an explicit currency; rates use decimal arithmetic.

```mermaid
classDiagram
    direction LR
    class Customer {
        +String customerId
        +VerificationStatus identityStatus
    }
    class Account {
        +String accountId
        +String currency
        +AccountStatus status
        +long availableMinor
    }
    class AccountOwnership {
        +String ownershipId
        +String signingRule
    }
    class Product {
        +String productId
        +String productType
        +long overdraftLimitMinor
        +String currency
    }
    class FixedDeposit {
        +String depositId
        +long principalMinor
        +String currency
        +Date maturityDate
        +String redemptionPolicyVersion
    }
    class Loan {
        +String loanId
        +LoanStatus status
        +String currency
        +long principalMinor
        +Decimal annualRate
        +Integer termMonths
    }
    class Instalment {
        +Date dueDate
        +long principalMinor
        +long interestMinor
    }
    class Cheque {
        +String chequeId
        +String currency
        +long amountMinor
        +ChequeStatus status
    }
    class DebitCard {
        +String cardToken
        +String processorReference
        +CardStatus status
    }
    class Branch {
        +String branchId
    }
    class Employee {
        +String employeeId
        +String role
    }
    class Complaint {
        +String complaintId
        +String status
    }
    Customer "1" -- "0..*" AccountOwnership
    Account "1" -- "1..*" AccountOwnership
    Product "1" -- "0..*" Account
    Branch "1" -- "0..*" Account
    Branch "1" -- "0..*" Employee
    Customer "1" -- "0..*" FixedDeposit
    FixedDeposit "0..*" --> "1" Account : funding and payout
    Customer "1" -- "0..*" Loan
    Loan "1" *-- "0..*" Instalment
    Loan "0..*" --> "1" Account : disbursement
    Account "1" -- "0..*" Cheque
    Account "1" -- "0..*" DebitCard
    Customer "1" -- "0..*" Complaint
    Complaint "0..*" --> "0..1" Employee : assigned to
```

[Diagram source](diagrams/02-domain-classes.mmd)

## Ledger and request classes

Ledger accounts include customer deposit liabilities and bank control accounts. An entry contains at least two positive postings, balanced by currency. Statements are derived from committed postings. Audit events are separate from financial journals; requests can fail without creating a journal.

```mermaid
classDiagram
    direction LR
    class Account {
        +String accountId
        +String currency
    }
    class LedgerAccount {
        +String ledgerAccountId
        +String currency
        +String accountType
    }
    class JournalEntry {
        +String journalId
        +String operationId
        +DateTime postedAt
        +String reversalOf
    }
    class Posting {
        +String side
        +String currency
        +long amountMinor
    }
    class TransferRequest {
        +String requestId
        +String idempotencyKey
        +String payloadHash
        +RequestStatus status
    }
    class Hold {
        +String holdId
        +String currency
        +long amountMinor
        +HoldStatus status
    }
    class AuditEvent {
        +String eventId
        +String actorId
        +String correlationId
        +DateTime timestamp
    }
    class OutboxEvent {
        +String eventId
        +String operationId
        +String deliveryStatus
    }
    Account "0..1" -- "1" LedgerAccount : customer deposit liability
    JournalEntry "1" *-- "2..*" Posting
    LedgerAccount "1" -- "0..*" Posting
    TransferRequest "1" --> "0..1" JournalEntry : successful posting
    TransferRequest "0..*" --> "1" Account : source
    TransferRequest "0..*" --> "1" Account : destination
    Account "1" -- "0..*" Hold
    JournalEntry "1" -- "0..*" OutboxEvent
    TransferRequest "1" -- "1..*" AuditEvent
```

[Diagram source](diagrams/02-ledger-classes.mmd)

## Internal transfer sequence

Same-bank, same-currency transfers commit both postings, the request result, audit and outbox in one database transaction. Durable idempotency keys are scoped to principal and operation type. A concurrent duplicate waits for, or retrieves, the original result; a pending response does not create another transfer.

```mermaid
sequenceDiagram
    actor C as Customer
    participant Auth as Access Control
    participant T as Transfer Service
    participant DB as Ledger Database
    participant N as Notification Worker
    C->>Auth: Authenticated transfer request and idempotency key
    Auth->>Auth: Verify ownership, signing rule and permission
    Auth->>T: Authorized command
    T->>DB: Begin transaction, claim unique request key
    alt Existing key with identical payload
        DB-->>T: Stored request result
        T->>DB: End request transaction
        T-->>C: Return same result
    else Existing key with different payload
        T->>DB: Roll back
        T-->>C: Reject key reuse conflict
    else New request
        T->>DB: Lock both accounts in stable ID order
        T->>DB: Check active states, currency, limits and available funds
        alt Checks fail
            T->>DB: Persist rejected result and audit event, commit
            T-->>C: Rejected with reason
        else Checks pass
            T->>DB: Insert balanced journal postings, result, audit and outbox
            T->>DB: Commit all changes together
            DB-->>T: Durable result
            T-->>C: Posted with operation ID
            N->>DB: Read committed outbox events
            N-->>C: Deliver notification with event ID
        end
    end
    Note over T,DB: Unknown commit outcome: look up the same request key before retrying
    Note over N,C: Notification failure does not undo a committed transfer
```

[Diagram source](diagrams/03-transfer-sequence.mmd)

## Loan approval and disbursement

Verification, approval and disbursement are distinct actions. The verifier/disbursement operator cannot approve their own loan case. An approval does not promise that funds have been posted.

```mermaid
sequenceDiagram
    actor C as Customer
    participant R as Receptionist
    participant V as Accountant
    participant M as Branch Manager
    participant L as Loan Service
    participant P as Posting Service
    C->>R: Application and documents
    R->>L: Create application
    V->>L: Record verification outcome
    alt Documents incomplete
        L-->>C: Awaiting documents, request corrections
    else Documents verified
        M->>L: Approve or reject with actor and reason
        L->>L: Check authority and separation of duties
        alt Rejected
            L-->>C: Rejection with reason
        else Approved
            L-->>C: Approved terms, not yet funded
            V->>L: Authorized disbursement with idempotency key
            L->>P: Check approval, recipient and current conditions
            P->>P: Commit balanced journal, Active status and audit atomically
            P-->>C: Disbursed with schedule and operation ID
        end
    end
```

[Diagram source](diagrams/03-loan-sequence.mmd)

## ATM communication view

Numbered edges approximate a UML communication view. A database transaction cannot make physical cash dispensing atomic. Holds and evidence-based reconciliation handle that boundary.

```mermaid
flowchart LR
    C[Customer]
    ATM[ATM device and journal]
    Auth[Card authorization adapter]
    W[Withdrawal Service]
    DB[(Ledger and holds)]
    Rec[Reconciliation worker]
    C -->|1. Card and protected PIN entry| ATM
    ATM -->|2. Protected authentication exchange| Auth
    Auth -->|3. Authorized session or rejection| ATM
    ATM -->|4. Withdrawal with unique device operation ID| W
    W -->|5. Reserve available funds and audit| DB
    W -->|6. Authorized amount and hold ID| ATM
    ATM -->|7. Dispense attempt and durable outcome| ATM
    ATM -->|8. Confirmed result; retry same operation ID| W
    W -->|9. Settle journal or release hold| DB
    ATM -->|10. Receipt or pending status| C
    Rec -->|11. Query uncertain device outcomes| ATM
    Rec -->|12. Resolve held request using evidence| W
```

[Diagram source](diagrams/04-atm-communication.mmd)

## Account state machine

Account status gates allowed operations. Available balance includes active holds and applicable overdraft policy. Authentication lockout does not change the account lifecycle.

```mermaid
stateDiagram-v2
    [*] --> PendingVerification
    PendingVerification --> Active: identity and product checks complete
    PendingVerification --> Rejected: onboarding declined
    Active --> Restricted: authorized restriction
    Restricted --> Active: authorized release
    Active --> Closing: closure requested
    Closing --> Active: closure cancelled
    Closing --> Closed: zero balance and no unresolved holds or obligations
    Rejected --> [*]
    Closed --> [*]
    note right of Restricted
        Allowed credits and blocked debits
        depend on the recorded restriction policy.
        Login lockout is a separate identity state.
    end note
```

[Diagram source](diagrams/05-account-states.mmd)

## Loan state machine

Overdue thresholds are configured product policies. Write-off requires an explicit decision and does not automatically forgive a debt or end collection. This is a design assumption, not a regulatory rule.

```mermaid
stateDiagram-v2
    [*] --> Applied
    Applied --> AwaitingDocuments: documents incomplete
    AwaitingDocuments --> UnderReview: documents verified
    Applied --> UnderReview: documents verified
    UnderReview --> Approved: authorized approval
    UnderReview --> Rejected: authorized rejection
    Approved --> Active: successful disbursement posting
    Approved --> Cancelled: approval expires or customer declines
    Active --> Overdue: scheduled payment missed under configured policy
    Overdue --> Active: arrears settled
    Overdue --> WrittenOff: explicit authorized accounting decision
    Active --> Closed: all amounts due settled
    WrittenOff --> Recovery: recovery workflow initiated
    Recovery --> Closed: recovery or approved settlement complete
    Rejected --> [*]
    Cancelled --> [*]
    Closed --> [*]
```

[Diagram source](diagrams/05-loan-states.mmd)

## Cheque state machine

Stop and clearing decisions are serialized against the same cheque record. External settlement needs reconciliation; an ambiguous outcome remains pending instead of being reported as cleared or returned.

```mermaid
stateDiagram-v2
    [*] --> Issued
    Issued --> Stopped: stop accepted before settlement
    Issued --> Presented: received for clearing
    Presented --> Returned: invalid details or unavailable funds
    Presented --> SettlementPending: checks pass and funds reserved
    SettlementPending --> Cleared: settlement posting confirmed
    SettlementPending --> Returned: confirmed settlement failure
    SettlementPending --> Stopped: stop wins serialized race before settlement
    Cleared --> [*]
    Returned --> [*]
    Stopped --> [*]
    note right of SettlementPending
        Stop and settlement must be serialized.
        A paid cheque cannot be marked stopped.
    end note
```

[Diagram source](diagrams/05-cheque-states.mmd)

## ATM withdrawal activity

A timeout is not proof of dispense failure. Retain the pending hold until device evidence resolves it, with an escalation policy for unresolved items. Notification and receipt failures cannot cause a second financial posting.

```mermaid
flowchart TD
    S([Start]) --> A[Authorize card session]
    A --> V{Authorized?}
    V -->|No| R[Reject and audit]
    V -->|Yes| Q[Validate amount, limits and active account]
    Q --> H{Available funds and ATM cash?}
    H -->|No| R
    H -->|Yes| Hold[Create hold and durable request]
    Hold --> D[Request device dispense]
    D --> O{Outcome proven?}
    O -->|Cash dispensed| P[Post balanced journal and settle hold atomically]
    O -->|No cash dispensed| Release[Release hold idempotently]
    O -->|Timeout or ambiguous| Pending[Keep pending hold and investigate]
    Pending --> Reconcile[Compare device journal and request record]
    Reconcile --> O
    P --> Receipt[Return posted receipt]
    Release --> Failed[Return failed receipt]
    R --> E([Finish])
    Receipt --> E
    Failed --> E
```

[Diagram source](diagrams/06-atm-activity.mmd)

## Authentication activity

Rate limits, lock duration and extra authentication are policy settings. The old fixed three-attempt account lock is replaced by identity-level controls and recovery. No password or PIN material belongs in logs.

```mermaid
flowchart TD
    S([Start]) --> Cred[Submit credentials]
    Cred --> Rate{Rate limit or identity lock?}
    Rate -->|Yes| Reject[Reject attempt without revealing identity existence]
    Rate -->|No| Verify[Verify credentials through identity service]
    Verify --> Valid{Valid?}
    Valid -->|No| Count[Record failed attempt and audit]
    Count --> Policy{Configured threshold reached?}
    Policy -->|Yes| Lock[Temporary identity lock and recovery process]
    Policy -->|No| Retry[Allow policy-controlled retry]
    Valid -->|Yes| MFA[Perform required additional authentication]
    MFA --> Passed{Successful?}
    Passed -->|No| Reject
    Passed -->|Yes| Session[Create short-lived session]
    Session --> Service[Authorize each service request]
    Service --> Logout[Logout or session expiration]
    Logout --> E([Finish])
    Reject --> E
    Lock --> E
    Retry --> E
```

[Diagram source](diagrams/06-login-activity.mmd)

## Component view

Choose a modular application with one transactional ledger database for this teaching design. Authorization is enforced by services as well as the gateway. Only PostingService creates financial journals; workflows persist their own state via repositories.

```mermaid
flowchart LR
    Channels[Existing branch, customer and ATM channels]
    Gateway[Access gateway]
    Identity[Identity and authorization]
    subgraph App[Modular banking application]
        Customer[Customer and onboarding]
        Accounts[Accounts and products]
        Payments[Transfers and withdrawal orchestration]
        Loans[Loan workflow]
        Cheques[Cheque workflow]
        Support[Complaints]
        Posting[Posting service]
        Persistence[Transactional persistence]
    end
    DB[(Relational ledger database)]
    Outbox[Outbox notification worker]
    Reconcile[Reconciliation worker]
    Channels --> Gateway
    Gateway --> Identity
    Gateway --> Customer & Accounts & Payments & Loans & Cheques & Support
    Payments & Loans & Cheques & Accounts --> Posting
    Customer & Accounts & Payments & Loans & Cheques & Support & Posting --> Persistence
    Persistence --> DB
    DB --> Outbox
    Reconcile --> Payments & Cheques
```

[Diagram source](diagrams/07-components.mmd)

## Deployment view

This is a logical topology, not a deployment manifest or availability claim. Redundancy, failover, backup restoration, key management and recovery objectives require separate infrastructure decisions.

```mermaid
flowchart LR
    Staff[Branch workstation]
    Customer[Customer device]
    ATM[ATM device]
    Edge[Private or public channel gateway]
    Adapter[Card network protocol adapter]
    App[Banking application process]
    Id[Identity service]
    DB[(Primary relational database)]
    Workers[Notification and reconciliation workers]
    Backup[(Encrypted backup storage)]
    Staff -->|TLS over bank network| Edge
    Customer -->|TLS| Edge
    ATM -->|Protected card-network channel| Adapter
    Adapter --> Edge
    Edge -->|Authenticated internal channel| App
    App --> Id
    App -->|TLS with restricted service identity| DB
    Workers --> DB
    DB --> Backup
```

[Diagram source](diagrams/08-deployment.mmd)

## Package dependencies

Domain classes do not depend on persistence or channel code. Infrastructure implements application ports. Existing channels are external context; this repository adds no frontend implementation.

```mermaid
flowchart TD
    Channels[Channel adapters]
    Application[Application services]
    Domain[Domain model and policies]
    Ports[Repository and event ports]
    Infrastructure[Infrastructure implementations]
    Channels --> Application
    Application --> Domain
    Application --> Ports
    Infrastructure --> Ports
    Infrastructure --> Domain
    note[Composition root wires implementations to ports]
    note -.-> Application & Infrastructure
```

[Diagram source](diagrams/09-packages.mmd)
