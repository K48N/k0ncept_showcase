# k0ncept

### Studio Booking Platform — Product & Technical Reference (English)

| | |
|---|---|
| **Audience** | Studio owners, investors, engineers, operators |
| **Status** | Production-ready (single-studio deployment) |
| **Last updated** | June 2026 |
| **Languages** | **English** (this file) · [Deutsch](pitch_readme.de.md) |
| **Companion docs** | [`progress_readme.md`](progress_readme.md) · [`supabase/migrations/README.md`](supabase/migrations/README.md) |

---

## Elevator pitch

**k0ncept** is a booking operating system for tattoo studios — not a form plugin on a spreadsheet.

Customers book on a branded website with live availability and WASM-optimized reference uploads. The studio runs one **live inbox** for deposits, confirmations, and walk-ins. Artists and owners manage calendar, media, POS, inventory, and artist settlements from role-based admin.

**Deposits stop being a chase:** structured bank-reference matching, portal pay-now (Stripe — wired, sandbox until keys), SEPA instructions, and optional GoCardless auto-match — all converging on `confirmDeposit()`.

**Paperless studio:** iPad kiosk for digital waivers (Art. 9 GDPR), hygiene LOT tracking, and automatic reference-image cleanup when the finished tattoo photo is uploaded.

**Compliant checkout:** POS with Fiskaly TSE and SumUp terminal integration — **fully implemented in code**, running in **mock/sandbox mode** until merchant API keys are added. Receipts, cash journal, and immutable transaction records are ready.

**Back-office ERP:** inventory auto-burn per completed session, guest-artist access expiry, and monthly **commission statements / Gutschriften** for freelance artists — never payroll language.

**Email in two lanes:** Resend outbound (queued, retried, branded) and Cloudflare inbound Posteingang. Instagram DMs are **architected but not built** (Phase 3 deferred).

Critical actions **persist first**; side effects run through retryable Postgres queues. Branding lives in `studio_settings` — **no redeploy** to change logo, hours, or accent color.

---

## Table of contents

1. [Executive summary](#1-executive-summary)
2. [Platform phases — what shipped](#2-platform-phases--what-shipped)
3. [Third-party integration status (live vs sandbox)](#3-third-party-integration-status-live-vs-sandbox)
4. [Problem & solution](#4-problem--solution)
5. [Technology stack](#5-technology-stack)
6. [Architecture overview](#6-architecture-overview)
7. [Architecture & system rationale (ASR)](#7-architecture--system-rationale-asr)
8. [Paperless studio & kiosk (Phase 4)](#8-paperless-studio--kiosk-phase-4)
9. [POS & KassenSichV (Phases 5 & 5.5)](#9-pos--kassensichv-phases-5--55)
10. [ERP & settlements (Phase 6)](#10-erp--settlements-phase-6)
11. [Core product surfaces](#11-core-product-surfaces)
12. [Payments & portal](#12-payments--portal)
13. [GDPR & compliance](#13-gdpr--compliance)
14. [Security model](#14-security-model)
15. [Operations & deployment](#15-operations--deployment)
16. [Known gaps & audit notes](#16-known-gaps--audit-notes)
17. [Production checklist](#17-production-checklist)
18. [Roadmap](#18-roadmap)

---

## 1. Executive summary

**k0ncept** replaces phone-only intake, spreadsheet calendars, WhatsApp deposit chasing, paper waivers, and manual commission spreadsheets with one integrated product:

| Domain | Capability |
|--------|------------|
| **Public** | Branded site, booking, gallery, guestbook, client portal (`/termin/:token`) |
| **Studio ops** | Real-time inbox, calendar, walk-in, stations, analytics |
| **Communication** | Outbound email queue + inbound Posteingang |
| **Payments** | Deposits via SEPA (live), Stripe/GoCardless (sandbox-ready) |
| **Paperless** | Digital waiver kiosk (`/kiosk/waiver/:token`), hygiene LOTs |
| **POS** | TSE-signed checkout, SumUp card flow, thermal receipt PDF, cash journal |
| **ERP** | Inventory tracking, guest-artist expiry, monthly commission statements |
| **Compliance** | GDPR export/erasure/retention, §14 UStG deposit invoices |

**Architectural principle:** bookings, payments, and POS transactions are written to Postgres **before** any third-party API call. Email, payment confirmation, and inventory side effects use **retryable queues** or non-blocking hooks with Sentry logging on failure.

**Timezone:** studio operations are intended for **Europe/Berlin**. Some batch jobs still use UTC day boundaries — see [§16](#16-known-gaps--audit-notes).

---

## 2. Platform phases — what shipped

| Phase | Name | Status | Summary |
|-------|------|--------|---------|
| **0** | Core architecture hardening | ✅ Integrated | `studio_stations`, station-aware overlap trigger, portfolio/reference cleanup, Stripe chargeback fields, offline conflict sync |
| **0.5** | Stations admin UI | ✅ Integrated | Owner manages workstations without SQL |
| **3** | Omnichannel CRM (Instagram) | ⏸️ Roadmap | Meta Graph API → Posteingang architected (ASR-021); **not implemented** — email Posteingang is live today |
| **4** | Paperless studio | ✅ Integrated | Waiver kiosk, hygiene LOTs, `medical_waivers` bucket, GDPR waiver shredding |
| **5** | Compliant checkout (POS) | ✅ Integrated | `pos_transactions`, Fiskaly TSE + SumUp services, receipt PDF, Kassenbuch |
| **5.5** | Sandbox / mock adapters | ✅ Integrated | Missing API keys → mock TSE signature + simulated SumUp terminal; `is_mocked_tse` on receipts |
| **6** | Automated ERP | ✅ Integrated | Inventory burn, guest artist expiry, monthly Provisionsabrechnungen / Gutschriften |

**Deploy:** migrations `0034`–`0038` via `supabase db push` **plus** app redeploy.

---

## 3. Third-party integration status (live vs sandbox)

This table is the **source of truth** for what is production-live vs code-complete-but-mocked.

| Integration | Code status | Runtime today | To go live |
|-------------|-------------|---------------|------------|
| **Resend** (outbound email) | ✅ Complete | **Live** (with API key) | Set `RESEND_API_KEY`, `EMAIL_FROM` |
| **Cloudflare** (inbound email) | ✅ Complete | **Live** (when configured) | Worker + `INBOUND_EMAIL_WEBHOOK_SECRET` |
| **Supabase** (DB, storage, realtime) | ✅ Complete | **Live** | `SUPABASE_URL`, service role key |
| **Stripe** (portal instant pay) | ✅ Complete | **Sandbox / off** | `STRIPE_SECRET_KEY`, `STRIPE_WEBHOOK_SECRET`, publishable key |
| **GoCardless** (SEPA reconcile) | ✅ Complete | **Sandbox / off** | GoCardless tokens + bank link; cron runs in framework mode |
| **Fiskaly** (TSE / KassenSichV) | ✅ Complete | **Mock mode** | `fiskaly_api_key`, `fiskaly_api_secret`, `fiskaly_tss_id` in `studio_settings` |
| **SumUp** (card terminal) | ✅ Complete | **Mock mode** | `sumup_merchant_code`; live POST to `/v0.1/checkouts` stubbed behind mock |
| **Google Places** (reviews) | ✅ Complete | **Optional** | `GOOGLE_PLACES_API_KEY` |
| **Meta / Instagram** (DMs) | 📋 Architected only | **Not built** | Phase 3 roadmap |

**Mock behaviour (Phases 5 & 5.5):**

- **Fiskaly absent:** SHA-256 mock signature, artificial 500 ms delay, `is_mocked_tse = true`, large **TEST RECEIPT — NO VALID TSE SIGNATURE** watermark on PDF.
- **SumUp absent:** 3 s simulated terminal read, fake `sumup_checkout_id` (`mock-sumup-…`).
- **Stripe / GoCardless absent:** portal shows SEPA path; instant pay tab and auto-reconcile stay disabled without errors.

No code rewrite is required to go live — **only credentials and a redeploy**.

---

## 4. Problem & solution

### 4.1 Studio pain points

| Pain | Without k0ncept |
|------|-----------------|
| Deposit chasing | Manual bank-app matching; ghost calendar holds |
| Paper waivers | Lost forms, Art. 9 GDPR risk, no audit trail |
| Cash register compliance | No TSE, no immutable receipts, Finanzamt exposure |
| Artist payouts | Spreadsheet commission math; Scheinselbständigkeit wording risk |
| Convention Wi-Fi | Walk-in booking dies when connectivity drops |
| Channel sprawl | Email, DMs, phone — no single queue |

### 4.2 What k0ncept delivers

Structured pipeline (`pending` → deposit → `approved` → `completed`), Postgres overlap authority, decoupled notifications, portal self-service, Posteingang, POS + ERP — one product, one database, role-based admin.

---

## 5. Technology stack

| Layer | Technology |
|-------|------------|
| Framework | Astro 6 SSR + React 19 islands |
| Backend | Supabase Postgres, Storage, Realtime |
| Validation | Zod |
| State machine | XState v5 (appointments) |
| PDF | pdf-lib (invoices, receipts, commission statements) |
| Images | Rust → WASM reference pipeline |
| Hosting | Vercel (+ bundled crons on Hobby plan) |
| Observability | Sentry (optional) |

---

## 6. Architecture overview

```
┌─────────────────────────────────────────────────────────────────────────────┐
│  PUBLIC: Homepage · Booking · Gallery · /termin/:token · /kiosk/waiver/:token │
└───────────────┬───────────────────────────────┬─────────────────────────────┘
                ▼                               ▼
         POST /api/appointments          Portal + Kiosk APIs
                │                               │
                ▼                               ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│  SUPABASE — Postgres · Storage · Realtime                                    │
│  appointments · studio_stations · pos_transactions · inventory_items         │
│  artist_settlements · email_jobs · payment_confirmation_jobs                 │
│  medical_waivers · settlement_invoices (private buckets, deny-all public)    │
└───────┬─────────────────┬──────────────────────┬────────────────────────────┘
        ▼                 ▼                      ▼
   email_jobs         Storage              Realtime inbox
        ▼
     Resend

┌─────────────────────────────────────────────────────────────────────────────┐
│  ADMIN: Inbox · Calendar · Kasse (/admin/pos) · Inventar · Abrechnungen      │
│  Posteingang · Setup · Benutzer · Profil (artist settlements tab)           │
└─────────────────────────────────────────────────────────────────────────────┘

  Stripe webhook ──► payment_confirmation_jobs ──► confirmDeposit()
  SEPA cron      ──► GoCardless reconcile       ──► confirmDeposit()
  Daily cron     ──► guest artist expiry · email queue · SEPA batch
  Monthly cron   ──► artist settlement generation (1st of month, 06:00 UTC)
```

---

## 7. Architecture & system rationale (ASR)

| ID | Topic | Decision | Rationale |
|----|-------|----------|-----------|
| **ASR-001** | Product scope | Single-studio vertical OS | Deep tattoo workflow > generic scheduler |
| **ASR-002** | Frontend | Astro + React islands | SSR performance + rich admin |
| **ASR-003** | Backend | Supabase stack | Postgres + Realtime + Storage in one |
| **ASR-004** | Email outbound | Resend + Postgres outbox | API routes never block on Resend |
| **ASR-005** | Email inbound | Cloudflare → Worker → webhook | Separate path from Resend |
| **ASR-006** | State machine | XState v5 server-side | Invalid transitions rejected |
| **ASR-007** | Overlap prevention | Postgres trigger (artist **or** station) | Final authority; survives races |
| **ASR-008** | Deposit holds | 7-day expiry on `awaiting_deposit` | Abandoned requests must not block calendar |
| **ASR-009** | Payments | Unified `confirmDeposit()` | Stripe, SEPA cron, admin share one path |
| **ASR-010** | Portal tokens | HMAC hash; rotated on confirm | `/termin` links unguessable |
| **ASR-011** | Branding | `studio_settings` live from DB | No redeploy for copy/colors |
| **ASR-012** | Reference images | WASM preprocess in browser | Serverless-safe uploads |
| **ASR-013** | Invoices | pdf-lib + embedded Helvetica | No external font fetch on Vercel |
| **ASR-014** | GDPR retention | Anonymize after N days; keep stats row | Compliance without killing analytics |
| **ASR-015** | Admin auth | DB users + role matrix | iPad/kasse logins without shared password |
| **ASR-016** | Inbound inbox | Shared row; studio view = filter | No duplicate mail storage |
| **ASR-017** | Email branding | Logo in template header | Visible in all mail clients |
| **ASR-018** | Artist identity | Name admin-managed; voice self-service | Studio controls identity |
| **ASR-019** | Queues in Postgres | No Redis for v1 | Sufficient at studio scale |
| **ASR-020** | Local-first | Optional Yjs | Convention Wi-Fi resilience |
| **ASR-021** | Instagram (planned) | Official Meta API only | 24h window; `HUMAN_AGENT` tag; no scraping |
| **ASR-022** | Reference cleanup | Delete `client_reference` attachments when session completes | Portfolio enforcer: finished tattoo photo replaces intake refs; reduces GDPR surface |
| **ASR-023** | KassenSichV / TSE | Fiskaly integration; immutable `pos_transactions` | German fiscal law requires signed, append-only register data; mock mode until TSS credentials |
| **ASR-024** | Commission statements vs payroll | PDF **Gutschrift / Provisionsabrechnung** only | B2B freelancer documentation; avoid “Lohn”, “Gehalt”, “Payslip” — Scheinselbständigkeit risk |
| **ASR-025** | Kiosk vs portal tokens | Separate `kiosk_token_hash` | Waiver tablet link must not rotate deposit portal URL |
| **ASR-026** | Private medical storage | `medical_waivers` deny-all public RLS | Art. 9 special-category data |
| **ASR-027** | Inventory burn | −1 session pack on `COMPLETE` | Consumables tied to finished sessions; low-stock email |

---

## 8. Paperless studio & kiosk (Phase 4)

| Feature | Detail |
|---------|--------|
| **Waiver kiosk** | `/kiosk/waiver/:token` — tablet signing; token from `kiosk_token_hash` |
| **Portal (unchanged)** | `/termin/:token` — deposit, postpone, cancel via `portal_token_hash` |
| **Storage** | Signed PDFs in private `medical_waivers` bucket |
| **Hygiene** | `hygiene_ink_lot`, `hygiene_needle_lot` on appointments |
| **Admin** | Waiver/hygiene panel in inbox detail; “Kiosk-Link erzeugen” API |
| **GDPR** | Retention cron purges waiver PDFs before appointment anonymization |

> Generating a kiosk link **never** rotates the customer portal token (migration `0036`).

---

## 9. POS & KassenSichV (Phases 5 & 5.5)

| Component | Path / table |
|-----------|--------------|
| Checkout UI | `/admin/pos` — `CashierView`, `Cashbook` |
| API | `GET/POST /api/admin/pos` |
| Transactions | `pos_transactions` — amount, method, TSE fields, `is_mocked_tse` |
| Fiskaly service | `src/services/pos/fiskalyService.ts` |
| SumUp service | `src/services/pos/sumupService.ts` |
| Receipt PDF | `src/services/pos/receiptService.ts` — legal fields + mock watermark |
| Settings columns | `fiskaly_*`, `sumup_merchant_code` on `studio_settings` |

**Payment methods:** cash, SumUp card, mixed, gift card (via existing gift-card redeem).

**Validation:** server rejects amounts ≤ 0 via Zod `.positive()` on `amountEuro`.

---

## 10. ERP & settlements (Phase 6)

| Component | Detail |
|-----------|--------|
| **Inventory** | `inventory_items`; auto −1 “Standard Session Pack” on appointment complete |
| **Low stock** | `studio_inventory_warning` email job (deduped 24 h) |
| **Guest artists** | `is_guest_artist`, `guest_access_expires_at`; daily cron sets `is_active = false` |
| **Settlements** | `artist_settlements` — revenue, material deduction, net commission |
| **PDF** | Provisionsabrechnung / Gutschrift → `settlement_invoices` bucket |
| **Owner UI** | `/admin/finances` — generate, mark paid, download |
| **Artist UI** | `/admin/profile` → **My statements** tab |
| **Cron** | `/api/cron/monthly-settlements` — 1st of month, 06:00 UTC |

**Commission formula:** `(POS revenue − material deductions) × commission_rate%`. Material = completed sessions × session-pack cost.

**Legal wording:** documents and UI use **commission statement / Gutschrift** only — never payroll terms.

---

## 11. Core product surfaces

Condensed reference — full detail in [`progress_readme.md`](progress_readme.md).

| Surface | Route | Notes |
|---------|-------|-------|
| Booking inbox | `/admin` | Realtime; deposit / approve / complete |
| Calendar | `/admin/calendar` | Per-artist week view |
| Walk-in | `/admin/book` | Direct booking |
| Posteingang | `/admin/messages` | Inbound email |
| Setup | `/admin/settings` | Branding, tax, email, toggles |
| Users | `/admin/users` | Roles, guest artist, commission % |
| Kasse | `/admin/pos` | POS checkout + journal |
| Inventar | `/admin/inventory` | Stock + goods receipt |
| Abrechnungen | `/admin/finances` | Commission statements |
| Client portal | `/termin/:token` | Deposits, self-service |
| Waiver kiosk | `/kiosk/waiver/:token` | Tablet signing only |

**Roles:** `admin` · `owner` · `artist` · `booking` — page and API matrix in `src/lib/admin/roles.ts`.

**Appointment flow:** XState-guarded `pending` → `awaiting_deposit` → `approved` → `completed`. Completing requires an **artist result photo**; then client references are deleted (ASR-022).

---

## 12. Payments & portal

| Path | Entry | Converges on |
|------|-------|--------------|
| Stripe webhook | `payment_intent.succeeded` | `scheduleConfirmDeposit()` → `confirmDeposit()` |
| GoCardless cron | Verwendungszweck match | `confirmDeposit()` |
| Admin manual | “Anzahlung erhalten” | `confirmDeposit()` |
| SEPA instructions | Portal + email | Customer transfer; studio confirms |

**Safeguards:** amount checks, Stripe PaymentIntent idempotency, `enqueueConfirmDepositSafe()` skips duplicate pending jobs, overlap trigger on approve.

**Not in scope of deposits:** POS checkout is a **separate** `pos_transactions` stream (session-day register), not auto-linked to deposit invoices.

---

## 13. GDPR & compliance

| Capability | Implementation |
|------------|----------------|
| Export | `/api/admin/gdpr/export` |
| Manual erasure | `/api/admin/gdpr/delete` |
| Retention cron | Weekly; anonymize completed appointments after `GDPR_RETENTION_DAYS` |
| Waiver shredding | Storage delete + null paths on anonymize |
| Special-category data | Waivers in private bucket; no public RLS |

### Supabase backups (Art. 9 note)

Deleting waiver PDFs from Storage **does not** remove them from Supabase **point-in-time / automated backups**. For full Art. 9 compliance, operators must align **backup retention and rotation** with their DPA and documented erasure procedures. This is an **operational** requirement, not yet automated in code. See [§16](#16-known-gaps--audit-notes).

---

## 14. Security model

| Layer | Approach |
|-------|----------|
| Admin auth | HttpOnly cookie sessions; scrypt password hashes |
| Portal / kiosk tokens | HMAC-SHA256 stored hashes; timing-safe compare |
| Storage | `medical_waivers`, `settlement_invoices` — deny-all public policies; access via service role + API role checks |
| RLS | Anon blocked on sensitive tables; server uses service role |
| Cron | `CRON_SECRET` header verification |
| Webhooks | Shared secrets (Stripe signature, inbound email) |

Settlement PDF download: artists may only fetch their own `artist_id`; owner/admin see all.

---

## 15. Operations & deployment

### 15.1 Migrations (incremental)

| Migration | Adds |
|-----------|------|
| `0034` | Stations, architecture hardening |
| `0035` | Waiver / hygiene fields, `medical_waivers` |
| `0036` | Separate `kiosk_token_hash` |
| `0037` | `pos_transactions`, Fiskaly/SumUp settings |
| `0038` | Inventory, settlements, guest artist fields |

```bash
supabase db push   # apply pending migrations
# redeploy app on Vercel
```

### 15.2 Cron schedule (`vercel.json`)

| Job | Schedule | Purpose |
|-----|----------|---------|
| `appointment-reminders` | 07:00 UTC daily | Week/day reminders (06–24 Europe/Berlin window) |
| `daily-maintenance` | 07:15 UTC daily | Email queue, newsletter, SEPA, **guest artist expiry** |
| `gdpr-retention` | 04:00 UTC Mondays | Anonymization batch |
| `monthly-settlements` | 06:00 UTC 1st | Commission statement generation |

### 15.3 Key environment variables

`SUPABASE_*`, `ADMIN_SECRET`, `RESEND_API_KEY`, `CRON_SECRET`, `STRIPE_*`, `GOCARDLESS_*`, `INBOUND_EMAIL_WEBHOOK_SECRET`, `SENTRY_DSN` (optional).

Fiskaly/SumUp keys: currently DB columns on `studio_settings` (no Setup UI yet — set via SQL or future admin panel).

---

## 16. Known gaps & audit notes

Principal-architect review (June 2026). Items below are **documented trade-offs or follow-ups**, not blockers for single-studio rollout.

### 16.1 Timezones (Europe/Berlin vs UTC)

| Area | Current behaviour | Risk |
|------|-------------------|------|
| Appointment reminders | Send window 06–24 **Europe/Berlin** | ✅ Correct |
| Monthly settlements | `previousMonthBounds()` uses **UTC** month; `periodUtcRange()` uses UTC midnight | ⚠️ Transactions near month boundaries may land in wrong period vs studio local books |
| POS cash journal | `listPosTransactionsForDay()` filters UTC day | ⚠️ “Today’s drawer” may differ from Berlin calendar day |
| GDPR retention | Cutoff on `start_time` ISO timestamps | Low risk; consistent but not explicitly Berlin |
| Settlement session count | Completed appointments filtered by `end_time` in UTC range | ⚠️ Edge cases on last day of month |

**Recommendation:** centralise `europeBerlinDayBounds()` and use in POS journal, settlements, and cron period selection.

### 16.2 Payment race conditions

| Scenario | Protection today | Gap |
|----------|------------------|-----|
| Stripe webhook vs admin “confirm deposit” | `confirmDeposit()` status guard; `enqueueConfirmDepositSafe()` skips if pending job or already approved; Stripe PI idempotency | No `SELECT FOR UPDATE` — two simultaneous requests can both read `awaiting_deposit`; second should fail on update, but duplicate **jobs** possible in narrow race |
| SumUp webhook vs manual POS | **N/A** | SumUp **webhook not implemented** — mock returns synchronously in checkout flow |

**Recommendation:** unique partial index on `payment_confirmation_jobs (appointment_id) WHERE status IN ('pending','processing')`; optional optimistic locking on appointments.

### 16.3 GDPR backups

Physical waiver delete + DB anonymization ✅. Supabase backup retention may still contain Art. 9 data → **document in DPA / ops runbook**; consider backup retention ≤ erasure SLA or Supabase project policies.

### 16.4 POS input validation

Server-side Zod `.positive()` blocks 0 and negative amounts ✅. UI should mirror; no duplicate TSE sign for same pending transaction if user double-clicks (button disable recommended).

### 16.5 Settlements vs deposits (commission logic)

**Current rule:** settlement revenue = sum of **`pos_transactions`** (completed) in period, attributed to artist via appointment → `studio_artists` → `admin_users.studio_artist_id`.

| Money event | In settlement? |
|-------------|----------------|
| POS checkout at session | ✅ Yes |
| Deposit (`confirmDeposit`, Stripe, SEPA) | ❌ **No** — not a `pos_transaction` |
| Deposit in prior month, tattoo this month | ❌ Deposit not counted; only POS in current period |

**Implication:** if the studio pays artist commission on **total session value**, the owner must either (a) ring the balance through POS at checkout, or (b) extend settlement logic to include appointment revenue / deposits. **Documented product gap.**

Material deductions count **completed appointments** in period (by `end_time`), independent of POS — may diverge from POS revenue if sessions completed without checkout.

### 16.6 Other notes

- Inventory burn runs **after** appointment update succeeds — not one SQL transaction with status change.
- Artists without `studio_artist_id` linked may miss POS attribution (falls back to `created_by_user_id` only when matching).
- Fiskaly/SumUp/Stripe live activation = credentials only; no structural rewrite needed.

---

<p align="center">
  <sub>
    <a href="pitch_readme.de.md">Deutsch</a> ·
    <a href="progress_readme.md">Build chronology</a> ·
  </sub>
</p>
