Stable Rainbow Sidewinder

Medium

# No deadline protection from MEV

## Summary
The protocol uses `block.timestamp` as the deadline argument while executing swaps (`_executeTrade`), which completely defeats the purpose of using a deadline.

This issue is on 
1. [BaseStakingVault._redeemFromNotional](https://github.com/sherlock-audit/2024-06-leveraged-vaults/blob/14d3eaf0445c251c52c86ce88a84a3f5b9dfad94/leveraged-vaults-private/contracts/vaults/staking/BaseStakingVault.sol#L170)
2. [BaseStakingVault._executeInstantRedemption](https://github.com/sherlock-audit/2024-06-leveraged-vaults/blob/14d3eaf0445c251c52c86ce88a84a3f5b9dfad94/leveraged-vaults-private/contracts/vaults/staking/BaseStakingVault.sol#L197)
3. [EthenaVault._stakeTokens](https://github.com/sherlock-audit/2024-06-leveraged-vaults/blob/14d3eaf0445c251c52c86ce88a84a3f5b9dfad94/leveraged-vaults-private/contracts/vaults/staking/EthenaVault.sol#L65)
4. [PendlePrincipalToken._stakeTokens](https://github.com/sherlock-audit/2024-06-leveraged-vaults/blob/14d3eaf0445c251c52c86ce88a84a3f5b9dfad94/leveraged-vaults-private/contracts/vaults/staking/protocols/PendlePrincipalToken.sol#L86)
5. [PendlePrincipalToken._executeInstantRedemption](https://github.com/sherlock-audit/2024-06-leveraged-vaults/blob/14d3eaf0445c251c52c86ce88a84a3f5b9dfad94/leveraged-vaults-private/contracts/vaults/staking/protocols/PendlePrincipalToken.sol#L155)
6. [Ethena._sellStakedUSDe](https://github.com/sherlock-audit/2024-06-leveraged-vaults/blob/14d3eaf0445c251c52c86ce88a84a3f5b9dfad94/leveraged-vaults-private/contracts/vaults/staking/protocols/Ethena.sol#L138-L158)


## Vulnerability Detail
Using block.timestamp as the deadline is effectively a no-operation that has no effect nor protection. Since `block.timestamp` will take the timestamp value when the transaction gets mined, the check will end up comparing block.timestamp against the same value, i.e. block.timestamp <= block.timestamp.

Passing block.timestamp as the expiry/deadline of an operation does not mean "require immediate execution" - it means "whatever block this transaction appears in, I'm comfortable with that block's timestamp". Providing this value means that a malicious miner can hold the transaction for as long as they like (think the flashbots mempool for bundling transactions), which may be until they are able to cause the transaction to incur the maximum amount of slippage allowed by the slippage parameter, or until conditions become unfavorable enough that other orders, e.g. liquidations, are triggered. Timestamps should be chosen off-chain, and should be specified by the caller to avoid unnecessary MEV.


## Impact
No MEV protection on using `block.timestamp` as deadline. Failure to provide a proper deadline value enables pending transactions to be maliciously executed at a later point. Transactions that provide an insufficient amount of gas such that they are not mined within a reasonable amount of time, can be picked by malicious actors or MEV bots and executed later in the detriment of the submitter.

See [this issue](https://github.com/code-423n4/2022-12-backed-findings/issues/64) for an excellent reference on the topic (the author runs an MEV bot).

## Code Snippet


```solidity
            Trade memory trade = Trade({
                tradeType: TradeType.EXACT_IN_SINGLE,
                sellToken: TOKEN_OUT_SY,
                buyToken: BORROW_TOKEN,
                amount: netTokenOut,
                limit: params.minPurchaseAmount,
    >>>>        deadline: block.timestamp,
                exchangeData: params.exchangeData
            });
```

## Tool used
Manual Review

## Recommendation
Add a deadline parameter to the input of functions used to execute swaps (`_executeTrade`). So that off-chain calculated deadline can be used against the MEV.
