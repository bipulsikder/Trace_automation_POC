# 09 — API and Code Map

Where everything lives in the PoC repository (`cargo-paf`), so a team can find, read and reuse any part.

---

## 1. Repository layout

```
src/
├─ app/
│  ├─ (app)/courier/        import · board · cases · drafts · links · admin
│  ├─ (app)/cargo/          import · board · send-queue · drafts · consol · igm · cases
│  ├─ l/[token]/            public customer link page
│  └─ api/                  route handlers (below)
├─ components/
│  ├─ compose/              compose editor · start compose · bulk send dialog
│  ├─ cargo/                invoice uploader · ready panel · thread panel
│  └─ courier/              thread modal · documents · link card · replies poller
├─ lib/
│  ├─ courier/              parse-pull-sheet · classify · build-import · readiness · comms/
│  ├─ cargo/                parse-sheet · build-import · parse-igm-sheet · igm-match ·
│  │                        consol-igm · arrival-notice-pdf · fees · next-action
│  ├─ compose/              draft-service · variables · preflight · send-draft ·
│  │                        send-cases · system-send
│  ├─ links/                create-link · link-token · resolve-link ·
│  │                        seed-default-docs · upload-fedex-doc
│  ├─ email/                smtp · imap · poll-and-ingest · ingest-email · awb-extract · render
│  ├─ graph/                send-mail · token (Microsoft Graph)
│  ├─ thread/               match-inbound · send-reply · text
│  ├─ followups/            sweep
│  ├─ tiff/                 pool · zip · checksum
│  └─ templates/            presets
├─ workers/                 tiff-worker
supabase/migrations/        0001 … 0056
scripts/                    seed-courier-templates · gen-cargo-sample-sheet · seed-phase1-demo
public/                     sample sheets · fedex-logo.png
```

---

## 2. API routes

All internal routes need a signed-in session and validate input with zod. Public link routes authenticate by token. Cron routes need `CRON_SECRET`.

### Courier

| Method | Route | Does |
|---|---|---|
| POST | `/api/courier/import/preview` | Parse pull sheet, classify, flag problems |
| POST | `/api/courier/import` | Create cases (batch), assign |
| POST | `/api/courier/cases/assign` | Assign / claim |
| GET/PATCH | `/api/courier/cases/[id]` | Case actions: docs received, submit to Admin, **hand-off to Cargo**, assign me, note |
| POST | `/api/courier/cases/[id]/draft` | Compose draft |
| GET | `/api/courier/cases/[id]/draft-status` | Draft state |
| POST | `/api/courier/cases/[id]/link` | Create / get link |
| GET | `/api/courier/cases/[id]/thread` | Conversation |
| POST | `/api/courier/cases/[id]/reply` | Reply on thread |
| POST | `/api/courier/cases/readiness` | Readiness for many cases (**both desks**) |
| POST | `/api/courier/cases/send` | Bulk send ready cases (**both desks**: stream, touch, resend) |
| PATCH | `/api/courier/drafts/[id]` | Edit draft |
| POST | `/api/courier/drafts/[id]/approve` · `/send` | Approve / send one |
| POST | `/api/courier/drafts/approve` · `/send` | Approve / send many |
| GET/POST | `/api/courier/ai-drafts` | AI reply drafts |
| POST | `/api/courier/ai-drafts/[id]/approve-send` | Agent approves AI draft |
| GET | `/api/courier/documents/[id]/view` | View a document |

### Cargo

| Method | Route | Does |
|---|---|---|
| POST | `/api/cargo/import/preview` | Parse sheet for chosen stream; ready / needs call / blocked / reused |
| POST | `/api/cargo/import` | Create / reuse cases, links, consol |
| POST | `/api/cargo/documents/bulk-upload` | Attach converted PDFs by AWB |
| POST | `/api/cargo/cases/[id]/documents` | Upload / replace FedEx document |
| POST | `/api/cargo/cases/[id]/draft` | Compose draft |
| POST | `/api/cargo/cases/[id]/stage` | IGM → payment → verify → DO gates |
| POST | `/api/cargo/cases/[id]/courier-check` | "Keep in cargo" |
| POST | `/api/cargo/cases/[id]/link` | Create / get link |
| GET/POST | `/api/cargo/cases/[id]/thread` · `/reply` | Conversation / reply |
| POST | `/api/cargo/cases/claim` · `/release` | Ownership |
| POST | `/api/cargo/drafts/send-batch` | Bulk send (delegates to `sendCases`) |
| PATCH | `/api/cargo/drafts/[id]` | Edit draft |
| POST | `/api/cargo/drafts/[id]/approve` · `/send`; `/drafts/approve` · `/send` | Approve / send |
| POST | `/api/cargo/consols/[id]/igm` | Upload IGM sheet → classified rows |
| POST | `/api/cargo/consols/[id]/apply` | Apply matched rows |
| POST | `/api/cargo/consols/[id]/build-updates` | Arrival notice PDFs, prepare Touch 2 |

### Public link

| Method | Route | Does |
|---|---|---|
| GET | `/api/l/[token]` | Link page data (no storage paths) |
| POST | `/api/l/[token]/documents` | Customer upload |
| GET | `/api/l/[token]/documents/[id]/download` | 300 s signed URL |
| POST | `/api/l/[token]/checklist/approve` · `/object` | Checklist approve / objection |

### Inbox and scheduled

| Route | Does |
|---|---|
| `/api/inbox/poll`, `/api/inbox/poll-now` | Fetch and match replies |
| `/api/cron/stale-thread-sweep` | Reminders on stale threads |
| `/api/admin/reset-test-data` | Scoped demo reset |

---

## 3. Key functions

| Function | File | What it does |
|---|---|---|
| `classifyReason` | `lib/courier/classify.ts` | Reason / Master → courier category, conflict flag |
| `detectTraps` | `lib/courier/classify.ts` | Missing email, 11-digit AWB, BOM pool, checklist suppressed |
| `detectCargoColumns` | `lib/cargo/parse-sheet.ts` | Finds cargo columns by synonyms |
| `extractEmail` (internal) | `lib/cargo/parse-sheet.ts` | First valid email across candidate columns |
| `resolveCargoType` | `lib/cargo/build-import.ts` | End Result + broker → clearance type or needs call |
| `computeReadiness` | `lib/courier/readiness.ts` | Module-by-module readiness per stream / touch |
| `needsCourierCheck` | `lib/courier/readiness.ts` | ≥ 10 pcs or > 70 kg/pc |
| `composeCaseDraft` | `lib/compose/draft-service.ts` | Template + variables + link + default docs → draft |
| `runPreflight` | `lib/compose/preflight.ts` | Fail-closed checks |
| `sendDraft` | `lib/compose/send-draft.ts` | Send, thread parent, resend tracking, thread state |
| `sendCases` | `lib/compose/send-cases.ts` | Bulk: readiness → compose → check → send |
| `parseIgmSheet` | `lib/cargo/parse-igm-sheet.ts` | Composite MAWB cell, precision guard |
| `classifyIgmRows` | `lib/cargo/igm-match.ts` | matched / changed / other / new / unreadable / duplicate |
| `applyIgmSheet`, `ensureArrivalNotice` | `lib/cargo/consol-igm.ts` | Write manifest; build PDF on link |
| `generateArrivalNoticePdf` | `lib/cargo/arrival-notice-pdf.ts` | Branded PDF with fee tiers |
| `resolveCargoFee` | `lib/cargo/fees.ts` | Approved fee by day offset |
| `createCaseLink` | `lib/links/create-link.ts` | One active link per case |
| `matchInbound` | `lib/thread/match-inbound.ts` | Reply → AWB |
| `runStaleThreadSweep` | `lib/followups/sweep.ts` | Reminders with dedupe |

---

## 4. Tests

| File | Covers |
|---|---|
| `lib/cargo/parse-sheet.test.ts` | Real 25-column header mapping; Value vs value class; pieces / weight |
| `lib/cargo/igm-sheet.test.ts` | Composite cell parsing, damaged AWBs, row states |
| `lib/cargo/arrival-notice-pdf.test.ts` | PDF builds with fee tiers |
| `lib/courier/readiness.test.ts` | Readiness per stream / touch, waiver, big-shipment check |
| `lib/app-url.test.ts` | Link base URL |
| + earlier suites | 67 tests in total |

Run: `npm test` · type check `npx tsc --noEmit` · build `npm run build`.

---

## 5. Scripts and samples

| File | Purpose |
|---|---|
| `scripts/seed-courier-templates.ts` | Seeds all courier + cargo templates and categories |
| `scripts/gen-cargo-sample-sheet.mjs` | Generates the synthetic 25-column cargo sheet |
| `scripts/seed-phase1-demo.ts` | Demo data |
| `public/sample-courier-pull-sheet.xlsx` | Synthetic courier pull sheet (every category) |
| `public/sample-cargo-ubond-sheet.xlsx` | Synthetic cargo sheet |
| `public/fedex-logo.png` | Arrival notice branding |
