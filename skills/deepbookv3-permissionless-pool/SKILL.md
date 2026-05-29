---
name: deepbookv3-permissionless-pool
description: Create permissionless pools, configure tick/lot/min sizes, feed DEEP price points, and update allowed contract versions in DeepBook V3.
---

# DeepBook V3: Permissionless Pool Creation

In DeepBook V3, anyone can permissionlessly create a trading `Pool` for any asset pair, provided a pool for that exact asset combination does not already exist.

---

## 1. Key Rules and Formulas

### Pool Parameters configuration

When creating a pool, the creator must specify three mathematical values:
1. **Tick Size**: The minimum price increment of the asset pair.
2. **Lot Size**: The minimum increment of the base asset that can be traded.
3. **Min Size**: The minimum order quantity allowed for the base asset.

#### Tick Size Formula
Tick size must be calculated using the following formula:
$$\text{Tick Size} = 10^{(9 - \text{base\_decimals} + \text{quote\_decimals} - \text{decimal\_desired})}$$

*   **Example**: Creating a SUI (9 decimals) / USDC (6 decimals) pool with a desired tick precision of 3 decimal places (representing an increment of 0.001).
    $$\text{Tick Size} = 10^{(9 - 9 + 6 - 3)} = 10^3 = 1,000$$
*   **Precision Constraint**: The desired decimal increment should be at most 1 basis point (0.01%) of the price of the base asset. For example, if the target is 3 decimals, then $0.001 / \text{price}$ must be $\le 0.0001$.

#### Lot Size and Min Size Rules
- **Lot Size**: Value in minimum units (e.g. MIST for SUI). Must be a power of 10, $\ge 1,000$, and $\le \text{Min Size}$. Typically representing a nominal value of $\$0.01 \text{ to } \$0.10$.
- **Min Size**: Value in minimum units. Must be a power of 10 and $\ge \text{Lot Size}$. Typically representing a nominal value of $\$0.10 \text{ to } \$1.00$.

### Creation Fee
Creating a permissionless pool requires a flat fee of **500 DEEP** tokens.

---

## 2. DEEP Fee Integration & Price Points

To allow the pool's traders to pay fees in DEEP (getting the 20% discount), two conditions must be met:
1. **Asset Requirement**: Either the base asset or the quote asset in the new pool must be **SUI** or **USDC**.
2. **Price Updates**: You must run a cron job (every 1-10 minutes) that calls `add_deep_price_point` to feed the conversion rate of DEEP into the pool.

### Reference Pools for DEEP Price:
- **USDC Pools**: Use `DEEP/USDC` pool ID: `0xf948981b806057580f91622417534f491da5f61aeaf33d0ed8e69fd5691c95ce` as reference.
- **SUI Pools**: Use `DEEP/SUI` pool ID: `0xb663828d6217467c8a1838a03793da896cbe745b150ebd57d82f814ca579fc22` as reference.

---

## 3. On-Chain API (Move)

The following functions are exposed for pool creation and upkeep:

- `public fun create_permissionless_pool<BaseAsset, QuoteAsset>(registry: &mut Registry, tick_size: u64, lot_size: u64, min_size: u64, fee_payment: Coin<DEEP>, ctx: &mut TxContext): Pool<BaseAsset, QuoteAsset>`
  Creates a new pool by paying the 500 DEEP fee.
- `public fun add_deep_price_point<BaseAsset, QuoteAsset, ReferenceBase, ReferenceQuote>(pool: &mut Pool<BaseAsset, QuoteAsset>, reference_pool: &Pool<ReferenceBase, ReferenceQuote>, clock: &Clock)`
  Updates the pool's conversion rate for DEEP fees using a reference pool.
- `public fun update_pool_allowed_versions<BaseAsset, QuoteAsset>(pool: &mut Pool<BaseAsset, QuoteAsset>, registry: &Registry)`
  Upgrades the allowed package versions in the pool. Must be executed after contract upgrades so the pool remains functional.

---

## 4. TS SDK Usage Examples

### Creating a Pool

```typescript
import { Transaction } from '@mysten/sui/transactions';
import { DeepBookClient } from '@mysten/deepbook-v3';

const client = new DeepBookClient(suiClient);

async function createSuiUsdcPool() {
    const tx = new Transaction();
    
    // Split 500 DEEP for the pool creation fee
    const [creationFeeCoin] = tx.splitCoins(tx.object('0xDEEP_COIN_OBJECT'), [500_000_000n]); // DEEP has 6 decimals
    
    client.deepbook.createPool(tx, {
        baseAssetType: '0x2::sui::SUI',
        quoteAssetType: '0xdba34672e30cb065b1f93e3ab55318768fd6fef66c15942c9f7cb846e2f900e7::usdc::USDC',
        tickSize: 1000n,            // Tick size calculated (e.g. 0.001)
        lotSize: 100_000n,          // Lot size (e.g. 0.0001 SUI)
        minSize: 1_000_000n,        // Min size (e.g. 0.001 SUI)
        feeCoin: creationFeeCoin,
    });
    
    await suiClient.signAndExecuteTransaction({
        transaction: tx,
        signer: keypair
    });
}
```

### Feeding DEEP Price Points

```typescript
async function feedDeepPrice(poolId: string, referencePoolId: string) {
    const tx = new Transaction();
    
    tx.moveCall({
        target: '0x337f4f4f6567fcd778d5454f27c16c70e2f274cc6377ea6249ddf491482ef497::pool::add_deep_price_point',
        typeArguments: [
            '0x...BASE_ASSET...', 
            '0x...QUOTE_ASSET...',
            '0xdeeb7a4662eec9f2f3def03fb937a663dddaa2e215b8078a284d026b7946c270::deep::DEEP',
            '0x0000000000000000000000000000000000000000000000000000000000000002::sui::SUI'
        ],
        arguments: [
            tx.object(poolId),
            tx.object(referencePoolId), // DEEP/SUI or DEEP/USDC
            tx.object('0x6')            // sui::clock::Clock object ID
        ]
    });
    
    await suiClient.signAndExecuteTransaction({
        transaction: tx,
        signer: keypair
    });
}
```
