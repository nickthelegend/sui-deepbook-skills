---
name: deepbookv3-staking-governance
description: Stake DEEP tokens, vote on proposals, submit fee/stake proposals, and claim maker rebates in DeepBook V3.
---

# DeepBook V3: Staking and Governance

DeepBook V3 introduces pool-specific governance and staking. Staking DEEP tokens allows traders to receive taker fee discounts, earn maker volume rebates, and participate in governing key pool parameters.

---

## 1. Key Concepts

### Staking Incentives
- **Taker Discounts**: Staking more than the required amount of DEEP reduces taker fees by half (to as low as 0.25 bps on stable pools and 2.5 bps on volatile pools).
- **Maker Rebates**: Active market makers receive rebates from the accumulated fees of the pool, proportional to their volume contribution, provided their active stake exceeds the threshold.
- **Active Epoch**: A user's stake submitted during epoch $N$ becomes active in epoch $N+1$.

### Governance & Voting Power
Governance operates on an epoch-by-epoch timeline. Users submit and vote on proposals to change three pool parameters:
1. **Taker fee rate**
2. **Maker fee rate**
3. **Stake required**

#### Voting Power Formula
Voting power $V$ is determined by the active DEEP stake $S$ relative to a cutoff $V_c$ (currently set to 100,000 DEEP):
$$V = \min(S, V_c) + \max(\sqrt{S} - \sqrt{V_c}, 0)$$
This formula allows linear representation for smaller stakers (up to 100,000 DEEP) and diminishing returns (square-root scaling) for larger stakers, preventing centralized governance control by single entities.

#### Quorum and Enactment
- **Quorum**: Reached when a proposal receives votes representing at least 50% of the total voting power of the pool.
- **Enactment**: Once quorum is met, parameter changes are queued and take effect in the next epoch.

### Parameter Limits

| Pool Type | Taker Fee Range (bps) | Maker Fee Range (bps) |
|---|---|---|
| **Volatile** | 1 to 10 | 0 to 5 |
| **Stable** | 0.1 to 1 | 0 to 0.5 |
| **Whitelisted** | 0 (Fixed) | 0 (Fixed) |

---

## 2. On-Chain API (Move)

The `pool` module exposes these endpoints for staking and governance:

- `public fun stake<BaseAsset, QuoteAsset>(pool: &mut Pool<BaseAsset, QuoteAsset>, balance_manager: &mut BalanceManager, trade_proof: &TradeProof, amount: u64, ctx: &mut TxContext)`
  Stakes DEEP tokens from the balance manager. The stake becomes active in the next epoch.
- `public fun unstake<BaseAsset, QuoteAsset>(pool: &mut Pool<BaseAsset, QuoteAsset>, balance_manager: &mut BalanceManager, trade_proof: &TradeProof, ctx: &mut TxContext)`
  Unstakes all active and inactive DEEP stake immediately back to the balance manager's balance. Casting votes are removed, and incentives are disabled.
- `public fun submit_proposal<BaseAsset, QuoteAsset>(pool: &mut Pool<BaseAsset, QuoteAsset>, balance_manager: &mut BalanceManager, trade_proof: &TradeProof, taker_fee: u64, maker_fee: u64, stake_required: u64, ctx: &mut TxContext)`
  Submits a fee and stake proposal. Automatically votes for own proposal. One proposal is allowed per balance manager per epoch.
- `public fun vote<BaseAsset, QuoteAsset>(pool: &mut Pool<BaseAsset, QuoteAsset>, balance_manager: &mut BalanceManager, trade_proof: &TradeProof, proposal_id: ID, ctx: &mut TxContext)`
  Casts all voting power towards a specific proposal. Replaces any prior vote cast during the current epoch.
- `public fun claim_rebates<BaseAsset, QuoteAsset>(pool: &mut Pool<BaseAsset, QuoteAsset>, balance_manager: &mut BalanceManager, trade_proof: &TradeProof, ctx: &mut TxContext)`
  Claims accumulated rebates for the balance manager and deposits them back into the manager's balance.

---

## 3. TS SDK Usage Examples

Using `@mysten/deepbook-v3` to interact with staking and governance:

### Staking DEEP Tokens

```typescript
import { Transaction } from '@mysten/sui/transactions';
import { DeepBookClient } from '@mysten/deepbook-v3';

const client = new DeepBookClient(suiClient);

async function stakeTokens(amount: bigint) {
    const tx = new Transaction();
    
    client.deepbook.stake(tx, {
        poolKey: 'SUI_DBUSDC',
        balanceManagerKey: '0xYOUR_BM_OBJECT_ID',
        amount,
    });
    
    await suiClient.signAndExecuteTransaction({
        transaction: tx,
        signer: keypair
    });
}
```

### Submitting a Proposal and Voting

```typescript
async function proposeAndVote(proposalId: string) {
    const tx = new Transaction();
    
    // 1. Submit proposal to change volatile fees
    client.deepbook.submitProposal(tx, {
        poolKey: 'SUI_DBUSDC',
        balanceManagerKey: '0xYOUR_BM_OBJECT_ID',
        takerFee: 2n,        // 2 bps
        makerFee: 1n,        // 1 bps
        stakeRequired: 50_000_000_000n // 50,000 DEEP
    });
    
    // 2. Or vote for another active proposal
    client.deepbook.vote(tx, {
        poolKey: 'SUI_DBUSDC',
        balanceManagerKey: '0xYOUR_BM_OBJECT_ID',
        proposalId,
    });
    
    await suiClient.signAndExecuteTransaction({
        transaction: tx,
        signer: keypair
    });
}
```
