---
name: deepbookv3-flash-loans
description: Perform uncollateralized atomic flash loans using the hot-potato receipt pattern in DeepBook V3.
---

# DeepBook V3: Flash Loans

Flash loans are uncollateralized loans that must be borrowed and fully repaid within the same Programmable Transaction Block (PTB). Users can borrow the base asset or the quote asset from any DeepBook V3 pool.

---

## 1. Key Concepts

### Hot-Potato Pattern
- DeepBook V3 utilizes Move's unique safety guarantees via a "hot-potato" struct.
- The `FlashLoan` struct represents the receipt. It has **no abilities** (no `drop`, `store`, `key`, or `copy`).
- Because a struct without `drop` cannot be discarded or ignored, the Move compiler and virtual machine enforce that the `FlashLoan` object **must** be consumed by the end of the transaction. The only way to consume it is to pass it to the corresponding repayment function.

### Mechanics & Constraints
- **Maximum Borrow Amount**: You can borrow up to the maximum amount of base/quote assets currently held in the pool's vault.
- **Atomicity**: The borrow and repayment occur in the same transaction. If any step fails or the repayment is insufficient, the entire transaction is reverted.
- **Intra-Pool Trading Conflict**: Borrowing from a pool and trading on that same pool within the same PTB can result in transaction failures. Trading requires liquidity movement within the vault; if that liquidity is borrowed, the pool may lack the funds to settle the trade.

---

## 2. On-Chain API (Move)

The `pool` module exposes the following endpoints for flash loans:

### Borrowing
- `public fun borrow_flash_loan_base<BaseAsset, QuoteAsset>(pool: &mut Pool<BaseAsset, QuoteAsset>, amount: u64, ctx: &mut TxContext): (Balance<BaseAsset>, FlashLoan<BaseAsset, QuoteAsset>)`
  Borrows base assets from the pool's vault. Returns the borrowed base asset balance and the `FlashLoan` receipt.
- `public fun borrow_flash_loan_quote<BaseAsset, QuoteAsset>(pool: &mut Pool<BaseAsset, QuoteAsset>, amount: u64, ctx: &mut TxContext): (Balance<QuoteAsset>, FlashLoan<BaseAsset, QuoteAsset>)`
  Borrows quote assets from the pool's vault. Returns the borrowed quote asset balance and the `FlashLoan` receipt.

### Repayment
- `public fun return_flash_loan_base<BaseAsset, QuoteAsset>(pool: &mut Pool<BaseAsset, QuoteAsset>, balance: Balance<BaseAsset>, receipt: FlashLoan<BaseAsset, QuoteAsset>)`
  Returns the borrowed base assets to the pool, destroying/unwrapping the `FlashLoan` receipt.
- `public fun return_flash_loan_quote<BaseAsset, QuoteAsset>(pool: &mut Pool<BaseAsset, QuoteAsset>, balance: Balance<QuoteAsset>, receipt: FlashLoan<BaseAsset, QuoteAsset>)`
  Returns the borrowed quote assets to the pool, destroying/unwrapping the `FlashLoan` receipt.

---

## 3. TS SDK Usage Examples

Using `@mysten/deepbook-v3` to execute a flash loan:

### Flash Loan Base Asset Example

```typescript
import { Transaction } from '@mysten/sui/transactions';
import { DeepBookClient } from '@mysten/deepbook-v3';

const client = new DeepBookClient(suiClient);

async function executeArbitrage(borrowAmount: bigint) {
    const tx = new Transaction();
    
    // 1. Borrow base asset from SUI_DBUSDC pool
    const [borrowedBase, borrowedQuote, flashLoanReceipt] = client.deepbook.borrowFlashLoan(tx, {
        poolKey: 'SUI_DBUSDC',
        borrowAmount,
        isBase: true, // true to borrow base token (SUI)
    });
    
    // 2. Perform DeFi / Arbitrage actions
    // For example: swap borrowed SUI on an external AMM for USDC, 
    // then trade USDC back to SUI to get a profit.
    // ... custom logic ...

    // 3. Repay the base asset before the transaction ends
    client.deepbook.returnFlashLoan(tx, {
        poolKey: 'SUI_DBUSDC',
        receipt: flashLoanReceipt,
        repaymentCoin: borrowedBase, // Must return the borrowed SUI coin
    });
    
    // Execute the PTB
    await suiClient.signAndExecuteTransaction({
        transaction: tx,
        signer: keypair
    });
}
```
