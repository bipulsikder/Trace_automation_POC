# 05 — What I Built, Step by Step

This is the submission: for every step of both flows, **the problem**, **what I built**, **how it works**, **where it is in the code**, and **how it was checked**.

---

## Summary

| # | Step | Courier | Cargo | Shared |
|---|---|:-:|:-:|:-:|
| 1 | Import the real sheet | ✅ | ✅ | |
| 2 | Decide the case (category / clearance type), never guess | ✅ | ✅ | |
| 3 | Ownership and work queues | ✅ | ✅ | |
| 4 | Secure link per AWB | | | ✅ |
| 5 | Documents: TIFF → PDF, bulk attach, default FedEx documents | 🟡 uploads only | ✅ | |
| 6 | Templates per category / clearance type | ✅ | ✅ | |
| 7 | Pre-send check (fail-closed) | | | ✅ |
| 8 | Readiness and bulk send, resend | ✅ | ✅ | |
| 9 | Consol two-touch with IGM sheet matching | | ✅ | |
| 10 | Arrival notice PDF | | ✅ | |
| 11 | Replies matched to AWB, thread per AWB | | | ✅ |
| 12 | AI reply drafts (a person sends) | ✅ | 🟡 | |
| 13 | Automatic reminders on silent threads | | | ✅ (schedule 🟡) |
| 14 | RBOE checklist approval on the link | ✅ | ⬜ FEBRK | |
| 15 | Admin queue / IGM → payment → DO gates | ✅ | ✅ (UTR typed 🟡) | |
| 16 | Courier → Cargo hand-off on the same case | ✅ | ✅ | |
| 17 | Big-shipment check (cargo vs courier) | | ✅ | |
| 18 | Boards, timeline, audit, team analytics | | | ✅ |
| 19 | Admin: templates, fee rules, users, test-data reset | | | ✅ |

**Checks:** TypeScript build clean · `npm run build` passes · **67 automated tests** (sheet parsing, readiness, IGM matching, arrival-notice PDF) · demo runs end to end with `MAIL_DRIVER=dry_run`.

---

## Step 1 — Import the real sheet

**Problem.** Courier and cargo sheets are messy: stray headers (cargo's first header is a time like `10:22 AM`), emails in the wrong column, Excel-damaged AWBs, 30+ spellings for one reason.

**What I built.**

| | Courier | Cargo |
|---|---|---|
| Input | 22-column pull sheet | 25-column uBond / Consol sheet; **operator picks the stream first** |
| Column finding | By header name with synonyms; `AWB` + `Reason` required | By header name with synonyms; some fields exact-only (e.g. `Value` is invoice value, never value class) |
| Preview | Category guess + flags: missing email, 11-digit AWB, checklist already sent, duplicate, CEP unresolved, BOM pool | Status per row: ready / needs call / blocked (reason) / reused from uBond; cells editable |
| Batch name | `COURIER-YYYY-MM-DD-AM/PM` | Consol record created for a Consol import |
| Sample | Downloadable synthetic sheet covering every category | Downloadable synthetic sheet in the real 25-column shape |

**Code.** `src/lib/courier/parse-pull-sheet.ts`, `src/lib/courier/build-import.ts`, `src/app/api/courier/import/*` · `src/lib/cargo/parse-sheet.ts`, `src/lib/cargo/build-import.ts`, `src/app/api/cargo/import/*` · UI `src/app/(app)/*/import/import-wizard.tsx`.
**Checked by.** `parse-sheet.test.ts` against the real cargo header row.

---

## Step 2 — Decide the case, never guess

**Courier.** `Master` is authoritative → 11 categories. The `Reason` shorthand (first token before the comma) is a strong signal; keyword fallbacks cover `INVMS`, `DESCI`, `ROD`, `PFEXP`, `VALDS`… Two strong matches = conflict flagged. Nothing matched = **blocked**.
→ `src/lib/courier/classify.ts` (categories loaded from `category_map`, not hard-coded).

**Cargo.** `End Result` + `FedEx Broker` → `nfbrk` / `febrk-jeena` / `febrk-sunimpex`. Bare `FEBRK`, `Calling`, `HOLD` → **Needs call**. Email = first column with a **valid** address across `ConsigneeEmailID`, `Standard Remarks`, etc.
→ `src/lib/cargo/build-import.ts` (`resolveCargoType`), `parse-sheet.ts` (`extractEmail`).

---

## Step 3 — Ownership and work queues

- Courier: owner from `Agent` / `Team`, or "assign to me" on import; BOM rows → pool; claim from the board.
- Cargo: claim / release per case; board filters Mine / Team.
- Every assignment recorded in `case_assignments` and the timeline.
→ `src/app/api/courier/cases/assign`, `src/app/api/cargo/cases/claim|release`.

---

## Step 4 — One secure link per AWB

**What I built.** `/l/<token>`: no login, mobile friendly. Shows status, FedEx documents, required documents with upload, today's fee (cargo), checklist approval (courier RBOE).

| Property | How |
|---|---|
| Token | Random; **only its SHA-256 hash is stored** |
| One per case | Re-used across uBond → Consol → DO and courier → cargo hand-off |
| Downloads | Fresh **5-minute signed URL** per click; storage paths never sent to the browser |
| Scope field | `role_scope` (`customer` today; `cha` designed) |
| Logging | Every open, upload, download, approval in `link_events` |

→ `src/lib/links/create-link.ts`, `link-token.ts`, `resolve-link.ts`, `src/app/l/[token]/page.tsx`, `src/app/api/l/[token]/*`.

---

## Step 5 — Documents

**Problem.** ACCS gives invoices as `.tif`; customers want PDF; attaching per mail is slow.

**What I built.**
- **Bulk drop** of `.tif` / `.pdf` / `.zip` on the import screen or board.
- **TIFF → PDF in the browser**, in a Web Worker pool, so large multi-page TIFFs never reach the server unconverted; checksum to avoid duplicates; zip unpacking.
- **Matched to the AWB by filename** (`<AWB>.tif`, `<AWB>_1.tif`); unmatched files listed.
- **Default FedEx documents** (DO format, bank details, circular, tariff) placed on every cargo link by its template.
- Per-case upload / replace of FedEx documents.

→ `src/workers/tiff-worker.ts`, `src/lib/tiff/{pool,zip,checksum}.ts`, `src/components/cargo/invoice-uploader.tsx`, `src/app/api/cargo/documents/bulk-upload`, `src/lib/links/seed-default-docs.ts`, `upload-fedex-doc.ts`.

---

## Step 6 — Templates

- **Courier:** 11 templates, one per category, each with a required-document list; RBOE in approval mode.
- **Cargo:** `nfbrk`, `febrk-jeena`, `febrk-sunimpex` (pre-alert / Touch 1), `cargo_arrival_notice` (Touch 2), `cargo_reminder`; plus `courier_reminder`.
- Variables filled from the case: `{AWB}`, `{CONSIGNEE_NAME}`, `{REASON}`, `{MAWB}`, `{IGM_NUMBER}` (*"will be shared once generated"* when empty, never blank), `{ARRIVAL_DATE}`, `{CLEARANCE_TYPE}`, `{DO_FEE}` (from fee rules), `{UTR}`.
- Editable by admins in the app; seeded from presets.

→ `src/lib/templates/presets.ts`, `src/lib/compose/variables.ts`, `scripts/seed-courier-templates.ts`, migration `0053`.

---

## Step 7 — Pre-send check (fail-closed)

Every mail is checked **immediately before sending**:

| Check | Blocks when |
|---|---|
| Template | No active template for the category / type |
| Recipient | Missing or invalid email |
| Link | No active secure link |
| Variables | Any unresolved `{…}` |
| Attachments | Template tries to attach a file: **never allowed** |
| Category / broker | Unknown category, bare FEBRK |

A blocked mail never leaves; its reasons are shown on the row.
→ `src/lib/compose/preflight.ts`.

---

## Step 8 — Readiness and bulk send

**Problem.** The VBA script sent everything in one go; the PoC must be as fast, but safe.

**What I built.**
- **Readiness per AWB**, shown as a module strip on the board: template · link · document pack · email · broker · invoice (uBond) · big-shipment check.
- **Send all ready**: one action composes each mail from its matched template, checks it, sends with a concurrency limit, and returns a per-row result (sent / skipped with reason / failed). No per-case drafting needed.
- **Resend** with count and last-resent time.
- Courier also keeps the **draft → approve → send** path for review before sending.

→ `src/lib/courier/readiness.ts` (+ tests), `src/lib/compose/send-cases.ts`, `draft-service.ts`, `send-draft.ts`, `src/components/compose/bulk-send-dialog.tsx`, `src/components/cargo/ready-panel.tsx`, `src/app/api/courier/cases/{readiness,send}`, `src/app/api/cargo/drafts/send-batch`.

---

## Step 9 — Consol two-touch with IGM matching

**Problem.** The Consol morning mail can't carry MAWB/IGM; at 14:00 they are copied into new mails by hand; new AWBs appear.

**What I built.**
1. **Touch 1** in the morning with *"will be shared once generated"*; waived if a uBond mail went in the last 24 h.
2. **IGM sheet upload** on the consol: reads the composite cell `MAWB-… IGM-… Flight-… Dt-…`; keeps the raw cell for audit.
3. **Classify each row by AWB only**: matched · changed (before/after) · other consol · **new (TP / part)** · unreadable · duplicate.
4. **Apply**: MAWB, IGM, flight, date, pieces, weight written to every matched case; consol timestamps stamped.
5. **Touch 2** as a **reply in the Touch 1 thread** (`In-Reply-To`), same link.

→ `src/lib/cargo/parse-igm-sheet.ts`, `igm-match.ts`, `consol-igm.ts`, `src/app/api/cargo/consols/[id]/{igm,apply,build-updates}`, `src/app/(app)/cargo/consol/[id]/igm-workbench.tsx`, migrations `0054`, `0055`.
**Checked by.** `igm-sheet.test.ts` (composite cell parsing, damaged-AWB rejection, row states).

---

## Step 10 — Arrival notice PDF

FedEx-branded one-page PDF per AWB: consignee, HAWB, MAWB, IGM, flight and date, pieces, weight, part-arrival code, DO charges for arrival day and next day **from approved fee rules**, payment instructions. Generated after Apply and placed on the AWB's link before Touch 2.
→ `src/lib/cargo/arrival-notice-pdf.ts` (+ test), logo `public/fedex-logo.png`.

---

## Step 11 — Replies and threads

- Inbox polled (IMAP); each mail matched to its AWB by thread headers, then AWB in the subject or body.
- One conversation per AWB, quoted history stripped.
- Thread state: waiting customer / reply needed / stale, shown on the boards.
- Replies sent from the case go on the same thread.
→ `src/lib/email/{imap,poll-and-ingest,ingest-email,awb-extract}.ts`, `src/lib/thread/{match-inbound,send-reply,text}.ts`, migration `0050`.

---

## Step 12 — AI reply drafts

- Courier: reply classified (acknowledgement / needs answer / escalate / ignore); AI drafts from this AWB's facts; agent approves, edits or rejects. **Auto-send exists as a route but is switched off.**
- Cargo: thread composer; AI draft to be extended 🟡.
- Without an AI key the feature hides; everything else works.
→ `src/lib/courier/comms/{classify-reply,generate-draft-reply}.ts`, `src/app/api/courier/ai-drafts/*`.

---

## Step 13 — Automatic reminders

A sweep finds stale threads (no customer reply after the threshold, 4 h default), sends one templated reminder per stale period through the same pipeline, and records `REMINDER_SENT` to avoid repeats. Works for both desks.
→ `src/lib/followups/sweep.ts`, `src/app/api/cron/stale-thread-sweep`. 🟡 Needs a production schedule.

---

## Step 14 — Checklist approval on the link

Courier RBOE: customer reviews the checklist and clicks **Approve** or **Raise objection** with a comment. Recorded on the case and in link events.
→ `src/app/api/l/[token]/checklist/{approve,object}`. ⬜ To extend to the FEBRK duty checklist.

---

## Step 15 — Close-out: Admin queue (courier) and IGM → payment → DO (cargo)

| Courier | Cargo |
|---|---|
| Docs received → submit → **Admin queue** → BOE filed or returned with reason | **IGM filed → payment requested → payment verified → DO issued**, each only after the previous one |
| | Fee for today's day offset from **approved** fee rules; "no approved rule" shown instead of a guess |
| | DO placed on the link + short thread mail |

→ `src/app/api/courier/cases/[id]`, `src/app/(app)/courier/admin`, `src/app/api/cargo/cases/[id]/stage`, `src/lib/cargo/fees.ts`, `src/app/(app)/cargo/igm/igm-board.tsx`, migration `0051`.

---

## Step 16 — Courier → Cargo hand-off

One action with a reason: the case's division flips to Cargo, keeping AWB, link, documents, thread and timeline (`HANDED_OFF_TO_CARGO`). No duplicate case.
→ `src/app/api/courier/cases/[id]` (`handoff_cargo`), `case-actions.tsx`.

---

## Step 17 — Big-shipment check

Cargo AWBs with **≥ 10 pieces or > 70 kg per piece** are held before the first mail until a person decides "Keep in cargo". Who and when is recorded.
→ `src/lib/courier/readiness.ts` (`needsCourierCheck`), `src/app/api/cargo/cases/[id]/courier-check`, `courier-check-banner.tsx`, migration `0056`.

---

## Step 18 — Visibility

- **Courier board**: triage-first groups (blocked → needs email → needs assign → draft ready → ready to send → waiting customer → failed), urgency badges, pull progress, quick actions, keyboard shortcuts.
- **Cargo board**: readiness groups, module strip, send / resend.
- **Consols**, **IGM & DO**, **Send queue**, **Drafts**, **Links** pages.
- **Case timeline** and **audit log** for every action; **team analytics**.

---

## Step 19 — Admin

Templates, fee rules (approval required before customers see them), users and roles (admin / lead / operator / reviewer / viewer, per desk), mailboxes, and a **scoped test-data reset** (courier / cargo / all, dry-run preview first).

---

## Database work in this PoC

| Migration | What it added |
|---|---|
| 0047 | One link model for both desks |
| 0048 | `sending` status on drafts (no double-send) |
| 0049 | Cargo workspace |
| 0050 | Inbound matching |
| 0051 | Cargo fees and timeline |
| 0052 | Board run column |
| 0053 | Arrival notice and reminder templates |
| 0054 | Streams (uBond / Consol), touches, resend tracking, consol timestamps |
| 0055 | Consol manifest fields; pieces / weight / contents on cases |
| 0056 | Big-shipment check fields |

---

## Not built yet (designed)

| Item | Where designed |
|---|---|
| One-click case for new IGM-sheet AWBs (TP / part) | Doc 04 §5 |
| Part-arrival highlight on the board | Doc 04 §5 |
| "IGM not generated — ICEGATE issue" mail; day-2 DO notice | Doc 04 §8 |
| CHA view: documents, checklist upload, "BOE filed" | Doc 04 §7 |
| FEBRK duty checklist approval on the link | Doc 04 §7 |
| Cargo AI reply drafts | Doc 05 step 12 |
| Manager SLA view (age per stage) | Doc 02 §8 |
| Browser end-to-end tests | Doc 07 |
