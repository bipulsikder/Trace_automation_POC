# 07 — Technical Architecture

How the PoC is built, so an engineering team can rebuild it on FedEx's platform. The **design** (flows, rules, data model) is what to keep; the **technology choices** are PoC choices and can be swapped.

---

## 1. Stack

| Layer | PoC choice | Role | Swap-friendly? |
|---|---|---|---|
| Web app | **Next.js 16** (App Router), **React 19**, TypeScript | Pages (server components) + API (route handlers) in one app | Any web stack |
| UI | Tailwind CSS, shadcn / Base UI, lucide icons, TanStack Query | Screens | Yes |
| Database | **PostgreSQL** (Supabase) with row-level security | All data | Any Postgres |
| Auth | Supabase Auth (email login) | Internal users | → Entra ID / SSO |
| Files | Supabase Storage, private bucket `case-documents` | Invoices, uploads, PDFs | → Blob / S3 |
| Outbound mail | Pluggable driver: `smtp` (nodemailer) · `graph` (Microsoft Graph via MSAL) · `dry_run` | Sending | Yes, one interface |
| Inbound mail | IMAP poll (imapflow + mailparser) | Replies | → Graph subscription |
| Excel | exceljs | Sheet import, sample sheets | Yes |
| PDF | pdf-lib | Arrival notice; TIFF pages → PDF | Yes |
| TIFF | utif2 in a browser **Web Worker** pool | Invoice conversion on the agent's machine | Server-side also possible |
| AI | Google Gemini | Reply classification + drafts; never sends | Any LLM, one module |
| Scheduling | Cron routes protected by `CRON_SECRET` | Reminder sweep, inbox poll | Any scheduler |
| Validation | zod on every API input | | |
| Tests | vitest | 67 unit tests | |

---

## 2. Component view

```mermaid
flowchart TB
    subgraph Browser["Browsers"]
        AG["Agent UI<br/>Import · Boards · Consol · IGM & DO · Case"]
        TW["TIFF worker pool<br/>(Web Workers)"]
        CU["Customer / CHA<br/>/l/&lt;token&gt;"]
        AG --- TW
    end

    subgraph App["Next.js server"]
        direction TB
        API["API route handlers<br/>/api/courier/* · /api/cargo/* · /api/l/* · /api/cron/*"]
        subgraph LIB["Domain modules (src/lib)"]
            direction LR
            L1["courier/<br/>parse · classify · readiness"]
            L2["cargo/<br/>parse · build-import · IGM · fees · PDF"]
            L3["compose/<br/>draft · variables · preflight · send"]
            L4["links/<br/>create · resolve · documents"]
            L5["email + thread/<br/>driver · ingest · match · reply"]
            L6["followups/<br/>sweep"]
        end
        API --> LIB
    end

    DB[("PostgreSQL<br/>RLS on every table")]
    ST[("Private storage<br/>case-documents")]
    MAIL["Mail server<br/>SMTP / Graph"]
    IMAP["Inbox<br/>IMAP"]
    AI["LLM<br/>(drafts only)"]

    AG -->|session| API
    CU -->|token| API
    LIB --> DB
    LIB --> ST
    L3 --> MAIL
    L5 --> IMAP
    L5 --> AI
```

---

## 3. Main flows as sequence diagrams

### 3.1 Bulk send (cargo "Send all ready" / courier bulk send)

```mermaid
sequenceDiagram
    autonumber
    actor A as Agent
    participant API as /api/courier/cases/send
    participant SC as sendCases()
    participant R as readiness
    participant D as draft-service
    participant P as preflight
    participant M as mail driver
    participant DB as Postgres
    A->>API: caseIds, stream, touch
    API->>SC: sendCases(...)
    SC->>DB: load cases, links, docs, drafts (batched)
    SC->>R: computeReadiness per case
    R-->>SC: ready / blocked + reasons
    loop each ready case (concurrency-limited)
        SC->>D: composeCaseDraft(template by type/category)
        D->>DB: ensure link, seed default docs, resolve fee
        SC->>P: runPreflight
        alt pass
            SC->>M: send (0 attachments, In-Reply-To for Touch 2)
            M-->>SC: message id
            SC->>DB: draft sent · last_outbound_at · timeline · audit
        else fail
            SC->>DB: draft blocked + reasons
        end
    end
    SC-->>A: per-case result: sent / skipped (reason) / failed
```

### 3.2 IGM sheet → Touch 2

```mermaid
sequenceDiagram
    autonumber
    actor I as IGM agent
    participant W as IGM workbench
    participant API as /api/cargo/consols/[id]
    participant PI as parse-igm-sheet
    participant MA as igm-match
    participant CI as consol-igm
    participant PDF as arrival-notice-pdf
    participant SC as sendCases(touch=post_igm)
    I->>W: upload IGM .xlsx
    W->>API: POST /igm
    API->>PI: parse composite MAWB cell, precision guard
    API->>MA: classify by AWB only
    MA-->>W: matched / changed / other / NEW / unreadable / duplicate
    I->>W: Apply
    W->>API: POST /apply → CI writes MAWB, IGM, flight, pcs, wt
    I->>W: Build arrival notices
    W->>API: POST /build-updates → PDF per AWB → on link
    I->>W: Send Touch 2
    W->>SC: reply in Touch 1 thread, same link
```

### 3.3 Customer reply

```mermaid
sequenceDiagram
    autonumber
    participant C as Customer
    participant IN as Inbox (IMAP)
    participant PO as poll-and-ingest
    participant MI as match-inbound
    participant AI as classify + draft
    actor A as Agent
    C->>IN: reply mail
    PO->>IN: poll (every ~2 min / "check now")
    PO->>MI: match by In-Reply-To / References → AWB in subject/body
    MI->>MI: store on case thread · last_inbound_at · thread_state = reply needed
    MI->>AI: classify route, draft reply (courier)
    AI-->>A: draft on the thread (never auto-sent)
    A->>C: edit + send (same thread)
```

### 3.4 Customer downloads a document

```mermaid
sequenceDiagram
    autonumber
    participant C as Customer browser
    participant API as /api/l/[token]/documents/[id]/download
    participant DB as Postgres
    participant ST as Storage
    C->>API: GET with token
    API->>DB: hash(token) → active link → document belongs to link?
    API->>ST: create signed URL (300 s)
    API->>DB: link_events: download
    API-->>C: redirect to signed URL
```

---

## 4. Design rules the code enforces

| Rule | Where | Why |
|---|---|---|
| **Zero attachments on every mail** | `send-draft.ts`, `preflight.ts`, `send-reply.ts` | One source of truth (the link); revocable; logged |
| **Fail-closed pre-send check** | `preflight.ts` | A wrong mail is worse than a late one |
| **One active link per case** | `create-link.ts` | Continuity across touches, stages and desks |
| **Token stored as hash; signed URLs; no storage paths to the browser** | `link-token.ts`, `resolve-link.ts` | A database leak does not expose links or files |
| **Fees only from approved `fee_rules`** | `cargo/fees.ts`, `variables.ts` | No typed or stale amount reaches a customer |
| **Operator chooses the stream (uBond / Consol)** | import route | Same columns; can't be inferred safely |
| **AWB is the only match key** | `igm-match.ts`, `match-inbound.ts` | Names vary |
| **Damaged AWBs rejected** | `parse-igm-sheet.ts`, `parse-sheet.ts` | Excel already lost the digits |
| **Unknown = blocked** | `classify.ts`, `build-import.ts` | No default template, no guessed broker |
| **AI never sends** | `generate-draft-reply.ts` | Human accountability |
| **One pipeline for both desks** | `compose/*` with `division` param | Built once, behaves the same |
| **Status-guarded sends** (`sending` state) | migration 0048, `send-draft.ts` | No double-send on retry |

---

## 5. Configuration

| Variable | Purpose |
|---|---|
| `NEXT_PUBLIC_SUPABASE_URL`, anon / service keys | Database and storage |
| `MAIL_DRIVER` | `smtp` · `graph` · `dry_run` |
| SMTP_* / Graph tenant, client, mailbox | Sending identity |
| IMAP_* | Reply mailbox |
| `CRON_SECRET` | Protects scheduled routes |
| `GEMINI_API_KEY` | AI drafts (optional) |
| `NEXT_PUBLIC_APP_URL` | Base URL in customer links |

`dry_run` composes, checks and records everything but sends nothing: safe for demos and shadow runs.

---

## 6. Security built into the PoC

| Area | What is in place |
|---|---|
| Internal access | Login; roles `admin` / `lead` / `operator` / `reviewer` / `viewer`; per-desk access |
| Data | Row-level security on every table; service key only in server code (`server-only` guard) |
| Inputs | zod validation on every route |
| Customer links | Hashed tokens; one case per link; 5-min signed downloads; full event log; revocable; expiry supported |
| Mail | Zero attachments; validated recipients; team identity; dry-run mode |
| Audit | `audit_logs`, `case_timeline_events`, `message_drafts` (exact content sent), `link_events` |

---

## 7. Recommendations for the production build

| Area | Recommendation |
|---|---|
| Identity | Company SSO; map roles from groups |
| Mail | Send from the desk shared mailboxes through the company mail platform; receive by push, not polling |
| Jobs | Move bulk send, PDF generation, reminder sweep and inbox ingest to a queue with retries |
| Files | Virus scan on customer uploads; server-side TIFF conversion as a fallback |
| Data | Host in an approved region; retention policy per document type |
| Tests | Keep the 67 unit tests' cases as acceptance cases; add end-to-end tests: import → send → IGM → Touch 2 → DO |
| Observability | Per-send logs with message id; dashboards for failed sends, stale threads, unmatched replies |
