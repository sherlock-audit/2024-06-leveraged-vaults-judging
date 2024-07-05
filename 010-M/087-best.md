Precise Vinyl Beetle

Medium

# The integration with `Kelp:WithdrawManager` is not correct

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
