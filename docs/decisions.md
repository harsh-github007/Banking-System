# Design decisions

## 1. One transactional ledger boundary

Use a modular application and one relational ledger database for this project. An internal transfer's postings, request result, audit and outbox commit together. Stable-order account locking or an equivalent serializable strategy prevents concurrent overspending.

This is simpler to reason about than separate account-writing services. Notifications are eventually delivered and deduplicated by event ID; they do not determine financial success. Cross-bank transfers and foreign exchange require a different settlement model and are outside scope.

## 2. Immutable journals instead of editable transaction history

A posted entry is not updated or deleted. Corrections are authorized, linked compensating entries. Deposit balances derive from credits minus debits; available funds additionally include holds and authorized overdraft.

Example, using minor units in one currency:

| Event | Debit | Credit |
| --- | --- | --- |
| Transfer 10,000 from A to B | Customer deposit liability A: 10,000 | Customer deposit liability B: 10,000 |
| Cash deposit of 10,000 to A | Bank cash asset: 10,000 | Customer deposit liability A: 10,000 |
| Cash withdrawal of 10,000 from A | Customer deposit liability A: 10,000 | Bank cash asset: 10,000 |

These examples omit fees, taxes and interest. Those introduce additional postings; they do not relax balancing.

## 3. Idempotency is stored with the financial result

Scope keys by principal and operation type; store a hash of all financially relevant request fields. Identical retries reuse the original result, including after a timeout. Reusing a key with changed fields is an error. A unique constraint and transaction isolation serialize competing requests.

If the outcome of a commit is unknown, query the same key. Never submit a new key solely because the previous response was lost. Key expiry and archival policy must be agreed before implementation.

## 4. Physical cash requires reconciliation

An ATM cannot participate atomically in the bank's database transaction. Reserve funds, persist the request and device operation identity, and settle or release only after evidence. Unknown dispense outcomes remain pending; automatic release could let a customer keep dispensed cash without a debit.

Reconciliation compares the durable device journal with the bank request. Confirmed outcomes are handled idempotently. Unresolved cases need operational escalation, not endless retries or blind hold expiry. Counter cash has a similar evidence and till-reconciliation boundary.

## 5. Product and identity policies are explicit

A fixed deposit is a term product with a redemption policy; it is not an unrestricted Account subtype. Current-account overdrafts are product policies. Joint account ownership carries a signing rule. Cheque stop/settlement races require one serialized outcome.

Loan approval and funding are separate. Write-off is an explicit accounting action with a possible recovery workflow, not an automatic consequence of a universal number of overdue days. Credential lockout belongs to identity access; financial restrictions belong to the account.

## 6. Original coursework remains historical

The revised Mermaid documents are canonical for this revision. Preserve the original Word and StarUML assets without silently rewriting their structure or claiming they match the new design. A later native StarUML revision can use the requirements and sources here as its specification.
