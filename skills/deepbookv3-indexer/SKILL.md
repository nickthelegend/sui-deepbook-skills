---
name: deepbookv3-indexer
description: Query real-time L2 order books, historical volumes, candlestick charts, and wallet portfolios using the DeepBook V3 Indexer REST API.
---

# DeepBook V3: Indexer API

The DeepBook V3 Indexer is a centralized data service provided by Mysten Labs. It aggregates, parses, and serves real-time and historical on-chain events from the DeepBook V3 protocol.

---

## 1. Public Indexer Endpoints

- **Mainnet**: `https://deepbook-indexer.mainnet.mystenlabs.com`
- **Testnet**: `https://deepbook-indexer.testnet.mystenlabs.com`

---

## 2. Token Volume & Price Conversions

Volumes returned by the indexer endpoints are expressed in the **smallest unit** of the corresponding asset (e.g. MIST for SUI). To get the standard asset unit, divide the returned integer by $10^{\text{Scalar}}$ using these values:

| Asset | Scalar | Asset | Scalar |
|---|---|---|---|
| **SUI** | 9 | **ALKIMI** | 9 |
| **DEEP** | 6 | **AUSD** | 6 |
| **USDC** | 6 | **BETH** (Bridged ETH) | 8 |
| **USDT** | 6 | **DRF** | 6 |
| **NS** | 6 | **IKA** | 9 |
| **SEND** | 6 | **LZWBTC** (LayerZero WBTC)| 8 |
| **WAL** | 9 | **TYPUS** | 9 |
| **SUIUSDE**| 6 | **USDSUI** | 6 |
| **WUSDC** | 6 | **WUSDT** | 6 |
| **xBTC** | 8 | | |

*   **Example**: If the SUI/USDC pool returns a volume of `1,000,000,000` base asset units:
    $$\text{Volume in SUI} = 1,000,000,000 / 10^9 = 1 \text{ SUI}$$

---

## 3. Core REST API Endpoints

### Pool & Market Metadata
- **`GET /get_pools`**
  Returns details for all pools (IDs, decimals, symbols, names, min size, lot size, tick size).
- **`GET /assets`**
  Returns contract metadata and allowed statuses (deposit/withdrawal flags) for all coins traded.
- **`GET /deep_supply`**
  Returns the total supply of DEEP tokens.
- **`GET /margin_supply`**
  Returns total supply balances of assets in margin pools.

### Order Book & Trades
- **`GET /orderbook/:pool_name?level={1|2}&depth={integer}`**
  Returns L2 order book bids and asks sorted best-to-worst.
  - `level=1`: Best bid/ask only.
  - `level=2` *(Default)*: Full L2 depth.
  - `depth`: E.g., `100` returns 50 bids and 50 asks.
- **`GET /trades/:pool_name?limit=&start_time=&end_time=&maker_balance_manager_id=&taker_balance_manager_id=`**
  Returns the most recent matches (taker direction, maker/taker fee breakdown, pay-in-DEEP flags).
- **`GET /ohclv/:pool_name?interval=<1m|5m|15m|30m|1h|4h|1d|1w>&start_time=&end_time=&limit=`**
  Returns candlestick data: `[timestamp, open, high, low, close, volume]`.

### Historical Volume
- **`GET /historical_volume/:pool_names?start_time=&end_time=&volume_in_base=`**
  Returns historical volume for comma-separated pools. Volume is quote-denominated unless `volume_in_base=true`.
- **`GET /all_historical_volume?start_time=&end_time=&volume_in_base=`**
  Returns historical volume for all pools combined.
- **`GET /historical_volume_by_balance_manager_id/:pool_names/:balance_manager_id`**
  Returns maker/taker volume arrays: `[maker_volume, taker_volume]`.
- **`GET /historical_volume_by_balance_manager_id_with_interval/:pool_names/:balance_manager_id?interval=86400`**
  Returns interval-segmented volume stats.

### User & Portfolio Queries
- **`GET /orders/:pool_name/:balance_manager_id?limit=&status=`**
  Returns resting/active orders. Status can be comma-separated list: `Placed`, `Canceled`, `Filled`.
- **`GET /order_updates/:pool_name?limit=&status=Placed&balance_manager_id=`**
  Returns recent order updates (places/cancels).
- **`GET /portfolio/:wallet_address`**
  Returns full portfolio (margin positions, collaterals, LP shares, equity summary).
- **`GET /deposited_assets/:balance_manager_ids`**
  Returns asset names deposited in comma-separated managers.
- **`GET /get_points?addresses=`**
  Returns trading rewards points accumulated for comma-separated addresses.

### Health Status
- **`GET /status?max_checkpoint_lag=100&max_time_lag_seconds=60`**
  Checks indexer sync status. Returns `"status": "OK" | "UNHEALTHY"` and checkpoint lag stats.
