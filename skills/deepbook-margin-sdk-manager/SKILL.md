---
name: deepbook-margin-sdk-manager
description: Manage margin accounts, deposit collateral, borrow assets, repay debt, and liquidate undercollateralized accounts using the TypeScript SDK.
---

# DeepBook Margin SDK: Margin Manager

The Margin Manager SDK exposes functions to create accounts, deposit/withdraw collateral, borrow assets, repay debt, and monitor risk metrics.

---

## 1. SDK API Reference (`client.deepbook.marginManager`)

### Account Management & Initialization
- `newMarginManager(poolKey: string)`
  Creates and shares a new margin manager.
  - *Returns*: `(tx: Transaction) => void`
- `newMarginManagerWithInitializer(poolKey: string)`
  Creates a new margin manager along with an initializer. Must be shared afterward.
  - *Returns*: `{ manager: TransactionArgument, initializer: TransactionArgument }`
- `shareMarginManager(poolKey: string, manager: TransactionArgument, initializer: TransactionArgument)`
  Shares the initializer-based margin manager.
  - *Returns*: `(tx: Transaction) => void`
- `depositDuringInitialization(manager: TransactionArgument, poolKey: string, coinType: string, params: { amount?: number; coin?: TransactionArgument })`
  Deposits assets into a margin manager in the same PTB prior to calling `shareMarginManager`. Specify *either* `amount` or `coin`.

### Collateral Deposits & Withdrawals
- `depositBase(managerKey: string, amount: number)` / `depositQuote(managerKey: string, amount: number)` / `depositDeep(managerKey: string, amount: number)`
  Deposits base, quote, or DEEP collateral into a margin manager.
- `withdrawBase(managerKey: string, amount: number)` / `withdrawQuote(managerKey: string, amount: number)` / `withdrawDeep(managerKey: string, amount: number)`
  Withdraws assets from the manager. Subject to the Minimum Withdraw Risk Ratio if there is active debt.

### Borrow & Repay Loans
- `borrowBase(managerKey: string, amount: number)` / `borrowQuote(managerKey: string, amount: number)`
  Borrows assets from the corresponding margin pool. Verified against the Minimum Borrow Risk Ratio.
- `repayBase(managerKey: string, amount?: number)` / `repayQuote(managerKey: string, amount?: number)`
  Repays borrowed debt. If no `amount` is specified, the maximum available balance up to the total debt will be repaid.

### Referral Setup
- `setMarginManagerReferral(managerKey: string, referral: string)`
  Links a minted `DeepBookPoolReferral` to the margin manager.
- `unsetMarginManagerReferral(managerKey: string, poolKey: string)`
  Removes a linked referral.

### Liquidation
- `liquidate(managerAddress: string, poolKey: string, debtIsBase: boolean, repayCoin: TransactionArgument)`
  Liquidates an undercollateralized margin manager.
  - *Returns*: `(tx: Transaction) => void`

---

## 2. Read-Only Query Functions

These query the on-chain state without modifying it:
- `managerState(managerAddress: string)`: Returns comprehensive status including risk ratio, assets, debts, and Pyth prices.
- `owner(managerAddress: string)` / `deepbookPool(managerAddress: string)` / `marginPoolId(managerAddress: string)`
- `baseBalance(managerAddress: string)` / `quoteBalance(managerAddress: string)` / `deepBalance(managerAddress: string)`
- `borrowedShares(managerAddress: string)` / `borrowedBaseShares(managerAddress: string)` / `borrowedQuoteShares(managerAddress: string)` / `hasBaseDebt(managerAddress: string)`
- `balanceManager(managerAddress: string)` / `calculateAssets(managerAddress: string)` / `calculateDebts(managerAddress: string)`

---

## 3. Transaction Block Examples

### Example: Create, Initialize, and Borrow USDC
```typescript
import { Transaction } from '@mysten/sui/transactions';

async function setupAndBorrow() {
    const tx = new Transaction();
    const managerKey = 'MARGIN_MANAGER_1';

    // 1. Deposit 100 SUI as base collateral
    tx.add(traderClient.client.deepbook.marginManager.depositBase(managerKey, 100));

    // 2. Borrow 500 USDC from the quote margin pool
    tx.add(traderClient.client.deepbook.marginManager.borrowQuote(managerKey, 500));

    const result = await traderClient.signAndExecute(tx);
    console.log('Transaction effects:', result);
}
```

### Example: Execute Liquidation
```typescript
import { Transaction } from '@mysten/sui/transactions';

async function performLiquidation(undercollateralizedManager: string, debtAmount: number) {
    const tx = new Transaction();
    
    // Split the repayment coin from the gas coin (e.g. 500 USDC)
    const [repayCoin] = tx.splitCoins(tx.gas, [tx.pure.u64(debtAmount)]);

    tx.add(
        traderClient.client.deepbook.marginManager.liquidate(
            undercollateralizedManager, 
            'SUI_DBUSDC', 
            false, // Debt is quote asset (USDC)
            repayCoin
        )
    );

    await traderClient.signAndExecute(tx);
}
```
