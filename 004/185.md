Formal Eggshell Sidewinder

high

# Incorrect sign being used when checking skew

## Summary

The incorrect sign is being used when checking skew, leading to the `checkSkewMax` function reverting unexpectedly. As a result, the LPs will be unable to withdraw their assets under certain conditions.

## Vulnerability Detail

Following is the comment on the `checkSkewMax` function. Noted that the `_additionalSkew` is the additional skew added by either opening a long or closing an LP position.

```solidity
File: FlatcoinVault.sol
294:     /// @notice Asserts that the system will not be too skewed towards longs after additional skew is added (position change).
295:     /// @param _additionalSkew The additional skew added by either opening a long or closing an LP position.
296:     function checkSkewMax(uint256 _additionalSkew) public view {
```

At Line 124 below, the `stableModule.stableWithdrawQuote` function will return the amount of collateral (rETH) that the callers are expected to receive when withdrawing `withdrawAmount` amount of UNIT token. Note that the amount of collateral received (`expectedAmountOut`) is a positive value and is stored in the `uint256` variable.

In Line 132 below, the `vault.checkSkewMax` function is executed with `additionalSkew` set to a positive `expectedAmountOut`, which is incorrect. This is because when collateral is removed from the system, the `additionalSkew` should be a negative value indicating an outflow of collateral (rETH) from the system.

https://github.com/sherlock-audit/2023-12-flatmoney/blob/main/flatcoin-v1/src/DelayedOrder.sol#L132

```solidity
File: DelayedOrder.sol
109:     function announceStableWithdraw(
110:         uint256 withdrawAmount,
111:         uint256 minAmountOut,
112:         uint256 keeperFee
113:     ) external whenNotPaused {
114:         uint64 executableAtTime = _prepareAnnouncementOrder(keeperFee);
115: 
116:         IStableModule stableModule = IStableModule(vault.moduleAddress(FlatcoinModuleKeys._STABLE_MODULE_KEY));
117:         uint256 lpBalance = IERC20Upgradeable(stableModule).balanceOf(msg.sender);
118: 
119:         if (lpBalance < withdrawAmount)
120:             revert FlatcoinErrors.NotEnoughBalanceForWithdraw(msg.sender, lpBalance, withdrawAmount);
121: 
122:         // Check that the requested minAmountOut is feasible
123:         {
124:             uint256 expectedAmountOut = stableModule.stableWithdrawQuote(withdrawAmount);
125: 
126:             if (keeperFee > expectedAmountOut) revert FlatcoinErrors.WithdrawalTooSmall(expectedAmountOut, keeperFee);
127: 
128:             expectedAmountOut -= keeperFee;
129: 
130:             if (expectedAmountOut < minAmountOut) revert FlatcoinErrors.HighSlippage(expectedAmountOut, minAmountOut);
131: 
132:             vault.checkSkewMax({additionalSkew: expectedAmountOut});
133:         }
```

## Impact

The `checkSkewMax` check will revert unexpectedly, leading to LPs being unable to withdraw their assets.

## Code Snippet

https://github.com/sherlock-audit/2023-12-flatmoney/blob/main/flatcoin-v1/src/DelayedOrder.sol#L132

## Tool used

Manual Review

## Recommendation

Consider negating the `expectedAmountOut` when checking the skew to indicate an outflow of collateral (rETH) from the system.

```diff
- vault.checkSkewMax({additionalSkew: expectedAmountOut});
+ vault.checkSkewMax({additionalSkew: -expectedAmountOut});
```