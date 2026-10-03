# Trustless Work x Pollar Integration Guide

A step-by-step guide for integrating Pollar's abstracted wallet infrastructure with the Trustless Work escrow API. This tutorial takes you from zero to a working escrow deployment.

---

## Table of Contents

1. [Prerequisites](#prerequisites)
2. [Environment Setup](#environment-setup)
3. [Wallet Initialization via Pollar](#wallet-initialization-via-pollar)
4. [Escrow Deployment via Trustless Work API](#escrow-deployment-via-trustless-work-api)
5. [End-to-End Example](#end-to-end-example)
6. [Common Pitfalls & Gotchas](#common-pitfalls--gotchas)

---

## Prerequisites

Before starting, ensure you have the following:

- **Node.js** v18+ installed ([nvm](https://github.com/nvm-sh/nvm) recommended)
- **npm** v9+ or **yarn** v1.22+
- A **Git** account with access to the [Trustless Work repository](https://github.com/Trustless-Work/trustlesswork-clonable-backoffice)
- An **EVM-compatible wallet** (e.g., MetaMask) connected to [Base](https://base.org) (the primary chain for both systems)
- Familiarity with basic HTTP requests and JSON

### Recommended Tools

| Tool | Purpose |
|------|---------|
| [Postman](https://www.postman.com/) | API testing |
| [curl](https://curl.se/) | CLI HTTP requests |
| [VS Code](https://code.visualstudio.com/) | Editor with Git integration |
| [Hardhat](https://hardhat.org/) | (Optional) Local EVM testing |

---

## Environment Setup

### Step 1: Clone the Repository

```bash
git clone https://github.com/Trustless-Work/trustlesswork-clonable-backoffice.git
cd trustlesswork-clonable-backoffice
git checkout -b docs/pollar-tw-integration-tutorial spike/pollar-wallet-integration
```

### Step 2: Install Dependencies

```bash
npm install
# or
yarn install
```

### Step 3: Configure Environment Variables

Create a `.env` file in the project root:

```env
# Trustless Work API
TW_API_URL=https://api.trustless.work/v1
TW_API_KEY=your_api_key_here

# Pollar SDK
POLLAR_CHAIN_ID=8453  # Base mainnet
POLLAR_PROJECT_ID=your_pollar_project_id
```

> **Note:** Obtain your Trustless Work API key from the [dashboard](https://dashboard.trustless.work) and your Pollar project ID from the [Pollar console](https://console.pollar.xyz).

### Step 4: Verify Installation

```bash
npm run test
```

All tests should pass before proceeding.

---

## Wallet Initialization via Pollar

Pollar provides an abstracted wallet layer that simplifies account abstraction (ERC-4337) for end users. Follow these steps to initialize a wallet.

### Step 1: Install the Pollar SDK

```bash
npm install @pollar/sdk
# or
yarn add @pollar/sdk
```

### Step 2: Initialize the Pollar Client

```typescript
import { PollarClient } from '@pollar/sdk';

const pollar = new PollarClient({
  projectId: process.env.POLLAR_PROJECT_ID,
  chainId: Number(process.env.POLLAR_CHAIN_ID), // 8453 for Base
});

// Create or restore a user account
const userAddress = await pollar.createAccount({
  email: 'developer@example.com',
  // Or use social login:
  // socialProvider: 'google',
});

console.log('Wallet initialized:', userAddress);
```

### Step 3: Sign a Payload

Pollar handles signature abstraction. You can sign messages without exposing private keys:

```typescript
const signedPayload = await pollar.signTypedData({
  address: userAddress,
  domain: {
    name: 'TrustlessWorkEscrow',
    version: '1',
    chainId: 8453,
    verifyingContract: '0x...', // Escrow contract address
  },
  types: {
    EscrowDeployment: [
      { name: 'beneficiary', type: 'address' },
      { name: 'amount', type: 'uint256' },
      { name: 'currency', type: 'address' },
    ],
  },
  value: {
    beneficiary: '0xBENEFICIARY_ADDRESS',
    amount: '1000000000000000000', // 1 ETH in wei
    currency: '0x0000000000000000000000000000000000000000', // ETH (native)
  },
});

console.log('Signed payload:', signedPayload);
```

---

## Escrow Deployment via Trustless Work API

Once the wallet is initialized, deploy an escrow using the Trustless Work API.

### Step 1: Set Up the TW Client

```typescript
import axios from 'axios';

const TW_BASE_URL = process.env.TW_API_URL || 'https://api.trustless.work/v1';

const twClient = axios.create({
  baseURL: TW_BASE_URL,
  headers: {
    'Content-Type': 'application/json',
    'X-API-Key': process.env.TW_API_KEY,
  },
});
```

### Step 2: Deploy an Escrow

```typescript
interface EscrowDeployment {
  beneficiary: string;
  amount: string;       // In wei (smallest unit)
  currency: string;     // 0x... for ERC-20, 0x000...000 for native token
  deployer: string;     // The Pollar-managed wallet address
  nonce: number;
}

async function deployEscrow(deployment: EscrowDeployment, signature: string) {
  const response = await twClient.post('/escrows/deploy', {
    ...deployment,
    signature,
    signatureType: 'eip712', // or 'personal_sign'
  });

  return response.data;
}
```

### Step 3: Verify Deployment

```typescript
async function verifyEscrow(escrowId: string) {
  const response = await twClient.get(`/escrows/${escrowId}`);
  return response.data;
}
```

---

## End-to-End Example

Here's a complete, annotated script that ties everything together:

```typescript
// e2e-integration.ts
import { PollarClient } from '@pollar/sdk';
import axios from 'axios';

const POLLAR_PROJECT_ID = process.env.POLLAR_PROJECT_ID!;
const TW_API_KEY = process.env.TW_API_KEY!;
const TW_API_URL = process.env.TW_API_URL || 'https://api.trustless.work/v1';

// ─── Configuration ───────────────────────────────────────────────
const CHAIN_ID = 8453; // Base mainnet
const BENEFICIARY = '0xBENEFICIARY_ADDRESS';
const DEPLOYER_EMAIL = 'developer@example.com';
const ESCROW_AMOUNT = '1000000000000000000'; // 1 ETH in wei
const CURRENCY = '0x0000000000000000000000000000000000000000'; // Native ETH

// ─── Step 1: Initialize Pollar Wallet ───────────────────────────
async function initWallet() {
  const pollar = new PollarClient({
    projectId: POLLAR_PROJECT_ID,
    chainId: CHAIN_ID,
  });

  const address = await pollar.createAccount({ email: DEPLOYER_EMAIL });
  console.log(`[1] Wallet created: ${address}`);
  return { pollar, address };
}

// ─── Step 2: Sign Escrow Deployment Payload ─────────────────────
async function signDeployment(pollar: PollarClient, address: string) {
  const signature = await pollar.signTypedData({
    address,
    domain: {
      name: 'TrustlessWorkEscrow',
      version: '1',
      chainId: CHAIN_ID,
      verifyingContract: '0xESCROW_CONTRACT_ADDRESS', // From TW docs
    },
    types: {
      EscrowDeployment: [
        { name: 'beneficiary', type: 'address' },
        { name: 'amount', type: 'uint256' },
        { name: 'currency', type: 'address' },
      ],
    },
    value: {
      beneficiary: BENEFICIARY,
      amount: ESCROW_AMOUNT,
      currency: CURRENCY,
    },
  });

  console.log('[2] Payload signed successfully');
  return signature;
}

// ─── Step 3: Deploy Escrow via Trustless Work API ───────────────
async function deployEscrow(signature: string, address: string) {
  const twClient = axios.create({
    baseURL: TW_API_URL,
    headers: {
      'Content-Type': 'application/json',
      'X-API-Key': TW_API_KEY,
    },
  });

  const response = await twClient.post('/escrows/deploy', {
    deployer: address,
    beneficiary: BENEFICIARY,
    amount: ESCROW_AMOUNT,
    currency: CURRENCY,
    signature,
    signatureType: 'eip712',
    nonce: Math.floor(Date.now() / 1000),
  });

  const { escrowId, status } = response.data;
  console.log(`[3] Escrow deployed! ID: ${escrowId}, Status: ${status}`);
  return { escrowId, status };
}

// ─── Main ────────────────────────────────────────────────────────
async function main() {
  try {
    console.log('=== Trustless Work x Pollar Integration ===\n');

    const { pollar, address } = await initWallet();
    const signature = await signDeployment(pollar, address);
    const result = await deployEscrow(signature, address);

    console.log('\n✅ Integration complete!');
    console.log(`   Escrow ID: ${result.escrowId}`);
    console.log(`   Status:    ${result.status}`);
  } catch (error) {
    console.error('❌ Integration failed:', error);
    process.exit(1);
  }
}

main();
```

### Running the Script

```bash
npx ts-node e2e-integration.ts
```

Expected output:
```
=== Trustless Work x Pollar Integration ===

[1] Wallet created: 0xYOUR_WALLET_ADDRESS
[2] Payload signed successfully
[3] Escrow deployed! ID: esc_abc123xyz, Status: pending

✅ Integration complete!
   Escrow ID: esc_abc123xyz
   Status:    pending
```

---

## Common Pitfalls & Gotchas

### 1. Chain Mismatch

**Problem:** Pollar is configured for Base (chainId `8453`), but the Trustless Work API expects a different chain.

**Solution:** Always verify both systems are pointing to the same network. Check your environment variables:

```bash
echo $POLLAR_CHAIN_ID  # Should be 8453 for Base
echo $TW_CHAIN_ID      # Should match
```

### 2. Signature Type Mismatch

**Problem:** The `signatureType` field in the API request doesn't match the signing method used.

**Solution:** Use `eip712` for typed data signatures (recommended) and `personal_sign` only for simple message signing. The escrow contract must support the signature type you choose.

### 3. Insufficient Funds for Gas

**Problem:** The Pollar wallet has tokens but no native gas (ETH on Base).

**Solution:** Fund the wallet with native ETH via the [Pollar funding flow](https://docs.pollar.xyz/funding) or use Pollar's built-in gas sponsorship if enabled.

### 4. Expired Nonce

**Problem:** The escrow deployment nonce is stale because too much time passed between signing and submission.

**Solution:** Generate the nonce and sign the payload in the same operation, and submit immediately:

```typescript
const nonce = Math.floor(Date.now() / 1000);
const signature = await pollar.signTypedData({ ..., nonce });
const result = await twClient.post('/escrows/deploy', { nonce, ... });
```

### 5. Missing API Key Header

**Problem:** The Trustless Work API returns a `401 Unauthorized` error.

**Solution:** Verify the `X-API-Key` header is set correctly:

```bash
echo $TW_API_KEY  # Ensure it's not empty
```

### 6. ERC-20 Currency Address Format

**Problem:** Passing an incorrect or unverified ERC-20 contract address.

**Solution:** Only use currency addresses from the [approved list](https://docs.trustless.work/currencies) or verify them on-chain before deployment.

---

## Quick Reference

| Concept | Resource |
|---------|----------|
| Pollar SDK Docs | https://docs.pollar.xyz |
| Trustless Work API Docs | https://docs.trustless.work/api |
| Base Block Explorer | https://basescan.org |
| Escrow Contract ABI | `./contracts/abis/Escrow.json` |

---

*Last updated: 2025-07-15*
*Author: Trustless Work Developer Relations*
