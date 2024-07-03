Modern Amber Blackbird

High

# _claimRewardToken() will update accountRewardDebt even when there is a failure during reward claiming, as a result, a user might lose rewards.

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
