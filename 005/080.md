Hidden Laurel Cod

Medium

# The _getValueOfWithdrawRequest function uses different methods for selecting assets in various vaults.


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
