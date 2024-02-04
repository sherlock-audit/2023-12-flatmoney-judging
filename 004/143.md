Orbiting Cyan Finch

medium

# In LeverageModule.executeOpen/executeAdjust, vault.checkSkewMax should be called after updating the global position data

## Summary

[[checkSkewMax](https://github.com/sherlock-audit/2023-12-flatmoney/blob/main/flatcoin-v1/src/FlatcoinVault.sol#L296)](https://github.com/sherlock-audit/2023-12-flatmoney/blob/main/flatcoin-v1/src/FlatcoinVault.sol#L296) is used to assert that the system will not be too skewed towards longs after additional skew is added. However, the `stableCollateralTotal` used by this function is a variable that will [[be updated by updateGlobalPositionData](https://github.com/sherlock-audit/2023-12-flatmoney/blob/main/flatcoin-v1/src/FlatcoinVault.sol#L205)](https://github.com/sherlock-audit/2023-12-flatmoney/blob/main/flatcoin-v1/src/FlatcoinVault.sol#L205). Therefore, `checkSkewMax` should be executed after `updateGlobalPositionData`. Otherwise, there is no guarantee whether newly opened positions will make the system more skew towards long side.

## Vulnerability Detail

```solidity
File: flatcoin-v1\src\LeverageModule.sol
080:     function executeOpen(
081:         address _account,
082:         address _keeper,
083:         FlatcoinStructs.Order calldata _order
084:     ) external whenNotPaused onlyAuthorizedModule returns (uint256 _newTokenId) {
......
101:->       vault.checkSkewMax({additionalSkew: announcedOpen.additionalSize});
102: 
103:         {
104:             // The margin change is equal to funding fees accrued to longs and the margin deposited by the trader.
105:->           vault.updateGlobalPositionData({
106:                 price: entryPrice,
107:                 marginDelta: int256(announcedOpen.margin),
108:                 additionalSizeDelta: int256(announcedOpen.additionalSize)
109:             });
......
140:     }
```

L101, [[vault.checkSkewMax](https://github.com/sherlock-audit/2023-12-flatmoney/blob/main/flatcoin-v1/src/FlatcoinVault.sol#L296-L307)](https://github.com/sherlock-audit/2023-12-flatmoney/blob/main/flatcoin-v1/src/FlatcoinVault.sol#L296-L307) internally calculates `longSkewFraction` by the formula `((_globalPositions.sizeOpenedTotal + _additionalSkew) * 1e18) / stableCollateralTotal`. This function guarantees that `longSkewFraction` will not exceed `skewFractionMax` ([[120%](https://github.com/sherlock-audit/2023-12-flatmoney/blob/main/flatcoin-v1/scripts/deployment/configs/FlatcoinVault.config.js#L9)](https://github.com/sherlock-audit/2023-12-flatmoney/blob/main/flatcoin-v1/scripts/deployment/configs/FlatcoinVault.config.js#L9)).

However, `stableCollateralTotal` will [[be updated in updateGlobalPositionData](https://github.com/sherlock-audit/2023-12-flatmoney/blob/main/flatcoin-v1/src/FlatcoinVault.sol#L205)](https://github.com/sherlock-audit/2023-12-flatmoney/blob/main/flatcoin-v1/src/FlatcoinVault.sol#L205).

- When `profitLossTotal` is positive value, then `stableCollateralTotal` will decrease.
- When `profitLossTotal` is negative value, then `stableCollateralTotal` will increase.

Assume the following:

```data
stableCollateralTotal = 90e18
_globalPositions = {  
    sizeOpenedTotal: 100e18,  
    lastPrice: 1800e18,  
}
A new position is to be opened with additionalSize = 5e18.  
fresh price=2000e18
```

We explain it in two situations:

1. `checkSkewMax` is called before `updateGlobalPositionData`.

```data
longSkewFraction = (_globalPositions.sizeOpenedTotal + additionalSize) * 1e18 / stableCollateralTotal 
                 = (100e18 + 5e18) * 1e18 / 90e18 
                 = 1.16667e18 < skewFractionMax(1.2e18)
so checkSkewMax will be passed.
```

2. `checkSkewMax` is called after `updateGlobalPositionData`.

```data
In updateGlobalPositionData:  
PerpMath._profitLossTotal calculates
profitLossTotal = _globalPositions.sizeOpenedTotal * (int256(price) - int256(globalPosition.lastPrice)) / int256(price) 
                = 100e18 * (2000e18 - 1800e18) / 2000e18 = 100e18 * 200e18 /2000e18 
                = 10e18 
_updateStableCollateralTotal(-profitLossTotal) will deduct 10e18 from stableCollateralTotal. 
so stableCollateralTotal = 90e18 - 10e18 = 80e18.  

Now, checkSkewMax is called:  
longSkewFraction = (_globalPositions.sizeOpenedTotal + additionalSize) * 1e18 / stableCollateralTotal 
                 = (100e18 + 5e18) * 1e18 / 80e18 
                 = 1.3125e18 > skewFractionMax(1.2e18)
```

Therefore, **this new position should not be allowed to open, as this will only make the system more skewed towards the long side**.

## Impact

The `stableCollateralTotal` used by `checkSkewMax` is the value of the total profit that has not yet been settled, which is old value. In this way, when the price of collateral rises, it will cause the system to be more skewed towards the long side.

## Code Snippet

https://github.com/sherlock-audit/2023-12-flatmoney/blob/main/flatcoin-v1/src/LeverageModule.sol#L101-L109

https://github.com/sherlock-audit/2023-12-flatmoney/blob/main/flatcoin-v1/src/LeverageModule.sol#L166

## Tool used

Manual Review

## Recommendation

```solidity
File: flatcoin-v1\src\LeverageModule.sol
080:     function executeOpen(
081:         address _account,
082:         address _keeper,
083:         FlatcoinStructs.Order calldata _order
084:     ) external whenNotPaused onlyAuthorizedModule returns (uint256 _newTokenId) {
......
101:---      vault.checkSkewMax({additionalSkew: announcedOpen.additionalSize});
102: 
103:         {
104:             // The margin change is equal to funding fees accrued to longs and the margin deposited by the trader.
105:             vault.updateGlobalPositionData({
106:                 price: entryPrice,
107:                 marginDelta: int256(announcedOpen.margin),
108:                 additionalSizeDelta: int256(announcedOpen.additionalSize)
109:             });
+++              vault.checkSkewMax(0); //0 means that vault.updateGlobalPositionData has added announcedOpen.additionalSize.
......
140:     }
```

Also, if `announcedAdjust.additionalSizeAdjustment` is greater than 0 in [[executeAdjust](https://github.com/sherlock-audit/2023-12-flatmoney/blob/main/flatcoin-v1/src/LeverageModule.sol#L166)](https://github.com/sherlock-audit/2023-12-flatmoney/blob/main/flatcoin-v1/src/LeverageModule.sol#L166), similar fix is required.