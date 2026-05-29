---
name: deepbookv3-sdk-flash-loans
description: Borrow and repay base or quote assets atomically inside a Programmable Transaction Block (PTB) using the TS SDK.
---

# DeepBook V3: Flash Loans SDK

The `@mysten/deepbook-v3` SDK allows developers to borrow base or quote tokens from a pool's vault without collateral, perform arbitrate operations, and repay the loan in a single atomic Programmable Transaction Block (PTB).

---

## 1. Primary SDK Methods

All flash loan methods return a transaction-building function that appends calls to a `Transaction` object:

### Borrowing Methods
- `borrowBaseAsset(poolKey, borrowAmount)(tx)`
  Borrows base asset tokens. Returns a tuple: `[baseCoin, flashLoanReceipt]` transaction arguments.
- `borrowQuoteAsset(poolKey, borrowAmount)(tx)`
  Borrows quote asset tokens. Returns a tuple: `[quoteCoin, flashLoanReceipt]` transaction arguments.

### Repayment Methods
- `returnBaseAsset({ poolKey, borrowAmount, baseCoinInput, flashLoan })(tx)`
  Repays the borrowed base assets and settles the hot-potato receipt. Returns the residual coin argument (unused coins).
- `returnQuoteAsset({ poolKey, borrowAmount, quoteCoinInput, flashLoan })(tx)`
  Repays the borrowed quote assets and settles the hot-potato receipt. Returns the residual coin argument.

---

## 2. Example: Multi-Step Arbitrage PTB

The following example demonstrates how to borrow 1 DEEP from the `DEEP_SUI` pool, execute intermediate swaps, and return the assets to settle the flash loan.

```typescript
import { Transaction } from '@mysten/sui/transactions';
import { DeepBookClient } from '@mysten/deepbook-v3';

const client = new DeepBookClient(suiClient);

async function executeFlashLoanArbitrage() {
    const tx = new Transaction();
    const borrowAmount = 1; // 1 DEEP (SDK handles decimals internally)
    const userAddress = keypair.toSuiAddress();

    // 1. Borrow 1 DEEP from the DEEP_SUI pool
    const [deepCoin, flashLoanReceipt] = tx.add(
        client.deepbook.flashLoans.borrowBaseAsset('DEEP_SUI', borrowAmount)
    );

    // 2. Perform a swap using the borrowed DEEP as gas/fee coin
    const [baseOut, quoteOut, deepOut] = tx.add(
        client.deepbook.swapExactQuoteForBase({
            poolKey: 'SUI_DBUSDC',
            amount: 0.5,
            deepAmount: 1, // Using the borrowed DEEP coin
            minOut: 0,
            deepCoin: deepCoin, // Provide the borrowed DEEP object reference
        })
    );
    
    // Transfer first swap outputs to user address
    tx.transferObjects([baseOut, quoteOut, deepOut], userAddress);

    // 3. Execute a second swap to get back DEEP for repayment
    const [baseOut2, quoteOut2, deepOut2] = tx.add(
        client.deepbook.swapExactQuoteForBase({
            poolKey: 'DEEP_SUI',
            amount: 10,
            deepAmount: 0,
            minOut: 0,
        })
    );
    
    tx.transferObjects([quoteOut2, deepOut2], userAddress);

    // 4. Return the borrowed DEEP to settle the loan
    // baseOut2 represents the returned DEEP coin asset
    const loanRemain = tx.add(
        client.deepbook.flashLoans.returnBaseAsset({
            poolKey: 'DEEP_SUI',
            borrowAmount,
            baseCoinInput: baseOut2,
            flashLoan: flashLoanReceipt,
        })
    );

    // 5. Send any remaining residual coins to the user
    tx.transferObjects([loanRemain], userAddress);

    await suiClient.signAndExecuteTransaction({
        transaction: tx,
        signer: keypair,
    });
}
```
