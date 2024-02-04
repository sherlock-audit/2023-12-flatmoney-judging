Formal Eggshell Sidewinder

high

# Inconsistent in the margin transferred to LP during liquidation when settledMargin < 0

## Summary

There is a discrepancy in the approach of computing the expected gain and loss per share within the `liquidate` and `_stableCollateralPerShareLiquidation` functions, which will lead to an unexpected revert during the liquidation process.

Liquidation is the core component of the protocol and is important to the solvency of the protocol. If the liquidation does not execute as intended, underwater positions and bad debt accumulate in the protocol, threatening the solvency of the protocol.

## Vulnerability Detail

At Line 109 below, the `settledMargin` is greater than 0, a portion (or all) of the margin will be sent to the liquidator and LPs. If the `settledMargin` is negative, the LPs will bear the cost of the underwater position's loss.

When the `liquidate` function is executed, the `liquidationInvariantChecks` modifier will be triggered to perform invariant checks before and after the execution.

https://github.com/sherlock-audit/2023-12-flatmoney/blob/main/flatcoin-v1/src/LiquidationModule.sol#L85

```solidity
File: LiquidationModule.sol
085:     function liquidate(uint256 tokenId) public nonReentrant whenNotPaused liquidationInvariantChecks(vault, tokenId) {
086:         FlatcoinStructs.Position memory position = vault.getPosition(tokenId);
087: 
088:         (uint256 currentPrice, ) = IOracleModule(vault.moduleAddress(FlatcoinModuleKeys._ORACLE_MODULE_KEY)).getPrice();
089: 
090:         // Settle funding fees accrued till now.
091:         vault.settleFundingFees();
092: 
093:         // Check if the position can indeed be liquidated.
094:         if (!canLiquidate(tokenId)) revert FlatcoinErrors.CannotLiquidate(tokenId);
095: 
096:         FlatcoinStructs.PositionSummary memory positionSummary = PerpMath._getPositionSummary(
097:             position,
098:             vault.cumulativeFundingRate(),
099:             currentPrice
100:         );
101: 
102:         // Check that the total margin deposited by the long traders is not -ve.
103:         // To get this amount, we will have to account for the PnL and funding fees accrued.
104:         int256 settledMargin = positionSummary.marginAfterSettlement;
105: 
106:         uint256 liquidatorFee;
107: 
108:         // If the settled margin is greater than 0, send a portion (or all) of the margin to the liquidator and LPs.
109:         if (settledMargin > 0) {
// Do something
138:         } else {
139:             // If the settled margin is -ve then the LPs have to bear the cost.
140:             // Adjust the stable collateral total to account for user's profit/loss and the negative margin.
141:             // Note: We are adding `settledMargin` and `profitLoss` instead of subtracting because of their sign (which will be -ve).
142:             vault.updateStableCollateralTotal(settledMargin - positionSummary.profitLoss);
143:         }
```

The `liquidationInvariantChecks` modifier will trigger the `_stableCollateralPerShareLiquidation` function internally. This function will compute the `expectedStableCollateralPerShare` based on the remaining margin (also called `settledMargin`), as shown in Line 130 below. 

Assume that the liquidated position's current `settledMargin` is -3 (marginDeposit=2, accruedFee=0, PnL=-5). 

In this case, at Line 142 above within the `liquidate` function, the expected gain or loss of the LP is computed via `settledMargin - positionSummary.profitLoss`, which is equal to +2 (-3 - (-5))

However, within the invariant check (`_stableCollateralPerShareLiquidation`) below, at Line 150 below, the expected gain or loss of the LP is computed only via the `settledMargin` (remaining margin), which is equal to -3.

Thus, there is a discrepancy in the approach of computing the expected gain and loss per share within the `liquidate` and `_stableCollateralPerShareLiquidation` functions, and the `expectedStableCollateralPerShare` computed will deviate from the actual gain/loss per share during liquidation, leading to an unexpected revert during the check at Line 152 below during the liquidation process.

https://github.com/sherlock-audit/2023-12-flatmoney/blob/main/flatcoin-v1/src/misc/InvariantChecks.sol#L118

```solidity
File: InvariantChecks.sol
118:     function _stableCollateralPerShareLiquidation(
119:         IStableModule stableModule,
120:         uint256 liquidationFee,
121:         int256 remainingMargin,
122:         uint256 stableCollateralPerShareBefore,
123:         uint256 stableCollateralPerShareAfter
124:     ) private view {
125:         uint256 totalSupply = stableModule.totalSupply();
126: 
127:         if (totalSupply == 0) return;
128: 
129:         int256 expectedStableCollateralPerShare;
130:         if (remainingMargin > 0) {
131:             if (remainingMargin > int256(liquidationFee)) {
132:                 // position is healthy and there is a keeper fee taken from the margin
133:                 // evaluate exact increase in stable collateral
134:                 expectedStableCollateralPerShare =
135:                     int256(stableCollateralPerShareBefore) +
136:                     (((remainingMargin - int256(liquidationFee)) * 1e18) / int256(stableModule.totalSupply()));
137:             } else {
138:                 // position has less or equal margin than liquidation fee
139:                 // all the margin will go to the keeper and no change in stable collateral
140:                 if (stableCollateralPerShareBefore != stableCollateralPerShareAfter)
141:                     revert FlatcoinErrors.InvariantViolation("stableCollateralPerShareLiquidation");
142: 
143:                 return;
144:             }
145:         } else {
146:             // position is underwater and there is no keeper fee
147:             // evaluate exact decrease in stable collateral
148:             expectedStableCollateralPerShare =
149:                 int256(stableCollateralPerShareBefore) +
150:                 ((remainingMargin * 1e18) / int256(stableModule.totalSupply()));
151:         }
152:         if (
153:             expectedStableCollateralPerShare + 1e6 < int256(stableCollateralPerShareAfter) || // rounding error
154:             expectedStableCollateralPerShare - 1e6 > int256(stableCollateralPerShareAfter)
155:         ) revert FlatcoinErrors.InvariantViolation("stableCollateralPerShareLiquidation");
156:     }
```

## Impact

Liquidation is the core component of the protocol and is important to the solvency of the protocol. If the liquidation does not execute as intended, such as in the scenario mentioned above where the invariant check will revert unexpectedly, underwater positions and bad debt accumulate in the protocol, threatening the solvency of the protocol.

## Code Snippet

https://github.com/sherlock-audit/2023-12-flatmoney/blob/main/flatcoin-v1/src/misc/InvariantChecks.sol#L118

## Tool used

Manual Review

## Recommendation

Ensure that the formula used to compute the expected `StableCollateralPerShare` during the invariant check is consistent with the formula used within the actual liquidation function.

```diff
// position is underwater and there is no keeper fee
// evaluate exact decrease in stable collateral
expectedStableCollateralPerShare =
    int256(stableCollateralPerShareBefore) +
-   ((remainingMargin * 1e18) / int256(stableModule.totalSupply()));
+	(((remainingMargin - positionSummary.profitLoss) * 1e18) / int256(stableModule.totalSupply()));    
```