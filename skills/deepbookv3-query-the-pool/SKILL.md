---
name: deepbookv3-query-the-pool
description: Query order book depth, execute dry-run swaps, validate pre-trade parameters, and view balances in DeepBook V3.
---

# DeepBook V3: Query the Pool

DeepBook V3 exposes rich read-only query APIs. These functions allow developers and trading bots to fetch order book depth, simulate trades (dry runs), query user balances, fetch pool configurations, and check pre-trade parameters.

---

## 1. Primary Query APIs

### Order Book Depth (Level 2)
- **Range-Based Depth**: Returns price and quantity vectors within a specified price range.
  `getLevel2Range(pool, price_low, price_high, is_bid)`
- **Tick-Based Depth**: Returns bid/ask prices and quantities up to a specified number of ticks.
  `getLevel2Ticks(pool, ticks)`

### Swap Simulations (Dry Runs)
- **Quote for Base (Exact Base)**: Get simulated quote output and DEEP fee for a given base asset input.
- **Base for Quote (Exact Quote)**: Get simulated base output and DEEP fee for a given quote asset input.
- **Generic Dry Run**: Simulates swap exact amount and returns `(base_out, quote_out, deep_fee_required)`.
- **Reverse Dry Run**: Simulates the exact input required to obtain a specific target output.

### Balance & Fee Queries
- **Locked Balances**: Retrieve the amount of base, quote, and DEEP tokens locked in open orders for a specific `BalanceManager`.
- **Fee Requirements**: Retrieve estimated taker and maker fees in DEEP for a potential order quantity and price.

### Account & Pool Parameters
- **Pool Parameters**: Returns current `(taker_fee, maker_fee, stake_required)`.
- **Next Epoch Proposal**: Returns trade parameters of the leading proposal for the next epoch.
- **Book Parameters**: Returns `(tick_size, lot_size, min_size)`.
- **Pre-trade Validation**: Checks if an order can be placed with current params, returning a boolean or a detailed error code vector.

---

## 2. On-Chain API (Move)

The `pool` module exposes the following getter functions:

```move
// Check if the pool is whitelisted
public fun is_whitelisted<BaseAsset, QuoteAsset>(pool: &Pool<BaseAsset, QuoteAsset>): bool

// Get mid price
public fun mid_price<BaseAsset, QuoteAsset>(pool: &Pool<BaseAsset, QuoteAsset>): u64

// Get open order IDs for a balance manager
public fun open_orders_ids<BaseAsset, QuoteAsset>(pool: &Pool<BaseAsset, QuoteAsset>, balance_manager: &BalanceManager): vector<u128>

// Get detailed Order struct from ID
public fun order<BaseAsset, QuoteAsset>(pool: &Pool<BaseAsset, QuoteAsset>, order_id: u128): Order

// Get locked balances for a balance manager
public fun locked_balance<BaseAsset, QuoteAsset>(pool: &Pool<BaseAsset, QuoteAsset>, balance_manager: &BalanceManager): (u64, u64, u64)

// Get book parameters
public fun book_params<BaseAsset, QuoteAsset>(pool: &Pool<BaseAsset, QuoteAsset>): (u64, u64, u64)

// Pre-trade validation for limit orders
public fun pre_trade_validate_limit<BaseAsset, QuoteAsset>(pool: &Pool<BaseAsset, QuoteAsset>, balance_manager: &BalanceManager, price: u64, quantity: u64, is_bid: bool): (bool, vector<u64>)
```

---

## 3. TS SDK Usage Examples

Using `@mysten/deepbook-v3` to query pools and order books:

### Querying Order Book Depth

```typescript
import { DeepBookClient } from '@mysten/deepbook-v3';

const client = new DeepBookClient(suiClient);

async function getMarketDepth() {
    const poolKey = 'SUI_DBUSDC';
    
    // Fetch bids and asks up to 20 ticks from mid-price
    const depth = await client.deepbook.getLevel2Ticks(poolKey, 20);
    
    console.log("Bids:", depth.bidPrice, depth.bidQuantity);
    console.log("Asks:", depth.askPrice, depth.askQuantity);
}
```

### Dry Run / Simulating a Swap

```typescript
async function simulateSwap(amountIn: bigint) {
    const poolKey = 'SUI_DBUSDC';
    
    // Simulate swapping base (SUI) for quote (USDC)
    const simulation = await client.deepbook.getAmountOut({
        poolKey,
        amountIn,
        isBase: true, // Swapping Base (SUI) -> Quote (USDC)
    });
    
    console.log(`Expected output: ${simulation.amountOut}`);
    console.log(`Required DEEP Fee: ${simulation.deepFee}`);
}
```

### Checking Locked Balances

```typescript
async function checkBalances(bmId: string) {
    const poolKey = 'SUI_DBUSDC';
    
    const locked = await client.deepbook.getLockedBalance({
        poolKey,
        balanceManagerKey: bmId,
    });
    
    console.log(`Locked Base: ${locked.baseAmount}`);
    console.log(`Locked Quote: ${locked.quoteAmount}`);
    console.log(`Locked DEEP: ${locked.deepAmount}`);
}
```
