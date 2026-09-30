# Banking System: UML Design

A UML design for the core system of a retail bank branch. It covers customers and staff, savings, current and fixed deposit accounts, cash and transfers, cheques, ATM withdrawals and loans. It was drawn for a university software engineering course in 2022 and revised in 2026, so that all nine diagrams describe the same system with the same names.

This is a design, not an implementation. Every diagram below is written in [Mermaid](https://mermaid.js.org/) and renders directly on GitHub. The class diagram is too large for GitHub's viewer, so it is shown as an image, with its Mermaid source underneath. The original StarUML files are kept in [`original/`](original/).

**Contents**

1. [Use case diagram](#1-use-case-diagram)
2. [Class diagram](#2-class-diagram)
3. [Sequence diagrams](#3-sequence-diagrams)
4. [Communication diagram](#4-communication-diagram)
5. [State machine diagrams](#5-state-machine-diagrams)
6. [Activity diagrams](#6-activity-diagrams)
7. [Component diagram](#7-component-diagram)
8. [Deployment diagram](#8-deployment-diagram)
9. [Package diagram](#9-package-diagram)

## Actors

| Actor | Role |
| --- | --- |
| Customer | Holds accounts, uses the counter, ATM and internet banking |
| Receptionist | First point of contact; checks slips and forwards requests |
| Cashier | Handles cash deposits, withdrawals and cheques at the counter |
| Accountant | Keeps account records, verifies loan documents |
| Branch Manager | Approves loans and sets branch products and rates |
| Administrator | Manages staff and system access |

## 1. Use case diagram

What each actor can do. Every staff and customer use case requires the user to be logged in (`include`); the ATM authenticates with card and PIN instead.

```mermaid
flowchart LR
    Customer(["👤 Customer"])
    Receptionist(["👤 Receptionist"])
    Cashier(["👤 Cashier"])
    Accountant(["👤 Accountant"])
    Manager(["👤 Branch Manager"])
    Admin(["👤 Administrator"])

    subgraph System["Banking System"]
        UC1(["Open account"])
        UC2(["Deposit or withdraw cash"])
        UC3(["Transfer funds"])
        UC4(["View balance and statement"])
        UC5(["Withdraw cash at ATM"])
        UC6(["Manage cheques"])
        UC7(["Open fixed deposit"])
        UC8(["Apply for loan"])
        UC9(["Verify loan documents"])
        UC10(["Approve or reject loan"])
        UC11(["Raise complaint"])
        UC12(["Manage customers"])
        UC13(["Manage employees"])
        UC14(["Set products and rates"])
        AUTH(["Log in"])
    end

    Customer --- UC1 & UC3 & UC4 & UC5 & UC7 & UC8 & UC11
    Receptionist --- UC1 & UC8 & UC11 & UC12
    Cashier --- UC2 & UC6
    Accountant --- UC9 & UC12
    Manager --- UC10 & UC14
    Admin --- UC13

    UC3 & UC4 & UC7 & UC11 -. include .-> AUTH
    UC8 -. include .-> UC9
    UC9 -. include .-> UC10
```

## 2. Class diagram

The domain model. `Account` is abstract, and every change to a balance is recorded as a `Transaction`, so the statement is always the full history.

![Class diagram](docs/class-diagram.png)

<details>
<summary>Mermaid source</summary>

The same diagram as text, also in [`docs/class-diagram.mmd`](docs/class-diagram.mmd).

```text
classDiagram
    direction TB
    class Bank {
        +name: String
        +openBranch(ifsc) Branch
    }
    class Branch {
        +ifsc: String
        +address: String
        +openAccount(customer, type) Account
    }
    class Customer {
        +customerId: String
        +name: String
        +address: String
        +aadhaarVerified: Boolean
        +accounts() List~Account~
    }
    class Employee {
        <<abstract>>
        +employeeId: String
        +name: String
    }
    class Receptionist
    class Cashier
    class Accountant
    class BranchManager {
        +approve(loan)
        +reject(loan, reason)
    }
    class Account {
        <<abstract>>
        +accountNumber: String
        +balance: Decimal
        +status: AccountStatus
        +deposit(amount) Transaction
        +withdraw(amount) Transaction
        +transferTo(account, amount) Transaction
        +statement(from, to) List~Transaction~
    }
    class SavingsAccount {
        +interestRate: Decimal
        +accrueInterest() Transaction
    }
    class CurrentAccount {
        +overdraftLimit: Decimal
    }
    class FixedDeposit {
        +principal: Decimal
        +interestRate: Decimal
        +termMonths: Integer
        +maturityValue() Decimal
        +closePrematurely() Transaction
    }
    class Transaction {
        +transactionId: String
        +type: TransactionType
        +amount: Decimal
        +balanceAfter: Decimal
        +timestamp: DateTime
    }
    class ChequeBook {
        +firstNumber: Integer
        +leaves: Integer
    }
    class Cheque {
        +number: Integer
        +payee: String
        +amount: Decimal
        +status: ChequeStatus
        +stop()
    }
    class DebitCard {
        +cardNumber: String
        +expiry: Date
        +pinHash: String
    }
    class Loan {
        +loanType: LoanType
        +principal: Decimal
        +annualRate: Decimal
        +termMonths: Integer
        +status: LoanStatus
        +emi() Decimal
        +schedule() List~Instalment~
    }
    class Complaint {
        +raisedOn: Date
        +description: String
        +resolved: Boolean
    }
    class LoanType {
        <<enumeration>>
        CAR
        HOME
        GOLD
        EDUCATION
    }

    Bank "1" *-- "1..*" Branch
    Branch "1" o-- "1..*" Employee
    Branch "1" -- "*" Account : holds
    Employee <|-- Receptionist
    Employee <|-- Cashier
    Employee <|-- Accountant
    Employee <|-- BranchManager
    Customer "1..*" -- "*" Account : owns
    Account <|-- SavingsAccount
    Account <|-- CurrentAccount
    Account <|-- FixedDeposit
    Account "1" *-- "*" Transaction
    Account "1" -- "*" ChequeBook
    ChequeBook "1" *-- "1..*" Cheque
    Account "1" -- "0..2" DebitCard
    Customer "1" -- "*" Loan : borrows
    Loan ..> LoanType
    BranchManager "1" -- "*" Loan : approves
    Customer "1" -- "*" Complaint : raises
    Employee "1" -- "*" Complaint : resolves
```

</details>

## 3. Sequence diagrams

**Cash deposit or withdrawal at the counter.** The receptionist checks the slip before it reaches the cashier, and a withdrawal is refused if the balance is too low.

```mermaid
sequenceDiagram
    actor C as Customer
    participant R as Receptionist
    participant K as Cashier
    participant A as Account
    C->>R: Hand in deposit or withdrawal slip
    R->>R: Check slip details
    alt Details wrong
        R-->>C: Return slip for correction
    else Details correct
        R->>K: Forward slip
        alt Deposit
            C->>K: Hand over cash
            K->>A: deposit(amount)
            A-->>K: Transaction
        else Withdrawal
            K->>A: withdraw(amount)
            alt Sufficient balance
                A-->>K: Transaction
                K-->>C: Pay out cash
            else Insufficient balance
                A-->>K: Refused
                K-->>C: Inform customer
            end
        end
        K-->>C: Updated passbook entry
    end
```

**Loan application.** The accountant verifies documents; only the branch manager can approve.

```mermaid
sequenceDiagram
    actor C as Customer
    participant R as Receptionist
    participant Ac as Accountant
    participant M as Branch Manager
    participant L as Loan
    C->>R: Apply for loan with documents
    R->>Ac: Forward application
    Ac->>Ac: Verify documents and income
    alt Documents incomplete
        Ac-->>C: Request missing documents
    else Documents complete
        Ac->>M: Forward for approval
        M->>M: Check eligibility and exposure
        alt Approved
            M->>L: approve(loan)
            L-->>Ac: Status APPROVED, EMI schedule
            Ac-->>C: Sanction letter and disbursal
        else Rejected
            M->>L: reject(loan, reason)
            M-->>C: Rejection with reason
        end
    end
```

## 4. Communication diagram

**Cash withdrawal at an ATM.** The numbers give the order of messages between the customer, the ATM and the bank's server. Mermaid has no communication-diagram type, so this is drawn as a graph with numbered edges.

```mermaid
flowchart LR
    C(["👤 Customer"])
    ATM["ATM"]
    S["Bank server"]
    A["Account"]
    C -- "1: insert card, 3: enter PIN, 6: enter amount" --> ATM
    ATM -- "2: read card, 4: verify PIN, 7: request withdrawal" --> S
    S -- "5: PIN valid, 10: approved" --> ATM
    S -- "8: check balance, 9: withdraw(amount)" --> A
    ATM -- "11: dispense cash, 12: print balance, 13: return card" --> C
```

## 5. State machine diagrams

**Cheque.** A cheque deposited for clearing is either paid or returned.

```mermaid
stateDiagram-v2
    [*] --> Received
    Received --> VerifyingDetails: read payee, amount, date, signature
    VerifyingDetails --> Returned: details invalid
    VerifyingDetails --> CheckingFunds: details valid
    CheckingFunds --> Returned: insufficient funds or stopped
    CheckingFunds --> Cleared: funds available
    Cleared --> [*]: amount credited to payee
    Returned --> [*]: returned to depositor with reason
```

**Loan.** A loan moves through application, approval and repayment; a missed payment makes it overdue until it is brought current or written off.

```mermaid
stateDiagram-v2
    [*] --> Applied
    Applied --> UnderReview: documents verified
    Applied --> Rejected: documents incomplete
    UnderReview --> Approved: manager approves
    UnderReview --> Rejected: manager rejects
    Approved --> Active: amount disbursed
    Active --> Overdue: EMI missed
    Overdue --> Active: arrears paid
    Overdue --> WrittenOff: unpaid 90+ days
    Active --> Closed: final EMI paid
    Rejected --> [*]
    Closed --> [*]
    WrittenOff --> [*]
```

## 6. Activity diagrams

**Internet banking session.** A user gets three login attempts before the account is locked.

```mermaid
flowchart TD
    start((" ")) --> login[Enter customer ID and password]
    login --> ok{Valid?}
    ok -- no --> tries{Third failure?}
    tries -- no --> login
    tries -- yes --> locked[Lock account and notify customer] --> stop
    ok -- yes --> menu{Choose service}
    menu -- enquiry --> enq{Balance or statement?}
    enq --> bal[View balance] --> more
    enq --> stmt[Download statement] --> more
    menu -- transfer --> tr{Own account or payee?}
    tr --> own[Transfer to own account] --> more
    tr --> payee[Pay a registered payee] --> more
    menu -- cheques --> ch{Request or stop?}
    ch --> book[Request cheque book] --> more
    ch --> stopc[Stop a cheque] --> more
    more{Another service?} -- yes --> menu
    more -- no --> logout[Log out] --> stop((("●")))
```

**Loan application.** Document checks run in parallel; the loan is priced only once all are complete.

```mermaid
flowchart TD
    s((" ")) --> type{Loan type}
    type --> car[Car] & home[Home] & gold[Gold] & edu[Education]
    car & home & gold & edu --> apply[Submit application]
    apply --> fork[/"fork"/]
    fork --> kyc[Verify Aadhaar and PAN] & addr[Verify proof of residence] & inc[Verify income proof]
    kyc & addr & inc --> join[/"join"/]
    join --> ok{All verified?}
    ok -- no --> ret[Return to customer] --> e
    ok -- yes --> decide{Manager approves?}
    decide -- no --> rej[Send rejection with reason] --> e
    decide -- yes --> price[Set rate and term, compute EMI schedule]
    price --> sanction[Issue sanction letter and disburse] --> e((("●")))
```

## 7. Component diagram

The application is split by business area. Every module reaches the database only through the persistence layer, and only after the security component has checked access.

```mermaid
flowchart LR
    subgraph Clients
        branch["Branch desktop client"]
        web["Internet banking"]
        atm["ATM"]
    end
    subgraph App["Banking application"]
        cust["Customers"]
        emp["Employees"]
        acc["Accounts<br/>(savings, current, FD)"]
        pay["Payments and transfers"]
        chq["Cheques"]
        loan["Loans"]
        comp["Complaints"]
        sec["Security<br/>(authentication, access control)"]
        per["Persistence"]
    end
    db[("Database")]

    branch & web & atm -->|API| sec
    sec --> cust & emp & acc & pay & chq & loan & comp
    pay --> acc
    chq --> acc
    loan --> acc
    cust & emp & acc & pay & chq & loan & comp --> per
    per -->|encrypted connection| db
```

## 8. Deployment diagram

```mermaid
flowchart LR
    subgraph Branch["Bank branch"]
        pc["Staff PC<br/>desktop client"]
    end
    atmn["ATM<br/>(ATM software)"]
    browser["Customer device<br/>web browser"]
    subgraph DC["Bank data centre"]
        appsrv["Application server<br/>banking application"]
        dbsrv[("Database server<br/>customers, accounts and transactions,<br/>loans, employees")]
    end
    pc -->|HTTPS over bank network| appsrv
    atmn -->|ISO 8583 over leased line| appsrv
    browser -->|HTTPS| appsrv
    appsrv -->|TLS| dbsrv
```

## 9. Package diagram

Dashed arrows show which package depends on which. Products and payments depend on the core, and nothing depends on the user interface.

```mermaid
flowchart TB
    ui["ui"]
    subgraph bank
        core["core<br/>Customer, Account, Transaction"]
        products["products<br/>SavingsAccount, CurrentAccount,<br/>FixedDeposit, Loan"]
        payments["payments<br/>transfers, Cheque, ChequeBook, DebitCard"]
        staff["staff<br/>Employee and roles"]
        support["support<br/>Complaint"]
    end
    security["security"]
    persistence["persistence"]
    ui -.-> core & products & payments & staff & support
    products -.-> core
    payments -.-> core
    support -.-> core & staff
    core & products & payments & staff & support -.-> persistence
    ui -.-> security
```

## Changes from the 2022 version

The original diagrams were drawn separately and did not agree with each other. This revision:
- **Class diagram:** it had only Bank, Account, Client and a savings subclass. It now includes every concept the other diagrams use: branches, staff roles, current accounts, fixed deposits, transactions, cheques, cards, loans and complaints.
- **Use case diagram:** removed use cases that inherited from "log in" and actors associated with themselves; "log in" is now included where it's needed.
- **State machine diagrams:** the staff roles drawn as states were replaced by the life cycles of a cheque and a loan.
- **Communication diagram:** the ATM withdrawal was split from the unrelated branch-reception messages.
- **Clean-up:** removed placeholder names (Class1, UseCase16, Interface1–5, Component1–2, Package9) and fixed spelling.
- **Format:** the diagrams are now written as text, so they can be read and reviewed on GitHub without StarUML.
