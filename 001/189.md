Formal Eggshell Sidewinder

high

# Discrepancies in the data used during the invariant checks leading to liquidation issue

## Summary

The liquidation might not execute as intended due to discrepancies in the data used during the invariant checks, leading to underwater positions and bad debt accumulated in the protocol, threatening the solvency of the protocol.

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

The `liquidationInvariantChecks` modifier will trigger the `_stableCollateralPerShareLiquidation` function internally. This function will compute the `expectedStableCollateralPerShare` based on the remaining margin (also called `settledMargin`), as shown in Line 130. Depending on whether the remaining margin is positive or negative, the logic for computing the `expectedStableCollateralPerShare` is different. 

If the remaining margin is positive, the LP will obtain whatever margin is left after sending the keeper fee. Thus, the `expectedStableCollateralPerShare` is likely to be the same or increase (First Branch). On the other hand, if the remaining margin is unhealthy, the LP will incur a loss, and `expectedStableCollateralPerShare` is likely to decrease (Second Branch).

The condition and logic at Line 130-151 below within the `InvariantChecks._stableCollateralPerShareLiquidation` function must mirror exactly the code at Line 109-143 above within the `LiquidationModule.liquidate` function. One slight discrepancy will cause the execution flow to switch to the wrong branch in the if-else statement. For instance, the ``LiquidationModule.liquidate` function executes the branch where (`settledMargin => 0`) while the invariant check executes the branch where (`settledMargin < 0`).

Thus, if there is any slight discrepancy or desync, the `expectedStableCollateralPerShare` computed will be inaccurate, leading to an unexpected revert during the check at Line 152 below during the liquidation process. It was found that there exists a discrepancy that led to this issue, and it will be described further below.

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

It was found that the discrepancy between the remaining margin (`settledMargin`) used within these two functions.

- In the `LiquidationModule.liquidate` function, the `settledMargin` is retrieved after the `vault.settleFundingFees` is executed, which settles the funding fees. As such, when the `settledMargin` of the liquidated position is retrieved, it will take into comparison the funding fees earned or lost till now.
- In the `InvariantChecks._stableCollateralPerShareLiquidation` function, the `settledMargin` is computed and store within the `invariantBefore` variable at Line 59 below before the `liquidate` function is executed at Line 69 below. Thus, the funding fee is not settled yet when the `settledMargin` is computed. The `settledMargin` without funding fee settlement will then be passed into the `_stableCollateralPerShareLiquidation` for use later at Line 78 below after the liquidation execution is completed.

https://github.com/sherlock-audit/2023-12-flatmoney/blob/main/flatcoin-v1/src/misc/InvariantChecks.sol#L56

```solidity
File: InvariantChecks.sol
56:     modifier liquidationInvariantChecks(IFlatcoinVault vault, uint256 tokenId) {
57:         IStableModule stableModule = IStableModule(vault.moduleAddress(FlatcoinModuleKeys._STABLE_MODULE_KEY));
58: 
59:         InvariantLiquidation memory invariantBefore = InvariantLiquidation({ // helps with stack too deep
60:             collateralNet: _getCollateralNet(vault),
61:             stableCollateralPerShare: stableModule.stableCollateralPerShare(),
62:             remainingMargin: ILeverageModule(vault.moduleAddress(FlatcoinModuleKeys._LEVERAGE_MODULE_KEY))
63:                 .getPositionSummary(tokenId)
64:                 .marginAfterSettlement,
65:             liquidationFee: ILiquidationModule(vault.moduleAddress(FlatcoinModuleKeys._LIQUIDATION_MODULE_KEY))
66:                 .getLiquidationFee(tokenId)
67:         });
68: 
69:         _; // execute LiquidationModule.liquidate()
70: 
..SNIP..
78:         _stableCollateralPerShareLiquidation(
79:             stableModule,
80:             invariantBefore.liquidationFee,
81:             invariantBefore.remainingMargin,
82:             invariantBefore.stableCollateralPerShare,
83:             invariantAfter.stableCollateralPerShare
84:         );
..SNIP..
```

## Impact

Liquidation is the core component of the protocol and is important to the solvency of the protocol. If the liquidation does not execute as intended, such as in the scenario mentioned above where the invariant check will revert unexpectedly, underwater positions and bad debt accumulate in the protocol, threatening the solvency of the protocol.

## Code Snippet

https://github.com/sherlock-audit/2023-12-flatmoney/blob/main/flatcoin-v1/src/LiquidationModule.sol#L85

https://github.com/sherlock-audit/2023-12-flatmoney/blob/main/flatcoin-v1/src/misc/InvariantChecks.sol#L118

https://github.com/sherlock-audit/2023-12-flatmoney/blob/main/flatcoin-v1/src/misc/InvariantChecks.sol#L56

## Tool used

Manual Review

## Recommendation

Ensure that `settledMargin` calculated within `LiquidationModule.liquidate` and `InvariantChecks._stableCollateralPerShareLiquidation` function are consistent. 

One possible solution is to settle the funding fee at the start of the `liquidationInvariantChecks` modifier.

```diff
modifier liquidationInvariantChecks(IFlatcoinVault vault, uint256 tokenId) {
+	// Settle funding fees accrued till now.
+	vault.settleFundingFees();

    IStableModule stableModule = IStableModule(vault.moduleAddress(FlatcoinModuleKeys._STABLE_MODULE_KEY));

    InvariantLiquidation memory invariantBefore = InvariantLiquidation({ // helps with stack too deep
        collateralNet: _getCollateralNet(vault),
        stableCollateralPerShare: stableModule.stableCollateralPerShare(),
        remainingMargin: ILeverageModule(vault.moduleAddress(FlatcoinModuleKeys._LEVERAGE_MODULE_KEY))
            .getPositionSummary(tokenId)
            .marginAfterSettlement,
        liquidationFee: ILiquidationModule(vault.moduleAddress(FlatcoinModuleKeys._LIQUIDATION_MODULE_KEY))
            .getLiquidationFee(tokenId)
    });
```