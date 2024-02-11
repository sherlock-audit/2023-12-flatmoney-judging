Scrawny Mossy Turtle

medium

# LimitOrder._closePosition() Revert for the case of within-range, leading to failure in normal casess.

## Summary
LimitOrder._closePosition() Revert for the case of within-range, leading to failure in normal casess.

## Vulnerability Detail

1. _closePosition() will check the price range as follows:

```javascipt
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

While we expect that when the price is within the range, the execution will go normally. However, the above code will revert even when the price is within the range. In other words, the code will revert when price is normal. 

## Impact
LimitOrder._closePosition() reverts for the case of within-range, leading to failure in normal casess.


## Code Snippet

[https://github.com/sherlock-audit/2023-12-flatmoney/blob/bba4f077a64f43fbd565f8983388d0e985cb85db/flatcoin-v1/src/LimitOrder.sol#L151-L161](https://github.com/sherlock-audit/2023-12-flatmoney/blob/bba4f077a64f43fbd565f8983388d0e985cb85db/flatcoin-v1/src/LimitOrder.sol#L151-L161)

## Tool used
VCode

Manual Review

## Recommendation
Change the logic so that normal price will not revert.
