Formal Eggshell Sidewinder

high

# `marginDepositedTotal` can be significantly inflated

## Summary

The `marginDepositedTotal` can be significantly inflated due to an underflow that occurs when casting int256 to uint256, leading to core functionalities and accounting of the protocol being broken and assets being stuck.

## Vulnerability Detail

https://github.com/sherlock-audit/2023-12-flatmoney/blob/main/flatcoin-v1/src/FlatcoinVault.sol#L233

```solidity
File: FlatcoinVault.sol
216:     function settleFundingFees() public returns (int256 _fundingFees) {
..SNIP..
226:         // Calculate the funding fees accrued to the longs.
227:         // This will be used to adjust the global margin and collateral amounts.
228:         _fundingFees = PerpMath._accruedFundingTotalByLongs(_globalPositions, unrecordedFunding);
229: 
230:         // In the worst case scenario that the last position which remained open is underwater,
231:         // we set the margin deposited total to 0. We don't want to have a negative margin deposited total.
232:         _globalPositions.marginDepositedTotal = (int256(_globalPositions.marginDepositedTotal) > _fundingFees)
233:             ? uint256(int256(_globalPositions.marginDepositedTotal) + _fundingFees)
234:             : 0;
235: 
236:         _updateStableCollateralTotal(-_fundingFees);
```

The `_globalPositions.marginDepositedTotal` is uint256. Thus, the value assigned to this state variable must always be a non-negative value. The logic in Lines 232-234 intends to ensure that the system can never have a negative margin deposited total.

The `_fundingFees` funding fee at Line 228 above can be positive (gain by long traders) or negative (loss by long traders). 

Assume that the `_fundingFees` is `-20` (negative indicating a loss by long traders and a win for LP stakers) and the current `_globalPositions.marginDepositedTotal` is 10. 

When Line 232 above execute, the condition `(int256(_globalPositions.marginDepositedTotal) > _fundingFees)` equal to `(+10 > -20)` and evaluate to True.

Subseqently, Line 233 will be executed `uint256(int256(_globalPositions.marginDepositedTotal) + _fundingFees)`, which lead to the following result:

```solidity
➜ uint256(int256(10) - 20)
Type: uint
├ Hex: 0xfffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffff6
└ Decimal: 115792089237316195423570985008687907853269984665640564039457584007913129639926
```

An integer underflow due to unsafe casting has occurred, which results in `marginDepositedTotal` being set to a significantly large number (`115792089237316195423570985008687907853269984665640564039457584007913129639926`) and become significantly inflated.

The operation `int256(10) - 20` results in a negative value (-10). When casting `int256(-10)` to uint256, Instead of throwing an error, Solidity handles this underflow by wrapping the result to the maximum value that `uint256` can represent and then subtracts the deficit, which results in the above large number.

Note that Solidity does not automatically check for overflows or underflows when casting.

## Impact

The `marginDepositedTotal` is one of the most important states in the system, along with the `stableCollateralTotal`. It represents the total amount of margin deposited or owned by the long traders. The entire functioning of the protocol relies on the proper accounting of the `marginDepositedTotal`. If the `marginDepositedTotal` is incorrect, as shown in the above example, the protocol and the vault are effectively broken.

In addition, the `0xfffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffff6` value in the `marginDepositedTotal` is already close to the max number supported by uint256. Thus, most operations such as opening/adjusting/closing position, funding fee settlement, or PnL accruing that increase the `_globalPositions.marginDepositedTotal` further will not work as it will result in an overflow. Since users cannot close their positions, that also means that their assets are stuck within the system.

## Code Snippet

https://github.com/sherlock-audit/2023-12-flatmoney/blob/main/flatcoin-v1/src/FlatcoinVault.sol#L233

## Tool used

Manual Review

## Recommendation

Consider performing the calculation in `int256` and then checking if the result is positive before casting it back to `uint256`. If the result is negative or zero, you can set `_globalPositions.marginDepositedTotal` to 0.

```diff
+ int256 newMarginDepositedTotal = int256(_globalPositions.marginDepositedTotal) + _fundingFees;
// In the worst case scenario that the last position which remained open is underwater,
// we set the margin deposited total to 0. We don't want to have a negative margin deposited total.
- _globalPositions.marginDepositedTotal = (int256(_globalPositions.marginDepositedTotal) > _fundingFees)
+ _globalPositions.marginDepositedTotal = (newMarginDepositedTotal > 0)
-	? uint256(int256(_globalPositions.marginDepositedTotal) + _fundingFees)
+	? uint256(newMarginDepositedTotal)
	: 0;
```