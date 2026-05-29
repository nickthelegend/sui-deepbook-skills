---
name: deepbook-margin-sdk
description: Install, configure, and initialize the DeepBook V3 TypeScript SDK client extension for margin trading.
---

# DeepBook Margin SDK: Client Setup & Configuration

The `@mysten/deepbook-v3` TypeScript SDK abstracts transaction block generation, allowing for direct programmatic interaction with the DeepBook Margin package.

---

## 1. Installation

Install the package via your preferred package manager:

```bash
npm install @mysten/deepbook-v3
yarn add @mysten/deepbook-v3
pnpm add @mysten/deepbook-v3
```

---

## 2. Configuration & In-Memory Maps

The SDK manages a `key:value` relationship of core assets, pools, and margin managers in memory:

1. **`CoinMap`**: Maps short string identifier keys to absolute on-chain TypeNames.
2. **`PoolMap`**: Maps short pool keys to on-chain pool IDs.
3. **`MarginManagerMap`**: Maps custom names to an object specifying the manager's address and target trading pool key.

### Default Assets (Pre-configured)

- **Testnet**: `DEEP`, `SUI`, `DBUSDC`, `DBUSDT`
- **Mainnet**: `DEEP`, `SUI`, `USDC`, `USDT`, `WETH`

---

## 3. Client Initialization

To initialize the client, extend your standard Sui Client (e.g. `SuiGrpcClient` or `SuiClient`) using the `deepbook` extension:

```typescript
import { deepbook, type DeepBookClient } from '@mysten/deepbook-v3';
import type { ClientWithExtensions } from '@mysten/sui/client';
import { decodeSuiPrivateKey } from '@mysten/sui/cryptography';
import { SuiGrpcClient } from '@mysten/sui/grpc';
import { Ed25519Keypair } from '@mysten/sui/keypairs/ed25519';

class DeepBookMarginTrader {
    client: ClientWithExtensions<{ deepbook: DeepBookClient }>;
    keypair: Ed25519Keypair;

    constructor(privateKey: string, env: 'testnet' | 'mainnet') {
        this.keypair = this.getSignerFromPK(privateKey);
        this.client = new SuiGrpcClient({
            network: env,
            baseUrl: env === 'mainnet'
                ? 'https://fullnode.mainnet.sui.io:443'
                : 'https://fullnode.testnet.sui.io:443',
        }).$extend(
            deepbook({
                address: this.getActiveAddress(),
            }),
        );
    }

    getSignerFromPK = (privateKey: string): Ed25519Keypair => {
        const { scheme, secretKey } = decodeSuiPrivateKey(privateKey);
        if (scheme === 'ED25519') return Ed25519Keypair.fromSecretKey(secretKey);
        throw new Error(`Unsupported scheme: ${scheme}`);
    };

    getActiveAddress() {
        return this.keypair.toSuiAddress();
    }
}
```

---

## 4. Initializing with Active Margin Managers

Before executing margin trades, you must declare your active margin managers in the configuration map during extension setup:

```typescript
import { deepbook, type MarginManager } from '@mysten/deepbook-v3';
import { SuiGrpcClient } from '@mysten/sui/grpc';

const marginManagers: { [key: string]: MarginManager } = {
    'MARGIN_MANAGER_1': {
        address: '0x_your_margin_manager_address',
        poolKey: 'SUI_DBUSDC', // Target trading pair
    },
};

const client = new SuiGrpcClient({
    network: 'testnet',
    baseUrl: 'https://fullnode.testnet.sui.io:443',
}).$extend(
    deepbook({
        address: '0x_user_address',
        marginManagers, // Pass active managers mapping
    }),
);
```
