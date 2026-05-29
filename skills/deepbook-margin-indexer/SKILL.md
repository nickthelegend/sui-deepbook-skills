---
name: deepbook-margin-indexer
description: Query margin-specific trade history, borrow/repay events, liquidations, positions, pool metrics, and collateral logs using the DeepBook Margin Indexer REST API.
---

# DeepBook Margin Indexer: REST API Reference

The DeepBook Margin Indexer endpoints provide real-time and historical query access for monitoring margin manager actions, debt levels, liquidations, lending pool states, and registry logs.

---

## 1. Connection Endpoints

- **Mainnet**: `https://deepbook-indexer.mainnet.mystenlabs.com/`
- **Testnet**: `https://deepbook-indexer.testnet.mystenlabs.com/`

### Common Query Parameters
Most endpoints support filtering via the following parameters:
- `start_time` (optional): Filter start in Unix timestamp seconds (default: 24h ago).
- `end_time` (optional): Filter end in Unix timestamp seconds (default: current time).
- `limit` (optional): Maximum records to return (default: 1).

---

## 2. Margin Manager Endpoints

### Get Margin Manager Creation Events
- **Path**: `/margin_manager_created?margin_manager_id=<ID>`
- **Response**:
```json
[
  {
    "event_digest": "0xabc123...",
    "digest": "0xdef456...",
    "sender": "0x1111...",
    "checkpoint": 12345678,
    "checkpoint_timestamp_ms": 1738000000000,
    "package": "0x2222...",
    "margin_manager_id": "0x1234...",
    "balance_manager_id": "0x5678...",
    "deepbook_pool_id": "0x9abc...",
    "owner": "0xabcd...",
    "onchain_timestamp": 1738000000000
  }
]
```

### Get Loan Borrowed Events
- **Path**: `/loan_borrowed?margin_manager_id=<ID>&margin_pool_id=<ID>`
- **Response**:
```json
[
  {
    "event_digest": "0xabc123...",
    "digest": "0xdef456...",
    "sender": "0x1111...",
    "checkpoint": 12345678,
    "checkpoint_timestamp_ms": 1738000000000,
    "package": "0x2222...",
    "margin_manager_id": "0x1234...",
    "margin_pool_id": "0x5678...",
    "loan_amount": 1000000000,
    "loan_shares": 1000000000,
    "onchain_timestamp": 1738000000000
  }
]
```

### Get Loan Repaid Events
- **Path**: `/loan_repaid?margin_manager_id=<ID>&margin_pool_id=<ID>`
- **Response**:
```json
[
  {
    "event_digest": "0xabc123...",
    "digest": "0xdef456...",
    "sender": "0x1111...",
    "checkpoint": 12345678,
    "checkpoint_timestamp_ms": 1738000000000,
    "package": "0x2222...",
    "margin_manager_id": "0x1234...",
    "margin_pool_id": "0x5678...",
    "repay_amount": 1000000000,
    "repay_shares": 1000000000,
    "onchain_timestamp": 1738000000000
  }
]
```

### Get Liquidation Events
- **Path**: `/liquidation?margin_manager_id=<ID>&margin_pool_id=<ID>`
- **Response**:
```json
[
  {
    "event_digest": "0xabc123...",
    "digest": "0xdef456...",
    "sender": "0x1111...",
    "checkpoint": 12345678,
    "checkpoint_timestamp_ms": 1738000000000,
    "package": "0x2222...",
    "margin_manager_id": "0x1234...",
    "margin_pool_id": "0x5678...",
    "liquidation_amount": 1000000000,
    "pool_reward": 10000000,
    "pool_default": 0,
    "risk_ratio": 800000000,
    "onchain_timestamp": 1738000000000,
    "remaining_base_asset": "500000000",
    "remaining_quote_asset": "2000000000",
    "remaining_base_debt": "0",
    "remaining_quote_debt": "0",
    "base_pyth_price": 100000000,
    "base_pyth_decimals": 8,
    "quote_pyth_price": 100000000,
    "quote_pyth_decimals": 8
  }
]
```

### Get Aggregated Margin Managers Metadata
- **Path**: `/margin_managers_info`
- **Response**:
```json
[
  {
    "margin_manager_id": "0x1234...",
    "deepbook_pool_id": "0x5678...",
    "base_asset_id": "0xabcd...",
    "base_asset_symbol": "SUI",
    "quote_asset_id": "0xefgh...",
    "quote_asset_symbol": "USDC",
    "base_margin_pool_id": "0x1111...",
    "quote_margin_pool_id": "0x2222..."
  }
]
```

### Get Current Margin Manager States (Liquidation Monitor)
Used to find positions whose risk ratios are approaching or below target liquidations.
- **Path**: `/margin_manager_states?max_risk_ratio=<FLOAT>&deepbook_pool_id=<ID>`
- **Response**:
```json
[
  {
    "id": 1,
    "margin_manager_id": "0x1234...",
    "deepbook_pool_id": "0x5678...",
    "base_margin_pool_id": "0x1111...",
    "quote_margin_pool_id": "0x2222...",
    "base_asset_id": "0xabcd...",
    "base_asset_symbol": "SUI",
    "quote_asset_id": "0xefgh...",
    "quote_asset_symbol": "USDC",
    "risk_ratio": "1.5",
    "base_asset": "1000000000",
    "quote_asset": "5000000000",
    "base_debt": "500000000",
    "quote_debt": "2000000000",
    "base_pyth_price": 100000000,
    "base_pyth_decimals": 8,
    "quote_pyth_price": 100000000,
    "quote_pyth_decimals": 8,
    "created_at": "2025-01-01 00:00:00",
    "updated_at": "2025-01-01 12:00:00",
    "current_price": "2.5",
    "lowest_trigger_above_price": null,
    "highest_trigger_below_price": null
  }
]
```

---

## 3. Margin Pool Endpoints

### Get Asset Supplied / Withdrawn Events
- **Paths**:
  - `/asset_supplied?margin_pool_id=<ID>&supplier=<ADDRESS>`
  - `/asset_withdrawn?margin_pool_id=<ID>&supplier=<ADDRESS>`
- **Response**:
```json
[
  {
    "event_digest": "0xabc123...",
    "digest": "0xdef456...",
    "sender": "0x1111...",
    "checkpoint": 12345678,
    "checkpoint_timestamp_ms": 1738000000000,
    "package": "0x2222...",
    "margin_pool_id": "0x1234...",
    "asset_type": "0x2::sui::SUI",
    "supplier": "0xabcd...",
    "amount": 1000000000,
    "shares": 1000000000,
    "onchain_timestamp": 1738000000000
  }
]
```

### Get Margin Pool Creation Events
- **Path**: `/margin_pool_created?margin_pool_id=<ID>`
- **Response**:
```json
[
  {
    "event_digest": "0xabc123...",
    "digest": "0xdef456...",
    "sender": "0x1111...",
    "checkpoint": 12345678,
    "checkpoint_timestamp_ms": 1738000000000,
    "package": "0x2222...",
    "margin_pool_id": "0x1234...",
    "maintainer_cap_id": "0x5678...",
    "asset_type": "0x2::sui::SUI",
    "config_json": {
      "margin_pool_config": {
        "supply_cap": 10000000000000,
        "max_utilization_rate": 950000000,
        "protocol_spread": 50000000,
        "min_borrow": 1000000
      },
      "interest_config": {
        "base_rate": 100000,
        "base_slope": 200000,
        "optimal_utilization": 800000000,
        "excess_slope": 500000
      }
    },
    "onchain_timestamp": 1738000000000
  }
]
```

### Get DeepBook Pool Enabled Status Updates
- **Path**: `/deepbook_pool_updated?margin_pool_id=<ID>&deepbook_pool_id=<ID>`
- **Response**:
```json
[
  {
    "event_digest": "0xabc123...",
    "digest": "0xdef456...",
    "sender": "0x1111...",
    "checkpoint": 12345678,
    "checkpoint_timestamp_ms": 1738000000000,
    "package": "0x2222...",
    "margin_pool_id": "0x1234...",
    "deepbook_pool_id": "0x5678...",
    "pool_cap_id": "0x9abc...",
    "enabled": true,
    "onchain_timestamp": 1738000000000
  }
]
```

### Get Interest Parameters Update Events
- **Path**: `/interest_params_updated?margin_pool_id=<ID>`
- **Response**:
```json
[
  {
    "event_digest": "0xabc123...",
    "digest": "0xdef456...",
    "sender": "0x1111...",
    "checkpoint": 12345678,
    "checkpoint_timestamp_ms": 1738000000000,
    "package": "0x2222...",
    "margin_pool_id": "0x1234...",
    "pool_cap_id": "0x5678...",
    "config_json": {
      "base_rate": 100000,
      "base_slope": 200000,
      "optimal_utilization": 800000000,
      "excess_slope": 500000
    },
    "onchain_timestamp": 1738000000000
  }
]
```

### Get Margin Pool Config Updates
- **Path**: `/margin_pool_config_updated?margin_pool_id=<ID>`
- **Response**:
```json
[
  {
    "event_digest": "0xabc123...",
    "digest": "0xdef456...",
    "sender": "0x1111...",
    "checkpoint": 12345678,
    "checkpoint_timestamp_ms": 1738000000000,
    "package": "0x2222...",
    "margin_pool_id": "0x1234...",
    "pool_cap_id": "0x5678...",
    "config_json": {
      "supply_cap": 10000000000000,
      "max_utilization_rate": 950000000,
      "protocol_spread": 50000000,
      "min_borrow": 1000000
    },
    "onchain_timestamp": 1738000000000
  }
]
```

### Get Supplier Cap & Referral Mint Events
- **Paths**:
  - `/supplier_cap_minted?supplier_cap_id=<ID>`
  - `/supply_referral_minted?margin_pool_id=<ID>&owner=<ADDRESS>`
- **Referral Mint Response**:
```json
[
  {
    "event_digest": "0xabc123...",
    "digest": "0xdef456...",
    "sender": "0x1111...",
    "checkpoint": 12345678,
    "checkpoint_timestamp_ms": 1738000000000,
    "package": "0x2222...",
    "margin_pool_id": "0x5678...",
    "supply_referral_id": "0x1234...",
    "owner": "0xabcd...",
    "onchain_timestamp": 1738000000000
  }
]
```

### Get Protocol Fee Increases & Referral Claims
- **Paths**:
  - `/protocol_fees_increased?margin_pool_id=<ID>`
  - `/referral_fees_claimed?referral_id=<ID>&owner=<ADDRESS>`
- **Fee Increase Response**:
```json
[
  {
    "event_digest": "0xabc123...",
    "digest": "0xdef456...",
    "sender": "0x1111...",
    "checkpoint": 12345678,
    "checkpoint_timestamp_ms": 1738000000000,
    "package": "0x2222...",
    "margin_pool_id": "0x1234...",
    "total_shares": 1000000000,
    "referral_fees": 100000,
    "maintainer_fees": 200000,
    "protocol_fees": 300000,
    "onchain_timestamp": 1738000000000
  }
]
```

---

## 4. Collateral Events Ledger

- **Path**: `/collateral_events?margin_manager_id=<ID>&type=<"Deposit" | "Withdraw">&is_base=<BOOLEAN>`
- **Response**:
```json
[
  {
    "event_digest": "0xabc123...",
    "digest": "0xdef456...",
    "sender": "0x1111...",
    "checkpoint": 12345678,
    "checkpoint_timestamp_ms": 1738000000000,
    "package": "0x2222...",
    "event_type": "Deposit",
    "margin_manager_id": "0x1234...",
    "amount": "1000000000",
    "asset_type": "0x2::sui::SUI",
    "pyth_decimals": 8,
    "pyth_price": "100000000",
    "withdraw_base_asset": null,
    "base_pyth_decimals": 8,
    "base_pyth_price": "100000000",
    "quote_pyth_decimals": 8,
    "quote_pyth_price": "100000000",
    "remaining_base_asset": "500000000",
    "remaining_quote_asset": "2000000000",
    "remaining_base_debt": "0",
    "remaining_quote_debt": "0",
    "onchain_timestamp": 1738000000000
  }
]
```

---

## 5. Admin & Registry Endpoints

- **Paths**:
  - `/maintainer_cap_updated?maintainer_cap_id=<ID>`
  - `/maintainer_fees_withdrawn?margin_pool_id=<ID>`
  - `/protocol_fees_withdrawn?margin_pool_id=<ID>`
  - `/pause_cap_updated?pause_cap_id=<ID>`
  - `/deepbook_pool_registered?pool_id=<ID>`
  - `/deepbook_pool_updated_registry?pool_id=<ID>`
  - `/deepbook_pool_config_updated?pool_id=<ID>`
- **Registry Pool Registered Response**:
```json
[
  {
    "event_digest": "0xabc123...",
    "digest": "0xdef456...",
    "sender": "0x1111...",
    "checkpoint": 12345678,
    "checkpoint_timestamp_ms": 1738000000000,
    "package": "0x2222...",
    "pool_id": "0x1234...",
    "config_json": {
      "base_margin_pool_id": "0x5678...",
      "quote_margin_pool_id": "0x9abc...",
      "risk_ratios": {
        "min_withdraw_risk_ratio": 1200000000,
        "min_borrow_risk_ratio": 1100000000,
        "liquidation_risk_ratio": 1000000000,
        "target_liquidation_risk_ratio": 1050000000
      },
      "user_liquidation_reward": 50000000,
      "pool_liquidation_reward": 10000000,
      "enabled": true
    },
    "onchain_timestamp": 1738000000000
  }
]
```
