# 12 — Glossary and Points to Confirm

## 1. Glossary

| Term | Meaning |
|---|---|
| **AWB / HAWB** | (House) Air Waybill: the shipment number (12 digits for FedEx). The key for everything |
| **MAWB** | Master Air Waybill: the airline's waybill for a consolidation of many AWBs. Known after IGM |
| **IGM** | Import General Manifest for the flight; gives the IGM number. ~14:00 for the morning flight |
| **BOE** | Bill of Entry: the customs declaration. "Prior BOE" = filed before arrival / IGM |
| **DO** | Delivery Order: FedEx's release document, issued against payment |
| **CHA** | Customs House Agent (customs broker). FedEx-nominated for cargo: **Jeena & Co.**, **Sunimpex** |
| **NFBRK** | Non-FedEx broker: customer's own CHA. **No FedEx clearance** |
| **FEBRK** | FedEx broker: **FedEx clearance** through Jeena or Sunimpex |
| **DDP / DDU** | Delivered duty paid (shipper pays) / unpaid (consignee pays) |
| **uBond** | Advance cargo sheet, 2–3 days before arrival; one pre-alert per AWB |
| **Consol** | Same-day consolidation sheet; two mails per AWB (Touch 1, Touch 2) |
| **Touch 1 / Touch 2** | Consol morning "arriving today" mail / afternoon arrival notice |
| **TP shipment** | Transhipment: can appear in the IGM sheet without being in the morning Consol |
| **Part shipment** | Only some pieces arrived (`Pcs code = P`) |
| **Pull sheet** | Courier exception list from the Classifier team |
| **Classifier team** | Team that flags courier shipments needing action |
| **Master** | Pull-sheet column holding the authoritative courier category |
| **RBOE** | Courier category where the customer only approves a FedEx-prepared checklist |
| **CI** | Commercial invoice |
| **PIMS** | Import monitoring registration required for certain items |
| **SVB / CBE14** | Special Valuation Branch / courier bill of entry form requiring certificates |
| **MOC / DGCA** | Ministry / aviation regulator NOC for restricted items |
| **R&R** | Return and repair / re-export |
| **HV / MV / LV** | High / medium / low value class |
| **FedEx Admin** | Team that files courier BOEs under FedEx's courier licence |
| **IGM team** | Internal team that produces IGM data and handles payment and DO |
| **Trace team** | Delhi team that owns customer communication for imports |
| **ACCS** | FedEx system holding shipment data and invoice images |
| **ICEGATE** | Indian Customs electronic gateway (IGM / BOE filing) |
| **UTR** | Bank reference of the customer's NEFT / IMPS payment |
| **KYC / IEC / AD code** | Customer identity documents / Importer-Exporter Code / bank Authorised Dealer code |
| **Secure link** | Per-AWB page `/l/<token>`: documents, uploads, status, fee, approval |
| **Readiness** | System check that an AWB has everything needed to send |
| **Pre-send check (preflight)** | Fail-closed check run on every mail right before sending |
| **Fee rule** | Approved DO fee per day offset (₹3,068 arrival day, ₹4,248 next day) |
| **Big-shipment check** | ≥ 10 pieces or > 70 kg per piece held for a courier / cargo decision |
| **dry_run** | Mail mode that records everything and sends nothing |

---

## 2. Points for ops / compliance to confirm

Found while reading the live templates and sheets. The system can follow either answer; the wording customers see should be decided by the owners.

| # | Point | What we found |
|---|---|---|
| 1 | **Jeena and Sunimpex templates name the wrong broker** | The Jeena template says Sunimpex will clear; the Sunimpex template names Jeena in its liability line. The PoC takes the broker from the case |
| 2 | BOE penalty start day | Delhi: ₹5,000 from day 1. Kolkata: from day 2 |
| 3 | Penalty basis | Templates: day bands. An older internal note: duty bands under / over ₹10 lakh |
| 4 | Legal citation | Text says Notification 34/2021; attachment folder says Circular 08/2021 |
| 5 | Document deadline | 3 hours vs 16:30 vs 14:30 (customs holiday) vs 18:00 office close |
| 6 | Duty advancement fee | 2% of invoice (Delhi FEBRK) vs 2% of duty + EDI (Kolkata) |
| 7 | "Service Tax" line in FEBRK templates | Replaced by GST since 2017 |
| 8 | Courier document lists | PIMS, Invoice Objection, Content, Duty, MOC lists pending ops confirmation |
| 9 | Reminder timing | 4 h stale default; 24 h post-arrival escalation suggested |
| 10 | Payment notifications | Should "payment received" notify the customer, or only "DO issued"? |
