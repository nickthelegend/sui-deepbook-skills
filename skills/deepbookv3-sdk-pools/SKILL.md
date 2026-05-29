---
name: deepbookv3-sdk-pools
description: Query pool state, fetch order book depth, create pools, register DEEP price points, and manage referral payouts using the TS SDK.
---

# DeepBook V3: Pools SDK

The `Pool` represents a market. The `@mysten/deepbook-v3` SDK provides interfaces to fetch order book depth, execute dry runs, configure pools, feed price points, and manage referral earnings.

> [!NOTE]
> **Standard Decimal Handling**: The SDK handles token unit conversions internally. Prices, quantities, and inputs should be supplied as standard decimals (e.g. `10.5` SUI, `0.0001` BETH). Returns are also formatted in standard decimals.

---

## 1. Primary SDK Methods

### Read-Only Queries
- `account(poolKey, balanceManagerKey)`
  Returns pool account data for a balance manager (stake, volume, open orders, settled/owed balances).
- `accountOpenOrders(poolKey, managerKey)`
  Returns a promise with an array of open order IDs.
- `getOrder(poolKey, orderId)`
  Returns details of a specific resting order.
- `getOrders(poolKey, orderIds)`
  Returns details for a vector of resting orders.
- `lockedBalance(poolKey, balanceManagerKey)`
  Returns the locked base, quote, and DEEP tokens.
- `poolTradeParams(poolKey)`
  Returns `(takerFee, makerFee, stakeRequired)`.
- `poolBookParams(poolKey)`
  Returns `(tickSize, lotSize, minSize)`.
- `midPrice(poolKey)`
  Returns the current mid-price of the pool.
- `whitelisted(poolKey)`
  Checks if the pool is whitelisted (zero-fee).
- `vaultBalances(poolKey)`
  Returns the total vault balances (base, quote, DEEP) held by the pool.
- `getPoolIdByAssets(baseType, quoteType)`
  Returns the object ID of the pool representing the asset pair.
- `getBalanceManagerIds(owner)`
  Returns all `BalanceManager` IDs owned by the specified address.
- `getPoolDeepPrice(poolKey)`
  Queries the pool's internal DEEP price point feed.

### Order Book Depth
- `getLevel2Range(poolKey, priceLow, priceHigh, isBid)`
  Returns price and quantity arrays in the specified price range.
- `getLevel2TicksFromMid(poolKey, ticks)`
  Returns bid and ask price/quantity arrays up to the specified tick count.

### Swap Simulations (Dry Runs)
- `getQuoteQuantityOut(poolKey, baseQuantity)`
  Simulates a base $\rightarrow$ quote swap. Returns quote out and DEEP required.
- `getBaseQuantityOut(poolKey, quoteQuantity)`
  Simulates a quote $\rightarrow$ base swap. Returns base out and DEEP required.
- `getQuantityOut(poolKey, baseQuantity, quoteQuantity)`
  Simulates swap. Either `baseQuantity` or `quoteQuantity` must be zero.

### Pool Administration (Transaction Builders)
- `createPermissionlessPool(params)(tx)`
  Creates a new pool. Parameters include: `baseCoinKey`, `quoteCoinKey`, `tickSize`, `lotSize`, `minSize`, and optional `deepCoin` payment coin.
- `addDeepPricePoint(targetPoolKey, referencePoolKey)(tx)`
  Feeds reference DEEP conversion rate into target pool.
- `updatePoolAllowedVersions(poolKey)(tx)`
  Upgrades allowed package versions inside the pool.

### Pool Referrals (Transaction Builders)
- `mintReferral(poolKey, multiplier)(tx)`
  Mints a new referral object for the pool. (Multiplier must be multiple of 0.1 and $\le 2.0$).
- `updateReferralMultiplier(poolKey, referral, multiplier)(tx)`
  Updates the referral's multiplier.
- `claimReferralRewards(poolKey, referral)(tx)`
  Claims accumulated referral rewards. Returns `{ baseRewards, quoteRewards, deepRewards }`.
- `getReferralBalances(poolKey, referral)` *(Read-Only)*
  Returns the referral's current accumulated fees.

---

## 2. Examples

### Fetching Order Book Depth

```typescript
import { DeepBookClient } from '@mysten/deepbook-v3';

const client = new DeepBookClient(suiClient);

async function checkMarket() {
    const poolKey = 'SUI_DBUSDC';
    
    // 1. Get current mid-price
    const price = await client.deepbook.midPrice(poolKey);
    console.log(`Mid Price: ${price}`);

    // 2. Fetch Level 2 depth
    const depth = await client.deepbook.getLevel2TicksFromMid(poolKey, 10);
    console.log("Bids:", depth.bid_prices, depth.bid_quantities);
    console.log("Asks:", depth.ask_prices, depth.ask_quantities);
}
```

### Creating a Pool and Minting Referral

```typescript
import { Transaction } from '@mysten/sui/transactions';

async function createMarketAndReferral() {
    const tx = new Transaction();

    // 1. Create SUI/USDC permissionless pool
    tx.add(client.deepbook.createPermissionlessPool({
        baseCoinKey: 'SUI',
        quoteCoinKey: 'USDC',
        tickSize: 0.001,      // Handled as standard decimals by the SDK
        lotSize: 0.1,
        minSize: 1.0,
    }));

    // 2. Mint referral with 1.2x multiplier
    const referral = tx.add(client.deepbook.mintReferral('SUI_USDC', 1.2));
    tx.transferObjects([referral], keypair.toSuiAddress());

    await suiClient.signAndExecuteTransaction({
        transaction: tx,
        signer: keypair
    });
}
```
