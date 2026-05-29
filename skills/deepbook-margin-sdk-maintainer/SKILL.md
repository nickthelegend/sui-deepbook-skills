---
name: deepbook-margin-sdk-maintainer
description: Configure pool parameters, create margin pools, enable/disable borrow permissions, and withdraw fees using the TypeScript SDK.
---

# DeepBook Margin SDK: Pool Maintenance & Administration

The Margin Maintainer SDK provides functions for managing margin pools, configuring interest rate kinks, enabling borrow paths, and claiming protocol/maintainer fees.

---

## 1. SDK API Reference (`client.deepbook.marginMaintainer`)

### Create Pools & Configurations
- `createMarginPool(coinKey: string, poolConfig: TransactionArgument)`
  Creates a new margin pool. Requires maintainer capability.
  - *Returns*: `(tx: Transaction) => void`
- `newProtocolConfig(coinKey: string, marginPoolConfig: MarginPoolConfigParams, interestConfig: InterestConfigParams)`
  Combines pool settings and interest rules into a protocol configuration object.
  - *Returns*: `(tx: Transaction) => TransactionArgument`
- `newMarginPoolConfig(coinKey: string, marginPoolConfig: MarginPoolConfigParams)`
  Creates a standalone margin pool configuration object.
  - *Returns*: `(tx: Transaction) => TransactionArgument`
- `newInterestConfig(interestConfig: InterestConfigParams)`
  Creates a standalone interest rate configuration object.
  - *Returns*: `(tx: Transaction) => TransactionArgument`

### Control Borrow Permissions
- `enableDeepbookPoolForLoan(deepbookPoolKey: string, coinKey: string, marginPoolCap: string)`
  Allows margin managers trading on the specified DeepBook pool to borrow from this margin pool.
  - *Returns*: `(tx: Transaction) => void`
- `disableDeepbookPoolForLoan(deepbookPoolKey: string, coinKey: string, marginPoolCap: string)`
  Disables borrowing for the specified DeepBook pool.
  - *Returns*: `(tx: Transaction) => void`

### Parameter Updates
- `updateInterestParams(coinKey: string, marginPoolCap: string, interestConfig: InterestConfigParams)`
  Updates interest rates, slopes, and optimal utilization benchmarks.
  - *Returns*: `(tx: Transaction) => void`
- `updateMarginPoolConfig(coinKey: string, marginPoolCap: string, marginPoolConfig: MarginPoolConfigParams)`
  Updates limits, supply caps, spreads, and minimum borrow thresholds.
  - *Returns*: `(tx: Transaction) => void`

### Fees Withdrawals
- `withdrawMaintainerFees(coinKey: string, marginPoolCap: string)`
  Withdraws accumulated maintainer fees.
  - *Returns*: `(tx: Transaction) => void`
- `withdrawProtocolFees(coinKey: string)`
  Withdraws accumulated protocol treasury fees.
  - *Returns*: `(tx: Transaction) => void`
- `adminWithdrawDefaultReferralFees(coinKey: string)`
  Withdraws default referral fees.
  - *Returns*: `(tx: Transaction) => void`

---

## 2. Configuration Parameters

```typescript
interface MarginPoolConfigParams {
    supplyCap: number;            // Max assets allowed in the pool
    maxUtilizationRate: number;   // e.g. 0.8 for 80% (upper borrow cap)
    referralSpread: number;       // e.g. 0.1 for 10% (spread percentage)
    minBorrow: number;            // Minimum loan size (to prevent spam)
}

interface InterestConfigParams {
    baseRate: number;             // APR at 0% utilization (e.g. 0.02 = 2%)
    baseSlope: number;            // APR slope before kink (e.g. 0.1 = 10%)
    optimalUtilization: number;   // Kink point (e.g. 0.8 = 80%)
    excessSlope: number;          // APR slope after kink (e.g. 1.0 = 100%)
}
```

---

## 3. Transaction Block Examples

### Example: Complete Margin Pool Creation & Initial Setup
```typescript
import { Transaction } from '@mysten/sui/transactions';

async function setupNewUSDCmarginPool(marginPoolCapId: string) {
    const tx = new Transaction();
    const coinKey = 'USDC';

    // 1. Generate configuration argument
    const poolConfig = traderClient.client.deepbook.marginMaintainer.newProtocolConfig(
        coinKey,
        {
            supplyCap: 10_000_000,     // 10M USDC
            maxUtilizationRate: 0.85,  // 85%
            referralSpread: 0.10,      // 10% spread
            minBorrow: 100,            // 100 USDC minimum
        },
        {
            baseRate: 0.02,            // 2% base
            baseSlope: 0.10,           // 10% base slope
            optimalUtilization: 0.80,  // 80% kink point
            excessSlope: 5.0,          // 500% excess slope
        }
    )(tx);

    // 2. Create the pool
    traderClient.client.deepbook.marginMaintainer.createMarginPool(coinKey, poolConfig)(tx);

    // 3. Enable SUI/USDC pool to borrow from this USDC pool
    traderClient.client.deepbook.marginMaintainer.enableDeepbookPoolForLoan(
        'SUI_DBUSDC', 
        coinKey, 
        marginPoolCapId
    )(tx);

    await traderClient.signAndExecute(tx);
}
```

### Example: Update Interest Parameters & Withdraw Fees
```typescript
import { Transaction } from '@mysten/sui/transactions';

async function updateRatesAndClaim(marginPoolCapId: string) {
    const tx = new Transaction();
    const coinKey = 'USDC';

    // Update pool interest rates
    traderClient.client.deepbook.marginMaintainer.updateInterestParams(
        coinKey, 
        marginPoolCapId, 
        {
            baseRate: 0.03, // Raise base rate to 3%
            baseSlope: 0.12,
            optimalUtilization: 0.75, // Lower kink to 75%
            excessSlope: 5.0,
        }
    )(tx);

    // Withdraw maintainer fees
    traderClient.client.deepbook.marginMaintainer.withdrawMaintainerFees(
        coinKey, 
        marginPoolCapId
    )(tx);

    await traderClient.signAndExecute(tx);
}
```
