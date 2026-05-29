---
name: deepbook-predict-vault
description: Supply liquidity to the shared vault, mint/burn PLP shares, monitor mark-to-market liabilities, and trigger post-settlement matrix compaction.
---

# DeepBook Predict: Vault & Liquidity

The Predict vault holds quote assets, tracks exposure, and acts as the counterparty for all binary and range option mints.

---

## 1. Vault Liquidity (PLP Shares)

Liquidity Providers (LPs) supply quote assets to a shared pool and receive `PLP` shares representing their proportional ownership.

### Minting Shares (`predict::supply`)
- **Initial LP**: If the vault is empty, `PLP` shares are minted **1:1** with the supplied quote asset quantity.
- **Subsequent LPs**: Shares are minted proportionally based on the supplied amount relative to the current net vault value:
  $$\text{PLP Minted} = \text{Supply Amount} \times \frac{\text{Total PLP Supply}}{\text{Net Vault Value}}$$

### Burning Shares & Withdrawing (`predict::withdraw`)
LP withdrawals burn `PLP` and return the quote asset.
- **Withdrawal Limiter**: Withdrawals are blocked if the requested amount would cause remaining vault funds to fall below the current total **maximum payout exposure** required to cover active option contracts.

---

## 2. Exposure & Liability Tracking

To ensure protocol solvency, the vault tracks:
- **Concrete Balances**: Actual reserve balance of quote assets.
- **Mark-to-Market (MtM) Liability**: Aggregate value of all active option positions based on current oracle price parameters.
- **Net Vault Value**: reserve balance minus MtM liabilities.
- **Max Payout**: The theoretical maximum payout if all outstanding options settle at their worst-case boundary.

---

## 3. Move Smart Contract API

LPs and operators manage liquidity using the following functions in the `predict` module:

- `public fun supply<QuoteAsset>(predict: &mut Predict, coin: Coin<QuoteAsset>, ctx: &mut TxContext): Coin<PLP>`
  Deposits quote assets into the vault and mints `PLP` shares.
- `public fun withdraw<QuoteAsset>(predict: &mut Predict, plp: Coin<PLP>, ctx: &mut TxContext): Coin<QuoteAsset>`
  Burns `PLP` shares to withdraw the user's proportional share of quote assets.
- `public fun compact_settled_oracle(predict: &mut Predict, oracle: &OracleSVI, cap: &OracleSVICap)`
  Compacks settled strike-matrix exposure into a constant-size `SettledOracleState` struct once the oracle transitions to the `Settled` lifecycle state. Reduces protocol storage fees.

### Vault Read Queries (`vault` module)
- `public fun total_balance(vault: &Vault): u64`
  Returns the total quote asset balance inside the vault.
- `public fun mtm_liability(vault: &Vault): u64`
  Returns the current aggregate mark-to-market liability.
- `public fun max_payout(vault: &Vault): u64`
  Returns the absolute maximum payout exposure.
- `public fun net_value(vault: &Vault): u64`
  Returns the net value (Balance - MtM Liability) of the vault.
