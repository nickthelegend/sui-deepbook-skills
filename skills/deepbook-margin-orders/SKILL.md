---
name: deepbook-margin-orders
description: Place, modify, and cancel limit and market orders, reduce debt using reduce-only orders, and handle staking/governance through the MarginManager using the pool_proxy module.
---

# DeepBook Margin: Trading & Orders

The `pool_proxy` module provides wrapper functions that allow a `MarginManager` to place, modify, and cancel orders on DeepBook V3 trading pools. It also exposes interfaces for staking, governance, and rebate collections.

---

## 1. Move Smart Contract API

All actions require the `MarginManager` to be associated with the target DeepBook trading pool.

### Order Placement
- `public fun place_limit_order<BaseAsset, QuoteAsset>(manager: &mut MarginManager, pool: &mut Pool<BaseAsset, QuoteAsset>, price: u64, quantity: u64, is_bid: bool, expire_timestamp: u64, restriction: u8, self_matching_option: u8, clock: &Clock, ctx: &mut TxContext)`
  Places a limit order. Active borrow positions are verified against risk limits dynamically.
- `public fun place_market_order<BaseAsset, QuoteAsset>(manager: &mut MarginManager, pool: &mut Pool<BaseAsset, QuoteAsset>, quantity: u64, is_bid: bool, self_matching_option: u8, clock: &Clock, ctx: &mut TxContext)`
  Places a market order.

### Reduce-Only Orders
- `public fun place_reduce_only_limit_order<BaseAsset, QuoteAsset>(manager: &mut MarginManager, pool: &mut Pool<BaseAsset, QuoteAsset>, price: u64, quantity: u64, is_bid: bool, expire_timestamp: u64, self_matching_option: u8, clock: &Clock, ctx: &mut TxContext)`
  Places a limit order that can **only** reduce an active debt position. Extremely useful if margin trading is temporarily disabled for a pool, letting traders close out positions.
- `public fun place_reduce_only_market_order<BaseAsset, QuoteAsset>(manager: &mut MarginManager, pool: &mut Pool<BaseAsset, QuoteAsset>, quantity: u64, is_bid: bool, self_matching_option: u8, clock: &Clock, ctx: &mut TxContext)`
  Places a reduce-only market order.

### Order Modifying & Cancellations
- `public fun modify_order<BaseAsset, QuoteAsset>(manager: &mut MarginManager, pool: &mut Pool<BaseAsset, QuoteAsset>, order_id: u128, new_quantity: u64, clock: &Clock, ctx: &mut TxContext)`
  Modifies the quantity of an active order.
- `public fun cancel_order<BaseAsset, QuoteAsset>(manager: &mut MarginManager, pool: &mut Pool<BaseAsset, QuoteAsset>, order_id: u128, ctx: &mut TxContext)`
  Cancels a single open order.
- `public fun cancel_orders<BaseAsset, QuoteAsset>(manager: &mut MarginManager, pool: &mut Pool<BaseAsset, QuoteAsset>, order_ids: vector<u128>, ctx: &mut TxContext)`
  Cancels a list of open orders.
- `public fun cancel_all_orders<BaseAsset, QuoteAsset>(manager: &mut MarginManager, pool: &mut Pool<BaseAsset, QuoteAsset>, ctx: &mut TxContext)`
  Cancels all open orders placed by the margin manager on the pool.

### Settlements & Rebates
- `public fun withdraw_settled_amounts<BaseAsset, QuoteAsset>(manager: &mut MarginManager, pool: &mut Pool<BaseAsset, QuoteAsset>, ctx: &mut TxContext)`
  Withdraws assets from settled trades back to the internal `BalanceManager` of the margin manager.
- `public fun claim_rebates<BaseAsset, QuoteAsset>(manager: &mut MarginManager, pool: &mut Pool<BaseAsset, QuoteAsset>, ctx: &mut TxContext)`
  Claims accumulated trading fee rebates.

---

## 2. Staking & Governance

Traders can participate in staking and governance through the `MarginManager` (inheriting traits from DeepBook V3).

### DEEP Staking
- `public fun stake<BaseAsset, QuoteAsset>(manager: &mut MarginManager, pool: &mut Pool<BaseAsset, QuoteAsset>, amount: u64, ctx: &mut TxContext)`
  Stakes DEEP tokens to receive voting power and trading fee rebates.
  - *Restriction*: Margin pools having DEEP as either the Base or Quote asset cannot stake DEEP.
- `public fun unstake<BaseAsset, QuoteAsset>(manager: &mut MarginManager, pool: &mut Pool<BaseAsset, QuoteAsset>, ctx: &mut TxContext)`
  Unstakes DEEP tokens.

### Governance Voting
- `public fun submit_proposal<BaseAsset, QuoteAsset>(manager: &mut MarginManager, pool: &mut Pool<BaseAsset, QuoteAsset>, target_pool_id: ID, value: u64, ctx: &mut TxContext)`
  Submits a proposal using staked voting weights.
- `public fun vote<BaseAsset, QuoteAsset>(manager: &mut MarginManager, pool: &mut Pool<BaseAsset, QuoteAsset>, target_pool_id: ID, vote_fee: u64, ctx: &mut TxContext)`
  Votes on active pool proposals.
