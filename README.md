# Sui DeepBook Skills

This package contains skills for Sui DeepBook V3 integration.

## Skills Included

- `deepbook-hello-world`: A hello world skill demonstrating how to build skills for Sui DeepBook.
- `DeepBookV3`: Full documentation, design overview, and integration guide for DeepBook V3.
- `deepbookv3-contract-information`: Contract addresses, supported coin types, and mainnet pool identifiers for Sui DeepBook V3.
- `deepbookv3-balance-manager`: Manage user balances, capabilities (TradeCap, DepositCap, WithdrawCap), deposits/withdrawals, and TradeProofs in DeepBook V3.
- `deepbookv3-orders`: Place, modify, cancel, and manage limit and market orders in DeepBook V3.
- `deepbookv3-flash-loans`: Perform uncollateralized atomic flash loans using the hot-potato receipt pattern in DeepBook V3.
- `deepbookv3-swaps`: Execute AMM-style immediate swaps with or without a BalanceManager in DeepBook V3.
- `deepbookv3-staking-governance`: Stake DEEP tokens, vote on proposals, submit fee/stake proposals, and claim maker rebates in DeepBook V3.
- `deepbookv3-permissionless-pool`: Create permissionless pools, configure tick/lot/min sizes, feed DEEP price points, and update allowed contract versions in DeepBook V3.
- `deepbookv3-query-the-pool`: Query order book depth, execute dry-run swaps, validate pre-trade parameters, and view balances in DeepBook V3.
- `deepbookv3-referral`: Mint pool referrals, manage multipliers, associate referrals with BalanceManager, and claim referral fees in DeepBook V3.
- `deepbookv3-ewma`: Understand and calculate Exponentially Weighted Moving Average (EWMA) Gas Price Penalties for takers in DeepBook V3.
- `deepbookv3-sdk`: Core installation, client extension setup, constants, and coin maps for the DeepBook V3 TypeScript SDK.
- `deepbookv3-sdk-balance-manager`: Create, deposit, withdraw, mint caps, generate TradeProofs, and manage referrals for BalanceManagers using the TS SDK.
- `deepbookv3-sdk-pools`: Query pool state, fetch order book depth, create pools, register DEEP price points, and manage referral payouts using the TS SDK.
- `deepbookv3-sdk-orders`: Submit, modify, cancel, and settle limit and market orders using the DeepBook V3 TypeScript SDK.
- `deepbookv3-sdk-flash-loans`: Borrow and repay base or quote assets atomically inside a Programmable Transaction Block (PTB) using the TS SDK.
- `deepbookv3-sdk-swaps`: Execute instant AMM-style swaps with or without a BalanceManager using the DeepBook V3 TypeScript SDK.
- `deepbookv3-sdk-staking-governance`: Stake DEEP, unstake, submit governance proposals, vote, and claim rebates using the DeepBook V3 TypeScript SDK.
- `deepbookv3-indexer`: Query real-time L2 order books, historical volumes, candlestick charts, and wallet portfolios using the DeepBook V3 Indexer REST API.
- `deepbook-margin`: High-level overview, architecture design, contract details, risk thresholds, and pool parameters for Sui DeepBook Margin.
- `deepbook-margin-manager`: Manage the MarginManager shared object, deposit/withdraw collateral, borrow assets, repay debt, calculate risk ratios, and execute liquidations.
- `deepbook-margin-pool`: Manage the MarginPool shared object, supply/withdraw liquidity, calculate interest rates, configure parameters, and execute maintainer admin functions.
- `deepbook-margin-orders`: Place, modify, and cancel limit and market orders, reduce debt using reduce-only orders, and handle staking/governance through the MarginManager using the pool_proxy module.
- `deepbook-margin-tpsl`: Create and manage conditional Take-Profit and Stop-Loss (TPSL) orders on the MarginManager, sorting triggers, and executing orders.
- `deepbook-margin-referral`: Implement the supply referral system, mint SupplyReferral objects, claim referred fees, and track protocol fee distributions.





