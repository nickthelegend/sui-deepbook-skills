---
name: deepbookv3-orders
description: Place, modify, cancel, and manage limit and market orders in DeepBook V3.
---

# DeepBook V3: Orders

DeepBook V3 provides a high-performance order execution system. Users can place, modify, and cancel limit or market orders. Orders execute against the central limit order book (CLOB) using price-time priority.

---

## 1. Key Concepts

### Order Options
When submitting an order, developers can specify order options:
1. **GTC (Good 'Til Cancelled) / No Restriction**: The order remains active in the book until it is fully filled or manually cancelled.
2. **IOC (Immediate Or Cancel)**: Any portion of the order that cannot be matched immediately is cancelled.
3. **FOK (Fill Or Kill)**: The order must be filled immediately in its entirety, or it is cancelled completely.
4. **Post Only**: The order is only placed if it behaves as a maker order (adds liquidity). If it would match immediately as a taker order, it is cancelled.

### Self-Matching Options
To prevent wash trading or unintentional execution against one's own orders, DeepBook V3 supports self-matching options:
1. **Self-Matching Allowed**: Buy and sell orders from the same balance manager can match.
2. **Cancel Taker**: If a match is detected, the incoming taker order is cancelled.
3. **Cancel Maker**: If a match is detected, the resting maker order in the book is cancelled.

### Fee Discount
- **`pay_with_deep = true`**: Trading fees are paid in DEEP tokens, which grants a **20% fee discount** compared to using input tokens.
- **`pay_with_deep = false`**: Fees are paid in the input asset token at the standard rate.

---

## 2. On-Chain API (Move)

The `pool` module exposes the following endpoints for managing orders:

### Order Creation
- `public fun place_limit_order<BaseAsset, QuoteAsset>(pool: &mut Pool<BaseAsset, QuoteAsset>, balance_manager: &mut BalanceManager, trade_proof: &TradeProof, client_order_id: u64, price: u64, quantity: u64, is_bid: bool, pay_with_deep: bool, expiration_ms: u64, order_option: u8, self_matching_option: u8, ctx: &mut TxContext): OrderInfo`
  Places a limit order. The quantity is specified in terms of the base asset. Returns an `OrderInfo` object containing matching details.
- `public fun place_market_order<BaseAsset, QuoteAsset>(pool: &mut Pool<BaseAsset, QuoteAsset>, balance_manager: &mut BalanceManager, trade_proof: &TradeProof, client_order_id: u64, quantity: u64, is_bid: bool, pay_with_deep: bool, self_matching_option: u8, ctx: &mut TxContext): OrderInfo`
  Places a market order by internally calling `place_limit_order` with `MAX_PRICE` (for bids) or `MIN_PRICE` (for asks) and immediately cancelling any remaining unfilled quantity.

### Order Modification
- `public fun modify_order<BaseAsset, QuoteAsset>(pool: &mut Pool<BaseAsset, QuoteAsset>, balance_manager: &mut BalanceManager, trade_proof: &TradeProof, order_id: u128, new_quantity: u64, new_expiration_ms: u64, ctx: &mut TxContext)`
  Modifies a resting limit order.
  > [!IMPORTANT]
  > Users can only **reduce** the quantity or **decrease** the expiration time. You cannot increase either parameter; to increase quantity or expiration, you must cancel and place a new order.

### Order Cancellation
- `public fun cancel_order<BaseAsset, QuoteAsset>(pool: &mut Pool<BaseAsset, QuoteAsset>, balance_manager: &mut BalanceManager, trade_proof: &TradeProof, order_id: u128, ctx: &mut TxContext)`
  Cancels a specific order. The order must be owned by the calling `BalanceManager`. Remaining funds are unlocked and returned to the manager's balance.
- `public fun cancel_orders<BaseAsset, QuoteAsset>(pool: &mut Pool<BaseAsset, QuoteAsset>, balance_manager: &mut BalanceManager, trade_proof: &TradeProof, order_ids: vector<u128>, ctx: &mut TxContext)`
  Atomic cancellation of multiple orders. If any cancellation fails, the entire transaction fails (no orders are cancelled).
- `public fun cancel_all_orders<BaseAsset, QuoteAsset>(pool: &mut Pool<BaseAsset, QuoteAsset>, balance_manager: &mut BalanceManager, trade_proof: &TradeProof, ctx: &mut TxContext)`
  Cancels all open orders for the given `BalanceManager` in the specified pool.

### Settlement
- `public fun withdraw_settled_amounts<BaseAsset, QuoteAsset>(pool: &mut Pool<BaseAsset, QuoteAsset>, balance_manager: &mut BalanceManager, trade_proof: &TradeProof, ctx: &mut TxContext)`
  Manually triggers a withdrawal of all settled trading funds from the pool to the user's `BalanceManager`. Note: standard order operations withdraw settled amounts automatically.
- `public fun withdraw_settled_amounts_permissionless<BaseAsset, QuoteAsset>(pool: &mut Pool<BaseAsset, QuoteAsset>, balance_manager: &mut BalanceManager, ctx: &mut TxContext)`
  A permissionless version that allows anyone to trigger withdrawal of settled funds to a `BalanceManager` without requiring a `TradeProof`.

---

## 3. Data Structures

- **`OrderInfo`**: Returned when placing limit or market orders. Contains temporary execution details (e.g. fills, prices, quantities). Dropped automatically at the end of transaction block execution.
- **`OrderDeepPrice`**: Represents the on-chain conversion rate of DEEP tokens to the reference asset (SUI or USDC) at the time of order execution, used for fee calculations.
- **`Fill`**: Represents the exact matched quantity and price details for a maker-taker trade execution.

---

## 4. TS SDK Usage Examples

Using `@mysten/deepbook-v3` to manage orders:

### Placing a Limit Order

```typescript
import { Transaction } from '@mysten/sui/transactions';
import { DeepBookClient } from '@mysten/deepbook-v3';

const client = new DeepBookClient(suiClient);

async function createLimitOrder() {
    const tx = new Transaction();
    
    client.deepbook.placeLimitOrder(tx, {
        poolKey: 'SUI_DBUSDC',
        balanceManagerKey: '0xYOUR_BM_OBJECT_ID',
        clientOrderId: '10001',
        price: 1_250_000_000,                  // Scaled price (taking into account decimals)
        quantity: 10_000_000_000,              // Scaled quantity in base asset (10 SUI)
        isBid: true,                           // true to Buy, false to Sell
        payWithDeep: true,                     // Pay with DEEP for a 20% discount
        expiration: Date.now() + 86400000,     // Expire in 24 hours (ms)
        orderOption: 0,                        // 0 = GTC, 1 = IOC, 2 = FOK, 3 = Post Only
        selfMatchingOption: 1,                 // 1 = Cancel Taker
    });
    
    await suiClient.signAndExecuteTransaction({
        transaction: tx,
        signer: keypair
    });
}
```

### Modifying and Cancelling Orders

```typescript
async function cancelAndModify(orderIdToCancel: string, orderIdToModify: string) {
    const tx = new Transaction();
    
    // 1. Cancel order
    client.deepbook.cancelOrder(tx, {
        poolKey: 'SUI_DBUSDC',
        balanceManagerKey: '0xYOUR_BM_OBJECT_ID',
        orderId: orderIdToCancel
    });
    
    // 2. Modify order (reduce size)
    client.deepbook.modifyOrder(tx, {
        poolKey: 'SUI_DBUSDC',
        balanceManagerKey: '0xYOUR_BM_OBJECT_ID',
        orderId: orderIdToModify,
        newQuantity: 5_000_000_000,           // Must be smaller than current quantity
        newExpiration: Date.now() + 3600000    // Must be smaller than original expiration
    });
    
    await suiClient.signAndExecuteTransaction({
        transaction: tx,
        signer: keypair
    });
}
```

---

## 5. Events Emitted

- **`OrderPlaced`**: Emitted when a limit order is injected into the order book.
- **`OrderFilled`**: Emitted when a resting maker order is matched and filled (partially or fully).
- **`OrderModified`**: Emitted when a resting order's size or expiration is successfully modified.
- **`OrderCanceled`**: Emitted when an order is removed from the book.
