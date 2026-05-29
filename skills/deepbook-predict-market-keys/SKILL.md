---
name: deepbook-predict-market-keys
description: Construct and read binary position MarketKey and vertical range RangeKey identifiers to query PredictManager positions.
---

# DeepBook Predict: Market Keys & Ranges

Predict uses structural keys to identify position entries in the `PredictManager` tables.

---

## 1. Binary Position Keys (`MarketKey`)

`MarketKey` represents a directional binary option position on an oracle's outcome at a specific strike and expiry timestamp.

### Structure Fields
- `oracle_id: ID`: The target `OracleSVI` object ID.
- `expiry: u64`: The Unix timestamp (in seconds) of option expiry.
- `strike: u64`: The target trigger price strike.
- `is_up: bool`: Option direction.
  - `true` (Up): Pays if the settlement price is above the strike.
  - `false` (Down): Pays if the settlement price is below/equal to the strike.

### Move API Constructors & Readers
- `public fun new(oracle_id: ID, expiry: u64, strike: u64, is_up: bool): MarketKey`
  Creates a new binary position key.
- `public fun up(oracle_id: ID, expiry: u64, strike: u64): MarketKey`
  Creates a new Up-market key (`is_up = true`).
- `public fun down(oracle_id: ID, expiry: u64, strike: u64): MarketKey`
  Creates a new Down-market key (`is_up = false`).
- `public fun oracle_id(key: &MarketKey): ID`
- `public fun expiry(key: &MarketKey): u64`
- `public fun strike(key: &MarketKey): u64`
- `public fun is_up(key: &MarketKey): bool`

---

## 2. Vertical Range Keys (`RangeKey`)

`RangeKey` represents a bounded vertical price range option. It pays out at settlement if the oracle price lands within the target price band.

### Structure Fields
- `oracle_id: ID`: The target `OracleSVI` object ID.
- `expiry: u64`: The Unix timestamp of option expiry.
- `lower_strike: u64`: Bounded lower price strike (exclusive).
- `higher_strike: u64`: Bounded upper price strike (inclusive).
  - *Constraint*: Payout is triggered if `lower_strike < SettlementPrice <= higher_strike`.

### Move API Constructors & Readers
- `public fun new(oracle_id: ID, expiry: u64, lower_strike: u64, higher_strike: u64): RangeKey`
  Creates a new range key. Aborts if `lower_strike >= higher_strike`.
- `public fun oracle_id(key: &RangeKey): ID`
- `public fun expiry(key: &RangeKey): u64`
- `public fun lower_strike(key: &RangeKey): u64`
- `public fun higher_strike(key: &RangeKey): u64`
