Formal Eggshell Sidewinder

high

# Revert when adjusting the position



## Summary

The margin adjustment will revert unexpectedly when executing. Margin adjustment is time-sensitive as long traders often rely on it to increase their position's margin when their positions are on the verge of being liquidated to avoid liquidation.

If the margin adjustment fails to execute due to an inherent bug within its implementation, the user's position might be liquidated as the transaction to increase its margin fails to execute, leading to a loss of assets for the traders.

## Vulnerability Detail

When announcing an adjustment order, if the `marginAdjustment` is larger than 0, `marginAdjustment + totalFee` number of collateral will be transferred from the user account to the `DelayedOrder` contract. The `totalFee` comprises the keeper fee + trade fee.

Let's denote `marginAdjustment` as $M$, keeper fee as $K$, and trade fee as $T$. Thus, the balance of `DelayedOrder` is increased by $(M + K + T)$ rETH.

https://github.com/sherlock-audit/2023-12-flatmoney/blob/main/flatcoin-v1/src/DelayedOrder.sol#L303

```solidity
File: DelayedOrder.sol
217:     function announceLeverageAdjust(
..SNIP..
300:         // If user increases margin, fees are charged from their account.
301:         if (marginAdjustment > 0) {
302:             // Sending positive margin adjustment and both fees from the user to the delayed order contract.
303:             vault.collateral().safeTransferFrom(msg.sender, address(this), uint256(marginAdjustment) + totalFee);
304:         }
```

When the keeper executes the adjustment order, the following function calls will happen:

```solidity
DelayedOrder.executeOrder() => DelayedOrder._executeLeverageAdjust() => LeverageModule.executeAdjust()
```

https://github.com/sherlock-audit/2023-12-flatmoney/blob/main/flatcoin-v1/src/LeverageModule.sol#L137

```solidity
File: LeverageModule.sol
147:     function executeAdjust(
..SNIP..
234:         // Sending keeper fee from order contract to the executor.
235:         vault.sendCollateral({to: _keeper, amount: _order.keeperFee});
```

When the `LeverageModule.executeAdjust` function is executed, it will instruct the FlatCoin vault to send the $K$ rETH (keeper fee) to the keeper address. Thus, the amount of rETH held by the vault is reduced by $K$. However, in an edge case where the amount of rETH held by the vault is $V$ and $V < K$, the transaction will revert. 

This is because, at this point, the `DelayedOrder` contract has not forwarded the keeper fee it collected from the users to the FlatCoin vault yet. The keeper fee is only forwarded to the vault after the `executeAdjust` function is executed at Line 608 below, which is too late in the above-described edge case.

This edge case might occur if there is low liquidity in the vault, a high keeper fee in the market, or a combination of both. Thus, the implementation should not assume that there is always sufficient liquidity in the vault to pay the keeper in advance and collect the keeper fee from the `DelayedOrder` contract later.

https://github.com/sherlock-audit/2023-12-flatmoney/blob/main/flatcoin-v1/src/FlatcoinVault.sol#L303

```solidity
File: DelayedOrder.sol
586:     /// @notice Execution of user delayed leverage adjust order.
587:     /// @dev Uses the Pyth network price to execute.
588:     /// @param account The user account which has a pending order.
589:     function _executeLeverageAdjust(address account) internal {
590:         FlatcoinStructs.Order memory order = _announcedOrder[account];
591:         FlatcoinStructs.AnnouncedLeverageAdjust memory leverageAdjust = abi.decode(
592:             order.orderData,
593:             (FlatcoinStructs.AnnouncedLeverageAdjust)
594:         );
595: 
596:         _prepareExecutionOrder(account, order.executableAtTime);
597: 
598:         ILeverageModule(vault.moduleAddress(FlatcoinModuleKeys._LEVERAGE_MODULE_KEY)).executeAdjust({
599:             account: account,
600:             keeper: msg.sender,
601:             order: order
602:         });
603: 
604:         if (leverageAdjust.marginAdjustment > 0) {
605:             // Sending positive margin adjustment and fees from delayed order contract to the vault
606:             vault.collateral().safeTransfer({
607:                 to: address(vault),
608:                 value: uint256(leverageAdjust.marginAdjustment) + leverageAdjust.tradeFee + order.keeperFee
609:             });
610:         }
```

## Impact

The margin adjustment will revert unexpectedly when executing, as shown in the scenario above. Margin adjustment is time-sensitive as long traders often rely on it to increase their position's margin when their positions are on the verge of being liquidated to avoid liquidation.

If the margin adjustment fails to execute due to an inherent bug within its implementation, the user's position might be liquidated as the transaction to increase its margin fails to execute, leading to a loss of assets for the traders.

## Code Snippet

https://github.com/sherlock-audit/2023-12-flatmoney/blob/main/flatcoin-v1/src/DelayedOrder.sol#L303

https://github.com/sherlock-audit/2023-12-flatmoney/blob/main/flatcoin-v1/src/LeverageModule.sol#L137

https://github.com/sherlock-audit/2023-12-flatmoney/blob/main/flatcoin-v1/src/FlatcoinVault.sol#L303

## Tool used

Manual Review

## Recommendation

Consider transferring the keeper fee to the vault first before sending it to the keeper's address to ensure that the transfer of the keeper fee will work under all circumstances.

```diff
function _executeLeverageAdjust(address account) internal {
    FlatcoinStructs.Order memory order = _announcedOrder[account];
    FlatcoinStructs.AnnouncedLeverageAdjust memory leverageAdjust = abi.decode(
        order.orderData,
        (FlatcoinStructs.AnnouncedLeverageAdjust)
    );

    _prepareExecutionOrder(account, order.executableAtTime);

+    if (leverageAdjust.marginAdjustment > 0) {
+        // Sending positive margin adjustment and fees from delayed order contract to the vault
+        vault.collateral().safeTransfer({
+            to: address(vault),
+            value: uint256(leverageAdjust.marginAdjustment) + leverageAdjust.tradeFee + order.keeperFee
+        });
+    }

    ILeverageModule(vault.moduleAddress(FlatcoinModuleKeys._LEVERAGE_MODULE_KEY)).executeAdjust({
        account: account,
        keeper: msg.sender,
        order: order
    });

-    if (leverageAdjust.marginAdjustment > 0) {
-        // Sending positive margin adjustment and fees from delayed order contract to the vault
-        vault.collateral().safeTransfer({
-            to: address(vault),
-            value: uint256(leverageAdjust.marginAdjustment) + leverageAdjust.tradeFee + order.keeperFee
-        });
-    }
```