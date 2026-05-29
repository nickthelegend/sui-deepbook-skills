---
name: deepbookv3-sdk-swaps
description: Execute instant AMM-style swaps with or without a BalanceManager using the DeepBook V3 TypeScript SDK.
---

# DeepBook V3: Swaps SDK

The `@mysten/deepbook-v3` TypeScript SDK provides simple wrapper methods to perform AMM-like swaps directly. Swaps can be executed with raw coin objects (direct swaps) or via a registered `BalanceManager`.

---

## 1. Parameters & Configuration

### `SwapParams`
Used for swaps **without** a `BalanceManager`. The SDK automatically fetches coins from the user's address unless explicit coin arguments are provided.
```typescript
interface SwapParams {
    poolKey: string;             // Pool name identifier (e.g. 'SUI_DBUSDC')
    amount: number;              // Input token quantity in standard decimals (e.g., 1.5 SUI)
    deepAmount: number;          // DEEP token quantity for fees (excess is returned)
    minOut: number;              // Slippage protection: minimum output expected
    baseCoin?: TransactionArgument;  // Optional base coin object reference
    quoteCoin?: TransactionArgument; // Optional quote coin object reference
    deepCoin?: TransactionArgument;  // Optional DEEP fee coin object reference
}
```

### `SwapWithManagerParams`
Used for swaps **with** a `BalanceManager`. Fees are paid automatically using the DEEP balance in the manager.
```typescript
interface SwapWithManagerParams {
    poolKey: string;
    balanceManagerKey: string;   // Registered manager key
    amount: number;              // Input token quantity in standard decimals
    minOut: number;              // Slippage protection: minimum output expected
    tradeCap?: TransactionArgument;    // Optional trading capability
    depositCap?: TransactionArgument;  // Optional deposit capability
    withdrawCap?: TransactionArgument; // Optional withdrawal capability
    baseCoin?: TransactionArgument;    // Optional base coin object reference
    quoteCoin?: TransactionArgument;   // Optional quote coin object reference
}
```

---

## 2. Primary SDK Methods

All swap methods return a builder function that takes a `Transaction` object as an argument:

### Swapping Without BalanceManager
- `swapExactBaseForQuote({ params: SwapParams })(tx)`
  Swaps exact base assets for quote assets. Returns `[baseOut, quoteOut, deepOut]` coin references inside the PTB.
- `swapExactQuoteForBase({ params: SwapParams })(tx)`
  Swaps exact quote assets for base assets. Returns `[baseOut, quoteOut, deepOut]` coin references inside the PTB.
- `swapExactQuantity(params & { isBaseToCoin: boolean })(tx)`
  Generic swap function. `isBaseToCoin` determines direction (true = base to quote).

### Swapping With BalanceManager
- `swapExactQuantityWithManager(params & { isBaseToCoin: boolean })(tx)`
  Performs swap using a balance manager. Returns `[baseOut, quoteOut]` coin references.

---

## 3. Examples

### Direct Swap (Without BalanceManager)

```typescript
import { Transaction } from '@mysten/sui/transactions';
import { DeepBookClient } from '@mysten/deepbook-v3';

const client = new DeepBookClient(suiClient);

async function directBaseToQuoteSwap() {
    const tx = new Transaction();
    const userAddress = keypair.toSuiAddress();

    // Swap 1 SUI for USDC. Overestimate DEEP fee to 1 DEEP
    const [baseOut, quoteOut, deepOut] = client.deepbook.swapExactBaseForQuote({
        poolKey: 'SUI_DBUSDC',
        amount: 1,
        deepAmount: 1,
        minOut: 0.95,
    })(tx);

    // Transfer output coins to user address
    tx.transferObjects([baseOut, quoteOut, deepOut], userAddress);

    await suiClient.signAndExecuteTransaction({
        transaction: tx,
        signer: keypair
    });
}
```

### Swap Using a BalanceManager

```typescript
async function managerBaseToQuoteSwap() {
    const tx = new Transaction();
    const userAddress = keypair.toSuiAddress();

    // Swap 2 SUI for USDC using balance manager MANAGER_1
    const [baseOut, quoteOut] = client.deepbook.swapExactQuantityWithManager({
        poolKey: 'SUI_DBUSDC',
        balanceManagerKey: 'MANAGER_1',
        amount: 2.0,
        minOut: 1.9,
        isBaseToCoin: true, // true to swap Base (SUI) -> Quote (USDC)
    })(tx);

    tx.transferObjects([baseOut, quoteOut], userAddress);

    await suiClient.signAndExecuteTransaction({
        transaction: tx,
        signer: keypair
    });
}
```
