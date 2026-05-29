---
name: deepbookv3-sdk-balance-manager
description: Create, deposit, withdraw, mint caps, generate TradeProofs, and manage referrals for BalanceManagers using the TS SDK.
---

# DeepBook V3: BalanceManager SDK

The `BalanceManager` is the core on-chain object that holds user funds. The `@mysten/deepbook-v3` SDK provides simple wrappers to manage these accounts, deposit/withdraw assets, delegate access via capabilities (caps), and generate `TradeProof` tokens.

---

## 1. Primary SDK Methods

### Creation & Sharing
- `createAndShareBalanceManager()(tx)`
  Creates a new `BalanceManager` shared object.
- `createBalanceManagerWithOwner(ownerAddress)(tx)`
  Creates a `BalanceManager` with a custom owner address. Returns the manager object.
- `shareBalanceManager(manager)(tx)`
  Shares a previously created (but unshared) `BalanceManager` object.

### Deposits & Withdrawals
- `depositIntoManager(managerKey, coinKey, amountToDeposit)(tx)`
  Deposits a specified amount of a coin type (identified by `coinKey`) into the balance manager.
- `withdrawFromManager(managerKey, coinKey, amountToWithdraw, recipient)(tx)`
  Withdraws a specified amount of a coin type and transfers it to the recipient's address.
- `withdrawAllFromManager(managerKey, coinKey, recipient)(tx)`
  Withdraws the entire balance of the specified coin and sends it to the recipient's address.
- `checkManagerBalance(managerKey, coinKey)` *(Read-Only)*
  Queries a balance manager's balance for a specific coin. Returns `{ coinType: string, balance: number }`.

### Proof Generation
- `generateProof(managerKey)(tx)`
  Automatically generates a `TradeProof`. If a `tradeCap` is registered in the client config, it generates the proof as a trader; otherwise, it generates it as the direct owner.
- `generateProofAsOwner(managerId)(tx)`
  Generates a proof as the owner of the balance manager.
- `generateProofAsTrader(managerId, tradeCapId)(tx)`
  Generates a proof using a delegated `TradeCap`.

### Capabilities (Delegation)
- `mintTradeCap(managerKey)(tx)`
  Mints a `TradeCap` object to delegate trading rights.
- `mintDepositCap(managerKey)(tx)`
  Mints a `DepositCap` object to delegate deposit rights.
- `mintWithdrawalCap(managerKey)(tx)`
  Mints a `WithdrawCap` object to delegate withdrawal rights.
- `depositWithCap(managerKey, coinKey, amountToDeposit)(tx)`
  Deposits funds into the manager using a delegated `depositCap`.
- `withdrawWithCap(managerKey, coinKey, amountToWithdraw)(tx)`
  Withdraws funds from the manager using a delegated `withdrawCap`.
- `revokeTradeCap(managerKey, tradeCapId)(tx)`
  Revokes a capability (and any corresponding deposit/withdraw caps).

### Referral & Registration Setup
- `setBalanceManagerReferral(managerKey, referral, tradeCap)(tx)`
  Sets a pool-specific referral (`DeepBookPoolReferral` ID) for the balance manager. Requires the `TradeCap` object.
- `unsetBalanceManagerReferral(managerKey, poolKey, tradeCap)(tx)`
  Removes a referral association for the specified pool. Requires the `TradeCap` object.
- `getBalanceManagerReferralId(managerKey, poolKey)(tx)`
  Queries the referral ID associated with the balance manager for a pool.
- `registerBalanceManager(managerKey)(tx)`
  Registers a balance manager with the global DeepBook Registry.

### Metadata Queries
- `owner(managerKey)(tx)`
  Returns the owner address of a balance manager.
- `id(managerKey)(tx)`
  Returns the object ID of a balance manager.
- `balanceManagerReferralOwner(referralId)(tx)`
  Returns the owner of a pool referral object.
- `balanceManagerReferralPoolId(referralId)(tx)`
  Returns the pool ID associated with a referral.

---

## 2. Examples

### Complete Creation & Setup

```typescript
import { Transaction } from '@mysten/sui/transactions';
import { DeepBookClient } from '@mysten/deepbook-v3';

const client = new DeepBookClient(suiClient);

async function setupManager() {
    const tx = new Transaction();
    const ownerAddress = '0xUSER_WALLET_ADDRESS';

    // 1. Create the manager with a custom owner
    const manager = tx.add(client.deepbook.balanceManager.createBalanceManagerWithOwner(ownerAddress));

    // 2. Share the balance manager shared object
    tx.add(client.deepbook.balanceManager.shareBalanceManager(manager));

    // 3. Mint capabilities and transfer them to the owner
    const tradeCap = tx.add(client.deepbook.balanceManager.mintTradeCap('MANAGER_1'));
    const depositCap = tx.add(client.deepbook.balanceManager.mintDepositCap('MANAGER_1'));
    const withdrawCap = tx.add(client.deepbook.balanceManager.mintWithdrawalCap('MANAGER_1'));
    
    tx.transferObjects([tradeCap, depositCap, withdrawCap], ownerAddress);

    await suiClient.signAndExecuteTransaction({
        transaction: tx,
        signer: keypair
    });
}
```

### Deposits & Withdrawals

```typescript
async function handleFunds() {
    const tx = new Transaction();

    // Deposit 100 USDC (SDK handles decimal scaling internally)
    tx.add(client.deepbook.balanceManager.depositIntoManager('MANAGER_1', 'USDC', 100));

    // Withdraw 50 SUI to an external address
    tx.add(client.deepbook.balanceManager.withdrawFromManager(
        'MANAGER_1',
        'SUI',
        50,
        '0xRECIPIENT_WALLET_ADDRESS'
    ));

    await suiClient.signAndExecuteTransaction({
        transaction: tx,
        signer: keypair
    });
}
```
