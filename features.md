# Merchant Engine UI – Features

Status legend:
- `DONE` – implemented in current UI (mocked or real as noted)
- `PARTIAL` – skeleton or mock exists, needs wiring or more fields
- `TODO` – not yet implemented in UI

---

## 1. POS / Invoice Creation

- [DONE] Single-invoice POS flow at `/pos` (console) and `/pay` (POS window)
- [DONE] Fields: `amount`, `customer_identifier`, `memo`
- [DONE] One active invoice at a time in POS window (input ↔ invoice)
- [DONE] Manual on-chain payment instructions using a configurable EVM deposit address (from Settings)
- [DONE] Respect engine config `invoice_expiry_seconds` instead of hard-coded 15 min
- [DONE] Use per-invoice HD-derived deposit address from the configured mnemonic (Electron engine, with SPA fallback)
- [DONE] Persist created invoices to engine (SQLite `invoices` table via Electron IPC, with localStorage fallback in pure SPA mode)

## 2. Invoice Lifecycle & Status

- [DONE] Frontend model aligned with DB statuses: `pending`, `detected`, `confirmed`, `expired`, `late_payment`, `underpaid`, `overpaid`, `swept`
- [DONE] POS and PaymentPage use `pending` → `detected` → `confirmed` → `expired` states with RPC-based detection against the configured deposit address
- [DONE] Dashboard shows recent invoices with richer fields (customer, memo, created/expiry)
- [DONE] Store detection / confirmation metadata in the local invoice store (tx_hash, confirmations, detected_at, confirmed_at)
- [TODO] Surface sweep info (swept_at, sweep reports) in UI
- [PARTIAL] Real-time or background-polled status updates: auto-polling on POS and PaymentPage, but no background engine loop yet

## 3. Payment Page (Public POS Window)

- [DONE] `/pay` – POS input view
- [DONE] `/pay/:invoiceId` – invoice detail view with status, QR placeholder, and manual details
- [DONE] URL-driven flow: Start Payment → navigate to `/pay/inv-xxxx`, Create New Payment → back to `/pay`
- [DONE] "Check Blockchain Status (RPC)" button plus auto-polling that query the configured RPC provider for USDT `Transfer` events to the deposit address and update invoice status (`pending` → `detected` / `confirmed`)
- [DONE] Load invoice by id from the engine when running under Electron, falling back to a shared local engine store (`engineInvoices` in `localStorage`) in SPA mode
- [DONE] Swap local engine store for real SQLite-backed engine access (Electron main process / Node layer)
- [TODO] Render actual QR from engine-provided payment URI

## 4. Dashboard

- [DONE] Engine-focused dashboard header and copy
- [PARTIAL] Summary stats (totals/pending/success rate computed in UI from real engine invoices; sweepable USDT still simplified)
- [DONE] Recent Invoices table showing: id, amount, customer/memo, index, deposit address, on-chain amount, status, created/relative expiry
- [DONE] Mock/partial sweep card (Sweepable USDT + disabled action)
- [DONE] Status pill colors for all invoice statuses
- [DONE] Link rows to `/pay/:invoiceId` in a separate POS window
- [PARTIAL] Show real aggregates sourced from engine DB (invoices are real; sweep_reports/prefund_batches still future)

## 5. Settings / app_config

- [DONE] Engine wallet configuration (network label + USDT address, Electron engine-backed)
- [DONE] UI fields corresponding to `app_config`:
  - `rpc_provider_url` → RPC Provider URL
  - `detection_mode` → Detection Mode (polling / watch-only)
  - `confirmation_threshold`
  - `invoice_expiry_seconds`
  - `prefund_count`
  - `prefund_gas_amount`
- [DONE] Persist app_config-like values to a local engine config (`engineAppConfig` in `localStorage`) and to the SQLite `app_config` table via Electron so POS and PaymentPage can read them
- [DONE] Use configured values in POS (expiry) and detection logic (confirmation threshold, RPC URL, USDT contract). Deposit address is derived per-invoice via HD from the configured mnemonic.
- [DONE] Persist app_config to real SQLite engine store and access via Electron/Node instead of `localStorage`

## 6. HD Wallet & Address Management

(Backed by merchant-core-ui + engine datamodel: `wallet_security`, `hd_state`, `addresses`, `address_gas_state`)

- [DONE] Local TS helpers in `src/utils` (HD derivation, prefund, RPC) back the engine behavior in Electron and the SPA fallback
- [DONE] UI representation of sweepable funds on dashboard (on-chain amounts + gas status per deposit address, sweep action still disabled)
- [TODO] Surface HD indices and address roles (master vs invoice) in UI when needed
- [TODO] Show prefunded/pending gas status from `address_gas_state`

## 7. Sweeping

- [DONE] Mock "Sweepable USDT" card on dashboard
- [PARTIAL] Real sweep UX exists in MerchantDapp (not yet wired into this console)
- [TODO] Engine-integrated sweep UI in this app:
  - List sweepable invoice addresses + balances
  - Trigger sweeps via engine / wallet
  - Show `swept` status and `sweep_reports` entries

## 8. Prefunding

- [DONE] Config fields for prefund count + gas amount in Settings
- [PARTIAL] UI to trigger prefund transactions from the master account to invoice deposit addresses (no SQLite-backed `prefund_batches` yet)
- [TODO] Show last prefunded index vs current_index (from `hd_state`)

## 9. Security / Wallet Setup

- [PARTIAL] UI for initial wallet_security setup (mnemonic + password form in Settings, SPA keeps mnemonic in localStorage for HD derivation; real encrypted `wallet_security` table still TODO)
- [TODO] Backup / recovery guidance surface

## 10. Logs & Diagnostics

- [TODO] Simple engine log/diagnostics view (high-level events from detection/sweep/prefund)

## 11. Auth / Multi-tenant (Out of Scope for Engine Console)

- [DONE] Removed landing page, login, KYC, profile, and payment methods tabs from this app
- [NOTE] Merchant identification, KYC, SaaS billing, and API key management live in outer platform, not in this single-merchant engine UI.
