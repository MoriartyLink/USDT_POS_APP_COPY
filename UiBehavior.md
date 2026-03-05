# UI Behavior Specification

## Overview

This document specifies the UI behavior, screen responsibilities, navigation, and in-memory password handling for the Merchant Invoice Management UI (USDT Merchant Engine).

Global requirement: **Engine password is entered once per app instance**. The password is stored only in-memory for the lifetime of the Electron renderer process and is never persisted to disk. As long as the app window remains open and the engine remains unlocked, all screens operate without re-prompting for the password. When the app is closed or the engine is explicitly locked, the user must re-enter the password at the next launch.

---

## Screen List

1. Engine Unlock
2. Main Shell (Layout)
3. Dashboard
4. POS System
5. Settings
6. Payment Page (public POS window)

---

## Navigation Map

- App startup → Engine Unlock (if Electron engine with configured wallet)
  - If unlock succeeds or no engine wallet yet → Main Shell
  - If engine not present (pure web) → Main Shell directly
- Main Shell
  - Top nav → Dashboard (`/dashboard`)
  - Top nav → POS (`/pos`)
  - Top nav → Settings (`/settings`)
- Dashboard
  - "Open POS Window" → opens new browser window at `/pay` (Payment Page, amount entry mode)
  - "View" on invoice → opens new browser window at `/pay/:invoiceId` (Payment Page, invoice view mode)
- POS System
  - Local flow only inside `/pos` and invoice QR view there
- Settings
  - Local configuration only inside `/settings`
- Payment Page
  - Direct entry via `/pay` (from POS window or external link)
  - Direct entry via `/pay/:invoiceId` (from dashboard or external link)

Password / engine state impact on navigation:
- If engine wallet is configured but **not unlocked** in this process:
  - At app start: Engine Unlock is shown.
  - POS System and Payment Page will surface engine-locked errors when trying to create invoices and will instruct the operator to unlock via Settings in the main console; they never prompt again for the password themselves.
- If engine is **unconfigured**: Engine Unlock will auto-bypass to Main Shell and Settings shows configuration workflow.

---

## 1. Engine Unlock Screen

**Route / Context**
- Not a URL route; rendered before the router inside `App` when an Electron `window.engine` is present and a wallet already exists.

**User Story**
- As a merchant operator, I want to unlock the engine wallet once when I start the console so that all subsequent operations (creating invoices, prefunding, on-chain checks) can run without repeatedly entering the password.

**Attributes / UI Elements**
- Password field: `engine password`
- Button: `Unlock Engine`
- Error banner (non-persistent, per-attempt)
- Informational text: explains that unlocking is per-session and that mnemonic is managed by the engine.

**Behavior & Logic**
- On mount:
  - If `window.engine` is not available → mark `enginePresent = false`, immediately call `onUnlocked()` (bypass unlock; pure SPA mode).
  - If `window.engine` exists:
    - Call `hasWallet()`:
      - If `false` → no wallet configured yet; call `onUnlocked()` (go to Settings to configure).
      - If `true` → show unlock form.
- On submit:
  - Require non-empty password.
  - Call `window.engine.unlockWallet(password)`.
  - On success:
    - Engine decrypts mnemonic and keeps it in its own secure in-memory store.
    - UI stores `engineWalletConfigured = true` and the `masterAddress` in `localStorage` for display.
    - Clear password field.
    - Call `onUnlocked()` to let the router render the Main Shell.
  - On failure:
    - Show inline error: "Failed to unlock engine wallet. Please check your password."
    - Do **not** persist password anywhere.

**Password Lifetime**
- Password value lives only in the component state during the unlock attempt and is cleared on success.
- Engine internally remains unlocked until:
  - App process is closed, or
  - Engine decides to lock (future enhancement).

---

## 2. Main Shell (Layout)

**Route**
- Wraps all authenticated console routes under `/*` (`/dashboard`, `/pos`, `/settings`).

**User Story**
- As a merchant operator, I want a consistent shell with navigation so that I can quickly switch between my dashboard, POS, and settings.

**Attributes / UI Elements**
- App title: "USDT Merchant Engine".
- Top navigation items:
  - Dashboard → `/dashboard`
  - POS → `/pos`
  - Settings → `/settings`
- Active link highlighting based on current path.

**Behavior & Logic**
- Navigation uses standard anchor tags (`href`) that integrate with React Router.
- Active item styling: determined by `location.pathname === item.href`.
- Children route content is rendered inside the main layout container.
- No direct password handling here; assumes Engine Unlock has already run.

---

## 3. Dashboard Screen

**Route**
- `/dashboard`

**User Stories**
- As a merchant operator, I want to see a high-level overview of my invoices and key stats (revenue, pending payments, success rate) so that I can understand the health of my USDT rail.
- As a merchant operator, I want to open a POS window directly from the dashboard so that my front-of-house staff can create invoices without accessing the full console.

**Attributes / UI Elements**
- Header with title and description.
- Buttons:
  - `Refresh` invoices.
  - `Open POS Window` → opens `/pay` in a new window.
- Stats cards:
  - Total Confirmed USDT.
  - Total Invoices.
  - Pending Payments.
  - Success Rate (confirmed vs expired).
  - Sweepable USDT (mocked for now, based on confirmed/detected invoices).
- Recent Invoices table:
  - Columns: Invoice ID, Amount, Customer/Memo, Deposit Address, On-chain Amount, Status, Manual Status, Created/Expires, Actions.
  - Actions per invoice:
    - `View` → opens `/pay/:invoiceId` in new window.
    - Manual status dropdown (all statuses) which updates the stored invoice status.

**Behavior & Logic**
- On mount:
  - If `window.engine` exists, load invoices via `engine.listInvoices()`; otherwise fall back to `loadInvoices()` from local storage.
  - Normalize statuses: any `pending`/`detected` invoice whose `expiresAt` is in the past is updated to `expired` via the appropriate engine/local update helper.
  - Sort invoices by `createdAt` descending and take up to the 50 most recent.
  - For all invoices with a `depositAddress`, query on-chain balances via `blockchainService.getBalancesForAddresses` on the configured USDT rail and attach the latest USDT and gas status for display in the On-chain Amount column.
- Status color mapping: visual chips per status (pending, detected, confirmed, expired, late_payment, underpaid, overpaid, swept).
- `Refresh` button:
  - Re-runs the load/normalize/sort pipeline.
- `Open POS Window`:
  - Uses `window.open(origin + '/pay', '_blank')` to start a standalone Payment Page session (for FOH use) which relies on engine/app_config.
- `View` invoice:
  - Uses `window.open(origin + '/pay/:invoiceId', '_blank')`.
- Manual status dropdown:
  - Changing the selection immediately patches the in-memory list and calls `updateInvoice(id, patch)` to persist.
  - For `confirmed`, also set `confirmedAt = now`.

**Password / Engine Interaction**
- Dashboard calls into the engine for invoice listing and updates when running under Electron, and falls back to local invoice storage only in SPA/dev mode.
- It assumes engine state (wallet unlocked, config set) is already satisfied by Engine Unlock and Settings.

---

## 4. POS System Screen

**Route**
- `/pos`

**User Stories**
- As point-of-sale staff, I want to generate per-invoice QR codes and payment links for specific customers and amounts so that they can pay USDT directly to the merchant-controlled invoice address.
- As point-of-sale staff, I want the POS to enforce that the engine is configured (wallet + app_config) and unlocked so that every generated invoice is valid and traceable.

**Attributes / UI Elements**
- Initial form state:
  - Invoice Amount (USDT, numeric input).
  - Customer Identifier (optional text, e.g., table/order id).
  - Memo / Note (optional text).
  - Button: `Generate Payment QR`.
- Generated invoice state:
  - Status indicator (Pending / Confirmed; countdown timer until expiry).
  - Amount and optional customer/memo.
  - QR Code (invoice deposit address encoded).
  - Deposit Address (text with `Copy Address` button).
  - Payment Instructions panel.
  - Actions:
    - `Copy Link` (payment URL).
    - `Share` (uses `navigator.share` or falls back to copy).
    - `Create New Payment` (resets to form view).
- Engine configuration alerts:
  - Blocking banner when engine/app_config is not ready with guidance to use Settings.

**Behavior & Logic**
- On mount:
  - If `window.engine` exists (Electron mode):
    - Call `hasWallet()`.
      - If `false` → show `engineReady = false` with message directing operator to Settings to configure wallet.
    - Call `getAppConfig()`:
      - If missing/invalid → `engineReady = false` with message to configure app_config in Settings.
      - If present → derive `invoiceExpirySeconds` from engine config and set `engineReady = true`.
  - If no `window.engine` (SPA/dev mode):
    - Read `engineAppConfig` from `localStorage`.
    - If missing/invalid → `engineReady = false` with message.
    - Else → set `invoiceExpirySeconds`.
- When `engineReady = false`:
  - Render a warning card only, explaining what must be done in Settings. No invoice form.
- On `Generate Payment QR`:
  - Validate amount > 0.
  - Determine expiry in seconds from `invoiceExpirySeconds`.
  - Behavior priority:
    1. **Electron engine present**:
       - Call `window.engine.createInvoice({ amount, currency: 'USDT', memo, customerIdentifier, invoiceExpirySeconds })`.
       - On success: extract `id`, `address`, `createdAt`, `expiresAt` from engine response; treat engine as the source of truth for HD index allocation.
       - On error:
         - If error message contains "Engine wallet is not unlocked in this session" → show `engineMessage` instructing operator to unlock engine in Settings (no password prompt here) and set `engineReady = false`.
         - If error message contains "Engine wallet is not configured" → show message to configure wallet.
         - For any other error → show generic engine failure message; do not fall back to local HD helpers.
    2. **No engine (dev fallback)**:
       - Call `allocateNextInvoiceAddress()` using mnemonic in localStorage.
  - Use the final `depositAddress` to:
    - Generate QR code image via `QRCode.toDataURL(depositAddress, ...)`.
    - Compute `expiresAt` as `now + expirySeconds`.
    - Build a local POS invoice object for display.
    - Build an `EngineInvoice` record and `upsertInvoice` into invoice store.
- Timer / status in POS invoice view:
  - Local countdown `formatTimeRemaining(expiresAt)` is displayed.
  - Status is `pending` in POS view; external confirmation is handled via dashboard/payment page.

**Password / Engine Interaction**
- POS never collects the password directly.
- If engine is locked:
  - `engine.createInvoice` fails with lock-related message.
  - POS displays guidance to unlock engine in the main console Settings.
  - Operator returns to main console (where Engine Unlock was run earlier or will be rerun) to unlock; once unlocked, POS can successfully call `createInvoice` without new password input.

---

## 5. Settings Screen

**Route**
- `/settings`

**User Stories**
- As an admin, I want to configure the engine wallet mnemonic and password so that the engine can derive deposit addresses and manage funds non-custodially.
- As an admin, I want to configure on-chain parameters (RPC URL, USDT contract, confirmation threshold, invoice expiry, detection mode, prefunding) so that the rail behaves as expected on the chosen network.
- As an admin, I want to unlock the engine wallet for a session when necessary, re-using the same password I configured earlier.

**Attributes / UI Elements (Wallet tab)**
- Wallet-related fields:
  - Wallet Seed (mnemonic) text area (visible only for SPA/dev configuration; in engine mode, mnemonic is managed by backend).
  - Engine password & confirmation (used to encrypt mnemonic in engine, or for later unlock calls).
  - Master Engine Address (read-only, derived from HD index 0 if wallet is configured).
  - Status texts (wallet configured, success/error messages).
- Network / app_config fields:
  - Network (fixed: Polygon EVM, informational).
  - Environment (testnet/mainnet label).
  - Invoice Expiry (seconds).
  - Confirmation Threshold.
  - RPC Provider URL.
  - USDT Contract Address.
  - Detection Mode (polling / watch-only).
  - Prefund: count, gas amount, and execute button.
  - Master MATIC balance with refresh.
  - Database Debug section for SQLite (developer only).

**Behavior & Logic: Wallet Configuration / Unlock**
- On mount in Electron mode:
  - `engine.hasWallet()` sets `engineHasWallet` and `walletConfigured`.
  - `engine.getAppConfig()` pre-fills UI fields with current persisted config.
- Wallet setup form submit:
  - Validation:
    - If creating a wallet (`!engineHasWallet`) or non-Electron dev mode → mnemonic required, at least 12 words.
    - Password required and must match confirmation.
  - Electron engine path:
    - If `!engineHasWallet`: call `engine.configureWallet(mnemonic, password)` → engine persists encrypted mnemonic and returns master address.
    - Else: call `engine.unlockWallet(password)` to unlock existing wallet for this session.
    - In both cases, store `engineWalletConfigured = true` and `engineMasterAddress` in localStorage and mark UI as verified.
  - SPA-only path:
    - Derive `masterAddress` via `deriveMasterAddressFromMnemonic(mnemonic)`.
    - Persist mnemonic and derived values in localStorage (`engineMnemonic`, `engineMasterAddress`, `engineNextInvoiceIndex` etc.).
- app_config submit (`handleAddressSubmit`):
  - Build config: `networkEnv`, `rpcProviderUrl`, `confirmationThreshold`, `invoiceExpirySeconds`, `detectionMode`, `prefundCount`, `prefundGasAmount`, `usdtContractAddress`.
  - Persist to localStorage `engineAppConfig`.
  - If Electron engine present, call `engine.setAppConfig` with the same values.
  - Mark `usdtAddress.isVerified = true` using stored master address.

**Behavior & Logic: Prefund**
- Validate wallet configured and RPC URL present.
- Electron engine mode:
  - Call `engine.prefund({ count, amountPerAddress, rpcProviderUrl, password })`.
  - Note: uses the same password; UI should encourage that the engine is already unlocked, but currently still passes password to engine.
  - Show textual summary of prefund transactions.
  - Refresh master balance.
- SPA-only mode:
  - Use `mnemonic` and `deriveInvoiceAddressFromMnemonic` across a range of HD indices starting from `engineNextInvoiceIndex` / `engineNextPrefundIndex`.
  - Call `runPrefund` to send small gas amounts to these addresses.
  - Update `engineNextPrefundIndex` after success.

**Password / Engine Interaction**
- Settings is the **primary** UI where the wallet password is first configured and can be re-used to unlock or prefund.
- Password is not stored in localStorage; only the engineMasterAddress and flags are persisted.
- Once unlocked, subsequent operations (createInvoice, prefund, etc.) should succeed without prompting again, as long as the engine process stays alive.

**Behavior & Logic: Database Debug (SQLite)**
- Visible only in the Settings wallet tab and intended for development/debug.
- Requires running inside the Electron engine shell; in pure SPA mode it shows an informational message and does not attempt DB access.
- On load (Electron engine present):
  - Calls `engine.debugListTables()` to enumerate all non-internal SQLite tables.
  - Automatically loads the first table, if any, via `engine.debugGetTable(table)` which returns column names, primary key, and up to a capped number of rows.
- Table selector:
  - Lets the operator choose any available table; changing selection reloads data via `debugGetTable`.
- Grid view:
  - Renders one editable row per record, with a text input per column.
  - Shows which column is the primary key (PK) when available.
- CRUD actions:
  - `Add Row` appends a new blank row client-side which can then be edited and saved.
  - `Save` on a row calls `engine.debugSaveRow({ table, primaryKey, originalPkValue, row })`:
    - If `originalPkValue` is present, issues an `UPDATE` by PK.
    - If `originalPkValue` is null/undefined, issues an `INSERT`.
    - After the operation, the table is reloaded and original/edited rows are refreshed from the engine.
  - `Delete` on a row calls `engine.debugDeleteRow({ table, primaryKey, pkValue })` when a PK and value are available, otherwise removes only the local unsaved row.
- Errors from any debug call are surfaced inline in a small error banner; the tool is explicitly documented as unsafe for production data.

---

## 6. Payment Page (Public POS Window)

**Routes**
- `/pay` → embedded invoice-creation UX for public POS window (amount/memo/customer entry, then QR view).
- `/pay/:invoiceId` → read/view an existing invoice by ID (usually opened from Dashboard or external links).

**User Stories**
- As a cashier, I want to open a dedicated POS payment window where I can enter an amount and show a QR code to the customer, without exposing the full console.
- As a customer, I want to open a payment link (e.g., from a QR code or link sent to me) and see the payment amount, address, and payment status in real time.

**Attributes / UI Elements**
- Pre-payment (no `payment` object yet, `/pay` route only):
  - Amount input (USDT).
  - Optional memo and customer ID.
  - Start payment button.
- Active invoice view:
  - Status (Pending / Detected / Confirmed / Expired) with timer until expiry.
  - Amount, merchant name, memo, customer identifier.
  - QR code (currently placeholder image; the actual encoded value is the deposit address; may be enhanced later).
  - Deposit address with copy.
  - Engine / chain messages (info about detection/confirmation or errors).
  - `Check Blockchain Status` action (calls RPC to confirm payment on-chain against the deposit address).

**Behavior & Logic: Engine / Config Bootstrapping**
- On mount:
  - If `window.engine` exists:
    - `hasWallet()` must be true; otherwise show engine misconfiguration message.
    - `getAppConfig()` must return config; else show message instructing to configure in Settings.
    - From config and fallback defaults, compute:
      - `invoiceExpirySeconds`.
      - `confirmationThreshold`.
      - `rpcProviderUrl`.
      - `usdtContractAddress`.
    - Set `engineReady = true`.
  - If no `window.engine`:
    - Read `engineAppConfig` from localStorage; derive the same values or fall back to polygon testnet defaults.
- If config is missing/invalid:
  - `engineReady = false` and a blocking warning is displayed, instructing operator to use Settings in the main console.

**Behavior & Logic: Starting a Payment (`/pay`)**
- Validate amount > 0.
- Require that `engineConfig` is present; if not, show message and set `engineReady = false`.
- Determine:
  - `expirySeconds` from config.
  - Local `id` from URL param (`invoiceId`) if present, else `inv-${timestamp}`.
- Engine priority:
  1. **Electron mode**:
     - Call `engine.createInvoice({ amount, currency: 'USDT', memo, customerIdentifier, invoiceExpirySeconds })`.
     - On success: trust engine’s `id`, `address`, `createdAt`, and `expiresAt`.
     - On error with lock/config messages:
       - Show message instructing operator to unlock or configure engine in Settings.
       - Do not fall back to local HD helpers; set `engineReady = false`.
  2. **No engine**:
     - Use `allocateNextInvoiceAddress()` to get a local dev deposit address.
- Construct local `PaymentDetails` with status `pending` and a basic QR placeholder (image data for now).
- Call `upsertInvoice` with an `Invoice` record containing id, amount, currency, address, status, qrCode, paymentLink, expiresAt, createdAt, and any memo/customerIdentifier.
- If route was `/pay` (no `invoiceId` param) → `navigate(/pay/:id)` to canonical invoice URL.

**Behavior & Logic: Direct Invoice View (`/pay/:invoiceId`)**
- If `payment` is not set yet but `invoiceId` is present:
  - Try `getInvoice(invoiceId)` from local invoice store.
  - If found:
    - Map stored invoice fields into `PaymentDetails`.
    - Status mapping:
      - `confirmed` → confirmed.
      - `expired` → expired.
      - `detected` → detected.
      - Other statuses → pending.
  - If not found:
    - Build a minimal pending invoice using `invoiceId`, placeholder QR, and a fallback `expiresAt` (15 minutes from now).
    - Attempt to allocate a deposit address via `allocateNextInvoiceAddress()` (best-effort in dev mode).

**Behavior & Logic: Timer / Expiry**
- While status is not `confirmed` or `expired`:
  - 1-second interval updates `timeRemaining` and, when `expiresAt` is in the past, sets invoice to `expired` and calls `updateInvoice(id, { status: 'expired' })`.

**Behavior & Logic: Chain Status Check & Auto Detection**
- While the invoice is active (not `confirmed`/`expired`), the Payment Page periodically polls the chain:
  - A `useEffect` interval calls the same detection logic every ~15 seconds using `getBlockNumber` and `getErc20Transfers`.
  - This keeps the status in sync even if the operator does not click the button.
- On explicit `Check Blockchain Status`:
  - Require `engineConfig` and `payment.depositAddress`.
  - Call `getBlockNumber` and `getErc20Transfers` with the configured RPC URL and USDT contract.
  - If no transfers → show message: no transfers detected yet.
  - For latest transfer:
    - Compute confirmations: `currentBlock - tx.blockNumber`.
    - If confirmations ≥ `confirmationThreshold`:
      - Set status to `confirmed`.
      - Update invoice with `status: 'confirmed'`, `txHash`, `confirmations`, `confirmedAt`.
    - Else:
      - Set status to `detected`.
      - Update invoice with `status: 'detected'`, `txHash`, `confirmations`, `detectedAt`.

**Password / Engine Interaction**
- Payment Page never asks for or stores the password.
- It relies on engine already being unlocked (via Engine Unlock / Settings) when `createInvoice` is invoked.
- If engine is locked, the error message is propagated and rendered; the operator must go back to the console and unlock once.

---

## Summary: Single Login / Password Handling

- The **only** UI surfaces that directly accept the engine password are:
  - Engine Unlock screen (at startup, per-session unlock), and
  - Settings (for initial configuration and unlock/prefund helper flows).
- Once the password has been successfully used to unlock the engine in this app instance:
  - The Electron main process keeps the wallet decrypted in its own memory.
  - All other screens (Dashboard, POS System, Payment Page) operate without any further password prompts.
- Passwords are never stored in localStorage or persisted to disk by the UI; only non-sensitive metadata such as `engineWalletConfigured` flags and `engineMasterAddress` are persisted.
- When the app is closed (renderer + main processes terminate), the in-memory password is lost, and at the next startup Engine Unlock will require re-entry of the password before sensitive operations can proceed.
