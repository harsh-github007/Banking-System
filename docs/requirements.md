# Requirements and acceptance scenarios

These rules specify the intended design. They are not implemented controls or passed runtime tests. The project covers one bank, internal same-currency transfers and branch/ATM workflows. External payment settlement, foreign exchange, tax, jurisdiction-specific compliance and production infrastructure are outside this model.

## Financial and workflow invariants

| ID | Rule | Design view |
| --- | --- | --- |
| R01 | Every monetary amount carries a currency and integer minor units; operation amounts are positive. Interest calculations use decimal arithmetic and a documented rounding policy. | Domain and ledger classes |
| R02 | Every committed journal has at least two positive postings; debits equal credits for each currency. A journal is immutable. Corrections create a linked compensating journal rather than editing history. | Ledger classes; transfer sequence |
| R03 | An internal transfer commits both postings, request result, audit event and outbox together. Both accounts use stable-order locking or an equivalent serializable concurrency guarantee. | Transfer sequence |
| R04 | Available funds equal posted balance plus approved overdraft minus active holds. Product policy, account status, signing rules and transaction limits apply before posting. | Account state; transfer sequence |
| R05 | Idempotency keys are unique per principal and operation type. Identical retries return the original result; changed payloads with the same key are rejected. Request result and financial posting cannot diverge. | Transfer sequence |
| R06 | ATM holds reserve funds before dispensing. Confirmed dispense settles once; confirmed non-dispense releases once. Ambiguous outcomes remain pending and enter reconciliation. | ATM communication and activity |
| R07 | Authentication and authorization are distinct. Ownership/signing rules are checked on each request; privileged staff permissions are scoped and auditable. Credential lockout does not freeze a bank account. | Use cases; authentication activity |
| R08 | Loan verification, approval and disbursement are separate recorded steps. An operator cannot approve a case they verified. Disbursement requires approval and creates balanced postings together with the Active state. | Loan sequence and state |
| R09 | A cheque cannot be both stopped and settled. Those transitions are serialized; externally uncertain settlement stays pending. | Cheque state |
| R10 | Fixed-deposit redemption applies its recorded term/policy version and posts a balanced journal. It is not an ordinary transfer from an unrestricted account. | Domain classes |
| R11 | An account closes only when its balance is zero and no holds, uncleared instruments or product obligations remain. Posted history remains accessible to authorized staff under retention policy. | Account state |
| R12 | Every sensitive attempt records actor, operation, correlation ID, outcome, reason and time. Audit records exclude credentials and sensitive document contents. Notifications retry independently using event IDs. | Ledger classes; components |

For a customer deposit liability account, a credit increases the customer's posted balance and a debit decreases it. For example, an internal transfer of 10,000 minor units debits the source deposit liability and credits the destination deposit liability by the same amount. The bank's cash control accounts are also represented in the ledger, so cash transactions do not create or destroy an unbalanced customer-only entry.

## Operational state vocabulary

Request status is `Pending`, `Posted` or `Rejected`. A pending request with an unknown external outcome is not reported as a rejection. A hold is `Active`, `Settled` or `Released`; settlement and release are mutually exclusive. Account, loan and cheque states are defined by their respective state diagrams. Card lifecycle is a separate `Active`, `Blocked` or `Expired` policy, independent of the associated account lifecycle.

## Authorization matrix

All staff permissions are branch-scoped unless an explicit exception grants a wider scope. Administrative access does not imply transaction or loan approval rights.

| Actor | Allowed operations | Required boundary |
| --- | --- | --- |
| Customer | Own-account enquiry, authorized transfers, product requests and complaints | Ownership, signing rule and additional authentication policy |
| Receptionist | Submit onboarding and loan applications; route complaints | Cannot approve loans or independently post cash |
| Cashier | Counter cash and cheque operations | Amount/account checks, cash reconciliation and assigned role |
| Accountant | Verify loan documents and request approved disbursement | Cannot approve the same case |
| Branch Manager | Approve/reject verified loans and authorize product/rate changes | Separation of duties, explicit reason and policy version |
| Administrator | Manage employee access | Cannot acquire financial approval authority through self-grant; privileged changes need separate authorization |

Counter cash has a physical reconciliation boundary as well. Count/verify cash before accepting deposits, serialize the financial posting, and match till/device evidence to the recorded operation. If cash handover and posting disagree, place the case into reconciliation; do not automatically repeat a withdrawal.

## Acceptance scenarios

These are implementation test specifications, not executable tests against a banking application.

| ID | Given / when | Expected result | Rules |
| --- | --- | --- | --- |
| A01 | Active same-currency accounts; permitted transfer within available funds | One journal, two balanced postings, one durable result and outbox event; source decreases and destination increases equally | R01–R05 |
| A02 | Same request key and identical payload retried after a lost response | Original result returned; no new journal or additional debit | R05 |
| A03 | Same request key reused with a different amount or destination | Explicit conflict; original operation remains unchanged | R05 |
| A04 | Two withdrawals race for funds sufficient for only one | At most one succeeds; available funds do not breach the permitted limit | R03–R04 |
| A05 | Failure after source-posting insertion but before transaction commit | Entire database transaction rolls back; neither account changes | R02–R03 |
| A06 | Commit succeeds but response or notification fails | Retry finds the committed result; notification can retry without reposting money | R05, R12 |
| A07 | Caller does not own the account or joint signing policy is unmet | Request rejected and audited before financial posting | R07, R12 |
| A08 | Zero/negative amount, currency mismatch, closed account or exceeded limit | Validation rejection; no journal created | R01, R04 |
| A09 | ATM confirms cash dispensed, then repeats confirmation | Hold settles and journal posts once | R05–R06 |
| A10 | ATM proves no cash was dispensed | Hold releases once; no cash-withdrawal journal | R06 |
| A11 | ATM times out with an unknown dispense outcome | Pending status and retained hold; evidence-based reconciliation, no automatic release or second dispense | R06 |
| A12 | Approver is also the verifier, or loan is not approved | Approval/disbursement rejected; reason and actor audited | R08, R12 |
| A13 | Disbursement retried after successful posting | One journal and one transition to Active; no double funding | R05, R08 |
| A14 | Cheque stop races with settlement | One valid serialized outcome; settled cheque cannot become stopped | R09 |
| A15 | Account closure requested with a hold or unsettled cheque | Closure blocked until obligations resolve | R11 |
| A16 | Posted journal needs correction | Original entry retained; authorized linked compensating entry balances independently | R02, R12 |
| A17 | Login policy threshold is reached | Identity session/credential access restricted; account lifecycle and funds unchanged | R07 |
| A18 | Early fixed-deposit redemption requested | Versioned policy, charges and rounding applied; resulting journal balances | R01–R02, R10 |

## Open decisions before implementation

- Define currencies, precision, interest accrual, rounding and posting cutoffs.
- Agree idempotency-key retention and recovery after an unknown commit outcome. Retention must exceed supported retry/reconciliation windows.
- Define signing rules, role scopes, privileged-change approval and loan approval limits.
- Define pending-hold escalation, device-journal evidence and reconciliation ownership; expiry alone is not proof that cash was not dispensed.
- Set account restrictions, cheque settlement integration, product maturity and overdue/recovery policies.
- Set privacy/retention, backup/restore, recovery objectives and operational monitoring for the intended jurisdiction and deployment.
