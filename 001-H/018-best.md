Daring Rusty Guppy

High

# Selling sUSDe is vulnerable to sandwich attack when staked token is DAI

## Summary
The protocol has functionality that makes a trade in 2 parts (legs). It only has slippage protection in the second part, but the second part is only executed in certain conditions, leaving the trade without slippage protection. 

## Vulnerability Detail
The protocol offers users the functionality to leverage stake and receive leveraged yield. This can be achieved by a **borrowed** token being wrapped or exchanged into a staking token that is pegged to the borrow token’s value.

Note that the user may instead deposit the borrow token directly.

One of the tokens Notional uses is Ethena's `USDe` and `sUSDe`. The user would be receiving leveraged `USDe` yield. In the case of Ethena/Notional the borrowed token is DAI.

Once a user wants to exit `BaseStakingVault::_redeemFromNotional` is called. There are two options instant redemption or through the withdraw request functionality. If instant redemption is used `EthenaLib::_sellStakedUSDe` is called.
```javascript
    function _executeInstantRedemption(
        address /* account */,
        uint256 vaultShares,
        uint256 /* maturity */,
        RedeemParams memory params
    ) internal override returns (uint256 borrowedCurrencyAmount) {
        uint256 sUSDeToSell = getStakingTokensForVaultShare(vaultShares);

        // Selling sUSDe requires special handling since most of the liquidity
        // sits inside a sUSDe/sDAI pool on Curve.
        return EthenaLib._sellStakedUSDe(
            sUSDeToSell, BORROW_TOKEN, params.minPurchaseAmount, params.exchangeData, params.dexId
        );
    }
```

The `_sellStakedUSDe` function has two trades. The first one swapping from `sUSDe` to `sDAI`. The second is only executed if the borrow token is NOT `DAI` as seen in the code snippet bellow.

<details>

```javascript
    function _sellStakedUSDe(
        uint256 sUSDeAmount,
        address borrowToken,
        uint256 minPurchaseAmount,
        bytes memory exchangeData,
        uint16 dexId
    ) internal returns (uint256 borrowedCurrencyAmount) {
        Trade memory sDAITrade = Trade({
            tradeType: TradeType.EXACT_IN_SINGLE,
            sellToken: address(sUSDe),
            buyToken: address(sDAI),
            amount: sUSDeAmount,
            limit: 0, // NOTE: no slippage guard is set here, it is enforced in the second leg
                      // of the trade.
            deadline: block.timestamp,
            exchangeData: abi.encode(CurveV2Adapter.CurveV2SingleData({
                pool: 0x167478921b907422F8E88B43C4Af2B8BEa278d3A,
                fromIndex: 1, // sUSDe
                toIndex: 0 // sDAI
            }))
        });


        (/* */, uint256 sDAIAmount) = sDAITrade._executeTrade(uint16(DexId.CURVE_V2));


        // Unwraps the sDAI to DAI
        uint256 daiAmount = sDAI.redeem(sDAIAmount, address(this), address(this));
        
=>      if (borrowToken != address(DAI)) {
            Trade memory trade = Trade({
                tradeType: TradeType.EXACT_IN_SINGLE,
                sellToken: address(DAI),
                buyToken: borrowToken,
                amount: daiAmount,
                limit: minPurchaseAmount,
                deadline: block.timestamp,
                exchangeData: exchangeData
            });


            // Trades the unwrapped DAI back to the given token.
            (/* */, borrowedCurrencyAmount) = trade._executeTrade(dexId);
        } else {
            borrowedCurrencyAmount = daiAmount;
        }
    }
``` 

</details>

There is NO slippage protection on the first trade. The reason being that slippage is checked in the second trade. 
However the second trade is only executed in the borrow token is NOT `DAI`. This opens the possibility of the trade being sandwich attacked by MEV bots stealing large portions of user funds, if the borrowed token is DAI, because the second trade would NOT be executed. Hence no slippage at all would be enforced in the transaction. 

## Impact
Loss of funds due to sandwich attack 

## Code Snippet
https://github.com/sherlock-audit/2024-06-leveraged-vaults/blob/14d3eaf0445c251c52c86ce88a84a3f5b9dfad94/leveraged-vaults-private/contracts/vaults/staking/protocols/Ethena.sol#L124-L167

## Tool used
Manual Review

## Recommendation
Add a slippage parameter to the first trade as well or use the `minPurchaseAmount` parameter as a minAmountOut:
```diff
    function _sellStakedUSDe(        
        uint256 sUSDeAmount,
        address borrowToken,
        uint256 minPurchaseAmount,
        bytes memory exchangeData,
        uint16 dexId
    ) {

...

        uint256 daiAmount = sDAI.redeem(sDAIAmount, address(this), address(this));

        if (borrowToken != address(DAI)) {
            Trade memory trade = Trade({
                tradeType: TradeType.EXACT_IN_SINGLE,
                sellToken: address(DAI),
                buyToken: borrowToken,
                amount: daiAmount,
                limit: minPurchaseAmount,
                deadline: block.timestamp,
                exchangeData: exchangeData
            });

            // Trades the unwrapped DAI back to the given token.
            (/* */, borrowedCurrencyAmount) = trade._executeTrade(dexId);
        } else {
            borrowedCurrencyAmount = daiAmount;
+           require(borrowedCurrencyAmount >= minPurchaseAmount, "NotEnoughtAmountOut");
        }
    }
```
