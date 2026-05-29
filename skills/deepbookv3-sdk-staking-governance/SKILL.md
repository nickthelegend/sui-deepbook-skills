---
name: deepbookv3-sdk-staking-governance
description: Stake DEEP, unstake, submit governance proposals, vote, and claim rebates using the DeepBook V3 TypeScript SDK.
---

# DeepBook V3: Staking & Governance SDK

Staking and governance in DeepBook V3 can be managed programmatically. The `@mysten/deepbook-v3` TypeScript SDK provides methods to stake DEEP, unstake, propose pool parameter changes, cast votes, and claim rebates.

---

## 1. Parameters & Configuration

### `ProposalParams`
Used when submitting a new governance proposal for a pool:
```typescript
interface ProposalParams {
    poolKey: string;             // Pool name identifier (e.g. 'DBUSDT_DBUSDC')
    balanceManagerKey: string;   // BalanceManager key registered in client
    takerFee: number;            // Proposed taker fee rate in standard decimals
    makerFee: number;            // Proposed maker fee rate in standard decimals
    stakeRequired: number;       // Proposed stake required in standard decimals
}
```

---

## 2. Primary SDK Methods

All staking and governance methods return a builder function that takes a `Transaction` object as an argument:

- `stake(poolKey, balanceManagerKey, stakeAmount)(tx)`
  Stakes the specified amount of DEEP tokens (in standard decimals) from the balance manager into the pool.
- `unstake(poolKey, balanceManagerKey)(tx)`
  Unstakes all active and inactive DEEP tokens immediately from the pool back to the balance manager.
- `submitProposal({ params: ProposalParams })(tx)`
  Submits a governance proposal. Automatically votes for the proposal.
- `vote(poolKey, balanceManagerKey, proposal_id)(tx)`
  Casts the balance manager's voting power (from active stake) to the specified proposal ID.
- `claimRebates(poolKey, balanceManagerKey)(tx)`
  Claims accumulated trading rebates for the balance manager in the pool.

---

## 3. Examples

### Staking and Unstaking

```typescript
import { Transaction } from '@mysten/sui/transactions';
import { DeepBookClient } from '@mysten/deepbook-v3';

const client = new DeepBookClient(suiClient);

async function manageStaking() {
    const tx = new Transaction();

    // 1. Stake 1,000 DEEP in SUI_DBUSDC pool (SDK handles decimal scaling internally)
    client.deepbook.governance.stake('SUI_DBUSDC', 'MANAGER_1', 1000)(tx);

    // 2. Or, unstake all SUI_DBUSDC pool stake
    // client.deepbook.governance.unstake('SUI_DBUSDC', 'MANAGER_1')(tx);

    await suiClient.signAndExecuteTransaction({
        transaction: tx,
        signer: keypair
    });
}
```

### Governance Proposals and Voting

```typescript
async function submitGovernanceProposal(activeProposalId: string) {
    const tx = new Transaction();

    // 1. Submit proposal to change pool parameters (e.g. Taker = 0.002, Maker = 0.001, Stake = 50,000 DEEP)
    client.deepbook.governance.submitProposal({
        poolKey: 'DBUSDT_DBUSDC',
        balanceManagerKey: 'MANAGER_1',
        takerFee: 0.002,
        makerFee: 0.001,
        stakeRequired: 50000,
    })(tx);

    // 2. Or, vote on another active proposal in the same transaction
    client.deepbook.governance.vote('DBUSDT_DBUSDC', 'MANAGER_1', activeProposalId)(tx);

    await suiClient.signAndExecuteTransaction({
        transaction: tx,
        signer: keypair
    });
}
```

### Claiming Rebates

```typescript
async function collectRebates() {
    const tx = new Transaction();

    // Claim maker rebates for pool 'SUI_DBUSDC'
    client.deepbook.governance.claimRebates('SUI_DBUSDC', 'MANAGER_1')(tx);

    await suiClient.signAndExecuteTransaction({
        transaction: tx,
        signer: keypair
    });
}
```
