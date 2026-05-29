---
name: deepbookv3-swaps
description: Execute AMM-style immediate swaps with or without a BalanceManager in DeepBook V3.
---

# DeepBook V3: Swaps

DeepBook V3 offers an AMM-style swap interface. This is highly suitable for decentralized exchanges, routers, or applications that need instant execution without locking assets in a `BalanceManager` or managing order cancellations.

---

## 1. Key Concepts

### Two Operational Modes
Swaps can be executed in two ways depending on system architecture:
1. **Without a `BalanceManager`**: Perfect for user-facing dApps. Traders swap raw `Coin` objects directly. The function returns the bought asset coin, any leftover input asset coin, and any remaining/unused fee tokens.
2. **With a `BalanceManager`**: Leverages the user's on-chain balance manager. Fees are paid using the manager's DEEP balance.

### Mechanics & Constraints
- **Direction**: Swapping from base to quote requires `base_in > 0` and `quote_in = 0`. Swapping from quote to base requires `quote_in > 0` and `base_in = 0`.
- **Trading Fees**: Swapping requires DEEP tokens to pay for trading fees.
- **Overestimating Fees**: For swaps without a balance manager, you can safely overestimate the DEEP fee coin input. DeepBook V3 calculates the exact fee, takes what is needed, and returns the rest.
- **Divisibility**: If the input quantity is not perfectly divisible by the pool's lot size, some input token quantity might remain unswapped and be returned.

---

## 2. On-Chain API (Move)

The `pool` module exposes these endpoints for swaps:

### Swapping Without BalanceManager
- `public fun swap_exact_base_for_quote<BaseAsset, QuoteAsset>(pool: &mut Pool<BaseAsset, QuoteAsset>, base_in: Coin<BaseAsset>, quote_in: Coin<QuoteAsset>, deep_in: Coin<DEEP>, min_out: u64, ctx: &mut TxContext): (Coin<BaseAsset>, Coin<QuoteAsset>, Coin<DEEP>)`
  Swaps base coins for quote coins. Unused base and DEEP coins are returned alongside the received quote coin.
- `public fun swap_exact_quote_for_base<BaseAsset, QuoteAsset>(pool: &mut Pool<BaseAsset, QuoteAsset>, base_in: Coin<BaseAsset>, quote_in: Coin<QuoteAsset>, deep_in: Coin<DEEP>, min_out: u64, ctx: &mut TxContext): (Coin<BaseAsset>, Coin<QuoteAsset>, Coin<DEEP>)`
  Swaps quote coins for base coins. Unused quote and DEEP coins are returned alongside the received base coin.
- `public fun swap_exact_quantity<BaseAsset, QuoteAsset>(pool: &mut Pool<BaseAsset, QuoteAsset>, base_in: Coin<BaseAsset>, quote_in: Coin<QuoteAsset>, deep_in: Coin<DEEP>, min_out: u64, ctx: &mut TxContext): (Coin<BaseAsset>, Coin<QuoteAsset>, Coin<DEEP>)`
  The underlying generic function called by both helpers above. One of `base_in` or `quote_in` must be a zero coin.

### Swapping With BalanceManager
- `public fun swap_exact_base_for_quote_with_manager<BaseAsset, QuoteAsset>(pool: &mut Pool<BaseAsset, QuoteAsset>, base_in: Coin<BaseAsset>, min_out: u64, balance_manager: &mut BalanceManager, trade_proof: &TradeProof, ctx: &mut TxContext): (Coin<BaseAsset>, Coin<QuoteAsset>)`
  Swaps base for quote. Assumes fees are paid using DEEP available in the `BalanceManager`. Returns the received quote and any leftover base coin.
- `public fun swap_exact_quote_for_base_with_manager<BaseAsset, QuoteAsset>(pool: &mut Pool<BaseAsset, QuoteAsset>, quote_in: Coin<QuoteAsset>, min_out: u64, balance_manager: &mut BalanceManager, trade_proof: &TradeProof, ctx: &mut TxContext): (Coin<BaseAsset>, Coin<QuoteAsset>)`
  Swaps quote for base. Assumes fees are paid using DEEP available in the `BalanceManager`. Returns the received base and any leftover quote coin.
- `public fun swap_exact_quantity_with_manager<BaseAsset, QuoteAsset>(pool: &mut Pool<BaseAsset, QuoteAsset>, base_in: Coin<BaseAsset>, quote_in: Coin<QuoteAsset>, min_out: u64, balance_manager: &mut BalanceManager, trade_proof: &TradeProof, ctx: &mut TxContext): (Coin<BaseAsset>, Coin<QuoteAsset>)`
  The underlying generic function for manager swaps. One of `base_in` or `quote_in` must be zero.

---

## 3. TS SDK Usage Examples

Using `@mysten/deepbook-v3` to execute swaps:

### Direct Swap (Without BalanceManager)

```typescript
import { Transaction } from '@mysten/sui/transactions';
import { DeepBookClient } from '@mysten/deepbook-v3';

const client = new DeepBookClient(suiClient);

async function swapDirect(amountIn: bigint, minAmountOut: bigint) {
    const tx = new Transaction();
    
    // Split the input coin to swap (e.g. SUI) from gas
    const [inputCoin] = tx.splitCoins(tx.gas, [amountIn]);
    
    // Split DEEP token to cover fees (safely overestimate)
    const [deepFeeCoin] = tx.splitCoins(tx.object('0xDEEP_COIN_OBJECT_ID'), [100_000_000n]);
    
    // Execute swap
    const [baseOut, quoteOut, deepOut] = client.deepbook.swap(tx, {
        poolKey: 'SUI_DBUSDC',
        inputCoin,
        deepFeeCoin,
        minOut: minAmountOut,
        payWithDeep: true,
    });
    
    // Transfer returned coins to the user
    tx.transferObjects([baseOut, quoteOut, deepOut], keypair.toSuiAddress());
    
    await suiClient.signAndExecuteTransaction({
        transaction: tx,
        signer: keypair
    });
}
```

### Swap With BalanceManager

```typescript
async function swapWithManager(amountIn: bigint, minAmountOut: bigint) {
    const tx = new Transaction();
    
    const [inputCoin] = tx.splitCoins(tx.gas, [amountIn]);
    
    // Execute swap using balance manager for fees
    const [baseOut, quoteOut] = client.deepbook.swapWithManager(tx, {
        poolKey: 'SUI_DBUSDC',
        balanceManagerKey: '0xYOUR_BM_OBJECT_ID',
        inputCoin,
        minOut: minAmountOut,
    });
    
    tx.transferObjects([baseOut, quoteOut], keypair.toSuiAddress());
    
    await suiClient.signAndExecuteTransaction({
        transaction: tx,
        signer: keypair
    });
}
```
