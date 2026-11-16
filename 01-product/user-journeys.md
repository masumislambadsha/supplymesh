# User Journey Diagrams

Four journeys are shown. The first three are the brief's suggested workflows; the fourth (organization setup) is the precondition for the rest. Each journey has:
- a **swim-lane flow** (who does what, including decision points and the unhappy paths), and
- an **experience map** (how each step feels for the user, scored 1–5), which shows where the product must reduce friction.

The technical view of the same flows (API calls, transactions, failure handling) is in [system-design.md](../04-system-design/system-design.md).

---

## Journey 1 — Supplier onboarding and verification

**Persona:** Rafi, a Procurement Manager, needs to start buying network equipment from a new vendor in Singapore.
**Goal:** the vendor becomes an `active` supplier that POs can be issued to, without Rafi chasing documents by email.

```mermaid
flowchart TD
    subgraph PM["Procurement Manager"]
        A1["Create supplier draft<br/>(name, country, category)"] --> A2{"Fill in details<br/>myself or invite supplier?"}
        A2 -->|"Invite"| A3["Send onboarding link<br/>to supplier contact"]
        A2 -->|"Myself"| A4["Enter profile and<br/>upload documents"]
    end
    subgraph SUP["Supplier contact (no account)"]
        B1["Open link (valid 7 days)"] --> B2["Complete profile,<br/>bank details"]
        B2 --> B3["Upload required documents<br/>(trade licence, tax cert)"]
        B3 --> B4["Submit"]
    end
    subgraph SYS["SupplyMesh"]
        C1{"Duplicate tax ID<br/>in this org?"}
        C2["Virus-scan files"]
        C3["Status: pending_verification<br/>notify verifiers"]
    end
    subgraph VER["Verifier (another Procurement Manager or Owner)"]
        D1["Review profile + each document"] --> D2{"All required<br/>documents valid?"}
        D2 -->|"No"| D3["Reject document with reason"]
        D2 -->|"Yes"| D4["Verify supplier"]
    end

    A3 --> B1
    B4 --> C1
    A4 --> C1
    C1 -->|"Yes"| X1["Blocked: link to existing supplier"]
    C1 -->|"No"| C2 --> C3 --> D1
    D3 --> B3
    D4 --> E1(["Supplier ACTIVE<br/>can receive POs"])
    D1 -.->|"Fraud / sanctions concern"| E2(["Supplier REJECTED"])
```

```mermaid
journey
    title Supplier onboarding: Procurement Manager experience
    section Start
      Create draft supplier: 4: Procurement Manager
      Send onboarding link instead of emailing forms: 5: Procurement Manager
    section Supplier completes
      Supplier fills profile on own time: 4: Supplier
      Upload documents from phone or laptop: 3: Supplier
    section Verification
      Get notified when ready to review: 5: Procurement Manager
      Review documents side by side: 4: Verifier
      Document rejected and re-requested: 2: Supplier
    section Done
      Supplier active and selectable on POs: 5: Procurement Manager
```

**Design choices this journey drives**
- The **self-service onboarding link** turns the worst part of today's process (chasing PDFs by email) into an async task for the supplier. It uses a token, not an account, so the MVP doesn't need a supplier login system.
- **Duplicate check happens before verification.** Finding out after someone has spent 20 minutes reviewing documents would be the most frustrating outcome.
- **Per-document rejection** sends the supplier back to re-upload just one file, not the whole form (the lowest-scored step above).
- **Four-eyes rule:** the person who changed the bank details (`bank_details_changed_by_member_id`) can't be the one who verifies them (BR-05).

---

## Journey 2 — Purchase request and multi-level approval

**Persona:** Nusrat, a software engineer (Employee), needs 3 laptops for new hires: estimated **$5,400**, above the $5,000 multi-approval threshold.
**Approvers:** Karim (IT Department Manager, limit $2,000), Rafi (Procurement Manager), Farhana (Finance Officer).

```mermaid
flowchart TD
    subgraph EMP["Employee (requester)"]
        A1["Create request: 3 laptops<br/>$1,800 each, required in 2 weeks"] --> A2["Save draft"]
        A2 --> A3["Submit"]
        A9["Edit and resubmit"]
    end
    subgraph SYS["SupplyMesh approval engine"]
        S1["Convert to base currency<br/>and snapshot workflow version"] --> S2{"Match rule<br/>by amount, dept, category"}
        S2 -->|"no rule"| S0["Block submit,<br/>alert Owner about policy gap"]
        S2 -->|"5,000 and above rule"| S3["Create steps:<br/>1 Dept Mgr, 2 Proc Mgr, 3 Finance"]
        S3 --> S4["Activate step 1,<br/>notify approver"]
        S6["Activate next step"]
        S7(["Request APPROVED<br/>notify requester + procurement"])
        S8(["Request REJECTED"])
    end
    subgraph APP["Approvers (in order)"]
        P1["Dept Manager reviews"] --> P1d{"Decision"}
        P2["Procurement Manager reviews"] --> P2d{"Decision"}
        P3["Finance Officer reviews"] --> P3d{"Decision"}
    end

    A3 --> S1
    S4 --> P1
    P1d -->|"approve"| S6
    S6 -->|"step 2"| P2
    P2d -->|"approve"| S6
    S6 -->|"step 3"| P3
    P3d -->|"approve"| S7
    P1d -->|"reject + comment"| S8
    P2d -->|"reject + comment"| S8
    P3d -->|"reject + comment"| S8
    P2d -->|"return: get 2nd quote"| A9
    A9 --> S1
    S4 -.->|"no action within SLA"| E1["Reminder, then<br/>escalate to Owner"]
```

```mermaid
journey
    title Purchase request: requester and approver experience
    section Requester
      Fill a short structured form: 4: Employee
      See who must approve and in what order: 5: Employee
      Wait without knowing status (today): 1: Employee
      Track live status and comments: 4: Employee
    section Approvers
      Get one inbox for everything waiting on me: 5: Department Manager
      See amount, budget context, history in one view: 4: Procurement Manager
      Must write a reason to reject: 3: Finance Officer
    section Outcome
      Notified instantly on final decision: 5: Employee
```

**Design choices this journey drives**
- **The approval chain is shown at submission.** "Who has it now?" is the requester's biggest frustration today (score 1). Because the steps are created up front, the UI can draw the whole route immediately.
- **"Return" differs from "reject".** Most real "no"s are "not like this" (e.g. get a second quote). Returning keeps the request and its history; rejecting ends it.
- **Karim's $2,000 limit doesn't block this request.** His step is an endorsement, and final authority comes from the Finance step. The limit (BR-09) only matters when a manager would be the **last** approver. For example, under rule 1 (< $500, department manager only), a newly promoted manager with a $300 limit couldn't finalize a $450 request on their own, so the engine would add the parent department's manager.
- **The SLA reminder/escalation** prevents the classic bottleneck: an approver on leave holds up a request for weeks.

---

## Journey 3 — Purchase order, goods receipt, invoice and payment

**Personas:** Rafi (Procurement Manager) creates the PO. Nusrat (requester) confirms delivery. Farhana and Jamal (Finance Officers) process the invoice.

```mermaid
flowchart TD
    subgraph PROC["Procurement Manager"]
        A1["Open approved request"] --> A2["Create PO: choose active supplier,<br/>confirm price and tax"]
        A2 --> A3{"PO total within 10%<br/>of approved estimate?"}
        A3 -->|"No"| A4["Cancel draft; requester raises<br/>a new request for approval"]
        A3 -->|"Yes"| A5["Issue PO: PDF<br/>emailed to supplier"]
    end
    subgraph SUPP["Supplier"]
        B1["Acknowledges PO"] --> B2["Delivers 3 laptops<br/>(1 damaged)"]
        B5["Sends invoice for 3 laptops"]
    end
    subgraph RCV["Receiver (requester)"]
        C1["Record goods receipt:<br/>2 accepted, 1 rejected (damaged)"]
    end
    subgraph FIN["Finance Officers"]
        D1["Farhana records invoice<br/>against PO, attaches PDF"] --> D2{"Duplicate<br/>invoice number?"}
        D2 -->|"Yes"| D3["Blocked"]
        D2 -->|"No"| D4["Run three-way match"]
        D4 --> D5{"Match result"}
        D5 -->|"Qty billed 3 but received 2"| D6["Discrepancy opened"]
        D6 --> D7["Request credit note,<br/>or accept with reason"]
        D5 -->|"Matched"| D8["Jamal approves invoice<br/>(not the recorder)"]
        D7 --> D8
        D8 --> D9["Record payment<br/>(full or partial)"]
    end

    A5 --> B1
    B2 --> C1
    C1 --> B5
    B5 --> D1
    D9 --> E1(["Invoice PAID, PO CLOSED<br/>supplier performance updated"])
```

```mermaid
journey
    title PO to payment: experience by role
    section Procurement
      PO pre-filled from approved request: 5: Procurement Manager
      Supplier gets PO by email automatically: 4: Procurement Manager
    section Receiving
      Confirm delivery in two clicks: 4: Employee
      Report a damaged item: 3: Employee
    section Finance
      Duplicate invoice caught automatically: 5: Finance Officer
      See PO, receipt, invoice side by side: 5: Finance Officer
      Resolve discrepancy with supplier: 2: Finance Officer
      Record payment and see invoice paid: 4: Finance Officer
```

**Design choices this journey drives**
- **Goods receipt is a first-class step.** Without it, the damaged laptop would have been paid for. This is the "three-way" in three-way match.
- **The 10% PO-vs-approval tolerance** (BR-13) stops a $5,400 approval from quietly becoming a $9,000 order.
- **Segregation of duties.** Farhana records the invoice and Jamal approves it. The database CHECK `chk_inv_sod` makes this unbypassable, even by a buggy client.
- **Discrepancy resolution is the lowest-scored step.** That is why the matching screen shows the PO line, the receipt line and the invoice line next to each other, with the variance pre-computed.

---

## Journey 4 — Organization setup (prerequisite)

```mermaid
flowchart LR
    A["Owner signs up<br/>creates organization"] --> B["System seeds 5 system roles<br/>+ default approval workflow"]
    B --> C["Owner creates departments<br/>and assigns managers"]
    C --> D["Owner invites members<br/>with role, department, limit"]
    D --> E["Owner reviews default<br/>approval rules"]
    E --> F{"Gaps in<br/>amount ranges?"}
    F -->|"Yes"| E
    F -->|"No"| G["Publish workflow v1"]
    G --> H(["Organization ready<br/>for first request"])
```

The **seeded default workflow** encodes the brief's example ($500 / $5,000 / multi-approval). A new organization can submit its first request in minutes, then adjust the policy. A blank rule editor would be the main reason for drop-off during onboarding.
