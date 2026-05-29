---
name: deepbook-predict-registry
description: Register new oracles, manage accepted quote assets, configure pricing spreads, set ask bounds, and pause trading using the registry module.
---

# DeepBook Predict: Registry & Protocol Administration

The `Registry` shared object coordinates governance and operator controls. It tracks deployed `Predict` shared objects, associates oracles with their respective operator capabilities, and manages asset configurations and risk settings.

---

## 1. Move Smart Contract API (`registry` module)

### Setup & Deployments
- `public fun create_predict<QuoteAsset>(registry: &mut Registry, admin_cap: &AdminCap, ctx: &mut TxContext)`
  Deploys the shared `Predict` object once for a specific quote asset, registering its object ID in the registry.

### Oracle Registration
- `public fun create_oracle_cap(registry: &mut Registry, admin_cap: &AdminCap, ctx: &mut TxContext): OracleSVICap`
  Mints a new `OracleSVICap` capability.
- `public fun create_oracle(registry: &mut Registry, cap: &OracleSVICap, underlying: String, expiry: u64, strike_grid: vector<u64>, ctx: &mut TxContext)`
  Creates a new `OracleSVI` object linked to the calling cap, and initializes the vault's strike-matrix grids for the specified strikes.

### Asset Management
- `public fun enable_quote_asset<QuoteAsset>(registry: &mut Registry, admin_cap: &AdminCap)`
  Enables a quote asset for position minting and LP supplies.
- `public fun disable_quote_asset<QuoteAsset>(registry: &mut Registry, admin_cap: &AdminCap)`
  Disables a quote asset.

### Pricing Configurations
- `public fun configure_pricing(registry: &mut Registry, admin_cap: &AdminCap, global_spread: u64, min_spread: u64, utilization_multiplier: u64, global_min_ask: u64, global_max_ask: u64)`
  Updates global spread variables, utilization factors, and ask price boundaries.
- `public fun override_oracle_ask_bounds(registry: &mut Registry, cap: &OracleSVICap, oracle_id: ID, min_ask: u64, max_ask: u64)`
  Overrides ask bounds for a specific oracle. Overrides can only **tighten** the global limits.

### Trading & Risk Controls
- `public fun pause_trading(registry: &mut Registry, admin_cap: &AdminCap)`
  Pauses all position minting.
- `public fun resume_trading(registry: &mut Registry, admin_cap: &AdminCap)`
  Resumes position minting.
- `public fun configure_risk(registry: &mut Registry, admin_cap: &AdminCap, max_total_exposure_pct: u64, withdrawal_limiter_pct: u64)`
  Updates the maximum total vault exposure percentage limits and withdrawal limits.

---

## 2. Structs & Capabilities

- `Registry`: The main shared coordinator object.
- `AdminCap`: The admin capability object that authorizes protocol parameter updates and asset listings.
- `OracleSVICap`: The oracle capability object that authorizes updates to spot/forward prices and SVI volatility maps.
