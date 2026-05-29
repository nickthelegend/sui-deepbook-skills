# DeepBookV3

URL: https://docs.sui.io/onchain-finance/deepbookv3/deepbook

DeepBookV3 is a next-generation decentralized central limit order book (CLOB) built on Sui. DeepBookV3 leverages Sui's parallel execution and lowtransaction **Transaction** A number of commands that execute on inputs to define the result of the transaction. fees to bring a highly performant, low-latency exchange on chain.

The latest version delivers new features including flash loans, governance, improved account abstraction, and enhancements to the existing matching engine. This version also introduces its own tokenomics with the [DEEP token](https://suivision.xyz/coin/0xdeeb7a4662eec9f2f3def03fb937a663dddaa2e215b8078a284d026b7946c270::deep::DEEP) , which you can stake for additional benefits.

DeepBookV3 does not include an end-user interface for token trading. Rather, it offers built-in trading functionality that can support token trades from decentralized exchanges, wallets, or other apps. The available SDK abstracts away a lot of the complexities of interacting with the chain and buildingprogrammable transaction blocks **Programmable transaction blocks** Define all user transactions on Sui. , lowering the barrier of entry for active market making.

info
The documentation refers to theDeepBook **DeepBook** A decentralized central limit order book (CLOB) built on Sui. standard as "DeepBookV3" to avoid confusion with the recently deprecated version ofDeepBook (DeepBookV2).

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

This documentation outlines the design of DeepBookV3, its public endpoints, and provides guidance for integrations. The SDK abstracts away a lot of the complexities of interacting with the chain and buildingprogrammable transaction blocks , lowering the barrier of entry for active market making.

## Open source

DeepBookV3 is open for community development. You can use the [Sui Improvement Proposals](https://github.com/sui-foundation/sips?ref=blog.sui.io) (SIPs) process to suggest changes to make DeepBookV3 better.

[## Design

Learn about DeepBookV3 design, including the Pool, PoolRegistry, and BalanceManager shared objects.

→](/onchain-finance/deepbookv3/design)
[## Contract Information

In this section

- BalanceManager
- Orders
- Flash Loans
- Swaps
- Staking and Governance
- Permissionless Pool Creation
+ 3 more

→](/onchain-finance/deepbookv3/contract-information)
[## DeepBookV3 SDK

In this section

- BalanceManager
- Pools
- Orders
- Flash Loans
- Swaps
- Staking and Governance

→](/onchain-finance/deepbookv3-sdk/)
[## Indexer

DeepBookV3 Indexer provides streamlined, real-time access to order book and trading data from the DeepBookV3 protocol. It acts as a centralized service to aggregate and expose critical data points.

→](/onchain-finance/deepbookv3/deepbookv3-indexer)

# Design

URL: https://docs.sui.io/onchain-finance/deepbookv3/design

At a high level, the DeepBookV3 design follows the following flow, which revolves around three shared objects:

- `Pool` : A sharedobject **Object** The basic unit of storage on Sui. that represents one market and is responsible for managing its order book, users, stakes, and so on. See the Pool shared object section to learn more.
- `PoolRegistry` : Used only during pool creation, it makes sure that duplicate pools are not created and maintainspackage **Package** Smart contracts on Sui. versioning.
- `BalanceManager` : Used to source a user's funds when placing orders. A single `BalanceManager` can be used between all pools. See [BalanceManager](/onchain-finance/deepbookv3/contract-information/balance-manager) to learn more.
![1](/assets/images/DBv3Architecture-65523469d7d2de40637da8d3675bf996.png)

## Pool sharedobject

All public facing functions take in the `Pool` sharedobject as a mutable or immutable reference. `Pool` is made up of three distinct components:

- `Book`
- `State`
- `Vault`
Logic is isolated between components and each component builds on top of the previous one. By maintaining a book, then state, then vault relationship, DeepBookV3 can provide data availability guarantees, improve code readability, and help make maintaining and upgrading the protocol easier.

![Pool Modules](/assets/images/pool-e04c69f5f1bb6fac875581fe0ea421de.png)

### Book

This component is made up of the main `Book`module **Module** A component of a Move package that defines interaction with on-chain objects. along with `Fill` , `OrderInfo` , and `Order`modules . The `Book` struct maintains two `BigVector<Order>` objects for bids and asks, as well as some metadata. It is responsible for storing, matching, modifying, and removing `Orders` .

When placing an order, an `OrderInfo` is first created. If applicable, it is first matched against existing maker orders, accumulating `Fill` s in the process. Any remaining quantity will be used to create an `Order`object and injected into the book. By the end of book processing, the `OrderInfo`object has enough information to update all relevant users and the overall state.

### State

`State` stores `Governance` , `History` , and `Account` . It processes all requests, updating at least one of these stored structs.

#### Governance

The `Governance`module stores data related to the pool's trading params. These parameters are the taker fee, maker fee, and the stake required. Stake required represents the amount of DEEP tokens that a user must have staked in this specific pool to be eligible for taker and maker incentives.

Everyepoch **Epoch** A period of time defined by the network. , users with nonzero stake can submit a proposal to change these parameters. The proposed fees are bounded.

table { width: 100%; display: inline-table; } th:nth-child(1), td:nth-child(1) { width: 15%; } th:nth-child(2), td:nth-child(2) { width: 15%; }
| min_value (bps) | max_value (bps) | Pool type | Taker or maker 
| 1 | 10 | Volatile | Taker 
| 0 | 5 | Volatile | Maker 
| 0.1 | 1 | Stable | Taker 
| 0 | 0.5 | Stable | Maker 
| 0 | 0 | Whitelisted | Taker and maker 

Users can also vote on live proposals. When a proposal exceeds thequorum **Quorum** A set of validators whose combined voting power is greater than 2/3 of the total. , the new trade parameters are queued to go live from the followingepoch and onwards. Proposals and votes are reset everyepoch . Users can start submitting and voting on proposals theepoch following their stake.Quorum is equivalent to half of the total voting power. A user's voting power is calculated with the following formula where V {V} V is the voting power, S {S} S is the amount staked, and V c {V_c} V c is the voting power cutoff. V c {V_c} V c is currently set to 100,000 DEEP.

V = min ⁡ ( S , V c ) + max ⁡ ( S − V c , 0 ) \LARGE V=\min\lparen S,V_c \rparen + \max\lparen \sqrt{S} - \sqrt{V_c} ,0 \rparen V= min ( S ,V c )+ max ( S− V c ,0 )

The following diagram helps visualize the governance lifecycle.

![DeepBookV3 Governance Timeline](/assets/images/governance-166bcc0f64efe0075432b3afc50f1f0a.png)

#### History

The `History`module stores aggregated volumes, trading params, fees collected and fees to burn for the currentepoch and previous epochs. During order processing, fills are used to calculate and update the total volume. Additionally, if the maker of the trade has enough stake, the total staked volume is also updated.

The first operation of everyepoch will trigger an update, moving the currentepoch data into historic data, and resetting the currentepoch data.

User rebate calculations are done in thismodule . During everyepoch , a maker is eligible for rebates as long as their DEEP staked is over the stake required and have contributed in maker volume. The following formula is used to calculate maker fees, quoted from the [Whitepaper:DeepBook **DeepBook** A decentralized central limit order book (CLOB) built on Sui. Token](/assets/files/deepbook-3e24e6e1deeb8cd860682c1fb473b597.pdf) document. Details on maker incentives can be found in section 2.2 of the whitepaper.

> The computation of incentives – which happens after anepoch ends and is only given to makers who have staked the required number of DEEP tokens in advance – is calculated in Equation (3) for a given maker i {i} i . Equation (3) introduces several new variables. First, M {M} M refers to the set of makers who stake a sufficient number of DEEP tokens, and M ˉ \bar{M} M ˉ refers to the set of makers who do not fulfill this condition. Second, F {F} F refers to total fees (collected both from takers and the maker) that a maker's volume has generated in a givenepoch . Third, L {L} L refers to the total liquidity provided by a maker – and specifically the liquidity traded, not just the liquidity quoted. Finally, the critical point p {p} p is the "phaseout" point, at which – if total liquidity provided by other makers' crosses this point – incentives are zero for the maker in thatepoch . This point p {p} p is constant for all makers in a pool andepoch .

Incentives for Maker i = max ⁡ [ F i ( 1 + ∑ j ∈ M ˉ F j ∑ j ∈ M F j ) ( 1 − ∑ j ∈ M ∪ M ˉ L j − L i p ) , 0 ] \LARGE \textsf {Incentives } \textsf {for } \textsf {Maker } i = \max\Bigg\lbrack F_i\Bigg\lparen 1 + \large\cfrac{\sum_{j \in \bar{M}} F_j} {\sum_{j \in M} F_j} \Bigg\rparen\Bigg\lparen \LARGE 1 - \large\cfrac{\sum_{j \in M \cup \bar{M}} L_j - L_i}{p}\Bigg\rparen \LARGE ,0 \Bigg\rbrack Incentives for Maker i= max[ F i ( 1+ ∑ j ∈ MF j∑ j ∈ M ˉF j ) ( 1− p∑ j ∈ M ∪ M ˉL j−L i ) ,0 ] (3)

In essence, if the total volume during anepoch is greater than the median volume from the last 28 days, then there are no rebates. The lower the volume compared to the median, the more rebates are available. The maximum amount of rebates for anepoch is equivalent to the total amount of DEEP collected during thatepoch . Remaining DEEP is burned.

#### Account

`Account` represents a single user and their relevant data. Everything related to volumes, stake, voted proposal, unclaimed rebates, and balances to be transferred. There is a one to one relationship between a `BalanceManager` and an `Account` .

Everyepoch , the first action that a user performs will update their account, triggering a calculation of any potential rebates from the previousepoch , as well as resetting their volumes for the currentepoch . Any new stakes from the previousepoch become active.

Each account has settled and owed balances. Settled balances are what the pool owes to the user, and owed balances are what the user owes to the pool. For example, when placing an order, the user's owed balances increase, representing the funds that the user has to pay to place that order. Then, if a maker order is taken by another user, the maker's settled balances increase, representing the funds that the maker is owed.

### Vault

Everytransaction **Transaction** A number of commands that execute on inputs to define the result of the transaction. that a user performs on DeepBookV3 resets their settled and owed balances. The vault then processes these balances for the user, deducting or adding to funds to their `BalanceManager` .

The vault also stores the `DeepPrice` struct. Thisobject holds up to 100 data points representing the conversion rate between the pool's base or quote asset and DEEP. These data points are sourced from a whitelisted pool, DEEP/USDC or DEEP/SUI. This conversion rate is used to determine the quantity of DEEP tokens required to pay for trading fees.

### BigVector

`BigVector` is an arbitrary sized vector-like data structure, implemented using an onchain B+ Tree to support almost constant time (log base max_fan_out) random access, insertion and removal.

Iteration is supported by exposing access to leaf nodes (slices). Finding the initial slice can be done in almost constant time, and subsequently finding the previous or next slice can also be done in constant time.

Nodes in the B+ Tree are stored as individual dynamic fields hanging off the `BigVector` .

## Place limit order flow

The following diagram of the lifecycle of an order placement action helps visualize the book, then state, then vault flow.

![Place limit order flow](/assets/images/placeorder-070837ca2be2d30e4a77afd64082957a.png)

### Pool

In the `Pool`module , `place_order_int` is called with the user's input parameters. In this function, four things happen in order:

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

To match an `OrderInfo` against the book, the list of `Order` s is iterated in the opposite side of the book. If there is an overlap in price and the existing maker order has not expired, then DeepBookV3 matches their quantities and generates a `Fill` . DeepBookV3 appends that fill to the `OrderInfo` fills, to use later in state. DeepBookV3 updates the existing maker order quantities and status during each match, and removes them from the book if they are completely filled or expired.

Finally, if the `OrderInfo`object has any remaining quantity, DeepBookV3 converts it into a compact `Order`object and injects it into the order book. `Order` has the minimum amount of data necessary for matching, while `OrderInfo` has the maximum amount of data for general processing.

Regardless of direction or order type, all DeepBookV3 matching is processed in a single function.

### State

The `process_create` function in `State` handles the processing of an order creation event within the pool's state: calculating thetransaction amounts and fees for the order, and updating the account volumes accordingly.

First, the function processes the list of fills from the `OrderInfo`object , updating volumes tracked and settling funds for the makers involved. Next, the function retrieves the account's total trading volume and active stake. It calculates the taker's fee based on the user's account stake and volume in DEEP tokens, while the maker fee is retrieved from the governance trade parameters. To receive discounted taker fees, the account must have more than the minimum stake for the pool, and the trading volume in DEEP tokens must exceed the same threshold. If any quantity remains in the `OrderInfo`object , it is added to the account's list of orders as an `Order` and is already created in `Book` .

Finally, the function calculates the partial taker fills and maker order quantities, if there are any, with consideration for the taker and maker fees. It adds these to the previously settled and owed balances from the account. Trade history is updated with the total fees collected from the order and two tuples are returned to `Pool` , settled and owed balances, in (base, quote, DEEP) format, ensuring the correct assets are transferred in `Vault` .

### Vault

The `settle_balance_manager` function in `Vault` is responsible for managing thetransfer **Transfer** Changing the owner of an asset. of any settled and owed amounts for the `BalanceManager` .

First, the function validates that a trader is authorized to use the `BalanceManager` .

Then, for each asset type the process compares `balances_out` against `balances_in` . If the `balances_out` total exceeds `balances_in` , the function splits the difference from the vault's balance and deposits it into the `BalanceManager` . Conversely, if the `balances_in` total exceeds `balances_out` , the function withdraws the difference from the `BalanceManager` and joins it to the vault's balance.

This process is repeated for base, quote, and DEEP asset balances, ensuring all asset balances are accurately reflected and settled between the vault and the `BalanceManager` .
