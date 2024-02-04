Formal Eggshell Sidewinder

high

# Incorrect skew check formula used during withdrawal

## Summary

The purpose of the long max skew (`skewFractionMax`) is to prevent the FlatCoin holders from being increasingly short. However, the existing controls are not adequate, resulting in the long skew exceeding the long max skew deemed acceptable by the protocol, as shown in the example in this report. 

When the  FlatCoin holders are overly net short, an increase in the collateral price (rETH) leads to a more pronounced decrease in the price of UNIT, amplifying the risk and loss of the FlatCoin holders and increasing the risk of UNIT's price going to 0.

## Vulnerability Detail

When the users withdraw their collateral (rETH) from the system, the skew check (`checkSkewMax()`) at Line 132 will be executed to ensure that the withdrawal does not cause the system to be too skewed towards longs and the skew is still within the `skewFractionMax`.

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

However, using the `checkSkewMax` function for checking skew when LPs/stakers withdraw collateral from the system is incorrect. The `checkSkewMax` function is specifically used when there is a change in position size on the long-trader side.

In Line 303 below, the numerator of the formula holds the collateral/margin size of the long traders, while the denominator of the formula holds the collateral size of the LPs. When the LP withdraws collateral from the system, it should be deducted from the denominator. Thus, the formula is incorrect to be used in this scenario.

https://github.com/sherlock-audit/2023-12-flatmoney/blob/main/flatcoin-v1/src/FlatcoinVault.sol#L303

```solidity
File: FlatcoinVault.sol
294:     /// @notice Asserts that the system will not be too skewed towards longs after additional skew is added (position change).
295:     /// @param _additionalSkew The additional skew added by either opening a long or closing an LP position.
296:     function checkSkewMax(uint256 _additionalSkew) public view {
297:         // check that skew is not essentially disabled
298:         if (skewFractionMax < type(uint256).max) {
299:             uint256 sizeOpenedTotal = _globalPositions.sizeOpenedTotal;
300: 
301:             if (stableCollateralTotal == 0) revert FlatcoinErrors.ZeroValue("stableCollateralTotal");
302: 
303:             uint256 longSkewFraction = ((sizeOpenedTotal + _additionalSkew) * 1e18) / stableCollateralTotal;
304: 
305:             if (longSkewFraction > skewFractionMax) revert FlatcoinErrors.MaxSkewReached(longSkewFraction);
306:         }
307:     }
```

Let's make a comparison between the current formula and the expected (correct) formula to determine if there is any difference:

Let `sizeOpenedTotal` be $SO_{total}$, `stableCollateralTotal` be $SC_{total}$ and `_additionalSkew` be $AS$. Assume that the `sizeOpenedTotal` is 100 ETH and `stableCollateralTotal` is 100 ETH. Thus, the current `longSkewFraction` is zero as both the long and short sizes are the same.

Assume that someone intends to withdraw 20 ETH collateral from the system. Thus, the `_additionalSkew` will be 20 ETH.

**Current Formula**

$$
\begin{align} 
skewFrac = \frac{SO_{total} + AS}{SC_{total}} \\
skewFrac = \frac{100 + 20}{100} = 1.2
\end{align}
$$

**Expected (correct) formula**

$$
\begin{align} 
skewFrac = \frac{SO_{total}}{SC_{total} - AS} \\
skewFrac = \frac{100}{100 - 20} = 1.25
\end{align}
$$

Assume the `skewFractionMax` is 1.20 within the protocol.

The first formula will indicate that the long skew after the withdrawal will not exceed the long max skew, and thus, the withdrawal will proceed to be executed. Immediately after the execution is completed, the system exceeds the `skewFractionMax` of 1.2 as the current long skew has become 1.25.

## Impact

The purpose of the long max skew (`skewFractionMax`) is to prevent the FlatCoin holders from being increasingly short. However, the existing controls are not adequate, resulting in the long skew exceeding the long max skew deemed acceptable by the protocol, as shown in the example above. 

When the FlatCoin holders are overly net short, an increase in the collateral price (rETH) leads to a more pronounced decrease in the price of UNIT, amplifying the risk and loss of the FlatCoin holders and increasing the risk of UNIT's price going to 0.

## Code Snippet

https://github.com/sherlock-audit/2023-12-flatmoney/blob/main/flatcoin-v1/src/FlatcoinVault.sol#L303

## Tool used

Manual Review

## Recommendation

The `checkSkewMax` function is designed specifically for long trader's operations such as open, adjust, and close positions. They cannot be used interchangeably with the LP's operations, such as depositing and withdrawing stable collateral.

Consider implementing a new function that uses the following for calculating the long skew after withdrawal:

$$
skewFrac = \frac{SO_{total}}{SC_{total} - AS}
$$

Let `sizeOpenedTotal` be $SO_{total}$, `stableCollateralTotal` be $SC_{total}$ and `_additionalSkew` be $AS$. 