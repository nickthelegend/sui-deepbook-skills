---
name: deepbook-margin-sdk-tpsl
description: Configure and submit Take-Profit and Stop-Loss (TPSL) orders, execute conditional orders as a keeper, and query trigger states using the TypeScript SDK.
---

# DeepBook Margin SDK: Take Profit & Stop Loss (TPSL)

The TPSL SDK allows users to configure automated conditional orders that execute automatically when target price thresholds are crossed.

---

## 1. SDK API Reference (`client.deepbook.marginTPSL`)

### Managing Conditional Orders
- `addConditionalOrder(params: AddConditionalOrderParams)`
  Registers a conditional order with the margin manager.
  - *Returns*: `(tx: Transaction) => void`
- `cancelConditionalOrder(marginManagerKey: string, conditionalOrderId: string)`
  Cancels a specific conditional order by ID.
  - *Returns*: `(tx: Transaction) => void`
- `cancelAllConditionalOrders(marginManagerKey: string)`
  Cancels all conditional orders linked to the margin manager.
  - *Returns*: `(tx: Transaction) => void`
- `executeConditionalOrders(managerAddress: string, poolKey: string, maxOrdersToExecute: number)`
  **Permissionless** execution loop. Executed by keepers/bots. Evaluates active spot prices and places triggered orders up to `maxOrdersToExecute`.
  - *Returns*: `(tx: Transaction) => void`

### Creation Helpers
- `newCondition(poolKey: string, triggerBelowPrice: boolean, triggerPrice: number)`
  Generates a condition object.
  - *Returns*: `(tx: Transaction) => TransactionArgument`
- `newPendingLimitOrder(poolKey: string, params: PendingLimitOrderParams)`
  Generates a pending limit order payload.
  - *Returns*: `(tx: Transaction) => TransactionArgument`
- `newPendingMarketOrder(poolKey: string, params: PendingMarketOrderParams)`
  Generates a pending market order payload.
  - *Returns*: `(tx: Transaction) => TransactionArgument`

---

## 2. Type Configuration & Parameters

```typescript
interface AddConditionalOrderParams {
    marginManagerKey: string;
    conditionalOrderId: number;
    triggerBelowPrice: boolean; // true for Stop-Loss (below), false for Take-Profit (above)
    triggerPrice: number;
    pendingOrder: PendingLimitOrderParams | PendingMarketOrderParams;
}

interface PendingLimitOrderParams {
    clientOrderId: number;
    price: number;
    quantity: number;
    isBid: boolean;
    orderType?: OrderType;               // Default: NO_RESTRICTION
    selfMatchingOption?: SelfMatching;    // Default: SELF_MATCHING_ALLOWED
    payWithDeep?: boolean;               // Default: true
    expireTimestamp?: number;
}

interface PendingMarketOrderParams {
    clientOrderId: number;
    quantity: number;
    isBid: boolean;
    selfMatchingOption?: SelfMatching;
    payWithDeep?: boolean;
}
```

---

## 3. Read-Only Query Functions

These endpoints fetch active configurations from the blockchain:
- `conditionalOrderIds(managerAddress: string)`: Returns all pending conditional order IDs.
- `conditionalOrder(managerAddress: string, orderId: string)`: Returns a specific conditional order's configuration.
- `lowestTriggerAbovePrice(managerAddress: string)`: Returns the lowest trigger price among trigger-above orders (returns `max_u64` if none).
- `highestTriggerBelowPrice(managerAddress: string)`: Returns the highest trigger price among trigger-below orders (returns `0` if none).

---

## 4. Transaction Block Examples

### Example: Create a Stop-Loss Order (Trigger Below)
```typescript
import { Transaction } from '@mysten/sui/transactions';

async function setStopLoss() {
    const tx = new Transaction();
    const managerKey = 'MARGIN_MANAGER_1';

    // Sell 50 tokens (isBid = false) if the price falls below 2.0
    traderClient.client.deepbook.marginTPSL.addConditionalOrder({
        marginManagerKey: managerKey,
        conditionalOrderId: 1,
        triggerBelowPrice: true, // Trigger on price drop
        triggerPrice: 2.0,
        pendingOrder: {
            clientOrderId: 100,
            quantity: 50,
            isBid: false,
            payWithDeep: true,
        },
    })(tx);

    await traderClient.signAndExecute(tx);
}
```

### Example: Create a Take-Profit Order (Trigger Above)
```typescript
import { Transaction } from '@mysten/sui/transactions';

async function setTakeProfit() {
    const tx = new Transaction();
    const managerKey = 'MARGIN_MANAGER_1';

    // Buy 50 tokens (isBid = true) if the price rises above 5.0 (limit order placed at 5.0)
    traderClient.client.deepbook.marginTPSL.addConditionalOrder({
        marginManagerKey: managerKey,
        conditionalOrderId: 2,
        triggerBelowPrice: false, // Trigger on price rise
        triggerPrice: 5.0,
        pendingOrder: {
            clientOrderId: 101,
            price: 5.0,
            quantity: 50,
            isBid: true,
            payWithDeep: true,
        },
    })(tx);

    await traderClient.signAndExecute(tx);
}
```

### Example: Keeper Order Execution
```typescript
import { Transaction } from '@mysten/sui/transactions';

async function executeTriggeredOrders(managerAddress: string) {
    const tx = new Transaction();

    // Keeper executes up to 10 triggered conditional orders permissionlessly
    traderClient.client.deepbook.marginTPSL.executeConditionalOrders(
        managerAddress, 
        'SUI_USDC', 
        10
    )(tx);

    await traderClient.signAndExecute(tx);
}
```
