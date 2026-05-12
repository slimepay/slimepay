# SlimePay — Technical Architecture

> Version 1.0 · May 2026

SlimePay is a multi-chain crypto-to-fiat payment platform that lets consumers and merchants receive value across 14+ blockchains and settle directly into Nigerian bank accounts (NGN). The system is composed of four major products sharing a common backend engine.

---

## Table of Contents

1. [System Overview](#1-system-overview)
2. [Product Suite](#2-product-suite)
3. [Technology Stack](#3-technology-stack)
4. [Blockchain Integrations](#4-blockchain-integrations)
5. [Supported Assets](#5-supported-assets)
6. [Core Service Architecture](#6-core-service-architecture)
7. [Data Flow](#7-data-flow)
8. [T9N SDK — Merchant Checkout](#8-t9n-sdk--merchant-checkout)
9. [Database Schema](#9-database-schema)
10. [External Integrations](#10-external-integrations)
11. [Security & Compliance](#11-security--compliance)
12. [Deployment](#12-deployment)
13. [API Surface](#13-api-surface)

---

## 1. System Overview

```
┌────────────────────────────────────────────────────────────────────────┐
│                          SlimePay Platform                             │
│                                                                        │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  ┌──────────┐  │
│  │  Consumer    │  │  Merchant    │  │    T9N SDK   │  │  Admin   │  │
│  │   App        │  │   Portal     │  │  (Checkout)  │  │Dashboard │  │
│  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘  └────┬─────┘  │
│         │                 │                  │               │         │
│         └─────────────────┴──────────────────┴───────────────┘         │
│                                    │                                   │
│                         ┌──────────▼──────────┐                        │
│                         │   Go Backend API     │                        │
│                         │  (PocketBase + Gin)  │                        │
│                         └──────────┬──────────┘                        │
│                                    │                                   │
│         ┌──────────────────────────┼──────────────────────────┐        │
│         │                          │                          │        │
│  ┌──────▼──────┐          ┌────────▼────────┐       ┌────────▼─────┐  │
│  │  SQLite /   │          │   Redis Cache   │       │  MongoDB     │  │
│  │ PocketBase  │          │ (sessions, rates│       │ (merchant    │  │
│  │  (primary)  │          │  rate limits)   │       │  portal)     │  │
│  └─────────────┘          └─────────────────┘       └──────────────┘  │
└────────────────────────────────────────────────────────────────────────┘

External services:
  Moralis (streams + balances)   │  Maplerad / SafeHaven (NGN settlement)
  Blockstream (Bitcoin UTXOs)    │  CoinGecko (exchange rates)
  Dojah (KYC / AML)             │  Resend (transactional email)
  TronGrid (Tron fees)           │  Hedera Mirror Node
  Divvi (on-chain referrals)     │  Slack (ops alerts)
```

---

## 2. Product Suite

| Product | Description | Primary Users |
|---------|-------------|---------------|
| **Consumer App** | Deposit crypto, receive NGN in bank account. Supports 14+ chains, KYC tiers, auto-settlement, BTC.H rewards. | End users |
| **Merchant Portal** | Dashboard + API key management + analytics for businesses accepting crypto. | Merchants |
| **T9N SDK** (`@tineon/t9n`) | Drop-in Shadow DOM checkout modal. One line of JS to accept crypto on any website. | Developers integrating checkout |
| **Admin Dashboard** | Internal tooling — manual sweeps, payout approval, BTC.H withdrawal management, merchant oversight. | SlimePay ops team |

---

## 3. Technology Stack

### Frontend
| Layer | Technology |
|-------|-----------|
| Framework | Next.js 15.5 (App Router) + React 19 |
| Styling | Tailwind CSS + Radix UI |
| State | Zustand + TanStack React Query |
| Web3 | Wagmi, Viem |
| Wallet SDKs | MetaMask SDK, Coinbase Wallet SDK, WalletConnect, Safe Global SDK, Gemini Wallet |
| Charts | Recharts |
| Forms | React Hook Form + Zod |
| SDK Isolation | Shadow DOM (T9N checkout modal) |

### Backend
| Layer | Technology |
|-------|-----------|
| Language | Go 1.21+ |
| Framework | PocketBase (embedded) + Gin-style router |
| Primary DB | SQLite via PocketBase |
| Merchant DB | MongoDB |
| Cache / Sessions | Redis (go-redis/v9) |
| Config | Viper (`.env` file) |
| Email | Resend (SMTP) |

### Infrastructure
| Component | Technology |
|-----------|-----------|
| Container | Docker (multi-stage builds) |
| Frontend port | 3000 |
| Backend port | 8090 |
| Encryption | Argon2 KDF + AES-256-GCM |

---

## 4. Blockchain Integrations

### 4.1 EVM Chains

SlimePay supports all EVM-compatible networks through a single `CryptoService` implementation using `go-ethereum`.

| Network | Symbol | Chain ID |
|---------|--------|----------|
| Ethereum | ETH | 1 |
| Polygon | MATIC | 137 |
| Binance Smart Chain | BNB | 56 |
| Arbitrum | ARB | 42161 |
| Avalanche C-Chain | AVAX | 43114 |
| Base | ETH | 8453 |
| Lisk | LSK | — |
| Flow | FLOW | — |
| Ronin | RON | 2020 |
| Sei | SEI | — |
| Fantom | FTM | 250 |

**Wallet generation:**
```
ECDSA secp256k1 keypair → Keccak256 hash → 0x... address
```

**Transaction signing:**
```
EIP-155 (replay-protected) → RPC broadcast
EIP-1559 (priorityFee + baseFee) where supported
ERC-20 transfers encoded via ABI transfer(address,uint256)
```

**Deposit detection:**  
Moralis Streams webhook → `logs[]` filtered for `Transfer` event topic → validate `to` address + decoded amount.

---

### 4.2 Bitcoin

| Attribute | Value |
|-----------|-------|
| Address format | P2WPKH bech32 (`bc1q…`) |
| Key library | `btcec` secp256k1 |
| UTXO source | Blockstream.info REST API |
| Broadcast | Blockstream.info `/tx` endpoint |
| Confirmation | Blockstream `/tx/:txid/status` |

**Fee calculation:**
```
vSize = base_overhead + (68 × n_inputs) + (31 × n_outputs)
fee   = vSize × fee_rate_sat_per_vbyte × 1.2   // 20% safety buffer
```

---

### 4.3 Solana

| Attribute | Value |
|-----------|-------|
| Address format | Base58 Ed25519 pubkey |
| Library | `solana-go` |
| Tokens | SPL Token Program |
| ATA | Auto-created if missing before first transfer |
| Confirmation | `GetSignatureStatuses` → `ConfirmationStatusFinalized` |

---

### 4.4 Tron

| Attribute | Value |
|-----------|-------|
| Key generation | `btcec` secp256k1 (same curve as Bitcoin/ETH) |
| Transport | gRPC to TronNode |
| Tokens | TRC-20 (primarily USDT) |
| Fee estimation | TronGrid API dynamic `energyPrice` + `bandwidthPrice` |
| Confirmation | `GetTransactionInfoByID` → `Transaction_Result_SUCCESS` |

---

### 4.5 Hedera

| Attribute | Value |
|-----------|-------|
| Native asset | HBAR |
| Token standard | HTS (Hedera Token Service) |
| BTC.H token | Wrapped Bitcoin on HTS |
| Balance queries | Hedera Mirror Node API |
| Account creation | `AccountCreateTransaction` (Hedera SDK) |

---

## 5. Supported Assets

### Native Currencies (13)
`BTC · ETH · MATIC · BNB · AVAX · SOL · TRX · LSK · FLOW · RON · SEI · FTM · HBAR`

### Stablecoins
| Token | Chains |
|-------|--------|
| USDT | Ethereum, Polygon, BSC, Arbitrum, Avalanche, Solana, Tron, Lisk, Flow, Ronin, Sei, Fantom |
| USDC | Avalanche |
| BTC.H (wrapped BTC) | Ethereum, Base, Polygon, BSC, Arbitrum, Avalanche, Hedera |

### Reward Token
| Token | Mechanism |
|-------|-----------|
| Slime Coins | Referral rewards (20 coins per referral completion) |
| BTC.H | 0.5 per completed transaction; withdraw at ≥ 20 BTC.H |

---

## 6. Core Service Architecture

### 6.1 CryptoService

Universal blockchain abstraction over all supported networks.

```
CryptoService
├── GenerateWallet(chain)                    → address + encrypted private key
├── GetBalance(address, chain, token)        → decimal balance
├── SignAndSendTx(wallet, network, to, amt)  → tx hash
├── ConfirmTx(network, txHash)               → confirmed bool
├── VerifyDeposit(network, txHash, addr, amt)→ valid bool
├── EstimateTxFee(wallet, network, to, amt)  → fee in native currency
├── GetExchangeRate(currency)                → USD rate (CoinGecko)
└── EnsureAddressInStream(address, network)  → Moralis stream subscription
```

Private keys are stored encrypted:
```
plaintext key
  → Argon2id(salt, params) → 32-byte derived key
  → AES-256-GCM(nonce, derivedKey, plaintext)
  → [salt || nonce || ciphertext] stored in DB
```

---

### 6.2 WalletService

Manages the two-level wallet hierarchy:

```
User Sub-Wallet (per user × network × currency)
     ↓  sweep on deposit
Master Wallet  (per network × currency — admin-owned)
     ↓  withdrawal
Recipient Bank Account (NGN)
```

- Sub-wallets created on demand via `GetOrCreateWallet(userID, network, currency)`  
- Master wallets are unique per `(network, currency)` pair, mutex-protected  
- All wallets auto-subscribed to Moralis Streams on creation

---

### 6.3 TransactionService

Orchestrates the full payment lifecycle:

```
[User initiates]
      │
   pending ──► expired (30 min timeout)
      │
   in_review  (flagged for manual approval)
      │
   processing  (sweep initiated)
      │
   pending_settlement  (awaiting bank payout)
      │
   completed  (NGN in recipient account + BTC.H reward allocated)
      │
   ──► failed / canceled / disputed / rejected
       sweep_failed / payout_failed (retryable)
```

KYC-based daily limits enforced at initialization:

| Tier | Single Tx Cap | Daily Cap |
|------|--------------|-----------|
| Tier 1 | 1,650,000 NGN | 1,650,000 NGN |
| Tier 2 | 4,650,000 NGN | 4,650,000 NGN |

---

### 6.4 TopUpService (Gas Management)

Ensures user wallets always have gas to execute token sweeps.

```
Startup
  └─ EnsureGasDistributors() → create 1 distributor wallet per network

Every 50s
  └─ TopUpNeedyWallets()
       → find wallets: token balance > 0, gas < estimated_fee
       → send 4× estimated_fee from distributor
       → record in wallet_topups

Every 15m
  └─ MonitorDistributorBalances()
       → alert via Slack if distributor < $50 USD equivalent

Every 60s
  └─ RetryFailedSweeps()
       → re-attempt transactions with sweep_status = "failed"
```

---

### 6.5 MerchantService

Multi-tenant B2B platform with isolated MongoDB storage.

**Authentication layers:**
- Email/password (bcrypt)
- Google OAuth2
- TOTP MFA
- API key signing (HMAC-SHA256, public + secret key pair)

**Checkout session lifecycle:**
```
POST /merchant/checkout/public/sessions  →  created
      │ (currency selected)
      ▼
address_generated  →  QR shown, 30-min timer
      │ (Moralis webhook fires)
      ▼
pending_confirmation
      │ (backend verifies amount)
      ▼
settled  →  merchant webhook callback fired
```

---

### 6.6 SettlementService

Resolves which deposits trigger auto-settlement vs. manual review.

- Per-user, per `(currency, network)` pair toggle  
- Configurable USD minimum threshold (default $500)  
- Providers: **Maplerad** (primary) and **SafeHaven** (fallback)  
- All settlement attempts logged; failures enqueued for retry

---

## 7. Data Flow

### 7.1 Consumer Deposit → NGN Settlement

```
User selects network/currency in Consumer App
         │
         ▼
Backend: GetOrCreateWallet(userID, network, currency)
         │
         ▼
Deposit address + QR code displayed
         │
         ▼
User sends crypto on-chain
         │
         ├── Moralis Stream webhook (real-time, primary path)
         └── RPC block poller (every 20s, fallback / auto-settlement)
         │
         ▼
ProcessDepositOnAddress()
   → validate address, amount, token
   → match to pending transaction OR create unmatched deposit record
         │
         ▼
   auto-settlement enabled?
   ├── YES → SweepToMasterWallet()
   │              └── TopUpService tops up gas if needed
   │         ConfirmTx() — waits for on-chain finality
   │         InitiatePayout(recipient bank, NGN amount)
   │         → Maplerad / SafeHaven
   └── NO  → transition to in_review → admin approval
         │
         ▼
Transaction status = completed
BTC.H reward allocated (+0.5)
Email notification sent
```

---

### 7.2 Merchant Checkout Flow

```
Merchant site (any framework)
  │  initializeT9n({ publicKey, amountNgn, ... })
  ▼
T9N SDK (Shadow DOM modal)
  │  POST /merchant/checkout/public/sessions
  ▼
Backend creates CheckoutSession (30-min expiry, MongoDB)
  │
  ▼ (customer picks currency)
  │  POST /sessions/:id/select-currency
  ▼
Backend: GetOrCreateWallet(merchantID, network, currency)
  │  Returns deposit address + expected_amount_crypto
  ▼
SDK displays QR code + countdown timer
  │
  ▼ (customer sends crypto)
Moralis Webhook → backend verifies → session = pending_confirmation
  │
  ▼
SweepToMasterWallet() → NGN settlement → merchant recipient account
  │
  ▼
SDK fires onSuccess hook → merchant receives callback_url redirect
```

---

### 7.3 BTC.H Reward Flow

```
Transaction reaches "completed"
  │
  ▼
btch_reward_allocated = false?
  │
  ▼
users.allocated_btch_balance += 0.5
btch_reward_allocated = true
  │
  ▼
User requests withdrawal (requires ≥ 20 BTC.H)
  │
  ▼
Admin reviews btc_h_withdrawals record
  ├── Approve → WalletService sends 20 BTC.H from admin master wallet (Hedera HTS)
  │              store tx_hash, email user confirmation
  └── Reject  → restore balance, notify user with reason
```

---

## 8. T9N SDK — Merchant Checkout

`@tineon/t9n` is the embeddable checkout SDK that any merchant can drop into their site.

### Architecture

```
Host page (any framework)
  └─ <script> loads T9N SDK bundle
       └─ Shadow DOM root attached
            └─ SlimePay checkout UI rendered in isolation
                 (no style leakage, max z-index overlay)
```

### Integration

```typescript
import { initializeT9n } from "@tineon/t9n";

const checkout = initializeT9n({
  publicKey:   "pk_live_xxx",
  amountNgn:   50000,
  currencies:  ["BTC", "ETH", "SOL", "TRX"],  // native-only
  customer: {
    email:     "buyer@example.com",
    name:      "Ada Obi",
    reference: "order_12345",
  },
  metadata:    { orderId: "12345" },
  callbackUrl: "https://yourstore.com/checkout/success",
  hooks: {
    onOpen:         () => {},
    onClose:        () => {},
    onStatusChange: (status) => console.log(status),
    onSuccess:      (data)   => console.log(data),
    onFail:         (err)    => console.error(err),
  },
});

// Mount a pre-styled button
checkout.mountButton("#pay-slot", { text: "Pay with Crypto", theme: "dark" });

// Or bind any existing element
checkout.bindTrigger("#my-button", "click");

// Or open programmatically
checkout.open();
```

### Supported Currencies (SDK)
Native currencies only: `BTC · ETH · MATIC · BNB · AVAX · SOL · TRX · LSK · FLOW · RON · SEI · FTM`

Stablecoins (USDT/USDC) are accessible via the direct Merchant API for advanced integrations.

---

## 9. Database Schema

### PocketBase / SQLite (Primary)

**`users`**
```
id, email, password_hash, username, verified, verified_at
tier (1|2), kyc1CompletedAt, kyc2CompletedAt
auto_settlement (bool), defaultRecipient → recipients.id
slime_coin_balance, allocated_btch_balance, btch_address
created, updated
```

**`crypto_wallets`**
```
id, user_id ("admin" for master wallets)
chain (EVM|SOLANA|TRON|BTC|HEDERA)
network (ethereum|polygon|solana|…)
currency (native symbol or token)
address, encrypted_priv_key
created, updated
```

**`transactions`**
```
id, user → users.id, recipient → recipients.id
status (pending|in_review|processing|pending_settlement|completed|failed|…)
sweep_status (pending|awaiting_gas|completed|failed)
amountInitiated, amountReceived (crypto units)
network, wallet (currency symbol)
exchangeRateAtTransaction, expiresAt
receipt (tx hash), transferReference
btch_reward_allocated (bool)
tenderedAt, approvedAt, completedAt
created, updated
```

**`recipients`**
```
id, user → users.id
bankCode, accountNumber, accountName, bankName
created, updated
```

**`btc_h_withdrawals`**
```
id, user → users.id, email
address (user Hedera address), amount (20)
status (pending|approved|rejected)
tx_hash, approved_by, approved_at
rejected_by, rejected_at, rejection_reason
created, updated
```

**`user_settlement_preferences`**
```
id, user → users.id
currency, network, enabled (bool), address
created, updated
```

**`referrals`**
```
id, referrer → users.id, referee → users.id
status (pending|completed), points_earned, completed_at
created, updated
```

---

### MongoDB (Merchant Portal)

**`merchants`** · **`merchant_api_keys`** · **`merchant_auth_sessions`**  
**`merchant_checkout_sessions`** · **`merchant_recipients`**  
**`merchant_wallets`** · **`merchant_payment_links`** · **`merchant_payouts`**

Key fields on `merchant_checkout_sessions`:
```
session_id (UUID), merchant_id, status
amount_fiat (NGN), selected_currency, selected_network
deposit_address, expected_amount_crypto
usd_to_ngn_rate, crypto_to_usd_rate
sweep_status, payout_status
customer_email, customer_reference, metadata
expires_at (30 min from creation), settled_at
```

---

## 10. External Integrations

| Service | Role | Notes |
|---------|------|-------|
| **Moralis** | Real-time deposit detection (Streams) + multi-chain balance queries | Primary detection path; multiple API keys with rotation on exhaustion |
| **Blockstream.info** | Bitcoin UTXO queries + transaction broadcast | No API key required |
| **CoinGecko** | Crypto → USD exchange rates | 30-min Redis cache |
| **ExchangeRate API** | USD → NGN fiat conversion | Cached alongside CoinGecko data |
| **TronGrid** | Tron energy/bandwidth fee estimation | REST API |
| **Hedera Mirror Node** | HBAR + HTS token balances and tx status | Public mirror node |
| **Maplerad** | NGN bank settlement (primary) | Supports instant + next-day |
| **SafeHaven** | NGN bank settlement (fallback) | Same interface, provider failover |
| **Dojah** | KYC identity + address verification | Webhook-driven completion |
| **Resend** | Transactional email (SMTP) | Reset links, approval notices, withdrawal confirmations |
| **Divvi SDK** | On-chain referral tracking and reward tags | Links referrer to on-chain action |
| **Slack** | Ops alerts | Low distributor balance, settlement errors, sweep failures |

---

## 11. Security & Compliance

### Private Key Encryption
```
Argon2id(password=systemSecret, salt=random32, time=1, memory=64MB, threads=4)
  → 32-byte derivedKey
AES-256-GCM(key=derivedKey, nonce=random12, plaintext=privateKeyBytes)
  → stored as [salt(32) || nonce(12) || ciphertext]
```

### API Security
- **Merchant API keys**: HMAC-SHA256 request signing (public + secret key pair)
- **Session tokens**: Redis-backed, short-lived JWTs
- **Rate limiting**: 10 req/s per user via Redis; per-endpoint overrides
- **HTTPS**: enforced in production

### KYC / AML (Dojah)
| Tier | Requirement | Daily Limit |
|------|-------------|-------------|
| Tier 0 | None | N/A (deposit only, no settlement) |
| Tier 1 | NIN / ID + selfie | 1,650,000 NGN |
| Tier 2 | Address verification | 4,650,000 NGN |

Suspicious transaction patterns escalate to `in_review` status pending admin approval.

### Data Isolation
- Merchant data is entirely in MongoDB, isolated from user PocketBase instance
- No cross-tenant wallet reuse — every user and merchant gets unique addresses per chain

---

## 12. Deployment

```
┌─────────────────────────────────┐
│         Docker Compose          │
│                                 │
│  ┌─────────────┐                │
│  │  Next.js 15  │ :3000         │
│  │  (frontend)  │               │
│  └─────────────┘                │
│                                 │
│  ┌─────────────┐                │
│  │  Go Backend  │ :8090         │
│  │  PocketBase  │               │
│  │  + pb_data/  │               │
│  └─────────────┘                │
│                                 │
│  ┌──────┐  ┌──────┐             │
│  │Redis │  │Mongo │             │
│  └──────┘  └──────┘             │
└─────────────────────────────────┘
```

**Build commands:**
```bash
# Frontend
pnpm build   # → .next/standalone

# Backend
go build ./cmd/main.go   # → ./server binary

# Docker
docker compose up --build
```

**Key environment variables:**
```
NEXT_PUBLIC_API_URL          Backend URL for frontend
NEXT_PUBLIC_TIER1_DOJAH_URL  KYC widget URL
MORALIS_API_KEYS             Comma-separated list (auto-rotation)
MAPLERAD_SECRET_KEY          Settlement provider
SAFEHAVEN_API_KEY            Settlement fallback
DOJAH_APP_ID / SECRET        KYC provider
RESEND_API_KEY               Email
REDIS_URL                    Cache / sessions
MONGO_URI                    Merchant portal DB
ENCRYPTION_SECRET            AES key derivation seed
COINGECKO_API_KEY            Exchange rates
```

---

## 13. API Surface

### Consumer

| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/register` | Email/password signup |
| POST | `/api/auth/google/callback` | Google OAuth exchange |
| GET | `/api/crypto/coins` | Supported coins + live rates |
| GET | `/api/crypto/deposit-info/:currency` | Deposit address + QR |
| POST | `/api/transactions/initialize` | Create pending transaction |
| POST | `/api/transactions/verify-deposit` | Manual deposit verification |
| POST | `/api/transactions/confirm-crypto-payment` | Confirm web3 wallet payment |
| GET | `/api/banks` | Nigerian bank list |
| GET | `/api/banks/resolve` | Account name lookup |
| GET | `/api/transaction-limits` | User tier limits and usage |
| POST | `/api/settlement/preferences` | Set auto-settlement pairs |
| GET | `/api/referrals/me` | User referral details |
| GET | `/api/leaderboard` | Referral leaderboard |

### Merchant Portal

| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/merchant/auth/register` | Create merchant account |
| POST | `/api/merchant/auth/login` | Email/password login |
| POST | `/api/merchant/auth/google/callback` | Google OAuth |
| POST | `/api/merchant/portal/api-keys` | Create API key |
| POST | `/api/merchant/portal/recipients` | Configure bank settlement |
| GET | `/api/merchant/portal/summary` | Dashboard analytics |
| POST | `/api/merchant/checkout/public/sessions` | Create checkout session |
| POST | `/api/merchant/checkout/sessions/:id/select-currency` | Pick crypto |
| POST | `/api/merchant/checkout/sessions/:id/confirm-payment` | Confirm receipt |
| GET | `/api/merchant/checkout/sessions/:id` | Poll session status |
| POST | `/api/merchant/wallets/address` | Generate wallet address |
| POST | `/api/merchant/wallets/sign-send` | Execute transfer |

### Admin

| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/admin/master-wallets` | Master wallet balances |
| POST | `/api/admin/master-withdraw` | Withdraw from master |
| POST | `/api/admin/sweep-wallet` | Manual sweep |
| POST | `/api/admin/payout` | Initiate manual payout |
| GET | `/api/admin/unswept-transactions` | View pending sweeps |
| POST | `/api/admin/btch-withdrawals/:id/approve` | Approve BTC.H withdrawal |
| GET | `/api/admin/merchants` | Merchant directory |
| POST | `/api/transactions/:id/approve` | Approve transaction |
| POST | `/api/transactions/:id/reject` | Reject transaction |

### Webhooks

| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/webhooks/moralis` | Deposit detection (Moralis Streams) |
| POST | `/api/verifications/webhook` | KYC provider callback (Dojah) |

---

*For the Stellar integration architecture (SEP-24, SEP-31, SDP, Wallets Kit), see the companion document: **Stellar Integration Architecture**.*
