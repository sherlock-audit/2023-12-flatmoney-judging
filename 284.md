Shiny Orange Barbel

high

# in `oracleModule.sol` contract if price.expo is less than 0, wrong prices will be recorded

## Summary
oracleModule:if price.expo is less than 0, wrong prices will be recorded
## Vulnerability Detail
here look at the function ` _getOffchainPrice()`

```solidity 

  function _getOffchainPrice() internal view returns (uint256 price, uint256 timestamp, bool invalid) {
        IPyth oracle = offchainOracle.oracleContract;
        if (address(oracle) == address(0)) revert FlatcoinErrors.ZeroAddress("oracle");

        try oracle.getPriceNoOlderThan(offchainOracle.priceId, offchainOracle.maxAge) returns (
            PythStructs.Price memory priceData
        ) {
            timestamp = priceData.publishTime;

            // Check that Pyth price and confidence is a positive value
            // Check that the exponential param is negative (eg -8 for 8 decimals)
            if (priceData.price > 0 && priceData.conf > 0 && priceData.expo < 0) {
                price = ((priceData.price).toUint256()) * (10 ** (18 + priceData.expo).toUint256()); // convert oracle expo/decimals eg 8 -> 18

                // Check that Pyth price confidence meets minimum
                if (priceData.price / int64(priceData.conf) < int32(offchainOracle.minConfidenceRatio)) {
                    invalid = true; // price confidence is too low
                }
            } else {
                invalid = true;
            }
        } catch {
            invalid = true; // couldn't fetch the price with the asked input param
        }
    }
```

If price is 5e-5 for example, it will be recorded as 5e5 If price is 5e-6, it will be recorded as 5e6.

As we can see, there is a massive deviation in recorded price from actual price whenever price's exponent is negative
## Impact
Wrong prices will be recorded. For example, If priceA is 5e-5, and priceB is 5e-6. But due to the wrong conversion,

There is a massive change in price(5e5 against 5e-5)
we know that priceA is ten times larger than priceB, but priceA will be recorded as ten times smaller than priceB. Unfortunately, current payoff functions may not be able to take care of these discrepancies

## Code Snippet
https://github.com/sherlock-audit/2023-12-flatmoney/blob/bba4f077a64f43fbd565f8983388d0e985cb85db/flatcoin-v1/src/OracleModule.sol#L163-L187
## Tool used

Manual Review

## Recommendation
one of the mitigation for this issue is In OracleModule.sol, _prices should be mapping(uint256 => Price) private _prices;, where Price is a struct that stores the price and expo:
```solidity 
struct Price{
    Fixed6 price,
    int256 expo
}
```
This way, the price exponents will be preserved, and can be used to scale the prices correctly wherever it is used.  

also you can check this reference too: https://github.com/sherlock-audit/2023-07-perennial-judging/issues/56