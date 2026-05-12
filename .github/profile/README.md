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
  Moralis        │  Maplerad / SafeHaven (NGN settlement)
  Blockstream             │  CoinGecko 
  Dojah (KYC / AML)               │  Resend 
  TronGrid (Tron fees)             │  Hedera Mirror Node
  Divvi         │  Slack (ops alerts)
```

---

## 2. Product Suite

| Product | Description | Primary Users |
|---------|-------------|---------------|
| **Consumer App** | Deposit crypto, receive NGN in bank account. Supports 14+ chains, KYC tiers, auto-settlement, BTC.H rewards. | End users |
| **Merchant Portal** | Dashboard + API key management + analytics for businesses accepting crypto. | Merchants |
| **T9N SDK** (`@tineon/t9n`) | One line of JS to accept crypto on any website. | Developers integrating checkout |

---

## 3. Technology Stack

### Frontend
| Layer | Technology |
|-------|-----------|
| Framework | Next.js 15 (App Router) + React 19 |
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
| Cache / Sessions | Redis |
| Email | Resend |

### Infrastructure
| Component | Technology |
|-----------|-----------|
| Container | Docker (multi-stage builds) |
| Encryption | Argon2id KDF + AES-256-GCM |

---

## 4. Blockchain Integrations

### 4.1 EVM Chains

SlimePay supports all EVM-compatible networks`.

| Network | Symbol |
|---------|--------|
| Ethereum | ETH |
| Polygon | MATIC |
| Binance Smart Chain | BNB |
| Arbitrum | ARB |
| Avalanche C-Chain | AVAX |
| Base | ETH |
| Lisk | LSK |
| Flow | FLOW |
| Ronin | RON |
| Sei | SEI |
| Fantom | FTM |

**Wallet generation:** ECDSA secp256k1 keypair → Keccak256 hash → Ethereum-compatible address

**Transaction signing:** EIP-155 replay-protected transactions; EIP-1559 fee model where supported

**Deposit detection:** Real-time webhook stream monitors incoming transfers and validates recipient address and amount.

---

### 4.2 Bitcoin

| Attribute | Value |
|-----------|-------|
| Address format | P2WPKH bech32 (`bc1q…`) |
| UTXO source | Public Bitcoin REST API |
| Confirmation | On-chain status check |

Fee estimation uses UTXO-based virtual size calculation with a safety buffer.

---

### 4.3 Solana

| Attribute | Value |
|-----------|-------|
| Address format | Base58 Ed25519 pubkey |
| Tokens | SPL Token Program |
| ATA | Auto-created if missing before first transfer |
| Confirmation | Finalized status check |

---

### 4.4 Tron

| Attribute | Value |
|-----------|-------|
| Transport | gRPC to TronNode |
| Tokens | TRC-20 (primarily USDT) |
| Fee estimation | Dynamic energy + bandwidth pricing |

---

### 4.5 Hedera

| Attribute | Value |
|-----------|-------|
| Native asset | HBAR |
| Token standard | HTS (Hedera Token Service) |
| BTC.H token | Wrapped Bitcoin on HTS |
| Balance queries | Hedera Mirror Node |

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
| BTC.H | Earned per completed transaction; withdrawable above a minimum threshold |

---

## 6. Core Service Architecture

### 6.1 CryptoService

Universal blockchain abstraction over all supported networks. Provides wallet generation, balance queries, transaction signing and broadcasting, deposit verification, fee estimation, and exchange rate lookups.

### 6.2 WalletService

Manages a two-level wallet hierarchy:

```
User Sub-Wallet  (unique per user × network × currency)
       ↓  swept on deposit
Master Wallet    (per network × currency — platform-owned)
       ↓  withdrawal
Recipient Bank Account (NGN)
```

Sub-wallets are created on demand and immediately enrolled in real-time deposit monitoring. Master wallets are unique per chain/currency pair with race-condition-safe creation.

---

### 6.3 TransactionService

Orchestrates the full payment lifecycle:

```
[User initiates]
      │
   pending ──► expired (30-min timeout)
      │
   in_review  (flagged for manual approval)
      │
   processing  (sweep to master wallet initiated)
      │
   pending_settlement  (awaiting bank payout)
      │
   completed  (NGN delivered to recipient + rewards allocated)
      │
   ──► failed / canceled / disputed / rejected
```

KYC-based daily limits enforced at transaction initialization:

| Tier | Requirement | Daily Limit |
|------|-------------|-------------|
| Tier 1 | NIN / ID + selfie | 1,650,000 NGN |
| Tier 2 | Address verification | 4,650,000 NGN |

---

### 6.4 Gas Management Service

Ensures user wallets always have sufficient native currency to execute token sweeps. Distributor wallets are maintained per network and automatically top up any wallet that holds tokens but lacks gas. Balance monitoring with ops alerts keeps distributors funded. Failed sweeps are automatically retried.

---

### 6.5 MerchantService

Multi-tenant B2B platform with isolated storage.

**Authentication layers:**
- Email/password
- Google OAuth2
- TOTP MFA
- HMAC-signed API keys (public + secret key pair)

**Checkout session lifecycle:**
```
Session created  →  currency selected  →  deposit address generated
       │
  QR code displayed (30-min timer)
       │
  Deposit detected  →  verification  →  settlement initiated
       │
  Merchant webhook callback fired  →  settled
```

---

### 6.6 SettlementService

Determines which confirmed deposits trigger automatic NGN payout vs. manual review.

- Configurable per user, per currency/network pair
- USD minimum threshold for auto-settlement
- Providers: Maplerad (primary) and SafeHaven (fallback)
- All attempts logged; failures enqueued for retry

---

## 7. Data Flow

### 7.1 Consumer Deposit → NGN Settlement

```
User selects network and currency
         │
         ▼
Unique deposit address generated and displayed with QR code
         │
         ▼
User sends crypto on-chain
         │
         ├── Real-time webhook (primary path)
         └── Block poller (fallback / auto-settlement path)
         │
         ▼
Deposit validated (address, amount, token)
         │
         ▼
   Auto-settlement enabled?
   ├── YES → Funds swept to master wallet
   │         On-chain finality confirmed
   │         NGN payout initiated to recipient bank
   └── NO  → Escalated to manual review → admin approval
         │
         ▼
Transaction completed
Rewards allocated
Email notification sent
```

---

### 7.2 Merchant Checkout Flow

```
Merchant site (any framework)
  │  T9N SDK initialised with public key + NGN amount
  ▼
T9N SDK (Shadow DOM modal) opens
  │
  ▼
Checkout session created (30-min expiry)
  │
  ▼ Customer picks currency
Deposit address generated and returned to SDK
SDK displays QR code + countdown timer
  │
  ▼ Customer sends crypto
Deposit detected → verified → session confirmed
  │
  ▼
Funds swept → NGN settlement → merchant recipient account
  │
  ▼
SDK fires onSuccess hook → merchant callback URL redirected
```

---


---

## 8. T9N SDK — Merchant Checkout

`@tineon/t9n` is the embeddable checkout SDK that any merchant can drop into their site.

### Architecture

```
Host page (any framework)
  └─ <script> loads T9N SDK bundle
       └─ Shadow DOM root attached
            └─ SlimePay checkout UI rendered in isolation
                
```

### Integration

```typescript
import { initializeT9n } from "@tineon/t9n";

const checkout = initializeT9n({
  publicKey:   "pk_live_xxx",
  amountNgn:   50000,
  currencies:  ["BTC", "ETH", "SOL", "TRX"],  // native currencies
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

**`users`** — Account identity, KYC tier, settlement preferences, reward balances

**`crypto_wallets`** — User and platform sub-wallets. One record per (user × network × currency). Stores wallet address and encrypted private key material. Master (platform) wallets follow the same structure.

**`transactions`** — Full lifecycle record per payment. Tracks status, amounts in both crypto and fiat equivalent, on-chain receipt reference, sweep status, and reward allocation flag.

**`recipients`** — Nigerian bank accounts linked to users (bank code, account number, account name).

**`btc_h_withdrawals`** — Pending and historical BTC.H reward withdrawal requests, including approval state and on-chain transfer reference.

**`user_settlement_preferences`** — Per-user, per currency/network auto-settlement toggles.

**`referrals`** — Referrer/referee pairs and reward completion state.

---

### MongoDB (Merchant Portal)

Isolated from the primary database. Collections cover merchant profiles, authentication sessions, checkout sessions, settlement recipients, wallets, payment links, and payout records.

Key fields on a checkout session: session ID, merchant reference, status, NGN amount, selected currency and network, deposit address, expected crypto amount, exchange rates at creation, customer reference, expiry timestamp.


---

## 10. Security & Compliance

### Private Key Encryption

Private keys are encrypted at rest using a two-step scheme:

1. **Key derivation** — Argon2id KDF stretches the platform secret with a per-key random salt into a 32-byte encryption key.
2. **Encryption** — AES-256-GCM encrypts the private key bytes with a random nonce.
3. **Storage** — Only the salt, nonce, and ciphertext are persisted; the platform secret never touches the database.

### API Security
- **Merchant API keys**: HMAC-SHA256 request signing (public + secret key pair)
- **Session tokens**: Redis-backed, short-lived tokens
- **Rate limiting**: Per-user, enforced at the API layer
- **HTTPS**: Enforced in production

### KYC / AML

| Tier | Requirement | Daily Limit |
|------|-------------|-------------|
| Tier 1 | NIN / Government ID + selfie | 1,650,000 NGN |
| Tier 2 | Address verification | 4,650,000 NGN |

Verification is handled by Dojah via webhook callbacks. Suspicious transaction patterns automatically escalate to manual review before any funds are moved.

### Data Isolation
- Merchant data is entirely in MongoDB, isolated from the user PocketBase instance
- No wallet reuse across users or merchants — every account gets unique addresses per chain


## 12. API Surface


### Merchant Portal

| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/merchant/auth/register` | Create merchant account |
| POST | `/api/merchant/auth/login` | Email/password login |
| POST | `/api/merchant/portal/api-keys` | Create API key |
| POST | `/api/merchant/portal/recipients` | Configure bank settlement |
| GET | `/api/merchant/portal/summary` | Dashboard analytics |
| POST | `/api/merchant/checkout/public/sessions` | Create checkout session |
| POST | `/api/merchant/checkout/sessions/:id/select-currency` | Select crypto |
| POST | `/api/merchant/checkout/sessions/:id/confirm-payment` | Confirm receipt |
| GET | `/api/merchant/checkout/sessions/:id` | Poll session status |
| POST | `/api/merchant/wallets/address` | Generate wallet address |
| POST | `/api/merchant/wallets/sign-send` | Execute transfer |

---
