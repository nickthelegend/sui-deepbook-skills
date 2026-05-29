---
name: deepbook-margin-pool
description: Manage the MarginPool shared object, supply/withdraw liquidity, calculate interest rates, configure parameters, and execute maintainer admin functions.
---

# DeepBook Margin: Margin Pool & Interest Rates

The `MarginPool` is a shared object that manages lending liquidity for a specific asset type. Suppliers deposit assets to earn variable interest compounded dynamically on every state-changing operation.

---

## 1. Interest Rate & Utilization Model

DeepBook Margin implements a piecewise linear ("kinked") interest rate curve:

### Formula
- If **$\text{Utilization} < \text{OptimalUtilization}$**:
  $$\text{BorrowRate} = \text{BaseRate} + \text{Utilization} \times \text{BaseSlope}$$
- If **$\text{Utilization} \ge \text{OptimalUtilization}$**:
  $$\text{BorrowRate} = \text{BaseRate} + \text{OptimalUtilization} \times \text{BaseSlope} + (\text{Utilization} - \text{OptimalUtilization}) \times \text{ExcessSlope}$$

Where:
- **$\text{Utilization}$**: $\frac{\text{Total Borrowed}}{\text{Total Supplied}}$
- **Base Rate**: Minimum APR at 0% utilization.
- **Base Slope**: APR growth rate below optimal utilization.
- **Optimal Utilization**: The curve kink point (typically 80%).
- **Excess Slope**: Steep APR growth rate above optimal utilization (typically 500%).

### Current Pool Parameters (Mainnet)

| Asset | Base Rate | Base Slope | Optimal Utilization | Excess Slope | Max Utilization |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **USDC** | 0% | 15% | 80% | 500% | 90% |
| **SUIUSDE** | 0% | 15% | 80% | 500% | 90% |
| **SUI** | 3% | 20% | 80% | 500% | 90% |
| **DEEP** | 5% | 25% | 80% | 500% | 90% |
| **WAL** | 5% | 25% | 80% | 500% | 90% |

The **Max Utilization** rate is capped at 90%, preserving at least 10% pool reserves for supplier withdrawals.

---

## 2. On-Chain API (Move)

 Lenders and users interact with the pool using the following methods:

### Supplier Cap Minting
- `public fun mint_supplier_cap(ctx: &mut TxContext): SupplierCap`
  Mints a new `SupplierCap` required to supply and withdraw liquidity. A single `SupplierCap` can be used across multiple margin pools.

### Supplying & Withdrawing Liquidity
- `public fun supply<Asset>(pool: &mut MarginPool<Asset>, cap: &SupplierCap, coin: Coin<Asset>, ctx: &mut TxContext): u64`
  Supplies assets to the pool. Returns the supplier's updated supply shares.
- `public fun withdraw<Asset>(pool: &mut MarginPool<Asset>, cap: &SupplierCap, shares: u64, ctx: &mut TxContext): Coin<Asset>`
  Withdraws assets by redeeming supply shares.
- `public fun withdraw_all<Asset>(pool: &mut MarginPool<Asset>, cap: &SupplierCap, ctx: &mut TxContext): Coin<Asset>`
  Withdraws all supplied assets belonging to the `SupplierCap` owner.

---

## 3. Maintainer / Administrative Operations

Maintainer capabilities are governed by the `MarginPoolCap` and administrative capabilities.

### API Methods
- `public fun create_margin_pool<Asset>(registry: &mut MarginRegistry, base_rate: u64, base_slope: u64, optimal_utilization: u64, excess_slope: u64, supply_cap: u64, max_utilization: u64, min_borrow: u64, referral_spread: u64, ctx: &mut TxContext)`
  Creates a new margin pool. Enforces that only one margin pool exists per asset type.
- `public fun enable_deepbook_pool<Asset>(pool: &mut MarginPool<Asset>, deepbook_pool_id: ID, cap: &MarginPoolCap)`
  Enables a specific DeepBook pool to borrow from this margin pool.
- `public fun disable_deepbook_pool<Asset>(pool: &mut MarginPool<Asset>, deepbook_pool_id: ID, cap: &MarginPoolCap)`
  Disables a specific DeepBook pool from borrowing.
- `public fun update_pool_params<Asset>(pool: &mut MarginPool<Asset>, base_rate: u64, base_slope: u64, optimal_utilization: u64, excess_slope: u64, supply_cap: u64, max_utilization: u64, min_borrow: u64, referral_spread: u64, cap: &MarginPoolCap)`
  Updates interest rate curves, borrow sizing limits, and supply configurations.
- `public fun withdraw_maintainer_fees<Asset>(pool: &mut MarginPool<Asset>, cap: &MarginPoolCap, ctx: &mut TxContext): Coin<Asset>`
  Claims accumulated maintainer protocol fees.
- `public fun withdraw_protocol_fees<Asset>(pool: &mut MarginPool<Asset>, cap: &MarginPoolCap, ctx: &mut TxContext): Coin<Asset>`
  Claims protocol treasury fees.

---

## 4. Pool Events Reference

- `struct MarginPoolCreated has copy, drop`
  - Fields: `pool_id: ID`, `asset_type: TypeName`
- `struct DeepbookPoolUpdated has copy, drop`
  - Fields: `pool_id: ID`, `deepbook_pool_id: ID`, `enabled: bool`
- `struct InterestParamsUpdated has copy, drop`
  - Fields: `pool_id: ID`, `base_rate: u64`, `base_slope: u64`, `optimal_utilization: u64`, `excess_slope: u64`
- `struct MarginPoolConfigUpdated has copy, drop`
  - Fields: `pool_id: ID`, `supply_cap: u64`, `max_utilization: u64`, `min_borrow: u64`, `referral_spread: u64`
- `struct SupplierCapMinted has copy, drop`
  - Fields: `cap_id: ID`, `owner: address`
- `struct AssetSupplied has copy, drop`
  - Fields: `pool_id: ID`, `supplier: ID`, `amount: u64`, `shares_minted: u64`
- `struct AssetWithdrawn has copy, drop`
  - Fields: `pool_id: ID`, `supplier: ID`, `amount: u64`, `shares_burned: u64`
- `struct MaintainerFeesWithdrawn has copy, drop`
  - Fields: `pool_id: ID`, `amount: u64`
- `struct ProtocolFeesWithdrawn has copy, drop`
  - Fields: `pool_id: ID`, `amount: u64`
- `struct ProtocolFeesIncreased has copy, drop`
  - Fields: `pool_id: ID`, `amount: u64`
