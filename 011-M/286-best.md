Powerful Cloud Elephant

medium

# `_profitLoss` function of the `PerpMath` calculate the `PnL` incorrectly.

## Summary
The calculation of `PnL` in `_profitLoss` function of `PerpMath` is wrong.

## Vulnerability Detail
The `_profitLoss` function calculates and returns `pnl` based on passed params of `position` and `price`.
```solidity
    function _profitLoss(FlatcoinStructs.Position memory position, uint256 price) internal pure returns (int256 pnl) {
        int256 priceShift = int256(price) - int256(position.lastPrice);
        int256 profitLossTimesTen = (int256(position.additionalSize) * (priceShift) * 10) / int256(price);

        if (profitLossTimesTen % 10 != 0) {
            return profitLossTimesTen / 10 - 1;
        } else {
            return profitLossTimesTen / 10;
        }
    }
```
First, following params may be passed to this function.
- `position.additionalSize` is 22
- `price` is 30
- `priceShift` is 3
In this case, the `profitLossTimesTen` is `22 * 3 * 10 / 30= 22`

Next, think of the following params.
- `position.additionalSize` is 20
- `price` is 30
- `priceShift` is 3
In this case, the `profitLossTimesTen` is `20 * 3 * 10 / 30 = 20`
In first condition the `PnL` is 1 and last conditin, `PnL` is 2.
This is not fair.

## Impact
Above wrong calculation may leads to loss of user's fund.

## Code Snippet
https://github.com/sherlock-audit/2023-12-flatmoney/blob/bba4f077a64f43fbd565f8983388d0e985cb85db/flatcoin-v1/src/libraries/PerpMath.sol#L175-L184

## Tool used

Manual Review

## Recommendation
The `_profitLoss` function should be updated as follow.
```diff
    function _profitLoss(FlatcoinStructs.Position memory position, uint256 price) internal pure returns (int256 pnl) {
        int256 priceShift = int256(price) - int256(position.lastPrice);
        int256 profitLossTimesTen = (int256(position.additionalSize) * (priceShift) * 10) / int256(price);

-       if (profitLossTimesTen % 10 != 0) {
-           return profitLossTimesTen / 10 - 1;
-       } else {
-           return profitLossTimesTen / 10;
-       }
+       return profitLossTimesTen / 10;
    }
```