# k0ncept

> A complete, legally compliant operating system for tattoo studios. From live booking and offline-first admin to POS integration and automated artist settlements.

![Astro](https://img.shields.io/badge/Astro-6-FF5D01?style=flat-square&logo=astro&logoColor=white)
![React](https://img.shields.io/badge/React-19-61DAFB?style=flat-square&logo=react&logoColor=black)
![Supabase](https://img.shields.io/badge/Supabase-DB%20%7C%20Auth%20%7C%20Storage-3ECF8E?style=flat-square&logo=supabase&logoColor=white)
![TypeScript](https://img.shields.io/badge/TypeScript-Strict-3178C6?style=flat-square&logo=typescript&logoColor=white)
![Rust](https://img.shields.io/badge/Rust-WASM-000000?style=flat-square&logo=rust&logoColor=white)

**k0ncept** replaces phone-only intake, spreadsheet calendars, WhatsApp deposit chasing, paper waivers, and manual commission spreadsheets with one integrated product. It is built to run a real business, featuring robust payment reconciliation, offline resilience for conventions, and strict German tax/GDPR compliance.

---

## 🚀 Core Capabilities

* **The Booking Engine**: Real-time availability with strict Postgres-level overlap prevention (checking both artist and physical station availability before saving).
* **Compliant POS Checkout**: Fully integrated mock/sandbox for SumUp card terminals and **Fiskaly TSE** for German KassenSichV compliance (immutable receipts and cash journal).
* **Offline-First Resilience**: Optional Yjs-powered local sync ensures walk-in bookings and calendar views work flawlessly even on terrible convention Wi-Fi.
* **Paperless Studio (Art. 9 GDPR)**: Dedicated iPad Kiosk mode for digital medical waivers, stored in private buckets, with automated shredding.
* **WASM Media Pipeline**: Client-side image processing (Rust) resizes and optimizes heavy reference photos before they hit the server.
* **Automated ERP**: Inventory auto-burn per completed session and automated monthly commission statements (Provisionsabrechnungen) generated as PDFs via `pdf-lib`.
* **Two-Lane Communication**: Resend for queued, retryable outbound emails and Cloudflare Workers for an integrated inbound inbox (Posteingang).

---

## 🛠️ Tech Stack

| Layer | Technology |
|---|---|
| **Frontend** | Astro 6 (SSR), React 19 (Islands), Tailwind CSS |
| **Backend** | Supabase (Postgres, Storage, Realtime) |
| **State Management** | XState v5 (Server-side appointment state machine) |
| **Media Processing** | Rust (compiled to WebAssembly) |
| **Infrastructure** | Vercel (Hosting + bundled crons on Hobby plan) |
| **Integrations** | Stripe, GoCardless, Resend, Cloudflare Email Routing |

---

## 🏗️ Architectural Principle

Critical actions (bookings, payments, POS transactions) are written to Postgres **before** any third-party API call. Side effects like email delivery, payment confirmation, and inventory updates run through retryable Postgres queues or non-blocking hooks to ensure the core application never blocks or loses data due to external outages.

---

## 📚 Documentation Deep Dive

This repository contains extensive architectural documentation. To see how the system is designed under the hood, explore the following:

* **[Architecture & Product Reference (Pitch)](pitch_readme.md)**: The complete ASR (Architecture & System Rationale), security model, and integration status.
* **[Build Chronology & Audit (Progress)](progress_readme.md)**: How the platform evolved, production incident logs, and current audits.
* **[Database Migrations](supabase/migrations/README.md)**: Schema evolution, state machine tables, and Postgres trigger logic.

---
