Boxy Garnet Wolverine

high

# Announced orders of a position are not deleted when liquidation happens

## Summary
Announced orders of a position are not deleted when liquidation happens

## Vulnerability Detail
When a position is liquidated, an existing announced order for that position is not deleted. This means that the user's order can still be executed after liquidation of the position. This is only an issue for leverageAdjut orders, and only if the points module is removed (and this has been said to be a temporary feature by the devs, so the points module is likely to be removed in the future).

## Impact
Once the points module is removed, a user's leverageAdjust order with positive `additionalSizeAdjustment` will still execute after the position has been liquidated.

## Code Snippet
https://github.com/sherlock-audit/2023-12-flatmoney/blob/main/flatcoin-v1/src/LeverageModule.sol#L147

## Tool used
Manual Review

## Recommendation
Delete announced orders of a position when the position is liquidated