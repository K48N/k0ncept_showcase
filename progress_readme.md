# k0ncept — progress, history & audit

| | |
|---|---|
| **Purpose** | How we got here, what shipped when, what’s left |
| **Companion** | [`pitch_readme.md`](pitch_readme.md) (EN) · [`pitch_readme.de.md`](pitch_readme.de.md) (DE) — full product & technical reference |
| **Last updated** | June 2026 |

---

## How to read this document

- **`pitch_readme.md`** — what the product *is* (for operators, investors, engineers).
- **This file** — how it *became* that (build chronology, fixes, audits, deploy notes).

---

## Executive summary (current state)

| Area | Status |
|------|--------|
| Public booking + portal | ✅ Production-ready |
| Admin booking inbox + deposit workflow | ✅ Core flows complete |
| Outbound email (Resend + queue) | ✅ Sync worker + cron retries |
| Inbound email (Cloudflare Posteingang) | ✅ Artist + studio inboxes, reply, state sync |
| Payment reconciliation | ✅ SEPA live; **Stripe + GoCardless framework-ready** (keys only) |
| Slot holds & overlap | ✅ 7-day auto-expiry + manual release |
| Multi-admin / roles | ✅ `admin` · `owner` · `artist` · `booking` |
| Studio branding (live Setup) | ✅ Logo, sign-off, fonts, accent — no redeploy |
| Artist profiles | ✅ Avatar + signature (`/admin/profile`) |
| WASM reference pipeline | ✅ Client-side optimize before upload |
| PDF Anzahlungsrechnung | ✅ On deposit confirm (when tax env set) |
| GDPR | ✅ Export, anonymize, automated retention |
| Local-first admin | ✅ Optional (Yjs) — convention Wi-Fi use case |
| Paperless waivers + kiosk | ✅ Separate token; GDPR PDF shredding |
| POS / KassenSichV | ✅ Implemented — mock mode until Fiskaly/SumUp keys |
| ERP (inventory + settlements) | ✅ Auto-burn, guest expiry, Provisionsabrechnungen |
| Instagram Omnichannel CRM | ⏸️ Phase 3 deferred — Posteingang email is live |
| Email retry button | ❌ Failures visible; manual re-trigger |
| Gmail/Apple Mail sync | ❌ Roadmap |

---

## How we got here

A phased build log — from “booking form on a website” to a full studio operating system. Phases overlap; order reflects when each capability became production-usable.

### Phase 1 — Foundation (booking core)

**Goal:** Replace spreadsheet intake with a real booking pipeline.

| Milestone | What shipped |
|-----------|--------------|
| Public booking form | Tattoo / piercing / laser types; consent timestamp |
| `appointments` schema | Status flow: `pending` → deposit → `approved` → `completed` |
| Admin inbox | Tabbed queue, detail panel, deposit request, approve/reject |
| Attachments | Supabase Storage + `appointment_attachments` |
| Overlap prevention | Postgres `prevent_double_booking` trigger per artist |
| Client portal | Token-based `/termin/:token`; view status + deposit info |

**Pain solved:** scattered inquiries → one structured queue.

---

### Phase 2 — Deposits & calendar (money on the calendar)

**Goal:** Holds block the calendar; abandoned requests must not block forever.

| Milestone | What shipped |
|-----------|--------------|
| Deposit workflow | Verwendungszweck generation; `awaiting_deposit` holds |
| 7-day hold expiry | `DEPOSIT_HOLD_DAYS`; manual **Vormerkung aufheben** |
| Portal postpone/cancel | Client actions → studio email + inbox badge |
| Per-artist calendar | `/admin/calendar`; filter by artist |
| Unified bank info | `DEPOSIT_PAYMENT_INFO` → portal + emails (one source) |
| Walk-in booking | `/admin/book` + studio notification email |

**Pain solved:** double-booking and “ghost holds” from unpaid requests.

---

### Phase 3 — Reliability layer (nothing blocks the save)

**Goal:** Third-party outages must not lose bookings or payments.

| Milestone | What shipped |
|-----------|--------------|
| `email_jobs` outbox | Booking returns `201` before Resend |
| Retry backoff | 1m → 5m → 15m → 1h → 4h |
| `email_delivery_log` | Failure visibility in Admin Tools |
| `payment_confirmation_jobs` | Stripe / SEPA confirm decoupled from HTTP |
| Unified `confirmDeposit()` | One path for webhook, cron, and admin |
| XState state machine | Invalid transitions rejected server-side |

**Key fix (2026):** `queueAppointmentEmail()` was fire-and-forget on Vercel — emails queued but never sent when the serverless function exited. **Fix:** synchronous `processEmailJobById()` in the queue handler so sends complete in-process.

**Pain solved:** “booking saved but no email” and silent deposit confirmations.

---

### Phase 4 — Studio identity (live branding)

**Goal:** Owner changes copy, colors, hours without redeploying the site.

| Milestone | What shipped |
|-----------|--------------|
| `studio_settings` singleton | Name, contact, hours, accent, socials from DB |
| SSR homepage | `prerender = false` — Setup saves appear immediately |
| Google reviews framework | Places API + Setup URL → Place ID |
| Guestbook toggle | Disable entirely from Setup |
| Custom fonts | Owner-editable font families (migration `0002`) |
| Role: `owner` | Studio ops without full platform admin |

**Pain solved:** developer-dependent branding changes.

---

### Phase 5 — Payments depth (stop chasing deposits)

**Goal:** End-to-end reconciliation — not just “here’s our IBAN.”

| Milestone | What shipped |
|-----------|--------------|
| Portal `DepositCheckout` | SEPA tab live; Stripe PaymentElement framework-ready |
| Stripe webhook | Amount validation, idempotency, `confirmDeposit()` |
| GoCardless SEPA cron | Bank feed match on Verwendungszweck; orphan alerts |
| Anzahlungsrechnung PDF | `pdf-lib` + §14 UStG fields on confirm |
| Payment safeguards | Job dedup, state machine guard, slot conflict check |

**Status today:** SEPA instructions and manual admin confirm are **live**. Stripe instant pay and GoCardless auto-match are **code-complete** — need live API keys and one-time bank link.

**Pain solved:** artists manually matching transfers in banking apps (still the default until GoCardless/Stripe keys are set).

---

### Phase 6 — Media & analytics (studio intelligence)

| Milestone | What shipped |
|-----------|--------------|
| WASM reference pipeline | Rust → browser resize, linework contrast, WebP |
| D3 analytics | Body placement heatmap, revenue trend, payment donut |
| Gallery + artist tags | Portfolio CMS linked to `studio_artists` |

**Pain solved:** huge reference uploads breaking serverless; no visibility into placement trends or deposit revenue.

---

### Phase 7 — Multi-user admin (roles)

**Goal:** Artists and Kasse get their own login — not one shared password.

| Milestone | What shipped |
|-----------|--------------|
| `admin_users` table | scrypt passwords, roles, activate/deactivate |
| Role matrix | Page + API access in `roles.ts` |
| Compact nav | Artist / Kasse layouts |
| Env kiosk secrets | `ARTIST_SECRET`, `BOOKING_SECRET`, `OWNER_SECRET` for iPad |

**Note:** Early audit (June 2026) listed multi-admin as “not started” — that was stale; roles shipped in the same development arc as owner Setup.

---

### Phase 8 — Inbound email (Posteingang)

**Goal:** Artist and studio mail in the admin — not a separate Gmail tab.

| Milestone | What shipped |
|-----------|--------------|
| Cloudflare Email Routing | Worker → `POST /api/webhooks/inbound-email` |
| `inbound_messages` | Routing: artist alias, studio addresses, `booking+<ref>@` |
| Studio collective inbox | Open artist mail visible to owner until reply / Bearbeitet |
| Mailbox UI | List preview, detail, read/archive/delete/bearbeitet |
| Unread badge | Nav count + realtime sync |
| Branded replies | `renderEmailLayout` + correct Reply-To per thread |
| E-Mail Setup | Alias mapping, diagnostics, webhook test |

**Pain solved:** inbound mail invisible to the booking system; no reply audit trail.

**Ops note:** Resend “Live” is **outbound only**. Inbound requires Cloudflare Worker + `INBOUND_EMAIL_WEBHOOK_SECRET` on Vercel.

---

### Phase 9 — Branding in email & artist voice (latest)

**Goal:** Emails look like the studio; artists sign replies as themselves.

| Milestone | What shipped |
|-----------|--------------|
| Studio logo upload | Favicon + email header + OG image |
| Configurable Grußformel | Default “Beste Grüße” in Setup |
| Artist `/admin/profile` | Profile image + optional signature (name/alias read-only) |
| Reply sign-off block | Grußformel → artist name → signature → optional avatar |
| Migration `0029` | `studio_logo_url`, `email_signoff_closing`, profile fields |

---

### Phase 10 — Convention resilience (optional, deliberate)

**Goal:** Walk-in booking works when Wi-Fi doesn’t.

| Milestone | What shipped |
|-----------|--------------|
| Yjs + IndexedDB | `LocalFirstProvider`, offline mutation queue |
| Replay on reconnect | FIFO sync to existing REST endpoints |
| Setup toggle | Off by default; `PUBLIC_ENABLE_LOCAL_FIRST` override |

**Why it exists:** tattoo conventions and guest spots routinely have **terrible Wi-Fi**. Local-first is not a demo feature — it targets real floor operations at events.

---

## Platform phases (product roadmap alignment)

These map to the engineering roadmap discussed in product planning — distinct from the chronological build log above.

| Phase | Name | Status | Summary |
|-------|------|--------|---------|
| **0** | Core architecture hardening | ✅ | `studio_stations`, station-aware overlap trigger, portfolio enforcer, Stripe chargeback fields, offline conflict sync |
| **0.5** | Stations admin UI | ✅ | Owner manages chairs/workstations without SQL |
| **3** | Omnichannel CRM (Instagram) | ⏸️ **Skipped for now** | Meta Graph API → Posteingang is architected (ASR-021) but **not built** — we prioritized German compliance depth (waivers, POS, ERP) over another inbound channel while email Posteingang already works |
| **4** | Paperless studio | ✅ | Digital waivers (`/kiosk/waiver`), hygiene LOTs, `medical_waivers` bucket, GDPR shredder |
| **5** | Compliant checkout (POS) | ✅ | `pos_transactions`, Fiskaly TSE + SumUp services, receipt PDF, Kassenbuch |
| **5.5** | Sandbox / mock adapters | ✅ | Missing API keys → mock TSE signature + simulated SumUp terminal; production-ready code paths |
| **6** | Automated ERP | ✅ | Inventory burn, guest artist expiry, monthly Provisionsabrechnungen / Gutschriften |

**Deploy note:** Phases 0–6 need migrations `0034`–`0038` via `supabase db push` **and** an app redeploy.

---

## Production incident log (fixes that shaped the build)

| Issue | Root cause | Fix |
|-------|------------|-----|
| Emails queued, never sent | Vercel function returned before worker ran | Synchronous `processEmailJobById()` in queue |
| Client booking 500 | Missing `site.url` in email builder | `emailStudioContext().siteUrl` |
| Admin “Anzahlung anfordern” crash | XState v5 `.transition()` / `actionExecutor` | `getNextSnapshot()` |
| Inbox crash on open | `getTime()` on ISO string in hold check | `new Date(iso).getTime()` |
| Inbound mail not arriving | Worker env / webhook secret mismatch | Documented Worker vars; diagnostics UI |
| Git vs production drift | Features deployed before commit | Incremental doc + migration index updates |

---

## Mega audit (June 2026 — current)

### ✅ Verified / healthy

| Area | Finding |
|------|---------|
| **Auth** | Cookie sessions; role-gated pages + APIs |
| **Overlap** | DB trigger + public form hints |
| **Deposit flow** | XState guards; unified `confirmDeposit()` |
| **Email outbound** | Queue + sync send; booking never blocked |
| **Email inbound** | Webhook + routing + Posteingang UI |
| **GDPR** | Export, anonymize, weekly retention cron |
| **Payments** | Amount checks, webhook idempotency, job dedup |
| **Invoicing** | PDF on confirm; Storage + attachment |
| **WASM** | Pre-built artifacts committed for Vercel |
| **Build** | `npm run build` passes |
| **Storage RLS** | `medical_waivers` + `settlement_invoices` — deny-all public; service role only |
| **Kiosk vs portal tokens** | Separate `kiosk_token_hash` / `portal_token_hash` (migration `0036`) |
| **POS mock mode** | Fiskaly/SumUp absent → simulated signatures; TEST-BON watermark on receipts |
| **ERP** | Inventory burn on complete; guest artist daily expiry; settlement PDF access gated by role |

### ⚠️ Accepted trade-offs

| Item | Notes |
|------|-------|
| Duplicate file validation | API + `createAppointment()` — defense in depth |
| Gmail sender avatar | Logo in email header; round avatar needs BIMI (roadmap) |
| Postgres queues vs Redis | Sufficient for single-studio load |
| Inventory burn vs POS | Not one SQL transaction — appointment completes first; inventory failure logged via Sentry, does not roll back status |
| WASM on CI | Committed `src/wasm/` — local `npm run wasm:build` after Rust edits |

### ❌ Still open

| Item | Priority |
|------|----------|
| Stripe + GoCardless live keys | High — config only |
| Fiskaly + SumUp live keys | High — POS mock mode today |
| Phase 3 Instagram CRM | High — deferred; email Posteingang sufficient for v1 |
| Email forwarding (Gmail / Apple Mail) | High — roadmap |
| One-click email retry | Medium |
| CI for overlap SQL tests | Medium |
| Consent withdrawal UI | Low |

---

## Original priority checklist (10 ideas)

| # | Idea | Status |
|---|------|--------|
| 1 | Hold expiry + release | ✅ 7-day + manual release |
| 2 | Email delivery log | ✅ Table + Admin Tools (retry UI ❌) |
| 3 | Portal token rotation | ✅ On confirm, deposit, postpone |
| 4 | Unify deposit payment info | ✅ `DEPOSIT_PAYMENT_INFO` |
| 5 | Per-artist availability | ✅ Trigger + API + calendar |
| 6 | Studio alert on deposit | ✅ Email on mark + auto-confirm |
| 7 | Align slot duration | ✅ 2h public hint; exact admin |
| 8 | GDPR delete flow | ✅ Export + anonymize + retention cron |
| 9 | Multi-admin / roles | ✅ **Done** (was stale as “not started”) |
| 10 | SQL trigger tests | ✅ Manual SQL; CI ❌ |

---

## Admin surface (current routes)

| Route | Purpose |
|-------|---------|
| `/admin` | Booking inbox (Realtime) |
| `/admin/book` | Walk-in / direct booking |
| `/admin/calendar` | Week view per artist |
| `/admin/gallery` | Portfolio CMS |
| `/admin/guestbook` | Guestbook CMS |
| `/admin/analytics` | D3 dashboard |
| `/admin/settings` | Studio Setup (logo, sign-off, tax, email) |
| `/admin/users` | Benutzer & roles |
| `/admin/inbox-routing` | E-Mail Setup (inbound) |
| `/admin/messages` | Posteingang |
| `/admin/profile` | Artist avatar + signature + **Meine Abrechnungen** |
| `/admin/pos` | Kasse (POS checkout + Kassenbuch) |
| `/admin/inventory` | Inventar + Wareneingang |
| `/admin/finances` | Provisionsabrechnungen (owner) |
| `/admin/tools` | GDPR, reminders, email preview |

---

## Key migrations (apply incrementally on existing DBs)

| Migration | Adds |
|-----------|------|
| `0001_k0ncept_setup.sql` | Full schema (fresh projects only) |
| `0003_inbound_messages.sql` | Inbound mail + artist aliases |
| `0023_realtime_appointments.sql` | Realtime publication |
| `0025_email_jobs.sql` | Email outbox |
| `0024_payment_infrastructure.sql` | Payment jobs |
| `0027_deposit_invoices.sql` | PDF invoicing |
| `0028_inbound_inbox_state.sql` | Read/archive/handled + threads |
| `0029_studio_branding_profiles.sql` | Logo, sign-off, artist profile |
| `0034`–`0038` | Stations, waivers, kiosk token, POS, ERP settlements |

See [`supabase/migrations/README.md`](supabase/migrations/README.md).

---

## Environment variables (deploy)

| Variable | Required | Purpose |
|----------|----------|---------|
| `SUPABASE_URL` / `SUPABASE_SERVICE_ROLE_KEY` | ✅ | Database + storage |
| `ADMIN_SECRET` | ✅ | Sessions + portal HMAC |
| `RESEND_API_KEY` / `EMAIL_FROM` | ✅ | Outbound email |
| `INBOUND_EMAIL_WEBHOOK_SECRET` | ✅ (if inbound) | Webhook + Cloudflare Worker |
| `CRON_SECRET` | ✅ | Cron routes |
| `DEPOSIT_PAYMENT_INFO` | Recommended | SEPA block |
| `STUDIO_NOTIFICATION_EMAIL` | Recommended | Studio alerts |
| `STRIPE_*` / `GOCARDLESS_*` | Optional | Instant pay + SEPA auto-match |
| `GOOGLE_PLACES_API_KEY` | Optional | Live reviews |

Full list: [`pitch_readme.md` §20](pitch_readme.md#20-infrastructure--operations).

---

## Deploy checklist

- [ ] Apply any pending migrations (`0028`, `0029`, …)
- [ ] Set Resend + inbound webhook secret (if using Posteingang)
- [ ] Cloudflare: catch-all → Worker (`cloudflare/email-inbound-worker.js`)
- [ ] Set `DEPOSIT_PAYMENT_INFO`, `STUDIO_NOTIFICATION_EMAIL`
- [ ] Optional: Stripe keys, GoCardless bank link, Places API key
- [ ] Upload studio logo in Setup; map artist aliases in E-Mail Setup
- [ ] Run `supabase/tests/prevent_double_booking.sql` once (in transaction)
- [ ] Verify: online booking → inbox → deposit email → portal → Posteingang reply

---

## File map (key paths)

```
src/lib/
  appointments/            — state machine, booking logic
  admin/                   — inbox CRUD, roles, profile, GDPR
  email/                   — templates, queue, delivery, signoff
  inbound/                 — routing, Posteingang, send-reply
  invoicing/               — Anzahlungsrechnung PDF
  erp/                     — settlement types, guest artist expiry
  media/                   — WASM reference pipeline
  local-first/             — Yjs offline sync (optional)
  branding/storage.ts      — Logo + profile image upload

src/services/
  pos/                     — Fiskaly, SumUp, receipt, checkout (mock-ready)
  erp/                     — settlement engine + Provisionsabrechnung PDF
  inventoryService.ts      — session pack burn + stock intake

src/components/
  admin/                   — Dashboard, Setup, Posteingang, Profil, Kasse, Inventar, …
  inbox/                   — InboxMailbox, MessageThread, realtime hooks

src/pages/api/
  appointments/            — Public booking + availability
  admin/                   — Authenticated studio APIs
  webhooks/                — Stripe, inbound email
  cron/                    — Reminders, email queue, GDPR, SEPA, guest expiry, monthly settlements

cloudflare/
  email-inbound-worker.js  — Inbound routing → Vercel webhook

supabase/migrations/       — Schema evolution
wasm/reference-processor/  — Rust image pipeline
```

---

<p align="center">
  <sub>Product reference · <a href="pitch_readme.md">pitch_readme.md</a></sub>
</p>
