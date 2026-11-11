# 02 — The System I Have in Mind

The target: one platform where **every AWB is a case**, **every case has one secure link**, and **every mail goes through one checked pipeline**, for both desks.

***

## 1. Five ideas the whole system is built on

| #     | Idea                                                                                                                                    | What it replaces                                                   |
| ----- | --------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------ |
| **1** | **One case per AWB**: owner, stage, category / clearance type, history                                                                  | Rows in Excel, memory                                              |
| **2** | **One secure link per AWB**: the customer sees status, downloads FedEx documents, uploads theirs, approves checklists, sees today's fee | 5–7 attachments in a single mails, no track of saw the mail or not |
| **3** | **One send pipeline**: template → check → send, in bulk, from a team identity, **with zero attachments**                                | VBA script, hand-typed Outlook mails                               |
| **4** | **One thread per AWB**: replies matched automatically; AI drafts the answer, a person sends it; silent threads get reminded             | Shared inbox                                                       |
| **5** | **One board per role**: everything visible by status, owner and what is blocking it                                                     | Asking around                                                      |

And one rule over all of them: **when unsure, stop and show why. Never guess.** An unknown courier category, a "FEBRK" with no broker, a damaged AWB or a missing email blocks that one row, with a reason, and never the whole batch.

***

## 2. CARGO — with the system

```mermaid
flowchart LR
    subgraph P1["① PREPARE  — minutes, not hours"]
        direction TB
        c1["Upload the sheet AS-IS<br/>choose uBond or Consol"]
        c2["✅ Type, broker, email read<br/>from the sheet automatically"]
        c3["✅ Drop all .tif invoices once<br/>→ PDF → on each AWB's link"]
        c4["✅ Ready / Needs call / Held<br/>shown per AWB"]
        c1 --> c2 --> c3 --> c4
    end

    subgraph P2["② SEND — one click"]
        direction TB
        c5["✅ Send all ready<br/>matched template per type<br/>checked · paced · no attachments"]
        c6["✅ Needs call list<br/>(Calling / HOLD / no broker)"]
        c5 --- c6
    end

    subgraph P3["③ AFTER ARRIVAL — one upload"]
        direction TB
        c7["✅ IGM sheet uploaded once<br/>matched by AWB to every case"]
        c8["✅ New AWBs (TP / part)<br/>highlighted"]
        c9["✅ Arrival notice PDF on link<br/>+ reply in the SAME thread"]
        c10["✅ IGM → payment → DO<br/>gated, fee from approved rules"]
        c7 --> c8 --> c9 --> c10
    end

    subgraph P4["④ ALWAYS OWNED"]
        direction TB
        c11["✅ Replies matched to AWB<br/>reply-needed on board"]
        c12["✅ Silent threads reminded"]
        c13["✅ Board, timeline, audit"]
        c11 --> c12 --> c13
    end

    P1 --> P2 --> P3 --> P4
```

## 3. COURIER — with the system

```mermaid
flowchart LR
    subgraph Q1["① PREPARE"]
        direction TB
        k1["Upload pull sheet<br/>22 columns validated"]
        k2["✅ Owner from Agent / Team<br/>BOM rows → pool"]
        k3["✅ Category from 'Master'<br/>11 categories · unknown = blocked"]
        k4["✅ Template + document list<br/>per category"]
        k1 --> k2 --> k3 --> k4
    end

    subgraph Q2["② SEND"]
        direction TB
        k5["✅ Draft per AWB already written<br/>with the secure link"]
        k6["✅ Review → approve → send<br/>in bulk"]
        k5 --> k6
    end

    subgraph Q3["③ AFTER SEND"]
        direction TB
        k7["✅ Customer opens link<br/>uploads / approves RBOE"]
        k8["✅ Reply matched to AWB<br/>AI drafts · agent sends"]
        k9["✅ Documents complete<br/>→ Admin queue → BOE filed"]
        k7 --> k8 --> k9
    end

    subgraph Q4["④ ALWAYS OWNED"]
        direction TB
        k10["✅ Silent threads reminded"]
        k11["✅ HV → Cargo on the SAME case<br/>with reason + history"]
        k12["✅ Link status: sent → opened →<br/>docs received"]
        k10 --> k11 --> k12
    end

    Q1 --> Q2 --> Q3 --> Q4
```

***

## 4. Both desks, one engine

```mermaid
flowchart TB
    subgraph COURIER["Courier desk"]
        ci["Pull sheet"] --> cd["Decide: category<br/>(11 types)"]
    end
    subgraph CARGO["Cargo desk"]
        gi["uBond / Consol / IGM sheet"] --> gd["Decide: clearance type<br/>NFBRK · FEBRK-Jeena · FEBRK-Sunimpex"]
    end

    cd --> E
    gd --> E

    subgraph E["Shared engine"]
        direction LR
        e1["Case"] --> e2["Secure link"] --> e3["Template"] --> e4["Pre-send check"] --> e5["Send<br/>(bulk, no attachments)"] --> e6["Thread<br/>replies · AI draft · reminders"]
    end

    E --> cb["Courier: Admin queue → BOE filed<br/>or hand-off to Cargo"]
    E --> gb["Cargo: IGM → payment → DO"]
```

The two desks differ only in **how the case is decided** (category vs clearance type) and **how it closes** (Admin BOE vs DO). Everything in the middle is shared, so it is built once.

***

## 5. Today vs with the system — side by side

### Cargo

| Step                  | Today                            | With the system                                                                      |
| --------------------- | -------------------------------- | ------------------------------------------------------------------------------------ |
| Sheet                 | Restructured by hand             | Uploaded as-is; operator picks uBond / Consol                                        |
| Broker + email        | VLookup + Outlook search         | Read from `End Result`, `FedEx Broker`, `Standard Remarks`                           |
| Unclear type          | Calls tracked in heads           | **Needs call** list                                                                  |
| Invoices              | `.tif` attached per mail         | Dropped once → PDF → on the link                                                     |
| Sending               | VBA, personal mailbox, 20–30 min | One click, team identity, every mail checked                                         |
| uBond then Consol     | Same AWB mailed twice            | Recognised; same case, same link, no second pre-alert                                |
| MAWB / IGM            | Copied into new mails at 14:00   | One IGM sheet upload updates every AWB; arrival notice PDF; reply in the same thread |
| New AWBs in IGM sheet | Ad hoc                           | Highlighted as "not in consol"                                                       |
| DO fee                | Table in the mail                | Today's fee on the link                                                              |
| Payment → DO          | Excel + mailbox                  | Gated board; DO on the link                                                          |
| Follow-up             | Memory                           | Automatic reminders                                                                  |

### Courier

| Step                | Today                         | With the system                             |
| ------------------- | ----------------------------- | ------------------------------------------- |
| Who works which AWB | Pick from the sheet           | Owner from the sheet; pool for BOM rows     |
| What to ask for     | Reason read by eye            | `Master` → category; unknown blocked        |
| Writing the mail    | Typed per AWB                 | Draft ready per AWB                         |
| Documents           | Attached to mails             | Uploaded on the link; checklist ticks       |
| RBOE                | Link pasted into a typed mail | RBOE template; approve / object on the link |
| Replies             | Shared inbox                  | On the AWB; AI draft; person sends          |
| Follow-up           | Memory                        | Automatic reminders                         |
| To Admin            | Forwarded mail                | Admin queue                                 |
| HV to Cargo         | Verbal                        | Same case moves, with reason and history    |

***

## 6. The customer's view: one link instead of a pile of mails

```mermaid
sequenceDiagram
    autonumber
    participant F as FedEx (system)
    participant C as Customer
    participant L as Secure link /l/…
    F->>C: Mail 1 — "Arriving today / action needed" + link
    C->>L: Opens link (no login, mobile)
    L-->>C: Status · FedEx documents (invoice PDF, DO format, bank details) · what is needed · today's fee
    C->>L: Uploads documents / approves checklist
    F->>C: Reply in same thread — arrival notice (MAWB / IGM / flight)
    Note over L: Arrival notice PDF added to the same link
    C->>F: Replies with UTR
    F->>C: Reply in same thread — DO issued
    Note over L: DO added to the same link
```

***

## 7. What stays outside the system

| Stays as it is                               | Why                                                          |
| -------------------------------------------- | ------------------------------------------------------------ |
| **ACCS**: shipments and invoices             | System of record. The PoC reads its exports (sheet + `.tif`) |
| **ICEGATE**: IGM and BOE filing              | Filed by the IGM team, the CHA, or FedEx Admin               |
| **The CHA's filing work**                    | FedEx does not hold a cargo clearance licence                |
| **Payments**: NEFT                           | The system records the UTR; money does not move through it   |
| **Decisions**: calls, approvals, final sends | A person always decides. AI only drafts                      |

***

## 8. Where the system can go next

These are designed, not built. Each is described in docs 03–05.

| Next                                                                    | Effect                                                |
| ----------------------------------------------------------------------- | ----------------------------------------------------- |
| One-click case for new IGM-sheet AWBs (TP / part shipments)             | No new AWB waits for a person to notice it            |
| CHA view of the case (Jeena / Sunimpex / own CHA)                       | Documents stop being forwarded; CHA marks "BOE filed" |
| FEBRK duty checklist approved on the link                               | Before-18:00 approvals tracked, not chased            |
| Invoices pulled from ACCS automatically                                 | The morning `.tif` step disappears                    |
| UTR matched to the bank statement                                       | Payment verified without a person checking            |
| "IGM not generated — ICEGATE issue" mail and day-1 / day-2 DO reminders | Every template moment covered                         |
| Manager SLA view                                                        | Stuck cases surfaced by age                           |

