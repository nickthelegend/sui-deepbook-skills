---
name: DeepBookV3
description: Complete documentation for Sui DeepBook V3 including protocol design, TypeScript SDK integration, and Move smart contract interactions.
---

# DeepBookV3

URL: https://docs.sui.io/onchain-finance/deepbookv3/deepbook

DeepBookV3 is a next-generation decentralized central limit order book (CLOB) built on Sui. DeepBookV3 leverages Sui's parallel execution and low transaction fees to bring a highly performant, low-latency exchange on chain.

The latest version delivers new features including flash loans, governance, improved account abstraction, and enhancements to the existing matching engine. This version also introduces its own tokenomics with the [DEEP token](https://suivision.xyz/coin/0xdeeb7a4662eec9f2f3def03fb937a663dddaa2e215b8078a284d026b7946c270::deep::DEEP) , which you can stake for additional benefits.

DeepBookV3 does not include an end-user interface for token trading. Rather, it offers built-in trading functionality that can support token trades from decentralized exchanges, wallets, or other apps. The available SDK abstracts away a lot of the complexities of interacting with the chain and building programmable transaction blocks, lowering the barrier of entry for active market making.

> [!NOTE]
> The documentation refers to the DeepBook standard as "DeepBookV3" to avoid confusion with the recently deprecated version of DeepBook (DeepBookV2).

## DeepBookV3 tokenomics

The DEEP token pays for trading fees on the exchange. Users can pay trading fees using DEEP tokens or input tokens, but owning, using, and staking DEEP continues to provide the most benefits to active DeepBookV3 traders on the Sui network.

As an example, governance determines the fee for paying in DEEP tokens, which is 20% lower than the fee for using input tokens.

Users that stake DEEP can enjoy taker and maker incentives. Taker incentives can reduce trading fees by half, dropping them to as low as 0.25 basis points (bps) on stable pairs and 2.5 bps on volatile pairs. Maker incentives are rebates earned based on maker volume generated.

## Liquidity support

Similar to order books for other market places, DeepBookV3's CLOB architecture enables you to enter market and limit orders. You can sell SUI tokens, referred to as an ask, can set your price, referred to as a limit order, or sell at the market's going rate. If you are seeking to buy SUI, referred to as a bid, you can pay the current market price or set a limit price. Limit orders only get fulfilled if the CLOB finds a match between a buyer and seller.

If you put in a limit order for 1,000 SUI, and no single seller is currently offering that quantity of tokens, DeepBookV3 automatically pools the current asks to meet the quantity of your bid.

## Transparency and privacy

As a CLOB, DeepBookV3 works like a digital ledger, logging bids and asks in chronological order and automatically finding matches between the two sides. It takes into account user parameters on trades such as prices.

The digital ledger is open so people can view the trades and prices, giving clear proof of fairness. You can use this transparency to create metrics and dashboards to monitor trading activity.

## Documentation

This documentation outlines the design of DeepBookV3, its public endpoints, and provides guidance for integrations. The SDK abstracts away a lot of the complexities of interacting with the chain and building programmable transaction blocks , lowering the barrier of entry for active market making.

## Open source

DeepBookV3 is open for community development. You can use the [Sui Improvement Proposals](https://github.com/sui-foundation/sips?ref=blog.sui.io) (SIPs) process to suggest changes to make DeepBookV3 better.

---

# Design

URL: https://docs.sui.io/onchain-finance/deepbookv3/design

At a high level, the DeepBookV3 design follows the following flow, which revolves around three shared objects:

- `Pool` : A shared object that represents one market and is responsible for managing its order book, users, stakes, and so on. See the Pool shared object section to learn more.
- `PoolRegistry` : Used only during pool creation, it makes sure that duplicate pools are not created and maintains package versioning.
- `BalanceManager` : Used to source a user's funds when placing orders. A single `BalanceManager` can be used between all pools. See BalanceManager in the Contract Information section to learn more.

![DeepBook V3 Architecture](/assets/images/DBv3Architecture-65523469d7d2de40637da8d3675bf996.png)

## Pool shared object

All public facing functions take in the `Pool` shared object as a mutable or immutable reference. `Pool` is made up of three distinct components:

- `Book`
- `State`
- `Vault`

Logic is isolated between components and each component builds on top of the previous one. By maintaining a book, then state, then vault relationship, DeepBookV3 can provide data availability guarantees, improve code readability, and help make maintaining and upgrading the protocol easier.

![Pool Modules](/assets/images/pool-e04c69f5f1bb6fac875581fe0ea421de.png)

### Book

This component is made up of the main `Book` module along with `Fill` , `OrderInfo` , and `Order` modules. The `Book` struct maintains two `BigVector<Order>` objects for bids and asks, as well as some metadata. It is responsible for storing, matching, modifying, and removing `Orders`.

When placing an order, an `OrderInfo` is first created. If applicable, it is first matched against existing maker orders, accumulating `Fill`s in the process. Any remaining quantity will be used to create an `Order` object and injected into the book. By the end of book processing, the `OrderInfo` object has enough information to update all relevant users and the overall state.

### State

`State` stores `Governance` , `History` , and `Account` . It processes all requests, updating at least one of these stored structs.

#### Governance

The `Governance` module stores data related to the pool's trading params. These parameters are the taker fee, maker fee, and the stake required. Stake required represents the amount of DEEP tokens that a user must have staked in this specific pool to be eligible for taker and maker incentives.

Every epoch, users with nonzero stake can submit a proposal to change these parameters. The proposed fees are bounded.

| min_value (bps) | max_value (bps) | Pool type | Taker or maker |
|---|---|---|---|
| 1 | 10 | Volatile | Taker |
| 0 | 5 | Volatile | Maker |
| 0.1 | 1 | Stable | Taker |
| 0 | 0.5 | Stable | Maker |
| 0 | 0 | Whitelisted | Taker and maker |

Users can also vote on live proposals. When a proposal exceeds the quorum, the new trade parameters are queued to go live from the following epoch and onwards. Proposals and votes are reset every epoch. Users can start submitting and voting on proposals the epoch following their stake. Quorum is equivalent to half of the total voting power. A user's voting power is calculated with the following formula where $V$ is the voting power, $S$ is the amount staked, and $V_c$ is the voting power cutoff. $V_c$ is currently set to 100,000 DEEP.

$$V = \min(S, V_c) + \max(\sqrt{S} - \sqrt{V_c}, 0)$$

The following diagram helps visualize the governance lifecycle.

![DeepBookV3 Governance Timeline](/assets/images/governance-166bcc0f64efe0075432b3afc50f1f0a.png)

#### History

The `History` module stores aggregated volumes, trading params, fees collected and fees to burn for the current epoch and previous epochs. During order processing, fills are used to calculate and update the total volume. Additionally, if the maker of the trade has enough stake, the total staked volume is also updated.

The first operation of every epoch will trigger an update, moving the current epoch data into historic data, and resetting the current epoch data.

User rebate calculations are done in this module. During every epoch, a maker is eligible for rebates as long as their DEEP staked is over the stake required and have contributed in maker volume. The following formula is used to calculate maker fees, quoted from the [Whitepaper: DeepBook Token](/assets/files/deepbook-3e24e6e1deeb8cd860682c1fb473b597.pdf) document. Details on maker incentives can be found in section 2.2 of the whitepaper.

> The computation of incentives – which happens after an epoch ends and is only given to makers who have staked the required number of DEEP tokens in advance – is calculated in Equation (3) for a given maker $i$. Equation (3) introduces several new variables. First, $M$ refers to the set of makers who stake a sufficient number of DEEP tokens, and $\bar{M}$ refers to the set of makers who do not fulfill this condition. Second, $F$ refers to total fees (collected both from takers and the maker) that a maker's volume has generated in a given epoch. Third, $L$ refers to the total liquidity provided by a maker – and specifically the liquidity traded, not just the liquidity quoted. Finally, the critical point $p$ is the "phaseout" point, at which – if total liquidity provided by other makers' crosses this point – incentives are zero for the maker in that epoch. This point $p$ is constant for all makers in a pool and epoch.

$$\text{Incentives for Maker } i = \max\left[ F_i\left( 1 + \cfrac{\sum_{j \in \bar{M}} F_j} {\sum_{j \in M} F_j} \right)\left( 1 - \cfrac{\sum_{j \in M \cup \bar{M}} L_j - L_i}{p}\right) ,0 \right]$$

In essence, if the total volume during an epoch is greater than the median volume from the last 28 days, then there are no rebates. The lower the volume compared to the median, the more rebates are available. The maximum amount of rebates for an epoch is equivalent to the total amount of DEEP collected during that epoch. Remaining DEEP is burned.

#### Account

`Account` represents a single user and their relevant data. Everything related to volumes, stake, voted proposal, unclaimed rebates, and balances to be transferred. There is a one to one relationship between a `BalanceManager` and an `Account`.

Every epoch, the first action that a user performs will update their account, triggering a calculation of any potential rebates from the previous epoch, as well as resetting their volumes for the current epoch. Any new stakes from the previous epoch become active.

Each account has settled and owed balances. Settled balances are what the pool owes to the user, and owed balances are what the user owes to the pool. For example, when placing an order, the user's owed balances increase, representing the funds that the user has to pay to place that order. Then, if a maker order is taken by another user, the maker's settled balances increase, representing the funds that the maker is owed.

### Vault

Every transaction that a user performs on DeepBookV3 resets their settled and owed balances. The vault then processes these balances for the user, deducting or adding to funds to their `BalanceManager`.

The vault also stores the `DeepPrice` struct. This object holds up to 100 data points representing the conversion rate between the pool's base or quote asset and DEEP. These data points are sourced from a whitelisted pool, DEEP/USDC or DEEP/SUI. This conversion rate is used to determine the quantity of DEEP tokens required to pay for trading fees.

### BigVector

`BigVector` is an arbitrary sized vector-like data structure, implemented using an onchain B+ Tree to support almost constant time (log base max_fan_out) random access, insertion and removal.

Iteration is supported by exposing access to leaf nodes (slices). Finding the initial slice can be done in almost constant time, and subsequently finding the previous or next slice can also be done in constant time.

Nodes in the B+ Tree are stored as individual dynamic fields hanging off the `BigVector`.

---

# Place limit order flow

The following diagram of the lifecycle of an order placement action helps visualize the book, then state, then vault flow.

![Place limit order flow](/assets/images/placeorder-070837ca2be2d30e4a77afd64082957a.png)

### Pool

In the `Pool` module , `place_order_int` is called with the user's input parameters. In this function, four things happen in order:

1. An `OrderInfo` is created.
2. The `Book` function `create_order` is called.
3. The `State` function `process_create` is called.
4. The `Vault` function `settle_balance_manager` is called.

### Book

The order creation within the book involves three primary tasks:

- Validate inputs.
- Match against existing orders.
- Inject any remaining quantity into the order book as a limit order.

Validation of inputs ensures that quantity, price, timestamp, and order type are within expected ranges.

To match an `OrderInfo` against the book, the list of `Order`s is iterated in the opposite side of the book. If there is an overlap in price and the existing maker order has not expired, then DeepBookV3 matches their quantities and generates a `Fill`. DeepBookV3 appends that fill to the `OrderInfo` fills, to use later in state. DeepBookV3 updates the existing maker order quantities and status during each match, and removes them from the book if they are completely filled or expired.

Finally, if the `OrderInfo` object has any remaining quantity, DeepBookV3 converts it into a compact `Order` object and injects it into the order book. `Order` has the minimum amount of data necessary for matching, while `OrderInfo` has the maximum amount of data for general processing.

Regardless of direction or order type, all DeepBookV3 matching is processed in a single function.

### State

The `process_create` function in `State` handles the processing of an order creation event within the pool's state: calculating the transaction amounts and fees for the order, and updating the account volumes accordingly.

First, the function processes the list of fills from the `OrderInfo` object, updating volumes tracked and settling funds for the makers involved. Next, the function retrieves the account's total trading volume and active stake. It calculates the taker's fee based on the user's account stake and volume in DEEP tokens, while the maker fee is retrieved from the governance trade parameters. To receive discounted taker fees, the account must have more than the minimum stake for the pool, and the trading volume in DEEP tokens must exceed the same threshold. If any quantity remains in the `OrderInfo` object, it is added to the account's list of orders as an `Order` and is already created in `Book`.

Finally, the function calculates the partial taker fills and maker order quantities, if there are any, with consideration for the taker and maker fees. It adds these to the previously settled and owed balances from the account. Trade history is updated with the total fees collected from the order and two tuples are returned to `Pool` , settled and owed balances, in (base, quote, DEEP) format, ensuring the correct assets are transferred in `Vault`.

### Vault

The `settle_balance_manager` function in `Vault` is responsible for managing the transfer of any settled and owed amounts for the `BalanceManager`.

First, the function validates that a trader is authorized to use the `BalanceManager`.

Then, for each asset type the process compares `balances_out` against `balances_in`. If the `balances_out` total exceeds `balances_in` , the function splits the difference from the vault's balance and deposits it into the `BalanceManager`. Conversely, if the `balances_in` total exceeds `balances_out` , the function withdraws the difference from the `BalanceManager` and joins it to the vault's balance.

This process is repeated for base, quote, and DEEP asset balances, ensuring all asset balances are accurately reflected and settled between the vault and the `BalanceManager`.

---

# Contract Information

URL: https://docs.sui.io/onchain-finance/deepbookv3/contract-information

This section contains contract addresses, supported coins, and pool information for DeepBookV3 on Sui Mainnet.

## Contract versions

DeepBookV3 uses upgradeable contracts. When a contract is upgraded, only `DEEPBOOK_PACKAGE_ID` needs to be updated - previous versions remain compatible unless noted. A redeployment would require updating `DEEPBOOK_PACKAGE_ID`, `REGISTRY_ID`, and all pool IDs.

### Current version

| Parameter | Value |
|---|---|
| Version | 6 |
| Package ID | `0x337f4f4f6567fcd778d5454f27c16c70e2f274cc6377ea6249ddf491482ef497` |
| Registry ID | `0xaf16199a2dff736e9f07a845f23c5da6df6f756eddb631aed9d24a93efc4549d` |

### Version history

| Version | Date | Package ID | Changes |
|---|---|---|---|
| 6 | Jan 7, 2026 | `0x337f4f4f6567fcd778d5454f27c16c70e2f274cc6377ea6249ddf491482ef497` | Final preparation for margin launch |
| 5 | Dec 18, 2025 | `0x2d93777cc8b67c064b495e8606f2f8f5fd578450347bbe7b36e0bc03963c1c40` | Improvements for referral system |
| 4 | Dec 9, 2025 | `0x00c1a56ec8c4c623a848b2ed2f03d23a25d17570b670c22106f336eb933785cc` | Referral system, penalty taker fees |
| 3 | Jun 11, 2025 | `0xb29d83c26cdd2a64959263abbcfc4a6937f0c9fccaf98580ca56faded65be244` | Small bug fix for creating balance manager |
| 2 | Apr 16, 2025 | `0xcaf6ba059d539a97646d47f0b9ddf843e138d215e2a12ca1f4585d386f7aec3a` | Input token fees, permissionless pool creation, gas improvements |
| 1 | Oct 10, 2024 | `0x2c8d603bc51326b8c13cef9dd07031a408a48dddb541963357661df5d3204809` | Original deployment |

## Supported coins

| Coin | Type | Decimals |
|---|---|---|
| **DEEP** | `0xdeeb7a4662eec9f2f3def03fb937a663dddaa2e215b8078a284d026b7946c270::deep::DEEP` | 6 |
| **SUI** | `0x0000000000000000000000000000000000000000000000000000000000000002::sui::SUI` | 9 |
| **USDC** | `0xdba34672e30cb065b1f93e3ab55318768fd6fef66c15942c9f7cb846e2f900e7::usdc::USDC` | 6 |
| **BETH** (Native Bridged ETH) | `0xd0e89b2af5e4910726fbcd8b8dd37bb79b29e5f83f7491bca830e94f7f226d29::eth::ETH` | 8 |
| **WUSDT** (Wormhole USDT) | `0xc060006111016b8a020ad5b33834984a437aaa7d3c74c18e09a95d48aceab08c::coin::COIN` | 6 |
| **WUSDC** (Wormhole USDC) | `0x5d4b302506645c37ff133b98c4b50a5ae14841659738d6d733d59d0d217a93bf::coin::COIN` | 6 |
| **NS** | `0x5145494a5f5100e645e4b0aa950fa6b68f614e8c59e17bc5ded3495123a79178::ns::NS` | 6 |
| **TYPUS** | `0xf82dc05634970553615eef6112a1ac4fb7bf10272bf6cbe0f80ef44a6c489385::typus::TYPUS` | 9 |
| **AUSD** | `0x2053d08c1e2bd02791056171aab0fd12bd7cd7efad2ab8f6b9c8902f14df2ff2::ausd::AUSD` | 6 |
| **DRF** | `0x294de7579d55c110a00a7c4946e09a1b5cbeca2592fbb83fd7bfacba3cfeaf0e::drf::DRF` | 6 |
| **SEND** | `0xb45fcfcc2cc07ce0702cc2d229621e046c906ef14d9b25e8e4d25f6e8763fef7::send::SEND` | 6 |
| **XBTC** | `0x876a4b7bce8aeaef60464c11f4026903e9afacab79b9b142686158aa86560b50::xbtc::XBTC` | 8 |
| **WAL** | `0x356a26eb9e012a68958082340d4c4116e7f55615cf27affcff209cf0ae544f59::wal::WAL` | 9 |
| **IKA** | `0x7262fb2f7a3a14c888c438a3cd9b912469a58cf60f367352c46584262e8299aa::ika::IKA` | 9 |
| **ALKIMI** | `0x1a8f4bc33f8ef7fbc851f156857aa65d397a6a6fd27a7ac2ca717b51f2fd9489::alkimi::ALKIMI` | 9 |
| **LZWBTC** (LayerZero WBTC) | `0x0041f9f9344cac094454cd574e333c4fdb132d7bcc9379bcd4aab485b2a63942::wbtc::WBTC` | 8 |
| **SUI_USDE** | `0x41d587e5336f1c86cad50d38a7136db99333bb9bda91cea4ba69115defeb1402::sui_usde::SUI_USDE` | 6 |
| **USDSUI** | `0x44f838219cf67b058f3b37907b655f226153c18e33dfcd0da559a844fea9b1c1::usdsui::USDSUI` | 6 |

## Pools

> [!NOTE]
> Taker and maker fees are subject to change based on governance proposals. See Staking and Governance for more information on how fees can be adjusted through the governance process.

| Market | Pool ID | Tick Size | Lot Size | Min Size | Taker Fee | Maker Fee |
|---|---|---|---|---|---|---|
| **DEEP/SUI** | `0xb663828d6217467c8a1838a03793da896cbe745b150ebd57d82f814ca579fc22` | 0.00001 | 1 DEEP | 10 DEEP | 0 bps | 0 bps |
| **DEEP/USDC** | `0xf948981b806057580f91622417534f491da5f61aeaf33d0ed8e69fd5691c95ce` | 0.00001 | 1 DEEP | 10 DEEP | 0 bps | 0 bps |
| **SUI/USDC** | `0xe05dafb5133bcffb8d59f4e12465dc0e9faeaa05e3e342a08fe135800e3e4407` | 0.00001 | 0.1 SUI | 1 SUI | 1 bps | 0 bps |
| **BETH/USDC** | `0x1109352b9112717bd2a7c3eb9a416fff1ba6951760f5bdd5424cf5e4e5b3e65c` | 0.001 | 0.0001 BETH | 0.001 BETH | 10 bps | 5 bps |
| **WUSDC/USDC** | `0xa0b9ebefb38c963fd115f52d71fa64501b79d1adcb5270563f92ce0442376545` | 0.00001 | 0.1 WUSDC | 1 WUSDC | 0 bps | 0 bps |
| **WUSDT/USDC** | `0x4e2ca3988246e1d50b9bf209abb9c1cbfec65bd95afdacc620a36c67bdb8452f` | 0.00001 | 0.1 WUSDT | 1 WUSDT | 1 bps | 0.5 bps |
| **NS/SUI** | `0x27c4fdb3b846aa3ae4a65ef5127a309aa3c1f466671471a806d8912a18b253e8` | 0.00001 | 0.1 NS | 1 NS | 10 bps | 5 bps |
| **NS/USDC** | `0x0c0fdd4008740d81a8a7d4281322aee71a1b62c449eb5b142656753d89ebc060` | 0.00001 | 0.1 NS | 1 NS | 10 bps | 5 bps |
| **TYPUS/SUI** | `0xe8e56f377ab5a261449b92ac42c8ddaacd5671e9fec2179d7933dd1a91200eec` | 0.00001 | 0.1 TYPUS | 1 TYPUS | 10 bps | 5 bps |
| **SUI/AUSD** | `0x183df694ebc852a5f90a959f0f563b82ac9691e42357e9a9fe961d71a1b809c8` | 0.0001 | 0.1 SUI | 1 SUI | 10 bps | 5 bps |
| **AUSD/USDC** | `0x5661fc7f88fbeb8cb881150a810758cf13700bb4e1f31274a244581b37c303c3` | 0.00001 | 0.1 AUSD | 1 AUSD | 1 bps | 0.5 bps |
| **DRF/SUI** | `0x126865a0197d6ab44bfd15fd052da6db92fd2eb831ff9663451bbfa1219e2af2` | 0.000001 | 1 DRF | 10 DRF | 10 bps | 5 bps |
| **SEND/USDC** | `0x1fe7b99c28ded39774f37327b509d58e2be7fff94899c06d22b407496a6fa990` | 0.000001 | 0.1 SEND | 1 SEND | 10 bps | 5 bps |
| **WAL/USDC** | `0x56a1c985c1f1123181d6b881714793689321ba24301b3585eec427436eb1c76d` | 0.000001 | 0.1 WAL | 1 WAL | 10 bps | 5 bps |
| **WAL/SUI** | `0x81f5339934c83ea19dd6bcc75c52e83509629a5f71d3257428c2ce47cc94d08b` | 0.000001 | 0.1 WAL | 1 WAL | 10 bps | 5 bps |
| **xBTC/USDC** | `0x20b9a3ec7a02d4f344aa1ebc5774b7b0ccafa9a5d76230662fdc0300bb215307` | 1 | 0.00001 xBTC | 0.00001 xBTC | 10 bps | 5 bps |
| **IKA/USDC** | `0xfa732993af2b60d04d7049511f801e79426b2b6a5103e22769c0cead982b0f47` | 0.000001 | 10 IKA | 10 IKA | 10 bps | 5 bps |
| **ALKIMI/SUI** | `0x84752993c6dc6fce70e25ddeb4daddb6592d6b9b0912a0a91c07cfff5a721d89` | 0.00001 | 0.1 ALKIMI | 1 ALKIMI | 10 bps | 5 bps |
| **LZWBTC/USDC** | `0xf5142aafa24866107df628bf92d0358c7da6acc46c2f10951690fd2b8570f117` | 1 | 0.00001 LZWBTC | 0.00001 LZWBTC | 10 bps | 5 bps |
| **SUIUSDE/USDC** | `0x0fac1cebf35bde899cd9ecdd4371e0e33f44ba83b8a2902d69186646afa3a94b` | 0.000001 | 0.1 SUIUSDE | 0.1 SUIUSDE | 1 bps | 0.5 bps |
| **SUI/SUIUSDE** | `0x034f3a42e7348de2084406db7a725f9d9d132a56c68324713e6e623601fb4fd7` | 0.0001 | 0.1 SUI | 0.1 SUI | 10 bps | 5 bps |
| **USDSUI/USDC** | `0xa374264d43e6baa5aa8b35ff18ff24fdba7443b4bcb884cb4c2f568d32cdac36` | 0.000001 | 0.1 USDSUI | 0.1 USDSUI | 1 bps | 0.5 bps |
| **SUI/USDSUI** | `0x826eeacb2799726334aa580396338891205a41cf9344655e526aae6ddd5dc03f` | 0.0001 | 0.1 SUI | 0.1 SUI | 10 bps | 5 bps |

---

# Developer Integration Guide (TypeScript SDK)

This guide details how off-chain applications interact with Sui DeepBook V3 using the official `@mysten/deepbook-v3` TypeScript SDK.

## Installation

To install the official DeepBook V3 SDK:

```bash
npm install @mysten/deepbook-v3 @mysten/sui
```

## Initialization

Initialize a `SuiClient` and extend it with the `deepbook` extension to create a `DeepBookClient`:

```typescript
import { deepbook } from '@mysten/deepbook-v3';
import { SuiClient } from '@mysten/sui/client';

const suiClient = new SuiClient({
    url: 'https://fullnode.mainnet.sui.io:443', // Or testnet: https://fullnode.testnet.sui.io:443
});

const client = suiClient.$extend(
    deepbook({
        address: '0xYOUR_ACTIVE_WALLET_ADDRESS',
    })
);

// You can now access deepbook methods via client.deepbook
```

## 1. BalanceManager Operations

The `BalanceManager` is a shared object representing a trader's account/vault on-chain. Almost all transactions require a `BalanceManager`.

### Create a BalanceManager

```typescript
import { Transaction } from '@mysten/sui/transactions';

async function createBalanceManager() {
    const tx = new Transaction();
    
    // Create the BalanceManager object
    client.deepbook.createBalanceManager(tx);
    
    const result = await suiClient.signAndExecuteTransaction({
        transaction: tx,
        signer: keypair, // Your active wallet signer
    });
    console.log('BalanceManager Created:', result);
}
```

### Deposit Funds into BalanceManager

Funding your balance manager is necessary before placing limit/market orders.

```typescript
import { Transaction } from '@mysten/sui/transactions';

async function depositFunds(balanceManagerKey: string, coinType: string, amount: bigint) {
    const tx = new Transaction();
    
    // Split the exact amount of coins from the sender's gas or active coin balance
    const [coin] = tx.splitCoins(tx.gas, [amount]); 
    
    // Deposit the split coin into the BalanceManager
    client.deepbook.depositIntoManager(tx, {
        balanceManagerKey, // e.g., '0xManagerAddress'
        coinType,          // e.g., '0x2::sui::SUI'
        coin,
    });
    
    await suiClient.signAndExecuteTransaction({
        transaction: tx,
        signer: keypair,
    });
}
```

### Withdraw Funds from BalanceManager

```typescript
import { Transaction } from '@mysten/sui/transactions';

async function withdrawFunds(balanceManagerKey: string, coinType: string, amount: bigint, recipient: string) {
    const tx = new Transaction();
    
    // Call withdraw
    client.deepbook.withdrawFromManager(tx, {
        balanceManagerKey,
        coinType,
        amount,
        recipient,
    });
    
    await suiClient.signAndExecuteTransaction({
        transaction: tx,
        signer: keypair,
    });
}
```

## 2. Order Placement

All orders require the `poolKey` of the pair and the `balanceManagerKey`.

### Place a Limit Order

```typescript
import { Transaction } from '@mysten/sui/transactions';

async function placeLimitOrder() {
    const tx = new Transaction();
    
    client.deepbook.placeLimitOrder(tx, {
        poolKey: 'SUI_DBUSDC',              // Pool Key configured in SDK constants
        balanceManagerKey: '0xYOUR_BM_ID',  // BalanceManager Object ID
        clientOrderId: '10001',             // Custom order ID to track the order
        price: 1500000000,                  // Scaled price (taking into account pool decimals)
        quantity: 1000000000,               // Scaled quantity
        isBid: true,                        // true for Buy (Bid), false for Sell (Ask)
        payWithDeep: true,                  // true to pay fee in DEEP tokens at a 20% discount
        expiration: Date.now() + 86400000,  // Optional expiration timestamp in ms (24 hours)
    });
    
    const response = await suiClient.signAndExecuteTransaction({
        transaction: tx,
        signer: keypair,
    });
}
```

### Place a Market Order

```typescript
import { Transaction } from '@mysten/sui/transactions';

async function placeMarketOrder() {
    const tx = new Transaction();
    
    client.deepbook.placeMarketOrder(tx, {
        poolKey: 'SUI_DBUSDC',
        balanceManagerKey: '0xYOUR_BM_ID',
        clientOrderId: '20002',
        quantity: 500000000,
        isBid: false,                       // Market sell
        payWithDeep: false,
    });
    
    await suiClient.signAndExecuteTransaction({
        transaction: tx,
        signer: keypair,
    });
}
```

### Cancel or Modify an Order

```typescript
import { Transaction } from '@mysten/sui/transactions';

// Cancel a single order
async function cancelOrder(orderId: string) {
    const tx = new Transaction();
    
    client.deepbook.cancelOrder(tx, {
        poolKey: 'SUI_DBUSDC',
        balanceManagerKey: '0xYOUR_BM_ID',
        orderId,                            // On-chain order ID returned during matching
    });
    
    await suiClient.signAndExecuteTransaction({
        transaction: tx,
        signer: keypair,
    });
}
```

## 3. Direct Swaps (Without BalanceManager)

Swaps are perfect for decentralized apps or routers that need immediate execution without locking funds in a `BalanceManager`.

```typescript
import { Transaction } from '@mysten/sui/transactions';

async function swapTokens(amountIn: bigint, minAmountOut: bigint) {
    const tx = new Transaction();
    
    // Split input coins to swap
    const [inputCoin] = tx.splitCoins(tx.gas, [amountIn]);
    
    // Execute swap
    const [baseOut, quoteOut, deepOut] = client.deepbook.swap(tx, {
        poolKey: 'SUI_DBUSDC',
        inputCoin,
        minOut: minAmountOut,               // Slippage protection limit
        payWithDeep: false,
    });
    
    // Transfer returned coins to the user
    tx.transferObjects([baseOut, quoteOut, deepOut], client.deepbook.address);
    
    await suiClient.signAndExecuteTransaction({
        transaction: tx,
        signer: keypair,
    });
}
```

## 4. Flash Loans

Borrow assets from DeepBook pools and return them within the same transaction block (PTB).

```typescript
import { Transaction } from '@mysten/sui/transactions';

async function executeFlashLoan(borrowAmount: bigint) {
    const tx = new Transaction();
    
    // 1. Borrow from the pool
    const [borrowedBase, borrowedQuote, flashLoanReceipt] = client.deepbook.borrowFlashLoan(tx, {
        poolKey: 'SUI_DBUSDC',
        borrowAmount,
        isBase: true, // true to borrow Base token, false to borrow Quote token
    });
    
    // 2. Perform arbitrary DeFi actions / Arbitrage with borrowedBase/borrowedQuote
    // ... custom logic here ...
    
    // 3. Return the loan before transaction ends
    client.deepbook.returnFlashLoan(tx, {
        poolKey: 'SUI_DBUSDC',
        receipt: flashLoanReceipt,
        // Provide the borrowed assets back, plus any required fees
        repaymentCoin: borrowedBase, 
    });
    
    await suiClient.signAndExecuteTransaction({
        transaction: tx,
        signer: keypair,
    });
}
```

## 5. Querying Market State

### Get Level 2 Order Book Depth

Queries current bids/asks in a specific price range.

```typescript
async function queryOrderBook() {
    const poolKey = 'SUI_DBUSDC';
    const priceLow = 0.9;
    const priceHigh = 1.1;
    
    // Fetch bids
    const bids = await client.deepbook.getLevel2Range(poolKey, priceLow, priceHigh, true);
    // Fetch asks
    const asks = await client.deepbook.getLevel2Range(poolKey, priceLow, priceHigh, false);
    
    console.log('Bids:', bids);
    console.log('Asks:', asks);
}
```

---

# Move Integration Guide (Smart Contracts)

Interacting with DeepBook V3 on-chain inside Sui Move smart contracts.

## Move.toml Dependency

Add the DeepBook package to your dependencies:

```toml
[dependencies]
Sui = { git = "https://github.com/mystenlabs/sui.git", subdir = "crates/sui-framework/packages/sui-framework", rev = "framework/mainnet" }
DeepBook = { git = "https://github.com/mystenlabs/deepbookv3.git", subdir = "contracts/deepbook", rev = "mainnet" }
```

## Creating & Depositing to BalanceManager

```move
module my_app::trader {
    use sui::coin::Coin;
    use sui::tx_context::TxContext;
    use deepbook::balance_manager::{Self, BalanceManager};

    // Create a new BalanceManager
    public entry fun new_manager(ctx: &mut TxContext) {
        let manager = balance_manager::new(ctx);
        balance_manager::share(manager);
    }

    // Deposit funds into an existing shared BalanceManager
    public fun deposit_funds<T>(
        manager: &mut BalanceManager,
        coin: Coin<T>,
        ctx: &mut TxContext
    ) {
        balance_manager::deposit(manager, coin, ctx);
    }
}
```

## Placing Orders in Move

```move
module my_app::trading {
    use sui::tx_context::TxContext;
    use deepbook::pool::{Self, Pool};
    use deepbook::balance_manager::BalanceManager;

    // Place a limit order on-chain
    public fun place_limit_order<Base, Quote>(
        pool: &mut Pool<Base, Quote>,
        balance_manager: &mut BalanceManager,
        client_order_id: u64,
        price: u64,
        quantity: u64,
        is_bid: bool,
        pay_with_deep: bool,
        expiration_timestamp: u64,
        ctx: &mut TxContext
    ) {
        pool::place_limit_order(
            pool,
            balance_manager,
            client_order_id,
            price,
            quantity,
            is_bid,
            pay_with_deep,
            expiration_timestamp,
            ctx
        );
    }
}
```
