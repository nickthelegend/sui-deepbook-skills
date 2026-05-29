---
name: deepbook-margin-sdk-pool
description: Supply assets, withdraw liquidity, manage supplier caps, and query pool metrics using the TypeScript SDK.
---

# DeepBook Margin SDK: Margin Pool

The Margin Pool SDK provides functions for supplying assets to earn interest, managing supplier capabilities, tracking referrals, and withdrawing referral fees.

---

## 1. SDK API Reference (`client.deepbook.marginPool`)

### Supplier Operations
- `mintSupplierCap()`
  Mints a new `SupplierCap` object that is used to supply and withdraw from pools.
  - *Returns*: `(tx: Transaction) => TransactionArgument`
- `supplyToMarginPool(coinKey: string, supplierCap: TransactionObjectArgument, amountToDeposit: number, referralId?: string)`
  Supplies liquidity into the margin pool. Users optionally associate deposits with a `referralId` to reward the referrer.
  - *Returns*: `(tx: Transaction) => void`
- `withdrawFromMarginPool(coinKey: string, supplierCap: TransactionObjectArgument, amountToWithdraw?: number)`
  Redeems supply shares to withdraw liquidity. If `amountToWithdraw` is omitted, it redeems all available shares.
  - *Returns*: `(tx: Transaction) => void`

### Supply Referral Actions
- `mintSupplyReferral(coinKey: string)`
  Mints a new `SupplyReferral` object for the specified asset type.
  - *Returns*: `(tx: Transaction) => void`
- `withdrawReferralFees(coinKey: string, referralId: string)`
  Withdraws accumulated referral fees belonging to the referrer.
  - *Returns*: `(tx: Transaction) => void`

---

## 2. Read-Only Query Functions

These query pool metrics and supplier positions without initiating transaction updates:
- `totalSupply(coinKey: string)`: Returns the total amount of assets supplied to the pool.
- `totalBorrow(coinKey: string)`: Returns the total amount of assets borrowed.
- `interestRate(coinKey: string)`: Returns the current borrow interest rate.
- `userSupplyShares(coinKey: string, supplierCapId: string)`: Returns a supplier's raw supply shares.
- `userSupplyAmount(coinKey: string, supplierCapId: string)`: Returns a supplier's total accrued asset balance.

---

## 3. Transaction Block Examples

### Example: Mint Cap & Supply USDC
```typescript
import { Transaction } from '@mysten/sui/transactions';

async function supplyLiquidity() {
    const tx = new Transaction();
    
    // 1. Mint a new supplier capability object
    const supplierCap = tx.add(traderClient.client.deepbook.marginPool.mintSupplierCap());
    
    // 2. Supply 1,000 USDC using the cap
    traderClient.client.deepbook.marginPool.supplyToMarginPool(
        'USDC', 
        supplierCap, 
        1000
    )(tx);
    
    // 3. Transfer the supplier cap object to the user's address to maintain ownership
    tx.transferObjects([supplierCap], tx.pure.address(traderClient.getActiveAddress()));

    await traderClient.signAndExecute(tx);
}
```

### Example: Supply USDC with Referral ID
```typescript
import { Transaction } from '@mysten/sui/transactions';

async function supplyWithReferrer(supplierCapId: string, referralId: string) {
    const tx = new Transaction();
    const supplierCap = tx.object(supplierCapId);

    traderClient.client.deepbook.marginPool.supplyToMarginPool(
        'USDC',
        supplierCap,
        1000,
        referralId // Referral will earn fees from accrued interest
    )(tx);

    await traderClient.signAndExecute(tx);
}
```

### Example: Withdraw All Liquidity
```typescript
import { Transaction } from '@mysten/sui/transactions';

async function withdrawAllUSDC(supplierCapId: string) {
    const tx = new Transaction();
    const supplierCap = tx.object(supplierCapId);

    // Omit amountToWithdraw parameter to claim and withdraw everything
    traderClient.client.deepbook.marginPool.withdrawFromMarginPool(
        'USDC',
        supplierCap
    )(tx);

    await traderClient.signAndExecute(tx);
}
```
