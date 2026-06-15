# k0ncept

### Studio-Booking-Plattform — Produkt- & Technikreferenz (Deutsch)

| | |
|---|---|
| **Zielgruppe** | Studio-Inhaber, Investoren, Entwickler, Betrieb |
| **Status** | Produktionsreif (Single-Studio-Deployment) |
| **Stand** | Juni 2026 |
| **Sprachen** | [English](pitch_readme.md) · **Deutsch** (diese Datei) |
| **Begleitdokumente** | [`progress_readme.md`](progress_readme.md) |

---

## Elevator Pitch

**k0ncept** ist kein Buchungsformular auf einem Excel-Sheet — sondern ein Betriebssystem für Tätowierstudios.

Kunden buchen auf einer gebrandeten Website mit Live-Verfügbarkeit und WASM-optimierten Referenz-Uploads. Das Studio steuert Anfragen, Anzahlungen, Walk-ins, Kasse, Inventar und Artist-Abrechnungen aus einer **Live-Inbox** und rollenbasiertem Admin.

**Schluss mit Anzahlungs-Jagd:** strukturierter Verwendungszweck, Portal-Zahlung (Stripe — im Code, Sandbox bis Keys), Überweisungsinfos und optional GoCardless-Auto-Match — alles läuft über `confirmDeposit()`.

**Paperless Studio:** iPad-Kiosk für digitale Einverständniserklärungen (Art. 9 DSGVO), Hygiene-Chargen (LOT) und automatisches Löschen der Kunden-Referenzbilder, sobald das Ergebnis-Foto hochgeladen ist.

**Rechtssichere Kasse:** POS mit Fiskaly-TSE und SumUp-Terminal — **vollständig implementiert**, derzeit im **Sandbox-/Mock-Modus**, bis Merchant-Keys hinterlegt sind. Belege, Kassenbuch und unveränderliche Transaktionsdaten sind fertig.

**Back-Office ERP:** Inventar-Auto-Verbrauch pro Session, Gast-Artist-Zugangsablauf und monatliche **Provisionsabrechnungen / Gutschriften** für freie Mitarbeiter — bewusst **ohne** Lohn- oder Gehalts-Sprache.

**E-Mail in zwei Bahnen:** Resend outbound (Queue, Retries, Branding) und Cloudflare-Posteingang. Instagram-DMs sind **konzipiert, aber nicht gebaut** (Phase 3 verschoben).

Kritische Aktionen werden **zuerst in Postgres persistiert**; E-Mail, Zahlungen und Inventar laufen über Queues oder nicht-blockierende Hooks. Branding liegt in `studio_settings` — **kein Redeploy** für Logo, Öffnungszeiten oder Akzentfarbe.

---

## Inhaltsverzeichnis

1. [Executive Summary](#1-executive-summary)
2. [Plattform-Phasen — Was live ist](#2-plattform-phasen--was-live-ist)
3. [Drittanbieter-Status (Live vs. Sandbox)](#3-drittanbieter-status-live-vs-sandbox)
4. [Problem & Lösung](#4-problem--lösung)
5. [Technologie-Stack](#5-technologie-stack)
6. [Architektur-Überblick](#6-architektur-überblick)
7. [Architecture & System Rationale (ASR)](#7-architecture--system-rationale-asr)
8. [Paperless Studio & Kiosk (Phase 4)](#8-paperless-studio--kiosk-phase-4)
9. [Kasse & KassenSichV (Phasen 5 & 5.5)](#9-kasse--kassensichv-phasen-5--55)
10. [ERP & Provisionsabrechnungen (Phase 6)](#10-erp--provisionsabrechnungen-phase-6)
11. [Produkt-Oberflächen](#11-produkt-oberflächen)
12. [Zahlungen & Portal](#12-zahlungen--portal)
13. [DSGVO & Compliance](#13-dsgvo--compliance)
14. [Security Model](#14-security-model)
15. [Betrieb & Deployment](#15-betrieb--deployment)
16. [Bekannte Lücken & Audit-Hinweise](#16-bekannte-lücken--audit-hinweise)
17. [Production Checklist](#17-production-checklist)
18. [Roadmap](#18-roadmap)

---

## 1. Executive Summary

**k0ncept** ersetzt Telefon-only-Intake, Excel-Kalender, WhatsApp-Anzahlungs-Chaos, Papier-Einverständnisse und manuelle Provisions-Tabellen durch ein integriertes System:

| Bereich | Funktion |
|---------|----------|
| **Öffentlich** | Gebrandete Site, Buchung, Galerie, Gästebuch, Kundenportal (`/termin/:token`) |
| **Studio** | Live-Inbox, Kalender, Walk-in, Arbeitsplätze (Stations), Analytics |
| **Kommunikation** | Outbound-E-Mail-Queue + Posteingang |
| **Zahlungen** | Anzahlung per SEPA (live), Stripe/GoCardless (sandbox-bereit) |
| **Paperless** | Waiver-Kiosk (`/kiosk/waiver/:token`), Hygiene-LOTs |
| **Kasse** | TSE-signierter Checkout, SumUp-Kartenflow, Kassenbon-PDF, Kassenbuch |
| **ERP** | Inventar, Gast-Artist-Ablauf, monatliche Provisionsabrechnungen |
| **Compliance** | DSGVO Export/Löschung/Retention, §14-UStG-Anzahlungsrechnung |

**Architektur-Prinzip:** Buchungen, Zahlungen und Kassenvorgänge landen **zuerst in Postgres**, bevor Drittanbieter-APIs laufen. E-Mail, Zahlungsbestätigung und Inventar nutzen **retry-fähige Queues** bzw. nicht-blockierende Hooks mit Sentry-Logging bei Fehlern.

**Zeitzone:** Betrieb ist auf **Europe/Berlin** ausgelegt. Einige Batch-Jobs nutzen noch UTC-Tagesgrenzen — siehe [§16](#16-bekannte-lücken--audit-hinweise).

---

## 2. Plattform-Phasen — Was live ist

| Phase | Name | Status | Kurzbeschreibung |
|-------|------|--------|------------------|
| **0** | Core Architecture Hardening | ✅ Integriert | `studio_stations`, Overlap-Trigger (Artist **oder** Station), Referenz-Cleanup, Chargeback-Felder |
| **0.5** | Stations-Admin-UI | ✅ Integriert | Inhaber verwaltet Arbeitsplätze ohne SQL |
| **3** | Omnichannel CRM (Instagram) | ⏸️ Roadmap | Meta Graph API → Posteingang (ASR-021); **nicht gebaut** — E-Mail-Posteingang ist live |
| **4** | Paperless Studio | ✅ Integriert | Waiver-Kiosk, Hygiene-LOTs, `medical_waivers`, DSGVO-Shredder |
| **5** | Compliant Checkout (POS) | ✅ Integriert | `pos_transactions`, Fiskaly + SumUp, Kassenbon, Kassenbuch |
| **5.5** | Sandbox / Mock-Adapter | ✅ Integriert | Fehlende Keys → Mock-TSE + simuliertes SumUp-Terminal |
| **6** | Automated ERP | ✅ Integriert | Inventar-Burn, Gast-Artist-Expiry, Provisionsabrechnungen |

**Deploy:** Migrationen `0034`–`0038` via `supabase db push` **plus** App-Redeploy.

---

## 3. Drittanbieter-Status (Live vs. Sandbox)

Diese Tabelle ist die **Referenz**, was produktiv läuft vs. was im Code fertig, aber noch gemockt ist.

| Integration | Code | Laufzeit heute | Live schalten |
|-------------|------|----------------|---------------|
| **Resend** (Outbound) | ✅ | **Live** (mit Key) | `RESEND_API_KEY`, `EMAIL_FROM` |
| **Cloudflare** (Inbound) | ✅ | **Live** (wenn konfiguriert) | Worker + `INBOUND_EMAIL_WEBHOOK_SECRET` |
| **Supabase** | ✅ | **Live** | URL + Service-Role-Key |
| **Stripe** (Portal) | ✅ | **Sandbox / aus** | Stripe-Keys + Webhook-Secret |
| **GoCardless** (SEPA-Match) | ✅ | **Sandbox / aus** | Tokens + Bank-Verknüpfung |
| **Fiskaly** (TSE) | ✅ | **Mock-Modus** | `fiskaly_*` in `studio_settings` |
| **SumUp** (Terminal) | ✅ | **Mock-Modus** | `sumup_merchant_code` |
| **Google Places** | ✅ | **Optional** | Places API Key |
| **Meta / Instagram** | 📋 Nur Architektur | **Nicht gebaut** | Phase 3 |

**Mock-Verhalten (Phasen 5 & 5.5):**

- **Kein Fiskaly:** Mock-SHA-256-Signatur, 500 ms Delay, `is_mocked_tse = true`, großer **TEST-BON — KEINE GÜLTIGE TSE**-Hinweis auf dem PDF.
- **Kein SumUp:** 3 s Terminal-Simulation, fake `sumup_checkout_id`.
- **Kein Stripe/GoCardless:** SEPA-Pfad bleibt nutzbar; Instant-Pay und Auto-Reconcile ohne Fehler deaktiviert.

**Kein Rewrite nötig** — nur Credentials und Redeploy.

---

## 4. Problem & Lösung

### 4.1 Schmerzpunkte im Studio

| Problem | Ohne k0ncept |
|---------|--------------|
| Anzahlungs-Jagd | Manuelles Matching in Banking-Apps |
| Papier-Waivers | Art.-9-Risiko, fehlende Nachweise |
| Kassen-Compliance | Keine TSE, kein revisionssicherer Beleg |
| Artist-Abrechnung | Excel, riskante Lohn-Wortwahl |
| Convention-WLAN | Walk-in bricht ab |
| Kanal-Chaos | E-Mail, DMs, Telefon — keine Queue |

### 4.2 Was k0ncept liefert

Strukturierte Pipeline, Postgres als Overlap-Autorität, entkoppelte Benachrichtigungen, Portal-Self-Service, Posteingang, Kasse + ERP — ein Produkt, eine Datenbank.

---

## 5. Technologie-Stack

| Schicht | Technologie |
|---------|-------------|
| Framework | Astro 6 SSR + React 19 Islands |
| Backend | Supabase Postgres, Storage, Realtime |
| Validierung | Zod |
| State Machine | XState v5 |
| PDF | pdf-lib (Rechnungen, Kassenbons, Gutschriften) |
| Bilder | Rust → WASM Referenz-Pipeline |
| Hosting | Vercel (+ Crons für Hobby-Plan) |
| Observability | Sentry (optional) |

---

## 6. Architektur-Überblick

```
┌─────────────────────────────────────────────────────────────────────────────┐
│  ÖFFENTLICH: Homepage · Buchung · Galerie · /termin/:token · /kiosk/waiver  │
└───────────────┬───────────────────────────────┬─────────────────────────────┘
                ▼                               ▼
         POST /api/appointments          Portal- + Kiosk-APIs
                ▼                               ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│  SUPABASE — Postgres · Storage · Realtime                                    │
│  appointments · studio_stations · pos_transactions · inventory_items           │
│  artist_settlements · email_jobs · payment_confirmation_jobs                 │
│  medical_waivers · settlement_invoices (private Buckets, deny-all public)    │
└───────┬─────────────────┬──────────────────────┬────────────────────────────┘
        ▼                 ▼                      ▼
   email_jobs         Storage              Realtime Inbox
        ▼
     Resend

┌─────────────────────────────────────────────────────────────────────────────┐
│  ADMIN: Inbox · Kalender · Kasse · Inventar · Abrechnungen · Posteingang    │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 7. Architecture & System Rationale (ASR)

| ID | Thema | Entscheidung | Begründung |
|----|-------|--------------|------------|
| **ASR-001** | Produktscope | Single-Studio Vertical OS | Tiefer Tattoo-Workflow > generischer Kalender |
| **ASR-002** | Frontend | Astro + React Islands | SSR + reiches Admin |
| **ASR-003** | Backend | Supabase | Postgres + Realtime + Storage |
| **ASR-004** | E-Mail outbound | Resend + Outbox | Kein Blockieren auf Resend |
| **ASR-005** | E-Mail inbound | Cloudflare → Worker | Getrennt von Resend |
| **ASR-006** | State Machine | XState v5 serverseitig | Ungültige Übergänge werden abgewiesen |
| **ASR-007** | Overlap | Postgres-Trigger (Artist **oder** Station) | Letzte Instanz bei Races |
| **ASR-008** | Anzahlungs-Holds | 7-Tage-Ablauf | Keine Geister-Blockaden |
| **ASR-009** | Zahlungen | Ein `confirmDeposit()` | Stripe, SEPA, Admin — ein Pfad |
| **ASR-010** | Portal-Tokens | HMAC-Hash, Rotation bei Confirm | `/termin` nicht erratbar |
| **ASR-011** | Branding | Live aus `studio_settings` | Kein Redeploy |
| **ASR-012** | Referenzen | WASM im Browser | Serverless-sicher |
| **ASR-013** | Rechnungen | pdf-lib + Helvetica | Kein Font-Fetch auf Vercel |
| **ASR-014** | DSGVO-Retention | Anonymisierung nach N Tagen | Analytics bleiben |
| **ASR-015** | Admin-Auth | DB-User + Rollen | iPad/Kasse ohne Shared Password |
| **ASR-016** | Posteingang | Eine Zeile, Studio-View = Filter | Keine Mail-Duplikate |
| **ASR-017** | E-Mail-Branding | Logo im Header | Sichtbar überall |
| **ASR-018** | Artist-Identität | Name vom Studio, Stimme selbst | Klare Rollen |
| **ASR-019** | Queues in Postgres | Kein Redis v1 | Ausreichend für Studio-Last |
| **ASR-020** | Local-first | Optionales Yjs | Convention-WLAN |
| **ASR-021** | Instagram (geplant) | Nur offizielle Meta-API | Kein Scraping |
| **ASR-022** | Referenz-Cleanup | `client_reference` löschen bei Complete | Ergebnis-Foto ersetzt Intake-Bilder; weniger DSGVO-Fläche |
| **ASR-023** | KassenSichV / TSE | Fiskaly; unveränderliche `pos_transactions` | GoBD/KassenSichV; Mock bis TSS-Credentials |
| **ASR-024** | Gutschrift vs. Lohn | Nur **Provisionsabrechnung / Gutschrift** | B2B-Freelancer; Vermeidung Scheinselbständigkeit — nie „Lohn“, „Gehalt“, „Payslip“ |
| **ASR-025** | Kiosk vs. Portal | Getrennte Token-Spalten | Waiver-Link darf `/termin` nicht invalidieren |
| **ASR-026** | Medizin-Storage | `medical_waivers` deny-all | Art. 9 Besondere Kategorien |
| **ASR-027** | Inventar-Burn | −1 Session Pack bei Complete | Verbrauch pro Session |

---

## 8. Paperless Studio & Kiosk (Phase 4)

| Feature | Detail |
|---------|--------|
| **Waiver-Kiosk** | `/kiosk/waiver/:token` — Tablet-Signatur; Token über `kiosk_token_hash` |
| **Kundenportal** | `/termin/:token` — Anzahlung, Verschieben, Storno via `portal_token_hash` |
| **Storage** | PDFs in privatem Bucket `medical_waivers` |
| **Hygiene** | Tinten- und Nadel-LOT auf Terminen |
| **Admin** | Waiver/Hygiene im Inbox-Detail; API „Kiosk-Link erzeugen“ |
| **DSGVO** | Retention-Cron löscht Waiver-PDFs vor Anonymisierung |

> Kiosk-Link erzeugen **rotiert nie** den Portal-Link des Kunden (Migration `0036`).

---

## 9. Kasse & KassenSichV (Phasen 5 & 5.5)

| Komponente | Ort |
|------------|-----|
| Checkout-UI | `/admin/pos` |
| API | `GET/POST /api/admin/pos` |
| Transaktionen | `pos_transactions` inkl. TSE-Felder, `is_mocked_tse` |
| Services | `fiskalyService.ts`, `sumupService.ts`, `receiptService.ts` |
| Settings | `fiskaly_*`, `sumup_merchant_code` auf `studio_settings` |

**Zahlungsarten:** Bar, SumUp-Karte, gemischt, Gutschein.

**Validierung:** Server lehnt Beträge ≤ 0 via Zod `.positive()` ab.

---

## 10. ERP & Provisionsabrechnungen (Phase 6)

| Komponente | Detail |
|------------|--------|
| **Inventar** | `inventory_items`; −1 Standard Session Pack bei Termin-Abschluss |
| **Warnung** | E-Mail-Job bei niedrigem Bestand |
| **Gast-Artists** | Ablaufdatum → täglicher Cron deaktiviert Login |
| **Abrechnungen** | `artist_settlements` — Umsatz, Materialabzug, Netto-Provision |
| **PDF** | Provisionsabrechnung / Gutschrift → Bucket `settlement_invoices` |
| **Inhaber** | `/admin/finances` — generieren, als bezahlt markieren, PDF |
| **Artist** | `/admin/profile` → Reiter **Meine Abrechnungen** |
| **Cron** | Monatlich am 1., 06:00 UTC |

**Formel:** `(POS-Umsatz − Materialabzug) × Provisionssatz %`. Material = abgeschlossene Sessions × Pack-Kosten.

**Rechtlich:** ausschließlich Provisions- und Gutschrift-Sprache — **keine** Lohnabrechnung.

---

## 11. Produkt-Oberflächen

| Oberfläche | Route |
|------------|-------|
| Inbox | `/admin` |
| Kalender | `/admin/calendar` |
| Walk-in | `/admin/book` |
| Posteingang | `/admin/messages` |
| Setup | `/admin/settings` |
| Benutzer | `/admin/users` |
| Kasse | `/admin/pos` |
| Inventar | `/admin/inventory` |
| Abrechnungen | `/admin/finances` |
| Kundenportal | `/termin/:token` |
| Waiver-Kiosk | `/kiosk/waiver/:token` |

**Rollen:** `admin` · `owner` · `artist` · `booking`.

**Ablauf:** Termin abschließen erfordert **Ergebnis-Foto**; danach werden Kunden-Referenzen gelöscht (ASR-022).

---

## 12. Zahlungen & Portal

| Pfad | Endet in |
|------|----------|
| Stripe-Webhook | `confirmDeposit()` |
| GoCardless-Cron | `confirmDeposit()` |
| Admin manuell | `confirmDeposit()` |
| SEPA | Kunde überweist; Studio bestätigt |

**Hinweis:** Anzahlungen sind **kein** Kassenvorgang — sie laufen über `appointments`, nicht über `pos_transactions`.

---

## 13. DSGVO & Compliance

| Funktion | Umsetzung |
|----------|-----------|
| Export | Admin-Tools API |
| Löschung | Manuelle Anonymisierung |
| Retention | Wöchentlicher Cron |
| Waiver-Shredding | Storage-Delete + DB-Felder nullen |

### Supabase-Backups (Art. 9)

Löschen aus Storage **entfernt Daten nicht** aus automatischen **Datenbank-Backups** von Supabase. Für vollständige Art.-9-Compliance müssen Betreiber **Backup-Aufbewahrung und Rotation** im AV-Vertrag und im Runbook regeln. Das ist eine **betriebliche** Pflicht, noch nicht im Code automatisiert.

---

## 14. Security Model

| Ebene | Maßnahme |
|-------|----------|
| Admin | Cookie-Sessions, scrypt |
| Tokens | HMAC, timing-safe |
| Storage | Waivers + Abrechnungs-PDFs: deny-all public; Zugriff nur service role + API mit Rollenprüfung |
| Cron | `CRON_SECRET` |
| Webhooks | Signatur / Shared Secret |

Artists sehen nur **eigene** Abrechnungs-PDFs.

---

## 15. Betrieb & Deployment

```bash
supabase db push   # Migrationen 0034–0038
# App auf Vercel redeployen
```

| Cron | Zeitplan | Aufgabe |
|------|----------|---------|
| Reminders | 07:00 UTC | Erinnerungen (Fenster 06–24 Europe/Berlin) |
| daily-maintenance | 07:15 UTC | E-Mail, SEPA, **Gast-Artist-Expiry** |
| gdpr-retention | Mo 04:00 UTC | Anonymisierung |
| monthly-settlements | 1. des Monats 06:00 UTC | Provisionsabrechnungen |

---

## 16. Bekannte Lücken & Audit-Hinweise

Review aus Principal-Architect-Perspektive (Juni 2026).

### 16.1 Zeitzonen (Europe/Berlin vs. UTC)

| Bereich | Verhalten | Risiko |
|---------|-----------|--------|
| Erinnerungen | 06–24 **Europe/Berlin** | ✅ |
| Monats-Abrechnungen | UTC-Monatsgrenzen | ⚠️ Grenzfälle am Monatsende |
| Kassenbuch „heute“ | UTC-Tag | ⚠️ kann vom Berlin-Kalendertag abweichen |
| Session-Zählung Material | `end_time` in UTC-Range | ⚠️ Randfälle |

**Empfehlung:** zentrale Hilfsfunktion `europeBerlinDayBounds()` für Kasse, Settlements und Crons.

### 16.2 Race Conditions bei Zahlungen

| Szenario | Schutz | Lücke |
|----------|--------|-------|
| Stripe-Webhook vs. Admin-Klick | Status-Guard, Job-Dedup, PI-Idempotenz | Kein `SELECT FOR UPDATE`; enge Race → evtl. doppelter Job |
| SumUp-Webhook vs. manuell | **N/A** | SumUp-Webhook **nicht implementiert** — Mock synchron im Checkout |

### 16.3 DSGVO-Backups

Siehe §13 — Backup-Rotation dokumentieren.

### 16.4 Kassen-Validierung

Server blockiert 0 € und negative Beträge ✅. UI sollte doppelt absichern (Button disable bei Submit).

### 16.5 Abrechnungen vs. Anzahlungen

**Regel heute:** Umsatz in der Provisionsabrechnung = Summe **`pos_transactions`** (completed) im Zeitraum.

| Ereignis | In Abrechnung? |
|----------|----------------|
| Kassen-Checkout am Termin | ✅ |
| Anzahlung (Stripe/SEPA/Admin) | ❌ **Nein** |
| Anzahlung Vormonat, Tattoo diesen Monat | ❌ Nur POS im aktuellen Monat |

**Konsequenz:** Wenn Provision auf **Gesamt-Session-Wert** basieren soll, muss der Restbetrag über die Kasse gebucht werden — oder die Settlement-Logik muss erweitert werden. **Produkt-Lücke, dokumentiert.**

Materialabzug zählt **abgeschlossene Termine** (nach `end_time`), unabhängig vom POS — kann von POS-Umsatz abweichen.

### 16.6 Weitere Punkte

- Inventar-Burn nicht in derselben DB-Transaktion wie Status-Wechsel.
- Artists ohne `studio_artist_id`-Verknüpfung: POS-Zuordnung kann fehlen.
- Live-Schaltung Fiskaly/SumUp/Stripe = nur Keys.

---

<p align="center">
  <sub>
    <a href="pitch_readme.md">English</a> ·
    <a href="progress_readme.md">Build-Chronologie</a> ·
  </sub>
</p>
