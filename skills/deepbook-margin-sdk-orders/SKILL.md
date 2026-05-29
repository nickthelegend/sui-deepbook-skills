---
name: deepbook-margin-sdk-orders
description: Submit and manage limit, market, and reduce-only orders, withdraw settled funds, stake DEEP, and vote on governance using the TypeScript SDK.
---

# DeepBook Margin SDK: Trading & Staking

The Orders SDK enables leveraged trading using the `poolProxy` module. Users can place and manage orders, modify active slots, withdraw settled amounts, and stake DEEP tokens for governance.

---

## 1. SDK API Reference (`client.deepbook.poolProxy`)

### Placing Orders
- `placeLimitOrder(params: PlaceMarginLimitOrderParams)`
  Places a limit order using leveraged funds.
  - *Returns*: `(tx: Transaction) => void`
- `placeMarketOrder(params: PlaceMarginMarketOrderParams)`
  Places a market order.
- `placeReduceOnlyLimitOrder(params: PlaceMarginLimitOrderParams)`
  Places a reduce-only limit order. Can **only** close or shrink an existing debt.
- `placeReduceOnlyMarketOrder(params: PlaceMarginMarketOrderParams)`
  Places a reduce-only market order.

### Modifying & Canceling Orders
- `modifyOrder(marginManagerKey: string, orderId: string, newQuantity: number)`
  Modifies the quantity of an active order. Note that `orderId` is the on-chain protocol ID.
- `cancelOrder(marginManagerKey: string, orderId: string)`
  Cancels a single active order.
- `cancelOrders(marginManagerKey: string, orderIds: string[])`
  Cancels multiple orders in a single call.
- `cancelAllOrders(marginManagerKey: string)`
  Cancels all active orders placed by this margin manager.

### Settlements & Rebates
- `withdrawSettledAmounts(marginManagerKey: string)`
  Withdraws assets from completed trades back to the internal manager balances.
- `withdrawMarginSettledAmounts(poolKey: string, marginManagerId: string)`
  **Permissionless** withdrawal function. Anyone can trigger this to withdraw settled balances to the specified manager.
- `claimRebate(marginManagerKey: string)`
  Claims accumulated trading rebates.
- `updateCurrentPrice(poolKey: string)`
  Triggers a Pyth oracle update to synchronize prices.

### Staking & Governance
- `stake(marginManagerKey: string, stakeAmount: number)`
  Stakes DEEP tokens to obtain fee discounts.
- `unstake(marginManagerKey: string)`
  Unstakes DEEP.
- `submitProposal(marginManagerKey: string, params: MarginProposalParams)`
  Submits a governance proposal.
- `vote(marginManagerKey: string, proposalId: string)`
  Votes on a proposal.

---

## 2. Types & Parameters

```typescript
interface PlaceMarginLimitOrderParams {
    poolKey: string;
    marginManagerKey: string;
    clientOrderId: string;
    price: number;
    quantity: number;
    isBid: bool;
    expiration?: number | bigint;
    orderType?: OrderType;
    selfMatchingOption?: SelfMatchingOptions;
    payWithDeep?: boolean;
}

interface PlaceMarginMarketOrderParams {
    poolKey: string;
    marginManagerKey: string;
    clientOrderId: string;
    quantity: number;
    isBid: bool;
    selfMatchingOption?: SelfMatchingOptions;
    payWithDeep?: boolean;
}

interface MarginProposalParams {
    takerFee: number;
    makerFee: number;
    stakeRequired: number;
}
```

---

## 3. Transaction Block Examples

### Example: Place Leveraged Buy Limit Order
```typescript
import { Transaction } from '@mysten/sui/transactions';

async function placeOrder() {
    const tx = new Transaction();

    traderClient.client.deepbook.poolProxy.placeLimitOrder({
        poolKey: 'SUI_DBUSDC',
        marginManagerKey: 'MARGIN_MANAGER_1',
        clientOrderId: '9988',
        price: 2.50,
        quantity: 100,
        isBid: true, // Buy order
        payWithDeep: true,
    })(tx);

    await traderClient.signAndExecute(tx);
}
```

### Example: Place Reduce-Only Order
```typescript
import { Transaction } from '@mysten/sui/transactions';

async function placeReduceOnly() {
    const tx = new Transaction();

    traderClient.client.deepbook.poolProxy.placeReduceOnlyLimitOrder({
        poolKey: 'SUI_DBUSDC',
        marginManagerKey: 'MARGIN_MANAGER_1',
        clientOrderId: '9989',
        price: 2.60,
        quantity: 50,
        isBid: true, // Buying back to reduce short debt
        payWithDeep: true,
    })(tx);

    await traderClient.signAndExecute(tx);
}
```

### Example: Modify and Cancel Orders
```typescript
import { Transaction } from '@mysten/sui/transactions';

async function cancelAllAndModify(orderId: string) {
    const tx = new Transaction();
    const managerKey = 'MARGIN_MANAGER_1';

    // Modify active quantity to 5 tokens
    tx.add(traderClient.client.deepbook.poolProxy.modifyOrder(managerKey, orderId, 5));

    // Cancel all open orders for this manager
    tx.add(traderClient.client.deepbook.poolProxy.cancelAllOrders(managerKey));

    await traderClient.signAndExecute(tx);
}
```
