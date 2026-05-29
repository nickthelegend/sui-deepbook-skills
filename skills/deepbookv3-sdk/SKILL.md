---
name: deepbookv3-sdk
description: Core installation, client extension setup, constants, and coin maps for the DeepBook V3 TypeScript SDK.
---

# DeepBook V3: SDK Core & Client Setup

The `@mysten/deepbook-v3` TypeScript SDK abstracts away raw Move smart contract calls into simple, unified client operations. It allows developers to programmatically build Sui transaction blocks (PTBs) and query the state of pools.

---

## 1. Installation

To install the official DeepBook V3 SDK:

```bash
npm install @mysten/deepbook-v3
# or
yarn add @mysten/deepbook-v3
# or
pnpm add @mysten/deepbook-v3
```

---

## 2. Client Initialization

To work with DeepBook V3, extend the standard `@mysten/sui` client with the `deepbook()` plugin.

### Setup Example

```typescript
import { deepbook, type DeepBookClient } from '@mysten/deepbook-v3';
import type { ClientWithExtensions } from '@mysten/sui/client';
import { SuiGrpcClient } from '@mysten/sui/grpc'; // or standard SuiClient
import { decodeSuiPrivateKey } from '@mysten/sui/cryptography';
import { Ed25519Keypair } from '@mysten/sui/keypairs/ed25519';

export class DeepBookMarketMaker {
    client: ClientWithExtensions<{ deepbook: DeepBookClient }>;
    keypair: Ed25519Keypair;

    constructor(privateKey: string, env: 'testnet' | 'mainnet') {
        const { scheme, secretKey } = decodeSuiPrivateKey(privateKey);
        if (scheme !== 'ED25519') throw new Error(`Unsupported scheme: ${scheme}`);
        
        this.keypair = Ed25519Keypair.fromSecretKey(secretKey);
        
        // Extend the Sui client with DeepBook client capabilities
        this.client = new SuiGrpcClient({
            network: env,
            baseUrl: env === 'mainnet'
                ? 'https://fullnode.mainnet.sui.io:443'
                : 'https://fullnode.testnet.sui.io:443',
        }).$extend(
            deepbook({
                address: this.keypair.toSuiAddress(), // Senders address
                // Optionally supply balance managers and admin cap
            }),
        );
    }
}
```

---

## 3. Configuration & Constants

### Deployed Registry and Package IDs
The SDK maintains references to the on-chain package and registry addresses inside `/utils/constants.ts`.

### Default Coins Map

The SDK comes preconfigured with default coin types:

- **Testnet Defaults**:
  - `DEEP`
  - `SUI`
  - `DBUSDC`
  - `DBUSDT`
- **Mainnet Defaults**:
  - `DEEP`
  - `SUI`
  - `USDC`
  - `USDT`
  - `WETH`

Traders can interact with custom assets by passing a custom `CoinMap` and `PoolMap` during client construction to register them in memory.

### Initialize with Existing BalanceManagers

To place trades, the client must know the addresses and optional `TradeCap` identifiers of the balance managers it has permission to operate. These are registered in a memory-based key-value map during client configuration:

```typescript
const balanceManagers = {
    MANAGER_1: {
        address: '0xEXISITING_BALANCE_MANAGER_ADDRESS',
        tradeCap: '0xOPTIONAL_TRADECAP_ID', // undefined if operating as direct owner
    },
};

const client = new SuiGrpcClient({ network, baseUrl }).$extend(
    deepbook({
        address: activeAddress,
        balanceManagers, // Maps MANAGER_1 key to its address & cap
    })
);
```
