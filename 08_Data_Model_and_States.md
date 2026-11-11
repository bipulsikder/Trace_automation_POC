# 08 — Data Model and State Machines

## 1. Entity diagram (core)

```mermaid
erDiagram
    CONSOLS ||--o{ AWB_CASES : groups
    AWB_CASES ||--o{ CASE_LINKS : "has (1 active)"
    CASE_LINKS ||--o{ LINK_DOCUMENTS : holds
    CASE_LINKS ||--o{ LINK_EVENTS : logs
    AWB_CASES ||--o{ MESSAGE_DRAFTS : "outbound mails"
    AWB_CASES ||--o{ EMAIL_EVENTS : "in + out log"
    AWB_CASES ||--o{ CASE_TIMELINE_EVENTS : story
    AWB_CASES ||--o{ CASE_ASSIGNMENTS : ownership
    AWB_CASES }o--|| APP_USERS : owner
    MESSAGE_DRAFTS }o--|| TEMPLATES : uses
    TEMPLATES }o--|| CATEGORY_MAP : "keyed by"
    BATCH_RUNS ||--o{ AWB_CASES : imported

    AWB_CASES {
        uuid id PK
        text awb
        text division "courier | cargo"
        text category "courier"
        text clearance_type "cargo"
        text stage "cargo stage"
        text current_status
        text thread_state
        uuid consol_id FK
        text mawb
        text igm_number
        date arrival_date
        int pieces
        numeric weight_kg
        text utr_no
        timestamptz do_issued_at
        uuid owner_user_id FK
        text handed_off_from
    }
    CONSOLS {
        uuid id PK
        text consol_ref
        text mawb "null until IGM"
        timestamptz pre_arrival_sent_at
        timestamptz igm_received_at
        timestamptz post_igm_sent_at
        text raw_mawb_cell
    }
    CASE_LINKS {
        uuid id PK
        uuid case_id FK
        text token_hash "SHA-256"
        text role_scope "customer | cha"
        timestamptz expires_at
        boolean active
    }
    LINK_DOCUMENTS {
        uuid id PK
        uuid link_id FK
        text direction "fedex | customer"
        text doc_key
        text storage_path "never sent to browser"
    }
    MESSAGE_DRAFTS {
        uuid id PK
        uuid case_id FK
        text division
        text status "draft→approved→sending→sent|failed|blocked"
        text pre_alert_type "ubond | consol"
        text touch "pre_arrival | post_igm"
        int resend_count
        jsonb blocked_reasons
    }
    TEMPLATES {
        uuid id PK
        text division
        text category_key
        text subject
        text body_html
        jsonb default_documents
    }
```

---

## 2. Tables by purpose

| Group | Tables | Purpose |
|---|---|---|
| **Case** | `awb_cases`, `consols`, `batch_runs`, `case_assignments`, `case_updates` | One row per AWB per desk; consol grouping; import batches; ownership |
| **Link** | `case_links`, `link_documents`, `link_events` | Secure link, its documents, its activity |
| **Messages** | `message_drafts`, `email_events`, `emails`, `ai_drafts` | Every outbound mail; inbound / outbound log; AI drafts |
| **Configuration** | `templates`, `category_map`, `fee_rules`, `penalty_rules`, `mailbox_configs`, `app_config` | Admin-editable behaviour |
| **People** | `app_users` | Role, desk, active |
| **Audit** | `audit_logs`, `case_timeline_events` | Who did what, per case story |
| **Legacy / earlier experiments** | `reminder_jobs`, `followup_schedules`, `call_tasks`, `clearance_fill*`, `training_examples`… | From earlier iterations; not needed for the flows in this pack |

### Key columns on `awb_cases`

| Group | Columns |
|---|---|
| Identity | `awb`, `division`, `consignee_name`, recipient email |
| Decision | `category` (courier), `clearance_type` (cargo: `nfbrk` / `febrk-jeena` / `febrk-sunimpex`) |
| Manifest | `mawb`, `igm_number`, flight, `arrival_date`, `pieces`, `weight_kg`, `contents`, `manifest_applied_at` |
| Lifecycle | `stage` (cargo), `current_status` (courier + shared), `admin_status` (courier) |
| Payment | DO payment status, `utr_no`, `do_issued_at` |
| Thread | `last_outbound_at`, `last_inbound_at`, `unread_count`, `thread_state` |
| Ownership | `owner_user_id`, `ownership_status`, `claimed_via` |
| Checks | `courier_check_cleared_at`, `courier_check_cleared_by` |
| Hand-off | `handed_off_from`, `handed_off_reason`, `handed_off_at` |

---

## 3. State machines

### 3.1 Cargo case stage

```mermaid
stateDiagram-v2
    direction LR
    [*] --> pre_alert: imported
    pre_alert --> arrived: IGM sheet applied
    pre_alert --> docs_complete: required uploads done
    arrived --> docs_complete
    docs_complete --> igm_filed
    arrived --> igm_filed
    igm_filed --> payment_pending: request payment<br/>(fee from rules)
    payment_pending --> payment_verified: UTR verified
    payment_verified --> do_issued: DO on link + mail
    do_issued --> released
    released --> closed
    closed --> [*]
```
Gate rule: **IGM before payment, payment before DO.** Enforced in `api/cargo/cases/[id]/stage`.

### 3.2 Thread state (both desks)

```mermaid
stateDiagram-v2
    direction LR
    [*] --> waiting_customer: we sent
    waiting_customer --> waiting_us: customer replied
    waiting_us --> waiting_customer: we replied
    waiting_customer --> stale: no reply > threshold (4 h)
    stale --> waiting_customer: reminder sent
    stale --> waiting_us: customer replied
    waiting_us --> resolved
    waiting_customer --> resolved
    resolved --> [*]
```

### 3.3 Outbound mail (message draft)

```mermaid
stateDiagram-v2
    direction LR
    [*] --> draft: composed
    draft --> blocked: pre-send check fails
    blocked --> draft: fixed (email, template, broker…)
    draft --> approved: approve (or implicit in bulk send)
    approved --> sending: claimed by sender (no double send)
    sending --> sent
    sending --> failed
    failed --> sending: retry
    sent --> sent: resend (count +1)
    sent --> [*]
```

### 3.4 Consol

```mermaid
stateDiagram-v2
    direction LR
    [*] --> intake: Consol sheet imported
    intake --> touch1_sent: pre_arrival_sent_at
    touch1_sent --> igm_received: IGM sheet applied
    igm_received --> touch2_sent: post_igm_sent_at
    touch2_sent --> [*]
```

### 3.5 Courier case

```mermaid
stateDiagram-v2
    direction LR
    [*] --> imported
    imported --> draft_ready
    draft_ready --> approved
    approved --> sent
    sent --> docs_received: uploads / RBOE approved / marked
    docs_received --> ready_for_admin: submit
    ready_for_admin --> boe_filed: Admin
    ready_for_admin --> returned: Admin returns with reason
    sent --> handed_off: HV → Cargo
    docs_received --> handed_off
    boe_filed --> [*]
    handed_off --> [*]
```

---

## 4. Configuration data

### `fee_rules` (cargo DO, Delhi)

| Clearance | Day offset from arrival | DO | Admin fee | GST 18% | **Total** | Approved |
|---|---|---|---|---|---|---|
| all | 0 (arrival day) | 2,600 | 0 | 468 | **3,068** | ✓ |
| all | ≥ 1 | 2,600 | 1,000 | 648 | **4,248** | ✓ |

Only rows marked approved are shown to customers. No match → "no approved rule", never a guess.

### `category_map`

| Desk | Keys |
|---|---|
| Courier | `courier_rboe`, `courier_ci_missing`, `courier_invoice_objection`, `courier_content`, `courier_duty`, `courier_pims`, `courier_cbe14`, `courier_port_return`, `courier_value_clarification`, `courier_rr`, `courier_moc`, `courier_reminder` |
| Cargo | `nfbrk`, `febrk-jeena`, `febrk-sunimpex`, `cargo_arrival_notice`, `cargo_reminder` |

---

> **Other gateways:** the PoC data is Delhi-only. Doc 13 adds `gateways`, `gateway_schedules`, `gateway_chas`, `gateway_holidays`, `user_gateways` and a `gateway_code` on cases, consols, batches, fee rules, penalty rules and templates.

## 5. Migrations added in this PoC

| # | Adds |
|---|---|
| 0047 | One link model for both desks |
| 0048 | `sending` status on drafts |
| 0049 | Cargo workspace: consols, stage, thread fields |
| 0050 | Inbound matching |
| 0051 | Cargo fees and timeline |
| 0052 | Board run column |
| 0053 | Arrival notice + reminder categories |
| 0054 | Streams, touches, resend tracking, consol timestamps, nullable consol MAWB |
| 0055 | Consol manifest fields; pieces / weight / contents on cases |
| 0056 | Big-shipment check fields |
