# Non-Custodial Merchant Engine v0.1
## Requirements Specification (EARS Format)

These requirements describe the **desktop engine** (Electron main process + SQLite database + mnemonic vault)
that will back the Merchant Engine UI. The current React/Vite SPA in this repository acts as a **development
demo harness**:

- It simulates the engine tables with `localStorage` (see `datamodel.md`).
- It uses dev-only helpers (e.g. `engineMnemonic`, `engineNextInvoiceIndex`, manual RPC polling) so the UI can
	exercise flows before the real engine exists.

Where the SPA temporarily violates or omits a requirement (for example, storing a mnemonic in `localStorage` or
allowing more than one pending invoice), that behavior is considered **out of scope for this specification** and
must be removed or replaced when wiring the UI to the real engine.

---

# 1. Ubiquitous Requirements

### UR-1 — Non-Custodial Guarantee
The system SHALL never transmit, log, or externally store the merchant’s mnemonic or private keys.

_Note: The v0.1 SPA demo keeps a plaintext mnemonic in browser `localStorage` under a dev-only key so it can
derive HD addresses. This is a temporary violation for development only and is **not permitted** in the real
engine implementation._

### UR-2 — Deterministic HD Derivation
The system SHALL derive addresses using the path `m/44'/60'/0'/0/index`.

### UR-3 — Master Account Rule
The system SHALL treat HD index `0` as the master consolidation account.

### UR-4 — Monotonic Index Allocation
The system SHALL allocate invoice addresses using a strictly increasing index counter that SHALL never decrease or reuse a previously allocated index.

### UR-5 — Atomic Allocation
The system SHALL allocate a new HD index and insert the corresponding invoice record within a single atomic SQLite transaction.

### UR-6 — Single Active Invoice
The system SHALL allow only one invoice in state `pending`, `detected`, or `confirmed` at any time.

### UR-7 — Stateless Core
The domain engine SHALL not retain in-memory global state between operations.

### UR-8 — Explicit Operations
The system SHALL not execute blockchain transactions without explicit merchant action.

### UR-9 — Full Operational Logging
The system SHALL record all blockchain interactions and invoice transitions in SQLite tables.

---

# 2. Wallet & Security Requirements

### ER-W1 — Mnemonic Encryption
WHEN a merchant configures the wallet  
THEN the system SHALL encrypt the mnemonic using AES-256-GCM with a password-derived key.

### ER-W2 — Vault Storage
WHEN encryption completes  
THEN the system SHALL store only encrypted mnemonic, salt, IV, and KDF parameters.

### ER-W3 — Decryption
WHEN merchant provides password  
THEN the system SHALL decrypt mnemonic in memory only.

### SR-W4 — Password Failure
WHILE password is invalid  
THE system SHALL deny wallet access.

### CR-W5 — No Plain Storage
The system SHALL never store mnemonic in plaintext in SQLite.

---

# 3. Index Management Requirements

### ER-I1 — Index Allocation
WHEN an invoice is created  
THEN the system SHALL increment `current_index` and bind it permanently to that invoice.

### SR-I2 — No Reuse
WHILE an index has been allocated  
THE system SHALL never reassign that index to another invoice.

### ER-I3 — Mnemonic Reset
WHEN mnemonic changes  
THEN the system SHALL reset index state and accounting database.

---

# 4. Invoice Lifecycle Requirements

## Invoice States
`pending`, `detected`, `confirmed`, `expired`, `late_payment`, `underpaid`, `overpaid`, `swept`

### ER-L1 — Invoice Creation
WHEN merchant creates invoice  
THEN system SHALL:
- Allocate new HD index
- Derive deposit address
- Store invoice with amount, expiry, memo, customer_identifier
- Set status = `pending`

### ER-L2 — Detection
WHEN polling detects a USDT transfer equal to invoice amount  
THEN system SHALL set status = `detected`.

### ER-L3 — Underpayment
WHEN transfer amount < invoice amount  
THEN system SHALL set status = `underpaid`.

### ER-L4 — Overpayment
WHEN transfer amount > invoice amount  
THEN system SHALL set status = `overpaid`.

### ER-L5 — Confirmation
WHEN confirmations ≥ configured threshold  
THEN system SHALL set status = `confirmed`.

### ER-L6 — Expiry
WHEN current time exceeds expiry  
AND invoice not confirmed  
THEN system SHALL set status = `expired`.

### ER-L7 — Late Payment
WHEN payment is detected after expiry  
THEN system SHALL set status = `late_payment`.

### SR-L8 — Creation Guard
WHILE any invoice is `pending`, `detected`, or `confirmed`  
THE system SHALL block creation of new invoice.

---

# 5. Sweep Requirements

### ER-S1 — Manual Sweep
WHEN merchant initiates sweep  
AND invoice status = `confirmed`  
THEN system SHALL:
- Transfer USDT to master account
- Transfer remaining MATIC to master account
- Record sweep transaction
- Update invoice status = `swept`
- Log sweep report

### SR-S2 — Sweep Guard
WHILE invoice status ≠ `confirmed`  
THE system SHALL prevent sweep.

---

# 6. Prefunding Requirements

### ER-P1 — Manual Prefunding
WHEN merchant executes prefunding  
THEN system SHALL:
- Derive next N indices from `hd_state.current_index`
- Transfer configured MATIC amount
- Record prefund batch in `prefund_batches`
- Update `address_gas_state` for each derived address

### SR-P2 — No Automatic Prefunding
The system SHALL not execute prefunding without explicit merchant action.

### SR-P3 — Skip Used/Swept Addresses
WHILE an address has already been used for an invoice and its funds swept to the master account  
THE system SHALL NOT include that address in any future prefunding batch.

---

# 7. Detection Requirements

### ER-D1 — Polling Mode
IF detection_mode = `polling`  
THEN system SHALL monitor only the active invoice address.

### SR-D2 — No Active Invoice
WHILE no active invoice exists  
THE system SHALL not poll blockchain.

---

# 8. Configuration Requirements

### ER-C1 — Expiry Configuration
WHEN merchant sets expiry duration  
THEN system SHALL apply it to new invoices only.

### ER-C2 — Confirmation Threshold
WHEN merchant sets confirmation threshold  
THEN system SHALL apply it for confirmation logic.

### ER-C3 — RPC Configuration
WHEN merchant sets RPC provider URL  
THEN system SHALL use it globally.

---

# 9. Backup & Recovery Requirements

### ER-B1 — Backup Export
WHEN merchant exports backup  
THEN system SHALL generate encrypted file containing wallet vault and SQLite database.

### ER-B2 — Restore
WHEN valid backup is imported  
THEN system SHALL restore wallet, configuration, and accounting state.