Formal Eggshell Sidewinder

medium

# Leverage Calculation

## Summary

The formula for calculating leverage in the Flatcoin protocol does not follow the industry-standard method for calculating leverage. As a result, it might lead to a different interpretation of "leverage" in the context of the Flatcoin protocol compared to traditional finance. Users might be confused, leading to wrong trading decisions being made and potentially incurring losses.

## Vulnerability Detail

Flatcoin protocol uses the following formula to calculate leverage:

$$
\text{Leverage} = \frac{(\text{Margin} + \text{Size}) \times 1e18}{\text{Margin}}
$$

In this case, the formula adds the margin to the size and then divides it by the margin, which deviates from the standard way of calculating leverage.

https://github.com/sherlock-audit/2023-12-flatmoney/blob/main/flatcoin-v1/src/LeverageModule.sol#L416

```solidity
File: LeverageModule.sol
413:     /// @notice Asserts that the position to be opened meets margin and size criteria.
414:     /// @param _margin The margin to be deposited.
415:     /// @param _size The size of the position.
416:     function checkLeverageCriteria(uint256 _margin, uint256 _size) public view {
417:         uint256 leverage = ((_margin + _size) * 1e18) / _margin;
..SNIP..
```

In traditional finance and most trading platforms, leverage is usually calculated as the ratio of the total position size to the trader's own invested capital (margin), as shown below:

**Standard Leverage Calculation**:

$$
\text{Leverage} = \frac{\text{Total Position Size}}{\text{Trader's Margin}}
$$

For example, if a trader uses \$1,000 of their own money (margin) to control a $10,000 position, the leverage is 10x.

## Impact

This might lead to a different interpretation of "leverage" in the context of the Flatcoin protocol compared to traditional finance. Users might be confused, leading to wrong trading decisions being made and potentially incurring losses.

## Code Snippet

https://github.com/sherlock-audit/2023-12-flatmoney/blob/main/flatcoin-v1/src/LeverageModule.sol#L416

## Tool used

Manual Review

## Recommendation

Consider using the leverage calculation commonly used in the industry, as shown below:

$$
\text{Leverage} = \frac{\text{Total Position Size}}{\text{Trader's Margin}}
$$