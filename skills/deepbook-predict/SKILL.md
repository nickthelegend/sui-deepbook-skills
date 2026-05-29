---
name: deepbook-predict
description: General overview, network targets, integration workflows, server endpoints, and live event triggers for Sui DeepBook Predict.
---

# DeepBook Predict: Expiry-Based Prediction Markets

DeepBook Predict is an expiry-based prediction market protocol on Sui. It enables applications to build decentralized prediction markets where users mint and redeem binary positions or vertical ranges against oracle-driven prices, while liquidity providers supply quote assets to a shared vault to receive vault shares (`PLP`).

---

## 1. Testnet Deployment Information (provisional)

Smart contracts and identifiers are pinned to the `predict-testnet-4-16` branch of the `deepbookv3` repository:

| Parameter | Value |
| :--- | :--- |
| **Network** | Testnet |
| **Public Server URL** | `https://predict-server.testnet.mystenlabs.com` |
| **Predict Package ID** | `0xf5ea2b3749c65d6e56507cc35388719aadb28f9cab873696a2f8687f5c785138` |
| **Predict Registry** | `0x43af14fed5480c20ff77e2263d5f794c35b9fab7e2212903127062f4fe2a6e64` |
| **Predict Shared Object** | `0xc8736204d12f0a7277c86388a68bf8a194b0a14c5538ad13f22cbd8e2a38028a` |
| **Current Quote Asset (DUSDC)** | `0xe95040085976bfd54a1a07225cd46c8a2b4e8e2b6732f140a0fc49850ba73e1a::dusdc::DUSDC` |
| **DUSDC Currency ID** | `0xf3000dff421833d4bb8ed58fac146d691a3aaba2785aa1989af65a7089ca3e9c` |
| **DUSDC Decimals** | 6 |
| **PLP Coin Type** | `0xf5ea2b3749c65d6e56507cc35388719aadb28f9cab873696a2f8687f5c785138::plp::PLP` |

---

## 2. Integration Workflows

### User Position Flow
1. Fetch active markets/oracles from the public Predict server.
2. Retrieve or create a shared `PredictManager` object for the user.
3. Deposit enabled quote assets (e.g. DUSDC) into the `PredictManager`.
4. Call `get_trade_amounts()` or `get_range_trade_amounts()` to preview pricing.
5. Mint a binary position or vertical range.
6. Payouts are claimed at expiry and settled back into the `PredictManager`.

### Liquidity Provider Flow
1. Supply quote assets (DUSDC) directly into the shared vault via `predict::supply`.
2. Proportional `PLP` shares are minted and transferred to the LP.
3. To withdraw, LPs call `predict::withdraw`, which burns `PLP` and returns the quote asset (subject to max payout checks).

---

## 3. Public Server REST Indexer Endpoints

Base URL: `https://predict-server.testnet.mystenlabs.com`

### Protocol & Markets
- `GET /status`: Server health check.
- `GET /predicts/:predict_id/state`: Main Predict shared object state and configurations.
- `GET /predicts/:predict_id/oracles`: List of associated oracles.
- `GET /oracles/:oracle_id/state`: Current oracle state parameters.
- `GET /predicts/:predict_id/quote-assets`: Allowlist of quote assets.
- `GET /oracles/:oracle_id/ask-bounds`: Resolved oracle ask bounds.

### Vault & LP Data
- `GET /predicts/:predict_id/vault/summary`: Current vault balance and max payouts.
- `GET /predicts/:predict_id/vault/performance?range=ALL`: Vault historical performance.
- `GET /lp/supplies`: LP supply history logs.
- `GET /lp/withdrawals`: LP withdrawal history logs.

### User Manager & Portfolios
- `GET /managers`: Active Predict manager lists.
- `GET /managers/:manager_id/summary`: Manager summary (owner, balances).
- `GET /managers/:manager_id/positions/summary`: Manager position summary (active quantities).
- `GET /managers/:manager_id/pnl?range=ALL`: Manager PnL over a selected range.

### Price & Trade History
- `GET /oracles/:oracle_id/prices`: Oracle spot/forward price logs.
- `GET /oracles/:oracle_id/prices/latest`: Latest indexed price push.
- `GET /oracles/:oracle_id/svi`: Oracle SVI history.
- `GET /oracles/:oracle_id/svi/latest`: Latest indexed SVI update.
- `GET /positions/minted` / `/positions/redeemed`: History of binary mints and redeems.
- `GET /ranges/minted` / `/ranges/redeemed`: History of range mints and redeems.
- `GET /trades/:oracle_id`: Historical trades for an oracle.

---

## 4. Live Sui Event Streams

For real-time Tape/UI updates, subscribe to Sui checkpoint event streams filtered by the Predict package ID and watch:
- `oracle::OraclePricesUpdated`: Emitted on high-frequency spot/forward updates.
- `oracle::OracleSVIUpdated`: Emitted on volatility updates.
- `oracle::OracleSettled`: Emitted when the first post-expiry update freezes the settlement price.
- `oracle::OracleActivated`: Emitted when an oracle becomes active.
