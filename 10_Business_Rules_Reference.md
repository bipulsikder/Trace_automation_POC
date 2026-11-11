# 10 — Business Rules Reference

Every rule the PoC applies, in one place. Use it as the specification and the QA checklist for the production build.

---

## A. Import rules

| ID | Rule | Desk |
|---|---|---|
| A1 | `AWB` and `Reason` columns are required; other columns found by header synonyms | Courier |
| A2 | Operator chooses **uBond** or **Consol** before parsing; the stream is never inferred | Cargo |
| A3 | Cargo sheet read as-is: first header may be a time (`10:22 AM`) = agent name column | Cargo |
| A4 | `Value` = invoice value; value class only from an exact `Value Class` header | Cargo |
| A5 | Email = first **valid** address across email column, `Standard Remarks`, other cells; `0` / phone numbers ignored | Cargo |
| A6 | AWB rejected if scientific notation, fractional, or < 10 digits | Both |
| A7 | Courier AWB of 11 digits flagged | Courier |
| A8 | Duplicate AWB in the same import → one case | Both |
| A9 | Cargo AWB imported in the last 24 h → **reuse** the case and link | Cargo |
| A10 | `BOM` / `BOM CEP` rows → pool queue | Courier |
| A11 | Batch name `COURIER-YYYY-MM-DD-AM/PM`; clash → `-2` | Courier |

## B. Decision rules

### B1. Courier category

| Order | Signal |
|---|---|
| 1 | `Master` column (authoritative) |
| 2 | `Reason` prefix (text before first comma) matched to category label / aliases |
| 3 | `Reason` contains an alias |
| 4 | Keyword fallback (`INVMS` → CI Missing, `INVIC` → Invoice Objection, `DESCI` → Content, `ROD` → Duty, `PIMS`, `SVB`/`CBE14`, `PFEXP` → Port Return, `VALDS` → Value Clarification, `EXP_DOCS`/`R&R`, `MOC`/`DGCA`, `RBOE`) |
| — | Two strong matches → **conflict** flag. Nothing → **blocked** |

### B2. Cargo clearance type

| `End Result` | `FedEx Broker` | Result |
|---|---|---|
| NFBRK | any | `nfbrk` |
| FEBRK | Jeena | `febrk-jeena` |
| FEBRK | Sunimpex | `febrk-sunimpex` |
| FEBRK | empty / other | **Needs call** |
| Calling | any | **Needs call** |
| HOLD | any | **Needs call** (do not send) |
| empty / other | — | **Needs call** |

## C. Readiness rules (before any send)

| Module | Ready when | Applies to |
|---|---|---|
| Template | Active template for the category / type | Both |
| Link | Active secure link | Both |
| Pack | Default FedEx documents on the link | Cargo |
| Email | Valid recipient | Both |
| Broker / category | Resolved | Both |
| Invoice | Invoice PDF on the link | **Cargo uBond** |
| Big-shipment | Not (≥ 10 pieces or > 70 kg per piece), or cleared by "Keep in cargo" | Cargo |
| Touch 1 waiver | uBond pre-alert sent in last 24 h → Touch 1 skipped | Cargo Consol |
| Touch 2 | MAWB + IGM + flight present; arrival notice PDF on link | Cargo Consol |

## D. Pre-send check (fail-closed, at the moment of sending)

| Check | Blocked reason shown |
|---|---|
| No active template | "No active template for <type>" |
| Invalid / missing recipient | "No valid recipient" |
| No active link | "No secure link" |
| Unresolved `{VARIABLE}` | "Unresolved variable <name>" |
| Template has fixed attachments | "Attachments not allowed" |
| Unknown category / bare FEBRK | "Category / broker not resolved" |

## E. Sending rules

| ID | Rule |
|---|---|
| E1 | **Zero attachments** on every mail; documents only on the link |
| E2 | One mail per AWB (courier extra IDs in CC) |
| E3 | Bulk send is concurrency-limited; each case independent; per-row result |
| E4 | A draft in `sending` cannot be sent again (no double send) |
| E5 | Resend increments `resend_count`, records `last_resent_at` |
| E6 | Consol Touch 2 is a reply (`In-Reply-To`) to the Touch 1 mail; same link |
| E7 | Every send records exact subject / body / recipients and message id |
| E8 | `MAIL_DRIVER=dry_run` records everything and sends nothing |

## F. Template variables

| Variable | Source | When empty |
|---|---|---|
| `{AWB}` | Case | — (required) |
| `{CONSIGNEE_NAME}` | Case | "Customer" |
| `{REASON}` | Courier sheet | — |
| `{MAWB}`, `{IGM_NUMBER}` | IGM sheet | *"will be shared once generated"* |
| `{ARRIVAL_DATE}` | IGM sheet / case | — |
| `{CLEARANCE_TYPE}` | Case | — |
| `{DO_FEE}` | Approved `fee_rules` for today's day offset | "no approved rule" (blocks if required) |
| `{UTR}` | Case | — |
| Link | `create-link` | Required |

## G. IGM sheet rules

| ID | Rule |
|---|---|
| G1 | Composite cell parsed with `MAWB[-\s]*(\d+)\s+(\d+)\s*,?\s*IGM[-\s]*(\d+)\s*,?\s*Flight[-\s]*([^,\s]+)\s*,?\s*Dt\s*-?\s*(.+)` (case-insensitive) |
| G2 | Raw cell stored for audit |
| G3 | Match **by AWB only**; company shown, never used to match |
| G4 | Row states: matched · changed (before / after) · other consol · **new** · unreadable · duplicate |
| G5 | Only matched / changed rows applied; other consol moved only on confirmation |
| G6 | **New** rows (TP / part shipments) highlighted, never created silently |
| G7 | Apply writes MAWB, IGM, flight, flight date, pieces, weight; stamps `manifest_applied_at`, consol `igm_received_at` |

## H. Fees and penalties (Delhi, from the live templates)

> Values in sections H, J and the Consol timings are **Delhi's**. In production they become gateway settings (doc 13, rules G1–G8).

| Item | Rule |
|---|---|
| DO | ₹2,600 + admin fee (₹0 arrival day; ₹1,000 from next day) + 18% GST → **₹3,068 / ₹4,248** |
| Source | Approved `fee_rules` only; day offset from `arrival_date` |
| Late BOE (as quoted in Delhi templates) | ₹5,000/day from day 1 for 3 days; ₹10,000/day from day 4 |
| FEBRK clearance | ₹3,000 + GST; AAI / warehouse and duty at actuals; duty advancement fee |

## I. Stage gates (cargo)

| From | To | Allowed when |
|---|---|---|
| any pre-IGM | `igm_filed` | MAWB + IGM present |
| `igm_filed` | `payment_pending` | Fee rule resolves |
| `payment_pending` | `payment_verified` | UTR entered |
| `payment_verified` | `do_issued` | DO uploaded → placed on link; thread mail |

## J. Thread and reminder rules

| ID | Rule |
|---|---|
| J1 | Reply matched by `In-Reply-To` / `References`, then AWB in subject / body |
| J2 | Matched reply → `thread_state = waiting_us`, `last_inbound_at` |
| J3 | Our reply → `waiting_customer`, `last_outbound_at` |
| J4 | No customer reply for 4 h (default) → `stale` |
| J5 | Stale → one template reminder per stale period; `REMINDER_SENT` dedupe |
| J6 | AI drafts only; auto-send off. Escalation / legal / VIP / urgent → human only. Out-of-office / bounce → ignore |

## K. Courier close-out and hand-off

| ID | Rule |
|---|---|
| K1 | RBOE: approval only; customer approves or raises objection on the link |
| K2 | Docs received → submit → Admin queue → BOE filed or returned with reason |
| K3 | HV hand-off: same case, `division` → cargo, reason required, history and link kept |

## L. Link rules

| ID | Rule |
|---|---|
| L1 | One active link per case |
| L2 | Token random; only SHA-256 hash stored |
| L3 | Downloads by 300-second signed URL; storage path never returned |
| L4 | Every view / upload / download / approval logged |
| L5 | Link page shows status, FedEx documents, required uploads, today's fee (cargo), checklist (courier RBOE) |
| L6 | `role_scope` limits what a link shows (`customer`; `cha` designed) |
